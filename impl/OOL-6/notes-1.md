# OOL-6 implementation notes, round 1: downloads, checksum verification, flow helper, API-key auth, README

Task: OOL-6 (OOL-3c, child 3 of OOL-3). Design carried from OOL-3: `design/OOL-3/architecture-2.md`
(ARCHITECTURE round 2) and `design/OOL-3/threat_model-2.md` (THREAT_MODEL round 2); read with
`design/OOL-5/architecture-1.md`, whose section 5 says nothing OOL-6 builds changes. Coding
principles read at `145008cdcc23`. Built: 3.7 (`WithAPIKey` and its 401 recovery), 3.8 (downloads
and checksum verification, `IdleTimeout` on the wrapped downloads), 3.9 (flow helper, `Candidates`
cap), 3.10 (documented in README), the README of 3.11 in full. Tests: 4.1 cases 13c (= 30), 14b,
28, 29, 30, 31, 31b, 32-38, 38b; 4.2 cases 4, 7, 8; 4.3 cases 2-4.

## Pull request

| PR | Repository | Base | Head commit | Base tip merged |
|---|---|---|---|---|
| https://github.com/oolong-project/tea-client-go/pull/3 | oolong-project/tea-client-go | `2026-11-oolong` | `55a19b24dccce3231af1922aff69c5ae9338fc22` | `6d422b1` (the merge of OOL-5; the branch starts there, the base has not moved) |

Commits: `8ffb648` (downloads, checksums, `WithAPIKey`, the classifier fix, `go.mod`), `5e14904`
(flow helper, case 2 assertions), `55a19b2` (integration and live cases, examples, package doc,
README). Each is signed and carries the ReARM trailers. No regeneration, nothing under `spec/`,
`teaapi/` or `.github/`. One PR; oolong-project/oolong is not changed. Merge order: this PR only.

## What changed

- **`checksum.go`** (3.8 step 5): `VerifyChecksums`, `ChecksumError` (wraps `ErrChecksumMismatch`),
  `ErrNoUsableChecksum`, `ErrUnsupportedContentEncoding`. A plan is built from the format's
  checksums before any request: every `algValue` of a specification algorithm must be lowercase hex
  of that algorithm's length (BLAKE3 included, 64), else `*UnexpectedResponseError`; unknown
  `algType`s are ignored; at least one supported algorithm other than SHA-1 must be listed, or
  SHA-1 with `AllowSHA1Only`, else `ErrNoUsableChecksum` (no checksums, BLAKE3 only and unknown only
  included). Supported: SHA-1, SHA-256/384/512, SHA3-256/384/512 (`crypto/sha3`), BLAKE2b-256/384/512
  (`golang.org/x/crypto/blake2b`). Every supported entry must match; the first mismatch in listed
  order is the `*ChecksumError`. `Verified` lists the algorithms in first-listed order.
- **`download.go`** (3.8): `ArtifactRef`, `DownloadOptions`, `DownloadSource` (closed:
  `DownloadSourceServer`, `DownloadSourceExternal`), `DownloadResult`, `DownloadArtifact`,
  `DownloadArtifactToFile`, `DownloadLatestArtifact`, `DownloadSignature`. Source: `f.Url` (an
  anonymous request, below) or `/artifact/{uuid}/{version}/download?mediaType=` through the
  generated client; 200 only; `Content-Encoding` other than empty or `identity` left by the
  transport is refused; bytes stream through `io.MultiWriter(w, hashers...)`; the doer's kinds give
  `DownloadBytes`/`SignatureBytes` and `IdleTimeout`. The file helper: `os.CreateTemp` in the
  target's directory, mode 0600, verify, `Sync`, `Close`, `Rename`; removed on any error.
