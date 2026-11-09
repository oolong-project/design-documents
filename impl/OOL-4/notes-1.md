# OOL-4 implementation notes, round 1: tea-client-go foundation

Task: OOL-4, child 1 of OOL-3 (OOL-3a). Built from the carried design `design/OOL-3/architecture-2.md`
(section 5 child 1) and `design/OOL-3/threat_model-2.md`, both round 2, read after assignment.
Coding principles read at `145008cdcc23`.

## Pull request

| PR | Repository | Base | Head commit | Base tip merged |
|---|---|---|---|---|
| https://github.com/oolong-project/tea-client-go/pull/1 | oolong-project/tea-client-go | `2026-11-oolong` | `4d16b0c7fa04196bd3cca07bd01c0b622e96b420` | `82d03dac23f9` (LICENSE only; the branch was cut from it and the base has not moved) |

oolong-project/oolong is not changed (its `2026-11-oolong` head is only the integration-test
server). Merge order: one PR.

**Open: the CI workflow is not on the PR.** `.github/workflows/ci.yaml` (design 3.11) is written
and committed locally (commit `9c6a47a144ad317c97d7f0857d9186edc7391e04` on a local branch
`ool-4-ci-pending`, a child of the PR head), but the push was refused: the git credential on this
host has no GitHub `workflow` scope, and GitHub refuses any push that creates or changes a file
under `.github/workflows/` without it. The task is parked for the operator with that question.
Once a credential with the scope is available, that commit is pushed to the PR branch as is. Until
then the PR runs no CI, and this round cannot be signed off PASSED.

## What changed

Commits, in order (each signed, with the session trailers):

1. `e19b5b3` pin the spec: `spec/openapi.yaml`, `spec/tea-well-known.schema.json` and
   `spec/LICENSE` are byte copies from CycloneDX/transparency-exchange-api at `962a0228...`
   (blob ids checked: `c96baa3e...`, `b0804f1a...`, `0808751c...`); `spec/SPEC_PIN.md` names paths,
   commit, blob ids, retrieval date and the licence of `spec/`.
2. `14b9ddc` module setup: `go.mod` (go 1.26.0, no tool line), `tools.mod`/`tools.sum` with the
   `tool` directive for oapi-codegen v2.8.2, `teaapi/generate.go`, `teaapi/oapi-codegen.yaml`
   (models + client), `teaapi/doc.go` (source, pin, Apache-2.0 note, 4(b) transformation).
3. `4c5fff5` `teaapi/tea.gen.go` only: the generator's output, not edited.
4. `84ea19d` package `tei` (3.3): `Parse` with the six rules in order, `*SyntaxError` naming the
   rule and echoing at most 64 input bytes, `Type.Known`, `PURL()`, `ParseLegacyURN` per the
   table, `ParseCompat`.
5. `63ceecd` package `tea`:
   - `client.go`: `New`, every option of 3.4 except `WithAPIKey` (departure 1), `Limits` with all
     fields and defaults of the table, `TraceEvent`/`TraceKind`/`Stage`, default transport (clone
     of `http.DefaultTransport`, TLS handshake 10 s, response header 30 s, no `Client.Timeout`),
     user agent from build info, `Client.API`.
   - `base.go`: URL policy (D4): `parseBase`, `under` (with the comment naming the spec section
     and the departure, as section 5 asks), dot-segment refusal (`.`, `..`, `%2e` forms), userinfo,
     non-ASCII host; `APIBase`, `ParseAPIBase`, `SupportedVersions`, spec patterns for `rootUrl`
     and SemVer, x/mod for build-metadata-insensitive matching.
   - `transport.go`: the call kinds on the request context; the bounded doer (D3): URL policy on
     the request, per-kind byte limit after transport decoding with `Content-Length` refused up
     front, non-2xx bodies cut at `ErrorBodyBytes`, per-kind time bound through the context
     (`WithTimeoutCause(..., ErrTimeout)` for `AttemptTimeout`/`CallTimeout`, a cancel-cause plus
     `time.AfterFunc` reset on every read for `IdleTimeout`), error classification (caller's
     context wins, else `ErrTimeout`, URLs in `*url.Error` redacted); `Generated()` path
     classification; `CheckRedirect` (no redirect for `kindToken`, limit, URL policy); the auth
     transport (D5: removes `Authorization` off every configured root and sets the root's bearer
     under it, on every hop; leaves `kindToken` as built; sets User-Agent; trace events).
   - `errors.go`: `ErrorCode` closed set, `APIError` with `Is`, `UnexpectedResponseError`,
     `URLPolicyError`, all sentinels of 3.6 (incl. `ErrTimeout`, `ErrTooManyResults`,
     `ErrTooManyComponents`, `ErrPageLimit`), sanitising (control characters dropped, 64/200
     bytes), `WWW-Authenticate` challenge parsing.
   - `api.go`: the 22 typed GET wrappers of 3.5 (all resource operations except downloads) and
     `Generated()` with the doc comment of D3/section 5, plus the F-10 note (a token request
     through `Generated()` cannot carry Basic: the auth transport replaces or removes it).
   - `pagination.go`: query/page types with closed sort-field types and local checks; the
     `hasNext` without token check; eight iterators bounded by `Limits.Pages`.
   - `auth.go`: the one token request (3.7) and `API.RequestToken`; `*TokenError`.
   - `internal/oolongtest`: the 4.2 harness exactly as specified (`OOLONG_BASE_URL`, else
     `OOLONG_SERVER_DIR` with fixtures copied into `t.TempDir()`, free port, clean env with a
     per-run random `PAGE_TOKEN_SECRET`, poll `/.well-known/tea` for 200 within 30 s, output shown
     only on failure, kill and wait in cleanup; skip, or fail with `TEA_INTEGRATION_REQUIRED=1`).
