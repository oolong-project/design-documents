# OOL-6 implementation notes, round 2: fixes for test run 1 (T-1 to T-5)

Task: OOL-6 (OOL-3c, child 3 of OOL-3). Answers `tests/OOL-6/run-1.md` (BOARD_TEST_REPORT round 1,
REJECTED, T-1 to T-5 open). Round 1 (`impl/OOL-6/notes-1.md`) still describes the build, the
departures and the earlier mutation results; only what is below changed. Design carried from
OOL-3: `design/OOL-3/architecture-2.md`, `design/OOL-3/threat_model-2.md` (round 2). Coding
principles read at `145008cdcc23`.

## Pull request

| PR | Repository | Base | Head commit | Base tip merged |
|---|---|---|---|---|
| https://github.com/oolong-project/tea-client-go/pull/3 | oolong-project/tea-client-go | `2026-11-oolong` | `1255c66434a4555fec43854b8193ba650a25fa9e` | `6d422b1` (the base has not moved; nothing to merge) |

The tested head of run 1, `55a19b2`, is an ancestor of the new head: one commit, `1255c66`, was
added on top; nothing was rewritten or force-pushed. It changes `auth.go`, `checksum.go`,
`transport.go` and the tests `auth_test.go`, `checksum_test.go`, `download_test.go`,
`transport_test.go`. No `.github/` content (CI still deferred), no `teaapi`, `spec/` or `go.mod`
change. Merge order: this PR only.

## Review items

| Id | Status | Commit |
|---|---|---|
| T-1 | fixed | `1255c66` |
| T-2 | fixed | `1255c66` |
| T-3 | fixed (tests; the code was right) | `1255c66` |
| T-4 | fixed (test) | `1255c66` |
| T-5 | fixed (test) | `1255c66` |

Nothing is disputed and nothing remains open.

### T-1: VerifyChecksums with MaxBytes = math.MaxInt64

`checksum.go`: the `io.LimitReader(r, o.MaxBytes+1)` that detects input longer than the cap is
used only for `0 < MaxBytes < math.MaxInt64`; at `math.MaxInt64` there is no room for the extra
byte and no input can be longer, so the reader is not limited. `n > o.MaxBytes` is unchanged.
Test: `TestChecksumRules` (`checksum_test.go`) runs `MaxBytes` `math.MaxInt64` and
`math.MaxInt64 - 1`: the genuine `"abc"` SHA-256 verifies, and the empty input's SHA-256 (what
the bug compared against) is a `*ChecksumError`.

### T-2: the bounded body reader with a limit of math.MaxInt64

`transport.go`, `boundedBody.Read`: the room left is `rem := limit - n` (never negative, `n` stops
at the limit); a read is sliced to `rem+1` only when `len(p) > rem`, so `rem+1 <= len(p)` cannot
overflow. The count no longer adds before comparing: when `k > rem` the read is cut to `rem` and
the body is too large (strict) or cut (error bodies); otherwise `n += k`, which stays at most the
limit. This covers every kind (JSON, error bodies, well-known, discovery, token, download,
signature) and both entry points, `Limits` and `DownloadOptions.MaxBytes`. Tests:

- `TestDownloadExtremeLimits` (`download_test.go`): `DownloadArtifact` with `MaxBytes`
  `math.MaxInt64`, with `Limits.DownloadBytes` `math.MaxInt64`, and `DownloadArtifactToFile` with
  `MaxBytes` `math.MaxInt64` each verify the fixture bytes, and with a wrong checksum each give a
  mismatch; `DownloadSignature` with `MaxBytes` and with `Limits.SignatureBytes` `math.MaxInt64`.
- `TestExtremeByteLimits` (`transport_test.go`): every byte limit `math.MaxInt64` (and the count
  limits `math.MaxInt`): a typed JSON call, `Generated()` download and signature, a `404` under
  `ErrorBodyBytes` `math.MaxInt64` still classified, and a whole `FullFlow` (well-known,
  `/discovery`, JSON) on the flow fixture.
- `TestBoundedBodyNearMaxInt64` (`transport_test.go`): the reader alone with `n` three bytes below
  a limit of `math.MaxInt64` and ten bytes to read: it returns 3 bytes, `n` at the limit, and
  `ErrResponseTooLarge` (strict) or `io.EOF` (error bodies). A real body cannot get there (about
  8 EiB), so the count overflow is held by this unit test.

### Overflow audit (all hand-written packages)

Every place a limit or size plus one, a `Read` length, a byte count or a duration is computed from a
caller- or server-supplied value:

