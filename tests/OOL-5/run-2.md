# OOL-5 test run 2: tea-client-go discovery (PR #2), re-test of T-1 to T-8

Task: OOL-5 (OOL-3b, child 2 of OOL-3). Verdict: PASSED. All eight items from run 1 are fixed
as asked, and each fix is shown by a mutation that the new tests kill and the round-1 tests did
not. The full suites, the integration tests (harness and in-cluster server) and the live
discovery and failover check pass on the scratch merge. Nothing is OPEN.

Inputs: `impl/OOL-5/notes-2.md` (DETAILED_DESIGN round 2), `tests/OOL-5/run-1.md` (round 1),
`design/OOL-5/architecture-1.md`, the carried `design/OOL-3/architecture-2.md` and
`design/OOL-3/threat_model-2.md` (round 2 of each), and coding principles read at `145008cdcc23`.

## What was tested

| PR | Head tested | Base tip | Merge |
|---|---|---|---|
| https://github.com/oolong-project/tea-client-go/pull/2 | `b6df0fdf1bacdaad9a237122ca2203ed7ccf7903` | `2026-11-oolong` @ `1c85712` (not moved since run 1) | scratch merge `--no-ff` of the head into the base tip: clean (merge commit `3d487bd`, local only; the merged tree is the head's tree) |

History since run 1: the tested head of run 1, `50265c3`, is an ancestor of `b6df0fd`; two
commits were added on top and nothing was rewritten or force-pushed (`refs/pull/2/head` and
`ool-5-tea-client-discovery` both point at `b6df0fd`). `ebdffa2` changes `discovery.go` (T-7,
T-8: 4 lines changed, `codeText` and the `strconv` import removed); `b6df0fd` changes
`discovery_test.go` (+144 lines over all items). Both commits carry an SSH signature and both
ReARM trailers. Against the base the PR still changes the same nine files as in run 1; nothing
under `.github/`, `spec/`, `teaapi/`, `tea-spec/` or `src/generated-nest/`.

oolong-project/oolong is not changed by OOL-5; its `2026-11-oolong` is still `e1c9e6e`, the
commit run 1 used. Integration and live checks use a harness-started server built from that
commit and the CI-built oolong-server image of that commit running with the TEA 1.0 fixtures in
namespace `oolong-tester-ool-4` (the deployment run 1 used). tea-client-go has no CI workflow
yet (deferred by the operator), so there is no CI build to check.

Tools: Go 1.26.10 in the `golang:1.26` pod on the local k3s; Node 24.14.1 for the
harness-started server.

## Results

| # | Check | Result |
|---|---|---|
| 1 | `gofmt -l .` | empty |
| 2 | `go build ./...` | passed |
| 3 | `go vet ./...`, also with `-tags integration` and `-tags live` | clean |
| 4 | `go test -count=1 -v ./...` | passed: 57 top-level tests and examples, 112 subtests, 0 failed, 0 skipped |
| 5 | `go test -race -count=3 ./...` | passed |
| 6 | Flake check on the new and changed tests plus the timing ones (`TestDiscoveryErrorUnwrapsEveryAttempt`, `TestDiscoverPURLValidation`, `TestDiscoverResultsCap`, `TestDiscoverAttemptBound`, `TestDiscoverNoFailover`, `TestDiscoverAttemptTimeout`, `TestDiscoveryTrace`, `TestDiscoverFailover`) at `-count=20`, then `-race -count=10` | passed, no flake |
| 7 | `go mod tidy -diff` | empty |
| 8 | `go generate ./...` then `git diff --exit-code` | no diff |
| 9 | Fuzzers, 30 s each: `FuzzDecodeWellKnown` (about 1.61M execs), `FuzzDecodeDiscovery` (about 1.74M), `FuzzParse` and `FuzzParseLegacyURN` in `tei` | no crasher, no corpus left |
| 10 | Plain-ASCII sweep (`LC_ALL=C /usr/bin/grep -nP "[^\x00-\x7F]"`) over the nine changed files | no hit |
| 11 | No change under `.github/`, `spec/`, `teaapi/`, `tea-spec/`, `src/generated-nest/` | passed |
| 12 | History and commits: `50265c3` an ancestor of the head, nothing rewritten; trailers and signatures on `ebdffa2`, `b6df0fd` | passed |
| 13 | Integration (4.2) through the harness, `OOLONG_SERVER_DIR` at a built oolong-server (`e1c9e6e`), `TEA_INTEGRATION_REQUIRED=1` | 6 of 6 passed |
| 14 | Integration with `OOLONG_BASE_URL` at the in-cluster CI-built server, `TEA_INTEGRATION_REQUIRED=1` | 6 of 6 passed |
| 15 | Harness with no `OOLONG_*`: 6 skips naming the variables; with only `TEA_INTEGRATION_REQUIRED=1`: 6 failures naming them | passed (2 checks) |
| 16 | Mutation checks for T-1 to T-8 (below) | 16 of 16 killed |
| 17 | Live check (below) | 66 of 66 client checks and 6 plain HTTP requests as expected |