6. `4d16b0c` README (minimal, with Versioning and an empty Changes list, and Licence; also the
   URL and credential policy, the caller-transport note, SSRF note, regeneration and tests).
7. (local, not pushed) `9c6a47a` `.github/workflows/ci.yaml`: jobs `test`, `integration`,
   `record-release` as 3.11 says (see departure 2 for the checkout path).

`go.mod` requirements: `github.com/oapi-codegen/runtime` v1.8.0, `github.com/google/uuid` v1.6.0,
`golang.org/x/mod` v0.42.0 (newest at start); indirect `github.com/apapsch/go-jsonmerge/v2`
v2.0.0. Nothing is copied from rearm-client-go.

## Departures from the design

1. **`WithAPIKey` is not exported yet.** 3.4's option list names it, but its whole behaviour is
   3.7's recovery, which the split gives to OOL-3c (OOL-6). Exporting an option that silently does
   nothing would be a trap for a caller, so OOL-6 adds the option with its behaviour. The
   credential list and the auth transport are shaped for it (`credential` holds the root).
2. **CI checks oolong out into `_oolong/`, not `oolong/`.** `oolong/oolong-server/node_modules`
   contains a `.go` file (`flatted/golang/pkg/flatted/flatted.go`), so a checkout inside the module
   under `oolong/` would make `go test ./...` and `go vet ./...` treat it as a package of this
   module. The go tool ignores directories starting with `_`. The env var is then
   `OOLONG_SERVER_DIR=$GITHUB_WORKSPACE/_oolong/oolong/oolong-server`; everything else as 3.11.
3. **`ComponentQuery` and the type names.** 3.5 names `ProductQuery`, `ReleaseQuery`,
   `ReleasePage`, `CollectionPage`. Components need their own sort-field type, so
   `ComponentQuery` exists too. Sort fields are `ProductSortField`, `ComponentSortField`,
   `ReleaseSortField` (product and component releases share one set in the spec) and
   `CollectionSortField`; paging fields sit in an embedded `Page`. Per-parent iterators are
   `AllProductReleasesOf`, `AllComponentReleasesOf`, `AllProductReleaseCollectionsOf`,
   `AllComponentReleaseCollectionsOf`.
4. **`ParseAPIBase` refuses an unsupported version** (`ErrInvalidArgument`), comparing with
   build metadata ignored (D10). The design says the client supports exactly 1.0.0 but does not
   say where a base for another version is refused; this is the earliest place.
5. **`golang.org/x/crypto` is not in `go.mod` yet**: only OOL-6's BLAKE2b code imports it, and
   `go mod tidy` drops an unused requirement.
6. **Test 2's wrapper set** asserts every method this child builds; the download helpers of 3.8
   belong to OOL-6.
7. **Additions the design implies but does not spell out:** a `/token` `404` wraps
   `ErrNotImplemented` whatever its body (the spec: a server without auth answers `404`);
   `*TokenError` (named in 3.7) is built here because the token request is; the token response is
   bounded by `ErrorBodyBytes` (3.7) and the call by `CallTimeout`; the URL policy is also checked
   in the auth transport on every hop, as a backstop behind `CheckRedirect`; no local minimum on
   collection and artifact version path parameters (the spec defines none).

## Tests

Unit tests (default build, offline, `httptest`; data in temp servers only):

