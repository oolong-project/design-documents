# OOL-5 implementation notes, round 2: fixes for test run 1 (T-1 to T-8)

Task: OOL-5 (child 2 of OOL-3). Answers `tests/OOL-5/run-1.md` (BOARD_TEST_REPORT round 1, REJECTED,
T-1 to T-8 open). Round 1 (`impl/OOL-5/notes-1.md`) still describes the build, the departures and
the earlier mutation results; only what is below changed. Design: `design/OOL-5/architecture-1.md`,
with OOL-3's `design/OOL-3/architecture-2.md` and `design/OOL-3/threat_model-2.md` (round 2).
Coding principles read at `145008cdcc23`.

## Pull request

| PR | Repository | Base | Head commit | Base tip merged |
|---|---|---|---|---|
| https://github.com/oolong-project/tea-client-go/pull/2 | oolong-project/tea-client-go | `2026-11-oolong` | `b6df0fdf1bacdaad9a237122ca2203ed7ccf7903` | `1c85712` (the base has not moved; nothing to merge) |

The tested head of run 1, `50265c3`, is an ancestor of the new head: two commits were added on
top, nothing was rewritten. `ebdffa2` changes `discovery.go` (T-7, T-8); `b6df0fd` changes
`discovery_test.go` (all items). Test against the head: at `ebdffa2` alone, one old assertion of
the T-7 reason still expects the code and fails. No `.github/` content (CI still deferred).
Merge order: one PR.

## Review items

