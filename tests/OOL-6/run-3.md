# OOL-6 test run 3: tea-client-go downloads, checksum verification, flow helper, API-key auth, README (PR #3)

Task: OOL-6 (OOL-3c, child 3 of OOL-3). Verdict: PASSED. This run re-tests the coder's round 3
(`impl/OOL-6/notes-3.md`), which answers test run 2 (`tests/OOL-6/run-2.md`). The one open item,
T-6, is fixed and RESOLVED: the new `TestVerifyChecksumsStopsAtMaxBytes` fails on the run-2 mutant
that drops the `LimitReader` in `VerifyChecksums`, in under a second, and on four further cap
mutants. The regression run on what will land passes: the static checks, the unit suite with
`-race`, the fuzzers, the integration tests (harness and in-cluster server) and the local live
check (144 of 144). Nothing is open.

Inputs: `impl/OOL-6/notes-3.md` (DETAILED_DESIGN round 3), with rounds 1 and 2
(`notes-1.md`, `notes-2.md`), the carried `design/OOL-3/architecture-2.md` (ARCHITECTURE round 2)
and `design/OOL-3/threat_model-2.md` (THREAT_MODEL round 2), coding principles read at
`145008cdcc23`.

## What was tested

| PR | Head tested | Base tip | Merge |
|---|---|---|---|
| https://github.com/oolong-project/tea-client-go/pull/3 | `4ae95d242aa9948523a0e8e9b2722c4cdda8017a` | `2026-11-oolong` @ `6d422b1` (unchanged since run 1) | scratch merge of the head with the base tip: already up to date (the base is an ancestor of the head), so the merged tree is the head's tree |

History: run 2 tested `1255c66`. `1255c66` is an ancestor of `4ae95d2`; one commit (`4ae95d2`,
"OOL-6: T-6 test that VerifyChecksums stops reading past MaxBytes", signed) was added on top and
nothing was rewritten. `git diff 1255c66 4ae95d2` touches only `checksum_test.go` (43 lines added:
the `io` import, a `countingReader` test type and the new test). No product code, README,
example, `.github/`, `spec/` or `teaapi/` file changed, so every product-code result of run 2
(boundary probes, overflow audit, README against the API) still holds for this tree byte for byte.

oolong-project/oolong is not changed by OOL-6; its `2026-11-oolong` is still `e1c9e6e`. The
integration harness ran the oolong-server build of `e1c9e6e` from run 1; the in-cluster server is
the CI-built `oolong-server:2026-11-oolong` image with the TEA 1.0 fixtures (namespace
`oolong-tester-ool-4`). tea-client-go has no CI workflow yet (operator decision). Tools: Go
1.26.10 in a `golang:1.26` pod on the local k3s, Node 24 for the harness-started server.

## Results

What was re-run in this round, all on the merged tree `4ae95d2`:

| # | Check | Result |
|---|---|---|
| 1 | `gofmt -l .` | empty |
| 2 | `go build ./...`; `go vet ./...`, also with `-tags integration` and `-tags live` | passed, clean |
| 3 | `go mod tidy -diff` | empty |
| 4 | `go generate ./...` then `git status --porcelain` | no diff |
| 5 | `go test -count=1 -v ./...` | passed: 80 top-level tests (70 in the root package, 10 in `tei`), 112 subtests, 0 failed, 0 skipped |
| 6 | `go test -race -count=3 ./...` | passed |
| 7 | Flake check: `TestVerifyChecksumsStopsAtMaxBytes` and `TestChecksumRules` at `-count=50`; `TestVerifyChecksumsStopsAtMaxBytes` at `-race -count=20` | passed, no flake |
| 8 | Fuzzers, 20 s each: `FuzzDecodeWellKnown`, `FuzzDecodeDiscovery`, `FuzzParse`, `FuzzParseLegacyURN` | no crasher, no new corpus file |
| 9 | Plain-ASCII sweep (`LC_ALL=C grep -nP "[^\x00-\x7F]"`) over the file changed since run 2 (`checksum_test.go`) | no hit |
| 10 | No change under `.github/`, `spec/`, `teaapi/` (whole PR against the base) | passed |
| 11 | Integration (4.2) through the harness, `OOLONG_SERVER_DIR` at the built oolong-server (`e1c9e6e`), `TEA_INTEGRATION_REQUIRED=1` | 9 of 9 passed |
| 12 | Integration with `OOLONG_BASE_URL` at the in-cluster server, `TEA_INTEGRATION_REQUIRED=1` | 9 of 9 passed |
| 13 | Mutation checks, checksum area and cap mutants (below) | 18 of 18 killed |
| 14 | Live check (below) | 144 of 144 client checks pass |

