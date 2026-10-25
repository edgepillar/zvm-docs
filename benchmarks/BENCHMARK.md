<a id="evidence-status"></a>

> **Historical method and results.** These records describe the runs and
> corrections stated below. Their cited raw JSON runs are absent from this
> repository snapshot. [Evidence status](../EVIDENCE.md#historical-benchmarks)
> separates this retained narrative from reproducible measurements and current
> implementation or production claims.

# ZVM benchmark — method

**Measured 2026-09-15 on the ZVM prototype** (`C:\dev\zenon\zvm`), against its
own NoM network. Every number in the results section came from a run of
`zvm bench suite`; the raw per-transaction records are in `bench/results/*.json`
and the tables are regenerated with `zvm bench report`.

> **Which client these numbers are.** Everything below the *Results* heading was
> measured on **op-geth v1.101702.2**. The chains moved to **op-reth 1.10.2** on
> 2026-09-16 (D2), which is a different execution client and a different storage
> engine, so those numbers are a baseline to compare against rather than a
> description of what runs today.
>
> **Read [the 2026-09-17 fleet run](#results-2026-09-17--the-fleet-benchmark-re-run-on-both-chains)
> first.** That is the current measurement of every service at scale on both
> chains, in `bench/fleet-v1/` and `bench/fleet-v2/`, with tables in
> [`bench/FLEET-v1.md`](results/FLEET-v1.md) and
> [`bench/FLEET-v2.md`](results/FLEET-v2.md). For the single-workload matrix,
> [Both networks, after every fix](#both-networks-after-every-fix-2026-09-16-final)
> is still the reference, in `bench/results-v1-fixed/` and
> `bench/results-v2-fixed/`, with tables in
> [`bench/RESULTS-v1-fixed.md`](results/RESULTS-v1-fixed.md) and
> [`bench/RESULTS-v2-fixed.md`](results/RESULTS-v2-fixed.md).
>
> The op-reth sections between here and there are kept because each one is the
> evidence for a defect it uncovered, and each predates at least one fix:
> `results-reth-v1/` and `results-reth-v2/` are the first runs on the new
> client, `results-reth-v2-final/` is chain 71 after the re-bid timer but before
> the `Account.publish` deadlock was found, and `results-reth-v1-retest/` and
> `results-noncefix/` are the two controls that isolated the harness's nonce
> bug. **Where they disagree with the final section, the final section wins.**

Results follow in [the historical results](#results-2026-09-17--the-fleet-benchmark-re-run-on-both-chains). This section is what was measured and
what the numbers do and do not mean.

## What is under test

| Layer | What runs |
|---|---|
| L1 | go-zenon, **momentum version 1** (dynamic plasma inactive, as on mainnet): one pillar, one RPC node, chain 70, 10 s momentums |
| Execution | op-geth v1.101702.2, OP forks through Isthmus, 300 M gas limit per block |
| Derivation | `zvm node`: one momentum in, one block out, over the Engine API |
| Batch lane | `zvm relayer`: eight fused NoM accounts, zlib, ≤ 8 unconfirmed blocks each |
| Load | `zvm bench`, on the host, against the stack |

The L1 nodes and op-geth were containers throughout. The four Go services
(node, relayer, bridge, faucet) ran on the host for the matrix and as
containers for the corrected ERC-20 re-run, which is the only run whose
numbers depend on that difference; both talk to the same op-geth over the same
loopback ports, so it changes scheduling, not topology.

Host: 16 logical CPUs, 64 GB, Windows 11 with Docker Desktop. Everything — the
two Zenon nodes, op-geth, the services and the load generator — ran on that one
machine, so these are *co-located* numbers.

## The contracts

Ordinary ones, compiled unmodified:

- **OpenZeppelin v5** ERC-20, ERC-721 and ERC-1155 (solc 0.8.30, optimizer on, `--evm-version prague`)
- **Uniswap V2**, from its published npm artifacts: factory and router02, this tree's own WZNN wrapper, and a pair with liquidity
- a **storage-churn** contract that writes 20 cold slots per call
- a **disperse** helper, used only to fund benchmark wallets

`UniswapV2Router02` is worth naming: its init code is **22,398 bytes**, larger
than NoM's 16 KB account-block limit. It deployed through the ZVM's chunk and
assemble messages, which is the mechanism that keeps a 24 KB contract
deployable on a chain whose data blocks are 16 KB.

## The lanes

- **`batch`** — secp256k1 keys signing ordinary EIP-1559 transactions, exactly
  as MetaMask or Rabby would, submitted to the relayer, which packs them into
  NoM blocks and pays the plasma. Gas is paid on the ZVM.
- **`native`** — ed25519 Zenon keys publishing account blocks to the inbox,
  exactly as the Syrius extension does. No EVM signature; the gas budget is what
  the block's plasma bought; nothing is paid on the ZVM.

## What is measured

Per transaction: when it was submitted, which block it landed in, when that
block appeared, the gas it used, and whether it reverted. Per block: gas, the
ZVM's byte and plasma cost on NoM, how many NoM account blocks carried it, and
how long derivation took. Every 10 s: container CPU, memory and chaindata size.

**Latency** is submit → inclusion, wall clock. It includes the relayer's
batching wait (up to 1 s), the wait for the next momentum (up to ~10 s), and
derivation. It is not a measure of op-geth.

**Throughput** is transactions included ÷ the run's wall clock.

## What these numbers are not

- **Not mainnet.** Mainnet runs the same momentum version but with ~100 pillars
  and real network latency; its momentum interval is ~11.25 s and its blocks are
  capped at 100 account blocks per momentum, as here.
- **Not a sequencer's latency.** A based rollup is as fast as its L1. Ten
  seconds is the floor, not a bug.
- **Not multi-machine.** Everything shares one host, so network and disk
  contention are absent, and CPU contention is exaggerated.
- **Not a security claim.** Throughput says nothing about the peg, which is a
  committee (see `DECISIONS.md` D14).

## Reproducing

```sh
docker compose -f docker-compose.yml up -d --build
zvm bench setup                       # deploys the contract set
zvm bench suite                       # the matrix below
zvm bench report -in bench/results -out bench/RESULTS.md
```

For both chains, `bench.ps1 both` (a script in the deprecated zvm tree) ran the matrix against chain 70 and then
chain 71. It is **sequential on purpose**: they share one machine, and run
together the numbers measure the contention rather than the rulesets.

`bench suite` waits between steps for the relayer's queue to drain *and*, on a
dynamic-plasma chain, for `nextFusionPrice` to fall back under 1.2×. Both
matter: a saturating step can leave tens of thousands of transactions queued and
the price at 20×, and the next step would then measure the previous one. When it
gives up waiting it says so on stderr, and that step's row should be read with
suspicion.

---

## Results, 2026-09-17 — the fleet benchmark, re-run on both chains

**Read this section first.** It is the current measurement of every service at
scale, and it supersedes the fleet numbers further down: those were taken on
op-geth, before Karst, before the burned native fee, and before six defects in
the harness and three in the relayer were found by re-running it. Tables:
[`bench/FLEET-v1.md`](results/FLEET-v1.md) (chain 70) and
[`bench/FLEET-v2.md`](results/FLEET-v2.md) (chain 71); raw phases in
`bench/fleet-v1/` and `bench/fleet-v2/`.

### What had to be fixed before it measured anything

The first attempt carried **63.7 tx/s against 200 offered** on chain 70 while
the L1 sat idle. Nothing was wrong with the chain:

- the relayer's tracker asked the execution client for **a receipt per
  transaction per batch**, fell minutes behind, and left every account parked
  at `MaxInFlight` waiting for confirmations that had already happened;
- `headFor` held **one mutex across `eth_getBalance`**, and since its cache is
  emptied every block, 100,000 distinct senders serialized the whole lane;
- `pickShard` **counted every sender, for every shard, on each new sender's
  first submission**, so submissions decayed from 200/s to 35/s as senders
  accumulated;
- at 600 tx/s the relayer **ran out of ephemeral ports**, because it dialled
  the engine on Go's default transport (two idle connections per host) — the
  pool the load generator got in the first benchmark, which the services never
  did.

The harness was also mismeasuring: it counted **910 reverts that never
happened** (a block whose receipts it failed to fetch left those records at
status 0), refused **787 of 1,000 withdrawals** for want of funding it had not
sized, reverted 409 of 599 farm calls on estimates taken from stale state, and
could not publish on the native lane at all once native calls were priced —
they declared no gas, and their wallets held no ZNN to burn. Each fix is in
`progress.md` under 2026-09-17, with a test that fails without it.

### Chain 70 (mainnet's rules): all 22 phases, 1 h 26 m

Zero phase errors. No L1 repairs needed. The settle wait — which holds a phase
until the relayer is empty and the fees are back near their floor — never gave
up. The two independent nodes agree at block **4,101** with matching state
roots, and the follower's engine restart recovered in **5.3 s**.

| Phase | Offered | Included | Failed | Reverted | tx/s | p50 s | NoM blk/momentum |
|---|---:|---:|---:|---:|---:|---:|---:|
| 1k wallets | 200 | 23,988 | 0 | 0 | 200 offered, all landed | 16.5 | 58.4 |
| 10k wallets | 200 | 23,998 | 0 | 0 | 189.3 | 17.5 | 62.2 |
| 100k wallets | 200 | 99,768 | 0 | 0 | 200.1 | 17.8 | 61.7 |
| **Saturation** | 600 | 100,722 of 100,789 | 0 | 0 | **378.4** | 65.7 | 48.0 |
| Uniswap swaps | 100 | 11,999 | 0 | 0 | 69.4 | 15.5 | 45.0 |
| NFT mints | 100 | 11,999 | 0 | 0 | 101.0 | 10.3 | 52.2 |
| Marketplace listings | 100 | 11,999 | 0 | 0 | 102.1 | 14.9 | 58.6 |
| Farm | 100 | 11,999 | 0 | 0 | 92.8 | 12.0 | 49.1 |
| Native transfers | 15 | 1,799 | 0 | 0 | 7.1 | 64.0 | 99.9 |
| Native NFT mints | 6 | 719 | 0 | 0 | 5.5 | 9.0 | 79.9 |
| API under load | 200 | 23,984 | 0 | 0 | 185.7 | 16.8 | 64.0 |

The saturation row is the one to read carefully: **559.4 tx/s offered and
submitted** for three minutes, with 67 transactions still in flight when the
drain window closed and nothing refused. Wallet count is still not a limit —
1k, 10k and 100k all deliver the offered 200 tx/s — and the native lane holds
its 7.1 tx/s, the same figure the 2026-09-15 run measured.

Services, at the same scale: **1,000 of 1,000 deposits credited**, the vault's
receive queue drained in 187 s; **980 native accounts fused** and topped up
with ZNN to burn (plasma usable at p50 148 s); **900 media uploads at
93 MB/s** with every size and type limit enforced; **8 of 8 verifications**
exact. The faucet served one request of 60 and refused all four spoofing
probes — that is its hardened cooldown working, since every request here comes
from one host.

### Chain 71 needs its own offered rates, and that is a result

Dynamic plasma prices data, and a relayed transfer is data: a momentum admits
about 60 KB, a transfer costs ~90 NoM bytes, so the lane carries **~44 tx/s**
where chain 70 carries 200. The fleet's defaults oversubscribe chain 71 by 5x
(scaling) and 15x (saturation), and at that offering each phase measured the
previous phase's queue rather than the ruleset: the settle wait hit its
15-minute cap with 926 still queued at **10x the fusion price**. Its run is
therefore offered what it can carry, with a deliberately oversubscribed
saturation phase to find the ceiling.

## Results, 2026-09-16 — both chains rebuilt on Karst, both forks live

Everything in this section was measured on chains that were **destroyed and
re-genesised the same day** on op-reth/Karst, with the burned native fee from
momentum 10 and the gas and wire forks at momentum 200. Nothing here is
inherited from the op-geth numbers further down, and the two are not
comparable: the client, the EVM and the fork set all changed. Tables are in
[`bench/RESULTS-v1.md`](results/RESULTS-v1.md) and
[`bench/RESULTS-v2.md`](results/RESULTS-v2.md).

Measured momentum cadence is **≈15 s**, not the nominal 10 s. Every latency
below is against that, and every "offered" figure is what the harness tried to
submit.

### The two chains, side by side

Fourteen steps on each, two minutes apiece, every step started from a quiet
chain (see *Reproducing*). Full tables in [`bench/RESULTS-v1.md`](results/RESULTS-v1.md)
and [`bench/RESULTS-v2.md`](results/RESULTS-v2.md); superseded runs are kept in
each directory's `superseded/`.

| offered | scenario | v1 tx/s | v2 tx/s | v1 p50 | v2 p50 |
|---:|---|---:|---:|---:|---:|
| 5 | transfer, batch | 5.0 | 5.0 | 9.6s | 9.8s |
| 25 | transfer, batch | 25.0 | 25.0 | 10.6s | 11.9s |
| 100 | transfer, batch | **100.0** | 86.9 | 15.3s | 30.0s |
| 400 | transfer, batch | **399.3** | 125.4 | 15.7s | 101.4s |
| 25 | erc20, batch | 25.0 | 25.0 | 12.3s | 12.0s |
| 250 | erc20, batch | **201.6** | 129.5 | 18.3s | 109.2s |
| 25 | erc721, batch | 25.0 | 25.0 | 11.3s | 10.6s |
| 25 | erc1155, batch | 25.0 | 25.0 | 11.9s | 10.2s |
| 20 | swap, batch | 20.0 | 20.0 | 13.6s | 9.8s |
| 10 | churn, batch | 10.0 | 10.0 | 9.6s | 9.3s |
| 3 | deploy, batch | 3.0 | 3.0 | 9.9s | 10.7s |
| 5 | transfer, native | 5.0 | 5.0 | 8.0s | 7.9s |
| 15 | transfer, native | **13.5** | 8.2 | 61.0s | **9.9s** |
| 10 | erc20, native | **10.0** | 8.4 | 18.8s | **8.5s** |

**Up to 25 offered tx/s the two chains are the same chain.** Every step at or
below 25 delivers what is offered on both, within noise. Dynamic plasma costs
nothing until something is contended.

**Above that the batch lane diverges, by about 3.2× at saturation.** Chain 70
delivered **399.3 of 400 offered at p50 15.7 s** — it did not saturate, so its
ceiling is higher still. Chain 71 managed 125.4 at p50 101.4 s. The reason is
visible in the NoM columns rather than the EVM ones: at that step chain 70 put
**41 account blocks in each momentum** and chain 71 **8**.

**The native lanes make different trades rather than one beating the other.** At
15 offered, v1 includes more (13.5 against 8.2) by *queueing* — p50 61.0 s,
p90 166 s, sitting against the 100-blocks-a-momentum cap it reaches at 95.
v2 includes less but answers in 9.9 s, because it refuses at submission instead
of queueing. Note where v2 goes that v1 cannot: **181 account blocks in one
momentum**, because a v2 momentum is bounded by plasma rather than by a count.

**And the price is in the per-transaction cost.** The same native transfer costs
**23,176 plasma on v1 and 42,587 on v2** — not because the transaction is
different, but because a block pays `ceil(base × nextFusionPrice / 1000)` and
the price had risen under the rollup's own load.

### Two rules read off one saturated block

Block **#273** on chain 70 carried **10,716 transactions**:

| | |
|---|---:|
| `gasUsed` | 225,139,830 |
| `gasLimit` | 300,000,000 |
| relayed ceiling (`gasLimit − nativeFloor`) | **225,000,000** |
| `blobGasUsed` | **0** |
| `excessBlobGas` | 0 |

- **The elastic native share (D10) binds, exactly.** The batch lane stopped at
  its ceiling; the ~140 k above it is the mandatory L1-info deposit, which is
  not charged to the lane.
- **Jovian's DA-footprint rule is inert with a zero scalar** — the last open
  Karst question, and the answer is yes. A footprint that were computed and
  compared would not read zero on a block at 75 % of the gas limit.

### What a run costs, and who gets it

From momentum 1680 the chain prices its own transactions (D39): a minimum base
fee of **150 gwei**, senders offering a **100 gwei** tip, and a relayer that
refuses anything under **50 gwei**. Before that a relayed transaction paid
$0.00000084 and a native one $0.00084 — a 995× gap for identical work, and
about $6 of fees across 192,497 transactions.

Throughput is unchanged by it: the first repriced step delivered **24.98 tx/s of
25 offered**, with nothing rejected and nothing queued. What changed is the
money, and a benchmark step is now worth reporting in two currencies.

### A note on the fee market, for anyone comparing later runs

Every number in this section was measured **before the market fork** (D38),
with OP's inherited parameters: denominator 250, elasticity 6, a 50 M gas
target. From momentum 1440 both chains run **denominator 25, elasticity 30**,
a 10 M target. That does not change throughput — the gas limit and the elastic
native share are untouched — but it changes what a transaction *costs* during a
burst, and therefore what a sender's fee cap has to be. A matrix run across the
change is not comparable on fee, only on inclusion.

### The base fee does not follow NoM's price

Worth stating because the explorer now draws the two together on `/`: nothing
connects what the ZVM pays NoM to what its users pay. The base fee is ordinary
EIP-1559 over the ZVM's own gas, and through the whole benchmark these blocks
averaged **2.5 % full** against a 300 M limit, so the fee drifted *down*
(50.6 M → 30.5 M wei) while plasma per momentum spiked repeatedly — r = 0.16.
Over a window containing a saturation step the two read r = 0.71, but that is
one traffic surge lifting both, not the L1 reaching the L2. The L1's cost lands
on whoever posts the block, which is the subsidy [`docs/PLASMA.md`](../operations/PLASMA.md)
is about.

What *did* change after D38 is that the fee now answers the ZVM's own demand at
all. Measured immediately after the fork, 125 tx/s of load raised the base fee
**+5.4 % a block, 3.09 M → 4.06 M wei**, on blocks averaging ~23.5 M gas. The
same blocks under the old 50 M target would have moved it **−0.2 % a block**:
that level of activity was below target and therefore invisible.

### What dynamic plasma costs a rollup

This is the comparison the whole two-chain arrangement exists for, and the
answer has two halves: a fixed ceiling, and a price that moves.

**The ceiling.** A v2 momentum is capped at `MaxBasePlasmaInMomentum` —
4,200,000 by default, and a **hard** limit, enforced in both the pillar's
content selector and the verifier rather than an EIP-1559-style soft target. A
send costs `21,000 + 68 × len(data)`, so a full 16 KB NoM block costs
**1,135,112**:

| per momentum | v1 (chain 70) | v2 (chain 71) |
|---|---:|---:|
| rollup blocks admitted | 100, whatever their size | ~3.7 full ones |
| NoM bytes available | ~1.6 MB | ~60 KB |

Dynamic plasma prices data, and a rollup is data.

**The price.** Sampling chain 71's frontier momentum through the benchmark,
`nextFusionPrice` climbed from **1000 (1×) to 23,618 (23.6×)** under the
rollup's own load, while `nextWorkPrice` never moved from 1000 — this load is
fusion-paid, not PoW. Blocks admitted per momentum fell as it rose, from 36 to
4–11.

A block must pay `ceil(base × price / 1000)` in fused plasma, and **an account
cannot hold more than 10,500,000 plasma whatever it fuses**. Those two facts
meet:

| fusion price | a full 16 KB block needs | largest block one account can publish |
|---:|---:|---:|
| 1.00× | 1,135,112 | 16 KB (the protocol maximum) |
| 5.70× | 6,469,003 | 16 KB |
| **9.25×** | 10,500,000 | **16 KB — the threshold** |
| 20.17× | 22,898,614 — impossible | 7,346 bytes |
| 23.6× | 26,789,000 — impossible | 6,270 bytes |

Past **9.25×** a full NoM block cannot be published by any single account, so a
rollup under sustained load is forced into *smaller* blocks — which raises its
per-transaction overhead exactly when it needs throughput most, because each
block pays 21,000 base plasma however little it carries.

The relayer's logs say it plainly. Every account, repeatedly:

```
publishing batch … err="ledger.publishRawTransaction: rpc error -32000:
  not enough plasma on account" txs=13
```

That is the answer to the question the last matrix left open — why the v2 lane
throttles "at submission" rather than at inclusion. It is not a latency
signature to be interpreted: the node refuses the publish outright, because the
price rose underneath a block that was affordable when it was signed.

**It does not recover quickly.** With load removed the price decays by about
**5 % per momentum** — a half-life of roughly 13 momentums, or 3.4 minutes at
this devnet's ≈15 s cadence. Falling from 23.6× back to 1.2× takes about
**58 momentums, some 14 minutes**, during which the chain is still charging a
multiple for data nobody is contending for. A benchmark step that starts while the previous one's
price is still elevated is not measuring the same chain, so `bench suite` now
waits between steps for *both* the relayer's queue to drain and the fusion price
to fall back under 1.2× — a fixed 20-second pause was letting a saturating step
contaminate every step after it.

**And one measurement here was our own bug, not the protocol's.** The relayer's
footprint forecast converted its share into a *block count* on both rulesets,
costing every block as a full 16 KB one: chain 71 read `blocksPerMomentum: 3`
against chain 70's `89`. Its batches were ~2 KB, so its share actually paid for
about twenty-three. Any v1-vs-v2 number taken before 2026-09-16 carries that 30×
handicap on the v2 side. The budget is now held in the resource each ruleset
counts — blocks on v1, base plasma on v2 — and charged by each block's real size.

## Results, 2026-09-15 — op-geth, before Karst and before the forks

Kept because the method and most of the boundaries still hold, but the client,
the EVM and the fork set have all changed since: read the section above for what
these chains do now.

Fifteen runs, two minutes each, 2026-09-15: the fourteen-step matrix plus one
run offered 1,200 tx/s to find the batch lane's ceiling. Two early runs are
superseded by corrected re-runs and kept in `bench/results/superseded/`. The
tables are in [`bench/RESULTS.md`](results/RESULTS.md); this section says what
they mean.

### The headline

| | Native lane (Syrius) | Batch lane (MetaMask/Rabby) |
|---|---:|---:|
| Sustained inclusion, measured | **7.1 tx/s** | **~270 tx/s** |
| …on a full 10 s momentum schedule | 10 tx/s | ~385 tx/s |
| Most transactions in one block | 100 | **13,514** |
| NoM bytes per transfer | 32 | 62–74 at the ceiling |
| NoM plasma per transfer | 23,176 | 4,309–5,249 at the ceiling |
| NoM account blocks per momentum at the ceiling | 99.9 of 100 | 17.9–38.3 of 100 |
| Latency, p50 | 7.2 s unloaded | 15.6 s at 100 tx/s |

Both lanes carry ordinary EVM transactions with ordinary EVM gas costs — a
transfer is 21,000, an OpenZeppelin ERC-20 transfer 36,056, an ERC-721 mint
56,912, a Uniswap V2 swap 91,682. Nothing about being *based* changes what a
contract costs to run. What differs is what a transaction costs to **publish**,
and at the ceiling the two lanes differ there by a factor of four to five.

**Sustained**, here and in `RESULTS.md`, is not the `tx/s` column. `tx/s`
divides what was included by the two minutes of submission; *sustained* divides
what arrived after the first block carrying the run by the time from that block
to the last. They agree while the chain keeps up. Once it does not, inclusion
keeps draining after submission stops and `tx/s` overstates capacity — at 1,200
offered it reads 677.5 against a sustained 271.1. On a run only ten blocks long,
with gaps of 10 and 20 s, the sustained figure is good to about ±15 %.

### This L1 left a third of its momentum slots empty

Every gap between consecutive momentums, in every run, was **exactly 10 s or
exactly 20 s**, and 36–40 % were 20 s — at 5 tx/s as much as at saturation, so
it is not load. The network has one registered pillar (`embedded.pillar.getAll`
returns one, weight 0) and it produced every momentum there is; its log shows no
production attempt in the empty slots, so they were never its turn rather than
tried and missed. Why go-zenon's scheduling leaves them empty for a lone
zero-weight pillar was not established.

It matters because a based rollup inherits the L1's cadence exactly. The mean
momentum interval across runs was **12.0–15.7 s**, so every per-momentum limit
below becomes a per-second limit about 30 % lower than a 10 s schedule would
give. Mainnet's interval is about 11.25 s: the per-momentum figures carry over,
the per-second ones need rescaling.

### The native lane is capped by a count: 100 transactions a momentum

A native transaction is one NoM account block. A v1 momentum accepts **100
account blocks** (`go-zenon/chain/account_pool.go:28`), whatever they carry, so
the lane's ceiling is 100 transactions per momentum — 10 tx/s on a full 10 s
schedule, and **7.1 tx/s measured** on this L1's 14.1 s mean interval. A 32-byte
transfer and a 99-byte ERC-20 call occupy exactly one block each.

The measurement walks straight into it:

| Offered | Blocks per momentum | Sustained | p50 | p90 | max |
|---|---:|---:|---:|---:|---:|
| transfer, 5/s | 59.9 | 4.2 | 7.2 s | 16.6 s | 20.8 s |
| erc20, 10/s | 92.2 | 7.4 | 19.3 s | 67.6 s | 106.8 s |
| transfer, 15/s | **99.9** | **7.1** | 79.2 s | 159.3 s | 258.2 s |

At 60 % of capacity the lane is comfortable. At 92 % the p90 is already **four
times** the p50 — the ordinary shape of a queue approaching service rate, not a
bug. At 15 offered per second about 7 are accepted, the backlog never clears
inside the run, and the last transaction lands 258 s after it was sent.

Two consequences worth stating plainly. **This ceiling is NoM's, not the
ZVM's** — no change to derivation, op-geth or the node moves it, and on mainnet
it would be shared with every other user of the chain rather than ours alone.
And **a wallet that signs its own account blocks cannot be the only way in** if
the chain is ever busy: it is the trust-minimal path, not the scalable one.

The per-transaction plasma is an identity, which is a useful check on the
accounting: a 32-byte message costs `21,000 + 68 × 32 = 23,176` plasma
(`go-zenon/vm/constants/plasma.go:28`), and that is exactly what the harness
reports for both native transfer runs.

### The batch lane sustained ~270 tx/s, and NoM was never what stopped it

| Offered | Included | Sustained | Blocks/momentum | B/tx | Plasma/tx | Max tx/block | Build, mean / max |
|---|---:|---:|---:|---:|---:|---:|---:|
| transfer, 25/s | 2,999 | 22.7 | 55.2 | 88 | 9,834 | 471 | 162 / 261 ms |
| transfer, 100/s | 11,995 | 92.6 | 58.7 | 77 | 6,189 | 1,985 | 729 / 1,368 ms |
| erc20, 250/s | 29,920 | 185.5 | 49.2 | 86 | 6,198 | 4,271 | 1,569 / 2,808 ms |
| transfer, 400/s | 47,681 | **261.1** | 38.3 | 74 | 5,249 | 12,261 | 1,814 / 5,425 ms |
| transfer, 1,200/s | 81,312 of 134,286 | **271.1** | **17.9** | 62 | 4,309 | **13,514** | 1,863 / 7,200 ms |

Offered 400 and offered 1,200 converge on the same ~270 tx/s: that is the
ceiling. The counter-intuitive column is blocks per momentum — **the harder the
load, the fewer NoM blocks it used.** Heavier load makes the relayer's batches
larger, larger batches compress better (62 bytes per transfer at the ceiling
against 103 at the lightest load), and a batch that outgrows a 16 KB account
block goes in as chunks under one assemble message. At the ceiling the rollup
occupied **18 of the 100 account blocks** a momentum allows.

What bound it, in order:

1. **The relayer's cadence.** A chunked batch executes only when its assemble
   lands, so at 1,200 offered just **8 of 20** momentums carried a batch at all;
   the others derived empty blocks while chunks accumulated. Each momentum that
   did carry one included 10,164 transfers on average.
2. **The L2 block gas limit**, inside those blocks: the largest held 13,514
   transfers, 283.8 M gas of a 300 M limit (95 %).
3. **The L1's empty slots** (above): the same per-momentum capacity on a full
   10 s schedule would be about 385 tx/s.

Not on the list: op-geth, which used about 4 of 16 cores and built the largest
block in 7.2 s, inside its momentum; and NoM's capacity — account blocks,
plasma, block size, the pillar's ability to include them. The L1's *cadence*
cost a third of the throughput while its *capacity* sat mostly idle.

#### A defect this benchmark found

The first ERC-20 run failed 2,374 of 28,234 submissions with `Only one usage of
each socket address`, and the first 400/s run read 274.9 tx/s with a 13.4 s
block build and op-geth at 12.7 cores. Both were the relayer, not the chain: it
read the chain head *and* the sender's balance from op-geth on **every**
submission, two fresh connections per transaction, which exhausted the host's
ephemeral ports and buried op-geth in connection churn. Both quantities are
properties of a block — a base fee cannot change without one, and neither can a
balance — so they are now read once per block and cached against the head
(`internal/relayer/relayer.go`). Re-run under the same load: nothing refused,
op-geth at 3–4 cores, the largest build 5.4 s. The superseded runs are in
`bench/results/superseded/`.

At 1,200 offered, 949 submissions failed the same way at the *load generator*,
which opens its own connections to the relayer. That is the harness, not the
chain.

### Latency is a momentum, and stays one until saturation

p50 sits between 7.2 s and 15.6 s in every unsaturated run — one momentum,
which is the floor a based rollup has by construction. There is no sequencer to
make it smaller and no soft-confirmation to pretend otherwise. The relayer's
batching wait (≤ 1 s) is invisible next to it.

p90 clusters near 19 s in the light runs. That is the empty slots again: a
transaction arriving at the start of a 20 s gap waits nearly 20 s, and about a
third of the gaps are 20 s long.

### Derivation is cheap, and op-geth keeps up

| Transactions in the block | Build time |
|---|---:|
| ~60 (native) | 18–26 ms |
| ~300 (batch, 25/s) | 162–321 ms |
| ~1,300 (batch, 100/s) | 729 ms mean, 1,368 ms max |
| ~4,800 (batch, 400/s) | 1,814 ms mean, 5,425 ms max |
| ~10,200 (batch, 1,200/s) | 1,863 ms mean, **7,200 ms max** |

Derivation itself — reading the momentum, decoding messages, calling the Engine
API — is a few milliseconds. Everything above is op-geth executing the block,
and even the 13,514-transaction block fit inside the momentum that produced it.

### Per-transaction cost on NoM, by workload

| Workload | Gas | NoM B/tx | Plasma/tx |
|---|---:|---:|---:|
| ZNN transfer (batch, 25/s) | 21,000 | 88 | 9,834 |
| ERC-20 transfer | 36,056 | 97 | 10,464 |
| ERC-1155 mint | 31,135 | 104 | 10,581 |
| ERC-721 mint | 56,912 | 107 | 10,630 |
| Uniswap V2 swap | 91,682 | 122 | 12,260 |
| 20 cold storage slots | 475,555 | 106 | 15,197 |
| Contract deployment | 616,711 | **1,129** | **88,732** |

Calldata size drives the NoM cost and gas drives the L2 cost, and they are
almost independent: a storage-churn call costs 22× the gas of a transfer and
about the same number of bytes. Deployment is the outlier on both axes — it is
the only workload whose payload is large, and at 1,129 bytes and 88,732 plasma
it is roughly 9× a transfer to publish.

zlib does real work here: the relayer's posted ÷ raw ratio ranged from **0.34**
(Uniswap swaps, whose calldata is highly repetitive across wallets) to 0.90 (a
thin stream of transfers with little to compress against).

### What would have to change to go faster

- **To raise the native ceiling:** nothing in the ZVM. It is
  `MaxAccountBlocksInMomentum`, and that is consensus.
- **To raise the batch ceiling:** make the relayer land a batch every momentum
  instead of every two or three — batches that fit one account block, or chunk
  sets whose assembles arrive on a steady beat. The 300 M gas limit is then the
  next wall, at about 14,000 transfers a momentum.
- **To recover the third lost to empty slots:** run the L1 with more than one
  pillar, or find out why a lone pillar is not scheduled into every slot.
  Mainnet is not in this position.
- **To lower latency:** impossible below one momentum without a sequencer, which
  is the thing this design deliberately does not have.

### Caveats specific to these numbers

Everything ran on one host, so the load generator, the relayer, op-geth and both
Zenon nodes competed for the same 16 cores; at the ceiling op-geth used about 4
of them, and the load generator itself ran out of sockets. `chaindata grew 144,332 KB` in the ERC-721
run is a LevelDB compaction that happened to fall inside the window, not the
cost of 3,000 mints — the neighbouring runs grew 48–280 KB.

One correction is worth recording rather than quietly fixing, because it
changes how the earlier tables should be read. The harness first attributed a
run's NoM cost to the blocks falling inside a *clock* window ending 30 s after
submission stopped. That is fine until a run saturates — and then inclusions
keep draining for another 90 s, so the transactions were counted while the
blocks carrying them were not, dividing the per-transaction footprint by too
large a number. The window is now the blocks that actually carried the run,
first to last, taken from the per-transaction records. The check that it is
right: a native transfer is a 32-byte message in its own account block, so it
must cost exactly `21,000 + 68 × 32 = 23,176` plasma — and that is now what
both native transfer runs report, at 5 tx/s and at saturation alike. Under the
clock window the saturated run had claimed 18 bytes and 12,882 plasma for a
message that is provably 32 bytes.

`zvm bench report` therefore recomputes every summary from the stored
per-transaction records rather than trusting the summary written at run time,
so correcting arithmetic costs a re-render rather than a re-run.

---

# Every service, at scale — the fleet benchmark

**Measured 2026-09-15** on the same stack, with `zvm bench fleet`: 22 phases
in 1 h 19 min, plus two re-runs of the application phases (see *Corrections*).
It covers **100,000 EVM wallets** and **1,000 Zenon accounts fused for the
occasion**, and **362,913 derived transactions** in 18,578 NoM account blocks. It exercised every service the stack runs:
both lanes, the marketplace, collections and farm, the bridge in both
directions, the faucet, the media store, contract verification, the explorer
API, and the second node. Tables are in [`bench/FLEET.md`](results/FLEET.md),
raw phase files in `bench/fleet/*.json`, per-transaction records in
`bench/fleet/runs/`.

The first benchmark asked how fast. This one asks **where each service stops,
and what every transaction leaves behind on disk** — on the L1 nodes, in
op-geth, in the indexer.

## Method

Each phase is wrapped in two snapshots and sampled every 5 s in between.

| What | How |
|---|---|
| Bytes on disk | `du -b` (apparent) and `du -k` (allocated) inside each container: znnd's `nom/`, `consensus/`, `log/`; op-geth's `chaindata/` and `ancient/`; the node's `/data` and `media/`; the follower's and bridge's `/data` |
| The index, table by table | `zvm bench storage`, run **inside** the node container against the live SQLite file, read-only: `dbstat` per table and per index, row counts, pages in use |
| Bytes written | `docker stats` block I/O per container (Docker rounds to 3 significant figures, so per-phase figures at GB scale are good to ~10 MB) |
| Chain totals | `/api/ops` derivation totals: momentums, NoM blocks, messages, data bytes, plasma, txs, drops |
| Lag and queues | L1 frontier vs derived momentum, L2 head vs indexed head vs the follower's head, relayer and bridge status |

The harness changes that made 100,000 wallets possible, all in `internal/bench`:
- **Wallets are built and funded in parallel.** The first harness did one wallet at a time, sent 50 ZNN each, and could not fund more than ~3,000 from the deployer.
- **Load comes from a worker pool over a queue of idle wallets**, not one goroutine per wallet.
- **Every RPC client uses a pooled transport.** Go's default two idle connections per host is what exhausted ephemeral ports at 1,200 tx/s last time.
- **The `transfer-new` workload** pays an address that has never existed, the worst case for state growth.

```sh
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o bin/zvm-linux ./cmd/zvm   # the in-container probe
go build -o bin/zvm ./cmd/zvm      # in zvm-core
zvm bench fleet -list                                   # the 22 phases
zvm bench fleet                                         # all of them
zvm bench fleet -phases app- -app-rate 60               # or some
zvm bench fleet-report -in bench/fleet -out bench/FLEET.md
```

## The boundaries, in one table

| Service | Where it stops | What stops it |
|---|---|---|
| Wallet count | **not reached at 100,000** | 1k, 10k and 100k wallets ran at the same ~200 tx/s and ~16 s p50 |
| Batch lane, incompressible transfers | **195 tx/s** | the relayer's in-flight window: 8 accounts × 8 unconfirmed blocks |
| Batch lane, application calls | **~1,200–1,250 calls a block** | derivation admits by *declared* gas limit, so 95k–156k-gas calls fill 300 M long before they use it |
| Native lane | **100 a momentum, 7.1 tx/s**, at 1,000 wallets as at 60 | NoM's 100 account blocks per momentum |
| One heavy sender on the batch lane | **~8 large transactions per confirmation** | the relayer pins a sender to one of its 8 accounts |
| Onboarding a Zenon account | **~100 fusions a momentum**, plasma usable ~3 momentums later | the same 100-block budget |
| Bridge deposits | **~50 a momentum** | two account blocks each: the user's send and the vault's receive |
| Bridge withdrawals | **~96 a momentum**, after ~100 s of finality and scanning | one vault block each, from one account |
| Faucet | not reached: 60 drips in 1.2 s | four L1 blocks per drip; the 500-unconfirmed and 100-per-momentum rules bite near ~25 drips a momentum |
| op-geth | builds of **12.1 s**, longer than a momentum | executing 11,102 account-creating transfers; derivation still never fell more than one momentum behind |
| Media reads from the node | **323 req/s** for a 256 KB object at 32 readers | the node serves it; this is what a CDN hostname in front is for |
| Explorer API | endpoints that aggregate slow **5–15×** as their data grows | NFT items and candles computed per request |

Every limit that is NoM's is a count, and every one of them is the **same**
100-account-block budget: the native lane, the relayer's batches, fusions,
bridge sends and receives, faucet drips. Saturating one starves the others.
While the native lane ran at its ceiling, three relayed transfers took
**87–138 s** to land instead of ~15 s.

## Wallets are not a limit; compressibility is

| Wallets | Offered | Included | Sustained | p50 | p90 | Max tx/block | Build max |
|---:|---:|---:|---:|---:|---:|---:|---:|
| 1,000 | 200/s | 23,988 / 23,988 | 222.8 | 17.5 s | 24.1 s | 3,754 | 2.7 s |
| 10,000 | 200/s | 23,982 / 23,982 | 185.7 | 15.5 s | 23.3 s | 3,858 | 3.2 s |
| 100,000 | 200/s | 99,460 / 99,460 | 200.5 | 16.3 s | 24.8 s | 4,339 | 4.6 s |
| 100,000 | **600/s** | 100,932 / 100,932 | **195.0** | 146.6 s | 302.4 s | **11,102** | **12.1 s** |

Funding the 100,000 took 250 disperse transactions and **7 min 22 s**.
Nothing failed at any wallet count. The relayer held 102,023 senders in 336 MB.

The first benchmark's ceiling was ~270 tx/s; this one's is 195. The difference
is what the transactions contain. There, 200 senders paid 512 fixed addresses,
and batches compressed to **62 bytes a transfer**. Here, every transfer came
from one of 100,000 senders and paid a brand-new address: **89 bytes**. The NoM
bytes the relayer can have in flight are fixed — 64 blocks of 16 KB — so fewer
transfers fit. At saturation the in-flight window was full for the entire run,
and the relayer landed one burst every fourth momentum: 61 NoM blocks, exactly
11,102 transfers, 233 M gas of a 300 M limit. **Gas never bound; the window did.**

That block took op-geth 12.1 s to build — longer than the 10 s momentum it came
from — and derivation still lagged by at most one momentum, because the empty
momentums between bursts gave it time back. A relayer that landed a burst every
momentum would find this wall next.

## Application calls are capped by the gas they declare

| Workload (2,000 wallets, 60/s) | Included | Sustained | p50 | Gas used | NoM B/tx | Max calls/block |
|---|---:|---:|---:|---:|---:|---:|
| Swap ZNN → token (Uniswap V2) | 7,190 | 62.3 | 12.9 s | 95,509 | 120 | 1,224 |
| NFT mint (clone's public drop) | 7,188 | 55.6 | 12.8 s | 81,738 | 105 | 1,184 |
| NFT listing | 7,194 | 61.1 | 12.9 s | 155,551 | 103 | 1,224 |
| NFT purchase | 6,000 of 6,000 listings | 50.4 | 15.3 s | 96,332 | 105 | 1,251 |
| Farm deposit / harvest | 7,175 | 55.3 | 15.1 s | 84,277 | 123 | 1,207 |
| Native NFT mint (1,000 Zenon accounts, 6/s) | 719 | 5.3 | 8.8 s | 105,947 | — | 96 |

Derivation admits a relayed transaction while `gasSoFar + gasLimit ≤ 300 M`
(`spec/SPEC.md` §6), counting the limit the wallet **declared**, not what it used. A
wallet declares an estimate plus a margin — these phases used MetaMask's 30 % —
so 1,200-odd calls fill a block that has used under half its gas. At 60 offered
per second the lanes kept up. **At 100 per second the first run did not, and how
it failed is the most important thing this benchmark found:**

1. A block filled; derivation dropped the overflow as `block full` (4,900 drops).
2. The relayer requeued them, but each wallet's *later* nonces were already in
   other batches, which then landed first and were dropped as `nonce too high`.
3. Those were requeued too. 17,117 drops in the swap phase alone; swaps
   sustained **57 tx/s of 100 offered**, and 474 were never included.
4. **The relayer never gives up on a gapped transaction.** It requeues any
   dropped transaction whose nonce is unused, and a gapped transaction's nonce
   is always unused. After load stopped, it kept re-posting the same 335
   transactions every momentum — 8,040 posts and 8,040 drops in 335 idle
   seconds, **4 NoM blocks, 641 KB and 45.6 M plasma every 24 momentums, about
   160 MB of ledger a day, indefinitely**. Restarting the relayer cleared it
   (its state is in memory). See *Defects*, below.

## What a transaction leaves behind

Bytes per transaction, apparent size, from the 100,000-wallet phase (99,500
transfers to new addresses) unless noted:

| Where | Bytes per transfer | Notes |
|---|---:|---|
| NoM, as data | **89** | the relayer's zlib output |
| znnd ledger (`nom/`), each L1 node | **~190–220** | ~2.3× the data: account-block headers, hashes, indexes |
| znnd `log/` | 7–16 | |
| op-geth `ancient/` (bodies, receipts) | **97** | |
| op-geth `chaindata/`, as the running node reports it | 364 | mostly pebble's write-ahead logs, see below |
| **Explorer index** (node's SQLite) | **1,121** | `txs` 406, `addr_txs` 373, `tx_origin` 251, `messages` 91 |
| The follower's store, no index | **342** | what a node that only derives keeps |
| Written by the node container | **~28 KB** | SQLite WAL and checkpoints: ~25× what it keeps |
| Written by each znnd | ~5.7 KB | |
| Written by op-geth | ~0.7 KB | before its state is flushed |

The **indexer is the largest store per transaction** — five times the L1
ledger, three times the derivation store, eleven times op-geth's block data —
and writes the most by far. An explorer, not the chain, is what fills a disk.

Application calls cost the index more, because their logs are indexed too
(bytes per call, excluding `drops` rows the relayer defect inflated):

| Workload | Index B/call | Largest parts |
|---|---:|---|
| Swap | **5,224** | `logs` 2,728, `token_transfers` 940, `dex_events` 423 |
| NFT purchase | 3,784 | `logs` 1,524, `nft_sales` 531, `token_transfers` 509 |
| Farm | 3,449 | `logs` 1,520, `token_transfers` 603 |
| NFT listing | 2,958 | `logs` 976, `nft_listings` 275 |
| NFT mint | 2,596 | `logs` 632, `token_transfers` 502, `nft_tokens` 222 |
| Contract verification | **~510,000** | the 243 KB standard JSON input, stored in `verify_jobs` *and* `verified_contracts` |
| Media object | its own size | content-addressed files; the table row is negligible |

**op-geth's reported size is mostly write-ahead log.** The primary's `chaindata/`
read 550 MB apparent and **1.49 GB allocated** at the end. The follower — same
chain, same agreed state root at block 4,139 — was restarted gracefully and
then read **163 MB apparent, 163 MB allocated**, 85 MB of it ancient blocks.
The chain op-geth actually persists is a third of what a running node shows
and a ninth of what it allocates. Its restart took 2.9 s and it was caught up
5.2 s later.

Across the whole run (~303,600 derived transactions): each znnd's `nom/` grew
81 MB, the index 454 MB (465 → 919 MB), the follower's store 125 MB, media
256 MB. The node container wrote 9.7 GB; each znnd 3.1 GB; op-geth 0.4 GB.

## The native lane and onboarding

**990 Zenon accounts were fused, 50 QSR each, from two funders.** Every publish was
accepted — no plasma or unconfirmed-block refusals — in 158 s. The plasma became
usable at **~100 accounts a momentum**, and for all of them after 280 s. Unqueued,
a fusion is usable about three momentums (30 s in the smoke run) after it is
published, not at once.

With those 1,000 accounts the native lane carried **100 transactions a momentum,
7.1 tx/s** — the same ceiling 60 wallets reached in the first benchmark. It is
NoM's count, not anything per wallet.

A Zenon account with 50 QSR fused has **105,000 plasma**, about four native calls
in flight. The smoke run drove ten such accounts at 0.6 calls a second each and
was refused `not enough plasma on account` 53 times. One Syrius wallet cannot
fire faster than its fusion allows, whatever NoM's capacity.

## The bridge

| | Deposits (1,000 from 200 accounts) | Withdrawals (1,000 from 1,000 wallets) |
|---|---|---|
| Published | 1,000 in 2.8 s | 1,000 in 5.5 s, included in 2 blocks |
| Credited / paid | all, over 14 blocks: p50 **64 s**, max 264 s | all, p50 **184 s** end to end, max 246 s |
| NoM blocks | 2,000: each send plus the vault's receive | 1,000 payouts from the vault |
| Vault | height 33 → 1,033; nothing left unreceived | height 1,033 → 2,033; no refusals |

Withdrawals pay nothing for the first ~100 s — six blocks of finality and the
bridge's scan — and then ~96 a momentum.

## The faucet, media, verification

**Faucet.** 60 drips in 1.2 s, ~150 ms each, all credited on the ZVM within three
blocks. Each drip is four L1 blocks from one account. **The cooldown can be
walked around**: it is keyed on the first `X-Forwarded-For` hop, which the
explorer's proxy passes through from the client. A new address with a forged
header was paid immediately after the same client had been refused (probes in
`bench/fleet/16-faucet.json`).

**Media.** 100 signed sessions (p50 6 ms). 900 uploads, 255 MB, in 3.7 s: 9 ms
for 2 KB of JSON, 64 ms at 256 KB, 386 ms at 2 MB, ~1 s at 9.9 MB. Disk grew
by exactly the bytes uploaded. One byte over 10 MB, one byte over 256 KB of JSON,
and a non-image were refused; a duplicate was accepted again and, content-addressed, kept as one file. Reading
one 256 KB object back at 32 concurrent readers: **323 req/s, p50 96 ms**.

**Verification.** Eight BenchERC20 deployments submitted at once all verified,
p50 2.6 s, max 5.2 s: the compiler runs roughly one job at a time.

## The explorer API

Each endpoint was read at 1, 16 and 64 concurrent readers for 4 s, before the
benchmark and after. Everything answered 200. Selected, requests a second:

| Endpoint | 1 reader, before → after | 64 readers, before → after | p99 at 64, after |
|---|---:|---:|---:|
| `/api/blocks`, `/api/txs`, `/api/momentums` | 415–514 → 357–464 | 2,808–4,211 → 3,087–4,076 | 40–53 ms |
| `/api/search`, prices, verified marks | 730–810 → 505–754 | 11,300–13,900 → 10,300–12,200 | 13–17 ms |
| `/rpc` `eth_blockNumber` (proxy to op-geth) | 446 → 415 | 2,562 → 2,802 | 158 ms |
| `/api/tokens/{BNA}/holders` | 54 → 37 | 281 → 215 | 809 ms |
| **`/api/nft/collections/{c}/items`** | **234 → 16** | **1,507 → 46** | **3.3 s** |
| **`/api/dex/pairs/{p}/candles`** | **126 → 17** | **830 → 83** | 1.8 s |
| `/api/nft/collections` | 122 → 21 | 478 → 42 | 2.7 s |
| **`/api/info`** | **45 → 15** | **284 → 33** | 1.9 s |

Lists by block and transaction do not care how much history there is. The slow
four do: items and collections aggregate a collection that grew by ~14,000
tokens, candles are built per request from the pair's events, and `/api/info`
asks the relayer for its status — which **walks all 102,023 senders under the
relayer's single lock** (68–168 ms per call), pausing admission while it does.

**A busy explorer did not slow the chain.** 32 readers on the eleven heaviest
endpoints got 122 req/s (p50 209 ms, p99 1.1 s, no errors). The batch lane
beside them sustained 254 tx/s, derivation lagged at most one momentum, the
index at most two blocks. The node peaked at 6 of 16 cores.

## Resources at the ceiling

Peaks during the 600 tx/s saturation run: op-geth **7.2 cores, 2.2 GB**; its
follower 4.6 cores, 1.8 GB; the node 2.7 cores, 493 MB; the relayer 2.7 cores,
336 MB; the L1 RPC node 0.7 cores; the pillar 0.16 cores. NoM's nodes were the
least loaded machines in the stack throughout.

## Defects this benchmark found

In order of consequence, with what was done about each. **All six are now
addressed**; the numbers above this line predate the fixes and were not re-run,
so treat them as the measurement of the code that had the defects.

Three more were found on 2026-09-16 and fixed the same day, all three in the
*measuring apparatus* rather than the chain — which is worth its own note,
because each of them made the chain look worse than it is:

- **The relayer's footprint was denominated in blocks on both rulesets.** On v2
  the resource is base plasma, so costing every block as a full 16 KB one left
  chain 71 taking 3 blocks a momentum where its share paid for about 23. Any
  v1-vs-v2 comparison older than this date carries a 30× handicap on the v2 side.
- **A fresh chain's relayer posted nothing.** The budget started at zero and
  `Take(0)` read as "this momentum, already spent", so for the first 30
  momentums — the observation window — nothing was published at all and
  `zvm bench setup` timed out.
- **`bench suite` paused a fixed 20 seconds between steps.** That is not enough
  after a saturating step: the relayer can be holding tens of thousands of
  transactions and, on v2, the fusion price can be at 20×. The step after
  chain 70's 400-offered run measured 15.85 tx/s against an offered 25, and
  chain 71 re-run the same step cleanly at 24.99. It now waits for the queue to
  drain and the price to fall back under 1.2×.

1. **Relayer livelock on nonce gaps** — *fixed* (`requeuePlan`,
   `internal/relayer/relayer.go`). A dropped transaction was requeued whenever
   its nonce was unused, which a gapped one always is, and the sender's cursor
   was wound back to *that transaction's* nonce — so it was posted again every
   second, forever, at about 160 MB of NoM ledger a day. The cursor now goes
   back to the **account's** nonce and no further, which leaves a gapped entry
   in the queue but out of reach until the gap in front of it is filled. A
   transaction whose nonce something else has used, or that has been retried
   more than `-max-requeues` times, is discarded; one that waits longer than
   `-queue-ttl` is swept. The `discarded` counter now moves.
2. **Overload cascade** — *mitigated* (`-max-batch-gas`). Dropping one
   transaction as `block full` orphans the sender's later nonces already posted
   elsewhere. The relayer now stops filling a batch at the gas a block will
   actually admit from the batch lane — the elastic rule's ceiling once the gas
   fork is scheduled, the whole block before it — so it no longer posts
   transactions that are certain to be dropped. The cascade is bounded, not
   impossible: a block can still fill for reasons one relayer cannot see.
3. **The faucet's cooldown trusted `X-Forwarded-For`** — *fixed*
   (`clientKey`, `cmd/zvm/faucet.go`). A client writes that header itself and a
   proxy only appends to it, so the first hop was the caller's own claim and a
   new value in each request walked around the cooldown. The peer address is now
   the key, and the header is read only when the peer is a configured
   `-trust-proxies` address — and then only the hop that proxy appended.
4. **Relayer `Snapshot` was O(senders) under the global lock** — *fixed*. It
   walked all 102,023 senders per call (68–168 ms), pausing admission while it
   did, which is what made `/api/info` slow. The counts are now maintained as
   senders come and go.
5. **One sender pinned to one relayer account** — *partly fixed*. The account
   was chosen by hashing the sender's address, so an unlucky spread left
   accounts idle while one saturated; it is now the least-loaded account, and a
   sender with nothing in flight is re-placed on its next submission. **A single
   sender is still serialised on one account, deliberately**: NoM orders blocks
   by address within a momentum, so two of one sender's nonces posted from two
   accounts can arrive in the wrong order and the later one is dropped as
   "nonce too high". One account per sender is what makes the relayer's order
   the order derivation sees.
6. **Verification stored its input twice** — *fixed* (D33), ~510 KB per
   verified contract, now content-addressed in `verify_inputs`.

### And what the index did about its own numbers

The 1,121 B per transfer above included **373 B of `addr_txs`**, a row per
address per transaction that answered two questions an index on `txs` and one
counter answer better. Measured on a copy of this devnet's own 1.13 GB index
(684,348 transactions): `addr_txs` was **175.0 MB**, replaced by **15.2 MB** of
`addr_stats` counters plus indexes on `txs`. The migration is a one-off 32 s on
open, and the four endpoints that had slowed 5–15× answer from precomputed rows
— candles and collection statistics are now folded in as the index is written
(D21).

## Corrections — mistakes in the benchmark itself

- The first application run reused one wallet range across five phases. The
  stuck swaps then collided with the next phase's nonces
  (`replacement transaction underpriced`, `nonce too low`), so listing, buying
  and farming never ran. Each phase now has its own range. That run is kept in
  `bench/fleet/superseded/`: it is also the evidence for defects 1 and 2.
- The re-run then set gas limits at 1.3× an *average* from the smoke test. A
  wallet's first swap or mint writes fresh storage and needs more (143,812 and
  108,435 by `eth_estimateGas`). **All 7,191 swaps and 7,199 mints reverted out of
  gas**, and each retry was a first call again. Application workloads now
  estimate every call and add 30 %, as a wallet does; that run is also in
  `superseded/`, marked `out-of-gas`.
- The relayer livelock ran from the first application run until the relayer
  was restarted at the end. Every phase after 05 still in the results, both application
  re-runs included, therefore carries ~4 NoM blocks a momentum of its traffic. It inflates
  the NoM bytes and plasma per transaction of the smaller runs — the native
  transfer phase reads 123 B and 29,774 plasma for what is provably a 32-byte,
  23,176-plasma message — and it adds `drops` rows to the index. The native
  figures above use the identity, and the index figures exclude `drops`.
  Phases 00–05 (every wallet-scaling and saturation number) predate it.

## What these numbers are not

Everything in *What these numbers are not*, above, still applies: one host, one
pillar, co-located load generator, not mainnet. Additionally: op-geth's
per-phase chaindata growth is measured before its state is flushed (the
follower's restart is the flushed figure); Docker's block-I/O counters are
coarse at GB scale; and the per-transaction storage figures are for this
schema and these contracts — the index is a design choice, and the largest one.

---

# Re-measured on op-reth

**2026-09-16.** The chains moved from op-geth v1.101702.2 to **op-reth 1.10.2**
(D2). Everything above this heading is op-geth's; this section is the same
harness against the new client. The L2 was wiped and re-derived from NoM on both
chains, and the NoM ledgers were kept, so the contracts and their addresses are
the ones the earlier runs used.

## What changed, and what deliberately did not

Only the execution client. Both chains stayed on **Isthmus** rather than moving
to Karst at the same time, so that a difference in these numbers has one cause
rather than two. The genesis did change — the `L2ToL1MessagePasser` predeploy
had to be added before the two clients would agree on a genesis hash at all
(D2) — but that is a state-root change, not a rule change.

One config value moved with the regenerated genesis: chain 71's `gasPerSupply`
went from the inherited 1.37e14 to the **measured 1.15e14**, because this is a
new genesis rather than a chain continuing with a value it had already written.
It prices native calls on v2 and does not touch the batch lane.

## The storage engine is the visible difference

op-geth keeps everything in pebble under `chaindata`, with frozen blocks in
`ancient/`. op-reth keeps live state in an MDBX database under `db/` and history
in `static_files/`. **MDBX is one sparse file mapped at a fixed 4 GiB**, which
is a trap for any measurement taken with `du -b`:

Measured on chain 70 after it had re-derived its whole history (head 7,004):

| | apparent (`du -b`) | allocated (`du -sk`) |
|---|---:|---:|
| `op-reth /data/db` | 4,295,999,501 | **283 MB** |
| `op-reth /data/static_files` | 119,468,032 | **116 MB** |

The apparent size of `db` reads 4,295,999,501 bytes on an empty chain and never
moves, so a per-phase delta taken from it is always zero. `internal/fleet` now
reads **allocated** bytes for reth and apparent bytes for geth, which is the
measure that matches how each client actually stores; this document's older
per-phase storage columns are op-geth's apparent bytes and are not comparable to
reth's rows.

**`docker system df -v` is fooled by the same file**, which matters to anyone
sizing a disk from it:

| volume | docker reports | actually allocated |
|---|---:|---:|
| `zvm_op-reth-data` (chain 70) | 4.418 GB | ~400 MB |
| `zvm_op-geth-data` (same chain, before the swap) | 779.6 MB | not measured |

So op-reth looks like it needs ten times the disk it does. The honest comparison
for this chain is op-reth's **~400 MB allocated** against op-geth's 779.6 MB
apparent volume — and op-geth's *allocated* figure was historically the larger
of its two (the fleet benchmark measured 550 MB apparent against 1.49 GB
allocated, because pebble preallocates recycled write-ahead logs). The two
clients therefore mislead in opposite directions, and only allocated bytes
compare.

## Chain 71 — the first run, superseded

> **These two tables are kept as the evidence for a defect, not as a measurement
> of the chain.** They were taken before the relayer re-bid fix, and every
> number in them is bounded by it. The measurement that stands is
> [Chain 71, re-measured after the fix](#chain-71-re-measured-after-the-fix).

### The batch lane measures a relayer defect

**These numbers are not op-reth's ceiling.** The v2 batch lane is bounded by the
relayer wedge — under dynamic plasma a posted block
that falls behind the price is never re-bid, its transactions are eventually
discarded, and each discard strands the rest of that sender's nonces
permanently. Posting stops and does not resume without a restart.

| offered | measured | p50 | p90 | gas/tx | NoM B/tx | max tx/block |
|---:|---:|---:|---:|---:|---:|---:|
| transfer, 5 tx/s, 20 wallets | 4.99 | 16.6 s | 29.5 s | 21,000 | 84 | 102 |
| transfer, 25 tx/s, 50 wallets | 21.59 | 15.3 s | 30.2 s | 21,000 | 76 | 404 |
| transfer, 100 tx/s, 100 wallets | 36.47 | 32.0 s | 174.5 s | 21,000 | 73 | 776 |
| transfer, 400 tx/s, 200 wallets | **0.00** | — | — | — | — | 0 |

The last row is the wedge, not a capacity limit: `posted` and `included` were
byte-identical to the previous step's totals, and 13,171 submissions were
rejected outright. Above roughly 25 offered tx/s a v2 batch run measures this
defect and nothing else, so the remaining v2 scenarios below were each run with
the relayer restarted first.

### Chain 71, the rest of the matrix

Each of these was run with the relayer **restarted first**, so the wedge from a
previous step could not carry into the next. That is not how the chain runs in
production; it is what it took to measure anything above the wedge.

| Scenario | Offered | Measured | p50 | p90 | gas/tx | NoM B/tx | max tx/block |
|---|---:|---:|---:|---:|---:|---:|---:|
| erc20, 50 wallets | 25 | 24.26 | 26.4 s | 86.0 s | 34,248 | 76 | 730 |
| erc721, 50 wallets | 25 | 23.94 | 29.6 s | 123.6 s | 56,925 | 77 | 744 |
| erc1155, 50 wallets | 25 | 22.76 | 24.6 s | 92.1 s | 31,163 | 78 | 729 |
| swap, 40 wallets | 20 | 19.99 | 23.5 s | 59.2 s | 91,654 | 83 | 669 |
| churn, 25 wallets | 10 | 9.99 | 28.8 s | 75.4 s | 475,555 | 76 | 366 |
| deploy, 10 wallets | 3 | 2.99 | 26.7 s | 67.8 s | 616,711 | 345 | 119 |
| erc20, 150 wallets | 250 | 44.56 | 67.0 s | 255.0 s | 34,248 | 68 | 774 |

At and below 25 offered tx/s every scenario delivers what it is asked for. The
250 tx/s row is the same wedge arriving more slowly from a clean start: 44.56
sustained, p90 four minutes.

**The gas numbers are a cross-check, but read the column carefully.** `gas/tx`
is a **mean over the run**, not a per-transaction constant, so it moves with the
mix of cold and warm storage a run happens to hit — a wallet's first mint or
first transfer to a fresh recipient writes a zero slot and costs far more than
its second. That is why these do not line up exactly with op-geth's: a Uniswap
V2 swap reads 91,654 here against 91,682 there, an ERC-721 mint 56,925 against
56,912, and an ERC-20 transfer 34,248 against 36,056 — the last one a 1,808-gas
gap that is a different cold/warm mix, not a different EVM.

The clean comparison is the one workload with no storage variance at all: a
**bare transfer is exactly 21,000 on both clients**, in every run. That is the
claim worth making — swapping the execution client changed what a transaction
costs by nothing. The differences in this document are about publishing,
storage and the relayer, never about the EVM.

The three native-lane scenarios were not completed on chain 71: the run sat in
account onboarding — funding and fusing plasma for new Zenon accounts — for
forty minutes without producing a transaction, and was stopped. The native lane
is covered by the chain 70 matrix below instead, and why onboarding is that slow
on a dynamic-plasma chain is worth a look on its own.

## Chain 70 (mainnet's rules): op-reth against op-geth, step for step

The fourteen-step matrix plus the 1,200 tx/s ceiling run, on the same chain, the
same contracts and the same harness as the 2026-09-15 op-geth matrix. Full table
in [`bench/RESULTS-reth-v1.md`](results/RESULTS-reth-v1.md). **Sustained** is
the column to read — `tx/s` overstates capacity once the chain stops keeping up,
because inclusion keeps draining after submission stops.

| Scenario | Offered | op-geth tx/s (sust.) | op-reth tx/s (sust.) |
|---|---:|---:|---:|
| transfer, 20 w | 5/s | 5.0 (4.7) | 5.0 (4.5) |
| transfer, 50 w | 25/s | 25.0 (22.7) | 25.0 (23.3) |
| transfer, 100 w | 100/s | 99.9 (92.6) | 100.0 (**102.1**) |
| transfer, 200 w | 400/s | 397.3 (261.1) | 399.6 (**375.7**) |
| **transfer, 200 w** | **1200/s** | 677.5 (**271.1**) | 589.0 (**277.8**) |
| erc20, 50 w | 25/s | 25.0 (23.2) | 25.0 (23.3) |
| erc20, 150 w | 250/s | 249.3 (185.5) | 224.4 (**206.7**) |
| erc721, 50 w | 25/s | 25.0 (23.2) | 21.0 (19.5) → **25.0 re-run** |
| erc1155, 50 w | 25/s | 25.0 (25.9) | 21.0 (19.6) |
| swap, 40 w | 20/s | 20.0 (20.3) | 20.0 (20.3) |
| churn, 25 w | 10/s | 10.0 (9.3) | 8.4 (7.2) |
| deploy, 10 w | 3/s | 3.0 (2.8) | 3.0 (2.6) |
| transfer native, 20 w | 5/s | 5.0 (4.2) | 5.0 (4.6) |
| transfer native, 60 w | 15/s | 15.0 (7.1) | 15.0 (6.8) |
| erc20 native, 40 w | 10/s | 10.0 (7.4) | 10.0 (6.9) |

### The ceiling did not move, and that is the point

Pushed to 1,200 offered, where op-geth found its ceiling, the two clients agree:
**271.1 sustained on op-geth, 277.8 on op-reth** — a 2.5 % difference, inside the
±15 % this measurement is good for. The biggest block went from 13,514
transactions to 13,060.

That is the architecturally correct answer rather than a disappointing one. **A
based rollup is bounded by its L1's data capacity, not by its EVM.** What limits
this chain is how many bytes fit in a momentum and how often momentums arrive;
execution was never the constraint, so replacing the execution client does not
move the ceiling. The place the swap *does* show is storage, where op-reth holds
the same chain in about half the allocated bytes.

Where reth is visibly better is **keeping up before the ceiling**: at 400
offered it sustained 375.7 against op-geth's 261.1, and at 100 offered it
sustained the full offered rate. op-geth reached a higher raw `tx/s` at 1,200
(677.5 against 589.0) while sustaining no more, which is the draining artefact
the metric exists to expose. op-geth did push more gas per second at the ceiling
— 14.2 M against 12.4 M — which is worth revisiting on a workload heavier than
transfers.

### A harness fault the matrix exposed, and the control that isolated it

Three steps under-delivered — erc721 and erc1155 at exactly **2,520 of 2,999**,
churn at 1,007 of 1,200 — and the swap step failed outright with `preparing
wallets: replacement transaction underpriced`. None of it is the execution
client:

- the per-transaction records carry **239** `replacement transaction
  underpriced` errors on erc721 and **173** on erc1155, and **zero** on the
  erc20 step that delivered in full. These are *submission* failures: the load
  generator reused a nonce.
- the load generator takes wallet nonces from `eth_getTransactionCount`, which
  lags while the relayer is holding a backlog from earlier steps.
- **the control settles it.** erc721 re-run alone, against a relayer restarted
  first, delivered **24.99 tx/s in full** — same client, same chain, same step,
  the only variable being relayer backlog. The swap step likewise passed on a
  re-run at 19.99 of 20.

So those rows understate op-reth, and the honest comparison for them is the
re-run. It is also a real finding about the harness rather than the chain: **a
`zvm bench suite` run is not independent step to step**, because one step's
backlog corrupts the next step's wallet preparation. Steps should restart the
relayer between them, or take nonces from `pending` rather than `latest`.

## Chain 71, re-measured after the fix

The eleven batch-lane steps again, with the relayer re-bid timer and the nonce
fix in, and the relayer restarted between steps (the control above showed a
suite run is not independent step to step). Full table in
[`bench/RESULTS-reth-v2.md`](results/RESULTS-reth-v2.md), raw records in
`bench/results-reth-v2-final/`.

**Every step recorded `discarded 0` and `rejected 0`.** That is the headline: the
losses that dominated the first run are gone, not reduced.

| Scenario | Offered | Included | tx/s (sust.) | before the fix |
|---|---:|---:|---:|---:|
| transfer, 20 w | 5/s | 599 / 599 | 4.7 (4.5) | 4.99 |
| transfer, 50 w | 25/s | **2,999 / 2,999** | 24.5 (21.3) | 21.59 |
| transfer, 100 w | 100/s | 5,013 | 41.8 (18.4) | 36.47 |
| **transfer, 200 w** | **400/s** | **4,681** | **39.0 (16.6)** | **0.00** |
| erc20, 50 w | 25/s | **2,999 / 2,999** | 25.0 (19.4) | 24.26 |
| erc20, 150 w | 250/s | 5,615 | 46.8 (18.2) | 44.56 |
| erc721, 50 w | 25/s | **2,999 / 2,999** | 25.0 (20.8) | 23.94 |
| erc1155, 50 w | 25/s | **2,999 / 2,999** | 25.0 (20.3) | 22.76 |
| swap, 40 w | 20/s | **2,399 / 2,399** | 20.0 (16.7) | 19.99 |
| churn, 25 w | 10/s | **1,199 / 1,199** | 10.0 (8.5) | 9.99 |
| deploy, 10 w | 3/s | **359 / 359** | 3.0 (2.5) | 2.99 |

### What chain 71 can actually carry

**At or below 25 offered tx/s every scenario delivers in full** — 2,999 of 2,999,
every time, across five different workloads. Above that the chain does not
collapse and does not lose anything; the queue simply grows, which is what the
latencies show (p90 rising from 60 s at 25 offered to 256 s at 400). Sustained
inclusion tops out around **21 tx/s**.

That is close to the 25.4 tx/s measured on this ruleset before the client
changed, so **dynamic plasma costs roughly an order of magnitude against v1's
~271–278 sustained** — and that gap is the ruleset, not op-reth, which the v1
matrix on the same client and same host establishes.

The re-bid counter is the direct evidence the fix is doing continuous work
rather than having got lucky once: it fired **1 to 7 times per step**, on a
relayer restarted before each one, so the plasma price is genuinely moving under
load and blocks would be stranding without it.

### The native lane on chain 71: half explained

The native scenarios had not completed on chain 71 at all, and the obvious guess
was that dynamic plasma broke native onboarding. **That guess was half right.**

A native run still funds its wallets' *ZVM* balances through the relayer — the
lane is native on the way in, not on the way out — so the first native step had
been stalling inside the relayer wedge, during wallet preparation, before a
single native transaction was published. With the wedge fixed it runs:

| Scenario | Offered | Included | tx/s | p50 | NoM B/tx | max tx/block |
|---|---:|---:|---:|---:|---:|---:|
| transfer, native, 20 w | 5/s | **599 / 599** | 4.99 | 8.0 s | 32 | 100 |

That is full delivery, and identical to chain 70's native figure — the same 32
bytes per transaction and the same 100-per-momentum ceiling, because the native
lane's limit is NoM's account-block count and nothing about dynamic plasma
changes it at that rate.

The steps above 5 offered tx/s had also stalled, and the cause turned out to be
neither the lane nor the harness: **`znn.Account.publish` deadlocked against its
own mutex**. Its auto-rebid retry called `publish` again on the same goroutine,
and `sync.Mutex` is not reentrant, so the account blocked permanently — with no
context deadline able to break it. The branch needs the plasma price to have
risen since that account last published, so it could only ever fire on a
dynamic-plasma chain under enough load to move the price, which is why chain 70
never saw it. Fixed, and the same runs now complete:

| wallets | offered | before | after |
|---:|---:|---|---|
| 20 | 5/s | 40 s | — |
| 60 | 5/s | 90 s | — |
| 20 | 15/s | **hung** | **14.75 tx/s** (p50 8.2 s, max 181/block) |
| 60 | 15/s | **hung** | **14.82 tx/s** (p50 8.0 s, max 177/block) |

### Dynamic plasma *helps* the native lane — **superseded, see the correction at the end**

> This section was written from two isolated runs and its throughput claim does
> not survive the full matrix. The block-count arithmetic below is still right;
> the "seven times quicker" conclusion is not. See *Correction: dynamic plasma
> does not raise the native lane's throughput* at the end of this document.

This is the one place v2 beats v1, and it follows from how each ruleset counts.
A native send is one account block. **v1 caps a momentum at 100 account blocks
whatever they cost**; v2 caps on base plasma instead, so the ceiling is
`maxBasePlasmaInMomentum` 4,200,000 ÷ ~23,176 per send ≈ **181 per momentum** —
and the runs above measured 177 and 181, which is the arithmetic landing exactly.

The effect on latency is large. At 15 offered tx/s the native lane delivers
~14.8 on either chain, but includes at **p50 8.0 s on chain 71 against 59.4 s on
chain 70**: about seven times quicker, because on v1 the offered rate is over the
100-block ceiling and the queue never drains, while on v2 it fits under 181.

So the two lanes pull in opposite directions under dynamic plasma — the batch
lane loses an order of magnitude, the native lane nearly doubles its ceiling —
and which ruleset is better for a chain depends on which lane it expects to
carry.

---

# Both networks, after every fix (2026-09-16, final)

The fifteen-step matrix on **both** chains, on op-reth, after the four fixes of
this session: the relayer re-bid timer, the batch-height tracking, the harness
nonce source, and — the one that mattered most — the `Account.publish`
self-deadlock. The relayer is restarted before each step, because a step's
backlog otherwise corrupts the next step's wallet preparation.

Tables: [`bench/RESULTS-v1-fixed.md`](results/RESULTS-v1-fixed.md) and
[`bench/RESULTS-v2-fixed.md`](results/RESULTS-v2-fixed.md); records in
`bench/results-v1-fixed/` and `bench/results-v2-fixed/`. **Everything above this
heading predates at least one of those fixes.**

| Scenario | Offered | v1 incl. | v1 sust. | v1 p50 | v2 incl. | v2 sust. | v2 p50 |
|---|---:|---:|---:|---:|---:|---:|---:|
| transfer, 20 w | 5/s | 599 | 4.5 | 9.6 s | 599 | 4.0 | 27.8 s |
| transfer, 50 w | 25/s | 2,999 | 23.3 | 12.3 s | 2,999 | 19.9 | 24.7 s |
| transfer, 100 w | 100/s | 11,999 | 107.7 | 15.5 s | 8,868 | 32.6 | 56.4 s |
| transfer, 200 w | 400/s | 47,941 | 279.5 | 21.2 s | 5,475 | 18.6 | 125.3 s |
| erc20, 50 w | 25/s | 2,999 | 23.2 | 12.7 s | 2,999 | 18.1 | 28.6 s |
| erc20, 150 w | 250/s | 27,448 | 211.3 | 17.0 s | 6,168 | 20.1 | 128.1 s |
| erc721, 50 w | 25/s | 2,999 | 24.7 | 10.4 s | 2,999 | 12.4 | 93.5 s |
| erc1155, 50 w | 25/s | 2,999 | 27.1 | 11.7 s | 2,999 | 13.0 | 99.5 s |
| swap, 40 w | 20/s | 2,399 | 20.3 | 12.6 s | 2,399 | 17.0 | 29.4 s |
| churn, 25 w | 10/s | 1,199 | 9.8 | 9.3 s | 1,199 | 7.9 | 26.7 s |
| deploy, 10 w | 3/s | 359 | 3.0 | 9.8 s | 359 | 3.0 | 22.6 s |
| transfer native, 20 w | 5/s | 599 | 4.9 | 8.0 s | 597 | 4.8 | 8.1 s |
| transfer native, 60 w | 15/s | 1,799 | 6.6 | 58.6 s | 1,042 | 4.5 | 8.1 s |
| erc20 native, 40 w | 10/s | 1,199 | 6.9 | 31.0 s | 996 | 5.3 | 8.2 s |
| transfer, 200 w | 1200/s | 50,614 | 189.6 | 40.3 s | 6,245 | 21.8 | 120.7 s |

## What this says

**Chain 70 delivers every offered transaction, in every workload.** All fifteen
steps included their full count — 2,999 of 2,999 on each 25 tx/s workload, 599,
2,399, 1,199, 359. The losses that marked the earlier matrices are gone, and the
relayer finished the whole run at `discarded 0, rejected 0`.

**Chain 71 delivers in full up to 25 offered tx/s** and is capacity-bound above
it — 73.9 at 100 offered, ~50 at 250 and beyond. It does not collapse and it does
not wedge: its relayer ended at `rejected 0` with one re-bid, and the 123,801
still queued and 11,522 swept by `QueueTTL` are simply load the chain was never
going to carry, offered at up to 1,200 tx/s.

**Saturation figures are noisy and should not be read closely.** Sustained at
1,200 offered on chain 70 reads 189.6 here, 277.8 in the previous run, and 271.1
on op-geth. A two-minute window spans only eight to ten momentums and 36–48 % of
momentum slots are empty, so cadence luck dominates. The sub-saturation rows are
the trustworthy ones; treat the ceiling as "a few hundred tx/s", not a number.

## Correction: dynamic plasma does **not** raise the native lane's throughput

An earlier section of this document, written from two isolated runs, claimed the
native lane is "roughly seven times quicker to include" on chain 71 and that its
ceiling "nearly doubles". **The full matrix does not support that**, and this
supersedes it.

What survives is the latency observation and the block-count arithmetic:
`maxBasePlasmaInMomentum` 4,200,000 ÷ ~23,176 per send ≈ **181 blocks per
momentum** against v1's flat 100, and the runs do show 181 in a block. What does
not survive is the throughput claim:

| native, 15 offered | v1 | v2 |
|---|---:|---:|
| included | 1,799 | 1,042 |
| sustained | **6.6** | 4.5 |
| p50 | 58.6 s | **8.1 s** |

v1 includes *more* and queues for it; v2 includes *less* and never queues — its
p50 sits at ~8.1 s whether it is offered 5, 10 or 15 tx/s, which is the shape of
a lane limited **at the point of submission** rather than at inclusion. The
likely reason is the one thing v2 adds per block: each native send must pay
`ceil(base × price / 1000)`, the price rises under load, and an account's fused
plasma is finite — so accounts throttle themselves before the momentum's
block-count ceiling is ever the binding constraint. That is a hypothesis fitted
to the latency signature, not a measurement, and it is the obvious next thing to
test.

### Fees, measured after the repricing (2026-09-16)

Every step below ran with `minBaseFee` 150 gwei, senders tipping 100 gwei and
the relayer refusing under 50 (D39). Fees are read from `/api/fees/series`,
which agrees with the contract balances to tens of wei.

| chain | step | tx/s | txs | base (burned) | relayer | ZVM burn | total ZNN | USD | per tx |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|
| 70 | transfer @25 | 24.98 | 3,009 | 9.45 | 3.15 | 3.15 | 15.74 | $2.36 | $0.000785 |
| 70 | transfer @100 | 99.95 | 12,008 | 37.79 | 12.60 | 12.60 | 62.99 | $9.45 | $0.000787 |
| 70 | transfer @400 | 377.13 | 45,277 | 144.16 | 47.53 | 47.53 | 239.23 | $35.88 | $0.000793 |
| 70 | erc20 @250 | 201.42 | 24,191 | 124.41 | 41.47 | 41.47 | 207.34 | $31.10 | $0.001286 |
| 71 | transfer @25 | 24.98 | 3,009 | 9.45 | 3.15 | 3.15 | 15.74 | $2.36 | $0.000785 |
| 71 | transfer @400 | 106.02 | 13,824 | 43.48 | 14.49 | 14.49 | 72.47 | $10.87 | $0.000786 |
| | **total** | | **101,318** | | | | **613.52** | **$92.03** | |

Four things these numbers say:

- **Throughput is unchanged by the price.** erc20 at 250 offered read 201.42
  tx/s against 201.59 measured before the repricing; transfer at 400 read 377
  against 399. Both inside the band a two-minute window gives.
- **The price is flat per unit of gas, not per transaction.** A plain transfer
  costs $0.000785 and an ERC-20 transfer $0.001286 — the ratio 1.64 is exactly
  34,248 ÷ 21,000. That is the fee doing its job.
- **It creeps up under load.** Across chain 70's transfer steps the per-
  transaction cost rises $0.000785 → $0.000787 → $0.000793 as the base fee
  leaves its floor: sampled at **157–158 gwei** during saturation, against the
  150 gwei minimum.
- **Both rulesets charge the same.** Chain 70 and chain 71 agree to the wei on
  the same step — 15.744750 ZNN for 3,009 transactions on each.

For scale: this whole run earned **$92.03 in about fifteen minutes of load**,
where the chains' entire prior history — 192,497 transactions — had earned
$6.06.

**Re-measured once the lane stopped wasting NoM space and the target came down
to 75 M gas** (elasticity 4), same step, same chain:

| | before | after |
|---|---:|---:|
| dropped `block full` | 16.2 % of what the lane carried | **0.31 %** |
| throughput at 400 offered | 377.13 tx/s | **399.11 tx/s** |
| NoM bytes per transaction | 73–85 | **69** |
| base fee under load | 150.0 → 157.9 gwei | **150.0 → 273.9 gwei** |

Both numbers move the right way at once: the chain carries more and writes less,
because a batch the engine would refuse is no longer bought. Treat the fee range
as the headline — a base fee that reaches 83 % over its floor under load and
returns to it when quiet is a fee market, where 157 gwei was a rounding error on
a fixed price.