- **`auth.go`, `transport.go`, `client.go`** (3.7): `WithAPIKey(root, id, secret)`; the token
  request moved from `*API` to the doer (`API.RequestToken` unchanged). `doer.Do` runs one request
  (`do`) and, on a recoverable `401`, drains it, drops the cached token if the request carried it,
  makes the token request to `<API base>/token` (refused with `ErrInvalidArgument` and no request
  when the API base is not under the key root), caches the token (`expires_in` minus 30 s) and
  retries once with it (the token travels on the retry's context, so the auth transport still
  decides every hop). The auth transport records which token it sent, adds the cached API-key token
  under the root, and sends no credential on any hop of a request marked anonymous.
- **`transport.go`, per-call byte limit**: `DownloadOptions.MaxBytes` travels on the context and
  replaces the limit of a `kindDownload` or `kindSignature` request only.
- **`flow.go`** (3.9): `FlowOptions`, `Candidate`, `AmbiguousError`, `FlowResult`, `FlowComponent`,
  `FullFlow`, `Candidates`, `FlowFromRelease`, as the design describes.
- **README** (3.11 in full, 3.10): what the module is, the packages, install, the `FullFlow`
  example (copied from `ExampleClient_FullFlow`), discovery, verified downloads (BLAKE3 and SHA-1
  policy, signatures not verified and the residual of T3), options, limits and time bounds tables,
  URL and credential policy (the "under" rule, `WithAPIKey`, the `WithHTTPClient` note), errors,
  server-side SSRF note, legacy URNs, the CLI mapping table of 3.10, regenerating, dependencies,
  tests (unit, integration, live), Versioning with the empty Changes list, Licence. It says the
  repository has no CI workflow yet and describes none.
- **`doc.go`**, examples (`ExampleClient_FullFlow`, `ExampleAPI_DownloadArtifactToFile`,
  `ExampleWithAPIKey`; no `// Output:`, never run).
- **Dependencies**: `golang.org/x/crypto` v0.58.0 (newest release on 2026-11-09) is now a direct
  requirement; it was already in the graph at v0.46.0 through `oapi-codegen/runtime`.
  `golang.org/x/sys` v0.49.0 becomes indirect (imported by `blake2b`). `golang.org/x/mod` stays at
  v0.42.0, its newest release. `go mod tidy -diff` is empty.

## Departures and decisions the design left open

1. **Classifier fix in OOL-4's doer (design D3 quotes the signature templates wrongly).** D3 lists
   `/artifact/{uuid}/latest/signature` and `/artifact/{uuid}/{v}/signature` as `kindSignature`, and
   OOL-4 built exactly that; the pinned specification's paths are
   `/artifact/{uuid}/latest/signature/download` and `/artifact/{uuid}/{artifactVersion}/signature/download`
   (`teaapi/tea.gen.go`, operations `downloadLatestArtifactSignature` and
   `downloadArtifactSignatureByVersion`). So every `Generated()` signature download was classified
   as JSON (`JSONBytes`, `CallTimeout`). The classifier now matches the specification's paths (5
   segments); the 4-segment form is JSON. `TestClassify` is updated and `TestGeneratedIsBounded`
   gains signature cases. "Implement the pinned TEA specification" decides it; the wrapped
   `DownloadSignature` sets its kind itself and was never affected.
2. **No credential to an external URL, even under a credential root.** D5 decides the header by the
   root; 3.8 step 1 says no credential is attached to `f.Url` (and `f.SignatureUrl`), and the spec
   says a TEA token is never sent to an external URL. A hostile server could list a `url` under its
   own root, so external requests carry an anonymous mark on their context and the auth transport
   sends no credential on any of their hops. Tested (case 30, an external `url` under the root).