Not re-run in this round, and why: the run-2 regression list of 62 mutants outside the checksum
area and the run-2 tester boundary probes. The product code is byte-identical to `1255c66`, where
they ran, and the suite only gained a test, which cannot let a killed mutant survive. Not run at
all: the live suite (4.3 cases 2-4), which is hard-wired to the public demo deployment; this run
touches no shared deployment, so as in runs 1 and 2 it counts as 3 skipped.

Counts in the index: 196 passed (12 checks: rows 1-7 and 9-12; 4 fuzzers; the 18 design cases of
run 1, whose tests are unchanged or extended and all pass; 18 killed mutants; 144 live client
checks), 0 failed, 3 skipped (4.3 cases 2-4).

## Review items from runs 1 and 2

### T-1: VerifyChecksums with MaxBytes = math.MaxInt64 verifies forged bytes

RESOLVED in run 2; the product code is unchanged since then. `TestChecksumRules` still covers
`MaxInt64` and `MaxInt64-1`, and the fix mutant F1 is still killed by it (row 13).

### T-2: DownloadArtifact panics with MaxBytes or Limits.DownloadBytes = math.MaxInt64

RESOLVED in run 2; `transport.go` is unchanged. `TestDownloadExtremeLimits`, `TestExtremeByteLimits`
and `TestBoundedBodyNearMaxInt64` pass (row 5); live sections d1, d2 and d4 to d7 pass with no panic.

### T-3: Anonymous mark tested only on the first hop and not for the 401 recovery

RESOLVED in run 2; unchanged. `TestDownloadSource` and `TestAPIKeyRecovery` pass; live d8 and d9 pass.

### T-4: The temp file's directory in DownloadArtifactToFile is not asserted

RESOLVED in run 2; unchanged. `TestDownloadArtifactToFile` passes, also under `-race -count=3`.

### T-5: The retry with a token that is not cached is untested

RESOLVED in run 2; unchanged. `TestAPIKeyUncachedToken` passes.

### T-6: VerifyChecksums reading past MaxBytes is untested; dropping its LimitReader survives

RESOLVED. New `TestVerifyChecksumsStopsAtMaxBytes` (`checksum_test.go`). It is not hollow:

- The source is a `countingReader` that would serve 64 MiB of `x` bytes and counts every byte it
  hands out. With `MaxBytes` 1 and 1 MiB the test asserts both `errors.Is(err,
  ErrResponseTooLarge)` and that the reader served at most `MaxBytes + 1` bytes. The source is
  finite on purpose: without the cap the call still returns (after reading 64 MiB), so the test
  fails on the count instead of hanging the suite. 64 MiB is 64 times the larger cap, so a cap
  that is merely loose (doubled) is caught too.
- The positive side: a 3-byte input with `MaxBytes` 3 is read whole (count 3) and verified against
  the SHA-256 of `xxx`; I checked the digest in the test independently
  (`cd2eb083...6860`).

The run-2 mutant, re-applied on `4ae95d2` (`src = io.LimitReader(r, o.MaxBytes+1)` replaced with
`src = r`), with an explicit `-timeout 30s`:

