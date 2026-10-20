# Decisions

Every choice that is expensive to reverse, why it was made, and what reversing
it costs. Written the way `projects/zenoglyphs/ASSUMPTIONS.md` is: a decision
without its cost of reversal is not recorded, it is just asserted.

Dated 2026-09-15 unless noted. The plan this implements is *Mainnet-Adjacent
State Machine* (2026-09-11), option 4, Path A.

**Review of 2026-09-15.** The operator's questions and calls on D2–D34 and on
the open questions are answered under each decision, marked **Review**. Where a
call changes a decision, the entry says so and what is built versus decided.
[What changed](#what-the-review-changed) at the end lists the work it created.

---

## The shape

### D1 · A based rollup, not a sidechain or an optimistic rollup

NoM orders; op-geth executes; anyone can re-derive. **Prior art:** Facet on
Ethereum and Kasplex and Igra on Kaspa all landed here, and Facet arrived by
migrating *away* from an inscription-envelope VM. Zenon has already run the
alternative — Supernova, a validator-set sidechain — and it is dormant, its
history gone with its RPC.

**Reversing:** a different ordering source is a different chain. Nothing
derived would survive.

### D2 · op-geth as the execution client, not op-reth

The user chose it (2026-09-14). It is Go like the rest of this tree, and it is
Facet's choice.

Worth knowing: Optimism ended op-geth support on 2026-05-31 in favour of
op-reth, and release v1.101702.3 (2026-08-31) says plainly "we don't support
running op-geth from this release. This is a library release only." The last
release with a published image is **v1.101702.2**, which is what the prototype
pins, image and library both.

**Reversing:** op-reth reads the same Engine API and the same OP genesis
format. Swap the container and re-sync; derivation does not change.

#### Review — is there a reason to switch today, other than support?

Yes, and the strongest one is not support but **security**. Optimism's notice
says patches and critical fixes were issued *through* 2026-05-31 and then
stopped. Any go-ethereum vulnerability disclosed since is unfixed in the image
this chain runs, and will stay unfixed. A client that will never be patched is
acceptable for a devnet and not for anything holding value.

What op-reth has that op-geth does not, as of 2026-09:

| | op-geth v1.101702.2 | op-reth |
|---|---|---|
| Security fixes | none since 2026-05-31 | current |
| Forks | through Jovian; Optimism states it does not support Karst | **Karst** (OP Upgrade 19, activated on OP mainnet chains 2026-07-08): the Osaka EVM — `CLZ` opcode (EIP-7939), a 2²⁴ per-transaction gas cap (EIP-7825), MODEXP bounds and repricing (EIP-7823, EIP-7883), P256VERIFY repriced (EIP-7951). op-geth "can no longer follow the canonical chain" on OP chains. |
| Historical state proofs | no | `--proofs-history`: `eth_getProof` against old blocks without an archive node — what a light client or a proof-based exit would read |
| In-process indexing | no | Execution Extensions (ExEx): code that runs inside the node on every committed and reverted block. Our indexer is a separate reader today and is the largest store per transaction (1,121 B/transfer); an ExEx is the architecture reth offers for it |
| Disk | our fleet benchmark measured op-geth's running `chaindata` at 550 MB apparent, **1.49 GB allocated**, for 163 MB of real chain after a restart — pebble write-ahead logs | static files for history, MDBX for state; not measured here |
| Stability | frozen | reth 1.0 declared production-ready June 2024; 2.0 April 2026; the client every OP chain now runs |

Costs specific to the ZVM, which are what make this more than a container swap:

1. **D12 depends on op-geth's exact error text.** `internal/l2/engine.go:77`
   matches `failed to force-include tx: 0x…`. op-reth handles an invalid forced
   transaction in its own way and says so in its own words. The rebuild rule
   has to be re-established against op-reth and re-checked for determinism —
   this is the one derivation-level risk.
2. **EIP-7825 caps a transaction at 16,777,216 gas** once Karst is on. The spec's
   `maxTxGas` for relayed transactions is 30,000,000 and must come down to match
   (`maxNativeGas` already is 16,777,216).
3. The Go code keeps importing op-geth as a *library* for types and genesis
   building, which is exactly what v1.101702.3 is published for. Nothing Go-side
   is lost.
4. Every benchmark number in `docs/BENCHMARK.md` is op-geth's and would need
   re-running.

**Recommendation:** switch before any public testnet, with (1) as the gate.
The testnet is a new genesis anyway, so it is also when D3's forks move.

**Decided 2026-09-15: upgrade to op-reth.** Not built. The migration is gated on
re-establishing D12's rebuild rule against op-reth's refusal behaviour, which is
the one place derivation depends on the client's own words. EIP-7825's 2²⁴
per-transaction cap arrives with it (D3): `maxTxGas` drops from 30,000,000 to
16,777,216, which is **below what two measured workloads declare** — the
benchmark's contract deploy (1.4 M declared, fine) is not the problem, but a
relayed transaction may currently declare up to 30 M and some wallets will. The
gas rule has to be re-fitted to the new ceiling and the benchmark re-run against
it; that is the same exercise as D10's share and D9's budget, so all three move
together.

#### Built 2026-09-16: both chains now run op-reth

`op-reth 1.10.2` is the execution client on chain 70 and chain 71, through
`docker-compose.reth.yml` and `docker-compose.reth-dp.yml`. The L2 state was
wiped and both chains re-derived from their own NoM history, which was kept:
derivation is a function of the L1, so every contract came back at the same
address and no deployment had to be redone.

**The blocker was not the one this decision predicted.** D12's refusal text was
already handled — the payload comparison added on 2026-09-16 made the rebuild
rule client-agnostic before the client changed, and in the whole re-derivation
of both chains no block needed a second attempt. What actually stopped the
migration was **the genesis block**: op-geth and op-reth hashed the *same*
`genesis.json` differently.

| field | op-geth | op-reth |
|---|---|---|
| `withdrawalsRoot` | `0x0000…0000` | `0x56e81f17…` (empty-trie root) |

Every other field was identical, so the chain would not start: `rollup.json`
recorded op-geth's `0x2fc01573…` and reth offered `0x6fa26153…`, and the
deriver refuses a genesis it does not recognise (`node/deriver.go`). That guard
is what turned a silent chain split into a startup error.

The cause is that **Isthmus sets the header's `withdrawalsRoot` to the storage
root of the `L2ToL1MessagePasser` predeploy, and this chain never deployed it** —
value leaves through `ZVMBridge`, so OP's withdrawal path is unused. With the
account absent the spec says nothing, and each client filled the hole its own
way: op-geth with the zero hash, op-reth with the empty-trie root.

**The fix is to make the account exist**, so the rule reads something real. The
`L2ToL1MessagePasser` is now a predeploy like the others, its runtime code read
off OP Mainnet — through the proxy's EIP-1967 implementation slot, because
`0x42…16` there is a proxy and this chain wants the implementation, not OP's
proxy machinery. Its storage stays empty, both clients hash the same empty-trie
root, and the genesis is `0xb4e1ea6b…` on both chains — verified on the op-geth
side by `tools/genesis-cmp` and on the op-reth side by `eth_getBlockByNumber`.

The lesson generalises past this one field: **a chain that declares a fork
inherits every rule of that fork, including the ones that reference predeploys
it does not think it needs.** Two clients only agree about a missing account by
accident. `tools/genesis-cmp` prints op-geth's genesis header both from the
committed file and from `BuildGenesis`, which is what tells a genesis mismatch
apart from a stale committed file; it is worth running before any new genesis.

### D3 · Forks through Isthmus (Prague), Jovian and later off

Isthmus gives a modern EVM (EIP-7702, BLS precompiles, transient storage,
EIP-2935) and requires nothing of derivation. **Jovian would**: it makes the
first transaction's DA-footprint scalar consensus-critical and meters blocks by
DA footprint as well as gas, which changes what a block can hold for reasons
that have nothing to do with NoM.

op-geth also pins the pairs: Shanghai=Canyon, Cancun=Ecotone, Prague=Isthmus;
a mismatch is refused at startup.

**Reversing:** a fork bump is a scheduled activation time in the chain config
and a re-derivation from that height, or a new genesis while nothing is at
stake.

#### Review — what these terms mean, and the trade of each

A **fork** is a named bundle of rule changes that activates at a time. Ethereum
names its upgrades (Shanghai, Cancun, Prague, Osaka); the OP Stack names its own
(Canyon, Ecotone, Isthmus, Jovian, Karst) and each OP fork *includes* one
Ethereum upgrade's EVM changes plus OP-specific ones. That is why op-geth insists
on the pairs: the EVM a contract sees is set by the Ethereum name, the rollup
rules by the OP name.

| OP fork | EVM it carries | What it gives a contract or a user here |
|---|---|---|
| Canyon | Shanghai | `PUSH0`: every Solidity ≥ 0.8.20 contract uses it |
| Ecotone | Cancun | **transient storage** (EIP-1153, `TSTORE`/`TLOAD`): storage that clears at the end of the transaction — cheap reentrancy locks, and Uniswap v4's accounting depends on it; `MCOPY` |
| **Isthmus** (current) | Prague | **EIP-7702**: a MetaMask account can temporarily run contract code, which is how wallets offer batched calls and gas sponsorship without a new address. **BLS12-381 precompiles** (EIP-2537): cheap verification of BLS signatures, the kind validator committees and many bridges use. **EIP-2935**: the last 8,191 block hashes readable from state |
| Jovian | none | OP-only. A **DA footprint limit**: each transaction's estimated compressed size, times a scalar, counts against the block's gas limit. A **configurable minimum base fee**. A new operator-fee formula. The L1-info deposit and the header's `extraData`/`blobGasUsed` fields change layout |
| Karst | Osaka | `CLZ`, the per-transaction gas cap, MODEXP and P256 repricing (see D2); an L2 contract-manager for upgrading predeploys through the consensus layer; fault-proof changes. **op-reth only** |

What each choice buys and costs the ZVM:

- **Stay at Isthmus.** Nothing to re-derive, the spec and vectors stand, op-geth
  keeps working. The cost grows over time: compilers and frameworks move their
  default target to Osaka, and bytecode using `CLZ` will not deploy here. The
  mitigation is `evmVersion: prague` in tooling, which our verifier already
  records per contract.
- **Adopt Jovian.** The DA footprint limit prices *Ethereum* data, which this
  chain does not post; with the scalar written as zero its per-spec
  contribution is zero, so it would be inert — that zero behaviour must be
  confirmed against the client before relying on it. The **minimum base fee is
  worth having**: on an idle chain EIP-1559 decays the base fee towards a few
  wei — it read **250 wei** at block 4,732 on 2026-09-15, which the explorer
  used to print as `2.50e-7 gwei` — and a floor keeps
  priority-fee auctions short when load returns. The cost is a spec change to
  our L1-info deposit layout (§8) and new vectors.
- **Adopt Karst.** The current EVM, and the only fork line with a maintained
  client. Requires op-reth (D2), lowers `maxTxGas` to 16,777,216, and needs the
  contract-manager checked for being inert without OP's L1 contracts.

**Recommendation:** at the op-reth switch, start the testnet genesis on Karst
with Jovian's minimum base fee set and its DA footprint scalar zero. Doing it at
a new genesis costs no re-derivation.

**Decided 2026-09-15: switch to Karst.** Not built, and it carries D2 with it —
op-geth cannot follow Karst, so the fork and the client move in one step.

#### The genesis half is verified; the metering half is not (2026-09-16)

The client moved first, and **deliberately without Karst**: both chains stayed on
Isthmus so the benchmark changed one variable rather than two. But the Karst
genesis itself is now checked against the client that will run it.

`zvm l2-genesis -evm karst -min-base-fee 1000000` produces
`0x94c2bb186ce6c4d8ed56afeaa18a80b3f9162307c37ce094938450f03fb11d8f`, and
op-reth started on that same file agrees: the same hash, the same empty-trie
`withdrawalsRoot`, `extraData 0x01000000fa0000000600000000000f4240` — Jovian's
17-byte encoding, version byte `01`, denominator 250, elasticity 6, and the
minimum base fee `0x0f4240` in the last eight — and a fork schedule listing
**Jovian @0**. It starts clean.

**What is still unconfirmed is the claim this decision leans on**: that a DA
footprint scalar of zero makes Jovian's second metering rule inert. A genesis
cannot answer that; it needs transactions on a running Karst chain, where the
question is whether a block's DA footprint is computed and compared at all when
the scalar is zero, or short-circuited. The client is the authority and the spec
is ambiguous, so this stays open until it is measured — and until then, nothing
should be planned on the assumption that the rule is free.

#### Built 2026-09-16: both chains start on Karst

Both networks were re-genesised onto `karst` with `minBaseFee` 1,000,000 wei,
which is what "start the testnet on Karst" meant in practice: op-reth accepted
the genesis on both, `maxTxGas` came down to EIP-7825's 16,777,216, and
derivation has run from momentum 2 with no special case anywhere. The two
chains still differ in exactly one thing — the dynamic-plasma spork (D4) — so a
difference between them is still the ruleset and not the EVM.

`evmVersion` for verification stays `prague`: Osaka's EVM runs Prague bytecode,
and nothing in this tree emits `CLZ`.

#### Fixed 2026-09-16: the benchmark harness's own gas rule, re-fitted

The "gas rule has to be re-fitted to the new ceiling" above named the risk but
not the bug: `internal/bench/run.go`'s wallet-funding step chunked recipients
into groups of 400 at 50,000 gas each, sized for the old 30,000,000 ceiling —
200,000 + 400×50,000 = 20,200,000, which EIP-7825's 16,777,216 now refuses
outright. Every wallet-dependent fleet phase failed at `preparing wallets`
before submitting anything, which is why a fleet run against chain 70 showed
zero new transactions despite running for several minutes: `01-fund` and every
phase after it rejected immediately, and only the read-only `00-api-baseline`
phase had done any real work. `fundingChunk` is now derived from the live
`rollup.Config.MaxTxGas` (331 recipients at today's cap, not a constant sized
for a ceiling that no longer applies), so a future gas-cap change re-fits
itself instead of silently breaking funding again.

### D4 · The ZVM gets its own NoM network, running momentum v1

Chain 70, one pillar and one RPC node, **dynamic plasma left inactive** — so
momentums are version 1, the rules mainnet runs. The shared devnet (chain 69)
runs v2 with dynamic plasma, carries other projects' traffic, and would make
every benchmark number a measurement of someone else's load.

**Reversing:** point `ZVM_L1_RPC` at another node and regenerate `l2/`. The
ZVM's start height and genesis change, so the chain restarts.

#### Review — test on both, permanently

**Built 2026-09-15.** Both rulesets now run side by side and stay that way, so
nothing has to be ported to the second one later:

| | v1 — chain 70 | v2 — chain 71 |
|---|---|---|
| Momentums | version 1, no fee market: mainnet's rules | version 2, dynamic plasma **active from genesis** |
| Started by | `docker compose up -d` | `docker compose -f docker-compose.yml -f docker-compose.dp.yml --profile dp up -d` |
| Ports | L1 45997, explorer 47000, op-geth 47545 | L1 46997, explorer 48000, op-geth 48545 |
| Burned fee (D9) | off | **on** from momentum 10, so the congestion term is exercised |
| Written by | `zvm l1-genesis` | `zvm l1-genesis -chain-id 71 -dynamic-plasma` |

`zvm bench` already takes `-config/-l1/-eth/-api/-submit`, so every scenario runs
against either chain with no code change.

**Measured the same day, same load** (batch lane, transfer, 200 wallets, 120 tx/s
offered, 90 s, from a quiet chain):

| | v1 | v2 |
|---|---:|---:|
| Sustained | **119.9 tx/s** (kept up with what was offered) | **25.4 tx/s** |
| p50 / p90 | 14.4 s / 21.4 s | 38.9 s / 102.8 s |
| Largest L2 block | 2,485 transfers | 681 transfers |
| Fusion price after | — (no fee market) | 1,000 → 1,100 |

**Dynamic plasma costs this rollup roughly five times its throughput**, and the
cause is not the ZVM: a momentum's user blocks are capped at
`maxBasePlasmaInMomentum` (4,200,000), a 16 KB block costs 1,135,112 of it, and a
relayer account can hold at most 10.5 M plasma (5,000 fusion units × 2,100) — a
protocol ceiling, not a funding choice. On a native-lane run the price rose
1,000 → 1,689 and the chain still admitted 151 blocks in one momentum, against
v1's hard cap of 100.

Two consequences worth stating before anyone plans around dynamic plasma:

- **`maxBasePlasmaInMomentum` is the number that matters**, and it is settable
  with `SetVariables` on the Plasma contract. The measurement above is at its
  default.
- **More relayer accounts, not more QSR**, is how a relayer buys capacity: each
  account is capped at 10.5 M plasma however much is fused to it.

---

## Addresses and messages

### D5 · The inbox is unspendable; the vault is not

The inbox `z1zvmxxxxxxxxxxxxxxxxxxxxxxxxxxxxxafdtd6` has type byte `0x13`,
which neither a keypair (`0x00`) nor an embedded contract (`0x01`) can produce
— zenoglyphs proved that shape works on a live devnet with its `0x16` burn
sink. The **vault** is an ordinary keyed address, because value that leaves has
to be spendable by someone.

Keeping them apart means the protocol's identity is fixed and unspendable while
custody can be rotated (a new committee, a new key, a FROST resharing) without
changing what a ZVM transaction is addressed to.

**Reversing:** merging them is a spec change and a re-derivation; both
addresses are consensus.

**One inbox per rollup, not per network (2026-09-15).** Derivation reads every
message sent to the inbox on its L1, so two rollups sharing a network *and* an
inbox derive each other's calls, each against its own state — where the sender
usually has no balance, so they revert. That happened here: the committee
rehearsal ran a second rollup on chain 70 and its withdrawals appear as reverted
transactions on the main chain. `zvm l2-genesis -inbox-tag <word>` now derives a
distinct unspendable inbox (`z1zvm<tag>…`); one rollup on a network needs no tag.
The tag is bech32 data characters, whose alphabet has no 1, b, i or o.

#### Review — benefits and drawbacks of merging

**Prior art** keeps them apart everywhere: the OP Stack sends batches to a
data-only `BatchInbox` and value to the `OptimismPortal` contract; Arbitrum has a
`SequencerInbox` separate from its `Bridge`; Facet has only an inbox because it
has no canonical bridge.

| Merge into one keyed address | |
|---|---|
| **Gains** | One address for users to learn. Slightly simpler classification in derivation. Nothing on NoM gets cheaper: a deposit that carries a call is already one block (SPEC §7). |
| **Loses: identity tied to a key** | The address would have to be spendable, so the protocol's identity becomes the committee's key. Until resharing exists (D15), changing the committee changes that address — a hard fork for every wallet, relayer and hard-coded integration. |
| **Loses: the vault's chain fills with messages** | Every batch, chunk and native call becomes an unreceived send *to the vault*. Left unreceived, they bury the deposits the committee must receive; received, they double the ZVM's NoM footprint, the way deposits already cost two blocks each (fleet benchmark). |
| **Loses: the burn sink** | D9's burned-fee design needs an address nobody can spend from. The inbox is that address today. |

**Kept separate.**

### D6 · One account, two spellings

A NoM address and an EVM address are the same 20 bytes. No aliasing, no
mapping, no registry. A Zenon user's ZVM account *is* their Zenon account.

**Reversing:** aliasing (OP's approach for L1 contracts) would break every
address shown, held or hard-coded.

#### Review — do we need a mapping at all?

We do not have one, and this decision is the reason: there is **no mapping**.
Every ZVM address *is* convertible to a Zenon address and back, because the
conversion is only a change of spelling — the same 20 bytes, written as `0x…` or
bech32 `z1…`. `zvm constants` and the explorer already show both.

What cannot be converted is **control**, and no mapping could provide it:

- A Zenon account (first byte `0x00`) is controlled by an Ed25519 key. On the
  ZVM it can act only through the native lane (D28); it has no secp256k1 key, so
  it cannot sign an EVM transaction or an EIP-712 order.
- A MetaMask account is controlled by a secp256k1 key. Its `z1…` spelling
  almost never starts with `0x00`, and no Ed25519 key can produce it, so value
  sent there *on NoM* is lost. The bridge contract already refuses withdrawals to
  such an address.

A registry that links "this MetaMask account belongs to the owner of that Zenon
account" is possible — a signed claim from both keys — but it is an application
feature, not something consensus needs. OP and Arbitrum alias only L1
*contracts*, to stop an L1 contract impersonating an L2 one at the same address;
NoM has no user-deployed contracts to collide with.

**Kept, with a UI rule to add:** show the `z1…` spelling of an address only when
its first byte is `0x00`, and warn when a NoM recipient does not start with it.

**Decided 2026-09-15: no secp256k1 key in Syrius for now.** The operator may add
one later. Until then the split stands as above, and it is what forces D28's
marketplace decision: a Zenon account cannot produce an EIP-712 signature, so it
cannot make an off-chain order. The route that would give it one *without* a
second key is an ERC-1271 smart account verifying Ed25519 — worth costing
before assuming a secp256k1 key in Syrius is the only answer.

### D7 · Messages are RLP, typed by their first byte

`0x01` native call, `0x02` relayed batch, `0x03` chunk, `0x04` assemble. RLP
because the payload is already an EVM-shaped thing and RLP is what an EVM
client has; a type byte because it costs one byte and makes a new lane additive.

**Prior art:** Facet's `0x46` transaction is `rlp([chain_id, to, value, gas,
data, mine_boost])` sent to one inbox, with the L1 sender as `msg.sender`. The
ZVM drops the chain id (the inbox address and the NoM chain identifier already
name the chain) and the boost (there is no gas token to mine).

**Reversing:** a wire change is a hard fork of the ZVM at an activation height,
or a new chain. zvm-core's `vectors/vectors.json` exists to make an accidental change fail
loudly.

#### Review — `mine_boost` may come back with the gas change

Noted. In Facet, `mine_boost` is padding a user adds to burn more L1 gas and so
mint more of Facet's gas token. If D9's burned fee lands, the ZVM analogue is a
native call that burns more than its gas needs, for a purpose the protocol
rewards. The encoding cost is known: `rlp([to, value, gas, data])` is a fixed
four-item list, so a fifth item is a wire change at an activation height — the
decoder accepts an optional trailing item from that height and the vectors gain
a case. Decide it together with D9, not before.

### D8 · Chunks belong to their author and expire

An assemble may only name chunks published by the same NoM account, within
8,640 momentums. Otherwise a deriver must keep every chunk anyone ever posted,
forever, and one account could assemble another's bytes.

**Reversing:** relaxing is a spec change; tightening (shorter TTL) invalidates
assemblies in flight.

#### Review — relax it now, given the benchmarks?

**Prior art:** the OP Stack's channel frames are the same shape — frames are
accepted only from the configured batcher address and a channel times out after
a fixed number of L1 blocks.

What the benchmarks say:

- **The rule never bound.** Every chunk set in the fleet run was assembled
  within a few momentums of its first chunk, by its own author. The TTL is
  roughly a thousand times longer than any assembly took, and the `chunks`
  table was empty at the end of the run.
- **What did bind was cadence.** At 1,200 tx/s offered only 8 of 20 momentums
  carried a batch, because a chunked batch executes only when its assemble
  lands. Neither relaxation touches that; landing smaller batches every
  momentum does, and that is relayer behaviour, not spec.

| Relaxation | Would enable | Would cost |
|---|---|---|
| Any author may assemble | relayers sharing chunk sets; a data market | every deriver stores every chunk from anyone for the full TTL; anyone can re-assemble another account's native call under their own address |
| Longer TTL | nothing measured needed it | more deriver storage, no benefit |
| Larger `maxAssembledBytes` (512 KB) | contracts above ~500 KB, which the EVM's 24 KB code limit already rules out | a larger worst-case message |

**Kept.** The chunk-gas change decided under *Still open* (count the whole
payload's plasma, each chunk once) is the one chunk rule that does change.

---

## Gas and admission

### D9 · A native call's gas is the plasma its NoM block paid, times 16

The plan's prototype used 8 gas per plasma against a hand-written interpreter.
op-geth executes the same work far faster, so 16 is defensible and still
bounded: a full 16 KB block buys 18 M gas, capped at 16,777,216 per call.

**Reversing:** changing the constant changes every native transaction's gas
limit and therefore every block's hash from that height on.

#### Review — adopt zenoglyphs' burned ZNN/QSR fee

**Decided and built, 2026-09-15.** Off on this chain until an operator sets an
activation height; rehearsed live (below). zenoglyphs has run the same
arithmetic on its devnet since momentum 55,100
(`projects/zenoglyphs/research/evm-fee-design.md`); `internal/rollup/fee.go` is
the port, `spec/SPEC.md` §6 the rule, zvm-core's `vectors/vectors.json` the pinned cases.

```
gas = min( usedPlasma × gasPerPlasma ,      the byte bound, unchanged
           feeGas ,                          what the burn bought
           maxNativeGas , requested )

