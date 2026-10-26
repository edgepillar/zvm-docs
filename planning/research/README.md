# Requirements before implementing future protocol proposals

**Status: proposed review framework.** This directory turns the 24 ideas in
[FUTURE.md](../FUTURE.md) into explicit research questions and acceptance work.
Every entry is `REQUIREMENTS_ONLY`: no entry has an accepted protocol, qualified
vectors, independent review or activation authorization through this document.

Use the [proposal register](PROPOSALS.md) to select a question, then complete a
separate [proposal record](PROPOSAL-TEMPLATE.md). A research prototype may explore
alternatives in isolation. Consensus rules, custody operations and distribution
require their own accepted specification and authority before adoption.

## Source baseline

The register follows the numbering of
[FUTURE.md at Docs `35c5be8a59059a9e1f3aab40c980ef6cd18cf0be`](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/FUTURE.md).
The following are source baselines, rather than evidence of a deployed chain:

| Source | What it establishes for this review |
| --- | --- |
| [SPEC at the same Docs revision](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/spec/SPEC.md) | The described prototype's inputs, accounting, mandatory queue, finality label and vector interface |
| [UPGRADES at the same revision](https://github.com/Zenoglyphs-VM/zvm-docs/blob/35c5be8a59059a9e1f3aab40c980ef6cd18cf0be/planning/UPGRADES.md) | A dated coordination proposal, including unresolved enforcement and operator questions |
| [Core `29e6d9b30be94de3772c850c7a1e9811585a8833`](https://github.com/Zenoglyphs-VM/zvm-core/tree/29e6d9b30be94de3772c850c7a1e9811585a8833) | The implementation baseline to compare before proposing a derivation or encoding change |
| [Bridge `3f8f1197a91e530590f9e1c21ae55d358f62296d`](https://github.com/Zenoglyphs-VM/zvm-bridge/tree/3f8f1197a91e530590f9e1c21ae55d358f62296d) | The separate custody implementation baseline to compare before proposing signer or payout changes |

A completed proposal must pin the exact files, source and vectors it actually
reviews, including the selected NoM implementation and execution client. These
repository pins alone do not establish the behavior of their dependencies.
Refresh the comparison when the selected source changes; retain the old record.
Historical throughput, timing, spork dates and security shorthand in FUTURE.md
are inputs to investigate, not parameters or safety evidence to inherit.

## Requirements shared by every proposal

Each register entry inherits all seven requirements below. An entry's additional
cases supplement them; they do not replace them. A proposal must justify any
requirement marked inapplicable, with a reviewable scope argument.

| ID | Required specification and evidence |
| --- | --- |
| R1: canonicality and finality | Identify the chain/genesis, accepted header/history authority, source completeness, confirmation rule and recovery on conflicting or unavailable history. Distinguish a depth-based label from an independently verified consensus guarantee. Pin how contract sends, successful receives and resulting state are associated with a momentum. |
| R2: identity and replay | Define canonical bytes and domain separation for chain, protocol version, source event, destination, asset, amount and sequence. Specify duplicates, retries, restart/restore, reorg removal and re-inclusion, concurrent delivery and cross-chain reuse. A generic event ID or a fresh database alone is insufficient. |
| R3: state and accounting | Define transitions, ordering, mandatory-queue interaction, irreversible effects, backing, fees, rounding, resource bounds and failure outcomes. State which actor can mint, burn, lock, refund, sign or halt and what prevents an effect from occurring twice. |
| R4: threat model | Name trusted and adversarial roles, compromised thresholds, censorship, withholding, stale or dishonest RPC, crash/restore, key loss, equivocation and dependency failures. Separate safety from liveness and state the residual trust explicitly. |
| R5: economics and capacity | Quantify or leave unresolved the capital at risk, maximum exposure, incentives, griefing/option costs, liquidity, shared NoM capacity, fees and worst-case growth. State the measurement method and source/platform for every numerical claim. No favorable workload or bond-size assumption substitutes for a bound. |
| R6: vectors and qualification | Commit exact input bytes, selected prior state and rules, expected transitions/outputs or refusal, and provenance. Cover positive, negative, boundary, conflicting-history and interrupted/replayed cases. Compare independent implementations where needed. Keep proposed cases, executed results, hosted CI and independent review separate. |
| R7: governance and activation | Identify the proposed change class, authorized decision process and affected operators. Specify review, compatibility, configuration/bytecode/client binding, pre-activation cancellation, activation boundaries, old-client behavior and recovery after activation. Record unresolved authority explicitly. A contract-only design still needs deployment and custody decisions. |

R1 must resolve how an observation becomes an accepted fact before R3 can cause
an irreversible external payment. R2 must remain effective across every restore
path accepted by R4. R5 must include the worst case permitted by R3, not just the
successful examples in R6. A green test or a published specification cannot supply
the separate decisions in R7.

## Proposed change classes and decision roles

These are review-routing suggestions. They assign no individual, team, key or
permission and do not establish that any existing governance mechanism is active.

| Class | Additional decision and compatibility requirements |
| --- | --- |
| Contract or service policy | Contract/application maintainers review bytecode, authority, migration and accounting; custody operators review any signing, limits, capital or refund policy. Decide whether existing derivation facts are sufficient before calling the change contract-only. |
| ZVM derivation change | Protocol maintainers and independent reviewers specify old/new rules and complete vectors; node/client operators review the chain-specific activation and halt/recovery procedure. Committee operators separately decide which state they will accept for custody. |
| Execution-client change | Execution-client and protocol maintainers review semantics, gas/resource costs and cross-client agreement. Operators accept the exact client/node/configuration combination and the incompatible-client behavior. |
| NoM consensus change | NoM protocol maintainers and the relevant governance/validator process must accept the L1 specification and activation. ZVM, bridge and execution-client review cannot grant that authority. |
| Prover or signer policy | Cryptographic reviewers assess the statement, verifier, setup/key assumptions and soundness; wallet/signer maintainers and custody operators accept verification and fallback policies. Decide whether this merely adds a local check or changes on-chain enforcement. |

Some entries need several classes. The register preserves the original suggested
class while identifying decisions that could change it. In particular, an idea
classified as a contract in FUTURE.md must be reclassified if it needs an event,
state commitment, resource rule or system call absent from the selected baseline.

## Proposed review sequence

1. **Define the observation boundary first.** Start with F01's history, successful
   embedded-call outcomes, ordering and recovery model. Reuse that specification
   only after its provenance and finality assumptions are accepted. F02, F04,
   F05, F08, F10 and F11 depend on parts of this work.
2. **Study value and custody separately.** For F02 and F03, specify settlement,
   refund, late-payment and exposure accounting before designing happy paths.
   Resolve independent custody review and operator ownership alongside the
   derivation work. This directory supplies neither review nor custody acceptance.
3. **Choose a proof statement before a proof system.** F13, F15, F20 and F23
   need explicit trusted anchors, public inputs, data availability, history
   transitions and verifier upgrades. An L2 proof and an L1-enforced withdrawal
   remain different deliverables.
4. **Resolve authority before scheduling.** Complete the R7 record for the chosen
   entry, including its dependencies. Rehearse the accepted old/new boundaries
   in an isolated environment before requesting release or activation authority.

This sequence is a proposal for reducing research uncertainty. It gives no start
date, deployment date, production priority or presumed community approval.

## Vector and result boundary

The register supplies candidate scenarios and intended review questions. They are
**not executable vectors**. An exact vector requires all of the following:

- Versioned input and prior-state bytes, their digests and selected source/rules.
- A precise transition trace and expected output bytes/state digest, or a precise
  refusal with an invariant describing which state remains unchanged.
- Chain/domain, ordering and activation context, including the same-height fork
  alternative and restart/re-inclusion behavior where relevant.
- Independent review of the oracle, followed by actual runner/version/platform,
  complete exit/output evidence and a source-bound result record.

If the specification has not decided a behavior, record `DECISION_REQUIRED`.
Do not invent a passing expected value to make the case runnable. Synthetic
examples must be labeled and cannot establish real-chain canonicality, proof
soundness, economic adequacy or hardware crash durability.

## Acceptance record

An accepted research proposal must link its completed record, source comparison,
normative specification, exact vectors, result evidence, independent reviews and
the authorized decision for each applicable role. Record the decision scope,
unresolved risks and any prerequisite proposals. Keep personal/access details
out of public records while retaining enough role and source information to
assess authority privately through the project's accepted process.

Documentation acceptance means the research requirements were reviewed. Protocol
acceptance, implementation conformance, release, deployment and activation each
need their own evidence and decision. All 24 entries remain open until their
individual records establish the required scope.
