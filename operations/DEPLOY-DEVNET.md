# Deploying the ZVM onto the shared devnet (chain 69)

> These are the procedures for the chain 69 deployment behind devnet.zenon.foo,
> which is operated from a deployment tree on the devnet's host (the deprecated
> `zvm` repository, not published). The compose files, gateway and scripts they
> name are there; the services they run are built from the published
> repositories. A local devnet to work against is `zvm-fleet/devnet/`.

**Written 2026-09-17; revised 2026-09-18 after the procedure was reviewed and
partly rehearsed (G7, G8).** How to run a ZVM on the go-zenon devnet that is already
running on `127.0.0.1:35997`, with the vault held by a FROST quorum instead of a
single dev key.

This is a different shape from `docker-compose.yml`, which genesises the ZVM's
*own* NoM L1 (chain 70) and runs a pillar and an RPC node for it. Here the L1
already exists, is not ours, and keeps producing whether or not we are watching.
Everything below follows from that.

The reasoning behind the pieces is in [`DECISIONS.md`](../spec/DECISIONS.md) — D4 and D5
for why chain 70 exists at all, D15 for the committee.

**Two scripts do what steps 0–6 below describe.** The steps are the reasoning and
the fallback; the scripts are what to actually run, and they are safe to re-run:

```powershell
.\deploy-chain69.ps1             # steps 0-5: build, ceremony, funding, genesis, up
.\verify-chain69.ps1             # step 6, checks 1-9
.\verify-chain69.ps1 -Threshold  # and check 10, which stops signers
```

Read the steps before running them once. After that, the scripts are the honest
version: they refuse a chain that is not 69 on momentum v2, they top each account
up to its target rather than paying it twice, and they will not bring the stack up
on missing bind mounts.

---

## First, the word "trustless"

**A FROST committee is not a trustless bridge, and nothing in this document makes
it one.** It is worth being blunt about this before the deploy, not after.

Zenon cannot verify a proof of ZVM state, so there is no way for NoM to check
that a payout is owed. Value leaves the ZVM because a quorum of signers says it
may. What FROST changes is *who has to be wrong* for the vault to be looted:
with a single dev key it is one compromised host, with *t*-of-*n* it is *t*
colluding or compromised members. That is a real and worthwhile improvement — it
is the same custody model wZNN relies on today — but the honest name for it is a
**threshold-trust peg**, not a trustless one.

What the committee still cannot do, and this is the part that makes it tolerable:
it **cannot mint on the ZVM**. Deposits are derived facts that every node
computes from NoM; no signature is involved. The committee's power is confined to
refusing to pay, or paying the wrong party, out of the vault.

`README.md`'s "What it is not" section says the same thing and stays true.

---

## The environment, as measured 2026-09-17

Read off the live node, not assumed:

| | |
|---|---|
| L1 RPC | `http://127.0.0.1:35997` |
| Chain identifier | **69** |
| Momentum version | **2** — dynamic plasma is active |
| Frontier height | ~99,346 on 2026-09-17; **105,286** on 2026-09-18 — 5,940 momentums in about a day, a **14.5 s** average, which is the ragged cadence the old genesis timestamp predicts rather than the nominal 10 s |
| `nextFusionPrice` | 1000 (1×, uncontested right now) |
| L1 genesis timestamp | 1739923200 (2025-02-19) |
| Sporks | `az`, `htlc`, `libp2p` (h20), `dynamic-plasma` (h30) all active; `bridge-liq` off |

The chain carries unrelated traffic — other projects' tokens, glyph
collections, an airdrop unit — which is exactly what D5 warned about when it
chose a dedicated L1 for benchmarking. For *running* a ZVM this is fine; for
*measuring* one it is not, and no number taken here should go in
`docs/BENCHMARK.md`.

### Someone has already run a ZVM here — measured 2026-09-18

The dev mnemonic is public, and on a shared chain the service accounts are not
ours alone. Read off chain 69 while rehearsing `deploy-chain69.ps1`:

| Index | Role | Blocks | Fused | ZNN | Last published |
|---|---|---:|---:|---:|---:|
| 200 | old vault | 46 | 1,000 QSR | 4.72 | momentum 58,501 |
| 201 | faucet | 45 | 1,000 QSR | 4.69 | 58,502 |
| 202–209 | relayers | 34–49 | 1,000 QSR each | 0 | 58,501–58,502 |
| 210 | bench funder | 50 | 1,000 QSR | 4.65 | 58,502 |

A ZVM stack ran against chain 69 around momentum 58,500 — **about eight days and
47,500 momentums before this was written** — and was abandoned. Three things
follow, and all three are handled:

- **The accounts are not empty**, so funding has to top up rather than assume
  zero. `deploy-chain69.ps1` reads each account first and sends only the
  difference, which is also what makes it safe to re-run. The old 1,000 QSR
  fusions are why its table asks for 4,000 more, not 5,000.
- **Nothing is live on them.** Zero unreceived blocks, and 47,500 momentums of
  silence. The script refuses to fund an account that published within the last
  500 momentums, because two processes publishing from one NoM address fight over
  the same account chain and there is no polite outcome.
- **Index 200 is irrelevant to us.** The old stack's vault was a dev key; ours is
  a FROST group address that has never existed before. Its 4.72 ZNN stays where
  it is.

The chain-70 and chain-71 stacks running on this host use the same mnemonic
indices, and that is *not* a conflict: an account's chain is per-network, and a
block carries its chain id, so index 202 on chain 70 and index 202 on chain 69 are
independent ledgers.

**The funding account is dev mnemonic index 3**, `z1qp3yph55qgresyytz83anynr2f4z39x2z3ej3e`:

| | |
|---|---|
| ZNN | 37,765.98 |
| QSR | 363,815.40 |
| Already fused | 10,300 QSR → 21,630,000 plasma |

That last row is the surprise, and it drives several decisions below: **21.6 M
plasma on one account is more than v1 permits.** Under mainnet's rules
`MaxFusionUnitsPerAccount` is 5,000 (go-zenon `vm/constants/plasma.go:51`), which
caps an account at 10,500,000 plasma. Under dynamic plasma `dp/dp.go:24` raises
the same constant to 100,000,000 — effectively no ceiling. Chain 69 runs dynamic
plasma, so the per-account plasma limit that `HANDOFF.md` lists under "Things to
know" ("an account cannot hold more than 10.5 M plasma … relayer capacity is
bought with more accounts, not more QSR") **is a v1 fact and does not hold here**.

---

## What this needed, and why — all of it done

These were not risks to watch. They were things that did not work against this
target, and each is now fixed in the tree. Nothing has been deployed; the chain is
untouched.

G1–G6 are code, found on 2026-09-17. G7 and G8 are the procedure and the compose
file, found on 2026-09-18 by reviewing this document against the tree and running
the parts that could be run without writing to the chain.

### G1 · There was no command that funds an L1 account — added `zvm fund`

`zvm l1-genesis` writes balances and fusions straight into a genesis
(`cmd/zvm/l1genesis.go:160`). On a chain that is already running, that door is
shut, and nothing else in the tree sends plain ZNN/QSR to an arbitrary address:
`zvm deposit` only pays the vault, `zvm call` only reaches the inbox.

**Done:** `cmd/zvm/fund.go`.

```sh
zvm fund -l1 http://127.0.0.1:35997 -l1-chain-id 69 -from-index 3 \
  -to <z1… or index> -znn 8000 -qsr 60000 -fuse 5000
```

Two NoM properties shaped it, and the second one was missing from the first
draft of this plan:

- **Fusing for a beneficiary needs no signature from the beneficiary.** Plasma
  is credited by the embedded contract, so anyone can fuse for anyone. That is
  what makes a committee-held vault fundable at all — the quorum never has to
  sign a bootstrap transaction.
- **A transfer is not a credit until the recipient receives it.** A send sits in
  the recipient's unreceived list until they publish a receive. Funding an
  account therefore means publishing *its* receive too, which `zvm fund` does
  whenever `-to` is a dev mnemonic index and it holds the key. For the FROST
  vault it cannot and does not need to: the bridge receives what the vault is
  sent, as its ordinary job. The command says which case it took rather than
  leaving it to be discovered.

The fusion is published *before* the transfers, because an account with plasma
receives cheaply and one without computes proof of work per block.

It refuses, before signing anything: a node whose chain id is not the one
asserted, a funder that cannot cover the total, a fusion under NoM's ten-QSR
minimum or not in whole QSR, and `-to` equal to the funder. Verified against the
live node — the chain-id guard correctly refuses `-l1-chain-id 70` against
chain 69, and the shortfall message names what index 3 actually holds.

### G2 · The faucet's top-up amounts were hardcoded, and larger than the funder

`cmd/zvm/faucet.go:69-70` fixes `L2Floor` at 1,000 ZNN and `L2TopUp` at
**100,000 ZNN**, with no flags. `Faucet.KeepFunded` deposits that amount through
the vault whenever the faucet's ZVM balance falls under the floor
(`internal/faucet/faucet.go:169`). Index 3 holds 37,765 ZNN *in total*, so on
chain 69 the faucet would try to spend nearly three times the entire budget,
fail, `acct.Reset()`, and log a warning every fifteen seconds forever.

