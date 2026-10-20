# Where a node stops, and what it should do instead

**Written 2026-09-16.** The halt analysis D12 asks for: every path that ends in
this node refusing to build the next block, how likely each one is, and what is
done about it.

The rule that creates them is [D12](../spec/DECISIONS.md#d12--the-engine-has-the-last-word-and-we-rebuild-without-the-transaction-it-named):
the execution engine has the last word on a transaction, derivation removes the
one it named and rebuilds — but **a refused deposit, payout or native call is
not a drop**. The value a deposit records has already moved on NoM, and a
native call is a signed NoM block no node may decide to ignore. So a node that
cannot include one has learned that its idea of the chain and its engine's
disagree, and the only safe thing left is to stop.

---

## What "halt" actually means here

It is worth being exact, because the word suggests something the code does not
do. `node.Deriver.Run` treats every derivation error the same way: log it,
sleep (one second, doubling to thirty), and try the same momentum again. So a
halted node

- **does not exit**, and its container does not restart-loop;
- **keeps answering** `/api/*`, the RPC proxy and `/status`, from state that
  stops advancing;
- **reports the gap**: `/status` carries `lagMomentums`, which grows by one
  every ten seconds once derivation stops.

That is already closer to *quarantine* than to *crash*, which is the second
question D12 asks. What it was missing is that the node did not *say* it was
stuck: a reader got a confident answer from a chain that stopped an hour ago.
**The gap between "stalled" and "quarantined" is honesty, not mechanism** — and
since 2026-09-16 the node closes it. `GET /healthz` answers 503 once derivation
has stalled, `/api/info` carries `derivation.stalled` and
`derivation.sinceProgressMs` beside the lag, and both compose files point their
healthcheck at it, so a stuck node is taken out of rotation rather than left
serving a dead chain.

**Stalled is not the same as behind.** A node working through a long history is
behind and perfectly healthy; a node level with the frontier has nothing to
derive, which is an idle chain's normal state. Stalled means both at once — NoM
holds momentums this node has not derived, *and* its head has not moved for
`-stall-after` (90 s by default, nine momentums at this devnet's cadence).

---

## The paths

Ordered by how much they should worry an operator. "Consensus" means the node
and its peers would build different chains if it carried on; those are the ones
where stopping is right.

| # | Path | Trigger | Likelihood | Consensus? |
|---|---|---|---|---|
| 1 | **Mandatory transaction refused** — `Plan.Drop` on a deposit, payout, burn mirror or native call | the engine cannot apply a transaction derivation says it must | low, and always a bug somewhere | yes |
| 2 | **Mandatory transaction silently missing** — `Plan.Missing` reports one absent from the payload | an engine that skips rather than errors (op-reth's behaviour is its own; see D2) | low; not yet observed on op-reth | yes |
| 3 | **Unplanned transaction** — `Plan.Unplanned` finds one the plan did not ask for | the engine added something: a client bug, or the wrong client | very low | yes |
| 4 | **`newPayload` is not VALID** | the block the engine built, handed back to it, is refused | very low | yes |
| 5 | **Forkchoice not VALID** | the engine will not move its head to a block it just built | very low | yes |
| 6 | ~~**Mandatory gas exceeds the block limit**~~ | **closed 2026-09-16**: from the gas fork the overflow is deferred, not forced (see [below](#the-one-with-a-number-on-it)) | — | was yes |
| 7 | **Momentum structure is wrong** — `BuildPlan` cannot match blocks to content | the L1 node answered inconsistently | low; a node bug or a proxy | no: retrying is right |
| 8 | **NoM reorganised below the rollup's start height** | not observed; go-zenon does not reorg in normal operation | negligible | yes |
| 9 | **Genesis mismatch at startup** | the engine's genesis is not the one `rollup.json` names | operator error, caught immediately | n/a: refuses to start |
| 10 | **The engine or the L1 is unreachable** | a container is down | common, and harmless | no: retrying is right |
| 11 | **The engine came back short** — it restarted having lost blocks this node's store still has, and answers every forkchoice `SYNCING` | an execution client that keeps recent canonical blocks in memory | **op-reth: observed**, on an ordinary container restart | no: rewinding to the engine is right |
| 12 | **The L1 stops taking an account's blocks** — the pillar never received some of them, so it refuses every later one | the devnet's single p2p link dropped while the account was publishing | **chain 70: observed**, under fleet load | no: the node derives the empty momentums correctly; the L1 needs mending |

Paths 7 and 10 are not halts in D12's sense at all — the node has learned
nothing about the chain, only about its own plumbing, and the retry loop is the
correct response. They are in the table because from the outside they look
identical to the others, which is itself part of the problem.

### Path 11, which the op-reth migration found

Not predicted, and worth stating plainly because it is **the one failure on this
page produced by an ordinary operator action**: restarting a container.

op-reth holds recent canonical blocks in memory and flushes them behind
`--engine.persistence-threshold`. Stop the container and those blocks are gone.
Measured on chain 70: the engine came back at block **7,214** with this node's
store at **7,217**.

On an ordinary Ethereum node that is harmless — the missing blocks are re-fetched
from peers. **A based rollup has no peers to fetch them from.** Those three
blocks exist nowhere but here; they are a function of NoM, and the only thing
that can produce them is this deriver. So op-reth answered `SYNCING` to every
forkchoice, forever, and the node retried forever against a head that could
never arrive. `zvm-op-geth` did not show this, which is why the rule that
handles it is new.

The fix is to treat `SYNCING` as what it is — the engine telling us it does not
have our head — and rewind to what it does have:
`Deriver.reconcileWithEngine`, the same reconciliation `Init` runs at startup,
is now also called from the derive loop on `SYNCING`. Those momentums are then
simply derived again. Before it, the only cure was restarting the node so `Init`
ran; after it, the node heals itself.

### Path 12, which the fleet benchmark found

Not a node halt at all — the node is right the whole time — but from outside it
is indistinguishable from one: the explorer shows blocks with no transactions,
the relayer's queue grows, and nothing lands.

Each devnet is two go-zenon nodes and **one** p2p link between them. Account
blocks reach the pillar only by gossip over that link, and gossip is sent once.
On 2026-09-17 (UTC), under the fleet benchmark's load, chain 70's RPC node lost the pillar nine
times in an hour. Momentums are re-downloaded after a reconnect; **unconfirmed
account blocks are not**. So every block published during a gap stayed in the
RPC node's pool and never reached the pillar, and from then on:

- the pillar refused each later block of those accounts, logging
  `account-block previous block is missing`;
- the RPC node went on counting their plasma as spent, so the relayer's accounts
  failed with `not enough plasma on account`;
- chain 70 produced **empty momentums for 75 minutes**, from 2259.

A block that arrives out of order is dropped the same way, so a gap can also
open while one is being filled, if the account keeps publishing.

**Diagnose:** `ledger.getUnconfirmedBlocksByAddress` on the RPC node lists
blocks whose height is above `accountHeight` from the pillar's own
`ledger.getAccountInfoByAddress`, and the oldest of them acknowledges a
momentum well behind the frontier. The pillar's RPC is on the devnet network
only, so ask it from inside its container.

**Mend:** hand the pillar what gossip dropped, in height order —
`ledger.publishRawTransaction` with each block exactly as the RPC node returned
it. They are valid and signed; the pillar takes them and the next momentum
carries them. `zvm bench l1-heal -profile v1` (or `v2` with that chain's flags)
does this for the relayer's accounts, the vault, the funders and the benchmark
wallets, and `zvm bench fleet` runs the same healer throughout (`-heal`,
default on), recording every repair in the phase's `l1Heals` so a result
measured on a mended network says so.

**What it is not:** a protocol fix. A network with more than one path between
producer and publishers does not lose blocks this way, and go-zenon has since
merged libp2p fixes (peer close, peer-limit bookkeeping) that postdate the
2026-09-01 devnet image these chains run. Rebuilding that image is the real
remedy, and it restarts both networks' producers.

### What is no longer on this list

- **A fee-priced call that buys no gas.** Before D9 a native call with no fee
  would have derived to a transaction with a zero gas budget and been refused
  by the engine — path 1, on ordinary user behaviour. The fee rule makes such a
  message **invalid**: it derives to no transaction at all, so there is nothing
  to refuse. D9 closed this one.
- **The fee split.** The derived `settle` call is on the `fees` lane, which
  `Plan.Drop` is allowed to remove. If the engine will not run it, the fees stay
  in the splitter for the next block and derivation carries on. It was written
  that way on purpose: a derived convenience must never be able to stop a chain.

### The one with a number on it

Path 6 is the only one whose safety is arithmetic rather than assertion, so it
is worth doing the arithmetic.

Mandatory gas in a block is

```
1,000,000              the L1 info deposit, every block
+ 250,000 × n          each deposit, payout and burn mirror
+ ≤ 1,520,000          the fee split, when the gas fork is active
```

Against a 300,000,000 gas limit that allows **n ≤ 1,196** mandatory messages in
one momentum. What can a momentum actually hold?

| Ruleset | Bound on mandatory messages | Mandatory gas | Margin |
|---|---:|---:|---:|
| v1 (chain 70, mainnet's rules) | 100 account blocks per momentum | 26 M | **11×** |
| v2 (chain 71, dynamic plasma) | `maxBasePlasmaInMomentum` 4,200,000 ÷ 21,000 per empty send = 200 | 51 M | **6×** |

So it cannot happen today on either chain, and the margin is wide. **But the v2
bound is not a constant of the protocol**: `maxBasePlasmaInMomentum` is settable
on chain with `SetVariables` on the Plasma contract. Raised about six times, to
~25 M, a momentum could carry enough deposits to put a ZVM block past its own
gas limit — and every ZVM node would stop at the same momentum, together.

That was the honest statement of path 6: *the ZVM's worst case is bounded by a
NoM parameter that someone else can change.*

**Built 2026-09-16, at the gas fork.** Derivation now caps what mandatory
messages may claim in one block — `mandatoryShareBp`, half the block here, 600
messages — and carries the overflow into the next block instead of forcing it.
Deposits were the hard case, because the value has moved and they cannot be
dropped, but they can be **deferred**: a mint one block later is still correct
as long as every node defers the same ones by the same rule. The rule is strict
arrival order, so nothing overtakes anything, and the queue is derivation input
like the chunk store — a rewind forgets what the rewound momentums queued and
released.

Two consequences worth stating:

- **The margin stops mattering.** Whatever `maxBasePlasmaInMomentum` is set to,
  a momentum carrying more mandatory messages than a block holds now costs a
  block of delay rather than the chain. Raising it can make the ZVM slower to
  credit a deposit; it can no longer stop it.
- **It is consensus**, which is why it waited for an activation height. A node
  that deferred when its peers forced would build a different chain, so the
  rule starts exactly at `forks.gas` (spec §8) and not a block earlier.

Pinned by `internal/derive/defer_test.go` (order, first-in-first-out, and
nothing deferred before the fork) and `internal/store/deferred_test.go` (the
queue survives a rewind).

---

## What to change

1. ~~**Say it.**~~ **Built 2026-09-16.** `GET /healthz` is 200 while the node is
   keeping up and 503 once it has stalled, and both compose healthchecks ask it,
   so a load balancer takes a stuck node out. `/api/info` reports `stalled` and
   `sinceProgressMs` beside `lag`. Pinned by `internal/node/stall_test.go`.
2. ~~**Name the cause once, loudly.**~~ **Built 2026-09-16.** The retry loop
   logs a *new* failure at error level and repeats of the same one at debug, and
   says so once when derivation resumes. The first one was always the
   interesting one; now it is the one you see.
3. **Keep serving reads, labelled.** Quarantine is better than exit: the state
   is still correct as of the momentum it stopped at, and an operator debugging
   a halt wants to query it. This is what the code already does, and `/healthz`
   now makes it deliberate rather than a side effect of the retry loop.
4. ~~**Defer, don't force** (path 6).~~ **Built 2026-09-16**, at the gas fork:
   mandatory messages are capped per block and the overflow is carried, so the
   one path on this page whose safety was arithmetic no longer depends on a
   NoM parameter somebody else sets.

Paths 1–5 stay halts. A node that cannot reproduce its peers' block has exactly
one safe move, and it is to stop making blocks.

## Checking this page

Paths 1, 2, 3 and 6 are testable without a network, and the plan-level ones are
pinned in `internal/derive` (`Plan.Missing`, `Plan.Unplanned`, `Plan.Drop`'s
refusal to remove a mandatory transaction). Paths 4, 5 and 9 need an engine and
are exercised by bringing the stack up against a mismatched genesis. Path 8 has
never been produced on a running network, here or on NoM.
