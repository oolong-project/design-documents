# OOL-5 test run 1: tea-client-go discovery (PR #2)

Task: OOL-5 (OOL-3b, child 2 of OOL-3). Verdict: REJECTED. Discovery works as designed in every
unit, integration and live check run here (nothing failed when run), but eight items are filed
against the tests and two doc or message details: five rules the design decides (3.6 steps 5-6,
Q-2, Q-4) are untested, or a design-listed case asserts only part of what it names, and in each
the mutation that breaks the rule leaves the suite green. All items are about the code
(`DETAILED_DESIGN`); none needs a design change.

Inputs: `impl/OOL-5/notes-1.md` (DETAILED_DESIGN round 1), `design/OOL-5/architecture-1.md`
(the child's ARCHITECTURE round 1, answering Q-1 to Q-4), the carried
`design/OOL-3/architecture-2.md` and `design/OOL-3/threat_model-2.md` (round 2 of each), and
coding principles read at `145008cdcc23`.

## What was tested

| PR | Head tested | Base tip | Merge |
|---|---|---|---|
| https://github.com/oolong-project/tea-client-go/pull/2 | `50265c322aaec2123c664fef9ea3fa277288325d` | `2026-11-oolong` @ `1c85712` (not moved) | scratch merge `--no-ff` of the head into the base tip (clean; the base is an ancestor of the head, so the merged tree is the head's tree) |

Commits on the PR: `8bd03e1`, `ceea128`, `7269db8`, `16ee9fa`, `50265c3`. Each is signed (one
SSH key) and carries both ReARM trailers. Nine files changed against the base: `README.md`,
`client.go`, `discovery.go`, `discovery_fuzz_test.go`, `discovery_test.go`, `example_test.go`,
`integration_test.go`, `live_test.go`, `transport.go`. Nothing under `.github/`, `spec/` or
`teaapi/`.

oolong-project/oolong is not changed by OOL-5. Integration and live checks use oolong
`2026-11-oolong` @ `e1c9e6e` (unchanged since OOL-4): a harness-started server built from that
commit, and the CI-built oolong-server image of that commit running with the TEA 1.0 fixtures in
namespace `oolong-tester-ool-4` (the deployment OOL-4 run 2 used). The repository has no CI
workflow yet (deferred by the operator, below), so there is no CI image build to check.

Tools: Go 1.26.10 in a `golang:1.26` pod on the local k3s; Node 24 for the harness-started
server. Sub-agents: a sonnet agent ran the suites; an opus agent judged case coverage and ran
mutations; the live check and the verdict are this hop's own.

## Results

| # | Check | Result |
|---|---|---|
| 1 | `gofmt -l .` | empty |
| 2 | `go build ./...` | passed |
| 3 | `go vet ./...`, also with `-tags integration` and `-tags live` | clean |
| 4 | `go test -count=1 -v ./...` | passed: 55 top-level tests and examples, 105 subtests, 0 failed, 0 skipped |
| 5 | `go test -race -count=3 ./...` | passed |
| 6 | Flake check: `TestDiscoverAttemptTimeout`, `TestDiscoverAttemptBound`, `TestDiscoveryTrace`, `TestDiscoverFailover` at `-count=20`, then `-race -count=10` | passed, no flake |
| 7 | `go mod tidy -diff` | empty |
| 8 | `go generate ./...` then `git diff --exit-code` | no diff |
| 9 | Fuzzers, 30 s each: `FuzzDecodeWellKnown` (about 1.77M execs), `FuzzDecodeDiscovery` (about 1.86M), `FuzzParse` and `FuzzParseLegacyURN` in `tei` | no crasher, no corpus left |
| 10 | Plain-ASCII sweep (`LC_ALL=C grep -nP "[^\x00-\x7F]"`) over the nine changed files | no hit |
| 11 | No change under `.github/`, `spec/`, `teaapi/`, `tea-spec/`, `src/generated-nest/` | passed |
| 12 | Commit trailers and signatures on the five PR commits | passed |
| 13 | Integration (4.2) through the harness, `OOLONG_SERVER_DIR` at a built oolong-server (`e1c9e6e`), `TEA_INTEGRATION_REQUIRED=1` | 6 of 6 passed (OOL-4's 3 and discovery cases 1-3) |
| 14 | Integration with `OOLONG_BASE_URL` at the in-cluster CI-built server | 6 of 6 passed |
| 15 | Harness with no `OOLONG_*`: 6 skips with how to set them; with only `TEA_INTEGRATION_REQUIRED=1`: 6 failures naming the variables | passed (2 checks) |
| 16 | Case coverage against design 4.1, 4.2, 4.3 (below) | every listed case has a test; two assert only part of what they name (T-1, T-2) |
| 17 | Mutation checks (below): 13 of the coder's 26 mutants re-run, 19 new | 31 killed or equivalent, 4 survived (T-3 to T-6) |
| 18 | Live check (below) | 61 of 61 client checks and 6 plain HTTP requests as expected |

Counts in the index: 177 passed (55 unit tests and examples, 12 integration runs, 2 harness
checks, 4 fuzzers, 27 killed mutants, 61 live client checks, 6 plain HTTP checks, 10 other checks:
1-3, 5-8, 10-12), 8 failed (T-1 to T-8, one per item), 0 skipped. The one equivalent mutant is
counted nowhere.

The live suite (4.3, `-tags live`, `TEA_LIVE=1`) was not run: it reaches the public demo
deployment, which this hop does not touch. `TestLiveDiscover` was read and asserts what 4.3 case 1
names (1 to `DiscoveryResults` results, non-nil uuid, at least one server, `SelectServer` gives
an `https` base and `1.0.0`), gated on the tag and `TEA_LIVE=1`.

## Case coverage (design 4.1, 4.2, 4.3)

Tests are in `discovery_test.go` unless named.

| Case | Test(s) | Verdict |
|---|---|---|
| 9b | `TestDiscoverLocalReARMShape`, `TestSelectServerUnderAllowance` | covered (via 21b and 27b) |
| 11b | `TestDiscoverByteLimits` | partly: see T-2 |
| 13b | `TestDiscoveryCredentials` | covered, with positive controls |
| 17 | `FuzzDecodeWellKnown`, `FuzzDecodeDiscovery` | covered; they also check that what is accepted is usable |
| 21 | `TestDiscoverRequestForm` | covered (raw query, TEI decoded once, literal `+`) |
| 21b | `TestDiscoverLocalReARMShape` | covered, including no request to `OTHER` |
| 22 | `TestDiscoverSelection`, `TestSortCandidates` | covered |
| 23 | `TestWellKnownValidation` | covered: every refusal listed, unknown property accepted, TEA-shaped 404 with attempts 2 gives 2 requests |
| 24 | `TestDiscoverFailover`, `TestDiscoverUppercaseUUID` | covered: every listed sub-case, incl. 429, 204, refused-whole with the URL-policy reason, TLS failure |
| 25 | `TestDiscoverNoFailover` | covered (TEA 404, 401, 403, 400; no request to the second endpoint) |
| 26 | `TestDiscoverAttemptBound`, `TestBackoffDelay` | covered (exact counts, URLs without query, doubling backoff, cancel during backoff) |
| 26b | `TestDiscoverAttemptTimeout` | covered |
| 26c | `TestDiscoverResultsCap` | partly: see T-1 |
| 27, 27b | `TestDiscoverResultsAndSelectServer`, `TestSelectServerUnderAllowance` | covered (no fallback past a refused first server; `purl=` not `tei=`) |
| 39b | `TestDiscoveryTrace` | covered (exact event order; no TEI, no `?`) |
| 4.2 case 1-3 | `TestIntegrationDiscover`, `TestIntegrationDiscoverUnknownAndPURL`, `TestIntegrationDiscoverWrongPath` | covered; case 3 asserts the amended Q-1 rule (`*NotResolvedError`, `RawCode` `Not Found`, one discovery request) |
| 4.3 case 1 | `TestLiveDiscover` | covered (read, not run) |

## Mutation checks

In a git worktree of the scratch merge, one mutation at a time, the discovery tests re-run.

Re-run from the coder's table, all killed: TEA 404 at `/discovery` fails over; 400/401/403 fail
over; results cap removed; successful well-known not counted; no round-robin; no backoff wait;
`SelectServer` skips D4; well-known TEA-shaped 404 stops; 429 stops; any 2xx is success; caller
context ending counted as a failed attempt; TEI not query-encoded; refused root skipped instead of
refusing the answer whole.

New, killed: priority ties reversed; backoff cap removed; backoff cap 8 s; results cap off by one
(32 refused); default `DiscoveryResults` 33; query encoding applied twice; build metadata makes a
version unsupported; `Attempt.URL` keeps the query; D4 skipped for well-known `url`s;
`SelectServer` falls back to the first allowed server; backoff ignores the context;
`SelectServer` treats an absent priority as 0; domain not lowercased. Redirect limit raised to
1000: killed only by OOL-4's `TestRedirects` (the doer is shared; not an item).

Equivalent: the `DiscoverPURL` base D4 pre-check removed (the doer's URL check still refuses the
base before any request, with the same error).

Survived (items): the bound-1 pre-check in `discover` removed (T-3); well-known 400 and 403 turned
into failed attempts (T-4); `DiscoverPURL` without the results cap (T-5); `DiscoveryError.Unwrap`
returning only the last attempt's error (T-6).

## Live checks

Build: the client at the scratch merge (`50265c3` on `1c85712`), run as a throwaway program (not
committed) in the Go pod, against the in-cluster oolong server (CI-built image of oolong
`e1c9e6e`, TEA 1.0 fixtures) and a fixture HTTP server started by the program inside the same
pod, which serves well-known documents and `/discovery` answers per scenario, plus a TLS server
with an untrusted certificate. Below, `<oolong>` is the in-cluster server's http origin, allowed
with `WithInsecureHTTPBase`. Nothing ran against oolong-demo.rearmhq.com or any shared deployment.

Plain HTTP against `<oolong>`, as expected:

- `GET /.well-known/tea`: `200`, `{"schemaVersion":1,"endpoints":[{"url":"<oolong>","versions":["1.0.0"],"priority":1}]}`.
- `GET /v1.0.0/discovery?tei=<fixture TEI, query-encoded>`: `200`, one result `22222222-...`, server `<oolong>`, `["1.0.0"]`.
- Unknown TEI: `404 {"error":"OBJECT_UNKNOWN"}`. `?purl=pkg:generic/oolong-fixture/widget@1.0.0`: `200`, the same release.
- No parameter: `400 {"error":"INVALID_REQUEST","message":"provide exactly one of tei or purl"}`.
- `GET /nope/v1.0.0/discovery?tei=x`: `404` with Nest's body `{"message":...,"error":"Not Found","statusCode":404}` (the Q-1 shape).

Go client, 61 of 61 passed:

- Against oolong's own well-known: `Discover` of the fixture TEI gives one result; `SelectServer`
  gives `<oolong>` and `1.0.0`; `GetProductRelease` there succeeds; no discovery trace event holds
  the TEI or a `?`. Unknown TEI: `*NotResolvedError` `OBJECT_UNKNOWN` after one discovery request.
  `DiscoverPURL` finds the release; an unknown PURL is `*NotResolvedError`; `not-a-purl` is
  `ErrInvalidArgument` before any request.
- Failover, a well-known document listing a failing endpoint (priority 1) and `<oolong>` (0.5): the
  first fails exactly once and `<oolong>` answers, within about 25-45 ms, for each of: connection
  refused (dead port), a host that does not resolve (DNS), `503`, `text/html` `404`, `429`, `204`,
  invalid JSON, 33 results (over the default cap), a body over `DiscoveryBytes`, a result listing
  `<oolong>` next to an `http` root not under an allowance (refused whole; reason "server 1:
  rootUrl refused by the URL policy"), a server `priority` of 2, a `302` to an `http` URL not
  under an allowance, and a TLS server with an untrusted certificate.
- Results cap: one endpoint answering 33 results is asked 3 times (round-robin to the bound) and
  `errors.Is(err, ErrTooManyResults)` holds; 32 results are accepted; with `DiscoveryResults` 1,
  32 results are refused and oolong's single result is accepted; `DiscoverPURL` against a base
  answering 33 results ends in `ErrTooManyResults`.
- Selection: `[2.0.0 prio 1, dead 1.0.0 prio 0.2, <oolong> 1.0.0 prio 0.9]` goes to `<oolong>`
  with no request to the other two; `1.0.0-rc.1` only: `*DiscoveryError` at well-known, no
  discovery request; `1.0.0+build.7` is used verbatim (`/v1.0.0+build.7/discovery`; oolong
  answers its framework 404, so `*NotResolvedError`, as Q-1 decides).
- Bound: one dead endpoint with the default bound gives one well-known request, 3 discovery
  requests, 2 backoffs and a `*DiscoveryError` listing 3 attempts with no query; with
  `WithDiscoveryAttempts(1, ...)` only the well-known request is made (Q-2).
- No failover: a TEA `404` at the first endpoint (`*NotResolvedError`), a `401` (`*APIError`),
  and `<oolong>/nope` (`*NotResolvedError`, `RawCode` `Not Found`, Q-1) make no request to
  `<oolong>`.
- Well-known stage: refused as a failed attempt at stage well-known, with no discovery request:
  not JSON, `schemaVersion` 2 and `"1"`, empty `endpoints`, a `url` with a query, userinfo, a
  trailing slash, a dot segment, an uppercase scheme, an `http` url not under an allowance,
  version `v1.0.0`, priority 1.5, a document over `WellKnownBytes`, a `302` to a non-allowed
  `http` URL, and a TEA-shaped `404` (two requests with attempts 2, Q-4). Unknown properties are
  accepted. `400`, `401` and `403` at the well-known location stop with `*APIError` after one
  request (Q-4). A well-known that stalls after its headers times out on each attempt
  (`ErrTimeout`, `AttemptTimeout` 300 ms) and the bound ends discovery in about 1.3 s; a stalled
  first `/discovery` with a caller `http.Client` that has no timeouts fails over in about 320 ms;
  a caller deadline during a stall ends `Discover` with `context.DeadlineExceeded`, not
  `ErrTimeout`, and no failed attempt.
- URL policy and scheme: with no `WithWellKnownLocation`, the client requests only
  `https://<domain>/.well-known/tea` (never http); `SelectServer` on a client without the
  `<oolong>` allowance refuses the result with `*URLPolicyError`, and `API` refuses that base; a
  well-known document listing a non-allowed `http` base is refused with no request to it;
  `WithWellKnownLocation` with an `http` location and no allowance is refused by `New`
  (`ErrInvalidOption`).

The live run is where T-3 to T-5's behaviour was checked: the code does what the design says;
the items are about the tests that should hold it there.

## T-1: case 26c's "asked again until the bound" sub-case is not exercised

`discovery_test.go:917` (`TestDiscoverResultsCap`). Design 4.1 case 26c: "with one endpoint only
and attempts exhausted (that endpoint asked again until the bound, step 6),
`errors.Is(err, ErrTooManyResults)` holds on the `*DiscoveryError`". The test uses
`WithDiscoveryAttempts(2, ...)`, which is the well-known fetch plus one `/discovery` request, and
asserts `many.count() != 1`: the endpoint is never asked again, so `errors.Is` is only checked
over a single attempt. Run it with the default bound (or 3+) and assert the repeated requests
(the live check saw 3 requests and `ErrTooManyResults`).

## T-2: case 11b at the discovery stage does not assert errors.Is through the DiscoveryError

`discovery_test.go:979` (`TestDiscoverByteLimits`, "discovery" subtests). Design 4.1 case 11b:
the failed attempt's `Attempt.Err` is `ErrResponseTooLarge` "(`errors.Is` through
`*DiscoveryError`)". The well-known subtest checks `errors.Is(err, ErrResponseTooLarge)`; the
discovery subtest checks only `de.Attempts[0].Err`. Add the same assertion on `err`.

## T-3: WithDiscoveryAttempts(1) ending Discover after the well-known fetch is untested

`discovery.go:380` (the `exhausted()` check at the start of `discover`); doc at `client.go:263`.
Design 3.4 and 3.6 step 6 (Q-2): "With a bound of 1, `Discover` makes the well-known request only
and ends in `*DiscoveryError`". No test sets the bound to 1 with a valid well-known document.
Removing the check (so a `/discovery` request is made and can succeed) leaves the suite green.
Add a sub-case: attempts 1, valid document, assert `*DiscoveryError` and no discovery request.

## T-4: well-known 400 and 403 stopping with APIError is untested

`discovery.go:443` (`classifyStatus`), test at `discovery_test.go:640`. Design 3.6, the
well-known paragraph after step 5 (Q-4): "`400`, `401` and `403` there stop with `*APIError`, as
at `/discovery`". Only `401` at the well-known stage is tested; making `400` and `403` failed
attempts at the well-known stage (keeping `401`) leaves the suite green. Run the subtest for `400`
and `403` too (one request, `*APIError`, not `*DiscoveryError`).

## T-5: DiscoverPURL's answer validation is untested

`discovery.go:182`. Design 3.6 step 5: "every `DiscoveryResult` that `Discover` or `DiscoverPURL`
returns lists only roots this client's D4 accepts", and the results cap applies to every
`/discovery` answer. No test gives `DiscoverPURL` an invalid answer (too many results, a refused
root, a bad body); running `DiscoverPURL` without the `DiscoveryResults` cap leaves the suite
green. Add at least the cap and a refused-root case for `DiscoverPURL`.

## T-6: DiscoveryError.Unwrap is only tested where every attempt has the same cause

`discovery.go:91`. Design 3.6: `Unwrap() []error` returns "each `Attempt.Err`, so
`errors.Is(err, ErrTimeout)` etc. work". Every test that uses `errors.Is` through a
`*DiscoveryError` has attempts of one kind, so an `Unwrap` that returns only the last attempt's
error leaves the suite green. Add one assertion where the cause sits in an earlier attempt (for
example a stalled first endpoint, `ErrTimeout`, then a `503`).

## T-7: Attempt.Reason quotes a server-supplied error code, against its doc comment

`discovery.go:450` and `discovery.go:67`. The doc comment of `Attempt.Reason` says "never
response-body content" (notes-1 says the same), but a TEA-shaped `404` at the well-known stage
gets the reason `HTTP 404 (TEA error <code>)`, where an unknown code is `RawCode` from the body
(sanitised, at most 64 bytes); it reaches `Attempt.Reason`, the `TraceAttemptFailed` event and
`DiscoveryError.Error()`. Design 3.6 allows the sanitised code in `APIError` and
`NotResolvedError`, so this is not a leak against the design, but the code and its documented
contract disagree and `TestDiscoveryErrorsCarryNoBody` does not cover the path. Either drop the
code from the reason or correct the comment and the notes.

## T-8: stop errors repeat their context prefix

`discovery.go:444`. `classifyStatus` wraps the `*APIError` as `fmt.Errorf("%s: %w", op, e)`, and
`APIError.Error()` already starts with the same `Operation`, so the errors read
`discovery: discovery: HTTP 401 unknown error code "UNAUTHORIZED"` and
`fetching .well-known/tea: fetching .well-known/tea: HTTP 400 INVALID_REQUEST` (seen in the live
check). The coding principles ask to wrap errors with context (`fmt.Errorf("fetching release
%s: %w", id, err)`); the same context twice adds nothing. Drop the extra prefix. Cosmetic.

## Not tested, by decision

- CI `live` job of `ci.yaml` (3.11): not on the PR, as the operator decided for OOL-4 ("for this
  round don't build CI"); notes-1 departure 6 says so and does not claim it. The checks a CI run
  would make were run by hand (checks 1-10). It stays for the follow-up that sets CI up.
- The live suite against the public demo (4.3): not run from this hop (shared deployment); read
  only.
- Downloads, checksums, the flow helper, `Candidates`, `WithAPIKey`: OOL-6.

## Threat model

Rows for child 3b (T1, T2, T4, T5, T6, T7, T13) all have their listed tests (11b, 26, 26b, 26c;
17; 13b; 21b, 23, 24, 27, 27b; 21, 23, 24; 24, 26, 26c, 27; 39b), with the partial assertions of
T-1 and T-2 above. No security case outside the threat model was added.