**Done:** six flags on `zvm faucet` — `-drip-znn`, `-drip-qsr`,
`-drip-fuse-qsr`, `-drip-l2-znn`, `-l2-floor`, `-l2-top-up`. **Every default is
the old constant**, so chains 70 and 71 behave exactly as before and nothing
needs re-reading there. `docker-compose.chain69.yml` sets the small profile: 10
ZNN / 100 QSR / 50 QSR fused per drip, floor 200 ZNN, top-up 2,000 ZNN.

### G3 · The bridge's fusion ceiling was a v1 constant

`internal/bridge/plasma.go:23` hardcodes `maxFusedToAccount = 5,000 QSR` with the
comment "no account usefully holds more than 5,000 fusion units". True on chain
70; false on chain 69, per the `dp/dp.go` constant above.

The consequence is not a loss of funds, it is a feature that silently never
runs. `PlanFunding` computes `Headroom = 5,000 − fusedToVault`, so once the
vault is fused at or above 5,000 QSR it reports **"the vault is already at NoM's
per-account plasma ceiling"** and D8's self-funding from withdrawal-fee revenue
stops before it starts. `CheckFuse` is the *signer* policy, so the committee
would also refuse a larger fusion proposed by the coordinator.

**Done:** the ceiling is now read from the ruleset in force rather than assumed.
`znn.MaxFusedAmountForAccount(momentum)` answers 5,000 QSR on a v1 momentum and
`DPMaxFusionUnitsPerAccount` on a v2 one, `PlanFunding` reads the frontier
momentum to pick, and `internal/bridge/plasma.go` no longer carries a constant
for it. A regression test pins both answers and the 10,300-QSR account that v1's
figure would have called over the limit.

One consequence worth stating, because an earlier draft of this file got it
wrong twice: **on chain 69 nothing is "under the 5,000 ceiling", because 5,000
is not the ceiling in force.** It is 100,000,000 QSR. So the vault's fusion is
not sized to leave room under a cap; it is sized for the plasma it needs. What
bounds D8's self-funding here is fee revenue, not headroom — see step 6's check 7, which
is the step that actually makes it fuse anything.

### G4 · The committee compose file ran op-geth, which cannot follow Karst

`docker-compose.committee.yml` starts `op-geth-c` on
`op-geth:v1.101702.2`. `HANDOFF.md` records that op-geth is retired and cannot
follow either chain since the Karst migration, and that this stack "needs
re-genesising before it runs again".

**Done:** `docker-compose.chain69.yml` is a new file rather than an override of
the old one — op-reth throughout, no L1 services, the committee folded into the
main stack instead of a parallel one.

`docker-compose.committee.yml` has since been re-genesised and repaired too,
because the chain-70 rehearsal is what validates the ceremony before chain 69
gets one (2026-09-18):

- `op-geth-c` is replaced by `reth-c` on `op-reth:latest`, with the `/dev/tcp`
  healthcheck the main compose file's reth services use — the op-reth image
  carries no `wget`, so the old probe would have failed even once the client
  was right.
- `committee/devnet/l2` was regenerated (`-evm karst -min-base-fee 1000000
  -inbox-tag zvmcttee`). The old genesis predated `mandatoryShareBp`, and every
  service refused to start against it.
- `-inbox-tag zvmcttee` gives the rehearsal its own inbox. It previously shared
  the default inbox with the main chain-70 ZVM, which is the same hazard
  `-inbox-tag zvm69` avoids on chain 69.
- `signer-4` and `signer-5` were added under a `committee5` profile for the
  resharing rehearsal, and the bridge's `-signers` list is now
  `${ZVM_COMMITTEE_SIGNERS:-…three…}`, because `NewQuorum` calls `/info` on
  every URL it is given and fails if one is unreachable.

### G5 · Ordering — the ceremony comes first

The vault address is an *input* to `zvm l2-genesis` (`-vault`), and with a
committee that address is the group key, which does not exist until the ceremony
finishes. So the ceremony runs before the genesis, not after. (An earlier draft
of this plan had them the other way round.)

### G6 · Smaller things — all handled in the compose file

- `-l1-explorer=http://localhost:46000` pointed at the `l1-explorer` profile of
  the *chain 70* stack, which does not serve chain 69. Dropped.
- `-volumes=…,nom-l1=$NOM_DATA_ROOT/nom-l1` walks the L1 node's data directory for
  `/ops`. Chain 69's volume is not ours; only `engine=` remains.
- Every `depends_on: l1-rpc` and the `l1-pillar`/`l1-rpc` services are gone, and
  each service that dials the L1 carries an `extra_hosts` entry for
  `host.docker.internal` so the same file works on Linux as on Docker Desktop.
- **Not fixed, because it is a budget and not a bug:** the native-lane benchmark
  wants 200 wallets at 1,000 ZNN each (`internal/devnet/devnet.go:80`). That is
  200,000 ZNN against a 37,765 ZNN budget, so the native-lane fleet benchmark
  stays out of scope on this chain unless the wallet count and float are cut by
  an order of magnitude.

### G7 · The procedure did not run — found by reviewing it, 2026-09-18

Everything above was about code. This is about the steps themselves, and each of
these was reproduced rather than reasoned about. Four of them stopped the deploy
outright:

- **`zvm` is not on PATH, and `zvm.exe` in this tree predates G1–G3.** It is
  dated 2026-09-16; its own usage list has no `fund` and no `reshare`. Steps 1
  and 2 called a bare `zvm`. **Fixed:** step 0 builds the binary and every step
  calls it by path.
- **`committee round1 -out round1/` fails: nothing creates `round1/`.**
  `committee.WriteJSON` → `writeNew` (`internal/committee/ceremony.go:351`) opens
  with `O_CREATE|O_EXCL` and no `MkdirAll`:
  `open round1\1.round1.json: The system cannot find the path specified.`
  Worse, the obvious retry fails too, because the sealed state was written before
  the output was: `writing p1/1.state: open p1/1.state: The file exists.`
  **Fixed:** step 1 creates both directories and starts from a clean slate.
- **Capturing the transcript fingerprint could abort the script.** `committee
  round2` without `-confirm` prints the fingerprint on stdout and **exits 1** —
  that status *is* the "not confirmed yet" signal. Wherever
  `$PSNativeCommandUseErrorActionPreference` is on, `$ErrorActionPreference =
  'Stop'` turns it into a terminating error, and because it fires on the
  assignment statement it kills the script *without* `$fp` ever being set. It is
  off on this host (PowerShell 7.6.6) and on in other configurations, which is
  the worst kind of difference to leave in a procedure. **Fixed:** that one call
  sets both preferences inside a function scope, so nothing else relaxes.
- **`zvm withdraw` has no default `-to`.** `cmd/zvm/withdraw.go:44` parses it
  unconditionally, so the verification's withdraw died on `address "": …` before
  touching anything. **Fixed.**

And four where the step ran but did not check what it claimed:

- **A successful `zvm deposit` does not prove the committee works.** The ZVM
  credits a deposit from the *send*, which every node derives independently
  (`docs/PLASMA.md` §2) — so the L2 receipt arrives whether or not the vault ever
  received anything. **Fixed:** step 6's check 5 reads the vault's own NoM account.
- **`bridge /status` does not report the signers.** `internal/bridge/bridge.go:513`
  has the vault, its balances, the fee counters and the funding plan, and nothing
  about the quorum; the signers' only endpoint is `/info`, which needs the bearer
  token and is not published to the host. **Fixed:** step 6's check 3 reads the
  coordinator's log line, which is written only after `NewQuorum` has had `/info`
  from every signer, and the signers now carry a healthcheck that presents the
  token from the file they authenticate against.
- **`-fund-plasma` could not fire, so the headroom it was given was untestable.**
  `fundableQSR()` is `feesQSR − fusedQSR`, QSR only, and `PlanFunding` floors to
  whole QSR against NoM's ten-QSR minimum. At 25 bps that needs ≥4,000 QSR of
  *QSR* withdrawals before one fusion is possible, and the old verification moved only ZNN.
  **Fixed:** step 6's check 7 deposits and withdraws QSR sized to earn 25 QSR of fee.
- **Two services were never exercised at all.** The verification used only `0x01`
  inbox messages — deposits and native withdrawals — so **the relayer and the
  faucet were started and never asked to do anything.** A batch (`0x02`) is the
  lane the relayer exists for, and the faucet's ZVM float arrives as a deposit
  *through the vault*, which makes `KeepFunded` a second independent thing the
  quorum has to sign. **Fixed:** step 6's checks 8 and 9.

One more thing this review got wrong on its first pass and is worth recording
because it is the kind of mistake that reads as reassurance: an earlier version of
the re-genesis note in *Things to know* said reusing the committee means "the ZNN
and QSR already in the vault still belong to the new rollup". They do not. A
rollup that restarts at a later momentum never derives the earlier deposit sends,
so every L2 balance is abandoned while its backing stays in the vault. The note now
says to drain the vault first.