feeGas = ⌊ amount × gasPerSupply × priceDen ÷ (supply(token, epoch) × priceNum) ⌋
```

- **The burn needs no burn address.** The inbox is already unspendable by
  anyone (D5), so a native call's fee is simply the value its NoM block carries,
  which derivation ignored until now.
- **The price is a fraction of the token's own supply**, computed from NoM's
  published emission schedule — a pure function of the momentum's timestamp — so
  every node reaches the same number offline, with no oracle and no exchange
  rate. Each chain's anchor (`fee.refZnn`, `fee.refQsr`, `fee.refEpoch`,
  `fee.genesisTimestamp`) is consensus and lives in `rollup.json`;
  `zvm l2-genesis -fee-from N` reads it from the L1 and writes it.
- **Congestion is read off the carrier block**, as the ratio of the plasma it
  paid to what its bytes nominally cost. On a v1 momentum that is exactly 1, so
  the rule needs no second activation if dynamic plasma ever arrives.
- **A call that burns nothing is invalid**, not free: it derives to no
  transaction at all rather than to a call with a budget it cannot run on.
- **A deposit-carried call is capped, not burned** (`maxDepositCallGas`,
  1,000,000): a deposit's amount is bridged, so without the cap that path would
  be the way to buy native gas for nothing.
- **Every publisher pays through one path.** `derive.PublishCall` quotes the fee
  after the block's plasma is settled and attaches it before signing, with a
  0.5 % margin for the epoch boundary a publisher cannot predict. The CLI, the
  faucet, the benchmark, `apps setup` and the browser all go through it, and the
  node quotes for the browser at `/api/fee` — zenoglyphs' activation day broke
  its faucet precisely because one publisher had not been taught to pay.

**Rehearsed on chain 70, 2026-09-15**, by activating it on the committee
rollup eight momentums ahead — the upgrade path an operator would use, editing
`rollup.json` and restarting the node:

| | |
|---|---|
| Before momentum 5,141 | a call with no burn derives normally: 21,000 gas, plasma alone |
| After | the same call burns **0.0025 ZNN** for 21,000 gas, and a 60,000-gas call burns 0.0072 ZNN (≈ 12 raw ZNN per gas at this devnet's supply) |
| A publisher still on the old config | the same NoM block derives **invalid — "no fee: a native call burns ZNN or QSR to the inbox for its gas"** on the priced chain, and normally on the unpriced one. Both chains read the same block |
| The explorer | shows the burn, the gas it bought, and the price multiplier |

**Settled 2026-09-16 by measuring this engine.** `gasPerSupply` was the last
consensus value in the fee rule taken on trust. Over chain 70's 6,432 derived
momentums — 27.2 billion gas in 666 s of build time — the loaded blocks (≥ 1 M
gas, 517 of them) built at **46,088 gas/ms** and the busiest band (> 50 M, 207
blocks) at 47,654. The figure is the whole derivation step rather than
`evm.Call` alone, deliberately: what an operator pays for is the derive loop.
Holding zenoglyphs' intent — the whole supply buys about 29 days of compute —
gives **1.15e14**, now the default for the next genesis. Chains that already
wrote a `gasPerSupply` keep theirs; changing it on a live chain is a fee change
at an activation height.

#### The relayed lane's fee: half to the relayer, half burned

**Decided 2026-09-16. Not built.** Two rules settled with it:

- **"Burned" on the ZVM means locked in the vault forever** — the backing ZNN
  stays on NoM and nothing can redeem it — *unless* the committee also sends the
  matching amount to an unspendable NoM address. Both are honest; only the
  second shows up in NoM's own supply, and that is the version to aim at.
- **A self-relayed transaction pays itself.** Anyone may post their own batch,
  and then they are the relayer. Allowed for now; revisitable.

**Attribution is the part that changes derivation.** An EVM block has one
`coinbase`, but one ZVM block carries batches from many relayers, so the block
cannot name who to pay. Derivation *does* know — every message records its NoM
author — so the payment has to be an explicit derived transaction: after the
user transactions, a system call to a fee-splitter predeploy carrying the
per-author amounts computed from that block's own messages. Every node computes
the same list from the same momentum, so it stays deterministic; nothing is read
from outside the block.

`mine_boost` (D7) feeds the same split on the burn side, and is what gives a
native caller the priority knob replace-by-fee gives the batch lane.

### D10 · Native calls get at most half of each block

They pay no ZVM gas, so nothing prices them; without a share, a single
well-fused Zenon account could fill every block and push relayed users out
through the base fee. Half is a guess made explicit — the plan lists the
footprint cap as a decision the operator owes an answer to.

**Reversing:** a parameter change at an activation height.

#### Review — other ratios

**Prior art:** the OP Stack meters its forced-inclusion lane — deposits from
L1 — with an absolute budget, 20,000,000 guaranteed gas per L1 block, priced by
an EIP-1559 market that burns L1 gas; its L2 block gas limits have since grown far
past that, so the share has shrunk rather than been kept proportional.
Ethereum's EIP-4844 gives blobs a separate budget with a separate price rather
than a share of one.

What the lane can physically use: at most 100 NoM blocks a momentum (v1), and a
native call's gas limit is its plasma × 16 — about 370,000 for a transfer and
790,000 for the measured native NFT mint. So **100 ordinary native calls declare
40–80 M gas, 13–27 % of a 300 M block.** The half-block cap only binds for
deployment-sized calls: nine calls at the 16.7 M maximum fill it.

| Rule | Native gets | Relayed is guaranteed | Binds when |
|---|---|---|---|
| ½ (today) | 150 M | 150 M | ~9 maximum-size calls |
| ⅓ | 100 M | 200 M | ~6 maximum-size calls |
| ¼ | 75 M | 225 M | ~4 maximum-size calls; 100 ordinary calls still fit |
| 10 % | 30 M | 270 M | ~40 NFT mints — below the lane's own ceiling, so it would drop ordinary use |
| OP-style absolute, 20 M | 20 M | 280 M | ~25 NFT mints |
| **Elastic: ¼ floor, ¾ ceiling** | 75 M always, up to 225 M when relayed demand leaves room | 75 M always | only under simultaneous heavy demand on both lanes |

The elastic rule is two passes over the same ordered list: admit every relayed
transaction up to ¾, then native calls up to what is left (at least ¼), then
execute in content order. It does not change execution order, only which
transactions are admitted.

**Recommendation:** ¼ now, as a constant, because it never binds on the ordinary
calls measured and protects relayed users better than ½. Move to the elastic
rule together with D9, because once native calls burn a fee, their share can be
priced rather than only capped.

### D11 · Deposits and payouts are mandatory; everything else can be dropped

Their gas is reserved before any optional transaction is admitted, because the
value they record has already moved on NoM. A deposit that "did not fit" would
be value taken and not credited.

**Reversing:** none sensibly. This is the invariant that makes the peg safe to
reason about.

### D12 · The engine has the last word, and we rebuild without the transaction it named

op-geth refuses a forced transaction by naming it (`failed to force-include tx:
0x…`, carried in the JSON-RPC error's *data*). Derivation removes exactly that
one and rebuilds. Deterministic, because the same inputs and the same client
produce the same refusals — and cheap, because a nonce/fee pre-check against
the parent state catches almost everything first.

A refused **deposit or native call** is not a drop: it is a consensus failure
and the node halts rather than inventing a chain.

**Reversing:** a different rule (skip-list, deposits-only fallback) is a
derivation change and re-derives the chain.

**Decided 2026-09-15: revisit after the move to op-reth (D2).** The rule reads
op-geth's refusal text, and the client is changing; the halt analysis is worth
doing once, against the client that will run it.

#### Review — in plain terms

Derivation builds the list of transactions for a block *before* anything is
executed, using the state of the previous block. Some transactions only turn out
to be invalid while the block is being executed, because of something earlier
in the same block.

Example: Alice has 1 ZNN. A relayer's batch carries her transaction sending 1 ZNN
to Bob, and a second relayer's batch, later in the same momentum, carries her
transaction sending 1 ZNN to Carol with the next nonce. Checked against the
previous block, both look fine. Executed in order, the second one has no balance
left.

Derivation hands op-geth the list and says "build exactly this". op-geth
refuses and names the transaction it could not include. Derivation removes that
one transaction, asks again, and repeats until op-geth accepts. Every node runs
the same list through the same client, gets the same refusals in the same
order, and so builds the same block — which is what makes the rule safe for a
chain with no sequencer.

Two kinds of transaction are never removed this way: **deposits**, because the
ZNN already moved on NoM, and **native calls**, because a signed NoM block
is not something a node may decide to ignore. If op-geth refuses one of those,
the node stops rather than produce a chain other nodes might not.

The rule's weak point is the one D2 names: it reads op-geth's error message.

#### What the op-reth migration actually showed (2026-09-16)

**The rule was never exercised, and that is the finding.** Across the full
re-derivation of both chains on op-reth — chain 70's entire history and chain
71's, plus benchmark load on top — **every block was built on the first
attempt**. Not one refusal, on either client.

The reason is that `prevalidate` does its job: a relayed transaction whose nonce
or fee the parent state already rules out is dropped before the engine is asked,
so the cases that would provoke a refusal mostly never reach it. What remains is
the genuinely concurrent case — two batches in one momentum whose transactions
only conflict once executed in order — and the relayer makes that rare by
serialising one sender onto one account (D18, defect 5).

So op-reth's refusal wording is **still unknown**, and the honest statement is
that the ordering of D12's three branches has not been tested against it. This
matters less than it did, because the third branch — comparing the payload the
engine returned against the list it was given — needs no wording at all and is
what a client that silently skips a transaction leaves behind. The first two are
a round-trip optimisation over it.

Two consequences worth carrying:

- **The client swap did not need D12 re-established.** D2 called this "the one
  derivation-level risk" and it was retired before the client changed, by making
  the rule client-agnostic rather than by learning reth's prose.
- **The real client-dependency was somewhere nobody looked**: the genesis block
  (D2). A rule that reads an error message is at least visibly client-specific;
  a header field that two clients fill differently for a missing account is not.

#### The one halt this rule caused is closed (2026-09-16)

D12's rule says a refused *mandatory* transaction stops the node, and
`docs/HALT.md` path 6 was the case where that could happen without anybody
misbehaving: more deposits and payouts in one momentum than a block's gas limit
can hold. From the gas fork they no longer have to fit. Derivation caps what
mandatory messages may claim (`mandatoryShareBp`) and **defers** the overflow to
the next block, in strict arrival order, so the queue is the same on every node
(spec §8). A deposit deferred is still a deposit; a chain that stopped was not
recoverable without an operator.

This is the only consensus change on that page, which is why it waited for an
activation height rather than shipping as a fix.

### D13 · NoM's order is the ZVM's order

Momentum content order is `address ‖ height ‖ hash`, so a lower address sorts
first and address grinding buys priority — this tree has two vanity generators.
The alternative the plan offers, shuffling by the momentum hash, moves that
power to the pillar producing the momentum. Neither is fair; NoM's own order is
at least the one everything else on NoM already lives with.

**Reversing:** a derivation change, and a re-derivation.

#### Review — keep mainnet's order, and handle dynamic plasma's too

Read in go-zenon (`testnet` branch, commit `590f43c`) to settle what "both
cases" means, because it turns out the two rules differ in *which blocks get
in*, not in *the order they are in*:

| | v1 (mainnet today) | v2, dynamic plasma |
|---|---|---|
| Which blocks enter a momentum | `accountPool.GetNewMomentumContent`: accounts are walked in **Go map iteration order** and the walk stops at 100 blocks (`chain/account_pool.go:433`). Under congestion, which accounts make it is effectively random. | `pillar/content_selector.go`: contract blocks first, then user accounts merged by **price**, highest first, filling the base-plasma budget |
| Order they are committed in | `nom.NewMomentumContent` sorts by `address ‖ height ‖ hash` (`chain/nom/momentum_content.go:41`) | **the same sort** — `pillar/worker_momentum.go:43` calls the same function |
| Within one account | height order | height order (the selector preserves ancestor before descendant) |

So **derivation already handles both correctly**: it takes the committed order,
and the committed order is the same byte sort under either rule
(`internal/derive/plan.go:103`). Address grinding buys position *within* a
momentum in both. What dynamic plasma adds is that price, not luck, decides
*which* momentum a block lands in — and the relayer is what has to cope with
that (D18).

**Kept, with a test to add:** a derivation vector or a DP-network run (D4) that
feeds a v2 momentum through `BuildPlan`, so "handles both" is checked rather than
read.

---

## The peg

### D14 · A committee vault, not a burn receipt

The plan's open decision 2. A burn is trustless but never redeemable 1:1, which
makes "value out" a counterparty search rather than a peg. The committee is the
same trust wZNN already relies on, and it is bounded: the committee cannot mint
on the ZVM, and every payout it makes is derived back into the bridge contract,
which flags anything that matches no withdrawal.

**Prior art:** Igra locks KAS at an L1 address to mint iKAS 1:1; Kasplex
bridges KRC-20 through a multisig; Facet chose a burn and has no canonical
bridge.

**Reversing:** switching to burn receipts makes existing ZVM ZNN unbacked;
switching to the NoM bridge contract (`SetNetwork`, no spork) needs the bridge
administrator and the orchestrators, and is the documented next step.

**Review:** the operator's answer to "what ZNN is inside" confirms this; see
*Still open*.

### D15 · A FROST committee, created by a distributed key ceremony

**Changed 2026-09-15 from "one key now, FROST documented as the replacement".
Built and rehearsed.**

`projects/amm/frost` proved on devnet (2026-09-13) that a FROST(Ed25519)
quorum can hold an ordinary `z1…` address and that go-zenon cannot tell. What
that spike did not have is a way to create a committee on a real network: its
key was split by a trusted dealer, one process that held the whole secret.

**Built** (`internal/committee`, `zvm committee`, `docs/COMMITTEE.md`):

- **Distributed key generation**: the FROST paper's KeyGen — Pedersen DKG with a
  proof of knowledge of each constant term — which is what Zcash Foundation's
  audited `frost-core` implements. No machine holds the vault key at any point.
  File-based, so members can run it air-gapped; round-2 values are encrypted
  and authenticated to their recipient; members compare a transcript fingerprint
  out of band before anything secret is sent; `zvm committee audit` recomputes
  the vault address from public files.
- **RFC 9591 signing**, ported from the amm spike **with a correction**: that code
  left the group public key out of the binding-factor input (RFC 9591 §4.4). Its
  signatures verified, because verification never sees binding factors, but they
  were not RFC 9591 signatures. The port now reproduces the RFC's Appendix E.1
  test vector exactly, and a bad signature share names its signer.
- **A signer daemon per member** that checks every block against *its own*
  nodes: it signs only receives of sends to the vault, and payouts that match a
  finalized, unpaid `ZVMBridge` withdrawal exactly, at least `-payout-depth`
  momentums old, at the vault's next height, never twice.
- **The bridge as an untrusted coordinator** (`zvm bridge -group -signers`): it
  holds no share and falls back to another quorum when a signer refuses.
- `zvm l2-genesis -vault` names a committee's address.

**Rehearsed on the ZVM's local L1 (chain 70), 2026-09-15**, as a second rollup whose
vault is a 2-of-3 committee (`docker-compose.committee.yml`):

| Check | Result |
|---|---|
| Three members, files only | one vault address, `z1qq33pc5jjqatxj0p3v9pdqur4apm6qjzps76y9`; identical fingerprints and byte-identical `group.json` from all three; audit reproduces it from round 1 alone |
| Deposits | two sends to the vault received by quorum signature and confirmed |
| Withdrawal | 2 ZNN recorded in L2 block 16, paid from the vault by quorum, derived back as paid **118 s** later |
| A hostile coordinator asking for: an untagged send, a replay of a paid withdrawal, a larger amount, a withdrawal that does not exist, a block ahead of the frontier | all five refused by each signer, each with its reason |
| Two of three signers stopped | the bridge stops paying and names both; one restarted, the stalled withdrawal paid |

**Not built:** resharing (so changing membership is a new vault address),
rate limits on outflow, and any external review. `docs/COMMITTEE.md` lists them.

**Reversing:** none needed for the committee itself. Once resharing exists, the
signer set can change without the vault address changing.

### D16 · ZNN is native and scaled ×10¹⁰; QSR is an ERC-20 with 8 decimals

The ZVM's gas token has to be 18 decimals to look like every other EVM chain.
QSR keeps NoM's 8 decimals so that an amount means the same number on both
layers, which is what a bridge audit needs.

**Reversing:** changing either is a token migration.

### D17 · Withdrawals wait six momentums on devnet, thirty on testnet and mainnet

Six is a devnet number, chosen so a round trip is demonstrable in two minutes.
The bridge orchestrators treat 90 momentums as final on mainnet.

**Reversing:** a configuration change; longer is always safe, shorter is not.

#### Review — 30 momentums on testnet and mainnet

**Decided: 30.** On mainnet's measured 11.25 s interval that is about 5.6 minutes,
against about 17 minutes at the orchestrators' 90.

| | 6 | **30** | 90 |
|---|---|---|---|
| Wait on mainnet | ~1.1 min | **~5.6 min** | ~17 min |
| Who uses it | this devnet | — | the Zenon bridge's orchestrators (`confirmationsToFinality`) |

There is no measurement in this tree of how deep a NoM momentum reorganisation
can go — go-zenon does not reorganise in normal operation, and nothing has
tested it under a pillar fault. So 30 is a judgement that a reorg deeper than
five minutes is not a realistic risk for the amounts this bridge will hold at
first, not a derived bound.

Two consequences, both now reflected in the code:

- **The payout delay is the signers' setting, not the ZVM's.** Each signer
  enforces `-payout-depth` against its own NoM node, so a coordinator cannot
  shorten it. The ZVM's own `finalityDepth` stays at 6 (see *Still open*): that
  is when a block is labelled finalized, which wallets and the explorer read, and
  it does not move value.
- **Size-tiered delay is the natural next step.** Wormhole's Governor holds
  transfers above a notional limit for 24 hours; the same rule here would pay
  small withdrawals at 30 momentums and large ones at 90. `Policy.Check` is where
  it goes.

---

## The services

### D18 · Relayers shard senders across eight NoM accounts

A sender's transactions must stay in nonce order, and NoM orders blocks by
address within a momentum — so two accounts posting the same sender's
transactions can invert them. Each sender is pinned to one relayer account by a
hash of its address.

**Reversing:** none; it is a relayer implementation choice, not consensus.

#### Review — correct under both ordering rules, and now priced for both

The pinning is right under both (D13): within one NoM account, blocks are always
committed in height order, and across accounts both rules can invert — v1 by map
order when the momentum is full, v2 by price.

What did **not** survive dynamic plasma was how every service paid. All of it is
fixed and tested against the chain-71 network (D4):

1. **A block pays the network's price, not its base plasma.** `znn.Account` now
   pays `ceil(basePlasma × nextFusionPrice / 1000)` — which *is* base plasma on a
   v1 network, so one formula covers both. Paying base plasma while the price is
   above the minimum is what wedges an account.
2. **Replace-by-fee, automatically.** A block signed before the price moved is
   not refused, it is stranded, and every later block on that account with it.
   `Account.AutoRebid` (on by default) re-publishes the wedged block at the
   current price on the next publish; `zvm rebid` does it for an account no
   service is driving. **Measured: 22–24 blocks per account were wedged after a
   congested run, and rebidding cleared them.**
3. **The whole queue is re-chained.** A replacement changes the block's hash, so
   the blocks behind it — which name the old hash as parent — are dropped with
   it. Rebidding only the wedged block took an account from 24 queued blocks to
   1; re-chaining the tail took another from 23 queued to **confirmed height 25**,
   all of them landing.
4. **The relayer sizes blocks to what its plasma can afford.** At 1.8× price, a
   16 KB block costs ~2 M plasma and eight in flight need 16 M against the 10.5 M
   an account can hold — which showed up as `not enough plasma on account` under
   load. It now computes the affordable byte room per block from the account's
   available plasma and the current price, which comes out at a full block on a
   v1 network. After the change the same run produced **zero** plasma refusals.

#### The hole in (2), found by the op-reth benchmark (2026-09-16)

"…on the next publish" was the whole problem. **An account that has nothing new
to send never publishes, so it can never re-bid, so it never gets anything new
to send.** Measured on chain 71: a batch fell behind the price, was requeued and
eventually discarded, each discarded transaction left a nonce gap, `ready()`
stopped offering those senders, `poster` stopped calling publish — and the
relayer sat with 31,705 transactions queued, **nothing in flight and budget
left**, delivering 0.00 tx/s at 400 offered. Only a restart cleared it. The
`QueueTTL` sweep does not help: the missing nonce is one no client will resubmit.

Two changes close it:

- **The relayer re-bids on a timer** (`RebidInterval`, 20 s), independent of
  having anything to send. `Account.Rebid` already existed for `zvm rebid` and
  returns nil unless the oldest unconfirmed block is genuinely underpaying, so
  the check is cheap when there is nothing to do. `/status` counts `rebids`.
- **A batch is found again by height.** Re-bidding re-signs and re-chains every
  unconfirmed block, so their hashes change — and the tracker looked batches up
  by NoM block *hash*, which meant any re-bid silently orphaned its own batch
  until `DropAfter` requeued everything behind it. The batch now records the
  height it was published at, and a hash the node no longer knows is re-resolved
  to whatever stands at that height. This bug predates the timer: it applied to
  the publish-triggered re-bid too, which is why (2)'s measured success was on
  accounts the relayer was not tracking.

**Measured, same ladder, no relayer restarts between steps:** at 100 offered
tx/s the re-bidder fired eight times and **discards went from 2,605 to zero**,
with throughput 36.47 → 51.09. At 400 offered, with the relayer carrying the
previous step's backlog — the exact condition that had produced a frozen
`0.00 tx/s` — it delivered **28.00** and never stopped posting. The discards
that remain at 400 are ordinary overload swept by `QueueTTL`; `rebids` does not
move during that step, so they are not the wedge. Dynamic plasma still carries
far less than v1, but it no longer stops.

Signatures do not compress, so the gain is modest and the same either way:
measured **0.60** (21,477 bytes posted for 35,734 raw) on 120-byte transfers.
Brotli would do a little better and adds a dependency to something every node
must decode identically.

**Reversing:** a new compression flag value is additive — old batches keep
decoding.

#### Review — brotli against zlib, measured

**Measured 2026-09-15** by taking every batch the fleet benchmark posted to the
chain-70 L1 (4,253 batches, 42.2 MB decompressed, reassembled from chunks where
needed), decompressing it, and recompressing it with each codec. Size is posted ÷
raw; the percentage is against today's zlib level 9.

| Workload | Batches | Raw | zlib-9 (today) | brotli-11 | brotli-6 | zstd-19 |
|---|---:|---:|---:|---:|---:|---:|
| Transfers, 1k wallets | 561 | 2.8 MB | 0.791 | 0.768 (−3.0 %) | 0.769 (−2.8 %) | 0.772 (−2.4 %) |
| Transfers, 100k wallets, new recipients | 2,082 | 11.6 MB | 0.786 | 0.764 (−2.7 %) | 0.765 (−2.6 %) | 0.768 (−2.3 %) |
| Saturation, 600 tx/s | 625 | 11.7 MB | 0.774 | 0.755 (−2.5 %) | 0.756 (−2.4 %) | 0.759 (−1.9 %) |
| Uniswap swaps | 582 | 3.5 MB | 0.284 | 0.271 (−4.7 %) | 0.272 (−4.3 %) | 0.273 (−3.8 %) |
| NFT mints | 611 | 2.4 MB | 0.369 | 0.351 (−4.9 %) | 0.351 (−4.9 %) | 0.354 (−4.0 %) |
| NFT listings | 862 | 4.9 MB | 0.315 | 0.301 (−4.6 %) | 0.300 (−4.8 %) | 0.303 (−3.9 %) |
| Farm | 930 | 5.2 MB | 0.369 | 0.355 (−3.7 %) | 0.353 (−4.2 %) | 0.356 (−3.4 %) |
| **All** | 4,253 | 42.2 MB | **0.611** | **0.593 (−3.0 %)** | 0.593 (−2.9 %) | 0.596 (−2.5 %) |

| Speed, this host | Encode | Decode |
|---|---:|---:|
| zlib-9 | 16.5 MB/s | 93 MB/s |
| brotli-11 | **0.3 MB/s** | 79 MB/s |
| brotli-6 | 12.7 MB/s | 103 MB/s |
| zstd-19 | 8.6 MB/s | 403 MB/s |

**Brotli is 3 % smaller here: 2.5–3 % on transfers, 4–5 % on application calls.**
The ceiling is what it always was: a transfer is ~110 bytes of which 65 are a
signature, and a signature is random. Speed is not the obstacle at today's volume —
the saturated relayer produced about 21 KB of raw batch data a second, well inside
even brotli-11 — but brotli-11 is 50× slower than zlib for 0.1 % more, and brotli-6
gets the same size at zlib's speed, so 6 is the level to use.

**Prior art:** the OP Stack added brotli as a channel compression option in its
Fjord upgrade, and Arbitrum Nitro compresses batches with brotli. The Go library
both ecosystems use, `andybalholm/brotli`, is pure Go.

What 3 % buys here: the batch lane's ceiling is the relayer's in-flight window of
NoM bytes, so about 3 % more transactions a second at saturation (195 → ~201
tx/s), and 3 % less plasma per transaction. Under dynamic plasma, where every
byte is priced (D4), the same 3 % is 3 % of the fee.

**Recommendation:** add flag `0x02 = brotli` (quality 6) when the next wire change
ships — D9 or D28 — since the flag is additive and costs one decoder. Not worth a
change on its own. A bigger gain, if one is wanted, is structural rather than a
codec: OP's span batches store transaction fields column by column so that
repeated chain ids, gas prices and recipients compress together.

### D20 · Nobody pays the relayer or the deriver

The prototype's relayer posts for free. The plan's own conclusion is that
charging belongs on attestation and availability rather than on gas, because
derivation is optional and there is no protocol point at which payment can be
enforced.

**Reversing:** a fee market for the batch lane (priority fees to the relayer's
address through the L1 fee mechanism) is additive.

#### Review — how this plays out in practice

**A relayer's costs.** Plasma, which on mainnet means fused QSR: locked, not
spent, and returned when unfused. A 16 KB batch block needs about 1.1 M plasma;
the fleet benchmark's saturated relayer landed 61 such blocks in a burst. So the
cost is capital (QSR fused per relayer account, eight accounts) plus a machine,
not a per-transaction bill — **until dynamic plasma**, when congestion makes
plasma per byte cost more.

**Who runs one, then.** Four likely shapes, from prior art:

1. **A public good**, the way community members run Zenon's public RPC nodes
   today. Works while volume is low and someone cares.
2. **Wallets and apps run their own**, the way MetaMask's default RPC is run by
   its maker. The relayer URL is just the RPC a wallet points at.
3. **Paid by the protocol**, as Taiko pays the L1 proposer the L2 priority fees.
   Here one L2 block mixes many relayers' batches, so the block's single coinbase
   cannot name them; it needs a per-batch fee split in derivation — a consensus
   change. **This is the decided answer** (D9, "the relayed lane's fee").

**ERC-4337 is not part of the plan (decided 2026-09-16).** It was the obvious
fourth shape — a relayer that is also a bundler is paid in-band by the user
operation, with no protocol change — and it was measured against the alternative
and dropped, because on this chain it is the more expensive way to do the same
thing:

| Per relayed transfer | Relayer + fee split (measured) | As a UserOperation (estimated) |
|---|---:|---:|
| L2 gas | 21,000 | ~60,000–70,000 |
| NoM bytes, compressed | 89 | ~150–200 |
| A new account's first transaction | nothing extra | 150k–400k gas to deploy it |

The gas is EntryPoint's validation, prefund and refund accounting. **The bytes
matter more**: NoM bytes in flight are what bound this rollup — 195 tx/s on v1
and 25 tx/s on v2, neither of them gas — so a larger payload is a direct cut in
throughput and, under dynamic plasma, more plasma per transaction. It also does
not remove the relayer (a bundler is one), does not solve bootstrapping (the
account still needs ZNN in the EntryPoint prefund unless a paymaster pays), and
MetaMask does not send user operations for ordinary accounts — EIP-7702, which
Isthmus already gives us, is the cheaper route to batching and sponsorship.

The v0.7 EntryPoint and SenderCreator predeploys stay in genesis, unused:
removing them is a genesis change for no gain, and leaving them costs nothing.
What 4337 was going to be reached for — **an account that verifies a signature
scheme the EVM does not** — is an ERC-1271 question, not an EntryPoint one, and
that is where D6 and D28 need it.

**What happens when a relayer misbehaves.** It can refuse a transaction or go
down. It cannot reorder a sender's nonces or steal anything. The user's recourse
is another relayer, or **posting the batch themselves**: any NoM account may send a
`0x02` batch to the inbox. That property — nobody's permission is needed to get
in — is the rollup's censorship resistance, and it is D28's strongest argument.

**Spam.** Posting is free to the user and costs the relayer plasma, so a relayer
must reject a transaction that cannot pay its L2 gas before posting it. The
current relayer does that pre-check; the livelock defect in `docs/BENCHMARK.md` is
the one case where it re-posts transactions that will never pay.

**The deriver** needs no payment: whoever needs the ZVM's state runs one — the
committee's signers (D15 requires it), exchanges, explorers, app backends — the
way anyone who needs Ethereum's state runs a node.

### D21 · SQLite (modernc, pure Go) for the store and the index

No cgo, so the binary cross-compiles and the image stays small. The index is
disposable: it is rebuilt from op-geth and the derivation store.

**Reversing:** swap the driver; nothing in consensus touches it.

#### Review — options that scale

There are two stores here, and they should not get the same answer.

- **The derivation store** (chunks and message records, 342 B per transfer on the
  follower) is small, written by one process, and read by that process. SQLite
  is right for it, and for the bridge and signer journals.
- **The explorer index** is the problem: **1,121 B per transfer, 5,224 B per swap**,
  five times the NoM ledger. After the fleet run it held 612,158 transactions in
  919 MB. Its node container wrote **~28 KB per transaction** (WAL and
  checkpoints, 25× what it kept), and four endpoints slowed 5–15× as data grew,
  because they aggregate per request.

What that growth means:

| Sustained load | Transfers only (1.1 KB) | Application mix (~3 KB) |
|---|---:|---:|
| 1 tx/s | 35 GB/year | 95 GB/year |
| 10 tx/s | 350 GB/year | 950 GB/year |
| 195 tx/s (the measured ceiling) | 18.9 GB/day | — |

**Prior art:**

| System | Index store | Why |
|---|---|---|
| Blockscout | PostgreSQL | concurrent writers and readers, read replicas, partitioning; the most-deployed open explorer |
| The Graph, Ponder | PostgreSQL | the same, for application indexes |
| Dune, Goldsky, Allium | ClickHouse (or similar columnar) | analytics over billions of rows; 5–10× compression on logs and transfers; poor at updating a row |
| Otterscan | none — reads the node's own indexes (Erigon, reth) | the smallest footprint, and only what the node exposes |

**Options for this index, with the devnet's 919 MB in mind:**

1. **Stay on SQLite, fix the schema.** Precompute candles and collection
   aggregates instead of computing them per request; stop storing both
   `addr_txs` and `tx_origin` rows for the same fact; store verification input
   once. Probably halves the bytes. Keeps a single writer, no replicas, and
   every API read competing with checkpoints on one disk. Adequate into the
   hundreds of GB on one machine, not beyond.
2. **PostgreSQL for the index.** Table partitioning by block range, so old
   ranges can move to cheaper disk or be dropped and rebuilt; many readers
   without blocking the writer; read replicas for the API; mature backup. Costs
   a service in compose and a port of the SQL — SQLite-specific pieces are the
   upserts' syntax and `dbstat`. The pure-Go `pgx` driver keeps the no-cgo property.
3. **PostgreSQL plus ClickHouse** for logs, transfers and candles only, if
   analytics become the load.
4. **An op-reth Execution Extension (D2)** feeding option 2, so indexing follows
   the node's own commit and revert notifications instead of polling.

**Recommendation:** option 2 for the index, keep SQLite for the derivation store
and journals, and do option 1's aggregate precomputation regardless — it is what
fixes the slow endpoints, and it carries over. Migrating now is cheap: the index is
disposable and rebuilds from op-geth.

**Decided 2026-09-15: option 1 now, PostgreSQL as a future optimization.** Do the
schema work — precomputed candles and collection aggregates, one copy of each
fact, verification input stored once (done, D33) — and keep SQLite until the
volume argues otherwise. op-reth does not change this: its Execution Extensions
change how the indexer is *fed* (committed and reverted blocks pushed to it,
instead of polling) not where the rows live, so the same schema work is what
either arrangement needs.

### D22 · The UI transcribes the Zenon design tokens instead of importing nom-ui

`projects/amm` does the same, for the same reason: one binary, nothing fetched
at runtime, no GitHub dependency in the build. The cost is that the component
set is hand-written and small.

**Reversing:** adopting `nom-ui` is a UI-only change.

#### Review — what we need from nom-ui, and can we rip it

**It is already ripped.** `web/ui/package.json` has no `nom-ui` dependency; nothing
imports it, and the build fetches nothing from GitHub. What the ZVM took was
*values*, copied by hand into `web/ui/src/style.css`:

- the colour tokens (the green/blue/pink tri-colour, semantic success, warning and
  destructive colours, light and dark);
- the two fonts, Space Grotesk and JetBrains Mono, now from `@fontsource` packages;
- the radius and shadow scales, the plasma gradient, and the "ledger label" style.

Nothing else — no components, no composables. The remaining dependency is
**visual consistency** with the other Zenon surfaces, not code. To keep that
honest, record in `style.css` which `nom-ui` commit the values were copied from,
so a later token change upstream is a diff someone can choose to take.

---

## The application layer

Added 2026-09-15: dollar prices, trading charts, a farm and an NFT marketplace.

### D23 · Dollar prices come from DexScreener, and only through the node

ZNN and QSR have no dollar price on this chain; the bridged **wZNN** and **wQSR**
have one on Ethereum, which is what the Zenon community reads. The node polls
DexScreener once a minute and serves the answer at `/api/prices`; the page never
talks to DexScreener. That keeps the property every surface in this tree has —
the browser fetches nothing from anyone but its own node — and it means one
request a minute rather than one per visitor.

**The pair the operator names may not be there.** Asked for both wZNN/WETH
(`0xdac866…`) and wQSR/wZNN (`0xe6c614…`) in one request on 2026-09-15,
DexScreener returned only the second. So ZNN is priced from the **quote side** of
the wQSR/wZNN pool — `priceUsd ÷ priceNative` — and the answer says which side it
came from. Selection is by pool depth, so if DexScreener starts returning the
deeper wZNN/WETH pool, ZNN switches to it with no change here.

**Reversing:** another source implements `internal/prices`. Nothing else knows
where a dollar comes from.

### D24 · Every other token is priced from this chain's own pools

A ZVM token's price is derived from a pool against a token that already has one,
starting from wrapped ZNN and the QSR predeploy — the method Uniswap's own v2
subgraph uses (`findEthPerToken`: a whitelist of anchors, and a minimum
liquidity below which a pool may not price anything). Ours differs in two ways:
a derived token may price the next one, up to three hops, and the threshold is in
dollars on the priced side (default $25) rather than in ETH.

Only pools from the deployment's own factory may price a token. Anyone can
create a pool at any price; a price that anyone can set for the cost of a pool is
not a price.

**Reversing:** `-min-liquidity-usd` and the hop limit are parameters.

### D25 · A chart draws the pool's price after each trade, not the trade's own price

Every Swap, Mint and Burn is stored with the reserves the pair announced in the
Sync immediately before it, and the candle is built from that ratio. It is the
series Uniswap's own analytics draw. DexScreener instead charts each trade's
execution price, which includes that trade's impact and fee — so a large trade
prints a candle the pool was never at afterwards.

Volume is the swapped amounts, which is the same either way. Both are kept in
`dex_events`, so the other convention is a query away rather than a re-index.

**Reversing:** a different SQL aggregation over the same rows.

### D26 · A dollar candle uses the rate of its own time

Every price the node reads is kept (`price_samples`), and a candle in dollars is
converted at the sample at or before it. Candles from before this node's first
sample are converted at that first sample, and the page says so under the chart.
Converting a week of history at today's rate would draw a chart of today's price
with last week's shape.

**Reversing:** none needed; the samples are additive.

### D27 · The farm pays out a funded budget and can mint nothing

`ZVMFarm` is SushiSwap's **MiniChefV2** accounting — the farm Sushi runs where it
cannot mint — with PancakeSwap SmartChef's budget rule: emission stops at
`endTime`, and funding moves `endTime`. Nothing on the ZVM can be minted from
nothing, so a farm that promised more than it holds would fail its last harvests
instead.

Two departures from MiniChef, both deliberate: the reward-per-share scale is
1e36 rather than 1e12, because an 8-decimal reward (QSR) over an 18-decimal LP
supply rounds to nothing at 1e12; and reward debt rounds **up** on deposit and
down on withdrawal, so every rounding goes against the position and the sum of
all harvests cannot exceed what was emitted.

**Reversing:** a different farm is a different contract; nothing else depends on it.

### D28 · Marketplace orders become EIP-712 signatures, as on OpenSea and Blur

**Changed 2026-09-15. Decided, not built.** The original decision follows, then
the review.

*Original:* OpenSea's Seaport, Blur and LooksRare all sign orders off chain and
settle them on chain. That needs an EIP-712 signature, and **a Zenon account on the
native lane has none to give**: its only signature is the account block itself.
An order that is a transaction can be sent from either lane, and leaves no order
book for a server to hold — every node derives the same market from the same
logs. Zora's V3 Asks and Offers modules made the same trade for different
reasons. Listings do not escrow the token; offers escrow their ZNN.

#### Review — should the native lane exist at all? And the marketplace

**The case for keeping the native lane.** It is worth separating two things the
name covers:

- **Anyone with a NoM account can post to the inbox.** That includes posting a
  `0x02` batch of ordinary EVM transactions without any relayer. This is the
  rollup's **forced-inclusion path**, and it is the one property every serious
  rollup keeps: the OP Stack's deposits through `OptimismPortal`, Arbitrum's
  delayed inbox, and L2BEAT's Stage 1 requirement all exist because a sequencer
  or batcher can censor. The ZVM has no sequencer, which makes relayers the
  only chokepoint — and this path is the only thing that makes them optional.
  **This must stay whatever happens to the `0x01` message.**
- **The `0x01` native call**, where a Zenon key is `msg.sender` with no EVM
  signature. What only it provides:
  1. **Deposit-and-call in one NoM block** (SPEC §7): send ZNN to the vault and
     have it arrive already swapped, staked or bid. A MetaMask user's first
     action needs ZNN on the ZVM for gas; a Zenon user's does not.
  2. **Zenon accounts as actors on the ZVM** — including the vault committee, an
     exchange's hot wallet, a FROST quorum governing a ZVM contract, a pillar
     operator. None of them has a secp256k1 key, and a committee cannot
     cheaply get one.
  3. **Gas bought with plasma, or with D9's burn**, instead of bridged ZNN.

  What it costs: 100 transactions a momentum at most, no EIP-712, a gas budget
  set by plasma, and a wallet (Syrius) being set aside (D31). The code is one
  message type, already specified and pinned by vectors.

**Recommendation:** keep `0x01` in the protocol and stop building user-facing
features around it. Its users are machines and committees, not people clicking
"list".

**The marketplace decision: EIP-712 signed orders, OpenSea- and Blur-shaped.**

| | OpenSea (Seaport) | Blur | Here |
|---|---|---|---|
| Listing | EIP-712 order, signed off chain, no gas | EIP-712; **one signature over a Merkle root lists many items** | EIP-712, with Blur's bulk signing |
| Settlement | taker submits the order to Seaport | taker submits to the exchange contract | taker's transaction to the marketplace contract |
| Cancel | on-chain `cancel`, or bump a per-account counter to cancel all | on-chain nonce bump | the same: a counter bump cancels everything |
| Order book | OpenSea's API | Blur's API | the node's API, with every order re-checked against chain state before it is shown |
| Wallets | MetaMask, Rabby, others | same | MetaMask and Rabby; **Phantom only if it signs typed data for chain 7340470** — Phantom's EVM support covers the networks it lists, so this needs a live check before it is promised |

Consequences to carry into the build:

- **An order book now lives off chain.** D28's original property — every node
  derives the same market from logs — is lost for unfilled orders. Fills and
  cancels stay on chain. Serving orders from every node (gossiped, or posted to
  NoM as data) can restore it later.
- **Native-lane accounts can no longer list.** They can still *buy*, because a
  fill is a transaction.
- The existing on-chain listing contract, its indexer tables and
  `npm run verify:apps` are replaced, not extended.

#### Built 2026-09-16: `ZVMExchange`, and two ways in besides a signature

`contracts/apps/ZVMExchange.sol` replaces `ZVMMarketplace`. The order is Blur's
shape — one flat struct, a per-trader nonce whose bump cancels everything — and
the signing is Blur's too: one EIP-712 signature over the Merkle root of many
orders, each filled with its own proof, so listing fifty items is one wallet
prompt. `internal/orderbook` hashes orders exactly as the contract does, and
both are checked against the compiled bytecode in an in-process EVM; the
browser's third implementation is pinned by zvm-core's `vectors/vectors.json`.

Two escape hatches, both Seaport's:

- **ERC-1271.** A contract wallet signs through `SignatureChecker`, so an
  account abstraction that verifies something the EVM does not can list here.
  This is where an Ed25519-verifying account would plug in, if listing from a
  Zenon key ever matters enough to pay for it.
- **`approveOrders`.** An order can be put on chain by its trader instead of
  signed, which is what Seaport's `validate` is for. That is the answer for an
  account with no secp256k1 key — including a Zenon account on the native lane,
  whose account block *is* the authorisation — and it costs a transaction
  rather than a signature. The event carries the whole order, so the order book
  picks it up from the log with nobody submitting anything.

What changed from the plan: **bids are not escrowed.** Nothing holds a bidder's
ZNN; a bid is paid in wrapped ZNN with an allowance, as OpenSea's offers are
paid in WETH. That is why the node re-checks a bid's balance and allowance
before showing it — an unfunded bid is hidden rather than offered — and it is
the one place this market is weaker than the escrowed version it replaces: an
accepted bid can fail where before it could not. It buys the thing bids are
for: a bid on every item in a collection costs nothing to leave standing.

### D29 · A collection is an EIP-1167 clone

The factory deploys one `ZVMCollection` and every collection after it is a
45-byte proxy initialised in the same transaction — the pattern thirdweb,
Highlight and Zora's drop creator use to make creation cheap. Here the reason is
sharper: a native call's gas is the plasma its NoM block paid times 16, and a
full ERC-721 deployment does not fit that budget, while a clone and its
initialisation do. Measured live: **330,636 gas**, from a Zenon account, through
the inbox.

**Reversing:** a factory that deploys the full contract would work for relayed
wallets and lock native ones out.

### D30 · Media is content-addressed, and the CDN is a cache

Every image and metadata document is stored under the SHA-256 of its bytes and
served immutable. A URL can then never mean anything else, a cache may keep it
forever, and anyone who has the bytes can re-host them and be checked against the
name — which is what makes a hostname in front of the store a cache rather than
an authority. IPFS and Arweave get this from content addressing; OpenSea's
`i.seadn.io` is the cache half without it.

The canonical URL is `<media-url>/<sha256>.<ext>` (`-media-url`; on the first deployment it was
`https://nft.zenon.foo`), and
that is what goes on chain. Until that hostname is actually in front of a node,
`-media-cdn=false` has pages load the same objects from the node itself, so the
art shows either way. `docs/CDN.md` is the deployment.

