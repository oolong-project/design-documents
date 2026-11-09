# OOL-6 test run 2: tea-client-go downloads, checksum verification, flow helper, API-key auth, README (PR #3)

Task: OOL-6 (OOL-3c, child 3 of OOL-3). Verdict: REJECTED. This run re-tests the coder's round 2
(`impl/OOL-6/notes-2.md`), which answers test run 1 (`tests/OOL-6/run-1.md`). All five run-1
items are fixed and RESOLVED: the boundary values of every byte limit now verify genuine bytes and
refuse forged ones, nothing panics, and the round-1 mutants of T-3 to T-5 are killed by the new
tests. The suites, the integration tests (harness and in-cluster server), the fuzzers and the local
live check (144 of 144) pass. One new test gap is open: no test shows that `VerifyChecksums`
stops reading at `MaxBytes`, and dropping its `LimitReader` survives the suite (T-6, P3).

Inputs: `impl/OOL-6/notes-2.md` (DETAILED_DESIGN round 2) and `impl/OOL-6/notes-1.md` (round 1),
the carried `design/OOL-3/architecture-2.md` (ARCHITECTURE round 2: 3.7, 3.8, 3.9, 3.10, 3.11)
and `design/OOL-3/threat_model-2.md` (THREAT_MODEL round 2), coding principles read at
`145008cdcc23`.

## What was tested

| PR | Head tested | Base tip | Merge |
|---|---|---|---|
| https://github.com/oolong-project/tea-client-go/pull/3 | `1255c66434a4555fec43854b8193ba650a25fa9e` | `2026-11-oolong` @ `6d422b1` (unchanged since run 1) | scratch merge of the head with the base tip: already up to date (the base is an ancestor of the head), so the merged tree is the head's tree |

History: run 1 tested `55a19b2`. `55a19b2` is an ancestor of `1255c66`; one commit (`1255c66`,
"OOL-6: T-1 to T-5 from test run 1") was added on top, nothing was rewritten. It changes
`auth.go`, `checksum.go`, `transport.go` and four test files; no exported identifier is added or
changed (the only new declaration is the unexported `maxTokenSeconds`), so the README and the
examples need no change and still match the API as checked in run 1.

oolong-project/oolong is not changed by OOL-6; its `2026-11-oolong` is still `e1c9e6e`. The
integration harness ran the oolong-server build from run 1 (`e1c9e6e`); the in-cluster server is
the CI-built `oolong-server:2026-11-oolong` image with the TEA 1.0 fixtures (namespace
`oolong-tester-ool-4`). tea-client-go has no CI workflow yet (operator decision). Tools: Go
1.26.10 in a `golang:1.26` pod on the local k3s, Node 24 for the harness-started server.

## Results

