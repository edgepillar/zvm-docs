# zvm-docs guide

Everything written down about the **ZVM** — a based EVM on Zenon's Network of
Momentum — in one place, because the code is in ten repositories and the
reasons for it should not be scattered across all ten.

Zenon orders and stores every transaction; a derivation node turns each momentum
into exactly one EVM block and hands it to a stock op-reth, which executes it.
There is no sequencer: the order is whatever NoM committed, so any node with the
same NoM history rebuilds the same chain. Nothing about it needs a change to
Zenon — no spork, no new embedded contract, no pillar cooperation.

## Start here

| | |
|---|---|
| **[ARCHITECTURE.md](ARCHITECTURE.md)** | What each repository is, what depends on what, and why the lines are where they are |
| **[spec/SPEC.md](spec/SPEC.md)** | The consensus: what two independent nodes must agree on, byte for byte |
| **[spec/DECISIONS.md](spec/DECISIONS.md)** | D1–D37: what was chosen, the prior art, and what reversing it costs |
| **[planning/UPGRADES.md](planning/UPGRADES.md)** | How a rule change reaches every node: forks, halting stale nodes, the release timeline |
| **[planning/FUTURE.md](planning/FUTURE.md)** | Future improvements, not decided: what the ZVM could do with NoM, and which items need a spork |

## By what you are doing

**Reading the design.** `spec/SPEC.md` first — it is pinned by
zvm-core's `vectors/vectors.json`, which is the contract between the Go node, the CLI and the
browser, so a disagreement about the wire format is a failing test rather than a
forked chain. Then `spec/DECISIONS.md`, which answers "why is it like that" for
almost everything and says what each answer would cost to undo.

**Running one.**

| | |
|---|---|
| [operations/DEPLOY-DEVNET.md](operations/DEPLOY-DEVNET.md) | Standing up a chain from nothing, end to end |
| [operations/deploy-contract.md](operations/deploy-contract.md) | Deploying one contract to devnet 69 and verifying it in the explorer |
| [operations/HALT.md](operations/HALT.md) | Every path that stops a node, and what to do about each |
| [operations/PLASMA.md](operations/PLASMA.md) | Who pays for what on NoM, and the four places the payer is not the beneficiary |
| [operations/COMMITTEE.md](operations/COMMITTEE.md) | The vault's key ceremony, resharing, and the signer policy |
| [operations/PUBLIC.md](operations/PUBLIC.md) | Putting a node on a public hostname without handing it to anyone |
| [operations/CDN.md](operations/CDN.md) | Serving NFT media from a CDN hostname, as the first deployment did |

Read `HALT.md` before an incident, not during one. Derivation retries forever
rather than exiting, on purpose — which means a halted node passes every naive
health check and goes on answering confidently from a chain that stopped.
`/healthz` is the thing that tells you.

**Judging whether it is fast enough.** [benchmarks/BENCHMARK.md](benchmarks/BENCHMARK.md)
has the method and the headline numbers;
[benchmarks/results/](benchmarks/results/) has every run's tables, kept because
the interesting part is usually the difference between two of them. `RESULTS-*`
are single-workload matrices and `FLEET-*` are the whole-system runs — every
service at once, 100k EVM wallets, 1k fused Zenon accounts.

A warning that the tables repeat and that is easy to skim past: numbers measured
before **2026-09-16** were taken on op-geth, which cannot follow either chain
any more, and on chains that have since been destroyed and re-genesised. They
are kept as history, not as a baseline.

## What is not here

The raw benchmark output — hundreds of megabytes of per-phase JSON — stays with
the runs that produced it, and is not published. What is here is what someone
would read.

Each repository keeps its own README, describing that component. This repo
holds what spans them.