**Reversing:** the store is one interface; the hostname is a DNS record.

### D31 · Uploading takes a wallet signature, not an account

The node's disk and the operator's domain are what an upload spends, so an upload
is signed: `personal_sign` from an EVM wallet, or Syrius's own `signMessage` —
Ed25519 over the same text, answered with the public key that checks it. The
signature buys a session token good for a day, not one signature per file. No
password, no account, nothing published, nothing spent.

The message is never 32 bytes, which is the one length Syrius refuses to sign,
because that is the size of an account block hash.

**Reversing:** none sensibly. An open upload endpoint on a public hostname is a
free file host.

#### Review — deprecate Syrius support for now

**Decided. Not built.** MetaMask-compatible wallets become the only wallets the
UI supports, for transactions and for upload sessions. Applied across the
decisions:

- **Uploads** keep the signature rule and drop the Syrius branch. The message
  should become Sign-In with Ethereum (EIP-4361), the format wallets already
  render as a login rather than as an opaque string.
- **The native lane stays in the protocol** (D28) and in the CLI; only the UI
  paths go: Syrius connect, native-lane buttons on contract tabs, native listing.
- `npm run verify:syrius` and the Syrius branch of `verify:apps` are retired.
- D29's clone factory keeps its reason: it also makes collections cheap for EVM
  wallets.

