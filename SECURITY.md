# Security policy

## Supported versions

| Version | Supported |
| --- | --- |
| 0.1.0 | Yes |

## Reporting a vulnerability

Report a suspected vulnerability privately. Do not open a public issue, a pull request or a discussion for it.

- Contact: GitHub private vulnerability reporting: https://github.com/ASCENTIAN/prime-runtime/security/advisories/new
- Include: the version (`prt version`), your platform, the steps to reproduce, the expected and the observed behaviour, and the impact you believe it has.
- Do not include secrets, private keys or the content of files you did not intend to share. A redacted health report is enough for most reports.

## What we do

- We acknowledge a report, then confirm whether it is in scope.
- We fix confirmed issues in a new version and describe them in the changelog.
- We credit the reporter if they want to be named.

No response-time commitment is made for 0.1.0.

## Scope

In scope: the launcher, the kernel, the client sandbox, the service unit, the catalog and pin checks, the journal and receipts, and the release artifacts.

Out of scope, and documented as limits: writes outside the managed directory by a sandboxed client, network access of a sandboxed client, processes started outside the launcher, root, and platforms other than Linux x86_64. See [SECURITY_BOUNDARIES](docs/SECURITY_BOUNDARIES.md).

A report that describes one of these limits, without showing that the documented behaviour is wrong, is handled as a documentation issue.