| # | Check | Result |
|---|---|---|
| 1 | `gofmt -l .` | empty |
| 2 | `go build ./...`; `go vet ./...`, also with `-tags integration` and `-tags live` | passed, clean |
| 3 | `go mod tidy -diff` | empty |
| 4 | `go generate ./...` then `git status --porcelain` | no diff |
| 5 | `go test -count=1 -v ./...` | passed: 79 top-level tests (69 in the root package, 10 in `tei`), 112 subtests, 0 failed, 0 skipped |
| 6 | `go test -race -count=3 ./...` | passed |
| 7 | Flake check: `TestDownloadIdleTimeout`, `TestDownloadBounds`, `TestDownloadArtifactToFile`, `TestAPIKey*`, `TestFlowFromRelease`, `TestDownloadSource`, `TestDownloadExtremeLimits`, `TestExtremeByteLimits`, `TestBoundedBodyNearMaxInt64`, `TestChecksumRules` at `-count=20`; `TestDownloadArtifactToFile`, `TestAPIKey*`, `TestDownloadSource` at `-race -count=10` | passed, no flake (the new stalled-file sub-case of `TestDownloadArtifactToFile` is timing-based and held 30 of 30) |
| 8 | Fuzzers, 30 s each: `FuzzDecodeWellKnown`, `FuzzDecodeDiscovery`, `FuzzParse`, `FuzzParseLegacyURN` | no crasher, no new corpus file |
| 9 | Plain-ASCII sweep (`LC_ALL=C grep -nP "[^\x00-\x7F]"`) over the 20 files the PR changes | no hit |
| 10 | No change under `.github/`, `spec/`, `teaapi/` | passed |
| 11 | Integration (4.2) through the harness, `OOLONG_SERVER_DIR` at the built oolong-server (`e1c9e6e`), `TEA_INTEGRATION_REQUIRED=1` | 9 of 9 passed |
| 12 | Integration with `OOLONG_BASE_URL` at the in-cluster server, `TEA_INTEGRATION_REQUIRED=1` | 9 of 9 passed |
| 13 | README and examples against the exported API | unchanged since run 1 (no exported change in `1255c66`); examples compile under `go test` and `go vet` |
| 14 | Tester boundary probes (below) | 4 of 4 probe tests pass on `1255c66`; the same probes fail on `55a19b2` |
| 15 | Mutation checks (below) | 66 of 69 killed: the run-1 list 60 of 62 (2 equivalent, as in run 1; the 4 run-1 survivors of T-3 to T-5 now killed) and 6 of 7 fix mutants; 1 survivor is T-6 |
| 16 | Live check (below) | 144 of 144 client checks as the design says (43 from run 1, 101 new) |

Not run: the live suite (4.3 cases 2-4). It is hard-wired to the public demo deployment, and this
run touches no shared deployment; as in run 1 it is counted as 3 skipped.

Counts in the index: 248 passed (12 checks: rows 1-7 and 9-13; 4 fuzzers; 4 tester probe tests; the 18 design cases of run 1, whose tests are unchanged or extended and all pass; 66 killed mutants: 60 of the run-1 list and 6 fix mutants; 144 live client checks), 1 failed (T-6), 3 skipped (4.3 cases 2-4).

## Review items from run 1

### T-1: VerifyChecksums with MaxBytes = math.MaxInt64 verifies forged bytes

RESOLVED. `checksum.go` uses `io.LimitReader(r, MaxBytes+1)` only for `0 < MaxBytes <
math.MaxInt64`; at `math.MaxInt64` the input is read unlimited (no input can be longer) and
`n > MaxBytes` is unchanged. Checked:

- Unit probe `TestZZ2VerifyBoundaries` (tester, not committed): `MaxBytes` in `{MaxInt64,
  MaxInt64-1, MaxInt64-2, 2^62, MaxInt64-1024, 36, 35, 4, 3, 0}`: genuine `"abc"` verifies (also
  through a one-byte-at-a-time reader); 35 bytes of forged input are refused against both the
  empty input's SHA-256 (the run-1 failure) and the genuine digest, with `ErrChecksumMismatch` when
  the cap allows the input and `ErrResponseTooLarge` when it does not (`MaxBytes` 4, 3, 2); `-1` and
  `math.MinInt64` give `ErrInvalidArgument`. On `55a19b2` the same probe fails at `MaxInt64`
  (forged bytes verified, genuine bytes refused); on `1255c66` it passes.
- The coder's `TestChecksumRules` covers `MaxInt64` and `MaxInt64-1`; the fix mutant (the
  `LimitReader` used at `MaxInt64` again) is killed by it.
- Live (section d3): `VerifyChecksums` on the 137 bytes fetched from the fixture server, with
  `MaxBytes` at every value of the d list: genuine passes, a one-bit-flipped copy is a mismatch,
  and the flipped copy against the empty input's digest is a mismatch.

### T-2: DownloadArtifact panics with MaxBytes or Limits.DownloadBytes = math.MaxInt64