Reversing is cheap while the code stays in history: the extension integration
works on devnet today.

#### Built 2026-09-16

`core/syrius.ts` and the lane switch are gone, and with them every native-lane
button: connect, swap, pool, farm, contract write, deposit, create, list and
accept. What is left is one wallet, so pages that used to ask "which lane?" now
ask nothing.

- **Uploads are Sign-In with Ethereum** (EIP-4361), which wallets render as a
  sign-in rather than as an opaque string, and the message is bound to the
  node's own host — a page elsewhere cannot collect a signature that works
  here. The Ed25519 branch is gone from `internal/media`.
- **The encoders stayed.** `core/messages.ts` still encodes a native call, a
  chunk and an assemble, because the browser is one of the three
  implementations the golden vectors bind together. What went is the wallet.
- **The bridge page explains a deposit instead of making one**: a deposit is an
  ordinary Zenon send, and any Zenon wallet can make it.
- `verify:syrius` is retired; `verify:swap`, `verify:apps` and
  `verify:contracts` drive the same flows through one wallet.

The native lane itself is untouched in the protocol and in the CLI, and the
market's `approveOrders` path (D28) is what a Zenon account now sells through.

### D32 · The application contracts are named by a file, and only those are trusted

`zvm apps setup` writes `bench/apps.json`; the node re-reads it when it changes,
the way it already does for the Uniswap deployment. The indexer trusts marketplace
and factory events **only** from the addresses that file names — anyone can emit a
`Listed` event, and an indexer that believed them would show orders nobody can
fill. NFT ownership is not gated this way: it comes from Transfer events, so every
ERC-721 on the chain appears, whoever made it.