Counts in the index: 173 passed (57 unit tests and examples, 12 integration runs, 2 harness
checks, 4 fuzzers, 16 killed mutants, 66 live client checks, 6 plain HTTP checks, 10 other
checks: 1-3, 5-8, 10-12), 0 failed, 0 skipped.

Not run, as in run 1: the live suite (4.3, `-tags live`, `TEA_LIVE=1`), because it reaches the
public demo deployment, which this hop does not touch; it is unchanged since run 1, where it was
read and found to assert what 4.3 case 1 names. The CI `live` job of `ci.yaml` (3.11) is not on
the PR, by the operator's decision for this round (notes-1 departure 6).

## Mutation checks

In a git worktree of the scratch merge, one mutation at a time, reset with `git checkout`
between them; `go test -count=1 .` each time. The same mutants were also run against the round-1
merge (`50265c3` on the base) to show which the old tests let through.

| Item | Mutation | Round 1 tests | Round 2 tests (failing) |
|---|---|---|---|
| T-1 | with a single candidate, an `ErrTooManyResults` answer ends discovery at once (the endpoint is not asked again) | survived | killed (`TestDiscoverResultsCap`, `TestDiscoverPURLValidation/too_many_results`) |
| T-1 | any `ErrTooManyResults` answer ends discovery at once | killed | killed (`TestDiscoverResultsCap`, `TestDiscoverPURLValidation/too_many_results`) |
| T-1 | candidates not tried again after the last one | not re-run | killed (`TestDiscoverAttemptBound`, `TestDiscoverResultsCap`, all four `TestDiscoverPURLValidation` sub-cases) |
| T-2 | `DiscoveryError.Unwrap` leaves out `ErrResponseTooLarge` causes | killed by the well-known subtests only | killed, now also by `TestDiscoverByteLimits/discovery_Content-Length` and `/discovery_chunked` |
| T-2 | `DiscoveryError.Unwrap` returns nothing | not re-run | killed (ten tests, including both discovery byte-limit subtests) |
| T-3 | the bound check at the start of the `/discovery` stage removed (round 1's surviving mutant) | survived | killed (`TestDiscoverAttemptBound`) |
| T-4 | well-known `400` and `403` made failed attempts, `401` kept (round 1's surviving mutant) | survived | killed (`TestDiscoverNoFailover/400_at_the_well-known_stage`, `/403_at_the_well-known_stage`) |
| T-4 | well-known `400` only made a failed attempt | survived | killed (`/400_at_the_well-known_stage`) |
| T-4 | well-known `403` only made a failed attempt | survived | killed (`/403_at_the_well-known_stage`) |
| T-5 | `DiscoverPURL` answers checked with no results cap (round 1's surviving mutant) | survived | killed (`TestDiscoverPURLValidation/too_many_results`) |
| T-5 | `DiscoverPURL` answers' roots not checked against the URL policy | survived | killed (`/http_root_not_under_an_allowance`, `/https_root_then_a_refused_root`) |
| T-5 | `DiscoverPURL` answers not required to be a JSON array | survived | killed (`/not_an_array`) |
| T-6 | `Unwrap` returns only the last attempt's error (round 1's surviving mutant) | survived | killed (`TestDiscoveryErrorUnwrapsEveryAttempt`) |
| T-6 | `Unwrap` returns only the first attempt's error | survived | killed (`TestDiscoveryErrorUnwrapsEveryAttempt`) |
| T-7 | the well-known TEA 404 reason quotes the code and the raw code again | n/a (round 1 behaviour) | killed (`TestWellKnownValidation/TEA-shaped_404`, `TestDiscoveryErrorsCarryNoBody/well-known_TEA_404_code`) |
| T-8 | stops wrapped with the operation again (`fmt.Errorf("%s: %w", op, e)`) | n/a (round 1 behaviour) | killed (all six stop sub-cases of `TestDiscoverNoFailover`, both stages) |

The 16 killed mutants counted are the rows above, run against round 2; the round-1 runs are
evidence for the items and are not counted.

## Live checks

Build: the client at the scratch merge (`b6df0fd` on `1c85712`), run as round 1's throwaway
program (not committed) with five checks added for T-7 and T-8, in the Go pod, against the
in-cluster oolong server (CI-built image of oolong `e1c9e6e`, TEA 1.0 fixtures) and a fixture
HTTP server started by the program inside the same pod (well-known documents and `/discovery`
answers per scenario, plus a TLS server with an untrusted certificate). Below, `<oolong>` is the
in-cluster server's http origin, allowed with `WithInsecureHTTPBase`, and `<fixture>` the
pod-local fixture server. Nothing ran against oolong-demo.rearmhq.com or any shared deployment.

Plain HTTP against `<oolong>`, as expected and as in run 1:

- `GET /.well-known/tea`: `200`, `{"schemaVersion":1,"endpoints":[{"url":"<oolong>","versions":["1.0.0"],"priority":1}]}`.
- `GET /v1.0.0/discovery?tei=<fixture TEI, query-encoded>`: `200`, one result `22222222-...`, server `<oolong>`, `["1.0.0"]`.
- Unknown TEI: `404 {"error":"OBJECT_UNKNOWN"}`. `?purl=pkg:generic/oolong-fixture/widget@1.0.0`: `200`, the same release.
- No parameter: `400 {"error":"INVALID_REQUEST","message":"provide exactly one of tei or purl"}`.
- `GET /nope/v1.0.0/discovery?tei=x`: `404` with Nest's body (`"error":"Not Found","statusCode":404`), the Q-1 shape.

Go client, 66 of 66 passed. Run 1's 61 checks all pass again unchanged (discovery against
oolong's well-known, `SelectServer`, `GetProductRelease`, unknown TEI and PURL, failover for
connection refused, DNS, `503`, html `404`, `429`, `204`, invalid JSON, 33 results, oversized
body, a refused root, priority 2, a redirect to a non-allowed `http` URL and an untrusted
certificate; the results cap; selection; the bound; no failover on stops; every well-known
refusal; time bounds; URL policy and scheme). What changed shows in them as intended:

- The well-known TEA-shaped `404` now ends as `tea: discovery failed at stage well-known after 2
  attempt(s); last: <fixture>/wk/tea404/.well-known/tea: HTTP 404 (TEA error)` (run 1: `HTTP 404
  (TEA error OBJECT_UNKNOWN)`).
- Stops read `discovery: HTTP 401 unknown error code "UNAUTHORIZED"` and `fetching
  .well-known/tea: HTTP 400 INVALID_REQUEST` (run 1: the context twice).
- With a single endpoint answering 33 results and the default bound, the endpoint is asked 3
  times and `errors.Is(err, ErrTooManyResults)` holds (as in run 1; now also held by T-1's test).
- `WithDiscoveryAttempts(1, ...)` against oolong's own well-known makes only the well-known
  request and ends in `*DiscoveryError` at stage api-discovery (Q-2).

Added for this round:

- L11a, T-7: a well-known answering `404 {"error":"ZZMARKERCODE"}` (TEA-shaped, unknown code)
  with attempts 2: both attempts' `Reason` is exactly `HTTP 404 (TEA error)`; the marker is in
  neither the error text, any attempt's reason or URL, nor any trace event; the `*APIError` in
  `Attempt.Err` keeps it as `RawCode`, as design 3.6 allows.
- L11b, T-8: well-known `400`, `401`, `403`: each error contains `fetching .well-known/tea`
  exactly once (3 checks).
- L11c, T-8: a `/discovery` `401`: the error contains `discovery` exactly once.

One further check written for this round was discarded as mis-specified, not counted: it
assumed oolong answers `DiscoverPURL(base, "pkg:")` with `400`; oolong answers `404
OBJECT_UNKNOWN` (so `*NotResolvedError`, which is the design's rule for a TEA 404), and the
client cannot send a request with no parameter. The `400` stop at `/discovery` stays covered by
the fixture server's 401 here and by `TestDiscoverNoFailover`'s `400` sub-case.

## T-1: case 26c's "asked again until the bound" sub-case is not exercised

RESOLVED. `TestDiscoverResultsCap` now runs the one-endpoint sub-case with the default bound and
asserts 3 requests to the endpoint, 3 attempts, each with `errors.Is(a.Err, ErrTooManyResults)`
and the endpoint's URL, and `errors.Is(err, ErrTooManyResults)` on the `*DiscoveryError`. The
mutant that stops retrying a single candidate after a too-many-results answer survived the
round-1 tests and is killed now.

## T-2: case 11b at the discovery stage does not assert errors.Is through the DiscoveryError

RESOLVED. Both "discovery" subtests of `TestDiscoverByteLimits` now also check
`errors.Is(err, ErrResponseTooLarge)` on the returned error; an `Unwrap` that drops that cause
now fails them as well as the well-known subtests.

## T-3: WithDiscoveryAttempts(1) ending Discover after the well-known fetch is untested

RESOLVED. `TestDiscoverAttemptBound` has a bound-1 sub-case with a valid document and an
endpoint that would answer: `*DiscoveryError` at stage api-discovery, no attempt listed, one
well-known request, no `/discovery` request; bound 2 on the same servers succeeds (control).
Round 1's surviving mutant is killed.

## T-4: well-known 400 and 403 stopping with APIError is untested

RESOLVED. `TestDiscoverNoFailover` runs the well-known stop as a table over `400`, `401`, `403`:
`*APIError` with that status, not a `*DiscoveryError`, after one request with the default
bound. Round 1's mutant and the single-status variants are killed. (The table checks
`errors.Is` with the sentinel only for `403`; the status check on the `*APIError` is what Q-4
names, so this is enough.)

## T-5: DiscoverPURL's answer validation is untested

RESOLVED. New `TestDiscoverPURLValidation`: with `DiscoveryResults` 2 and a bound of 2, three
results, an `http` root not under an allowance, an allowed root followed by a refused one, and a
non-array body each give `*DiscoveryError` at stage api-discovery with 2 attempts and 2
requests, with the cause (`ErrTooManyResults`, `ErrInsecureURL`) through `errors.Is` and the
rule in the reason; two results are returned (control). Mutants removing the cap, the root URL
policy or the array check for `DiscoverPURL` only are each killed.

## T-6: DiscoveryError.Unwrap is only tested where every attempt has the same cause

RESOLVED. New `TestDiscoveryErrorUnwrapsEveryAttempt`: a stalled first endpoint (`ErrTimeout`)
and a `503` second (`ErrServer`); both causes hold through the `*DiscoveryError` and `Unwrap()`
gives 2 errors. `Unwrap` returning only the last, or only the first, attempt's error is killed.

## T-7: Attempt.Reason quotes a server-supplied error code, against its doc comment

RESOLVED, by the code change the report offered first. `classifyStatus` gives a well-known
TEA-shaped `404` the fixed reason `HTTP 404 (TEA error)`; `codeText` is gone, so no reason
carries body content and the `Attempt.Reason` doc comment is now true. Checked directly in the
live run (L11a) and by `TestDiscoveryErrorsCarryNoBody`'s new "well-known TEA 404 code" case,
which a mutant restoring the code fails.

## T-8: stop errors repeat their context prefix

RESOLVED. Stops return the `*APIError` itself, whose text already starts with the operation:
`discovery: HTTP 401 ...`, `fetching .well-known/tea: HTTP 400 INVALID_REQUEST` (live run, L8,
L9c, L11b, L11c). `TestDiscoverNoFailover` counts the context once at both stages; a mutant
restoring the double wrap fails all six stop sub-cases.

## Coding principles

The round-2 diff (`discovery.go`, `discovery_test.go`) is `gofmt` clean, `go vet` clean and
plain ASCII; errors keep context once, as the principles' wrapping rule asks; no body content in
reasons or messages. No departure found.

## Threat model

Unchanged since run 1: rows for child 3b (T1, T2, T4, T5, T6, T7, T13) have their listed tests;
the partial assertions run 1 noted for 26c and 11b (T-1, T-2) are now complete. No security case
outside the threat model was added.

## Sub-agents

A sonnet sub-agent ran the suites (checks 1-15) and reported raw results; the mutation checks,
the live check, the judgement against the design and the verdict are this hop's own (opus).
