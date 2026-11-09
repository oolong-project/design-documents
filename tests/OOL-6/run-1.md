# OOL-6 test run 1: tea-client-go downloads, checksum verification, flow helper, API-key auth, README (PR #3)

Task: OOL-6 (OOL-3c, child 3 of OOL-3). Verdict: REJECTED. The suites, the integration tests
(harness and in-cluster server) and the local live check all pass, every 4.1/4.2 case the task
names has a test, and the README matches the API. Two defects and three test gaps are open:
`VerifyChecksums` with `MaxBytes` set to the largest int64 verifies forged bytes (T-1, a T3
fail-open), the same value panics a download (T-2), and three mutants of behaviour the coder's
departures and the design rely on survive the suite (T-3 to T-5).

Inputs: `impl/OOL-6/notes-1.md` (DETAILED_DESIGN round 1), the carried
`design/OOL-3/architecture-2.md` (ARCHITECTURE round 2: 3.7, 3.8, 3.9, 3.10, 3.11; 4.1 cases
13c, 14b, 28-38b; 4.2 cases 4, 7, 8; 4.3 cases 2-4) and `design/OOL-3/threat_model-2.md`
(THREAT_MODEL round 2: T3, T4, T10, T13), coding principles read at `145008cdcc23`.

## What was tested

| PR | Head tested | Base tip | Merge |
|---|---|---|---|
| https://github.com/oolong-project/tea-client-go/pull/3 | `55a19b24dccce3231af1922aff69c5ae9338fc22` | `2026-11-oolong` @ `6d422b1` (the OOL-5 merge) | scratch merge `--no-ff` of the head into the base tip: clean (local merge commit `15f7ffb`; the base is an ancestor of the head, so the merged tree is the head's tree) |

oolong-project/oolong is not changed by OOL-6; its `2026-11-oolong` is `e1c9e6e`. The
integration harness ran a server built from that commit; the in-cluster server is the CI-built
`oolong-server:2026-11-oolong` image with the TEA 1.0 fixtures (namespace
`oolong-tester-ool-4`). tea-client-go has no CI workflow yet (operator decision), so there is no
CI build to check. Tools: Go 1.26.10 in a `golang:1.26` pod on the local k3s, Node 24 for the
harness-started server.

## Results

| # | Check | Result |
|---|---|---|
| 1 | `gofmt -l .` | empty |
| 2 | `go build ./...`; `go vet ./...`, also with `-tags integration` and `-tags live` | passed, clean |
| 3 | `go mod tidy -diff`; `go mod verify` | empty; all modules verified |
| 4 | `go generate ./...` then `git status --porcelain` | no diff |
| 5 | `go test -count=1 -v ./...` | passed: 75 top-level tests, 112 subtests, 0 failed, 0 skipped |
| 6 | `go test -race -count=3 ./...` | passed |
| 7 | Flake check: `TestDownloadIdleTimeout`, `TestDownloadBounds`, `TestAPIKeyRecovery`, `TestFlowFromRelease`, `TestDownloadArtifactToFile` at `-count=20`, then `-race -count=5` | passed, no flake |
| 8 | Fuzzers, 30 s each: `FuzzDecodeWellKnown` (about 1.64M execs), `FuzzDecodeDiscovery` (about 1.58M), `FuzzParse` and `FuzzParseLegacyURN` in `tei` (this child adds none) | no crasher |
| 9 | Examples (`ExampleClient_FullFlow`, `ExampleAPI_DownloadArtifactToFile`, `ExampleWithAPIKey`) compile in `go test` and `go vet` | passed |
| 10 | Plain-ASCII sweep (`LC_ALL=C grep -nP "[^\x00-\x7F]"`) over the 20 changed files | no hit |
| 11 | No change under `.github/`, `spec/`, `teaapi/` | passed |
| 12 | Integration (4.2) through the harness, `OOLONG_SERVER_DIR` at a built oolong-server (`e1c9e6e`), `TEA_INTEGRATION_REQUIRED=1` | 9 of 9 passed (incl. cases 4, 7, 8: `TestIntegrationFullFlow`, `TestIntegrationDownload`, `TestIntegrationExternalDownload`) |
| 13 | Integration with `OOLONG_BASE_URL` at the in-cluster server, `TEA_INTEGRATION_REQUIRED=1` | 9 of 9 passed |
| 14 | Live suite (4.3) compiled with `-tags live` and run without `TEA_LIVE` | compiles; both tests skip naming `TEA_LIVE=1` |
| 15 | Design cases mapped to tests and read clause by clause (below) | 18 of 18 have tests asserting each listed clause |
| 16 | Mutation checks (below) | 54 of 62 killed by the unit suite; 3 survivors are equivalent; 5 survivors are T-3 to T-5 |
| 17 | Tester probes for overflow and leaks | 2 findings (T-1, T-2), 1 leak probe passed |
| 18 | Live check (below) | 43 of 43 client checks as the design says |

