# Research proposal record template

**Template status: unfilled.** Copy this into a separately reviewed proposal.
Every unresolved field starts as `UNRESOLVED`; a heading or checklist is not
evidence that its requirement has been met. Apply all
[R1-R7 requirements](README.md#requirements-shared-by-every-proposal) and the
relevant [register entry](PROPOSALS.md).

## Record and scope

- Proposal ID, register IDs, revision and decision sought: `UNRESOLVED`.
- Research question and proposed behavior, including exclusions: `UNRESOLVED`.
- Proposed change classes, affected repositories and prerequisites: `UNRESOLVED`.
- Exact baseline source, specification, dependency and vector commits/digests:
  `UNRESOLVED`.
- Difference from the baseline and overlapping proposals/implementations:
  `UNRESOLVED`.

## R1: accepted history and finality

Specify chain/genesis and authenticated history authority, complete input and
successful-outcome evidence, historical state, confirmation/finality rule,
conflicting-history behavior and source unavailability. Identify what is verified
locally and what remains trusted. Current value: `UNRESOLVED`.

## R2: canonical bytes and replay

Supply an exact, versioned encoding and event/operation identity with domains for
chain, source, protocol, destination and epoch as applicable. Define duplicate,
concurrent, restored, removed/re-included and cross-chain cases. Identify durable
state and the accepted restore boundary. Current value: `UNRESOLVED`.

## R3: transitions and accounting

Specify states, prerequisites, ordered transitions, irreversible effects, failure
and compensation, authorization, amounts/fees/rounding, backing, resource/queue
bounds and invariants. Cover pending work during upgrades and key changes.
Current value: `UNRESOLVED`.

## R4: threats and residual trust

Name adversarial capabilities and trust in validators, RPC, clients, contracts,
provers, signers, operators and governance. Analyze threshold compromise,
censorship, withholding, key loss, crash/restore and dependency failure. Give
separate safety and liveness arguments and independent-review scope.
Current value: `UNRESOLVED`.

## R5: economics and capacity

Provide an exposure/liability bound, capital and liquidity assumptions, incentive
and griefing analysis, fee/funding ownership, capacity limits and worst-case
storage/compute growth. Bind quantitative evidence to source, workload, platform
and method; record unmeasured values explicitly. Current value: `UNRESOLVED`.

## R6: exact vectors and results

Create a versioned vector manifest. Each case must bind source/rules and prior
state, exact input bytes/digests, chain/order/activation context and exact expected
outputs/state digest or refusal invariant. Cover the register's candidate cases
and additional cases arising from the chosen design. For an undecided oracle use
`DECISION_REQUIRED`, without a claimed successful result.

Keep vector definitions and independent oracle review separate from runner
results. A result must identify the vector revision, implementation commit,
runner/platform, actual command, complete output/exit, skips and limitations.
Distinguish synthetic controls, native execution, hosted CI, independent review,
integration and live-chain evidence. Current value: `UNRESOLVED`.

## R7: decision, compatibility and activation

Identify proposed decision roles and verify their authority through the accepted
project process. Record required independent reviews, dependencies, accepted
source/vectors, client/configuration/bytecode bindings and compatibility cases.
Do not insert personal contact, credential, access or private operational data.

Specify pre-activation cancellation, both sides of the activation boundary,
old/unknown-client behavior, restart/replay, in-flight work, halt and recovery.
Record separate decisions for specification, implementation conformance, release,
deployment and chain activation. Current value: `UNRESOLVED`.

## Decision and remaining gates

- Decision scope, reviewed revision and authorized role record: `UNRESOLVED`.
- Accepted assumptions, evidence and unresolved risks: `UNRESOLVED`.
- Dependencies and independent-review decisions: `UNRESOLVED`.
- Conditions invalidating this decision when source, keys, rules or chain change:
  `UNRESOLVED`.
- Next action and owner role, without inferred release/activation permission:
  `UNRESOLVED`.

The default outcome for an incomplete record is `RESEARCH_OPEN`. Accepting the
record as documentation does not activate its protocol or authorize use of funds.