```
unmutated:     --- PASS: TestVerifyChecksumsStopsAtMaxBytes (0.00s)
F7 mutant:     checksum_test.go:243: MaxBytes 1: read 67108864 bytes, want at most 2
               checksum_test.go:243: MaxBytes 1048576: read 67108864 bytes, want at most 1048577
               --- FAIL: TestVerifyChecksumsStopsAtMaxBytes (0.19s)
               FAIL  github.com/oolong-project/tea-client-go  0.192s   (whole go test call 1.6 s)
whole unit suite with F7: FAIL, only TestVerifyChecksumsStopsAtMaxBytes fails (11.8 s)
```

Further cap mutants written for this round, each also killed (row 13): the cap at `MaxBytes+2`
(F8) and at `MaxBytes` (F9), both by `TestChecksumRules` and the new test; a doubled cap
`2*MaxBytes+1` (F10), by both; the cap applied only below 1 KiB (F11), by the new test alone. The
coder's own list in `notes-3.md` (F7, `+2`, no extra byte) agrees with these results.

## Mutation checks

One mutation at a time in a scratch copy of `4ae95d2`, reset with git between mutations, unit
suite per mutant with `-timeout 60s`. 18 of 18 killed:

| # | Mutant | Result |
|---|---|---|
| 1 | M01 to M07b (10 mutants): checksum comparison skipped or first only, BLAKE3 hashed as SHA-256 or counted strong, unknown algorithm counted strong, SHA-1 counted strong, uppercase hex accepted or normalised, wrong digest length accepted, SHA-384 length 64 | all killed (run-1 and run-2 tests) |
| 2 | M43: the `n > MaxBytes` check in `VerifyChecksums` disabled | killed |
| 3 | F1: `LimitReader` at `MaxInt64` again (T-1) | killed by `TestChecksumRules` |
| 4 | F6: `n >= MaxBytes` | killed by `TestChecksumRules`, `TestVerifyChecksumsStopsAtMaxBytes` |
| 5 | F7: no `LimitReader` (T-6) | killed by `TestVerifyChecksumsStopsAtMaxBytes` |
| 6 | F8: cap `MaxBytes+2` | killed by `TestChecksumRules`, `TestVerifyChecksumsStopsAtMaxBytes` |
| 7 | F9: cap `MaxBytes` | killed by `TestChecksumRules`, `TestVerifyChecksumsStopsAtMaxBytes` |
| 8 | F10: cap `2*MaxBytes+1` | killed by `TestChecksumRules`, `TestVerifyChecksumsStopsAtMaxBytes` |
| 9 | F11: cap only for `MaxBytes` below 1 KiB | killed by `TestVerifyChecksumsStopsAtMaxBytes` |

## Live checks

Against build `4ae95d2` (the merged tree), the run-2 throwaway `cmd/livecheck` (not committed)
copied unchanged into a scratch copy of `4ae95d2` and run in the Go pod against in-cluster servers
only: the local oolong server (the CI-built `oolong-server:2026-11-oolong` image, TEA 1.0
fixtures) and the stdlib fixture servers from run 1 (`oolong-tester-ool6-fix`,
`oolong-tester-ool6-fix2`, port 8080, namespace `oolong-tester-ool-4`). No shared deployment was
contacted. Result: 144 PASS, 0 FAIL, the same checks as run 2:

- a (10): `FullFlow` and server downloads against oolong.
- b (23): external downloads, checksum mismatch, SHA-1 only, uppercase refusal, truncation,
  oversize, gzip/br, stall, redirects, against the fixture servers.
- c (10): API-key recovery, token cache, `/token` redirect refusal, wrong secret; c7 scans every
  error and trace event for the key id, secret, Basic value and tokens: no leak.
- d (101): the boundary values of `MaxBytes` and every byte `Limits` field (`MaxInt64` and its
  neighbours, `2^62`, body length and neighbours, 0, negatives) for `DownloadArtifact`,
  `DownloadSignature`, `VerifyChecksums` (d3: genuine bytes verify, a one-bit-flipped copy is a
  mismatch), `DownloadArtifactToFile` and `FullFlow`; `ErrResponseTooLarge` below the body length
  (d6); d8 and d9 for the anonymous mark (T-3).

## Models

Verdict, mutation design, T-6 review, regression and live re-runs: claude-opus-5-5. No sub-agents
were used in this round.
