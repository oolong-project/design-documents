# OOL-6 implementation notes, round 3: fix for test run 2 (T-6)

Task: OOL-6 (OOL-3c, child 3 of OOL-3). Answers `tests/OOL-6/run-2.md` (BOARD_TEST_REPORT round 2:
T-1 to T-5 resolved, T-6 open). Rounds 1 and 2 (`impl/OOL-6/notes-1.md`, `notes-2.md`) still
describe the build and the earlier fixes; only what is below changed. Design carried from OOL-3:
`design/OOL-3/architecture-2.md`, `design/OOL-3/threat_model-2.md` (round 2). Coding principles
read at `145008cdcc23`.

## Pull request

| PR | Repository | Base | Head commit | Base tip merged |
|---|---|---|---|---|
| https://github.com/oolong-project/tea-client-go/pull/3 | oolong-project/tea-client-go | `2026-11-oolong` | `4ae95d242aa9948523a0e8e9b2722c4cdda8017a` | `6d422b1` (the base has not moved; nothing to merge) |

The tested head of run 2, `1255c66`, is an ancestor of the new head: one commit, `4ae95d2`, was
added on top. It changes only `checksum_test.go`; no code change, nothing rewritten or
force-pushed, no `.github/` content. Merge order: this PR only.

## Review items

| Id | Status | Commit |
|---|---|---|
| T-6 | fixed (test; the code was right) | `4ae95d2` |

Nothing is disputed and nothing remains open.

### T-6: VerifyChecksums stops reading past MaxBytes

New `TestVerifyChecksumsStopsAtMaxBytes` (`checksum_test.go`). A counting reader that would serve
64 MiB is given to `VerifyChecksums` with `MaxBytes` 1 and 1 MiB. The call must return
`ErrResponseTooLarge`, and the reader must have served at most `MaxBytes + 1` bytes. Without the
cap the whole 64 MiB would be read: the count fails the test, and the call still returns because
the source is finite, so the test never hangs. An input of exactly `MaxBytes` (3 bytes, `MaxBytes`
3) is read whole and verified against its SHA-256. The extreme values of round 2 (`MaxBytes`
`math.MaxInt64` and `math.MaxInt64 - 1` in `TestChecksumRules`) are unchanged and still pass.

### Other unbounded readers

None found. The package's readers are these:

- **`VerifyChecksums`:** reads a caller's reader, now covered by the test above.
- **`readBody`:** `io.ReadAll` in typed calls, discovery and the token request.
- **The two download `io.Copy` calls.**
- **The drain of a recovered `401` in `doer.Do`.**

The last three all read `resp.Body` returned by the doer, which is always the `boundedBody` (the
limit of the call's kind, or `ErrorBodyBytes` for non-2xx). Those limits are tested by cases 11a,
11b, 11c, 31, 33 and by `TestExtremeByteLimits` and `TestBoundedBodyNearMaxInt64` (round 2). The
`io.LimitReader` in `internal/oolongtest` is test support. `tei` and the option parsers read
strings of bounded length.

## Mutation checks (one-off, in a scratch copy; not committed)

All 3 killed.

| Mutation | Failing tests |
|---|---|
| `src = r` (the tester's F7: the `LimitReader` dropped) | `TestVerifyChecksumsStopsAtMaxBytes` |
| cap at `MaxBytes + 2` | `TestChecksumRules`, `TestVerifyChecksumsStopsAtMaxBytes` |
| cap at `MaxBytes` (no byte past it) | `TestChecksumRules`, `TestVerifyChecksumsStopsAtMaxBytes` |

## Tests run on the head `4ae95d2` (Go 1.26.0, Node 24.14.1), all passing

```sh
export PATH=$PATH:/usr/local/go/bin
gofmt -l .                                   # empty
go vet ./... && go vet -tags integration ./... && go vet -tags live ./...
go build ./...
go mod tidy -diff                            # empty
go generate ./... && git status --porcelain  # teaapi unchanged
go test -count=1 ./...                       # root 70 top-level tests, tei 10
go test -count=3 -race ./...
OOLONG_SERVER_DIR=<oolong>/oolong/oolong-server TEA_INTEGRATION_REQUIRED=1 go test -count=1 -tags integration -run Integration ./...   # 9 of 9, oolong e1c9e6e
TEA_LIVE=1 go test -count=1 -tags live -run Live ./...   # 2 of 2, 2026-11-09, public demo
LC_ALL=C /usr/bin/grep -nP "[^\x00-\x7F]" checksum_test.go   # nothing
```

Only the new test: `go test -count=1 -run 'TestVerifyChecksumsStopsAtMaxBytes|TestChecksum' .`