**Reversing:** a registry contract could replace the file; the trust question is
the same either way.

#### Review — what this means, and mainnet

"Named by a file" is not about filenames. It means: **the list of contract
addresses the explorer believes is kept in a configuration file**
(`bench/apps.json`), rather than on chain.

Why a list is needed at all: an EVM event is just a log line a contract emits.
Any contract can emit an event called `Listed` with any contents. If the indexer
showed every `Listed` event on the chain, anyone could make fake listings appear
on the marketplace page. So it believes `Listed` only from the marketplace
address in the file.

Filenames cannot collide, and neither can contract addresses — they are derived
from the deployer's address and nonce, or from a CREATE2 salt. **The mainnet risk is
different: the list is local.** Two explorers configured with different files
would show two different marketplaces, and nothing on chain says which is the
real one.

**For mainnet**, make the list something every node derives:

- deploy the application contracts at **fixed addresses in genesis**, as the
  bridge and QSR predeploys already are (`0x4200…0100`, `0x4200…0101`); or
- deploy them with **CREATE2 from a known factory and salt**, so the addresses are
  a pure function of published values; or
- a **registry contract** at a predeploy address, owned by whoever governs it.

**Correction, 2026-09-16: the factory is already here.** This chain's genesis
carries **Arachnid's deterministic deployer at
`0x4e59b44847b379578588920ca78fbf26c0b4956c`** — the same address Foundry and
every other EVM chain use — and the Safe singleton factory beside it. So the
decision below needs no new predeploy: it is a change to how `zvm apps setup`
deploys, and a published salt per contract.

