# Security Policy

## Threat surface

iam is a read-only consumer of mihi probes. It writes a fixed-shape
report to stdout and exits. iam does not spawn processes, open
network sockets, or write to the filesystem. The realistic threats:

- **mihi return-data abuse** — if mihi returns adversarially-large
  or malformed buffers (e.g., a probe parsed a hostile `/proc`
  file), iam's display formatter must still bound writes.
- **TTY escape injection** — if any mihi-returned string contains
  ANSI escape sequences (from a maliciously-crafted `/etc/os-release`,
  hostname, etc.), iam must sanitize before writing to stdout.

iam's surface is small; most security work lives upstream in mihi.

## Reporting Vulnerabilities

Report vulnerabilities privately to **security@agnos.dev**. Do not
open public issues for security bugs.

We will acknowledge receipt within 48 hours and provide a timeline
for a fix.
