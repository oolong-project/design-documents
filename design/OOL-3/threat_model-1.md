# OOL-3 threat model, round 1: tea-client-go

Task: OOL-3. Design: `design/OOL-3/architecture-1.md` (round 1). Coding principles at
`145008cdcc23`, chiefly "The Go client treats every server as untrusted" and "Secrets and
credentials". The library is new; it runs inside its caller (the ReARM CLI later, or any Go
program) and talks to whatever TEA servers a TEI or a configured base leads it to.

## 1. What is exposed

Assets:

- **A1, the caller's process and host**: memory, disk (download targets), CPU, terminal (error
  text a CLI prints).
- **A2, credentials**: a bearer token or an API key id and secret the caller configured for one
  TEA root.
- **A3, artifact integrity**: SBOMs, VEX and other documents the caller will act on.
- **A4, the caller's network position**: what the client can reach that an outsider cannot
  (relevant for server-side consumers).

Trust boundaries:

- **B1, TEI input**: from a user, a label, a file; untrusted text.
- **B2, the TEI's domain**: serves `.well-known/tea`; untrusted (any domain can be in a TEI).
- **B3, API servers**: the well-known endpoints and the `servers[]` roots; untrusted, may be
  hostile or compromised.
- **B4, external artifact locations** (`url`, `signatureUrl`, redirect targets): untrusted and
  outside the TEA server's control.
- **B5, the network path**: TLS is the only protection.

## 2. Threats and mitigations

- **T1, resource exhaustion by a hostile server (A1; B2, B3, B4).** Endless or huge bodies,
  gzip bombs, endless pagination, endless redirects, many endpoints, a release with thousands of
  components. Mitigation: per-kind byte limits applied after transport decoding and checked
  against `Content-Length` first (architecture D3, 3.4); redirect cap; page cap on iterators
  (3.5); total discovery attempt bound with capped backoff (3.6); component cap in the flow
  (3.9); every call under the caller's context. Tests: 4.1 cases 10, 11, 20, 26, 31, 33, 38.
- **T2, crash or confusion on malformed input (A1; B1, B3).** Malformed TEIs, invalid JSON,
  `null`, wrong types, missing bodies, unknown codes. Mitigation: `tei.Parse` is a strict
  hand-written grammar with length caps (3.3); every typed call maps an undecodable 2xx to
  `*UnexpectedResponseError`; no panic on server input (3.5, 3.6); unknown enum values are kept
  and refused only where acted on (D8). Tests: 4.1 cases 3, 4, 7, 15, 16, 17 (fuzz).
- **T3, forged or swapped artifact bytes (A3; B4, B5, B3 partly).** A tampered mirror behind an
  external `url`, a redirect to other bytes, a `latest` race, content-coding tricks. Mitigation:
  checksums from the revision's metadata are verified while streaming; every supported listed
  checksum must match; at least one strong algorithm required (SHA-1 only and BLAKE3 only fail
  closed); malformed `algValue` refused; the versioned endpoint is used so bytes and checksums
  belong to the same revision; only transport-negotiated gzip is decoded, other codings refused;
  `DownloadArtifactToFile` renames into place only after verification (3.8). Residual: a
  compromised TEA server can publish matching checksums for forged bytes; signatures would be
  the answer, and the spec does not define their technology (out of scope, stated in README).
  Tests: 4.1 cases 28, 29, 30, 31, 32, 34; 4.2 cases 7, 8.
- **T4, credential leakage (A2; B3, B4).** Sending the token to an external `url`, across a
  redirect to another origin or a subdomain, to a `servers[]` root on another host, to the
  well-known host; putting it in a URL, an error or a log. Mitigation: credentials bound to one
  root and added by the transport per request, redirects included (D5); never in URLs; errors
  carry no header values and only a sanitised server `message` (3.6, 3.7); API-key Basic only to
  `<root>/v<version>/token`, only after a Bearer challenge from that root, at most one recovery.
  Tests: 4.1 cases 12, 13, 14, 30; 4.2 case 8.
