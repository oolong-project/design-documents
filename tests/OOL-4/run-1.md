# OOL-4 test run 1: tea-client-go foundation (pinned spec, teaapi, tei, client, limits, errors, typed calls, harness)

Task: OOL-4 (child 1 of OOL-3). Verdict: **REJECTED**, 4 items OPEN (T-1 to T-4), all about the
code (`DETAILED_DESIGN`).

Inputs: `impl/OOL-4/notes-1.md` and `impl/OOL-4/notes-2.md` (DETAILED_DESIGN rounds 1 and 2), the
carried design `design/OOL-3/architecture-2.md` and `design/OOL-3/threat_model-2.md` (round 2 of
each, the parts the OOL-4 description names); coding principles read at `145008cdcc23`.

## What was tested

| PR | Head tested | Base tip | Merge |
|---|---|---|---|
| https://github.com/oolong-project/tea-client-go/pull/1 | `4d16b0c7fa04196bd3cca07bd01c0b622e96b420` | `2026-11-oolong` @ `82d03dac23f9` | scratch merge `--no-ff` of the head into the base tip; the merged tree is identical to the head (the base is an ancestor), so the tree tested is `4d16b0c` |

oolong-project/oolong is not changed by OOL-4. The integration and live checks use oolong
`2026-11-oolong` @ `e1c9e6e3068e`.

Tools: Go 1.26.10 (`golang:1.26` container on the local k3s; `go.mod` says `go 1.26.0`; Go is not
installed on the host), Node 24.14.1. Docker is not usable on the host.

## Results

| # | Check | Result |
|---|---|---|
| 1 | `gofmt -l .` | empty |
| 2 | `go build ./...` | passed |
| 3 | `go vet ./...` and `go vet -tags integration ./...` | clean |
| 4 | `go test -count=1 ./...` | passed (root package 23 tests, `tei` 5 tests, 2 fuzz seeds, 2 examples) |
| 5 | `go test -count=3 -race ./...` | passed |
| 6 | `go mod tidy -diff` | empty |
| 7 | `go generate ./...` then `git diff --exit-code` | no diff: `teaapi/tea.gen.go` is the generator's output |
| 8 | Spec pin: upstream CycloneDX/transparency-exchange-api fetched at `962a0228af1bfcc1232ac3f79239317ff33fdb04`; `spec/openapi.yaml`, `spec/tea-well-known.schema.json`, `spec/LICENSE` compared with `cmp` and `git hash-object` | byte-identical; blob ids `c96baa3e...`, `b0804f1a...`, `0808751c...` as the design says; no upstream `NOTICE`; `spec/openapi.yaml` also byte-identical to oolong's `tea-spec/openapi.yaml` |
| 9 | Generator pinned in `tools.mod` (oapi-codegen v2.8.2, `tool` directive), none in `go.mod`; `tea.gen.go` header names v2.8.2 | passed |
| 10 | `go test -fuzz='^FuzzParse$' -fuzztime=30s ./tei` | passed |
| 11 | `go test -fuzz='^FuzzParseLegacyURN$' -fuzztime=30s ./tei` | passed |
| 12 | Integration (4.2) through the harness: `OOLONG_SERVER_DIR=<oolong e1c9e6e>/oolong/oolong-server TEA_INTEGRATION_REQUIRED=1 go test -tags integration -run Integration ./...` (server built with `npm ci && npm run build`) | 3 of 3 passed (cases 5, 6, 9) |
| 13 | Integration with `OOLONG_BASE_URL` at the in-cluster CI-built server (Live checks) | 3 of 3 passed |
| 14 | Harness without `OOLONG_*`: skips with how to set them; with `TEA_INTEGRATION_REQUIRED=1`: fails | passed |
| 15 | Plain-ASCII sweep (`LC_ALL=C /usr/bin/grep -nP "[^\x00-\x7F]"`) over the 36 changed files | non-ASCII only in `spec/openapi.yaml` (upstream byte copy) and `teaapi/tea.gen.go` (generated from it), the two exclusions 3.11 names; every hand-written file clean |
| 16 | The PR has no `.github/`; README and code refer to no workflow | passed |
| 17 | Commits: six, each signed, each with the ReARM trailers | passed |
| 18 | Design cases in scope (4.1: 1-8, 9a, 10, 11a, 11c, 11d, 12, 13a, 14a, 15, 16, 18-20, 39a, 40; 4.2 harness and cases 5, 6, 9): a test exists and asserts the named behaviour | 26 of 27; 11a is T-1 |
| 19 | Threat model section 3, rows for child 3a (T1, T2, T4, T5, T6, T8, T11, T12, T13) | every listed case has a test; T12's CI drift check is in the deferred workflow (below), and was run by hand (check 7) |