| Id | Status | Commit |
|---|---|---|
| T-1 | fixed | `b6df0fd` |
| T-2 | fixed | `b6df0fd` |
| T-3 | fixed | `b6df0fd` |
| T-4 | fixed | `b6df0fd` |
| T-5 | fixed | `b6df0fd` |
| T-6 | fixed | `b6df0fd` |
| T-7 | fixed (code changed, as the report's first option) | `ebdffa2`, test `b6df0fd` |
| T-8 | fixed | `ebdffa2`, test `b6df0fd` |

Nothing is disputed and nothing remains open.

### T-1: case 26c now asks the one endpoint again until the bound

`TestDiscoverResultsCap`: the one-endpoint sub-case runs with the default bound of 4 (the
well-known fetch and three `/discovery` requests) and asserts 3 requests to the endpoint,
3 listed attempts, each with `errors.Is(a.Err, ErrTooManyResults)` and the endpoint's URL, and
`errors.Is(err, ErrTooManyResults)` on the `*DiscoveryError`.

### T-2: case 11b asserts errors.Is through the DiscoveryError at the discovery stage

`TestDiscoverByteLimits`, both "discovery" subtests (Content-Length and chunked) now also check
`errors.Is(err, ErrResponseTooLarge)` on the returned error.

### T-3: WithDiscoveryAttempts(1) is tested

`TestDiscoverAttemptBound`: a bound of 1 with a valid document and an endpoint that would answer
gives `*DiscoveryError` at stage api-discovery with no listed attempt, one well-known request and
no `/discovery` request; a bound of 2 on the same servers succeeds (the control).

### T-4: well-known 400 and 403 stop like 401

`TestDiscoverNoFailover`: the well-known sub-case is a table over 400, 401 and 403; each gives an
`*APIError` with that status, not a `*DiscoveryError`, after exactly one request (the default
bound would allow 4).

### T-5: DiscoverPURL's answer validation is tested

New `TestDiscoverPURLValidation`: with `DiscoveryResults` 2 and a bound of 2, an answer with 3
results (`ErrTooManyResults`), an `http` root not under an allowance (`ErrInsecureURL`), an `https`
root followed by a refused root (refused whole, reason names server 1) and a non-array body each
give `*DiscoveryError` at stage api-discovery with 2 attempts and 2 requests; 2 results are
returned (the control).

### T-6: DiscoveryError.Unwrap is tested with different causes

New `TestDiscoveryErrorUnwrapsEveryAttempt`: a first endpoint that stalls (`AttemptTimeout`
200 ms, `ErrTimeout`) and a second that answers `503` (`ErrServer`), bound 3. The attempts carry
those two causes in that order; `errors.Is(err, ErrTimeout)` and `errors.Is(err, ErrServer)` both
hold on the `*DiscoveryError`, and `Unwrap()` returns 2 errors.

### T-7: Attempt.Reason no longer quotes a server-supplied code

`classifyStatus` (`discovery.go`): a TEA-shaped `404` that is a failed attempt (the well-known
stage) now has the fixed reason `HTTP 404 (TEA error)`; the sanitised code stays on the `*APIError`
in `Attempt.Err`, which design 3.6 allows. The `Attempt.Reason` doc comment ("never response-body
content") is now true and unchanged; `codeText` is removed. Tests: `TestWellKnownValidation`'s
TEA-shaped 404 sub-case asserts the exact reason, and `TestDiscoveryErrorsCarryNoBody` gains a
"well-known TEA 404 code" case (a marker as the `error` value), checked in the error text, every
attempt's reason and URL, and every trace event.

### T-8: stop errors carry their context once

`classifyStatus`: a `400`, `401` or `403` at either stage returns the `*APIError` itself, whose
text already starts with the operation (`discovery` or `fetching .well-known/tea`). Tests: the
`/discovery` stop sub-cases of `TestDiscoverNoFailover` assert that `discovery` occurs once in the
error text, the well-known ones that `fetching .well-known/tea` occurs once.

## Mutation checks (one-off, in a scratch copy; not committed)

All 9 mutants killed.

| Item | Mutation | Failing tests |
|---|---|---|
| T-1 | candidates not retried after the last one | `TestDiscoverResultsCap` |
| T-2 | `DiscoveryError.Unwrap` returns nothing | `TestDiscoverByteLimits/discovery_Content-Length`, `/discovery_chunked` |
| T-3 | the bound check at the start of the `/discovery` stage removed (the tester's mutant) | `TestDiscoverAttemptBound` |
| T-4 | well-known 400 and 403 made failed attempts, 401 kept (the tester's mutant) | `TestDiscoverNoFailover/400_at_the_well-known_stage`, `/403_at_the_well-known_stage` |
| T-5 | `DiscoverPURL` runs with no results cap | `TestDiscoverPURLValidation/too_many_results` |
| T-5 | `DiscoverPURL` runs with an allowance for the refused http origin | `TestDiscoverPURLValidation/http_root_not_under_an_allowance`, `/https_root_then_a_refused_root` |
| T-6 | `Unwrap` returns only the last attempt's error (the tester's mutant) | `TestDiscoveryErrorUnwrapsEveryAttempt` |
| T-7 | the reason quotes the code again | `TestDiscoveryErrorsCarryNoBody/well-known_TEA_404_code`, `TestWellKnownValidation/TEA-shaped_404` |
| T-8 | the stop wrapped with the operation again | `TestDiscoverNoFailover` (400, 401, 403 at both stages) |

## Tests run on the head `b6df0fd` (Go 1.26.0, Node 24.14.1), all passing

```sh
gofmt -l .                                   # empty
go vet ./... && go vet -tags integration ./... && go vet -tags live ./...
go build ./...
go mod tidy -diff                            # empty
go generate ./... && git diff --exit-code    # teaapi unchanged
go test -count=1 ./...                       # root 47 tests (incl. 2 fuzz seed runs), tei 10
go test -race -count=3 ./...
go test -count=20 -run 'TestDiscoveryErrorUnwrapsEveryAttempt|TestDiscoverPURLValidation|TestDiscoverResultsCap' .
go test -run '^$' -fuzz='^FuzzDecodeWellKnown$' -fuzztime=20s .
go test -run '^$' -fuzz='^FuzzDecodeDiscovery$' -fuzztime=20s .
LC_ALL=C /usr/bin/grep -nP "[^\x00-\x7F]" discovery.go discovery_test.go   # nothing
# oolong 2026-11-oolong @ e1c9e6e (unchanged), npm ci && npm run build in oolong/oolong-server
OOLONG_SERVER_DIR=<oolong>/oolong/oolong-server TEA_INTEGRATION_REQUIRED=1 go test -count=1 -tags integration -run Integration ./...   # 6 of 6
TEA_LIVE=1 go test -count=1 -tags live -run Live ./...   # 1 of 1, 2026-11-09, public demo
```

Only the new and changed tests:
`go test -count=1 -run 'TestDiscoverResultsCap|TestDiscoverByteLimits|TestDiscoverAttemptBound|TestDiscoverNoFailover|TestDiscoverPURLValidation|TestDiscoveryErrorUnwrapsEveryAttempt|TestDiscoveryErrorsCarryNoBody|TestWellKnownValidation' .`

## For the tester

The only changes visible to callers are message texts: a well-known TEA-shaped 404's
`Attempt.Reason` is `HTTP 404 (TEA error)`, and stop errors read e.g. `discovery: HTTP 401` instead
of `discovery: discovery: HTTP 401`. No new dependency, no change to `teaapi`, `transport.go`,
README or the integration and live tests.
