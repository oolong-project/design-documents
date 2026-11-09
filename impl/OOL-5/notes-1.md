# OOL-5 implementation notes, round 1: tea-client-go discovery

Task: OOL-5, child 2 of OOL-3 (OOL-3b). Built from `design/OOL-5/architecture-1.md` (round 1, which
answers the coder's questions `questions/OOL-5/round-1.md` Q-1 to Q-4 on OOL-3's round 2), with
OOL-3's `design/OOL-3/architecture-2.md` for the sections round 1 does not repeat, and the threat
model `design/OOL-3/threat_model-2.md` (round 2, unchanged). Coding principles read at `145008cdcc23`.

## Pull request

| PR | Repository | Base | Head commit | Base tip merged |
|---|---|---|---|---|
| https://github.com/oolong-project/tea-client-go/pull/2 | oolong-project/tea-client-go | `2026-11-oolong` | `50265c322aaec2123c664fef9ea3fa277288325d` | `1c85712` (the branch was cut from it; the base has not moved, nothing to merge) |

oolong-project/oolong is not changed (its `2026-11-oolong` head `e1c9e6e` is the integration-test
server). Merge order: one PR. No `.github/` content (CI is deferred by the operator, see below).

## What changed

Commits, in order (each signed, with the session trailers):

1. `8bd03e1` `transport.go` (OOL-4 code, a fix): the bounded body now treats an `io.EOF` that arrives
   after the request context ended as the bound firing (`ErrTimeout`) or the caller's context error,
   not as a complete body. Over TLS, when a time bound cancels a body read, the transport sometimes
   reports the closed connection as a clean EOF, so a cut-off body passed as whole and was then
   decoded. Found as a flaky stalled-body discovery test (`TestDiscoverAttemptTimeout`, about 5 in 40
   runs before, 0 in 60 after). It applies to every bounded call, so it also hardens OOL-4's 11d and
   OOL-6's downloads.
2. `ceea128` `discovery.go` (new), `client.go` (doc comment of `WithDiscoveryAttempts` only):
   `Client.Discover`, `Client.DiscoverPURL`, `Client.SelectServer`, `DiscoveryResult`,
   `NotResolvedError`, `DiscoveryError`, `Attempt`, steps 1-6 of 3.6:
   - well-known URL from the lowercased domain or `WithWellKnownLocation`; the document validated by
     hand against `tea-well-known.schema.json` (exact key matching, unknown properties ignored), each
     `url` against the pattern and D4;
   - candidates ordered by greatest supported version (build metadata ignored), then priority
     (absent 1), then document order; the advertised version used verbatim in the path;
   - `/discovery?tei=` (or `purl=`) with `url.QueryEscape` once; GETs carry `kindWellKnown` /
     `kindDiscovery`, so `WellKnownBytes`, `DiscoveryBytes` and `AttemptTimeout` apply through the
     OOL-4 doer;
   - classification (`classifyStatus`): only `200` with a valid body succeeds; a TEA-shaped `404` at
     `/discovery` stops with `*NotResolvedError`; `400`, `401`, `403` stop with `*APIError` at either
     stage; everything else, any `404` at the well-known stage included, is a failed attempt;
   - a `/discovery` answer is validated whole: 1 to `DiscoveryResults` items (more is
     `ErrTooManyResults`, never truncated), uuids that `google/uuid` decodes, non-empty `servers`,
     each `rootUrl` matching the pattern and passing D4, versions SemVer, priority in [0, 1];
   - every discovery GET counts against `WithDiscoveryAttempts` (the successful well-known fetch
     included); the well-known fetch is retried within the bound; candidates are retried round-robin
     until the bound; backoff 250 ms doubling, capped at 4 s (`backoffDelay`), interrupted by the
     context; the caller's context ending returns its error and is never a failed attempt;
   - `TraceAttemptFailed` (stage, URL without query, reason) and `TraceBackoff` (wait) events;
     attempt reasons are fixed texts naming the rule or status, sanitised and at most 200 bytes, never
     body content or the TEI/PURL.
3. `7269db8` `integration_test.go` cases 1-3 of 4.2, `live_test.go` case 1 of 4.3.
4. `16ee9fa` README (discovery section, live test command) and `ExampleClient_Discover` (the README
   snippet is copied from it; it compiles and does not run).
5. `50265c3` the test sub-cases design round 1 added for Q-4 (cases 23 and 24); integration case 3's
   comment cites the decided rule.

## How Q-1 to Q-4 ended (design round 1)

All four confirmed what PR #2 already did; no code changed for them.

- **Q-1**: a TEA error is recognised by shape only; oolong's Nest `404` at `/discovery` gives
  `*NotResolvedError` (`ErrorCodeUnknown`, `RawCode` `Not Found`), no failover. Integration case 3
  asserts exactly that.