**Decided 2026-09-15: deterministic CREATE2 from a published factory and salt**,
with the addresses published in the chain's configuration. That is what the
industry does for *application* contracts — Safe, Uniswap and Foundry's
`forge create --create2` all deploy through a deterministic proxy so that an
address is a pure function of (factory, salt, init code) and anyone can
recompute it. Genesis predeploys are what OP uses for *system* contracts, which
these are not. The factory itself becomes a predeploy, so every ZVM chain has it
at the same address; `bench/apps.json` then caches addresses anyone can derive
rather than asserting ones only the operator knows. **Not built.**

### D33 · Sources are verified by recompiling standard JSON input, and matched the way Sourcify matches

**Prior art.** Etherscan, Sourcify and Blockscout all verify by recompiling what the
author submits and comparing bytecode, and all three accept solc's standard JSON
input, because Foundry, Hardhat and Remix all produce it. Sourcify's
distinction between an *exact* match (the metadata hash too, so the very files)
and a *partial* one (everything but the metadata) is the precise version of what
Etherscan labels loosely; Etherscan's "similar match" reuses a verification for
identical bytecode elsewhere.

**Here.** `internal/verify` recompiles and compares **runtime** code, with the bytes
the source does not determine — immutables, linked library addresses, a library's
own address — taken from the chain at the offsets solc reports. Exact and partial
are Sourcify's. When the node knows the deploying transaction, creation code is
compared too and the constructor arguments are what follows it. Identical runtime
code anywhere else shows the same sources, marked as borrowed. An EIP-1167 or
EIP-1967 proxy is called with its implementation's ABI, as Etherscan's "as proxy"
tabs do — the NFT collections are clones, so without this none of them could be
called.

Verified on the live chain, 2026-09-15: UniswapV2Factory and UniswapV2Pair from
their npm metadata with **solc 0.5.16** (999,999 runs, istanbul), and the
marketplace, collection implementation and farm with 0.8.30 — all exact, creation
code included.

**Reversing:** cheap. The table holds the input as submitted; a stricter rule re-runs
over it.

#### Review — is this currently a problem?

**Not for correctness.** 21 of the chain's 24 programs verify, the in-tree ones
exactly with creation code, and nothing has verified that should not have. Three
things become problems with scale, none of them today:

| | Today | When it bites |
|---|---|---|
| **The input is stored twice** (`verify_jobs` and `verified_contracts`), ~510 KB per contract | 31 contracts, 11.5 MB | ~10,000 verified contracts is ~5 GB of duplicate JSON. Fix: keep it once, reference it from the job |
| **One compile at a time** | 8 submitted together: p50 2.6 s, max 5.2 s | a deploy script verifying 100 contracts waits minutes; a public endpoint can be queued behind one large project. Fix: a small worker pool, with D34's isolation |
| **Borrowed sources** | correct and wanted for clones: every NFT collection shows its implementation's source | two unrelated contracts with identical bytecode show one author's comments and names. Etherscan's "similar match" has the same property; the label "borrowed" is what makes it honest |

### D34 · The node fetches solc releases, checked against soliditylang.org's list

**Prior art.** Sourcify and Blockscout's smart-contract-verifier download
compilers on demand from `binaries.soliditylang.org` and check each against the
SHA-256 in that host's `list.json`. Bundling compilers instead caps verification at
whatever versions the image shipped.

**Here.** The same: fetched on first use into `-solc-dir`, checked, kept. The page
still fetches nothing: the node does, as it does for prices. **Not every
linux-amd64 release is static**: 0.5.16, 0.6.6 and 0.8.30 are, but 0.8.12, 0.8.17 and
0.8.23 name glibc's `/lib64/ld-linux-x86-64.so.2` as their loader, and on alpine
`exec` then fails with "no such file" for a file that is there. The image installs
`gcompat`, which runs all three, and the node names the cause if it is missing.

The compiler runs on anyone's input, so it is fenced. Sources must be inline
(`urls` is refused), absolute imports and remappings to absolute paths are refused,
solc runs in an empty temporary directory with an empty environment and a
timeout, one job at a time, the model checker is removed, and a failure reports
solc's `message` fields but **never `formattedMessage`** — an excerpt of a file solc
had been tricked into opening, `/config/jwt.hex` say, would publish it.

**Reversing:** `-verify=false` turns it off; pointing `-solc-url` at a mirror keeps it
on without the public host.

#### Review — solc must be properly sandboxed

**Agreed. Decided, not built.** What is there is input filtering, not isolation: solc
runs as the node's own user, inside the node's container, and that container
mounts `/config/jwt.hex` and `/data`. The filtering stops solc being *asked* to read
those files. It does nothing if solc itself is made to misbehave — a compiler is a
large C++ program, and its input here is anyone's. What is within reach of that
process is serious:

- `/config/jwt.hex` authenticates the Engine API, which lets its holder hand
  op-geth arbitrary blocks — this node's view of the chain;
- `/data` holds the index, the media store and the upload session secret;
- the node's network position, next to op-geth, the relayer and the bridge.