3. **`WithAPIKey` recovery reading.** Any `401` whose first `WWW-Authenticate` challenge is Bearer
   triggers the one recovery, whether or not the request carried a token and whatever the `error`
   parameter; when it carried the cached token, that token is dropped first. This is the design's
   two bullets taken together ("On a 401 ... Bearer challenge ... makes the token request"; "a 401
   with invalid_token on a request that carried a cached token drops it"), bounded at one recovery
   per request. The request and the URL that answered the `401` (after redirects) must both be
   under the same API-key root, the method GET, the kind not `kindToken`, the request not anonymous,
   and the doer part of an `API` (discovery requests have no API base, so no recovery there).
   A token whose `expires_in` leaves nothing after the 30 s leeway is used for the retry but not
   cached. `WithAPIKey` refuses an empty id or secret (`ErrInvalidOption`, the values never echoed).
   Among credentials with equal roots the first configured applies (existing behaviour, now
   documented on `WithAPIKey`).
4. **Statuses on downloads.** A `406` always matches `ErrNoAcceptableFormat` (wrapped when the body
   is not the TEA code); other `4xx`/`5xx` are `*APIError`; a `2xx` other than 200 and an unfollowed
   `3xx` are `*UnexpectedResponseError`.
5. **Refused checksum metadata** is `*UnexpectedResponseError` with `Status` 200 (the status of the
   metadata response) and a reason naming the checksum's index, algorithm and expected length,
   never the value.
6. **Arguments refused before any request** (`ErrInvalidArgument`): a server-endpoint download with
   version below 1, a nil uuid or an empty `MediaType`; a nil writer; a negative `MaxBytes`; an
   empty path. A relative external `url` is `*UnexpectedResponseError`, no request.
7. **`DownloadOptions.MaxBytes`** overrides `SignatureBytes` for `DownloadSignature` too (it takes
   `DownloadOptions`); on `VerifyChecksums`, which has no client, it is an optional cap (no default).
8. **`DownloadResult.MediaType`** is the response's `Content-Type` media type (lowercase, without
   parameters), `""` when absent; it tells the caller what the server returned.
9. **`DownloadLatestArtifact`** picks the format by media type as the specification's `mediaType`
   selection compares (type and subtype case-insensitively, parameters exactly); none is an error
   wrapping `ErrNoAcceptableFormat`, after the metadata request only.
10. **Flow details.** `Candidates` returns the first failure with the result's index (a `Candidate`
    has no error field); `FullFlow` continues with the chosen candidate's already-read release (it is
    not read a second time); an unpinned
    component with no release fails with an error wrapping `ErrObjectUnknown`; a component CLE
    error other than `NOT_IMPLEMENTED`/`OBJECT_UNKNOWN` goes into that component's `Err` with its
    `Release` kept; `SkipCLE` skips the component CLEs too; the caller's context ending during the
    walk ends `FlowFromRelease` with `ctx.Err()` (wrapped) instead of recording it per component;
    zero discovery results (unreachable, `Discover` never returns them) is `ErrInvalidArgument`.
11. **CI.** No `.github/` content (operator decision for this round). This child's design adds no CI
    job (the `test`, `integration` and `record-release` jobs were OOL-4's, `live` OOL-5's), so
    nothing was appended to the deferred workflow file. Nothing here claims CI.

## Tests added

| Design case | Test (file) |
|---|---|
| 13c (= 30) | `TestDownloadSource` (`download_test.go`) |
| 14b | `TestAPIKeyRecovery`, plus `TestAPIKeyTokenCache`, `TestWithAPIKeyOption` (`auth_test.go`) |
| 28 | `TestChecksumKnownAnswers` (`checksum_test.go`; vectors from Python hashlib, `"abc"` and the 137-byte fixture) |
| 29 | `TestChecksumRules` (`checksum_test.go`), `TestDownloadRefusedBeforeRequest` (no request made) |
| 30 | `TestDownloadSource` |
| 31 | `TestDownloadBounds` |
| 31b | `TestDownloadIdleTimeout` |
| 32 | `TestDownloadArtifactToFile` |
| 33 | `TestDownloadSignature` |
| 34 | `TestDownloadStatusAndLatest` |
| 35 | `TestFlowOneResult` (`flow_test.go`) |
| 36 | `TestFlowFailures`, `TestFlowFromRelease` |
| 37 | `TestFlowSeveralResults` |
| 38 | `TestFlowComponentLimit` |
| 38b | `TestCandidatesCap` |
| 4.2 case 4 | `TestIntegrationFullFlow` (`integration_test.go`) |
| 4.2 case 7 | `TestIntegrationDownload` |
| 4.2 case 8 | `TestIntegrationExternalDownload` |
| 4.3 cases 2-4 | `TestLiveFlowAndDownload` (`live_test.go`) |
| departure 1 | `TestClassify`, `TestGeneratedIsBounded` (`transport_test.go`) |
| case 2 | wrapper-set assertions in `spec_test.go` gain the download and flow helpers and `WithAPIKey` |