- **Q-2**: every discovery GET is one attempt; `DiscoveryError.Attempts` lists only failed ones.
  With attempts 1, `Discover` ends after the well-known fetch (the option's doc says so).
- **Q-3**: candidates are retried round-robin until the bound.
- **Q-4**: other statuses fail over; every well-known `404` fails over; one refused root or
  out-of-range priority refuses the whole answer; `SelectServer` checks D4 on its first choice only,
  no fallback.

## Departures and additions

1. **The `transport.go` fix to OOL-4 code** (commit 1 above).
2. **`NotResolvedError.Is(ErrObjectUnknown)`**: true when `Code` is `OBJECT_UNKNOWN`, so callers can
   use the same sentinel as for typed calls. Not named in the design; additive.
3. **`DiscoverPURL` input checks**: the PURL must start with `pkg:`, be at most 2048 bytes (the TEI
   bound) of valid UTF-8 with no control characters (`ErrInvalidArgument`, input not echoed); the base
   is checked against D4 before any request. The design names no PURL rule.
4. **Strictness the schema implies**: well-known and discovery JSON keys are matched exactly (not
   `encoding/json`'s case-insensitive struct matching); `schemaVersion` must be a JSON number equal
   to 1 (`1.0` is an integer in draft-07; the string `"1"` is refused); `null` is refused wherever a
   value is required.
5. **`SelectServer` on a dot-segment or non-ASCII root** returns `*URLPolicyError` (`ErrInsecureURL`)
   before ordering, rather than silently dropping the server; other malformed caller input is
   `ErrInvalidArgument`, as 3.6 says.
6. **CI `live` job deferred.** By operator decision this round adds no `.github/` content (the coder
   credential cannot push workflow files). The job 3.11 describes (only on `workflow_dispatch`,
   `TEA_LIVE=1 go test -tags live -run Live ./...`, `contents: read`, pinned actions,
   `persist-credentials: false`) is written into the uncommitted workflow kept outside the repository
   for the follow-up that sets CI up, together with the `workflow_dispatch` trigger it needs. It is
   not claimed as done here.

## Tests

Unit tests (default build, offline, `httptest`, made-up data):

| Design case | Test(s) (`discovery_test.go` unless named) |
|---|---|
| 9b (= 21b, 27b) | `TestDiscoverLocalReARMShape`, `TestSelectServerUnderAllowance` |
| 11b | `TestDiscoverByteLimits` (Content-Length and chunked, one byte over and exactly at the limit) |
| 13b | `TestDiscoveryCredentials` (with positive controls: the bearer reaches `/discovery` and an API call at the root) |
| 17 | `FuzzDecodeWellKnown`, `FuzzDecodeDiscovery` (`discovery_fuzz_test.go`; they also assert that what is accepted is usable) |
| 21 | `TestDiscoverRequestForm` |
| 21b | `TestDiscoverLocalReARMShape` |
| 22 | `TestDiscoverSelection`, `TestSortCandidates` |
| 23 | `TestWellKnownValidation` (27 refusals, the TEA-shaped 404 sub-case, unknown properties accepted) |
| 24 | `TestDiscoverFailover` (24 sub-cases incl. 429, 204, 405, refused-whole, redirect to http, TLS failure), `TestDiscoverUppercaseUUID` |
| 25 | `TestDiscoverNoFailover` |
| 26 | `TestDiscoverAttemptBound`, `TestBackoffDelay` |
| 26b | `TestDiscoverAttemptTimeout` |
| 26c | `TestDiscoverResultsCap` |
| 27 | `TestDiscoverResultsAndSelectServer` |
| 39b | `TestDiscoveryTrace` |
| 3.6 (no body in errors) | `TestDiscoveryErrorsCarryNoBody` |

Integration (`integration_test.go`, `//go:build integration`): `TestIntegrationDiscover` (4.2 case 1),
`TestIntegrationDiscoverUnknownAndPURL` (case 2), `TestIntegrationDiscoverWrongPath` (case 3).
Live (`live_test.go`, `//go:build live`, runs only with `TEA_LIVE=1`): `TestLiveDiscover` (4.3 case 1).

Commands run on the head `50265c3`, all passing (Go 1.26.0, Node 24.14.1):

```sh
gofmt -l .                                   # empty
go vet ./... && go vet -tags integration ./... && go vet -tags live ./...
go build ./...
go mod tidy -diff                            # empty
go generate ./... && git diff --exit-code    # teaapi unchanged
go test -count=1 ./...                       # root 45 tests (incl. 2 fuzz seed runs), tei 10
go test -race -count=3 ./...                 # also -count=5 on 16ee9fa
go test -run '^$' -fuzz='^FuzzDecodeWellKnown$' -fuzztime=30s .   # 30 s on 16ee9fa (1.58M execs), 20 s on 50265c3
go test -run '^$' -fuzz='^FuzzDecodeDiscovery$' -fuzztime=30s .   # 30 s on 16ee9fa (1.82M execs), 20 s on 50265c3
LC_ALL=C /usr/bin/grep -nP "[^\x00-\x7F]" <touched files>          # nothing
# oolong 2026-11-oolong @ e1c9e6e, npm ci && npm run build in oolong/oolong-server
OOLONG_SERVER_DIR=<oolong>/oolong/oolong-server TEA_INTEGRATION_REQUIRED=1 go test -count=1 -tags integration -run Integration ./...   # 6 of 6
TEA_LIVE=1 go test -count=1 -tags live -run Live ./...   # 1 of 1, 2026-11-09, against https://demo.rearmhq.com
```

Only the new tests: `go test -count=1 -run 'Discover|Discovery|WellKnown|SortCandidates|Backoff|SelectServer' .`

## Mutation checks (one-off, in a scratch copy; not committed)

Every mutation below made the named tests fail (26 mutants, all killed).

| Mutation | Failing tests |
|---|---|
| TEA 404 at `/discovery` fails over | `TestDiscoverNoFailover` |
| 400/401/403 fail over | `TestDiscoverNoFailover` |
| results cap removed | `TestDiscoverResultsCap` |
| D4 not applied to learned bases | `TestWellKnownValidation`, `TestDiscoverLocalReARMShape`, `TestDiscoverFailover` |
| `SelectServer` skips D4 | `TestDiscoverResultsAndSelectServer`, `TestSelectServerUnderAllowance` |
| `SelectServer` does not refuse dot segments | `TestDiscoverResultsAndSelectServer` |
| successful well-known GET not counted | `TestDiscoverAttemptBound` |
| candidates not retried round-robin | `TestDiscoverAttemptBound` |
| well-known not retried | `TestDiscoverAttemptBound`, `TestDiscoverAttemptTimeout` |
| discovery GET sent as `kindJSON` | `TestDiscoverByteLimits`, `TestDiscoverAttemptTimeout`, `TestDiscoveryTrace` |
| well-known GET sent as `kindJSON` | `TestDiscoverByteLimits`, `TestDiscoveryTrace` |
| no ordering / priority ignored | `TestDiscoverSelection`, `TestSortCandidates` (+ `TestDiscoverResultsAndSelectServer`) |
| no backoff wait | `TestDiscoverAttemptBound` |
| caller context ending counted as a failed attempt | `TestDiscoverAttemptTimeout` (survived at first; killed after the caller-context test also ran with attempts 1 and checked for no attempt-failed event) |
| TEI not query-encoded | `TestDiscoverRequestForm` |
| no attempt-failed trace event | `TestDiscoveryTrace`, `TestDiscoverFailover` |
| uuid accepted unchecked | `TestDiscoverFailover` |
| bearer sent to every request | `TestDiscoveryCredentials` |
| reason built with body content | `TestDiscoveryErrorsCarryNoBody` |
| unknown TEA code not truncated | `TestDiscoverNoFailover` |
| PURL checks removed | `TestDiscoverResultsAndSelectServer` |
| well-known TEA-shaped 404 stops (Q-4) | `TestWellKnownValidation/TEA-shaped_404` |
| 429 stops (Q-4) | `TestDiscoverFailover/429` |
| any 2xx is success (Q-4) | `TestDiscoverFailover/204` |
| refused root skipped instead of refusing the answer whole (Q-4) | `TestDiscoverFailover/https_root_then_an_http_root_...` |

The `transport.go` fix is checked statistically: reverted, `go test -count=40 -run TestDiscoverAttemptTimeout .`
failed about 5 runs in 40; with it, 0 in 60.

## For the tester

- Integration needs the oolong server built locally (`npm ci && npm run build` in
  `oolong/oolong-server` at `2026-11-oolong`); the harness copies the fixtures into a temp dir.
  Case 3 pins oolong's current unmatched-route body; if oolong changes it, case 3 fails naming that.
- Timing tests use 200 ms bounds and assert completion within 1 s; they passed under `-race` (also
  `-count=5`). The backoff test expects backoffs `[40ms 80ms]` and at least 120 ms elapsed.
- TLS tests rely on all `httptest` TLS servers sharing one certificate (the client of one trusts
  all); the TLS-failure case starts a server with a certificate generated in the test.
- `TestDiscoveryCredentials` addresses servers as `localhost`, `sub.localhost` and `127.0.0.1`
  through OOL-4's dialer that maps `*.localhost` to 127.0.0.1; no DNS.
- The live test reaches the network only with `-tags live` and `TEA_LIVE=1`.
- Not in this child: downloads, checksums, the flow helper, `Candidates`, `WithAPIKey` (OOL-6).
