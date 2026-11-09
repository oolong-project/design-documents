# OOL-4 test run 2: re-test of the fixes for T-1 to T-4

Task: OOL-4 (child 1 of OOL-3). Verdict: **PASSED**. T-1 to T-4 from run 1 are all RESOLVED; no
new items; nothing OPEN.

Inputs: `impl/OOL-4/notes-3.md` (DETAILED_DESIGN round 3, the coder's answer to run 1), with
`notes-1.md` and `notes-2.md` still describing the build; the carried design
`design/OOL-3/architecture-2.md` and `design/OOL-3/threat_model-2.md` (round 2 of each); run 1
(`tests/OOL-4/run-1.md`); coding principles read at `145008cdcc23`.

## What was tested

| PR | Head tested | Base tip | Merge |
|---|---|---|---|
| https://github.com/oolong-project/tea-client-go/pull/1 | `f39e89cfc4a955c5725251b7dd9e41f830fca1c8` | `2026-11-oolong` @ `82d03dac23f9` (not moved since run 1) | scratch merge `--no-ff` of the head into the base tip; merged tree identical to the head's tree |

Run 1's head `4d16b0c` is an ancestor of `f39e89c`: four commits were added on top (`55f494e`
T-1, `bc3cb3c` T-2, `2dfce8d` T-3, `f39e89c` T-4), nothing rewritten. Each carries a signature
and both ReARM trailers. The diff since `4d16b0c` touches six files: `api.go`, `api_test.go`,
`base_test.go` (new), `tei/tei.go`, `tei/tei_test.go`, `transport_test.go`.

oolong-project/oolong is not changed by OOL-4; integration and live checks use oolong
`2026-11-oolong` @ `e1c9e6e3068e` (unchanged since run 1).

Tools: Go 1.26.10 in a `golang:1.26` pod on the local k3s (namespace `oolong-tester-ool-4`),
Node 24 for the harness-started server.

## Review items from run 1

| Id | Status | How it was verified |
|---|---|---|
| T-1 | RESOLVED | New case in `TestJSONByteLimit`: a 404 whose TEA error object starts after 8192 bytes of spaces, `ErrorBodyBytes` 4096, must be a non-TEA 404. Run 1's mutation re-run: `transport.go:168` `body.limit = 1 << 40` now fails the test (`TEA:true, Code:"OBJECT_UNKNOWN"`, want a non-TEA 404). |
| T-2 | RESOLVED | `decodeBody` reason is now `body does not decode as <type>`, no decoder text. New `TestDecodeErrorCarriesNoBody` (bad time, bad uuid, wrong type, syntax error, each with a marker). Mutation (reason built from `err.Error()` again) fails it: the `time` case shows the marker in the error, and all four lose the type name. Every non-test JSON decode of a 2xx body (typed calls, pagination, token response) goes through `decodeBody`; error-body parsing keeps only the sanitised `message`, as design 3.6 allows. |
| T-3 | RESOLVED | `SyntaxError.Input` is truncated at construction on every path (`syntaxErr`, legacy-form refusal, unsupported legacy type, `PURL()`). The hollow check is replaced (`len(Input) <= 64` and a prefix of the input) and new `TestSyntaxErrorInputTruncated` covers each path with about 1500-byte inputs. Mutations, each failing at least one test: `syntaxErr` stores the input whole; `truncate` a no-op; `PURL()` whole; legacy-form whole; legacy-type whole. |
| T-4 | RESOLVED | New `base_test.go` `TestParseAPIBaseVersion`: refuses `2.0.0`, `1.0`, `v1.0.0`, `""`, `1.0.0-rc.1`, `1.0.1`, `01.0.0`, `1.0.0+` with `ErrInvalidArgument` and a zero base; accepts `1.0.0`, `1.0.0+b`, `1.0.0+build.5` with the version kept verbatim. Mutations: the supported-version check removed fails it (`2.0.0`, `1.0.0-rc.1`, `1.0.1`); both checks removed fails it for every refused value. Removing only the `validVersion` call in `ParseAPIBase` survives, but that is an equivalent mutant: `supportedVersion` calls `validVersion` itself, so only the message wording changes. |

### T-1: RESOLVED, test 11a now holds the ErrorBodyBytes cut

Mutation `body.limit = 1 << 40` (transport.go:168): `TestJSONByteLimit` fails with
`TEA:true, Code:"OBJECT_UNKNOWN"`, want a non-TEA 404 (body cut).

### T-2: RESOLVED, decode errors carry no body content

Mutation (reason from `err.Error()`): `TestDecodeErrorCarriesNoBody` fails; the `time` case shows
`parsing time "SECRET-BODY-CONTENT-7f3a" ...` in the error.

### T-3: RESOLVED, tei.SyntaxError.Input is at most 64 bytes

Mutations on `syntaxErr`, `truncate`, `PURL()`, the legacy form and the legacy type each fail
`TestParseRefused` or `TestSyntaxErrorInputTruncated` (e.g. "Input is 1525 bytes, want 64").