How the limits are exercised: byte limits at the limit and one byte over (chunked 2048 with
`MaxBytes` 2047 and 2048; `SignatureBytes` 2047 and 2048), the `Content-Length` refusal against a
server that sends the length and no body (with the control that the same response under the limit
waits for the idle bound), `DownloadBytes` 1024 with and without a `MaxBytes` override,
`DiscoveryResults` 2 with 3 and 2 results and the default 32 with 33, `FlowComponents` 2 and 3 with
three refs, `IdleTimeout` 200 ms with `CallTimeout` also 200 ms against a 1.5 s trickle (so a
download under `CallTimeout` would fail).

## How to run them

```sh
export PATH=$PATH:/usr/local/go/bin
gofmt -l .                                   # empty
go vet ./... && go vet -tags integration ./... && go vet -tags live ./...
go build ./...
go mod tidy -diff                            # empty
go generate ./... && git status --porcelain  # teaapi unchanged
go test -count=1 ./...                       # root: 65 top-level tests pass (103 subtests), tei: 10
go test -count=3 -race ./...
go test -count=20 -run 'TestDownloadIdleTimeout|TestDownloadBounds|TestAPIKeyRecovery|TestFlowFromRelease' .
go test -run '^$' -fuzz '^FuzzDecodeDiscovery$' -fuzztime 10s .   # unchanged fuzzers, sanity only
go test -run '^$' -fuzz '^FuzzParse$' -fuzztime 10s ./tei
# oolong 2026-11-oolong @ e1c9e6e, npm ci && npm run build in oolong/oolong-server
OOLONG_SERVER_DIR=<oolong>/oolong/oolong-server TEA_INTEGRATION_REQUIRED=1 go test -count=1 -tags integration -run Integration ./...   # 9 of 9
TEA_LIVE=1 go test -count=1 -tags live -run Live ./...   # 2 of 2, 2026-11-09, public demo
LC_ALL=C /usr/bin/grep -nP "[^\x00-\x7F]" <changed files>   # nothing
```

Only this child's tests:
`go test -count=1 -run 'TestChecksum|TestDownload|TestAPIKey|TestWithAPIKey|TestFlow|TestCandidates|TestClassify|TestGeneratedIsBounded' .`

All of the above passed on `55a19b2` (Go 1.26.0, Node 24.14.1). The live run verified a
14,300,708-byte BOM of the demo with SHA-256 and found the unknown product `ErrObjectUnknown`. No
fuzz target is listed for this child.

## Mutation checks (one-off, in a scratch copy; not committed)

37 mutants, all killed. Two survived a first run and the tests were strengthened before the
commit: a `403` with a Bearer challenge triggering recovery (the test's `403` had no challenge; it
now carries `error="insufficient_scope"`) and recovery after a redirect off the key root (new
sub-case: a request under the root redirected to another server that answers `401 Bearer` makes no
token request and sends that server no credential).

