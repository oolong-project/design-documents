# OOL-4 implementation notes, round 3: fixes for test run 1 (T-1 to T-4)

Task: OOL-4 (child 1 of OOL-3). Answers `tests/OOL-4/run-1.md` (BOARD_TEST_REPORT round 1, REJECTED,
T-1 to T-4 open). Rounds 1 and 2 (`impl/OOL-4/notes-1.md`, `notes-2.md`) still describe the build,
the departures and the CI decision; nothing else changed. Design: `design/OOL-3/architecture-2.md`
and `design/OOL-3/threat_model-2.md` (round 2, carried). Coding principles read at `145008cdcc23`.

## Pull request

| PR | Repository | Base | Head commit | Base tip merged |
|---|---|---|---|---|
| https://github.com/oolong-project/tea-client-go/pull/1 | oolong-project/tea-client-go | `2026-11-oolong` | `f39e89cfc4a955c5725251b7dd9e41f830fca1c8` | `82d03dac23f9` (the base has not moved; nothing to merge) |

The tested head of run 1, `4d16b0c`, is an ancestor of the new head: four commits were added on top,
nothing was rewritten. No CI workflow, as decided in round 2. Merge order: one PR.

## Review items

| Id | Status | Commit |
|---|---|---|
| T-1 | fixed | `55f494e` |
| T-2 | fixed | `bc3cb3c` |
| T-3 | fixed | `2dfce8d` |
| T-4 | fixed | `f39e89c` |

### T-1: test 11a now asserts that a non-2xx body is cut at ErrorBodyBytes

The code was right; the test was hollow. `TestJSONByteLimit` (`transport_test.go`) gains a case: a
`404` with `application/json` whose body is 8192 spaces followed by `{"error":"OBJECT_UNKNOWN"}`,
with `ErrorBodyBytes` 4096. Cut at the limit, the body is only whitespace, so the result must be an
`*APIError` with status 404, `TEA` false and not `ErrObjectUnknown`. Unbounded, the decoder skips
the whitespace and finds the TEA code, so the assertion fails. The existing cases (a large 404 and
a large 500 still classified) are unchanged.

### T-2: decode errors carry no body content

`decodeBody` (`api.go`) now gives the fixed reason `body does not decode as <type>` (for example
`teaapi.ProductReleaseWithCollection`). It no longer appends the decoder's error text, which
quotes body values (a `time.ParseError` repeats the bad string). The doc comment says why (design
3.6). New test `TestDecodeErrorCarriesNoBody` (`api_test.go`) puts a marker string in a bad
`releaseDate` (time), a bad `uuid`, an object where a string belongs, and a syntax error after a
valid value. For each it asserts an `*UnexpectedResponseError`, that the error text does not contain
the marker, and that the reason names the type. `TestTypedCallUnexpectedJSON` (case 16) still
covers every typed call.

### T-3: tei.SyntaxError.Input is at most 64 bytes

Every `SyntaxError` is now built with the input truncated to 64 bytes: in `syntaxErr`, in the
legacy-form refusal of `Parse`, in the unsupported-type refusal of `ParseLegacyURN` and in `PURL()`.
`Error()` keeps its own truncation as a backstop. The doc comment was already right. Tests
(`tei/tei_test.go`):

- In `TestParseRefused` (case 4), the hollow check `len(se.Input) > len(c.in)` is replaced. It now
  checks that `Input` is at most 64 bytes and a prefix of the input. The 2049-byte case exercises
  it.
- New `TestSyntaxErrorInputTruncated`: inputs of about 1500 bytes through `PURL()`, `Parse`, the
  legacy form, an unsupported legacy type and an unsupported legacy hash algorithm. Each must give
  `len(Input) == 64` and an error message without more than 64 input bytes.

### T-4: ParseAPIBase's version refusal is tested

New `base_test.go` (named after `base.go`, where `ParseAPIBase` lives), test
`TestParseAPIBaseVersion`:

- `2.0.0`, `1.0`, `v1.0.0`, `""`, `1.0.0-rc.1`, `1.0.1`, `01.0.0` and `1.0.0+` are refused with
  `ErrInvalidArgument`, and the returned base is zero.
- `1.0.0`, `1.0.0+b` and `1.0.0+build.5` are accepted. `Version()` is kept verbatim, `String()` is
  `<root>/v<version as given>` and `RootURL()` is the root.

## Mutation checks (one-off, not committed; code restored after each)

| Mutation | Result |
|---|---|
| T-1: `body.limit = d.c.limits.ErrorBodyBytes` changed to `1 << 40` (the tester's mutation) | `TestJSONByteLimit` fails: the padded 404 is classified `TEA: true, OBJECT_UNKNOWN` |
| T-2: the reason built from `err.Error()` again | `TestDecodeErrorCarriesNoBody` fails: `parsing time "SECRET-BODY-..."` appears in the error |
| T-3: `syntaxErr` stores the input whole | `TestParseRefused` and `TestSyntaxErrorInputTruncated` fail |
| T-3: the unsupported legacy type stores the input whole | `TestSyntaxErrorInputTruncated` fails (Input 1525 bytes) |
| T-4: the supported-version check removed | `TestParseAPIBaseVersion` fails for `2.0.0` and `1.0.0-rc.1` |
| T-4: version matched by string equality (build metadata not ignored) | `TestParseAPIBaseVersion` fails for `1.0.0+b` and `1.0.0+build.5` |

One more mutation, dropping `semver.Canonical` before the comparison, made no test fail. It is an
equivalent mutant: `semver.Compare` ignores build metadata by itself, so behaviour does not change.

## Tests run on the head `f39e89c` (Go 1.26.0, Node 24.14.1), all passing

```sh
gofmt -l .                                        # empty
go vet ./... && go vet -tags integration ./...
go build ./...
go test -count=1 ./...                            # root: 25 tests and examples pass; tei: 10 pass
go test -count=1 -race ./...
go mod tidy -diff                                 # empty
LC_ALL=C /usr/bin/grep -nP "[^\x00-\x7F]" api.go api_test.go base_test.go transport_test.go tei/tei.go tei/tei_test.go   # nothing
# oolong 2026-11-oolong @ e1c9e6e (unchanged), npm ci && npm run build in oolong/oolong-server
OOLONG_SERVER_DIR=<oolong>/oolong/oolong-server TEA_INTEGRATION_REQUIRED=1 go test -count=1 -tags integration -run Integration ./...   # 3 of 3
```

To run only the new and changed tests:
`go test -count=1 -run 'TestJSONByteLimit|TestDecodeErrorCarriesNoBody|TestParseAPIBaseVersion' .`
and `go test -count=1 -run 'TestParseRefused|TestSyntaxErrorInputTruncated' ./tei`.

## For the tester

Nothing else changed: no new dependency, no change to `teaapi` (`go generate` is not needed), no
`.github/`. Only one change is visible to callers: `UnexpectedResponseError.Reason` for an
undecodable body now names the type instead of carrying decoder text.