RESOLVED. `boundedBody.Read` now computes `rem := limit - n` (never negative), slices to `rem+1`
only when `len(p) > rem` (so `rem+1 <= len(p)`), and compares `k > rem` instead of adding `k` to
`n` first. Checked:

- Unit probe `TestZZ2BoundedBodyOracle`: the reader against an oracle (from `n0` under limit `L`,
  a 10-byte source delivers `min(10, L-n0)` bytes and is too large iff `10 > L-n0`) for `L` in
  `{MaxInt64, MaxInt64-1, 2^62, 20, 10, 9, 1}`, `n0` 0 to 12 bytes below `L`, buffer sizes 1 to 64,
  strict and error-body mode, normal and one-byte reader: no panic, `n` stays within `[n0, L]`,
  bytes and errors as the oracle says. On `55a19b2` the same probe fails (error bodies not cut at
  the limit near `MaxInt64`, and panics).
- Unit probe `TestZZ2DownloadBoundaries`: `DownloadArtifact` and `DownloadSignature` from a server
  with `Content-Length` and from a chunked one, with `MaxBytes` and with every byte `Limits` field at
  `MaxInt64`, `MaxInt64-1`, `MaxInt64-L`, `MaxInt64-L-1`, `2^62`, `L+1`, `L` (L = body length)
  and `0`: genuine bytes verify, a wrong checksum is `ErrChecksumMismatch`; `L-1` and `1` give
  `ErrResponseTooLarge`; negative `MaxBytes` gives `ErrInvalidArgument`; negative
  `Limits.DownloadBytes` / `ErrorBodyBytes` are refused by `New`.
- The coder's `TestDownloadExtremeLimits`, `TestExtremeByteLimits` and
  `TestBoundedBodyNearMaxInt64` kill the two fix mutants (the old `limit-n+1` slice panics;
  `n+k > limit` fails the near-limit unit test).
- Live (sections d1, d2, d4 to d7): no panic at any value, against both the local oolong server and
  the fixture server.

Overflow audit, the same pattern elsewhere in the PR: I read every place a caller- or
server-supplied limit, size or duration takes part in arithmetic in the hand-written packages and
agree with the coder's table in `notes-2.md`. `Limits.validate` refuses negative byte and count
limits; the count limits (`DiscoveryResults`, `Pages`, `FlowComponents`, `Redirects`) are only
compared, never incremented past or used as an allocation size (`make` uses the length of the
received list, after the cap check); the time bounds go unchanged to `context.WithTimeoutCause` /
`time.AfterFunc`, and `time.Time.Add` drops the monotonic reading instead of overflowing. The one
further site, `expires_in * time.Second` from the token response, is saturated in `1255c66`; probe
`TestZZ2TokenExpiresBoundary` (`math.MaxInt`, `maxTokenSeconds` and its neighbours cached for 100
years; 31 cached; 30, 0, -1, `math.MinInt` not cached) passes, and the saturation mutant is killed
by `TestAPIKeyTokenCache`. No new overflow defect.

### T-3: Anonymous mark tested only on the first hop and not for the 401 recovery

RESOLVED. The two run-1 mutants are killed: anonymous mark only on the first hop (`X44`) by
`TestDownloadSource` (an external URL on another server redirecting into the bearer root);
`anonymous` dropped from the recovery guard (`M39`) by `TestAPIKeyRecovery` (an external URL under
the key's root answered `401` Bearer: `ErrUnauthorized`, one request without `Authorization`, no
token request). Live: d8 and d9 show the same against the fixture servers.

### T-4: The temp file's directory in DownloadArtifactToFile is not asserted

RESOLVED. `TestDownloadArtifactToFile` now stalls a download and sees `.stalled.json.tea-*` in the
target's directory, then an empty directory after `ErrTimeout`. The run-1 mutant
(`os.CreateTemp("", ...)`, `X12`) is killed by it. The sub-case held 30 of 30 runs (row 7).

### T-5: The retry with a token that is not cached is untested

