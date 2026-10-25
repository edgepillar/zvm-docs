# Contribution and release governance proposal

**Status: proposed, not adopted.** This directory provides a reviewable candidate
for organization guidance and repository templates. It assigns no person or team,
grants no permission, changes no organization setting and authorizes no release,
deployment, signing operation or chain activation. Source correspondence and a
passing documentation job do not establish governance acceptance.

## Source and authority

The reviewed Docs source is `35c5be8a59059a9e1f3aab40c980ef6cd18cf0be`.
Its [security policy](../SECURITY.md) directs reports to the affected repository's
private reporting channel. [UPGRADES](../planning/UPGRADES.md) describes a dated
proposal with unresolved operator decisions. The [committee runbook](../operations/COMMITTEE.md#roles)
describes ceremony roles and historical rehearsal; it also retains an external
review gap. Those sources do not identify currently authorized review teams,
release approvers or activation authority. The checker manifest pins their bytes.

A public metadata observation on 2026-10-25 found the organization `.github`
repository empty, without a default branch. Publicly visible ruleset responses
for that repository and Docs contained no entries. This observation does not
establish private rulesets, branch protections, bypass permissions, team access
or actual enforcement. An authorized maintainer must verify those separately.

The [review role record](review-roles.json) names responsibilities and areas,
with every assignment explicitly unassigned. It is a proposal, not a GitHub
permission record. No `CODEOWNERS` file is installed or generated from it.

## Review responsibilities

| Role | Responsibility before acceptance |
| --- | --- |
| Organization administrator | Verify effective access, rule enforcement, bypasses, private reporting and the approved mapping from responsibilities to authorized teams |
| Repository maintainer | Own scope and compatibility decisions, identify required reviews, and record the exact accepted source and unresolved limits |
| Protocol reviewer | Review specification/vector, derivation, fork/genesis and chain-identity changes, including old/new-source disagreement |
| Custody reviewer | Independently assess committee/signing, replay persistence, recovery and value/accounting changes |
| Security coordinator | Maintain a verified private response route and coordinate confidential triage, remediation and disclosure decisions |
| Release maintainer | Bind qualification to source and immutable build outputs, and obtain the separately required publication authority |
| Operator | Verify the intended deployment/chain/epoch and accepted activation/recovery procedure before operating a change |

An author may prepare evidence but cannot supply an independent review of the
same work. Repository access alone does not establish protocol, custody, release
or activation authority. A green check, merged PR or published artifact cannot
substitute for the other decisions. Unassigned or disputed required authority
remains unresolved; maintainers must not infer approval from silence.

## Candidate organization defaults

The following files are prepared under `organization-defaults/` for an explicit
maintainer adoption change in the organization `.github` repository:

| Candidate | Destination relative to the organization repository root |
| --- | --- |
| [Contribution guide](organization-defaults/CONTRIBUTING.md) | `CONTRIBUTING.md` |
| [Security guidance](organization-defaults/SECURITY.md) | `SECURITY.md` |
| [Support guidance](organization-defaults/SUPPORT.md) | `SUPPORT.md` |
| [PR template](organization-defaults/.github/PULL_REQUEST_TEMPLATE.md) | `.github/PULL_REQUEST_TEMPLATE.md` |
| [Non-sensitive bug template](organization-defaults/.github/ISSUE_TEMPLATE/bug_report.md) | `.github/ISSUE_TEMPLATE/bug_report.md` |
| [Proposal template](organization-defaults/.github/ISSUE_TEMPLATE/proposal.md) | `.github/ISSUE_TEMPLATE/proposal.md` |
| [Template chooser](organization-defaults/.github/ISSUE_TEMPLATE/config.yml) | `.github/ISSUE_TEMPLATE/config.yml` |

These nested files are inactive here. They contain no assignees, labels requiring
pre-existing configuration, individual identities, secret values or hard-coded
product security-report destination. The chooser links to GitHub's reporting
instructions; it does not redirect all product reports to an organization inbox.
Existing repository security policies and templates need individual review.

GitHub [default community files](https://docs.github.com/en/communities/setting-up-your-project-for-healthy-contributions/creating-a-default-community-health-file)
apply only where repository-specific files do not take precedence. A repository's
own issue-template configuration replaces the default issue-template directory.
Defaults are not copied into every clone. `CODEOWNERS`, release authority and
organization settings need separate configuration; a default license is not
provided by this mechanism.

## Adoption sequence

1. An authorized maintainer accepts or revises the responsibility model and
   records an approved source revision. Resolve the private escalation route,
   response ownership and release/activation decision process before relying on it.
2. The organization maintainer prepares the initial branch in the empty `.github`
   repository. Adopt the selected candidate files through a separately reviewed
   change, preserving any new repository-specific policy that appeared meanwhile.
3. Review each repository's overrides and confirm its non-sensitive issue flow
   and private security-report route. Record what actually inherits the defaults.
   Merely creating `SECURITY.md` does not enable private vulnerability reporting.
4. Verify real team/write assignments and effective branch/tag rules, including
   code-owner approval where required, stale-review handling, required checks and
   bypass permissions. Keep access details private; retain a sanitized acceptance
   record with the verified scope and remaining limits. Do not invent team handles.
5. Rehearse the contribution, confidential disclosure and release paths with
   benign examples. Record the source, actor role, decision and actual outcome.
   Publishing an advisory or release still needs its own authorization.

An adoption record must identify the selected source revision, repositories and
overrides, verified role authority, rules/configuration scope, actual rehearsal
results, approver decision and unresolved gates. A template checkbox is a
declaration until independently checked.

See the [private disclosure proposal](DISCLOSURE.md) and
[release and activation proposal](RELEASE.md). G01 remains incomplete until
authorized maintainers verify and adopt the real governance and enforcement.
