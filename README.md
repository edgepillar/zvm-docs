# zvm-docs

Everything written down about the ZVM, a based EVM on Zenon's Network of
Momentum, in one place. The code is in many repositories under
[Zenoglyphs-VM](https://github.com/Zenoglyphs-VM), and the reasons for it
should not be scattered across them.

| | |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | what each repository is, and what depends on what |
| [spec/SPEC.md](spec/SPEC.md) | the consensus, pinned by zvm-core's `vectors/vectors.json` |
| [spec/DECISIONS.md](spec/DECISIONS.md) | what was chosen, the prior art, and what reversing it costs |
| [planning/UPGRADES.md](planning/UPGRADES.md) | how a rule change reaches every node |
| [planning/FUTURE.md](planning/FUTURE.md) | improvements not yet decided |
| [operations/](operations/) | deploying, halts, plasma, the committee, public hostnames |
| [benchmarks/](benchmarks/) | the method and every run's tables |

Numbers measured before 2026-09-16 were taken on op-geth, on chains since
re-genesised. They are history, not a baseline.

MIT licence. The full guide, with how it works and why: [GUIDE.md](GUIDE.md).

[Source and benchmark evidence](EVIDENCE.md) records the current reference pins, historical raw-data gaps and documentation qualification limits.
