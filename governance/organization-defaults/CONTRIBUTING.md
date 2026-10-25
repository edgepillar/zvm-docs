# Contributing to ZVM repositories

This file is a candidate organization default. It becomes guidance only after
explicit maintainer adoption; repository-specific instructions take precedence.

## Choose the right channel

Use ordinary issues and pull requests for non-sensitive work. For a suspected
vulnerability, follow the affected repository's security policy and verified
private reporting route. Keep finding details out of public issues and PRs.
Use minimal synthetic examples and English public text. Omit personal paths,
credentials, wallet/signing material, account inventories and private diagnostics.

## Prepare a scoped change

Identify the repository, exact base/source revision, concrete problem and intended
behavior. Check existing issues and open PRs for overlap. Keep unrelated edits,
licenses, historical evidence and incomplete results intact. State any unresolved
product, protocol, custody or compatibility decision before claiming implementation.

Pin changed specification/vector, contract, dependency and configuration inputs.
Explain the source relationship between coupled repositories and submodules.
For consensus, genesis, fork, committee, financial or signing changes, identify
the additional independent review and real integration/activation gates.
Do not infer authority from a proposed document or a successful local test.

## Evidence and review

Run checks appropriate to the changed behavior. Record the exact source, actual
command/result and scope, distinguishing skipped or interrupted work. Separate
source inspection, local fixtures, hosted CI, integration, artifacts, deployment
and live operation. Reuse unchanged verified inputs and bounded scratch instead
of repeating expensive builds without a source or evidence reason.

A PR should explain the problem, resulting behavior, validation and remaining
limits. Required reviewers and approvers must be identified through the accepted
repository policy; unassigned or disputed authority remains unresolved. Authors
cannot provide independent review of their own work. A Draft communicates that
review or qualification remains open and does not request release or activation.

Source merge, version publication, environment deployment and chain activation
are separate authorized decisions. Passing checks or write access alone do not
establish any of the later decisions.
