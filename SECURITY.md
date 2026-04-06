# Security Policy

## Scope

bench2bash-skills is a collection of Markdown-based Claude AI skill files.
It contains no executable code, no authentication logic, and no network
services. The attack surface is essentially zero.

The only realistic security concern is **prompt injection** — malicious
content embedded in reference files that could cause a Claude instance
using these skills to behave unexpectedly.

## Reporting a vulnerability

If you believe you have found a prompt injection issue or any other
security-relevant problem in this repository:

1. **Do not open a public GitHub issue.**
2. Email **tamoghna.das@lboro.ac.uk** with the subject line:
   `[bench2bash-skills] Security`
3. Describe what you found and how it could be exploited.

You will receive a response within 7 days. If the issue is confirmed,
a fix will be released and you will be credited (unless you prefer
to remain anonymous).

## Supported versions

Only the latest release on `main` is actively maintained.