### T-4: RESOLVED, ParseAPIBase's version refusal is tested

Mutation removing the supported-version check: `TestParseAPIBaseVersion` fails for `2.0.0`,
`1.0.0-rc.1`, `1.0.1`. Removing only the separate `validVersion` call is an equivalent mutant.

The coder disputed nothing; no item needs a design change.

## Results

| # | Check | Result |
|---|---|---|
| 1 | `gofmt -l .` | empty |
| 2 | `go build ./...` | passed |
| 3 | `go vet ./...` and `go vet -tags integration ./...` | clean |
| 4 | `go test -count=1 ./...` | passed: 31 tests and 2 examples (root 25, `tei` 8 incl. 2 fuzz seed runs), 0 failed, 0 skipped |
| 5 | `go test -count=3 -race ./...` | passed |
| 6 | `go mod tidy -diff` | empty |
| 7 | `go generate ./...` then `git diff --exit-code` | no diff; header names oapi-codegen v2.8.2 |
| 8 | `go test -fuzz='^FuzzParse$' -fuzztime=30s ./tei` | passed (about 100k execs), no crasher, no corpus left |
| 9 | `go test -fuzz='^FuzzParseLegacyURN$' -fuzztime=30s ./tei` | passed (about 74k execs), no crasher |
| 10 | Integration (4.2) through the harness, `OOLONG_SERVER_DIR` at a built oolong-server (`e1c9e6e`), `TEA_INTEGRATION_REQUIRED=1` | 3 of 3 passed (cases 5, 6, 9) |
| 11 | Integration with `OOLONG_BASE_URL` at the in-cluster CI-built server | 3 of 3 passed |
| 12 | Harness with no `OOLONG_*`: skips with how to set them; with `TEA_INTEGRATION_REQUIRED=1` only: fails | passed |
| 13 | Plain-ASCII sweep (`LC_ALL=C grep -nP "[^\x00-\x7F]"`) over the 37 files changed against the base | hits only in `spec/openapi.yaml` (upstream byte copy) and `teaapi/tea.gen.go` (generated), the two exclusions 3.11 names; the six files changed this round are clean |
| 14 | No change under `tea-spec/` or `src/generated-nest/`; no `.github/` | passed |
| 15 | Commits since `4d16b0c`: four, each signed, each with both ReARM trailers | passed |
| 16 | Mutation checks for T-1 to T-4 (10 mutants, above) | 9 killed, 1 equivalent |
| 17 | Live check (below) | 35 of 35 run-1 checks plus 6 new, all passed |

Counts in the index (102 passed): 33 unit tests and examples, 6 integration runs (checks 10, 11),
1 harness check (12), 10 mutation checks (16), 41 live checks (17) and 11 other checks (1-3, 5-9,
13-15).

Unchanged since run 1 and not re-judged: the spec pin (no file under `spec/` changed), coverage
of the other design cases in scope (run 1 check 18; no test other than the ones named above
changed). `go mod tidy -modfile=tools.mod -diff` is not clean, but tidy against the tools modfile
pulls in the main module's imports by construction and `tools.mod` is unchanged since run 1, where
`go generate` reproduces the generated code; not an item.

## Live checks

Build: the CI-built image `registry.rearmhq.com/library/oolong-server:2026-11-oolong` (digest
`sha256:696876272b2e9609599bee8af13785311124731d615bd9039bb9810af977ffa2`, oolong `e1c9e6e`),
running in namespace `oolong-tester-ool-4` with the TEA 1.0 fixtures and the environment of
`oolong/docker-compose.tea-interop.yml`, as in run 1. The Go client ran at the merged tree
(`f39e89c`) in a Go pod in the same namespace. Nothing ran against a shared deployment.

- Run 1's throwaway live program (typed calls, error mapping, limits, URL policy, credentials,
  trace, user agent), not committed: **35 of 35 passed**, same results as run 1.
- New for this round (throwaway, not committed), all passed:
  - `ParseAPIBase(<origin>, "2.0.0")`, `"1.0"`, `"1.0.0-rc.1"`: `ErrInvalidArgument` before any
    request ("not supported" / "not SemVer").
  - `ParseAPIBase(<origin>, "1.0.0+build.5")`: accepted, path `/v1.0.0+build.5` verbatim.
  - `ParseAPIBase(<origin>, "1.0.0")` then `GetProduct(alpha)`: `Alpha Fixture`.
  - `tei.Parse` of a 3023-byte TEI: `*tei.SyntaxError`, `len(Input)` 64, message 95 bytes.

The decode-error and error-body changes cannot be provoked from oolong (it returns well-formed
bodies); they are covered by the unit tests and mutations above.

## Not tested, by decision

- **CI workflow (3.11 `ci.yaml`)**: deferred by the operator ("for this round don't build CI");
  unchanged from run 1. The checks that job would run were run by hand (checks 6, 7, 13).
- Discovery, downloads, checksums, flow, `WithAPIKey`, the live suite (4.3) and 4.2 cases 1-4, 7,
  8: not in this child (OOL-5, OOL-6).