Not run: the live suite (4.3 cases 2-4) itself. It is hard-wired to the public demo deployment,
and this run touches no shared deployment. Its structure was read against 4.3 (Discover bounds,
FullFlow release uuid equal to the discovery result's, a verified download of the first format
with a supported checksum under a 64 MiB cap, unknown uuid `ErrObjectUnknown`) and matches; the
coder reports a run on 2026-11-09. Counted as 3 skipped.

Counts in the index: 132 passed (13 checks: rows 1-7 and 9-14; 4 fuzzers; 18 design cases; 54
killed mutants; 43 live client checks), 5 failed (T-1 to T-5), 3 skipped (4.3 cases 2-4).

## Design cases and their tests

Each clause of each case was read in the test and checked to be asserted, not only exercised.

| # | Design case | Test | Verdict |
|---|---|---|---|
| 1 | 4.1 13c (= 30) | `TestDownloadSource` | covered, incl. an external `url` under the credential root |
| 2 | 4.1 14b | `TestAPIKeyRecovery`, `TestAPIKeyTokenCache`, `TestWithAPIKeyOption` | covered: one `POST /token` then one retry; `invalid_token` gives one refresh, a second gives `ErrUnauthorized` (2 token requests); `403` makes no token request; the automatic token request answered `307` to another port fails and the target gets 0 requests; id, secret (raw and form-encoded), Basic value and tokens absent from errors and every `TraceEvent` |
| 3 | 4.1 28 | `TestChecksumKnownAnswers` | covered: 10 algorithms on `"abc"` and the 137-byte fixture; BLAKE3 only |
| 4 | 4.1 29 | `TestChecksumRules`, `TestDownloadRefusedBeforeRequest` | covered |
| 5 | 4.1 30 | `TestDownloadSource` | covered: `302` to another origin followed without the header; `http` external URL refused with no request |
| 6 | 4.1 31 | `TestDownloadBounds` | covered: `Content-Length` refused before the body (stalling server, with a control); chunked over and exactly at the limit; negotiated gzip verified on the decoded bytes; `br` refused |
| 7 | 4.1 31b | `TestDownloadIdleTimeout` | covered, incl. no file left, never-sent headers, and `DownloadSignature` |
| 8 | 4.1 32 | `TestDownloadArtifactToFile` | covered as listed (see T-4 for the temp file's directory) |
| 9 | 4.1 33 | `TestDownloadSignature` | covered |
| 10 | 4.1 34 | `TestDownloadStatusAndLatest` | covered (`/latest`, then the versioned download) |
| 11 | 4.1 35 | `TestFlowOneResult` | covered |
| 12 | 4.1 36 | `TestFlowFailures`, `TestFlowFromRelease` | covered |
| 13 | 4.1 37 | `TestFlowSeveralResults` | covered |
| 14 | 4.1 38 | `TestFlowComponentLimit` | covered |
| 15 | 4.1 38b | `TestCandidatesCap` | covered |
| 16 | 4.2 case 4 | `TestIntegrationFullFlow` | covered, passes |
| 17 | 4.2 case 7 | `TestIntegrationDownload` | covered, passes |
| 18 | 4.2 case 8 | `TestIntegrationExternalDownload` | covered, passes: oolong sees no `/download`, the fake host no `Authorization`, the bearer reaches oolong |

README (3.11 in full, 3.10): every Go snippet compiles against the exported API (wrapped and
vetted); the FullFlow example is the design's four lines, taken from `ExampleClient_FullFlow`;
the default limits table matches `client.go`; it covers options, limits and time bounds, the
"under" rule and credentials, the `WithHTTPClient` note on `Authorization`, legacy URNs,
regenerating, unit/integration/live test instructions, Versioning with an empty Changes list,
Licence, the 3.10 CLI table, the BLAKE3 and SHA-1 policy and that signatures are not verified.
Observation, not an item: its Errors list omits `ErrInvalidArgument`, `ErrNoAcceptableFormat`,
`ErrUnsupportedContentEncoding` and `ErrChecksumMismatch` (3.11 does not ask for a complete list).

## Mutation checks

Run one at a time in a worktree of the scratch merge, reset with git, each edit confirmed
applied and compiling; unit suite (`go test -count=1 .`) and integration suite (harness) per
mutant. 62 mutants of my own plus re-runs of 21 from the coder's list (all killed again).

| # | Area | Mutants | Killed by |
|---|---|---|---|
| 1 | Checksums | skip compare; compare first only; BLAKE3 hashed as SHA-256; BLAKE3 or an unknown algorithm counted usable; SHA-1 strong; uppercase accepted (2); wrong length accepted; SHA-384 length 64; `VerifyChecksums` cap ignored | `TestChecksumKnownAnswers`, `TestChecksumRules`, `TestDownloadRefusedBeforeRequest`, `TestDownloadArtifactToFile` |
| 2 | Bounds and coding | `MaxBytes` ignored; `DownloadBytes` default off; no `Content-Length` pre-check; `br` accepted; coding not checked; no idle bound; whole-request `CallTimeout`; idle timer not reset; signature capped by `DownloadBytes` | `TestDownloadBounds`, `TestDownloadIdleTimeout`, `TestIdleTimeoutGenerated`, `TestDownloadSignature`, `TestGeneratedIsBounded` |
| 3 | File helper | temp not removed; rename on error; mode 0644; write `path` directly; `Content-Disposition` used | `TestDownloadArtifactToFile`, `TestDownloadIdleTimeout`, `TestIntegrationDownload` |
| 4 | Credentials | external url not anonymous; credential across a cross-host redirect; recovery on `403`; recovery twice; token request follows redirects; token base not under the root; key id or secret in an error; `Authorization` in trace (3 variants); `Location` echoed; cache ignores expiry; bearer on the token request; token not cached | `TestDownloadSource`, `TestBearerBoundToRoot`, `TestAPIKeyRecovery`, `TestRequestToken`, `TestTraceEvents`, `TestAPIKeyTokenCache` |
| 5 | Signatures and latest | `/latest/download` used; `SIGNATURE_NOT_FOUND` unmapped; download `404` unmapped; `signatureUrl` ignored | `TestDownloadStatusAndLatest`, `TestDownloadSignature`, `TestErrorMapping`, `TestIntegrationDownload` |
| 6 | Flow | `Candidates` cap off and off by one; components truncated; unpinned picks the oldest; `Select` index clamped; ambiguity loads candidates; a component failure aborts the flow; CLE `NOT_IMPLEMENTED` as error | `TestCandidatesCap`, `TestFlowComponentLimit`, `TestFlowOneResult`, `TestFlowSeveralResults`, `TestFlowFailures`, `TestFlowFromRelease`, `TestIntegrationFullFlow` |
| 7 | `apiKey.drop` a no-op (the coder's "equivalent, not run") | 1 | `TestAPIKeyTokenCache` |

Survivors that change nothing observable (equivalent): `tmp.Chmod(0o600)` replaced
(`os.CreateTemp` already creates 0600); `page.Results[0]` replaced by the last element (page
size 1). Survivors of design-relevant behaviour: T-3, T-4, T-5 below.

## Live checks

Local only, on the local k3s, namespace `oolong-tester-ool-4`; no shared or public deployment was
contacted. Build under test: the scratch merge `15f7ffb` (= PR head `55a19b2`) as a throwaway
`cmd/livecheck` program (not committed) in a `golang:1.26` pod. Servers: the in-cluster
`oolong-server:2026-11-oolong` image with the TEA 1.0 fixtures (service `oolong-tester-tea`),
and a stdlib Go fixture server (pod `oolong-tester-ool6-fix`, reachable under two service host
names `oolong-tester-ool6-fix` and `oolong-tester-ool6-fix2`) that serves artifact bytes (good,
mutated, truncated, oversized, gzip, `br`, stalled), redirects to the other host name, and a
fake TEA API that answers `401` with a Bearer challenge, implements `POST /v1.0.0/token` and can
answer `/token` with `307` to the other host. The fixture logs per request the method, host,
path and only whether `Authorization` was present and its scheme. Credentials were throwaway
values. Every origin was plain http, each under its own `WithInsecureHTTPBase`; https-only
behaviour (https to http refusal, TLS) is covered by the unit tests only.

| # | Case | Expected (design) | Result |
|---|---|---|---|
| 1 | `FullFlow` on `tei://localhost/uuid/22222222-2222-4222-8222-222222222222` via `WithWellKnownLocation` | discover, release, CLE, components | passed: well-known 200, discovery 200, release 200, release CLE 404 so `ReleaseCLE` nil, pinned component `4444...` 1.0.0, its CLE 404; `Alpha Fixture` 1.0.0, latest collection holds `5555...` v2 with 2 formats |
| 2 | `DownloadArtifact` JSON v2 | 137 bytes verified | passed: source server, Verified SHA-256 and SHA-1 |
| 3 | `DownloadArtifact` XML v2 | 57 bytes | passed: Verified SHA-256 |
| 4 | `DownloadArtifactToFile` JSON and XML | mode 0600, file name from the caller only | passed: `-rw-------`; oolong's `Content-Disposition` ignored |
| 5 | `DownloadSignature` JSON / XML | 30 bytes / `ErrSignatureNotFound` | passed / passed (`HTTP 404 SIGNATURE_NOT_FOUND`) |
| 6 | `DownloadLatestArtifact` by media type (exact, mixed case, `text/plain`) | verified; unknown type `ErrNoAcceptableFormat` after `/latest` only | passed (3) |
| 7 | External `url`, correct SHA-256, SHA-512, SHA-1, bearer configured for the oolong root | verified, no credential to the external host | passed: `auth=absent` |
| 8 | External `url`, mutated bytes, to a writer and to a file | `*ChecksumError`; existing target unchanged, no temp file | passed: `checksum mismatch: SHA-256`; target keeps its content, directory holds only the target |
| 9 | BLAKE3 only, unknown only, none, SHA-1 only without `AllowSHA1Only` | `ErrNoUsableChecksum`, no request | passed (4): 0 requests in the fixture log |
| 10 | SHA-1 only with `AllowSHA1Only` | verified | passed |
| 11 | Uppercase `algValue` | `*UnexpectedResponseError`, no request, value not echoed | passed |
| 12 | Truncated body (`Content-Length` 137, 60 bytes, close), to a file | error, nothing left behind | passed: `unexpected EOF`, directory empty |
| 13 | Oversized: `MaxBytes` 1024 and `Limits.DownloadBytes` 1024, with and without `Content-Length` | `ErrResponseTooLarge`; with the length, before reading | passed (4): 0 bytes written with the length |
| 14 | gzip (negotiated) / `br` | verified on decoded bytes / `ErrUnsupportedContentEncoding` | passed / passed |
| 15 | Stall after 1 KiB, `IdleTimeout` 500 ms | `ErrTimeout` | passed (about 0.5 s) |
| 16 | Redirect fix to fix2, fix2 allowed / not allowed | followed without `Authorization` / refused `ErrInsecureURL`, fix2 gets nothing | passed / passed |
| 17 | Bearer root on fix: API call (control); external `url` under that root; same redirecting to fix2 | Bearer; none; none on either hop | passed (3) |
| 18 | `WithAPIKey` first GET | `401` Bearer, one `POST /token` with Basic, retry with Bearer | passed (log: absent, Basic, Bearer) |
| 19 | Second call | cached token, no `/token` | passed |
| 20 | Metadata `302` to fix2; server download under the key root; download `302` to fix2; external `url` under the key root | fix2 never sees `Authorization`; Bearer only on fix; none on the external url | passed (4) |
| 21 | `/token` answers `307` to fix2 (through recovery and through `RequestToken`) | call fails, Basic to one URL, fix2 gets nothing | passed: `redirect from /token refused`, 0 fix2 requests |
| 22 | Wrong secret | `*TokenError`, no loop | passed: one `/token`, 2 requests in total |
| 23 | Leak scan | key id, secret, Basic values, tokens absent from errors and trace events | passed: 148 strings scanned, 0 hits; none in the fixture log either |

43 client checks in all (several rows hold more than one). Not exercisable live with these
fixtures (unit tests cover them): https behaviour, redirect cap and same-host or subdomain
redirects, `invalid_token` refresh and `403`/non-GET/non-Bearer cases, idle bound before headers,
external `signatureUrl`, ambiguity and the `Candidates` and `FlowComponents` caps, unpinned
components and a present CLE (oolong answers CLE `NOT_IMPLEMENTED`).

## Failed cases

### T-1: VerifyChecksums with MaxBytes = math.MaxInt64 verifies forged bytes (T3 fail-open)

`checksum.go:198`: `src = io.LimitReader(r, o.MaxBytes+1)`. For `o.MaxBytes == math.MaxInt64`
the sum overflows to a negative number, the `LimitReader` returns nothing, and the hashes are of
zero bytes. So any input verifies against the digest of the empty input, and genuine bytes fail.
A caller who means "no limit" by the largest int64 gets a verification that checks nothing. The
design (3.8 step 5, threat model T3) says every listed supported checksum must match the bytes.
Probe (tester-only file in the worktree, not committed):

```
--- FAIL: TestZZVerifyMaxInt64 (0.00s)
    zz_tester_test.go:26: FINDING: forged bytes verified against the empty-input SHA-256 with MaxBytes=MaxInt64: [SHA-256]
    zz_tester_test.go:33: FINDING: genuine bytes refused with MaxBytes=MaxInt64: tea: checksum mismatch: SHA-256: expected ba7816bf..., got e3b0c442...
```

Fix: guard the `+1` (or refuse such a limit with `ErrInvalidArgument`), with a test.

### T-2: DownloadArtifact panics with MaxBytes or Limits.DownloadBytes = math.MaxInt64

`transport.go:336`, `boundedBody.Read`: `max := b.limit - b.n + 1` overflows and `p[:max]`
panics (`slice bounds out of range [:-9223372036854775808]`). Reached through the new
`DownloadOptions.MaxBytes` and through `WithLimits(Limits{DownloadBytes: math.MaxInt64})`, which
`Limits.validate` accepts; the same for `DownloadSignature` and every bounded kind. The arithmetic
is in the base (`6d422b1`, line 222), but this PR adds the per-call entry point. A library must
not panic on an accepted option. Probe output:

```
--- FAIL: TestZZDownloadMaxInt64 (0.00s)
    zz_tester_test.go:54: FINDING: DownloadOptions.MaxBytes=MaxInt64: err=<nil> panic=runtime error: slice bounds out of range [:-9223372036854775808]
    zz_tester_test.go:62: FINDING: Limits.DownloadBytes=MaxInt64: err=<nil> panic=runtime error: slice bounds out of range [:-9223372036854775808]
```

Fix: saturate the arithmetic (or bound the accepted limits), with a test for both entry points.

### T-3: Anonymous mark tested only on the first hop and not for the 401 recovery

Departure 2 says an external download sends no credential "on any hop", and the recovery guard
says an anonymous request never triggers the token request. Two mutants survive the whole suite:

- `transport.go:417`: `cr != nil && !anonymous(r.Context())` to
  `cr != nil && !(anonymous(r.Context()) && r.Response == nil)`: a redirect hop of an external
  `url` that lands under a credential root would get the bearer. No test has an external `url`
  redirect into the root.
- `transport.go:195`: ` || anonymous(req.Context())` removed: an external `url` under an
  API-key root that answers `401` Bearer would make a token request. No test covers it.

The code is right today (the live check, rows 17 and 20, shows no credential on either hop of an
external redirect); the tests do not hold it (T4, case 30). Add both cases.

### T-4: The temp file's directory in DownloadArtifactToFile is not asserted

`download.go:131`: `os.CreateTemp(filepath.Dir(path), ...)` changed to `os.CreateTemp("", ...)`
survives. 3.8 step 6 (and T10) put the temporary file in the target's directory, so the rename
is atomic and stays on one file system. Case 32's assertions (target kept, no temp file left,
0600) all pass with the temp file elsewhere. Assert that the temporary file is created in the
target's directory (for example by observing it there during a stalled download).

### T-5: The retry with a token that is not cached is untested

`transport.go:180`: `retry := req.Clone(context.WithValue(..., tokenOverrideKey{}, ...))`
changed to `req.Clone(req.Context())` survives. Departure 3 says a token whose `expires_in`
leaves nothing after the 30 s leeway is used for the retry but not cached; with that mutant the
retry goes out without the token. Add a case with `expires_in` of 30 or less asserting the retry
carries the Bearer token and a later call makes a fresh token request.