- **T5, downgrade or interception (A2, A3; B5).** An `http` endpoint in a well-known document or
  in `servers[]`, an https-to-http redirect, a userinfo URL. Mitigation: https only; `http` only
  for a base byte-equal to one the caller configured; TLS verification left to the `http.Client`,
  no option to disable it (D4, 3.4). Tests: 4.1 cases 8, 9, 10, 23, 24, 27, 30.
- **T6, request smuggling through identifiers (A4; B1, B3).** A TEI or server-provided value
  that rewrites the request path or query: `%2F` in an id, a version with `/` or `?`, a
  `rootUrl` with query or userinfo, a uuid with path characters. Mitigation: TEI id grammar
  refuses `/`, `%2F`, `?`, `#` (3.3); query encoding applied once; versions must match the SemVer
  pattern, `rootUrl` and well-known `url` the spec patterns, before they enter a URL (3.6); path
  parameters are typed uuids and ints in the generated client (3.5). Tests: 4.1 cases 4, 18, 21,
  23, 24.
- **T7, discovery steering (A1, A3; B2, B3).** A hostile well-known document that lists many
  endpoints to slow the client, or a discovery answer that sends the client to another host for
  the release data. Mitigation: attempt bound (T1); `servers[]` roots go through the same https
  and allowance rules and credentials do not follow them (D4, D5). That a TEI's own domain
  decides where its data comes from is the protocol's trust model, not a flaw the client can fix.
  Tests: 4.1 cases 24, 26, 27.
- **T8, terminal and log injection (A1; B3).** A server `message` or unknown error code with
  control characters or megabytes of text, printed by a CLI. Mitigation: codes truncated to 64
  bytes, messages to 200 bytes, control characters removed; no response body otherwise reaches
  an error (3.6). Test: 4.1 case 15.
- **T9, SSRF for server-side consumers (A4; B1, B2).** A service that resolves user-supplied TEIs
  lets users make it fetch https URLs of their choosing, internal addresses included. Mitigation
  in the library: https only, bounded reads, no credentials off-root. Address filtering is left to
  the consumer's `http.Client` dialer, documented in README. Accepted residual; no test (nothing
  in the library to test).
- **T10, path traversal on download (A1; B3, B4).** A `Content-Disposition` file name with `../`.
  Mitigation: the library never derives a path from a server; the caller passes the target path;
  temp file in the target's directory, mode 0600 (3.8). Test: 4.1 case 32.
- **T11, misresolution through the legacy converter (A3; B1).** A converted `urn:tei:` resolving
  to a different release. Mitigation: conversion is explicit and never implicit; the result is a
  syntactically checked TEI; a mismatch yields `OBJECT_UNKNOWN`, not another release, since
  identity stays the server's exact-string comparison (3.3). Test: 4.1 case 6 (correct
  conversion); nothing further.
- **T12, supply chain of the library.** Generator and runtime dependencies pinned in `go.mod` /
  `go.sum`; generated output checked in and drift-checked in CI; spec file pinned by blob id; CI
  actions pinned by commit SHA with `permissions: {}` at the top (3.11). Tests: 4.1 case 1, the CI
  drift check.

## 3. Mitigations that need a test

All are already in the architecture's test list; no test was added here that the design did not
need:

| Threat | Tests (architecture section 4) |
| --- | --- |
| T1 | 4.1: 10, 11, 20, 26, 31, 33, 38 |
| T2 | 4.1: 3, 4, 7, 15, 16, 17 |
| T3 | 4.1: 28, 29, 30, 31, 32, 34; 4.2: 7, 8 |
| T4 | 4.1: 12, 13, 14, 30; 4.2: 8 |
| T5 | 4.1: 8, 9, 10, 23, 24, 27, 30 |
| T6 | 4.1: 4, 18, 21, 23, 24 |
| T7 | 4.1: 24, 26, 27 |
| T8 | 4.1: 15 |
| T10 | 4.1: 32 |
| T11 | 4.1: 6 |
| T12 | 4.1: 1; CI generator drift check |