### G8 · Three things in the compose file, same review

- **`-min-receive` was unset on the one chain where it matters.**
  `docs/PLASMA.md` §2 calls the vault's deposit receive "the only unpriced cost
  an outsider can run up on somebody else's account", and the floor is off by
  default. Chain 70 is private; chain 69 is shared and anyone can send the vault
  dust. **Done:** `-min-receive=ZNN:1,QSR:10`.
- **"Three signers, each with its own `-outflow-limit`" was not true.** All three
  shared one YAML anchor and therefore one identical limit — which behaves as a
  single limit with three copies. **Done:** the three commands are written out,
  with different limits and one different `-payout-depth`.
- **`-media-url=https://nft.zenon.foo` was copied from the chain-70 stack**,
  where cloudflared actually fronts that hostname (`docs/CDN.md`). Nothing fronts
  it here, so every chain-69 NFT would have minted a token URI pointing at
  another chain's node. **Done:** dropped, so token URIs are `/media/<hash>`.
  Note for anyone tempted to point it at this stack instead: `media.Wrap`
  (`internal/media/media.go:669`) gives the public hostname an objects-only
  handler, so `-media-url=http://localhost:49000` would take the explorer,
  the API and `/rpc` off the air.

### G9 · D8 fused QSR the ZVM could not see — found by deploying, 2026-09-19

The first real use of this stack found a funds-safety bug that no amount of reading
had. `zvm bridge -fund-plasma` fused the withdrawal fee twice, 25 QSR each time;
the vault's fused total on NoM went 5,000 → 5,050; and `ZVMBridge.fusedQSR()` read
**0**.

The mirror that records a fusion in ZVM state is gated on the gas fork
(`internal/derive/plan.go:346`, `KindFuse`), and this chain deliberately has no
fork scheduled (step 3). Neither `PlanFunding` nor `CheckFuse` checked that. So:

- `fundableQSR()` is `feesQSR − fusedQSR`, and `fusedQSR` could never rise, so the
  fundable figure never fell.
- Every `-fund-every` cycle would therefore plan the same fusion again. The only
  remaining bound was `Liquid` — the vault's QSR balance less pending payouts,
  which is **the QSR backing everybody's deposits**. It stopped here only because
  the test had withdrawn everything and left `Liquid` at zero.
- The next QSR deposit would have restarted it. The farm seeding deposits QSR, so
  this would have bitten within the hour.

**Fixed:** `bridge.FusionIsDerivable` gates funding on the gas fork being active
at the frontier, `PlanFunding` returns a reason instead of a plan, and `CheckFuse`
runs the same plan so every signer refuses independently. `/status` now says:

```
funding.reason: the gas fork is not active, so a fusion would not be derived
back into fusedQSR and D8 could not tell it had already happened
```

`internal/bridge/plasma_test.go` pins the five cases. **D8 self-funding therefore
does nothing on chain 69 and cannot**, because the gas fork is a re-genesis. That
is the honest position: the fee is still collected, it just accumulates instead of
buying plasma. The 50 QSR already fused stays fused, which is harmless — it is the
vault's own plasma either way.

Worth noting what the check that was *supposed* to cover this did wrong: it
asserted that the vault's fused total on NoM grew, which it did. Only the ZVM-side
number showed the problem. A check that reads one side of a mirror is not checking
the mirror.

### G10 · A Uniswap pair is permanent, so the token set is a one-way door

The swap page lists pairs from the indexer — `PairCreated` events off the factory
— not from the deployment file (`internal/api/markets.go`, `dexPairs`). `zvm bench
setup` deploys BNA and BNB and opens a BNA/BNB pool with 100 M of each, so the
first seeding of chain 69 put three pairs on the swap page that nothing short of a
re-genesis can remove, on a chain where a re-genesis abandons every balance.

**Fixed for the next chain, not this one:** `zvm bench setup -swap-only` deploys
WZNN, the factory and the router and stops — no BNA, no BNB, no ERC-721/1155, no
churn or disperse, no pair. `zvm dex seed -znn 0 -qsr-pool-znn 500` then opens
**only** ZNN/WQSR, and `apps setup` builds the farm's pool list from the pairs that
exist, so that one pool takes every reward. `deploy-chain69.ps1` does this by
default; `-FullBench` gets the old behaviour for a chain meant to be benchmarked,
where the scenarios need the full set.

Chain 69 keeps its BNA/BNB pools. Step 6's check 10 lists any pair that is not
ZNN/WQSR as a warning rather than a failure, for exactly that reason.

---

## Step 0 · Build, and check the L1 is the one you think it is

Everything below calls **`$zvm`**, one freshly built binary. Do not skip this:
the `zvm.exe` committed in this tree is dated 2026-09-16, which is before `fund`
existed, and it is not on `PATH` anyway.

```powershell
$ErrorActionPreference = 'Stop'
$repo = (Get-Location).Path              # C:\dev\zenon\zvm
$zvm  = Join-Path $repo "bin\zvm69.exe"  # a name nothing else in the tree uses
$L1   = "http://127.0.0.1:35997"

go build -o $zvm ./cmd/zvm

# Read the L1 straight off its RPC. `zvm balance` would be the obvious tool and
# is the wrong one here: it dials the ZVM too (cmd/zvm/tx.go:334) and fails
# until step 5 is up.
function Rpc($method, $params = @()) {
  $body = @{ jsonrpc = '2.0'; id = 1; method = $method; params = $params } |
    ConvertTo-Json -Depth 5 -Compress
  (Invoke-RestMethod -Uri $L1 -Method Post -ContentType 'application/json' -Body $body).result
}

$m = Rpc 'ledger.getFrontierMomentum'
"L1          chain $($m.chainIdentifier), momentum $($m.height), version $($m.version), fusion price $($m.nextFusionPrice)"
if ($m.chainIdentifier -ne 69) { throw "this node is chain $($m.chainIdentifier), not 69" }
if ($m.version -lt 2) { throw "momentum version $($m.version): dynamic plasma is off, so the G3 assumptions do not hold" }

$ZNN = 'zts1znnxxxxxxxxxxxxx9z4ulx'
$QSR = 'zts1qsrxxxxxxxxxxxxxmrhjll'
$funder = 'z1qp3yph55qgresyytz83anynr2f4z39x2z3ej3e'
$fi = Rpc 'ledger.getAccountInfoByAddress' @($funder)
$fpl = Rpc 'embedded.plasma.get' @($funder)
$has = { param($info, $zts) if ($info.balanceInfoMap.$zts) { [decimal]$info.balanceInfoMap.$zts.balance / 1e8 } else { 0 } }
"funder      $(& $has $fi $ZNN) ZNN, $(& $has $fi $QSR) QSR, " +
  "$([decimal]$fpl.qsrAmount / 1e8) QSR already fused -> $($fpl.maxPlasma) plasma"
```

Every command below re-asserts the chain id before it signs anything, so this is
belt and braces — but it costs nothing and it is the one check that tells you the
operator has not restarted chain 69 as something else since 2026-09-17.

**Stay in `$repo` for the whole procedure**, and keep this one PowerShell session:
`$zvm`, `$L1`, `$repo`, `$funder` and `Rpc` are used by every step, `$vault` comes
out of step 1, and the later steps pass relative paths like `l2-69/rollup.json`.
Step 1 is the only step that changes directory, and it changes back.

---

## Step 1 · The ceremony

A 2-of-3, all three members on this machine. Be clear about what that is worth:
**one machine holding all three shares is a rehearsal of the mechanism, not a
custody separation.** It proves the signing path, the policy and the payout
flow; it provides no protection against this host being compromised. A committee
that matters puts each share on a different machine run by a different operator,
each reading *its own* NoM and ZVM nodes — `docs/COMMITTEE.md` is emphatic that a
policy checked against the coordinator's nodes checks nothing.

The ceremony binds every proof to the chain id, so it must be **69** here.

Three things about the script below that are not obvious, and that an earlier
draft got wrong:

- **Every artifact is written with `O_CREATE|O_EXCL` and no `MkdirAll`**
  (`internal/committee/ceremony.go:351`). So the output directories must exist
  first, and **a half-finished ceremony cannot be re-run in place** — the sealed
  state is written before the round-1 package, so a retry trips over its own
  `1.state`. The script therefore starts from an empty `committee/chain69`
  and says so rather than leaving you to discover it.
- **Round 2 without `-confirm` exits non-zero on purpose.** The non-zero status
  *is* "the fingerprint has not been confirmed yet", and the fingerprint itself
  goes to stdout. So that one call has to tolerate its own exit code — otherwise,
  on any host where `$PSNativeCommandUseErrorActionPreference` is on, the script
  dies before `$fp` is assigned.
- **`Get-Random` is not a cryptographic RNG.** It stands in for passphrases and a
  bearer token that real members would choose themselves; it is a third
  devnet-only shortcut alongside the two in step 4, and it is the reason nothing
  here should ever hold value.