RESOLVED. New `TestAPIKeyUncachedToken` (`expires_in` 30): the product requests carry no token,
token 1 (the retry), no token, token 2, with two token requests. The run-1 mutant (retry cloned
without the token override, `X37`) is killed by it.

## New review items

### T-6: VerifyChecksums reading past MaxBytes is untested; dropping its LimitReader survives

Priority 3, about DETAILED_DESIGN. `VerifyChecksums` documents "With o.MaxBytes > 0, reading stops
with ErrResponseTooLarge past that many bytes", and 3.8 step 4 (whose rules `VerifyChecksums`
applies) caps the reader at `MaxBytes`. The cap is the `io.LimitReader(r, MaxBytes+1)` at
`checksum.go:202`. Mutant F7 replaces it with `src = r`: the function then reads the whole input
before the `n > MaxBytes` check, and with an endless or very large reader never returns or hashes
without bound. The whole unit suite and the integration tests pass with the mutant (every test
input is a few bytes, so reading it all and then refusing gives the same error).

A test that kills it, run as a tester probe (not committed): `VerifyChecksums` over an endless
reader with `MaxBytes` 1 MiB must return `ErrResponseTooLarge` promptly.

```
on 1255c66:          ok   github.com/oolong-project/tea-client-go  0.005s
with the F7 mutant:  --- FAIL: TestZZF7EndlessReader (5.00s)
                     zz_f7_test.go:29: endless reader, MaxBytes 1 MiB: still reading after 5 s
```

The code is right; the remedy is a test of this kind in `checksum_test.go`. The gap predates
round 2 (the same `LimitReader` line was in `55a19b2`), but the T-1 fix changed this exact
condition, so a test of what the cap does belongs with it.

## Tester probes

Unit probes in a scratch copy of `1255c66` (file `zz_r2_test.go`, not committed):
`TestZZ2VerifyBoundaries`, `TestZZ2BoundedBodyOracle`, `TestZZ2DownloadBoundaries`,
`TestZZ2TokenExpiresBoundary`, described under T-1 and T-2. All pass on `1255c66`. The first three
also ran on `55a19b2` (without the token probe, whose constant does not exist there) and fail
there, so they discriminate the fixed code from the run-1 code.

## Mutation checks

One mutation at a time in a scratch copy of `1255c66`, reset with git between mutations; each
mutant ran the unit suite (and, for the targeted set, the harness integration tests). The tester's
probe files were not present during mutation runs.

Targeted set, fix and round-1 survivors:

| # | Mutant | Result |
|---|---|---|
| 1 | M39: `anonymous` dropped from the 401-recovery guard (run-1 T-3) | killed by `TestAPIKeyRecovery` |
| 2 | X44: anonymous mark only on the first hop (run-1 T-3) | killed by `TestDownloadSource` |
| 3 | X12: temp file in `os.TempDir()` (run-1 T-4) | killed by `TestDownloadArtifactToFile` |
| 4 | X37: retry without the token override (run-1 T-5) | killed by `TestAPIKeyUncachedToken` |
| 5 | F1: `LimitReader(MaxBytes+1)` at `MaxInt64` again | killed by `TestChecksumRules` |
| 6 | F2: the old `limit - n + 1` slice | killed by `TestDownloadExtremeLimits` (panic) |
| 7 | F3: count compared as `n + k > limit` | killed by `TestBoundedBodyNearMaxInt64` |
| 8 | F4: `expires_in` not saturated | killed by `TestAPIKeyTokenCache` |
| 9 | F5: strict limit off by one (`k >= rem`) | killed by `TestDiscoverByteLimits`, `TestDownloadBounds`, `TestDownloadSignature`, `TestJSONByteLimit` |
| 10 | F6: `VerifyChecksums` cap off by one (`n >= MaxBytes`) | killed by `TestChecksumRules` |
| 11 | F7: `VerifyChecksums` without the `LimitReader` | survives: T-6 |