| Design case | Test(s) |
|---|---|
| 1 | `TestSpecPin`, `TestGoModHasNoToolLine` (`spec_test.go`) |
| 2 | compile-time assertions in `spec_test.go` (29 generated methods, 28 operations; tea wrapper set) |
| 3, 4, 5, 6 | `TestParseAccepted`, `TestParseRefused`, `TestParseLegacyForm`, `TestParseLegacyURN`, `TestPURLWrongType` (`tei/tei_test.go`); examples in `tei/example_test.go` |
| 7 | `FuzzParse`, `FuzzParseLegacyURN` |
| 8 | `TestNewRefusesOptions` (`client_test.go`) |
| 9a | `TestAllowanceCoversPathsUnderBase`, `TestUnderHelper` (`transport_test.go`) |
| 10 | `TestRedirects` |
| 11a | `TestJSONByteLimit` (content-length, chunked and gzip at exactly the limit and one over; gzip bomb; asserts gzip was negotiated; large 404/500 cut and classified) |
| 11c | `TestGeneratedIsBounded`, `TestClassify` |
| 11d | `TestCallTimeout`, `TestIdleTimeoutGenerated` |
| 12 | `TestAnonymousSendsNoAuthorization` (`auth_test.go`) |
| 13a | `TestBearerBoundToRoot` |
| 14a | `TestRequestToken` |
| 15 | `TestErrorMapping`, `TestParseChallenge` (`errors_test.go`) |
| 16, 18 | `TestTypedCallUnexpectedJSON`, `TestTypedCallRequests` (`api_test.go`; one table over all 23 typed calls) |
| 19, 20 | `TestLocalRefusals`, `TestIterator` (`pagination_test.go`) |
| 39a | `TestTraceEvents` (`trace_test.go`) |
| 40 | `TestUserAgent`, `TestModuleVersion` (`client_test.go`) |

Integration (`integration_test.go`, `//go:build integration`): `TestIntegrationPagination` (4.2
case 5), `TestIntegrationCollections` (case 6), `TestIntegrationBearerIgnored` (case 9; a
transport below the auth transport confirms the bearer was actually sent).

Commands run on the PR head, all passing (Go 1.26.0, Node 24.14.1):

```sh
gofmt -l .                                   # empty
go vet ./... && go vet -tags integration ./...
go build ./...
go test -count=1 ./...
go test -count=1 -race ./...                 # also run with -count=2
go mod tidy -diff                            # empty
go generate ./...                            # tea.gen.go unchanged (cmp)
go test -fuzz='FuzzParse$' -fuzztime=30s ./tei
go test -fuzz=FuzzParseLegacyURN -fuzztime=30s ./tei
LC_ALL=C /usr/bin/grep -nP "[^\x00-\x7F]" <every .go .yaml .md .mod file except spec/ and teaapi/tea.gen.go>   # nothing
# oolong 2026-11-oolong @ e1c9e6e, in a separate worktree: npm ci && npm run build in oolong/oolong-server
OOLONG_SERVER_DIR=<that>/oolong/oolong-server TEA_INTEGRATION_REQUIRED=1 go test -count=1 -tags integration -run Integration ./...
```

Without `OOLONG_*` the integration tests skip; with `TEA_INTEGRATION_REQUIRED=1` they fail
(checked). The live suite (4.3) is not part of this child (its cases are OOL-5's and OOL-6's).

**Mutation checks** (one-off, not committed; each restored after): every one made the named
tests fail.

| Mutation | Failing tests |
|---|---|
| auth transport no longer removes `Authorization` | 13a: redirect targets see the editor's Basic header; editor header off-root reaches the server |
| `kindToken` follows redirects | 14a: redirect target receives requests |
| `kindToken` not exempt (bearer replaces Basic) | 14a |
| no `CallTimeout`/`AttemptTimeout` bound | 11d: errors become the caller's deadline, not `ErrTimeout` |
| no idle timer | 11d idle: stalled download ends only at the caller's 5 s deadline |
| idle timer not reset on bytes | 11d idle: the trickling download is cut at 200 ms |
| no byte limit on 2xx | 11a, 11c |
| no path classification in the doer | 11c, 11d idle |
| http always allowed | 9a, 10 |
| dot segments not refused | 8, 9a |

## For the tester

- The integration suite needs Node 24 and an oolong checkout at `2026-11-oolong` with
  `npm ci && npm run build` run in `oolong/oolong-server`; the harness copies the fixtures into a
  temp dir and starts the server itself. `OOLONG_BASE_URL` works instead with a running server
  whose `SERVER_HOST` is the URL used.
- Timing tests (11d) use 100 to 300 ms bounds and assert completion within 1 s; they passed under
  `-race`. The trickle download takes about 1.5 s by design.
- `TestRedirects` and `TestBearerBoundToRoot` address test servers as `localhost`,
  `sub.localhost` and `127.0.0.1` through a dialer that maps `*.localhost` to 127.0.0.1; no DNS.
- Not in this child: discovery (`Discover`, `DiscoverPURL`, `SelectServer`; OOL-5),
  downloads, checksums, flow and `WithAPIKey` (OOL-6). `Limits.DiscoveryResults`,
  `WithWellKnownLocation` and `WithDiscoveryAttempts` are parsed and validated here and used by
  OOL-5.