```powershell
$root = Join-Path $repo "committee\chain69"
if (Test-Path $root) {
  throw "$root exists. The ceremony writes every file exclusively and cannot resume; " +
        "move it aside and start clean."
}
# round1/ and round2/ included: the tool will not create them.
'', 'p1', 'p2', 'p3', 'round1', 'round2' | ForEach-Object {
  New-Item -ItemType Directory -Force -Path (Join-Path $root $_) | Out-Null
}
Push-Location $root

# Three members, threshold 2.
& $zvm committee new -l1-chain-id 69 -threshold 2 -names alice,bob,carol -out ceremony.json

# Each member's passphrase. On a real ceremony these are chosen by their owners on
# their own machines and never written here. -Encoding ascii keeps Windows
# PowerShell 5.1 from prefixing a UTF-8 BOM, which would silently become part of
# the passphrase.
1..3 | ForEach-Object {
  -join ((1..32) | ForEach-Object { '{0:x}' -f (Get-Random -Max 16) }) |
    Set-Content -NoNewline -Encoding ascii "p$_/pass"
}

# Round 1: commitments, a proof of knowledge, and an X25519 key to receive at.
1..3 | ForEach-Object {
  & $zvm committee round1 -ceremony ceremony.json -id $_ -passphrase-file "p$_/pass" `
    -state "p$_/$_.state" -out round1/
}

# Round 2, pass one: no -confirm, so it verifies, prints the transcript
# fingerprint and exits 1 — that exit code is the "not confirmed yet" signal, so
# it is tolerated here and nowhere else.
#
# On a real ceremony every member runs this against the round-1 files *they*
# received and reads the fingerprint aloud to every other member; that comparison
# is the whole defence against a courier who swaps a file. Here all three read one
# directory on one disk, so the comparison is a rehearsal of a rehearsal: it
# proves the tooling agrees with itself and nothing about couriers.
# Both preferences are set inside the function, so they are restored on return and
# the rest of the script goes on failing fast.
function Get-Fingerprint {
  $ErrorActionPreference = 'Continue'
  $PSNativeCommandUseErrorActionPreference = $false
  $out = & $zvm committee round2 -ceremony ceremony.json -id 1 -passphrase-file p1/pass `
           -state p1/1.state -round1 round1/ 2>&1 | Out-String
  $m = ([regex]'transcript fingerprint: ([-0-9a-f]+)').Match($out)
  if (-not $m.Success) { throw "no transcript fingerprint in:`n$out" }
  $m.Groups[1].Value
}
$fp = Get-Fingerprint
Write-Host "transcript fingerprint: $fp"

