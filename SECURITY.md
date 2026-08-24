# Security Policy

## Threat surface

iam is a read-only consumer of mihi probes. It writes a fixed-shape
report to stdout and exits. iam does not spawn processes, open
network sockets, or write to the filesystem. The realistic threats:

- **mihi return-data abuse** — if mihi returns adversarially-large
  or malformed buffers (e.g., a probe parsed a hostile `/proc`
  file), iam's display formatter must still bound writes.
  *Enforced since v1.1.7*: every value column is capped at
  `IAM_VALUE_MAX` (256 bytes) inside `iam_copy_value`, so an
  over-long value truncates rather than costing a required line.
  Before v1.1.7 the write bounds held but the *line* did not — see
  F-003 in [`docs/audit/2026-08-23-v1.1.7-audit.md`](docs/audit/2026-08-23-v1.1.7-audit.md).
- **TTY escape injection** — if any mihi-returned string contains
  ANSI escape sequences (from a maliciously-crafted `/etc/os-release`,
  hostname, etc.), iam must sanitize before writing to stdout.
  *Enforced since v0.9.0* (F-001), extended at v1.1.7 to cover DEL
  (0x7F). C1 (0x80–0x9F) is deliberately not filtered: on a UTF-8
  terminal those bytes are multibyte continuation bytes, so filtering
  them would corrupt legitimate non-ASCII values without protecting
  against anything.

Note that a probe value is bounded at **copy** time, not at scan
time: `strlen` on a mihi-returned pointer still runs to the first NUL
(F-002, INFO). That invariant is mihi's to guarantee.

iam's surface is small; most security work lives upstream in mihi.

## Reporting Vulnerabilities

Report vulnerabilities privately to **security@agnos.dev**. Do not
open public issues for security bugs.

We will acknowledge receipt within 48 hours and provide a timeline
for a fix.