Regression set, the run-1 mutant list re-run on `1255c66`: the 62 mutants of run 1, unit suite per mutant (the first 18 also with the harness integration tests, all integration-green as expected for unit-killed mutants). 60 of 62 are killed, including the four run-1 survivors above and the run-1 `apiKey.drop` no-op. The 2 survivors are the same equivalent mutants as in run 1: `tmp.Chmod(0o600)` replaced (`os.CreateTemp` already creates the file 0600) and the unpinned release taken from the last element of a one-element page. In all: 66 of 69 mutants killed; 2 equivalent; 1 survivor filed as T-6.

## Live checks

Against build `1255c66` (the merged tree), from a throwaway `cmd/livecheck` in a scratch copy, run
in the Go pod against in-cluster servers only: the local oolong server (the CI-built
`oolong-server:2026-11-oolong` image, TEA 1.0 fixtures) and the stdlib fixture servers from
run 1 (`oolong-tester-ool6-fix`, `oolong-tester-ool6-fix2`, port 8080, namespace
`oolong-tester-ool-4`). No shared deployment was contacted. Result: 144 PASS, 0 FAIL.

- Sections a to c, the 43 run-1 checks (flow and server downloads against oolong; external
  downloads, checksum mismatch, SHA-1 only, uppercase refusal, truncation, oversize, gzip/br,
  stall, redirects; API-key recovery, token cache, `/token` redirect refusal, wrong secret, no
  credential in any error or trace event): 43 of 43 pass, as in run 1. The leak scan (c7) now also
  covers the errors and trace events of section d.
- d1 (24 checks): against oolong, `DownloadArtifact` (JSON, 137 bytes, artifact v2) with
  `DownloadOptions.MaxBytes` in `{MaxInt64, MaxInt64-1, MaxInt64-137, MaxInt64-138, 2^62, 138,
  137, 0}`: verified, 137 bytes; the same format with a wrong SHA-256: `ErrChecksumMismatch`;
  `DownloadSignature`: 30 bytes.
- d2 (16): external URL on the fixture server with the same values: genuine bytes verify; the
  server's mutated bytes (forged content, genuine checksums) are `ErrChecksumMismatch`.
- d3 (24): `VerifyChecksums` with the same values (described under T-1).
- d4 (20): a client with every byte `Limits` field at `MaxInt64`, `MaxInt64-1`, `MaxInt64-137`,
  `MaxInt64-138` and `2^62`: `FullFlow` on the oolong fixture TEI resolves release 1.0.0;
  `DownloadArtifact` verifies; a wrong checksum is a mismatch; an unknown artifact (`404` under
  `ErrorBodyBytes` at that value) is `ErrObjectUnknown`. (With all byte limits at 137 or 138 the
  153-byte `/discovery` answer is refused, correctly; those values are covered by d1 to d3 instead.)
- d5 (5): `DownloadArtifactToFile` with `MaxBytes` at those values: 137 bytes, one file in the
  directory.
- d6 (6): `MaxBytes` 136 and 1 against oolong, the fixture's `Content-Length` body and its chunked
  body: `ErrResponseTooLarge`.
- d7 (4): `MaxBytes` -1 and `math.MinInt64`: `ErrInvalidArgument`; `Limits.DownloadBytes` -1 and
  `math.MinInt64`: refused by `New`.
- d8 (1, T-3): a bearer token configured for `oolong-tester-ool6-fix2`; an external URL on
  `oolong-tester-ool6-fix` that answers `302` into the bearer's root: downloaded and verified, two
  requests, neither carries `Authorization`.
- d9 (1, T-3): an API key for `oolong-tester-ool6-fix` `/api`; an external URL under that root
  answered `401` with a Bearer challenge: `ErrUnauthorized`, one request without `Authorization`,
  no `/token` request.

## Models

Verdict, boundary probes, overflow audit, mutation design and live check: claude-opus-5-5; suite
runs (rows 1-12): sonnet.