A second, smaller gap: the SHA-256 check compares a binary against `list.json`
from the **same host** it was downloaded from, so it proves integrity in transit,
not that the host is honest.

**Prior art:** Blockscout runs verification as a separate service
(`smart-contract-verifier`) that the explorer calls over HTTP; that separation is
the isolation.

**To build:**

1. A **`verifier` container**: its own image, a non-root user, a read-only root
   filesystem, a size-limited scratch `tmpfs`, no volumes of the node's, CPU,
   memory and process limits, and network egress only to
   `binaries.soliditylang.org` (or none, with compilers pre-fetched). The node calls
   it over HTTP and keeps the Etherscan-shaped API (D35).
2. **Pin the hashes** of the compiler versions the chain has used in the binary, and
   check fetched compilers against Sourcify's or the solc GitHub release hashes as a
   second source.
3. With isolation in place, D33's worker pool becomes safe to add.

#### Built 2026-09-23: the pins, and bounds on what a submission can cost

**Item 2, for the versions this chain uses.** `verify/compilers.go` pins the
linux-amd64 SHA-256 of solc 0.5.16, 0.6.6 and 0.8.30 — the Uniswap V2 core and
periphery, and everything in `contracts/`. The values were read from two
independently hosted copies of the release list, binaries.soliditylang.org's
and the ethereum/solc-bin repository's on GitHub, and they agreed. A download
of a pinned version must match its pin as well as the host's list, and a pinned
binary already on disk is hashed once per process and removed if it does not
match. Other versions are still checked only against the host's own list.

**What a submission can cost is bounded.** Anyone can submit, and a garbage
source fails its compile in milliseconds. Before this, every input stayed in
`verify_inputs` forever, failed ones too, at up to 16 MB each. Now a finished
job drops its input unless a verified contract publishes it. Failed jobs are
kept for a day and verified ones for a week. Submissions are limited node-wide
(60 a minute by default), and the contract and compiler names have a length.

**Published sources are replaced only by something better,** Sourcify's rule:
an exact match may replace a partial one, and anything may replace a
verification once the code at the address has changed. A partial match leaves
comments, NatSpec and local names free, so letting one partial match replace
another let anyone rewrite what a verified contract's source said.

### D35 · Etherscan's API shape, so the tools work unchanged

**Prior art.** Blockscout and Routescan answer Etherscan's `module=contract` actions
for the same reason: `forge verify-contract --verifier etherscan` and hardhat-verify
speak nothing else.

**Here.** `/api` serves `verifysourcecode`, `checkverifystatus`, `getabi` and
`getsourcecode` with Etherscan's field names, its double-braced standard JSON and its
status strings. The farm was verified through it the way forge sends a request.

**Reversing:** none needed; the explorer's own `/api/verify` is the same service.

---

### D36 · An unfilled order lives in the node that was handed it

**Decided and built 2026-09-16.** A signed order is not chain state and cannot
be recovered from the chain, so it has to be kept somewhere. Three places were
possible: publish orders to NoM, gossip them between nodes, or keep them in
whichever node received them.

| | Cost | What it buys |
|---|---|---|
| Publish to NoM | NoM bytes for something free to produce — a listing nobody fills is spam a chain pays for | every node derives the same book |
| Gossip | a second network, and its abuse problem | one book, eventually |
| **Node-local** (chosen) | two nodes can show different books | nothing to spam; a listing costs nothing anywhere |

This is what every comparable market does: Seaport's orders live in OpenSea's
API, Blur's in Blur's, LooksRare's in its own, and aggregators read several.
Fills and cancellations are on chain everywhere, and that is the part that has
to be agreed.

What keeps it honest is that the book is never the authority on whether an
order is good: `checkOrder` is, and the node asks it — when the order arrives,
when the exchange names it in a log, and on a timer, because a listing also goes
stale when its token moves and that emits nothing the exchange sees.

**Consequences, stated rather than discovered later.** An order book is not
derived, so a node that loses its database loses the orders it held, while the
index it sits beside rebuilds itself from the chain. Orders are kept in a table
the index never drops, and the honest next step — if a chain ever wants one book
— is a feed between nodes, not a change of chain rules. That is additive: the
orders are already signed, so any node can check one it is handed.

### D37 · The withdrawal fee pays for the vault's plasma

**Decided and built 2026-09-16.** `docs/PLASMA.md` found three costs sitting on
the committee rather than on the accounts causing them, the worst being the
receive a deposit makes it publish. A floor bounded the damage; it paid for
nothing, because the bridge had no revenue.

It has one now: a withdrawal pays 25 basis points, at least 0.01 ZNN or 0.1
QSR. **Prior art is Zenon's own bridge**, which charges `FeePercentage` per
token pair on a wrap and accumulates it in the contract — the same shape,
including a floor so that a small transfer still pays for the work it causes.

Three choices inside it worth naming:

- **On the way out, not the way in.** A deposit is a derived fact that nobody
  signs; taking a cut of it would mean minting less than was sent, which makes
  the peg's arithmetic a rule rather than an identity. A withdrawal is a
  contract call, so it can simply pay less out.
- **The fee is kept by the vault, not sent anywhere.** It is the same asset the
  plasma is bought with, and it is already where it needs to be.
- **Spending it is derived.** The vault's fusion is a NoM block like any other
  vault movement, and it derives to `markFused`, so "what has been earned" and
  "what has been spent" are both ZVM state (spec §7). That is what lets a
  committee member check a fusion against the chain rather than against the
  coordinator.

**Reversing:** the fee is genesis state (`bridgeFee`, slots 10–12), so changing
it on a running chain is a re-genesis; setting it to zero leaves the mechanism
in place and unused.

### D38 · The fee market is the chain's own, not OP's defaults

**Decided and built 2026-09-16**, after the explorer put the base fee and NoM's
plasma cost on one axis and the base fee was visibly inert.

The ZVM inherited OP's Canyon parameters — `eip1559Denominator` 250,
`eip1559Elasticity` 6. A denominator is a **per-block** damping constant, and
OP chose 250 for two-second blocks. This chain derives one block per momentum,
about fifteen seconds, so in wall-clock terms the base fee moved roughly seven
times more slowly than OP intended and about forty times more slowly than
Ethereum: measured, **-0.4 % per block, a 62-minute time constant**, 3.7 hours
to fall from 0.031 gwei to the floor.

**Prior art.** Ethereum uses denominator 8 with 12-second blocks (±12.5 % a
block). OP Bedrock used 50 with two-second blocks and Canyon raised it to 250 to
damp volatility. Nobody runs 250 at fifteen seconds; that combination is an
accident of inheritance, not a choice anyone made.

**Chosen, after three steps on the devnets: Ethereum's own ratios, scaled to
this chain's block time** — `feeMarket` reads 25/30 from momentum 1440, 8/30
from 1510, and **6/2 from 1560**. The first two chased responsiveness by
lowering the target; the third is the one to keep, and the reasoning is worth
recording because the first two were wrong in an instructive way.

**Ethereum's ratios are already the right shape for this chain; what was wrong
was only the block time.** Ethereum runs a 30 M limit with a 15 M target on a
12-second block: `limit / target = 2`, ±12.5 % a block, a 96-second time
constant, and — the part that matters — a target its blocks actually sit at.
Applying those ratios here:

| | Ethereum | ZVM |
|---|---:|---:|
| limit / target | 2 | 2 (elasticity **2**) |
| target | 15 M gas | **150 M gas** |
| max move per block | ±12.5 % | ±16.7 % |
| time constant | 96 s | **90 s** (denominator **6** at ~15 s blocks) |

Denominator 6 rather than Ethereum's 8 because a denominator is a *per-block*
constant: 8 on a 15-second block gives a 120-second constant, where 6 reproduces
Ethereum's wall-clock behaviour. That is the only place our numbers differ, and
it differs precisely to keep the ratio.

**The target lands at 119 % of chain 70's demonstrated capacity** — measured
399 tx/s at saturation against a 476 tx/s target — where Ethereum's target is
about 100 % of its own. So the fee sitting at the minimum is not a
misconfiguration: it is the correct reading of a chain running at 1.4 % of what
it can carry. Ethereum's fee is high because Ethereum is full.

The two earlier steps lowered the target instead (10 M, elasticity 30) to make
ordinary traffic move the fee. That works — measured, load lifted the fee off
the floor at +5.5 % to +10.7 % a block — but it is asking the fee to report
congestion that is not there, and it costs the symmetry: elasticity 30 gives a
362 % rise against a 12.5 % fall. The 300 M limit is untouched throughout, so
D10's headroom is intact.

**It is a schedule, not a setting.** `feeMarket` is a list of
`{momentum, denominator, elasticity}`. That is not tidiness: the parameters
change the base fee of every block after them, and the base fee is state, so a
node syncing from genesis has to reproduce each period with the parameters that
period actually used. Editing a single pair in place would silently make old
blocks unreproducible — the failure would surface much later, as one node
disagreeing about a chain everyone thought was settled.

**What it costs.** The two knobs are not independent: the maximum rise per block
is `(elasticity − 1) / denominator`, so at 8/30 a block can raise the base fee
**362 % — 4.6× — in a single block** while it falls 12.5 %. Note that Ethereum's
8 is paired with elasticity **2**, which makes its rise and fall symmetric at
12.5 %; pairing 8 with elasticity 30 deliberately keeps the low target and
accepts a 29:1 asymmetry. The symmetric way to reach a 10 M target would be to
lower the *gas limit*, which costs the throughput D10 buys. That asymmetry is the
deliberate part — a fee market should notice congestion faster than it forgets
it — but it has one consequence worth stating: the conventional **2× base fee**
cap that wallets use survives well under one block of a busy chain, where
Ethereum's survives six. `Config.SuggestedFeeCap` therefore sizes headroom from
the sharpest entry in the schedule (four blocks of the worst case, capped at
64×, which at 8/30 is about 2.7 blocks) and
every in-tree sender uses it. A cap is a ceiling rather than a price, so the
only cost of headroom is the balance a sender must hold. **A wallet that is not
ours still uses 2×**, and its transactions will fall out during a burst; that
is the open edge of this decision.

**Why it is cheap to reverse.** Holocene made the parameters a per-block
payload input, so this is a derivation change gated at a height — no
re-genesis, no execution-client change, and the genesis extraData still carries
250/6 as the historical record. Reversing it is another height.

### D39 · What a transaction costs, and who gets it

**Decided and built 2026-09-16**, after the cumulative fee chart made the
numbers impossible to keep ignoring.

**The measurement.** 192,497 transactions had produced, in total:

| | | |
|---|---:|---|
| burned by native calls | 39.35 ZNN + 0.156 QSR | ~$5.90 |
| base fee | 1.0366 ZNN | ~$0.16 |
| owed to relayers | 0.00315 ZNN | ~$0.0005 |
| burned on the ZVM | 0.00315 ZNN | ~$0.0005 |

Per transaction that is **$0.00000084 relayed against $0.00084 native — a 995×
gap for identical work**, and about $6 of fees for nearly two hundred thousand
transactions.

**The thing nobody had checked.** `0x4200…0019` and `0x4200…0011`, the base-fee
and sequencer vaults, **have no code**. They are not predeployed, so the base
fee is not "waiting to be swept" — it is at an address that can never send, and
it is gone. The chain was already burning its base fee, silently, unmirrored,
and without anyone having decided to. So of what the relayed lane pays, **99.7 %
is destroyed and 0.3 % reaches the account that bought the plasma**.

**Prior art is the shape the chain already had.** EIP-1559 burns the base fee
and pays the priority fee to whoever did the work of including the transaction.
That is exactly this design — base burned, tip to the publisher. Nothing was
architecturally wrong; the price was simply zero, because the base fee sat on
Jovian's floor of 0.001 gwei and the tip defaulted to the same.

**Decided.**

| lever | was | is | kind |
|---|---:|---:|---|
| minimum base fee | 0.001 gwei | **150 gwei** | consensus, scheduled at a height |
| relayer's minimum tip | none | **50 gwei** | relayer policy |
| every in-tree sender's tip | 0.001 gwei | **100 gwei** | client default |

A 21,000-gas transfer now costs **0.00525 ZNN ≈ $0.00079** — parity with a
native call's $0.00084, which closes the 995× gap, and inside the ordinary
cheap-L2 band. **20 % of it reaches the relayer**, 80 % is burned. The same
185,246 relayed transactions would have paid $146 rather than $0.16.

Three notes on the shape of it:

- **`minBaseFee` joins the fee-market schedule** rather than being a single
  config value, for the reason D38 gives: it changes the base fee of every later
  block, so each period has to keep the floor it was built with or old blocks
  stop being re-derivable.
- **The tip floor is relayer policy, not consensus.** Derivation admits whatever
  NoM included; refusing to *publish* an underpriced transaction is a mempool's
  privilege, and it needs no fork. A different relayer may choose differently,
  which is correct — that is what makes it policy.
- **The floor is half the default tip, not equal to it.** An effective tip is
  `min(tipCap, feeCap − baseFee)`, so it *shrinks* as the base fee rises under a
  transaction that is already signed. A floor at the default would reject
  transactions that were correctly priced when they were signed, at exactly the
  moment the chain got busy and it mattered that they land.