| Site | Value from | Finding |
|---|---|---|
| `checksum.go`, `LimitReader(r, MaxBytes+1)` | caller | T-1, fixed |
| `transport.go`, `boundedBody.Read` slice length | caller (`Limits`, `MaxBytes`) | T-2, fixed |
| `transport.go`, `boundedBody.Read` count `n + k` against the limit | caller | overflowed only within `len(p)` of `math.MaxInt64`; fixed with T-2 |
| `auth.go`, `time.Duration(expires_in) * time.Second` | **server** (`token-response`) | **new site**: an `expires_in` above `math.MaxInt64 / 1e9` seconds wrapped (to a negative lifetime, so the token was not cached, or to an arbitrary one). Now saturated at the largest `time.Duration`. Test: `TestAPIKeyTokenCache` stores `math.MaxInt` and `math.MaxInt64/1e9 + 1` and expects the token cached 100 years later; `-5` is not cached |
| `transport.go`, `resp.ContentLength > limit`; `checksum.go` `n > MaxBytes` | server, caller | comparisons only, no arithmetic: safe |
| `discovery.go`, `backoffDelay` doubling | caller (`WithDiscoveryAttempts`) | doubles only while below 4 s, then `min`: cannot overflow |
| `pagination.go` page counter, redirect count, `Candidates` and `FlowComponents` checks, results cap | caller | comparisons of counters that grow by one per request: safe |
| time bounds (`AttemptTimeout`, `CallTimeout`, `IdleTimeout`) | caller | passed to `context.WithTimeoutCause` / `time.AfterFunc` unchanged: safe at `math.MaxInt64` |
| `tei` length and percent-escape indexing, `sanitize`, `truncateArg`, `parseChallenge` | input, server | index arithmetic bounded by `len` of the string: safe |

### T-3: the anonymous mark on later hops and for the 401 recovery

The code was right; tests now hold it. `TestDownloadSource`: an external `url` on another server
that answers `302` to a URL under the bearer's root: the redirect hop reaches the root server
without `Authorization` (and the external server gets none). `TestAPIKeyRecovery`: an external
`url` under the API key's root answered `401` with a Bearer challenge gives `ErrUnauthorized`,
one request without `Authorization`, and no token request.

### T-4: the temporary file's directory

`TestDownloadArtifactToFile` gains a stalled download (1 KiB then nothing, `IdleTimeout` 1 s) to
`<dir>/stalled.json`: while it stalls, a file named `.stalled.json.tea-*` is seen in `<dir>`;
the call then ends with `ErrTimeout` and `<dir>` is empty.

### T-5: the retry with a token that is not cached

New `TestAPIKeyUncachedToken` (`auth_test.go`): the key server issues tokens with
`expires_in` 30. Two product requests carry, in order, no token, `issued-access-token-1` (the
retry), no token (nothing cached), `issued-access-token-2`; two token requests in all. The key
server's `expires_in` is now settable (default 3600, as before).

## Mutation checks (one-off, in a scratch copy; not committed)

All 9 killed.

| Item | Mutation | Failing tests |
|---|---|---|
| T-1 | the `LimitReader(MaxBytes+1)` used for every `MaxBytes > 0` again | `TestChecksumRules` |
| T-2 | the slice at `limit - n + 1` again | `TestDownloadExtremeLimits` (the run-1 panic) |
| T-2 | the count compared as `n + k > limit` again | `TestBoundedBodyNearMaxInt64` |
| audit | `expires_in` not saturated | `TestAPIKeyTokenCache` |
| T-3 | anonymous mark only on the first hop (the tester's mutant) | `TestDownloadSource` |
| T-3 | `anonymous` removed from the recovery guard (the tester's mutant) | `TestAPIKeyRecovery` |
| T-4 | `os.CreateTemp("", ...)` (the tester's mutant) | `TestDownloadArtifactToFile` |
| T-5 | retry cloned without the token override (the tester's mutant) | `TestAPIKeyUncachedToken` |

The first run of the count mutation survived (only reachable near 8 EiB through HTTP); the unit
test `TestBoundedBodyNearMaxInt64` was added for it before the commit.

## Tests run on the head `1255c66` (Go 1.26.0, Node 24.14.1), all passing

```sh
export PATH=$PATH:/usr/local/go/bin
gofmt -l .                                   # empty
go vet ./... && go vet -tags integration ./... && go vet -tags live ./...
go build ./...
go mod tidy -diff                            # empty
go generate ./... && git status --porcelain  # teaapi unchanged
go test -count=1 ./...                       # root 69 top-level tests, tei 10
go test -count=3 -race ./...
go test -count=20 -run 'TestDownloadIdleTimeout|TestDownloadBounds|TestDownloadArtifactToFile|TestAPIKey|TestFlowFromRelease|TestDownloadSource' .
go test -count=10 -race -run 'TestDownloadArtifactToFile|TestAPIKey' .
go test -run '^$' -fuzz '^FuzzDecodeDiscovery$' -fuzztime 10s .
go test -run '^$' -fuzz '^FuzzDecodeWellKnown$' -fuzztime 10s .
go test -run '^$' -fuzz '^FuzzParse$' -fuzztime 10s ./tei
# oolong 2026-11-oolong @ e1c9e6e (unchanged)
OOLONG_SERVER_DIR=<oolong>/oolong/oolong-server TEA_INTEGRATION_REQUIRED=1 go test -count=1 -tags integration -run Integration ./...   # 9 of 9
TEA_LIVE=1 go test -count=1 -tags live -run Live ./...   # 2 of 2, 2026-11-09, public demo
LC_ALL=C /usr/bin/grep -nP "[^\x00-\x7F]" <changed files>   # nothing
```

Only the new and changed tests:
`go test -count=1 -run 'TestChecksumRules|TestDownloadExtremeLimits|TestExtremeByteLimits|TestBoundedBodyNearMaxInt64|TestAPIKeyTokenCache|TestDownloadSource|TestAPIKeyRecovery|TestDownloadArtifactToFile|TestAPIKeyUncachedToken' .`

## For the tester

Callers see no change other than that limits of `math.MaxInt64` now work, and a huge server
`expires_in` caches the token instead of not caching it. The stalled-file sub-case of
`TestDownloadArtifactToFile` takes about 1 s (the idle bound).
