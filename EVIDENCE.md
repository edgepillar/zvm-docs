# Source and benchmark evidence

## Current source references

This inventory reviews Docs `35c5be8a59059a9e1f3aab40c980ef6cd18cf0be`. The [specification](spec/SPEC.md),
[decisions](spec/DECISIONS.md) and the externally supplied Core encoding vectors
are exact byte inputs in the [checker manifest](tools/doc-check/manifest.json).
Core vectors are pinned to `29e6d9b30be94de3772c850c7a1e9811585a8833`. The specification's selected vector
heading and required JSON sections are checked without running a node, vector
generator, protocol test or performance measurement. Matching source bytes do
not establish semantic conformance, canonical chain authority or activation.

The portable checker source is pinned by [its provenance](tools/doc-check/UPSTREAM.json).
Every tracked Markdown file is inventoried. Local links and anchors, selected
references and typed evidence declarations run in a dedicated documentation job.
Remote link availability is separate and is not tested by this offline tool.

## Historical benchmarks

The [method](benchmarks/BENCHMARK.md#evidence-status) and ten files under
[benchmarks/results](benchmarks/results/) are retained historical narratives and
summary tables. The selected complete top-level source inventory contains no
raw `.json`, `.jsonl` or `.ndjson` benchmark runs. References to `bench/results`,
`bench/fleet` and earlier deployment layouts describe prior runs; those paths
do not supply retained raw evidence in this repository snapshot.

Each summary has an explicit `unavailable` benchmark declaration. Its original
numeric content is preserved. Reconstructing samples from a table, re-rendering
a report or agreeing with a headline cannot recover the original measurement.
No new benchmark was run. This is a snapshot-scoped evidence gap; operator
storage, other history and hosted artifacts have not been exhaustively searched.

For independently qualified reproduction, retain exact source and dependency
revisions, chain/config/genesis identity, runtime/platform, build and method
provenance, the actual command, run/attempt, complete exits and per-run raw input
digests. Keep failed or partial phases distinct, define offered/included/sustained
and latency populations, and state uncertainty. Apparent bytes, allocated bytes,
sparse maps and physically reclaimed space are different measures. Preserve
original samples and correction history before calculating a new aggregate.

## Plans and operations

[FUTURE](planning/FUTURE.md) contains proposals; [UPGRADES](planning/UPGRADES.md)
describes fork coordination. The former `HANDOFF.md` target is absent from the
selected snapshot. Neither a proposal nor that missing handoff proves accepted
implementation. Operations pages are instructions requiring their own reviewed
source, deployment, operator and rollback qualification. They were not executed
by documentation CI. Configurations must use operator-supplied local values;
public examples contain no mailbox identity or personal workstation path.

## Governance proposals

The [governance candidate](governance/) adds proposed review responsibilities,
private disclosure and release/activation decisions, with inactive organization
default templates. The selected existing security policy, upgrade proposal and
committee runbook are pinned separately. The [role record](governance/review-roles.json)
leaves all actors unassigned and authority/enforcement unestablished.
Documentation checks cannot verify actual private rules, responsible teams,
independent review, policy adoption or permission to publish or activate.

## Qualification limits

The checker validates selected bytes and record consistency. Human review still
owns semantics, source authority, English and privacy. No raw operator logs,
account inventories, endpoints, mailbox identities or credential values should be
added to public evidence. Automated guards are incomplete and do not establish
anonymity. Docs adoption does not complete raw benchmark provenance recovery,
independent maintainer acceptance, G02 or any production gate.