- **A fee cap is sized from the market the chain is heading into**, not the
  sharpest one ever scheduled. Letting a retired period govern meant a 64×
  headroom long after the market that justified it was gone.

**Measured after the fact.** At elasticity 2 the target sat at 119 % of chain
70's demonstrated capacity, so the fee reached only 157.9 gwei under saturation
— 5 % over its floor, a rounding error on a fixed price. Lowered to
`gasLimit / 4` = 75 M gas (~238 tx/s, about 60 % of capacity) it reaches
**273.9 gwei, 83 % over the floor**, on 11 of 60 blocks, and returns to 150 when
the load stops. The cost is a 3:1 rise-to-fall asymmetry instead of Ethereum's
1:1, which is the price of a target below a gas limit that was sized for burst
headroom rather than for demand.

**Still open, and now quantified.** The base fee is burned by accident rather
than by decision: it lands at a codeless address instead of `0x…dEaD`, so it is
never mirrored onto NoM and NoM's supply does not fall to match. Either
predeploy something at `0x4200…0019` that forwards to the splitter — a
re-genesis — or state the accidental burn as the intended one and mirror it.
Repricing makes that choice matter about three hundred thousand times more than
it did this morning.

## Still open — answered 2026-09-15

These were the plan's questions this prototype had answered with a default. The
operator has now made each call; the table records it and what it needs.

| Question | Was | **Decided** | Built? |
|---|---|---|---|
| Footprint cap | half the block for native calls, 300 M gas limit | **Dynamic, forecast-driven:** up to 90 % of NoM's capacity when the forecast is an empty momentum, down to 25 % or best effort when NoM is busy | no |
| Order inside a momentum | NoM's address sort | NoM's committed order, which is the same under v1 and dynamic plasma (D13); the relayer must price for v2 (D18) | derivation yes; relayer pricing no |
| Gas per plasma | 16 | zenoglyphs' burned ZNN/QSR fee, with plasma × 16 as the ceiling (D9) | **yes** — off until a chain sets `fee.fromHeight` |
| Finality depth | 6 momentums | **6** for the `finalized` label; payouts wait **30** on testnet and mainnet (D17) | yes: `-payout-depth` on each signer |
| What ZNN is inside | committee-bridged, redeemable 1:1 | committee-bridged (D14, D15); HTLC swaps as the trustless alternative; the Zenon bridge or a native mainnet contract later | committee yes |
| MetaMask in v1 | yes, through relayers | **the default for every transaction**; Syrius support dropped for now (D31) | no |
| Gas for a native call that arrives in chunks | only the assemble block's plasma counts, ~400,000 gas | **the whole payload's plasma** — every chunk plus the assemble — with each chunk counted once | no |

Notes on the three that need more than a sentence:

**The footprint cap is two different things, and only one is consensus.**

1. *How much of NoM the ZVM's relayers use* is **relayer policy**. Derivation cannot
   refuse a NoM block that has already been included — ignoring it would only waste
   the user's plasma — so this cap lives in the relayer, where it is cheap to change.
   The forecast has to come from what is committed: a moving average of non-ZVM
   account blocks (v1) or base plasma and `nextFusionPrice` (v2) over recent
   momentums, because a momentum's contents are unknown until it exists. Mainnet
   today averages **0.09 account blocks per momentum, 96 % empty**, so the forecast
   will sit at the 90 % end almost always; the 25 % floor is for a busy future. It
   matters more than it looks under v1, where a full momentum admits accounts in
   effectively random order (D13): a ZVM relayer flooding NoM would not get priority
   over other users, it would crowd them out at random.
2. *How much of a ZVM block native calls may use* is **consensus** (D10), with the
   recommendation there.

   *Built, and corrected on 2026-09-16.* The share was being converted into a
   **block count** on both rulesets, and on v2 that is the wrong denomination.
   A momentum there is capped at `MaxBasePlasmaInMomentum` (4,200,000 by
   default, and a hard ceiling in both the pillar's content selector and the
   verifier — not an EIP-1559-style soft target), and a send costs
   `21,000 + 68 × len(data)`. Costing every block as a full 16 KB one gives
   `4.2M ÷ 1,135,112 = 3` blocks per momentum, so a relayer whose batches are
   2 KB took three blocks where its share paid for twenty-three. Chain 71's
   `/status` read `blocksPerMomentum: 3` against chain 70's `89` — a 30×
   difference that was this policy being timid, and that would have been read
   off a v1-vs-v2 benchmark as dynamic plasma. The budget is now held in the
   resource the ruleset actually counts — blocks on v1, base plasma on v2 —
   charged by each block's real size after it is posted, and `/status` reports
   the unit. `blocksPerMomentum` survives as the full-block equivalent.

   The residual difference is the real one, and it is large: v1 admits up to
   **100 blocks of any size** per momentum, so a saturating rollup can put
   ~1.6 MB on NoM; v2 admits ~**3.7 full blocks**, or ~60 KB. Dynamic plasma
   prices data, and a rollup is data.

**What ZNN is inside.** The committee vault stays the canonical path. HTLC swaps are
already the trustless one: a user trades ZVM ZNN for NoM ZNN with a counterparty,
with no committee involved — `projects/amm` runs that shape on devnet. The Zenon
bridge becomes an option when its administrator adds the ZVM with `SetNetwork`,
which needs no spork; a native mainnet contract needs a go-zenon release and a spork, in
the window that opens about 2026-11-03.

**Gas for chunks: the rule that stops abuse.** Counting every chunk's plasma lets a
Zenon key deploy a contract larger than one NoM block, which today it cannot. The
abuse to prevent is counting the same plasma twice: publish chunks once, assemble
them twice, and get double the gas. So:

```
gas(assemble) = min( (plasma(assemble) + Σ plasma(chunk_i)) × gasPerPlasma , maxNativeGas , … )
```

with each chunk **consumed** by the first assemble that names it; a later assemble
naming a consumed chunk is invalid. Chunks already have to be the same author's and
younger than `chunkTtl` (D8), so the deriver's consumed-set is bounded by the TTL.
`maxNativeGas` still caps the result, and with D9 the burn term still applies. It is
a derivation change at an activation height.

---

## What the review changed

**Built and validated on 2026-09-15:**

- D17's size-tiered payout delay: `-payout-tiers ZNN:1000:90,QSR:100000:90` on
  each signer, checked by each member against its own node.
- D33: the standard JSON input is stored once, content-addressed
  (`verify_inputs`), instead of once per job *and* once per contract; two
  contracts compile at once instead of one.
- D34: **solc no longer runs in the node.** `zvm solcd` is a service that
  compiles and does nothing else, in a container with no volumes of the node's,
  a read-only root, an unprivileged user, a noexec scratch tmpfs and CPU, memory
  and process limits; the node reaches it over HTTP (`-verifier`). Proved end to
  end: a contract compiled inside the sandbox, deployed, and verified `exact`
  through the node — which now never executes a compiler on a stranger's input.
- The end-to-end suite runs against both rulesets (`npm run verify:*:v2`).

- D9's burned fee: `internal/rollup/fee.go`, `derive.PublishCall`, `/api/fee`,
  `zvm l2-genesis -fee-from`, spec §6 and new vectors. Off by default;
  rehearsed on chain 70 (see D9).
- **Both rulesets, permanently** (D4): a second NoM network with dynamic plasma
  active from genesis and a ZVM on it, `docker-compose.dp.yml`. Price-aware
  plasma payment, automatic replace-by-fee with the queue re-chained, and a
  relayer that sizes blocks to what its plasma affords (D18) — all measured
  against it, not predicted.
- D15's committee tooling: `internal/committee`, `zvm committee`, quorum mode in
  `zvm bridge`, `zvm l2-genesis -vault`, `docker-compose.committee.yml`,
  `docs/COMMITTEE.md`. Rehearsed on chain 70, results under D15.
- D19's measurement.

**Built 2026-09-16**, behind two activation heights so that nothing changes on a
chain until it sets one. `spec/SPEC.md` is version 0.2; zvm-core's `vectors/vectors.json`
version 2 pins every new encoding.

| Work | Decision | Where | Consensus? |
|---|---|---|---|
| `mine_boost`: a fifth call item that buys admission priority, not gas | D7, D9 | `derive/messages.go`, SPEC §5 | yes, from `forks.gas` |
| Elastic native share: ¼ floor held only to demand, ¾ ceiling | D10 | `derive/plan.go`, SPEC §6 | yes, from `forks.gas` |
| Chunk gas counts the whole payload, each chunk spent once | Still open | `derive/plan.go`, SPEC §6 | yes, from `forks.gas` |
| Relayed fee split, half the relayer and half burned, by **pull** | D9 | `contracts/src/ZVMFees.sol` | yes, from `forks.gas` |
| The burn mirrored on NoM (`ZVMB` → `markBurnedOnNoM`) | D9 | `derive/plan.go`, SPEC §7 | yes, from `forks.gas` |
| Brotli as flag `0x02`; **span batches** as `0x03` | D19 | `derive/span.go`, SPEC §5 | yes, from `forks.wire` |
| D12's rebuild rule made client-agnostic | D2, D12 | `l2/engine.go`, `node/deriver.go` | derivation behaviour |
| Karst genesis, Jovian minimum base fee, `maxTxGas` 16,777,216 | D3 | `zvm l2-genesis -evm karst` | new genesis |
| Forecast-driven footprint cap in the relayer, budgeted in blocks (v1) or base plasma (v2) | Still open | `relayer/forecast.go` | no |
| Precomputed aggregates in the index | D21 | `indexer/aggregates.go` | no |
| Deterministic CREATE2 application addresses | D32 | `apps/deterministic.go` | no |
| Committee resharing, keeping the group key | D15 | `committee/reshare.go` | no |
| Outflow limits in the signer policy | D17 | `committee/signer.go` | no |
| The halt analysis, and D8's plasma audit | D12, D8 | `docs/HALT.md`, `docs/PLASMA.md` | no |
| `gasPerSupply` measured on this engine: **1.15e14** | D9 | `rollup/fee.go` | yes, for the next genesis |
| `zvm fees address\|deploy\|status`, and the forks rehearsed live | D9 | `cmd/zvm/fees.go` | no |
| `zvm committee reshare`, run 2-of-3 → 3-of-5 | D15 | `cmd/zvm/reshare.go` | no |
| The op-reth compose override, validated but not run | D2 | `docker-compose.reth.yml` | no |

**Built 2026-09-16, after the migration.**

| Work | Decision | Where | Consensus? |
|---|---|---|---|
| **op-reth on both chains**, L2 re-derived from NoM, NoM ledgers kept | D2 | `docker-compose.reth.yml`, `docker-compose.reth-dp.yml` | no |
| `L2ToL1MessagePasser` predeploy, so both clients agree on a genesis | D2, D3 | `internal/l2/genesis.go`, `internal/l2/genesis_test.go` | **new genesis** |
| `tools/genesis-cmp`: committed genesis vs `BuildGenesis`, field by field | D2 | `tools/genesis-cmp` | no |
| A stalled node says so: `/healthz` 503, `stalled` on `/api/info` | D12 | `internal/node/deriver.go`, `internal/api/api.go` | no |
| The retry loop names a new failure once, loudly | D12 | `internal/node/deriver.go` | no |
| **The relayer re-bids on a timer**, so a wedged account can unstick itself | D18 | `internal/relayer/relayer.go` | no |
| A batch found again by height after a re-bid changed its hash | D18 | `internal/relayer/relayer.go` | no |
| A floor under deposits the vault receives promptly | D8 | `internal/bridge/bridge.go` | no |
| Client-neutral disk sizing: allocated for reth's sparse MDBX | D2, D21 | `internal/fleet/probe.go`, `report.go` | no |

Two things that were decided in the abstract and changed shape once they were
written down:

- **The fee splitter pays by pull, not by push.** `settle` is a derived
  transaction the chain depends on; if it pushed value, one relayer whose
  address reverts on receipt could stop every block. It writes `owed` and the
  relayer calls `withdraw` — from the native lane, since a NoM account has no
  EVM key, which is the usual case for a relayer.
- **The refusal rule stopped reading the client's prose.** D2 called
  re-establishing D12 against op-reth "the one derivation-level risk". The
  answer was to stop depending on the wording: compare the payload the engine
  returned against the list it was given, and whatever is missing was refused.
  The error text is still read, but only as a shortcut.

**Still not built** — in rough order of what blocks what.

| Work | Decision | Consensus? |
|---|---|---|
| Karst: a testnet genesis, and whether a zero DA scalar really disables Jovian's second metering rule | D3 | new genesis |
| Deferring mandatory gas instead of forcing it (`docs/HALT.md`, path 6) | D12 | yes |
| Re-measuring the **v2** matrix once the relayer re-bid is proven — the numbers on file are the wedge, not the client | D2, D18 | no |
| EIP-712 marketplace, Blur-style bulk listing | D28 | no |
| Remove Syrius from the UI; SIWE for uploads | D31 | no |
| An external review of `internal/committee` | D15 | no |
| Funding the vault's plasma from bridge revenue (the floor only bounds it) | D8 | no |
| PostgreSQL for the index, if volume ever argues for it | D21 | no |
| ERC-4337: **dropped**, not part of the plan | D20 | — |