# Round 2, pass two: send the private values. Each member recomputes the
# fingerprint from its own round-1 files and refuses if -confirm disagrees, so
# passing one value to all three is a check, not a shortcut.
1..3 | ForEach-Object {
  & $zvm committee round2 -ceremony ceremony.json -id $_ -passphrase-file "p$_/pass" `
    -state "p$_/$_.state" -round1 round1/ -out round2/ -confirm $fp
}

# Finish: each member decrypts what was addressed to them, checks it against the
# sender's commitments, and writes a sealed share plus the public group file.
1..3 | ForEach-Object {
  & $zvm committee finish -ceremony ceremony.json -id $_ -passphrase-file "p$_/pass" `
    -state "p$_/$_.state" -round1 round1/ -round2 round2/ -confirm $fp `
    -share "p$_/share" -group "p$_/group.json"
}

# Every group file must be byte-identical. If they are not, stop.
$hashes = 1..3 | ForEach-Object { (Get-FileHash "p$_/group.json" -Algorithm SHA256).Hash }
if (($hashes | Select-Object -Unique).Count -ne 1) { throw "group files differ — stop" }
Copy-Item p1/group.json group.json

# Recompute the vault from public data alone.
& $zvm committee audit -ceremony ceremony.json -round1 round1/ -group group.json
$vault = (Get-Content group.json | ConvertFrom-Json).address
if (-not $vault.StartsWith('z1')) { throw "no vault address in group.json" }
Write-Host "vault $vault"

# The coordinator authenticates to the signers with a bearer token. The signer
# refuses anything under 32 characters (cmd/zvm/committee.go:401).
-join ((1..48) | ForEach-Object { '{0:x}' -f (Get-Random -Max 16) }) |
  Set-Content -NoNewline -Encoding ascii token

# The state files have done their job; the shares replace them.
1..3 | ForEach-Object { Remove-Item "p$_/$_.state" }
Pop-Location
```

Expect `committee audit` to end with `ok: z1… is the 2-of-3 vault of ceremony …`.
Carry `$vault` forward — steps 2 and 3 both need it, and step 3 is where a typo
would become permanent.

On a real ceremony steps are separated by people and machines, the fingerprint is
compared out of band, and each member backs up `share` and its passphrase
**separately** — the file is useless without the passphrase and vice versa.

**On this machine, `committee/chain69/` is the vault.** Two of those three
sealed shares, with their passphrases, are the only thing that can move what the
vault holds; there is no key derivation to fall back on, no dev index behind it,
and this tree is not under git. Delete that directory and whatever the vault
holds on chain 69 is gone, on a chain that cannot be re-genesised. If the point
of the exercise is to hold value for more than an afternoon, copy it somewhere
else before step 5.

---

## Step 2 · Funding, sized to what index 3 actually holds

The dedicated-L1 allocation table (`internal/devnet/devnet.go:64`) hands the
faucet 10 M ZNN and 100 M QSR. Index 3 has 37,765 ZNN and 363,815 QSR, so those
numbers are off by three orders of magnitude and every amount here is rebuilt
from the balance rather than copied.

| Account | ZNN | QSR | Fuse (QSR) | Why |
|---|---:|---:|---:|---|
| Vault (group address) | — | — | **5,000** | Receives, payouts and burn mirrors — 10.5 M plasma. Not a ceiling-avoidance number: chain 69's ceiling is 100,000,000 QSR. |
| Faucet (201) | 8,000 | 60,000 | 5,000 | Drips on both layers, and deposits its own ZVM float through the vault. |
| Relayers (202–209) | — | — | 5,000 each = **40,000** | Batch posting only; they never move value. See below — this is the number the whole stack's throughput hangs on. |
| Bench funder (210) | 5,000 | 20,000 | 5,000 | Only if the relayed-lane benchmark is wanted. |
| **Total** | **13,000** | **80,000** | **55,000** | 135,000 QSR committed of 363,815 |

That leaves roughly **24,700 ZNN and 228,800 QSR** in index 3 — enough to redo
the whole thing several times, which matters on a chain that cannot be
re-genesised when something goes wrong. It also leaves room for check 7's 10,000
QSR round trip, which comes back less the fee.

**Fused QSR is locked, not spent** — and it stays index 3's. A fusion is a send to
NoM's plasma contract naming a beneficiary; cancelling it returns the QSR to
whoever fused it, not to the account that got the plasma. So every number in the
Fuse column is on loan from index 3 and recoverable, which is why being stingy
with it buys nothing and costs throughput. The corollary matters for the vault:
its plasma is on loan from a dev key, so a committee that mattered would fuse for
itself out of its own QSR rather than depend on index 3 not cancelling.

**Why the relayers get 5,000 each and not 1,500.** An earlier draft gave them
1,500 QSR — 3.15 M plasma — on the reasoning that a full 16 KB block's *base*
plasma is 1,135,112. That is a v1 figure. Under dynamic plasma a block must pay
`ceil(base × nextFusionPrice / 1000)`, so 3.15 M runs out at about **2.8×** and
the relayer's accounts start getting

```
err="ledger.publishRawTransaction: rpc error -32000: not enough plasma on account"
```

`docs/PLASMA.md` measured 23.6× on chain 71 under load, and chain 69's other
traffic moves the same price. 5,000 QSR is 10.5 M plasma, which carries a full
block to about **9.25×** — and that, not 1,500, is where the "past roughly 9.25×
a full block is unpublishable" figure in *Things to know* comes from. It is a
statement about a 5,000-QSR account, not about the chain: on chain 69 there is no
per-account ceiling, so an account fused higher goes higher. If the price sits
above 9× while you care about throughput, `zvm fund -to 202 -fuse 10000` raises
one relayer without re-genesising anything.

Eight relayer accounts is carried over from the dedicated stack, where it was
needed because v1 caps an account at 10.5 M plasma. On chain 69 that cap is
gone (see the measurement above), so the account count here buys **nonce
pipelining**, not plasma — `internal/relayer` runs its senders concurrently and
each needs its own chain. Four would very likely do; eight is kept so the
relayer's own defaults (`-accounts 8 -first-index 202`) need no thought.

`zvm fund` publishes the fusion first, then the transfers, then the recipient's
receives — a transfer nobody receives is not a balance. The vault is the one
entry that takes only a fusion: it needs plasma to publish, and its ZNN and QSR
arrive as deposits that the committee receives for itself.

```powershell
function Fund($to, $znn, $qsr, $fuse) {
  & $zvm fund -l1 $L1 -l1-chain-id 69 -from-index 3 -to $to -znn $znn -qsr $qsr -fuse $fuse
}

Fund $vault 0     0      5000     # the FROST group address
Fund 201    8000  60000  5000     # faucet
202..209 | ForEach-Object { Fund $_ 0 0 5000 }
Fund 210    5000  20000  5000     # bench funder (optional)
```

Each call prints the recipient's resulting balance and `plasma <current> of
<max>`; the max is what to read. The vault's line should end `0 ZNN, 0 QSR` and
`10500000` plasma — value arrives later, as deposits.

`zvm fund` is not idempotent: run it twice and you have sent twice. If one call
fails partway, read the account before re-running it rather than assuming.

---

## Step 3 · The L2 genesis, vaulted by the committee

```powershell
if (Test-Path "$repo\l2-69") { throw "l2-69 exists — a second genesis would orphan the volumes that derived from the first" }

$start = [int](Rpc 'ledger.getFrontierMomentum').height + 1

& $zvm l2-genesis -out l2-69 -l1 $L1 -l1-chain-id 69 `
  -start $start -evm karst -min-base-fee 1000000 `
  -inbox-tag zvm69 -vault $vault `
  -bridge-fee-bps 25 -bridge-min-fee ZNN:1000000,QSR:10000000
```

- **`-start` reads the frontier and adds one, and the failure mode is the
  opposite of what an earlier draft said.** `l2-genesis` reads momentum
  `start-1` for its timestamp (`cmd/zvm/l2genesis.go:93`), so a height that has
  gone *stale* still resolves — the frontier has moved past it — and costs only a
  few momentums of backfill on first boot. What fails, with `momentum N not
  found`, is a `-start` **ahead** of the frontier. Reading it immediately is still
  right, because it keeps the funding blocks from step 2 safely below `start`
  where nothing derives them; it is not a race against the lookup.
  The command also refuses outright if the node answers with a chain id other
  than 69 (`cmd/zvm/l2genesis.go:101`), which is the last line of defence against
  pointing this at the wrong node.
- `-evm karst` because op-geth is retired; op-reth is the only client here that
  can follow the chain.
- `-inbox-tag zvm69` gives this rollup its own inbox. Chain 69 is long-lived and
  shared, and two rollups sharing an inbox derive each other's calls
  (`inboxFor`, `cmd/zvm/l2genesis.go:261`). The tag becomes bech32 data
  characters, whose alphabet has no `1`, `b`, `i` or `o` — `zvm69` is clean; a tag
  like `bridge` would be rejected.
- `-vault $vault` is the whole point: the group address replaces `-vault-index`.
- Forks (`-gas-fork`, `-wire-fork`) and the burned-fee rule (`-fee-from`) are
  **left off for the first boot**. A chain that has derived past a fork height
  cannot un-set it — changing either is a re-genesis — so they are worth
  scheduling deliberately later rather than guessing now.
- Resulting L2 chain id is 7,340,400 + 69 = **7,340,469**.

The command writes `l2-69/genesis.json`, `l2-69/rollup.json` and `l2-69/jwt.hex`,
none of which collide with the existing `l2/` and `l2-dp/`.

---

## Step 4 · The compose file

`docker-compose.chain69.yml` is written and parses (`docker compose config`
validates it without starting anything). It is the 47xxx stack with the L1
removed, the committee folded in, op-reth in place of op-geth, and host ports in
a fresh 49xxx group so it runs alongside chains 70 and 71.

What it contains, and the reasoning that is not obvious from reading it:

- **No L1 services.** `l1-pillar` and `l1-rpc` are gone; every service that
  needs NoM dials `host.docker.internal:35997` and carries an `extra_hosts`
  entry so the file also works on Linux.
- **`bridge` is a coordinator, not a key holder.** It runs with `-group`,
  `-signers` and `-signer-token-file` instead of `-vault-index`. Compromising it
  cannot move the vault; it can only stop the vault moving.
- **Three signers**, each mounting only its own member directory, and each with a
  **genuinely different policy**: outflow limits of 5,000 / 2,500 / 7,500 ZNN per
  24 h, and signer 3 at `-payout-depth=10` where the others sit at 6. Three
  identical limits would behave as one limit with three copies, which is not what
  `-outflow-limit` is for. The commands are written out one by one because YAML
  cannot append to an anchored sequence.
  **Expect signer 3 to refuse the first few momentums of every payout** and say
  so in its log; `Quorum.SignBlock` drops a refusing member and forms a quorum
  from the rest (`internal/committee/coordinator.go:64`). That is the design
  working, not a fault.
- **`bridge` waits on the signers' healthchecks.** `committee.NewQuorum` calls
  `/info` on *every* URL in `-signers` and fails if one is unreachable, so the
  coordinator needs all three at startup even though it only needs two to sign.
  It retries rather than exiting, but `depends_on: service_healthy` turns a silent
  stall on first boot into ordinary startup. The signers' `/info` needs the bearer
  token, so the healthcheck presents it from the same file the signer
  authenticates against.
- **`bridge -min-receive=ZNN:1,QSR:10`.** Chain 69 is shared, so anyone can send
  the vault dust and the committee pays ~21,000 plasma to receive each one —
  `docs/PLASMA.md` §2 calls this the only unpriced cost an outsider can run up on
  somebody else's account. Below the floor a deposit waits until no withdrawal is
  outstanding. Nobody is short-changed by it: the ZVM credits a deposit from the
  *send*, which every node derives for itself, so only the vault's ability to
  re-spend those particular coins is deferred.
- **No `-media-url`.** The chain-70 stack writes token URIs under
  `https://nft.zenon.foo` because cloudflared fronts that hostname there
  (`docs/CDN.md`); nothing fronts it here, so copying the flag would have minted
  chain-69 NFTs whose images resolve to another chain's node. Unset, token URIs
  are `/media/<hash>`. Do not "fix" it by naming this stack's own hostname:
  `media.Wrap` gives the public hostname an objects-only handler, so
  `-media-url=http://localhost:49000` would take the explorer, the API and `/rpc`
  off the air.
- **`faucet` runs the small drip profile** from G2, so it never tries to spend
  more than the budget holds.
- **No `-dex` or `-apps`.** Those read `bench/deployment.json` and
  `bench/apps.json`, which `zvm bench setup` and `zvm apps setup` write. The
  explorer's swap and marketplace pages will say there is no deployment, which is
  true. Seeding them is a separate exercise and wants the bench funder from the
  table above.
- **`verifier` keeps its sandbox** — read-only root, unprivileged user, dropped
  capabilities, scratch tmpfs — and has no route to the L1 at all.

Three rehearsal shortcuts are in there deliberately; the first two are called out
in the file's own comments. All three signers read the same NoM and ZVM nodes as
the coordinator — and `docs/COMMITTEE.md` is blunt that a policy checked against
the coordinator's nodes checks nothing. Each passphrase is mounted next to the
share it seals. And the passphrases and the bearer token came out of
`Get-Random`, which is not a cryptographic RNG.

Ports:

| What | Where |
|---|---|
| Explorer, faucet, wallet, `/rpc` | http://localhost:49000 |
| ZVM JSON-RPC (op-reth) | http://localhost:49545 |
| Relayer status | http://localhost:49547/status |
| Bridge (coordinator) status | http://localhost:49548/status |
| Faucet status | http://localhost:49549/status |
| L1 (the operator's own, not ours) | http://127.0.0.1:35997 |

---

## Step 5 · Bring it up

The compose file bind-mounts `committee/chain69/group.json` and
`committee/chain69/token` **as files**. Docker creates a missing bind-mount source
rather than refusing, so bringing this up out of order leaves *directories* at
those paths, and the bridge then fails trying to read a group file that is a
folder. Worse, you have to delete them before a later attempt can work. Check
first; it costs a second.

```powershell
$want = @("l2-69\rollup.json", "l2-69\genesis.json", "l2-69\jwt.hex",
          "committee\chain69\group.json", "committee\chain69\token")
$want += 1..3 | ForEach-Object { "committee\chain69\p$_\share", "committee\chain69\p$_\pass" }
foreach ($p in $want) {
  $f = Join-Path $repo $p
  if (-not (Test-Path $f -PathType Leaf)) { throw "missing (or a directory): $p — do not bring the stack up yet" }
}
# The vault in the rollup config must be the group the committee actually holds;
# the bridge and every signer refuse to start otherwise, but say so here.
$cfgVault = (Get-Content "$repo\l2-69\rollup.json" | ConvertFrom-Json).vault
$grpVault = (Get-Content "$repo\committee\chain69\group.json" | ConvertFrom-Json).address
if ($cfgVault -ne $grpVault) { throw "rollup vault $cfgVault is not the group address $grpVault" }

docker compose -f docker-compose.chain69.yml up -d --build
```

`--build` rebuilds the shared `zvm:latest` tag, which the chain-70 and chain-71
stacks also run from. It is the same tree, so this is harmless — but those
containers pick the new image up the next time they restart, so do not be
surprised.

First boot takes a few minutes: op-reth initialises, the node derives from
`-start` up to the frontier, and the signers unseal their shares before the
coordinator's `depends_on` is satisfied.

---

## Step 6 · Verify, in this order

Do not skip to the next check when one fails — each rests on the one before.

**Check 1 · The L1 is reachable from inside a container.** The single most likely thing
to be wrong, and it surfaces as a confusing derivation stall rather than a
connection error. (The image carries GNU wget, not busybox's, so `--post-data`
works.)

```powershell
docker exec zvm69-node wget -qO- --header='Content-Type: application/json' `
  --post-data='{"jsonrpc":"2.0","id":1,"method":"ledger.getFrontierMomentum","params":[]}' `
  http://host.docker.internal:35997
```

**Check 2 · Derivation is moving.**

`/api/info` is nested — `l1`, `l2` and `derivation` — so read it by section:

```powershell
Invoke-RestMethod http://localhost:49000/healthz    # {"ok":true, …}
$i = Invoke-RestMethod http://localhost:49000/api/info
"L1  chain $($i.l1.chainId) frontier $($i.l1.frontier)"
"    inbox $($i.l1.inbox)"
"    vault $($i.l1.vault)"
"L2  $($i.l2.evm) chain $($i.l2.chainId) head $($i.l2.head) start $($i.l2.startMomentum)"
"    derived momentum $($i.derivation.momentum), lag $($i.derivation.lag), stalled $($i.derivation.stalled)"
```

Expect `l1.chainId` 69, `l2.evm` karst, `l2.chainId` **7340469**, `l1.vault` equal
to `$vault`, and `derivation.lag` falling towards single digits as it catches up
from `-start`.

`/healthz` answers 503 once derivation has stalled — behind NoM and no longer
catching up — which is the designed signal (`docs/HALT.md`). A node still working
through history from `-start` is behind but healthy, so `lag` high with
`stalled: false` on first boot is fine.

**Wait here until `derivation.lag` is in single digits before going on.** The next
checks use `zvm deposit` and `zvm withdraw`, whose `-wait` is 90 seconds by
default; against a node still catching up they time out on a deposit that was
perfectly good.

**Check 3 · The quorum formed.** `bridge /status` does *not* report the signers — it has
the vault, its balances, the fee counters and the funding plan
(`internal/bridge/bridge.go:513`) and nothing about the committee. The proof is
the coordinator's log line, which is written only after `NewQuorum` got `/info`
from **every** signer:

```powershell
docker logs zvm69-bridge 2>&1 | Select-String 'vault held by committee'
# vault held by committee address=z1… threshold=2 signers=http://signer-1:8601,…
docker compose -f docker-compose.chain69.yml ps signer-1 signer-2 signer-3
# all three: Up (healthy)
```

Each signer also logs `committee signer id=… vault=… threshold=2` once it has
unsealed its share and agreed with the rollup config.

**Check 4 · A deposit is credited — which does *not* yet prove the committee works.**

```powershell
& $zvm deposit -config l2-69/rollup.json -l1 $L1 -eth http://127.0.0.1:49545 `
  -index 3 -amount 10
```

Read that receipt for what it is. The ZVM credits a deposit from the **send**,
which every node derives independently (`docs/PLASMA.md` §2), so this succeeds
whether or not the vault ever received anything and whether or not a single
signer is alive. What it proves is derivation, the inbox tag and the bridge
contract — worth proving, just not the committee.

**Check 5 · The committee's first signature.** The vault's *receive* of that deposit is
the first block a quorum has to sign. Check the vault's own NoM account:

```powershell
$st = Invoke-RestMethod http://localhost:49548/status
"received=$($st.receivedThisRun) unreceived=$($st.unreceivedDeposits) " +
  "deferred=$($st.deferredDeposits) vaultZNN=$($st.vaultBalances.ZNN)"

# And independently of the coordinator, off the L1 itself:
$vi = Rpc 'ledger.getAccountInfoByAddress' @($vault)
"vault height $($vi.accountHeight), $(& $has $vi $ZNN) ZNN on NoM"
```

`receivedThisRun` ≥ 1, `vaultBalances.ZNN` at 10 ZNN (1000000000 base units), and
a vault `accountHeight` above zero all say the same thing: a FROST signature that
NoM accepted. The account height is the one that cannot be misreported by the
process you are testing — the vault had published nothing before this.

If `unreceivedDeposits` stays at 1 and `received` at 0, the quorum is not signing.
`docker logs zvm69-bridge` and each signer's log say which member refused and why.
A deposit under 1 ZNN would sit in `deferredDeposits` by design (`-min-receive`),
so test with 10, not 0.5.

**Check 6 · A withdrawal — the path that exercises the payout policy, the finality
depth and the fee.** `-to` is required; there is no default.

```powershell
$to = $funder   # index 3's NoM address, from step 0
& $zvm withdraw -config l2-69/rollup.json -l1 $L1 -eth http://127.0.0.1:49545 `
  -lane native -index 3 -to $to -amount 1 -follow
```

`-follow` waits up to six minutes for the vault to have paid it *and* the ZVM to
have derived the payout back into the bridge contract, so a pass covers the whole
round trip rather than just the request. Six momentums of `-payout-depth` at chain
69's ~14.5 s cadence is about 90 seconds before the quorum will even consider it;
signer 3 waits for ten and refuses until then, and the other two form the quorum
without it.

The last line will report that the recipient holds the payout as an **unreceived**
send, and that is correct, not a failure: NoM credits nothing until the recipient
publishes a receive, and nothing in this stack signs for index 3. The proof the
payout happened is the vault's ZNN balance dropping by the amount less the fee —
25 basis points, floored at 0.01 ZNN, so 1 ZNN out costs 0.01 and the vault keeps
it.

**Check 7 · D8's self-funding, which needs QSR and would otherwise never run.**
`-fund-plasma` fuses the **QSR** part of the withdrawal fee:
`fundableQSR()` is `feesQSR − fusedQSR`, and `PlanFunding` floors to whole QSR
against NoM's ten-QSR minimum. At 25 basis points that means no fusion is possible
until QSR withdrawals have earned ten QSR of fee — so a ZNN-only test leaves this
feature looking broken when it is merely unpaid. 10,000 QSR through the round trip
earns 25:

```powershell
& $zvm deposit -config l2-69/rollup.json -l1 $L1 -eth http://127.0.0.1:49545 `
  -index 3 -token QSR -amount 10000
& $zvm withdraw -config l2-69/rollup.json -l1 $L1 -eth http://127.0.0.1:49545 `
  -lane native -index 3 -to $to -token QSR -amount 10000 -follow
```

The vault pays out 9,975 QSR and keeps 25. Within `-fund-every=2m` the
coordinator should propose a 25-QSR fusion, every signer should check it against
its own nodes (`CheckFuse`: QSR, to the plasma contract, for the vault, no more
than revenue allows), and `/status` should show it:

```powershell
$st = Invoke-RestMethod http://localhost:49548/status
"feesQSR=$($st.feesQSR) fusedQSR=$($st.fusedQSR) fused blocks: $($st.fusedBlocksThisRun -join ',')"
$st.funding | Format-List
```

`fusedQSR` should reach 2500000000 (25 QSR) and the vault's fused total on NoM
should go 5,000 → 5,025. `funding.reason` explains any zero: `no fee revenue left
to fuse` is the normal steady state once it has run, and `the ZVM has derived to
momentum N and the vault's latest block is in M` is `PlanFunding` refusing to plan
on state it has not caught up with — correct, and it clears by itself.

Run this a third time and something more interesting happens: signer 2's
`-outflow-limit` of 25,000 QSR per 24 h binds, it refuses, and signers 1 and 3
form the quorum without it. That is the point of giving the three members
different limits — one member's caution is visible in its log without the peg
stopping — and it is worth seeing once.

**Check 8 · The faucet, which is also the second committee exercise.** The faucet
pays on both layers, and its ZVM float arrives as a *deposit through the vault* —
so `KeepFunded` succeeding is a second, independent thing the quorum had to sign
for.

```powershell
$who = (& $zvm evm-key -index 0 | Select-String 'address\s+(0x[0-9a-fA-F]+)').Matches[0].Groups[1].Value
Invoke-RestMethod -Uri http://localhost:49549/drip -Method Post `
  -ContentType 'application/json' -Body (@{ address = $who } | ConvertTo-Json)
Invoke-RestMethod http://localhost:49549/status | Format-List
```

The drip answers with `l1ZnnBlock`, `l1QsrBlock`, `l1FuseBlock`, `l2NomBlock` and
`l2Tx`. `/status` should show `l1Balances` shrinking from 8,000 ZNN / 60,000 QSR
and `l2Balance` at or above the 200-ZNN floor. If `l2Balance` is zero and the
faucet log repeats a top-up failure, that is `KeepFunded` trying to deposit 2,000
ZNN it does not have — check the ZNN column of step 2's table actually landed.

**Check 9 · The batch lane.** Nothing so far has used the relayer: deposits and
native withdrawals are `0x01` inbox messages, not `0x02` batches. An EVM-key
transaction goes through op-reth to the relayer and onto NoM as a batch:

```powershell
& $zvm evm-send -config l2-69/rollup.json -eth http://127.0.0.1:49545 `
  -l1 $L1 -key-index 0 -to $who -value 1
Invoke-RestMethod http://localhost:49547/status | Format-List
```

The relayer's `/status` is where its plasma budget is visible. This is the service
whose fused QSR step 2 argued about: if it reports refusals with `not enough
plasma on account`, read `nextFusionPrice` from the L1 and compare it against the
table in *Things to know*.

**Check 10 · Prove the threshold is real.** This is the check that turns a
configuration into a peg. Run it interactively — **the second withdrawal is
*meant* to fail**, so pasting the whole block under `$ErrorActionPreference =
'Stop'` will stop it before the `docker start`.

```powershell
docker stop zvm69-signer-3    # 2 of 3 — it must keep paying
& $zvm withdraw -config l2-69/rollup.json -l1 $L1 -eth http://127.0.0.1:49545 `
  -lane native -index 3 -to $to -amount 1 -follow
```
```powershell
docker stop zvm69-signer-2    # 1 of 3 — it must stop, and say why
& $zvm withdraw -config l2-69/rollup.json -l1 $L1 -eth http://127.0.0.1:49545 `
  -lane native -index 3 -to $to -amount 1 -follow
# expect: "withdrawal N not paid within 6 minutes", and the bridge log naming
# the members it could not reach
docker logs --tail 40 zvm69-bridge
```
```powershell
docker start zvm69-signer-2 zvm69-signer-3
Invoke-RestMethod http://localhost:49548/status | Select-Object -Expand payouts
```

A committee that keeps paying at *t*−1 is not a committee. Three things to know
before running it:

- **Do not restart the coordinator while a signer is down.** `NewQuorum` needs
  `/info` from every URL in `-signers`, so a bridge that restarts with signer 3
  stopped waits instead of running on two. Signing at *t* is a property of the
  running process, not of startup — which is a real liveness wart, not a
  subtlety of the test.
- **Stop signer 3 first, not signer 1.** With signer 3 at `-payout-depth=10` the
  remaining pair are both at 6, so the payout is considered as soon as the
  withdrawal is six momentums deep. Stopping signer 1 instead leaves 2 and 3, and
  every payout then waits for signer 3's ten — correct, but slower, and easy to
  misread as a hang.
- The outstanding withdrawal from the 1-of-3 attempt is not lost. It pays as soon
  as a second member is back; `docker start` both and watch `payouts` in
  `/status` clear.

---

---

## Step 7 · The swap and the farm

Deployed on chain 69 on 2026-09-19. Four commands, all paid for by the bench
funder (index 210) and the deployer EVM key 900, and all sized for this chain
rather than for the 10 M ZNN a genesised one hands out:

```powershell
$f = @('-config','l2-69/rollup.json','-l1',$L1,'-eth','http://127.0.0.1:49545',
       '-submit','http://127.0.0.1:49547','-api','http://localhost:49000')

# WZNN, the Uniswap V2 factory and the router, and nothing else (G10).
& $zvm bench setup @f -out bench/deployment-69.json -fund-znn 4000 -swap-only

# The one pair: ZNN against wrapped QSR. -znn 0 opens no benchmark-token pools.
& $zvm dex seed @f -deployment bench/deployment-69.json -znn 0 -qsr-pool-znn 500

# The marketplace, the collection factory and the farm. With only one pair on the
# chain, the farm stakes ZNN/WQSR alone and that pool takes every reward.
& $zvm apps setup @f -deployment bench/deployment-69.json -out bench/apps-69.json `
  -fund-znn 1000 -reward-qsr 5000 -farm-days 30 -pool-znn 500

# WZNN and WQSR earn their checkmark, so a copycat with the same name cannot.
& $zvm verify-tokens -api http://localhost:49000 -deployment bench/deployment-69.json
```

Three flags carry the whole difference from chains 70 and 71, and each defaults to
the old behaviour so those chains are untouched:

- **`-fund-znn`** on `bench setup` and `apps setup`. The deployer's ZVM float was
  hardcoded at 200,000 and 100,000 ZNN, deposited from index 210 — which holds
  8,000 here. Same shape of bug as G2, in two more places.
- **`-swap-only`** on `bench setup`, and **`-znn 0`** on `dex seed`. G10: a pair
  cannot be un-created.
- **`-reward-qsr`** and **`-pool-znn`**, so the farm's budget and the pool's depth
  fit what index 210 holds. `apps setup` also no longer asks for the pool's QSR
  when the pool already has liquidity, which is what let `dex seed` open it first.

What the node does with the result: `-dex=/bench/deployment-69.json` and
`-apps=/bench/apps-69.json`, with `./bench` mounted read-only. **Both files are
re-read per request** and ignored unless their `chainId` matches, so seeding a live
node makes the swap and marketplace pages appear without a restart — and the
chain-70 files sitting in the same directory are inert.

As deployed on chain 69 (with the benchmark set, before `-swap-only` existed):

| | |
|---|---|
| Uniswap V2 router | `0xa381ae1e4c9573e38ccd95d37646c891d9e4ca34` |
| WZNN | `0x91F5DDA8243e34697C99bE282e05bf48e35A0115` |
| **ZNN/WQSR pair** | `0x9f04D9f623B41269591B6F2cccf02DD6CccFe745` — 500 ZNN + 3,600 QSR at 7.2 |
| Farm | `0xDe8815a4Dc03c9D6480dc53a4A547290C8dec566`, 5,000 QSR over 30 days |
| Exchange | `0x6eA08d7546839C0AcBb8841BE41e3494c54d9079` |
| Collection factory | `0x43d8669B66712dcF21dEEB9F286Ff35CDBDE3A71` |
| Collections | Zenon Momentums (12 items), Plasma Shards (6, minted from a Zenon key) |

7.2 QSR per ZNN is the fallback ratio: `qsrPerZNN` asks this node's `/api/prices`
for what the bridged pair trades at, and falls back when it cannot be reached.

### Swap fees into the farm — `ZVMFarmFeeStrategy`

A Uniswap V2 swap charges 0.30%, which normally accrues to the liquidity
providers. When the factory names a `feeTo`, each pair diverts **one sixth of that
— 0.05% of volume** by minting `feeTo` new LP. `ZVMFarmFeeStrategy` is that
`feeTo`: `harvest` burns the LP it holds, sells both sides for QSR through the
router, and calls `ZVMFarm.fund`, which is permissionless. So every swap on every
pair buys QSR and lengthens the ZNN/WQSR farm's emission.

`apps setup` deploys it through the same CREATE2 deployer as the other four
contracts (salt `farm-fee-strategy`), calls `setFeeTo`, and seeds it. Harvest with:

```powershell
& $zvm apps harvest-fees @f -apps bench/apps-69.json -deployment bench/deployment-69.json
```

Four things about it are worth knowing before relying on it:

- **`fund` extends the budget, it does not raise the rate.** `rewardPerSecond` is
  fixed at construction, so fee revenue makes the farm last longer rather than pay
  faster. That is deliberate: emission is a promise the farm has to keep, and fee
  revenue is not predictable enough to promise with.
- **A pair mints the protocol fee only inside `mint` and `burn`.** Swapping accrues
  it in the reserves and mints nothing, so a pool whose liquidity never moves hands
  the strategy nothing however much it trades. `harvest` therefore burns a position
  of its own to force the mint, which means it needs a seed — `apps setup` gives it
  one, sized from the pool's reserves, because with QSR at 8 decimals against
  wrapped ZNN at 18 a seed that is dust on one side rounds to zero on the other.
  It also means one round of latency: a harvest realises what accrued up to the
  *previous* harvest and arms the next one.
- **0.05% is not tunable.** The one-sixth is fixed in the deployed pair's code, so
  changing the share means new pairs, which on a live chain means new pools.
- **There is no sweep and no owner.** Every path out of the strategy ends in
  `fund`, so the fees cannot be redirected to anyone — and a token with no route to
  QSR stays stuck, which is the price of that guarantee.

**Deployed and measured on chain 69, 2026-09-19.** `zvm apps enable-fees` adds it
to a chain whose marketplace already exists — which `apps setup` cannot, because
`ZVMFarm.addPool` reverts on a pool it already has, so a second full setup fails
partway and funds the farm twice on its way there.

| | |
|---|---|
| Strategy | `0xD7F40EC701018D0898dE890Db929055Dd9981083` (salt `farm-fee-strategy`) |
| `factory.feeTo` | the strategy ✓ |
| Seed | 4.045 QSR + 0.564 WZNN, one ten-thousandth of the pool |

What it did, end to end:

| Step | Farm `totalFunded` | Emission ends |
|---|---:|---|
| before | 5,000 QSR | 2026-10-19T03:11:43Z |
| 34 swaps, harvest 1 | 5,008.06 | +4,178 s |
| 24 swaps, harvest 2 | 5,008.41 | +183 s |

Harvest 1 returns the seed as well as the fee, so **harvest 2 is the honest
number: 0.3532 QSR of pure protocol fee from 24 swaps**, and the burn left
624,026,885,740 LP behind — so the next round is armed. The loop sustains itself
as long as volume keeps arriving between harvests; if it ever stops with no fee
accrued, the seed is consumed and `seed` has to be called again.

Two things the deploy itself taught, neither of which reading had:

- **The deployer holds neither side of the seed by the time it is needed.** The
  pool took its wrapped ZNN and the farm took its QSR, so the first attempt
  reverted. `armFeeStrategy` now provisions each side the way it arrives in the
  first place: QSR over the bridge, wrapped ZNN by wrapping some of the deployer's.
- **Only a pair the strategy holds LP in gets captured.** On chain 69 the three
  benchmark pairs report `strategy LP 0` after every harvest: their fees accrue in
  their reserves, but nothing mints or burns there, so there is no position to
  force the mint with. `zvm apps harvest-fees` prints that column for exactly this
  reason. Seeding them would capture them too — `seed` takes any pair and anyone
  may call it — and on a `-swap-only` chain the question does not arise, because
  ZNN/WQSR is the only pair.

And one thing to keep in mind about where the money lands: `fund` credits the farm,
not a pool, so it is split by allocation points. Chain 69's farm has three pools
(ZNN/QSR 60, ZNN/BNA 25, ZNN/BNB 15), so 60% of the fee revenue reaches ZNN/WQSR
stakers. A chain seeded with `-swap-only` has one pool, and all of it does.

## Things to know before touching this

- **Chain 69 is not ours to re-genesis, and the ZVM's own re-genesis is not
  free.** The mechanics are easy — a new start height and clean volumes; the L1
  keeps its history either way:
  ```powershell
  docker compose -f docker-compose.chain69.yml down -v
  Remove-Item -Recurse "$repo\l2-69"
  # then step 3 again with a fresh -start, and step 5
  ```
  **What it costs is every L2 balance.** Deposits are credited by deriving the
  sends that carried them, and a rollup that starts at a later momentum never
  derives the earlier ones. So the depositors' ZVM balances vanish while their ZNN
  and QSR stay sitting in the vault, backing nothing that any account on the new
  chain can withdraw. That is a loss for the depositor and an unspendable surplus
  for the committee — `fundableQSR()` will not touch it, because it is not fee
  revenue. **Drain the vault first** (withdraw every balance, check
  `/status`'s `lockedZNNOnZVM` and `depositedQSR` are back to zero) and only then
  re-genesis. While nothing but test deposits exists, this is academic; once the
  faucet has served anyone it is not.

  **Keep `committee/chain69/` either way.** The vault address is the group key, so
  reusing the committee is what keeps the old balance reachable at all; a fresh
  ceremony gives a new address and strands whatever the old vault still holds.
- **The shares are the only key to the vault.** Two of the three sealed shares,
  with their passphrases, are the whole of it — no dev index, no derivation path,
  no recovery. `committee/chain69/` lives in a tree with no git history on a chain
  that cannot be re-genesised. Losing two of three loses the balance permanently.
- **Dynamic plasma bounds the rollup, and other people share the bound.** A
  momentum admits `MaxBasePlasmaInMomentum` — 4,200,000 by default — as a hard
  ceiling in both the content selector and the verifier, and a full 16 KB block
  costs 1,135,112. That is **~3.7 rollup blocks per momentum against chain 70's
  100**, and chain 69's other traffic comes out of the same budget.
  `internal/relayer/forecast.go` already handles this correctly: it detects
  momentum version 2 from the chain, budgets in plasma rather than blocks, and
  subtracts what everyone else used before taking its share. Expect low
  throughput here and do not read it as a regression.
- **`nextFusionPrice` is the thing to watch under load**, and the number to watch
  it against is *your own fusion*, not a constant. It sits at 1× (`1000`) now. A
  block must pay `ceil(base × price / 1000)` in fused plasma, so a full 16 KB
  block at 1,135,112 base becomes unpublishable **by a given account** once the
  price passes that account's max plasma ÷ 1,135,112:

  | Fused QSR | Max plasma | Full block unpublishable past |
  |---:|---:|---:|
  | 1,500 | 3,150,000 | ~2.8× |
  | 5,000 | 10,500,000 | ~9.25× |
  | 10,000 | 21,000,000 | ~18.5× |

  The familiar "9.25×" is the 5,000-QSR row, which on a v1 chain is also the
  per-account ceiling and therefore a fact about everyone. **On chain 69 it is
  only a fact about how much you fused** — the ceiling here is 100,000,000 QSR
  (G3), so an account can be fused past it. `docs/PLASMA.md` measured 23.6× on
  chain 71 under load, which the 5,000-QSR row does not survive. `zvm rebid`
  unwedges an account stuck behind a block priced at a rate the chain has left
  behind; it cannot make an account afford a price, so raising the fusion with
  `zvm fund -fuse` is the actual remedy.
- **The genesis timestamp is from February 2025**, far in the past, so momentum
  cadence is ragged rather than the nominal 10 s. Measured directly on this chain:
  99,346 → 105,286 in about a day, a **14.5 s** average, matching what
  `internal/devnet` records for an old genesis. Nothing to fix; it is not our L1.
  Every latency figure should be read against the cadence actually observed —
  including `-payout-depth=6`, which is ~90 s here rather than 60.
- **A `feeMarket` change requires restarting every node**, including any second
  node used for `zvm compare`. The parameters ride in the block's `extraData`, so
  a stale node builds a different chain from the same inputs: matching state
  root, different block hash.
- **Two quorums exist during a reshare.** Between the incoming members finishing
  and the outgoing members destroying their shares, both sets can move the vault.
  Keep that window short. `zvm reshare` exists in the tree but has never been run
  against chain 69, and the rehearsal for it is the `committee5` profile of
  `docker-compose.committee.yml` on chain 70 (G4).
- **The committee's endpoints are not published to the host.** A signer serves
  only `/info`, `/commit` and `/sign`, all behind the bearer token
  (`internal/committee/signer.go:358`), and the compose file maps no port for
  them. Reach them with `docker exec` and the token from
  `committee/chain69/token`, or read the logs.
- **Nothing here pays the relayer.** The gas fork is off for the first boot, so
  the relayer is a volunteer spending its own fused QSR on other people's
  transactions — D20's position, fine for a devnet (`docs/PLASMA.md` §1).
- **This tree is not under git.** There is no undo.

---

## Where this stands

**Code, 2026-09-17:** G1–G4 and G6 are in the tree.

**Scripts, 2026-09-18:** `deploy-chain69.ps1` and `verify-chain69.ps1`. The deploy
script was rehearsed end to end against a throwaway root (`-Root <temp> -DryRun`):
the ceremony ran, three group files came out byte-identical, `audit` recomputed the
vault from the public transcript, the genesis was written at the live frontier plus
one, the funding plan came out of real chain reads, and the bind-mount preflight
passed. Three guard paths were fired on purpose and produce the message they
should: a re-run (ceremony skipped and audited, genesis kept), a half-finished
ceremony, and an account that looks live. Both scripts parse clean.

**Procedure, 2026-09-18:** G7 and G8. Re-verified today rather than asserted:
`go build ./...` and `go test ./...` pass; `docker compose -f
docker-compose.chain69.yml config` parses with the signer healthchecks and the
three distinct signer policies; step 0 ran against the live node and reports
chain 69, momentum version 2, `nextFusionPrice` 1000, and index 3 at 37,765.98
ZNN / 363,815.40 QSR / 10,300 QSR fused — the same figures measured on 2026-09-17;
and **step 1 was run start to finish in a scratch directory**, producing three
byte-identical group files and an `audit` that recomputes the vault from the
public transcript alone.

**Deployed, 2026-09-19.** Steps 0–7 have been run against chain 69. The committee
holds the vault at `z1qzxc0kq0fatfunnnr28rls746h753zsf7gp88f` and has signed three
deposit receives and six payouts, including QSR. Uniswap V2, the ZNN/WQSR pool at
`0x9f04…e745`, the marketplace with two collections, and a farm funded with 5,000
QSR are live and indexed by the node.

**Two bugs came out of doing it rather than reading it: G9 and G10.** G9 is the one
that mattered — `-fund-plasma` was fusing QSR the ZVM could not see, and the next
QSR deposit would have started it eating the balance backing everybody's deposits.
Nothing in the review found it; the first hour of running found it.

**Step 6's check 11 is the one to confirm by hand.** The payout journal and the
signer restarts are consistent with the threshold test having been run, but that is
inference, not a record — and it is the check that turns this from a configuration
into a working peg, so it is worth running deliberately:

```powershell
.erify-chain69.ps1 -Threshold
```

**Still open regardless:**

- The native-lane fleet benchmark does not fit the budget (G6), and no benchmark
  number from this chain belongs in `docs/BENCHMARK.md` anyway: chain 69 carries
  other people's traffic, which is the whole reason D5 built a dedicated L1.
- `internal/committee` has still never been reviewed by anyone outside this tree,
  which is the same open item `HANDOFF.md` carries. A threshold-trust peg is only
  as good as the implementation of the threshold.
- Three shares on one host is a rehearsal. Nothing in this document, run
  faithfully, produces custody worth trusting with value that matters.