Each case of check 18 was read against its test by a reviewing sub-agent and the doubtful ones
were checked by mutation in a throwaway copy (T-1's mutation was re-run by me). The coder's
departures 1 to 7 (notes-1) are acceptable under the design; departure 4 lacks a test (T-4).

## Live checks

Build: the CI-built image `registry.rearmhq.com/library/oolong-server:2026-11-oolong` (resolved to
`sha256:696876272b2e9609599bee8af13785311124731d615bd9039bb9810af977ffa2` at the time of the run),
deployed in namespace `oolong-tester-ool-4` with the TEA 1.0 fixtures (`oolong/fixtures/tea-1.0.0`
from oolong `e1c9e6e`, cloned by an init container) and the environment of
`oolong/docker-compose.tea-interop.yml` (`API_VERSION=v1.0.0`, `SERVER_HOST` the Service URL,
`PAGE_TOKEN_SECRET` random per deployment in a Secret). The Go client (tree `4d16b0c`) ran in a
Go pod in the same namespace. Discovery is not in this child, so `.well-known` was only fetched
by plain HTTP. Nothing ran against a shared deployment.

Plain HTTP (curl): `/.well-known/tea` 200 with the one endpoint; `/v1.0.0/products?pageSize=2`
200 with `hasNext` and a token; `/v1.0.0/product/<alpha>` 200; unknown product 404
`OBJECT_UNKNOWN`; product release CLE 404 `NOT_IMPLEMENTED`; `pageToken=bogus` 400
`INVALID_PAGE_TOKEN`; `pageSize=101` 400 `INVALID_REQUEST`; latest collection 200 (version 2);
`POST /v1.0.0/token` 404 `NOT_IMPLEMENTED`.

Go client (a throwaway `main` over the module, not committed), 35 of 35 passed:

| Area | Checks |
|---|---|
| URL policy | http base without `WithInsecureHTTPBase` refused with `ErrInsecureURL` before any request; an http base not under the allowance refused |
| Typed calls | `AllProducts` page size 2: Alpha, Beta, Delta, Epsilon, Gamma; `AllProductReleasesOf(alpha)` by createdDate asc: 0.9.0, 1.0.0, 1.1.0; `QueryProductReleases` by TEI: exactly release `22222222-...-222222222222`; `GetProduct`, `GetProductRelease` (1.0.0), `GetLatestProductReleaseCollection` (2), `GetProductReleaseCollection` v1, `ListProductReleaseCollections` (2), synthetic latest collection of `...223` (1), `GetComponentRelease`, `GetComponentReleaseCollection` v2, `QueryComponents`, `GetComponent`, `ListComponentReleases`, `GetLatestArtifact`, `GetArtifact` v1 |
| Error mapping | unknown uuid: `*APIError`, `ErrObjectUnknown`; CLE: `ErrNotImplemented`; bad page token: `ErrInvalidPageToken`; page size 101: `ErrInvalidArgument` locally; `RequestToken`: `ErrNotImplemented`, the secret not in the error; `Generated()` returns the raw 404 without error |
| Limits | `JSONBytes` 64: `GetProduct`, `Generated().GetProductWithResponse` and the iterator give `ErrResponseTooLarge`; `JSONBytes` equal to the body (211 bytes) succeeds, one byte less fails; `Pages` 2 gives `ErrPageLimit`; negative `JSONBytes` refused by `New` |
| Credentials, trace, UA | anonymous: no `Authorization`; `WithBearerToken` for the root: `Bearer` sent and ignored by oolong; 50 trace events, none with `?` or the secret; every request `User-Agent: tea-client-go/devel` |

## T-1: test 11a does not assert that a non-2xx body over ErrorBodyBytes is cut

Design 4.1 case 11a: "a non-2xx body over `ErrorBodyBytes` is cut and still classified". The test
(`transport_test.go:282-300`) sends `{"error":"OBJECT_UNKNOWN"}` followed by 256 KiB of spaces
with `ErrorBodyBytes` 4096 and asserts only the classification, which is the same whether or not
the body is cut. Mutation (in a copy): `transport.go:168` `body.limit = d.c.limits.ErrorBodyBytes`
changed to `body.limit = 1 << 40` (error bodies unbounded):

```
$ go test -count=1 -run "TestJSONByteLimit|TestErrorMapping" .
ok  	github.com/oolong-project/tea-client-go	0.018s
```

The bound itself is in the code; the test that should hold it is hollow (priority 1 under "Tests
exist"). Fix: a body whose decodable part ends beyond `ErrorBodyBytes` (expect `TEA: false`), or
count the bytes the server managed to write, or assert the read stops early.

## T-2: decode errors repeat response-body content

`api.go:104` builds `"body does not decode: " + err.Error()` from `encoding/json`, whose errors
quote server values (for a `time.Time` field, `time.ParseError` repeats the string verbatim). A
product release whose `releaseDate` is `"SECRET-BODY-CONTENT-..."` gives an error that contains
that string (sanitised and capped at 200 bytes, but still body content). Design 3.6: "A response
body never goes into an error beyond the sanitised `message`"; README.md:74-75 promises the same.
Fix: a fixed reason (e.g. which type failed to decode), plus a test that a sentinel value in a bad
field is absent from `err.Error()`.

## T-3: tei.SyntaxError.Input holds the whole input, against its doc comment and 3.3

`tei/tei.go:93-97` documents "Input is truncated to 64 bytes, so an error never repeats more of
an untrusted string than that", but `syntaxErr` (`tei/tei.go:120`) stores the input whole (up to
the full 2049+ bytes, or a whole legacy URN). `Error()` truncates, so the message assertion of
case 4 holds, but the exported field contradicts its doc and design 3.3 ("never echoes more than 64
bytes of input"). The test check `len(se.Input) > len(c.in)` (`tei/tei_test.go:97`) can never
fail. Fix: truncate when the error is built and assert `len(se.Input) <= 64`.

## T-4: ParseAPIBase's version refusal (departure 4) has no test

Departure 4 (notes-1) makes `ParseAPIBase` refuse an unsupported or non-SemVer version with
`ErrInvalidArgument`, comparing with build metadata ignored (`base.go:215-220`). The departure is
reasonable, but no test calls `ParseAPIBase` with a version other than `"1.0.0"`. Add cases for
`2.0.0`, `1.0`, `v1.0.0` (refused) and `1.0.0+b` (accepted).

## Not tested, by decision

- **CI workflow (3.11 `ci.yaml`).** The operator lifted the coder's hold with "for this round
  don't build CI, we will setup CI after this round"; notes-2 records it. The PR has no workflow
  and no CI reports on it, so no CI result was seen. The generator drift and ASCII checks that job
  would run were run by hand (checks 7 and 15). Not a review item. For the follow-up that adds
  CI: the design's checkout path `oolong/` inside the module makes `./...` pick up a `.go` file
  under oolong-server's `node_modules` (coder's departure 2, `_oolong/`); that part of 3.11 needs
  the design changed when CI is set up.
- Discovery, downloads, checksums, flow, `WithAPIKey`, the live suite (4.3) and 4.2 cases 1-4, 7,
  8: not in this child (OOL-5, OOL-6).
