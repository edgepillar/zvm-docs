# Private disclosure and escalation proposal

**Status: proposed, not adopted.** The existing [repository security policy](../SECURITY.md)
remains the published reporting guidance. This document assigns no responder,
promises no response deadline and changes no private advisory or reporting setting.

## Reporter path

Use the affected repository's published policy and confirm that its private
reporting route is available before supplying details. GitHub's
[private reporting instructions](https://docs.github.com/en/code-security/how-tos/report-and-fix-vulnerabilities/report-privately)
explain the platform route. A `SECURITY.md` file and enabled private reporting
are separate conditions. A generic organization policy cannot prove that every
repository has a working confidential intake.

Use a pseudonymous account if desired. Share only the information needed for
triage: the affected repository and exact source, impact, preconditions and an
owned synthetic reproduction. Review attachments and commit metadata before
sharing. Keep credentials, account inventories, signing material, personal paths,
operator logs and unrelated private data out of reports and public artifacts.
An advisory can display reporter credit; review credit and publication choices
with the maintainer instead of assuming anonymity.

If the stated route is absent or unanswered, preserve the report privately.
An authorized coordinator must establish a verified private alternative or an
escalation decision. A contact request can omit all finding details. Do not
turn unavailable private reporting into a public issue containing the finding,
or send the same private payload to unrelated repositories or services.

## Maintainer response and decisions

The proposed security coordinator verifies the affected scope and authorized
participants, acknowledges the private intake when available, and records
triage, remediation, review and disclosure decisions in that confidential channel.
Maintainers must choose response ownership, escalation contacts and any cadence;
this candidate does not claim an existing on-call service or service-level target.

Keep the original report, source pins, actual failing/passing evidence and exact
patch scope. A proposed fix, local test result, accepted report or draft advisory
is a separate state. Preserve incomplete results and unresolved impact. Evaluate
adjacent components privately before selecting a coordinated scope.

Agree on advisory content, credit, affected/fixed versions and release timing
with the authorized maintainers. Consent to an initial private report does not
authorize a later patch upload, public PR, additional recipient or disclosure.
Only publish a reviewed, sanitized advisory and qualified fix after the relevant
authorization and release decision. Do not disclose confidential technical details
in ordinary public issue or PR templates.
