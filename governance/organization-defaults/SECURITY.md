# Security reporting guidance

This is a candidate organization fallback for repositories without a local policy.
It requires explicit adoption and verification of the actual private reporting
route. Existing repository security guidance takes precedence.

For a suspected vulnerability, check the affected repository's **Security** tab
and use **Report a vulnerability** when enabled. GitHub's
[private reporting instructions](https://docs.github.com/en/code-security/how-tos/report-and-fix-vulnerabilities/report-privately)
describe that route. Adding this policy does not enable the feature.

Provide the affected source, impact, preconditions and an owned synthetic
reproduction through a verified private channel. Keep credentials, signing
material, account inventories, personal paths and unrelated operator data out
of attachments. A pseudonym is sufficient; review advisory credit and publication
choices before relying on anonymity.

If no verified private route is available, retain details privately while an
authorized maintainer establishes a channel or escalation decision. Public
contact requests must contain no vulnerability details. Do not open a public
finding or patch PR as a substitute for unavailable private intake.

Response ownership, affected-version scope, support commitments, bounty terms
and disclosure timing must be stated by the affected repository's authorized
maintainers. This fallback creates none of those commitments. Private intake,
patch sharing, additional recipients and public disclosure are separate decisions.
