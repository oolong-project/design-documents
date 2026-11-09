# OOL-3 threat model, round 2: tea-client-go

Task: OOL-3. Design: `design/OOL-3/architecture-2.md` (round 2). Coding principles at
`145008cdcc23`, chiefly "The Go client treats every server as untrusted" and "Secrets and
credentials". The library is new; it runs inside its caller (the ReARM CLI later, or any Go
program) and talks to whatever TEA servers a TEI or a configured base leads it to.

## Changes in this round

Follows the architecture's round 2 (answers to `review-items/OOL-3/design-reviewer/round-1.md`):

- **F-1**: T4 now names the `/token` redirect path (Basic credentials) and its mitigation, the
  no-redirect token request and the auth transport's off-root removal; tests 14a, 14b, 13a.
- **F-2**: T1 now names slow and stalled responses and the three time bounds; tests 11d,
  26b, 31b.
- **F-3**: T5 states the "under" allowance and dot-segment refusal; tests 9a, 21b, 27b. New
  T13 for the trace hook (debug output) not leaking secrets; test 39a, 39b.
- **F-4**: the table in section 3 gains a child column, so each threat's tests are traced to
  the child that writes them.
- **F-6**: T1 and T7 name the discovery-results cap; tests 26c, 38b.
- **F-9**: T12 names the separate `tools.mod` and the `runtime` graph; test 1.

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
  gzip bombs, endless pagination, endless redirects, many endpoints, a discovery answer with
  thousands of results (each costing a request in `Candidates`), a release with thousands of
  components, and **slow responses**: headers never sent, or a body that stalls or trickles,
  which no byte limit bounds. Mitigation: per-kind byte limits applied after transport
  decoding and checked against `Content-Length` first, for wrapped and `Generated()` calls
  alike (architecture D3, 3.4); per-kind time bounds applied through the request context, so
  they hold with a caller's `http.Client`: `AttemptTimeout` per discovery attempt (a stalled
  endpoint is a failed attempt and failover continues), `CallTimeout` per JSON call,
  `IdleTimeout` for downloads (no headers or no next byte for that long), and the caller's
  context over all (3.4, 3.6, 3.8); redirect cap; page cap on iterators (3.5); total
  discovery attempt bound with capped backoff (3.6); `DiscoveryResults` cap in `Discover` and
  `Candidates`, refused whole (3.6, 3.9); component cap in the flow (3.9). Tests: 4.1 cases 10,
  11a, 11b, 11c, 11d, 20, 26, 26b, 26c, 31, 31b, 33, 38, 38b.
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
  well-known host; putting it in a URL, an error or a log. And the **token request**: Go's
  `http.Client` copies an `Authorization` header set on a request to a redirect target on the
  same host at any port, or on a subdomain (verified on Go 1.26.0, architecture 3.0), so a
  `/token` that answers `302`/`307` to such a URL would receive the API key id and secret.
  Mitigation: credentials bound to one root and decided by the auth transport on every hop,
  redirects included, which also removes any `Authorization` header from a request outside a
  configured root (D5); the token request, for `API.RequestToken` and for `WithAPIKey`, follows
  no redirect at all, so Basic credentials reach exactly one URL and a `3xx` is an error (3.4,
  3.7); never in URLs; errors and trace events carry no header values and only a sanitised
  server `message` (3.6, 3.7); API-key Basic only to `<root>/v<version>/token`, only after a
  Bearer challenge from that root, at most one recovery. Tests: 4.1 cases 12, 13a, 13b, 13c,
  14a, 14b, 30, 39a; 4.2 case 8.
- **T5, downgrade or interception (A2, A3; B5).** An `http` endpoint in a well-known document or
  in `servers[]`, an https-to-http redirect, a userinfo URL, a dot-segment path that escapes an
  allowed base. Mitigation: https only; `http` only for a URL under a base the caller
  configured (same scheme, host and port, path by segments), never widened by anything a server
  sends; dot segments refused; TLS verification left to the `http.Client`, no option to disable
  it (D4, 3.4). Residual, stated in architecture section 5: on an origin the caller declared
  plain-http, a document chooses the path; that origin is already unprotected. Tests: 4.1
  cases 8, 9a, 10, 21b, 23, 24, 27, 27b, 30.
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
  and allowance rules and credentials do not follow them (D4, D5); at most `DiscoveryResults`
  results are accepted, so `Candidates` makes a bounded number of requests to hosts a document
  names (3.6, 3.9). That a TEI's own domain decides where its data comes from is the
  protocol's trust model, not a flaw the client can fix. Tests: 4.1 cases 24, 26, 26c, 27, 38b.
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
- **T12, supply chain of the library.** Runtime dependencies pinned in `go.mod` / `go.sum`; the
  generator pinned in a separate `tools.mod` / `tools.sum`, so its graph never enters a
  consumer's build; the `runtime` module's own requirements do enter a consumer's module graph
  as minimum versions (stated, architecture 3.1); generated output checked in and drift-checked
  in CI; spec files and `spec/LICENSE` pinned by blob id; CI actions pinned by commit SHA with
  `permissions: {}` at the top (3.11). Tests: 4.1 case 1 (including no `tool` line in `go.mod`),
  the CI drift check.
- **T13, secrets through the debug hook (A2; caller's logs).** `WithTrace` exists so a CLI can
  print what it requests; an event that carried a header, a query string (presigned artifact
  URLs hold secrets there) or a credential would put them in a terminal or a log. Mitigation:
  events carry scheme, host, port and path only, plus status and reason text that is already
  sanitised (architecture 3.4). Tests: 4.1 cases 39a, 39b, 14b.

## 3. Mitigations that need a test

All are in the architecture's test list (section 4); no test was added here that the design did
not need. The child column follows the architecture's split (section 5): 3a foundation, 3b
discovery, 3c downloads and flow.

| Threat | Tests (architecture section 4) | Child that writes them |
| --- | --- | --- |
| T1 | 4.1: 10, 11a, 11c, 11d, 20 / 11b, 26, 26b, 26c / 31, 31b, 33, 38, 38b | 3a / 3b / 3c |
| T2 | 4.1: 3, 4, 7, 15, 16 / 17 | 3a / 3b |
| T3 | 4.1: 28, 29, 30, 31, 32, 34; 4.2: 7, 8 | 3c |
| T4 | 4.1: 12, 13a, 14a, 39a / 13b / 13c (= 30), 14b; 4.2: 8 | 3a / 3b / 3c |
| T5 | 4.1: 8, 9a, 10 / 21b, 23, 24, 27, 27b / 30 | 3a / 3b / 3c |
| T6 | 4.1: 4, 18 / 21, 23, 24 | 3a / 3b |
| T7 | 4.1: 24, 26, 26c, 27 / 38b | 3b / 3c |
| T8 | 4.1: 15 | 3a |
| T10 | 4.1: 32 | 3c |
| T11 | 4.1: 6 | 3a |
| T12 | 4.1: 1; CI generator drift check | 3a |
| T13 | 4.1: 39a / 39b / 14b | 3a / 3b / 3c |

Case 17 fuzzes the discovery and well-known decoders, which OOL-3b builds, so 3b writes it.
