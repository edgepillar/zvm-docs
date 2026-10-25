# Release and activation decision proposal

**Status: proposed, not adopted.** This is a decision record outline for authorized
maintainers and operators. It creates no release, signature, deployment or chain
change and appoints no release or activation authority. The dated
[upgrade plan](../planning/UPGRADES.md) and [committee runbook](../operations/COMMITTEE.md)
retain their existing evidence limits.

## Separate decisions

| Decision | Evidence to review | Authority that must be established |
| --- | --- | --- |
| Accept source | Exact base/head, scope, independent reviews, selected compatibility/security results and unresolved failures | Repository maintainer and the required protocol/custody/security reviewers |
| Qualify outputs | Source/dependency/toolchain pins, immutable platform outputs, checksums, provenance, SBOM and independently verified build correspondence | Release maintainer with an independent artifact verifier |
| Publish a version | Accepted source and outputs, supported/unsupported platform statement, release notes, disclosures and update/rollback limits | Explicitly authorized release maintainer |
| Deploy an environment | Accepted release, exact configuration/genesis/epoch, deployment record, identity/code observations and rehearsed recovery | Authorized environment operator |
| Activate chain rules | Agreed specification/vectors, actual activation authority, old/new-node and engine rehearsals, custody/economic review and coordinated operator acceptance | The separately accepted chain governance and operators |

Source acceptance does not establish that an output was built from it. An artifact
signature identifies its signing authority only under an accepted trust policy.
Publication does not establish deployment, and deployment does not establish
chain activation. Committee custody or GitHub write access must not be treated
as activation authority without an accepted decision for that chain and scope.

## Reviewable release record

Retain the full source and dependency revisions, artifact/image digests, compiler
and platform identity, command/run/attempt and actual exits. Identify each test's
scope, skips, failures and incomplete output. Check that the accepted output is
the same immutable artifact proposed for distribution; do not rebuild a different
artifact after qualification and infer correspondence from a version name.

Record independent review decisions by their authorized roles. Keep personal
identities, access details, credentials, wallet material and private findings in
the appropriate private record. A public acceptance summary can name scope,
source/artifact pins, accepted role attestations and remaining limits without
exporting those private details.

Release notes must distinguish implemented behavior, historical results, pending
rehearsals and unsupported claims. Retain a clear hold when original evidence,
required review, real platform coverage, distribution provenance or authority is
missing. A passing unit suite cannot stand in for recipient accounting, custody
review, real integration or operation across a fork boundary.

## Activation and recovery

Before a chain change, authorized participants must agree on the applicable
chain/configuration/epoch, rule version and boundary, deployed source/engine pair,
notice/governance mechanism and actual readiness evidence. Exercise old/new-node
behavior before, at and after the boundary and record independent agreement.
Do not infer acceptance from a dated plan or edit an operating chain to make a
documentation example appear implemented.

Recovery must state what is reversible and what requires a new governed change.
Preserve original operation hashes and uncertain outcomes; do not re-sign,
replace or settle an operation automatically to make a rehearsal pass. Record
the actual incident/hold decision and operator handoff. This proposal does not
select activation windows, custody thresholds, release keys or live-chain actions.