| # | Mutation | Failing tests |
|---|---|---|
| 1 | checksum comparison skipped | `TestChecksumKnownAnswers`, `TestChecksumRules`, `TestDownloadArtifactToFile` |
| 2 | only the first listed checksum compared | `TestChecksumRules` |
| 3 | SHA-1 counted as strong | `TestChecksumRules`, `TestDownloadRefusedBeforeRequest` |
| 4 | BLAKE3 counted as supported | `TestChecksumKnownAnswers`, `TestChecksumRules`, `TestDownloadRefusedBeforeRequest` |
| 5 | `algValue` format not checked | `TestChecksumRules` |
| 6 | SHA3-256 computed with SHA-256 | `TestChecksumKnownAnswers` |
| 7 | BLAKE2b-384 computed as BLAKE2b-512 | `TestChecksumKnownAnswers` |
| 8 | `MaxBytes` override ignored | `TestDownloadBounds`, `TestDownloadSignature` |
| 9 | `Content-Length` pre-check removed | `TestDownloadBounds` |
| 10 | external URL not marked anonymous | `TestDownloadSource` |
| 11 | auth transport ignores the anonymous mark | `TestDownloadSource` |
| 12 | wrapped download sent as `kindJSON` | `TestDownloadBounds`, `TestDownloadIdleTimeout` |
| 13 | signature sent as `kindJSON` | `TestDownloadIdleTimeout`, `TestDownloadSignature` |
| 14 | content coding not checked | `TestDownloadBounds` |
| 15 | temporary file kept on error | `TestDownloadArtifactToFile`, `TestDownloadIdleTimeout` |
| 16 | `signatureUrl` ignored | `TestDownloadSignature` |
| 17 | `406` without a TEA body not mapped | `TestDownloadStatusAndLatest` |
| 18 | classifier back to `/signature` | `TestClassify`, `TestGeneratedIsBounded` |
| 19 | a second recovery after a failed retry | `TestAPIKeyRecovery` |
| 20 | `403` triggers recovery | `TestAPIKeyRecovery` |
| 21 | non-Bearer challenge recovers | `TestAPIKeyRecovery` |
| 22 | non-GET recovers | `TestAPIKeyRecovery` |
| 23 | token base not checked against the key root | `TestAPIKeyRecovery` |
| 24 | answering URL not checked against the key root | `TestAPIKeyRecovery` |
| 25 | cached token never used | `TestAPIKeyRecovery` |
| 26 | `expires_in` leeway ignored | `TestAPIKeyTokenCache` |
| 27 | `Candidates` cap removed | `TestCandidatesCap` |
| 28 | `Candidates` cap off by one | `TestCandidatesCap` |
| 29 | `FlowComponents` check removed | `TestFlowComponentLimit` |
| 30 | unpinned component sorted ascending | `TestFlowOneResult` |
| 31 | `Select` nil picks the first result | `TestFlowSeveralResults` |
| 32 | context check in the walk removed | `TestFlowFromRelease` |
| 33 | every CLE error treated as no CLE | `TestFlowFailures` |
| 34 | chosen candidate's release read again | `TestFlowSeveralResults` |
| 35 | `DownloadLatestArtifact` downloads another version | `TestDownloadStatusAndLatest` |
| 36 | media type compared exactly | `TestDownloadStatusAndLatest` |
| 37 | failed token request ignored (retry without token) | `TestAPIKeyRecovery` |

Equivalent, not run: making `apiKey.drop` a no-op. The retry uses the fresh token from its context
and `store` replaces the cached one, so dropping only matters when the token request then fails,
and the next request would get the same 401 and recovery either way.

## For the tester

- The integration server is a harness-started oolong at `2026-11-oolong` @ `e1c9e6e`. Case 8 needs
  nothing external: the test's transport answers `https://example.invalid` in-process and forwards
  the rest; it records the forwarded paths (no `/download`) and the headers (the bearer reaches
  oolong, none reaches the fake host).
- Timing tests (`TestDownloadIdleTimeout`, the stall case of `TestDownloadBounds`) use 200 to
  300 ms bounds and assert "within 1 s"; they passed 20 times in a row, also under `-race`.
- `TestLiveFlowAndDownload` downloads the demo's 14.3 MB BOM into `io.Discard` (64 MiB cap).
- Error texts to check for leaks: `TestAPIKeyRecovery` searches every returned error and every
  trace event for the key id, the secret (raw and form-encoded), the Basic value and the issued
  tokens. Checksum refusals are checked not to repeat the refused value.
- Callers see one change outside the new API: a `Generated()` signature download is now bounded by
  `SignatureBytes` and `IdleTimeout` (departure 1).
