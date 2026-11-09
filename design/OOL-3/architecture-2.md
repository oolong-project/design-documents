# OOL-3 architecture, round 2: tea-client-go, a Go client library for TEA v1.0.0

Task: OOL-3. Coding principles: `coding-principles/coding_principles.md` at commit
`145008cdcc23`. Read at: oolong-project/oolong `2026-11-oolong` @ `0fac816`,
oolong-project/tea-client-go `2026-11-oolong` @ `82d03da` (LICENSE only),
github.com/relizaio/rearm-client-go @ `7dc78ac`, github.com/relizaio/rearm-cli @ `82f908b`
(`cmd/tea.go`), and the TEA specification at CycloneDX/transparency-exchange-api commit
`962a0228af1bfcc1232ac3f79239317ff33fdb04` (`spec/openapi.yaml`, `discovery/readme.md`,
`discovery/tea-well-known.schema.json`, `tei-types/*.md`, `LICENSE`). Every fact in 3.0 was
checked against those, or by running the command it names.

## Changes in this round

Answers the design review of round 1 (`review-items/OOL-3/design-reviewer/round-1.md`,
release `45584459-5e4b-4e9b-8cd6-ac405946d444`). The question the board pinned on this round
(`questionsRelease` `45584459-...`) is that same review-items release; it carries no other
question. Prototypes run for this round are listed in 3.0 ("Round 2 prototypes").

- **F-1** (Basic credentials across a `/token` redirect): the token request follows no
  redirects; any `3xx` from `/token` is an error, for `API.RequestToken` and for the
  automatic `WithAPIKey` request alike; the auth transport also removes an `Authorization`
  header from any request outside a configured credential root. Sections D5, 3.7; tests 14a,
  14b; threat model T4.
- **F-2** (time bounds): new `Limits.AttemptTimeout`, `Limits.CallTimeout`,
  `Limits.IdleTimeout`, all context-based so they apply with a caller's `http.Client` too;
  new `ErrTimeout`. Sections 3.4, 3.6 step 5, 3.8 step 4; tests 11d, 26b, 31b; threat model
  T1.
- **F-3** (`--usehttp` and "under"): an allowance covers URLs under its base, "under"
  defined by path segments (D4, also used for D5's roots); the CLI passes the origin it
  already uses, which covers a local ReARM's `/tea` endpoint. Dot segments are refused.
  `WithTrace` added so the CLI's debug output stays possible (operator guidance: what
  `cmd/tea.go` exposes today stays achievable, safe by default). Sections D4, D5, 3.4, 3.10,
  5; tests 9a, 13a, 21b, 27b, 39a, 39b; threat model T5, T13.
- **F-4** (split vs tests): cases 9, 11, 13 and 14 split into sub-cases, each assigned to the
  child that builds the code (case 17, the discovery and well-known fuzzers, also moves to
  OOL-3b); per-child lists rewritten and checked against the threat model's table, which
  gains a child column. Section 5; threat model section 3.
- **F-5** (validation the generated layer cannot see): uuids are accepted when
  `google/uuid` parses them and paths are rebuilt canonically (lowercase); the call kind
  reaches the doer as a context value, and `Generated()` calls are classified by path. D3,
  3.5, 3.6; tests 11c, 16, 24.
- **F-6** (uncapped results): new `Limits.DiscoveryResults` (default 32), enforced in
  `Discover` and `Candidates`; new `ErrTooManyResults`. 3.6, 3.9; tests 26c, 38b.
- **F-7** (live test asserts data): the TEI-uuid assertion is dropped; replaced by the
  contract that the fetched release has the uuid discovery named. 4.3 cases 1-2.
- **F-8** (licensing): `spec/LICENSE` is the upstream Apache-2.0 file (read at the pin);
  `teaapi/doc.go`, `SPEC_PIN.md` and README state it; nothing is copied from
  rearm-client-go. 3.0, 3.1, 3.11; test 1.
- **F-9** (dependencies, versioning): the generator moves to `tools.mod` (prototyped); the
  real `go.mod` requirements and the module-graph effect are stated; release and user-agent
  policy added. 3.0, 3.1, D1, 3.4, 3.11, 5; tests 1, 40; threat model T12.

## 1. Problem

The ReARM CLI talks to TEA servers with ad-hoc code in `cmd/tea.go` that targets an older
draft (the `urn:tei:` form, the well-known base reused for every call, an interactive prompt
inside discovery). This task builds `github.com/oolong-project/tea-client-go`: a typed Go
client for TEA v1.0.0, generated from the pinned OpenAPI document, with TEI parsing,
discovery as the spec defines it, verified artifact downloads and a flow helper, so that the
CLI's TEA commands can later become thin wrappers.

Out of scope: switching the CLI (a later task; this design names what it needs, 3.10);
signature verification (the spec does not define the technology); HTTP caching
(`ETag`/`If-None-Match`, discovery caching); any server change in oolong-project/oolong;
RFC 8693 token exchange and grant types other than `client_credentials`.

## 2. Repositories and branches

- oolong-project/tea-client-go, branch `2026-11-oolong` (PRs against it, never `main`):
  everything below.
- oolong-project/oolong: not changed. Its `2026-11-oolong` head is the integration-test
  server (4.2), checked out read-only by the tests and by CI.

## 3. Design

### 3.0 Facts this design rests on (checked)

- **Spec pin.** `tea-spec/SPEC_PIN.md` in oolong names repository
  `https://github.com/CycloneDX/transparency-exchange-api`, path `spec/openapi.yaml`, commit
  `962a0228af1bfcc1232ac3f79239317ff33fdb04`, git blob `c96baa3e102c195895643fab81ae468ee54b8182`,
  OpenAPI 3.1.1, `info.version` 1.0.0, prefix `/v1.0.0`. The file fetched from
  `raw.githubusercontent.com` at that commit has that blob id and is byte-identical to oolong's
  `tea-spec/openapi.yaml`. `discovery/tea-well-known.schema.json` at the same commit has blob
  `b0804f1ae79c8ec72adea6d5c12e8f75f6036729` (oolong does not carry it).
- **Operations.** The document has 28 operations, all `GET` except `POST /token`:
  `discover`; `queryProducts`, `getProduct`, `listProductReleases`, `getProductCle`;
  `queryProductReleases`, `getProductRelease`, `getProductReleaseCle`,
  `listProductReleaseCollections`, `getLatestProductReleaseCollection`,
  `getProductReleaseCollectionByVersion`; `queryComponents`, `getComponent`,
  `listComponentReleases`, `getComponentCle`; `queryComponentReleases`, `getComponentRelease`,
  `getComponentReleaseCle`, `listComponentReleaseCollections`,
  `getLatestComponentReleaseCollection`, `getComponentReleaseCollectionByVersion`;
  `getLatestArtifact`, `downloadLatestArtifact`, `downloadLatestArtifactSignature`,
  `getArtifactByVersion`, `downloadArtifactByVersion`, `downloadArtifactSignatureByVersion`;
  `requestToken`. Global security is `bearerAuth` or `{}` (anonymous allowed).
- **Closed sets in the spec.** `checksum-type`: SHA-1, SHA-256, SHA-384, SHA-512, SHA3-256,
  SHA3-384, SHA3-512, BLAKE2b-256, BLAKE2b-384, BLAKE2b-512, BLAKE3. `checksum.algValue`:
  lowercase hex, 40 to 128 chars, length must match the algorithm. `unknown-error-type`:
  OBJECT_UNKNOWN, NOT_IMPLEMENTED, NO_ACCEPTABLE_FORMAT, SIGNATURE_NOT_FOUND, INVALID_REQUEST,
  INVALID_PAGE_TOKEN. `identifier-type`: CPE, TEI, PURL, COMPLIANCE_DOCUMENT. `uuid`:
  lowercase pattern. Page size 1..100, default 25; sort fields per resource.
- **Artifact bytes.** `artifact-format.url`, when present, is always outside the TEA API and
  clients shall fetch from it; checksums are then required. When absent, content comes from
  `/artifact/{uuid}/{artifactVersion}/download?mediaType=...`. Checksums are over the bytes
  before HTTP content coding. A TEA access token is never sent to an external URL, nor to a
  different origin on a `302`. `Repr-Digest` "does not replace verification against the
  published checksums".
- **TEI** (`discovery/readme.md` "TEI syntax", "Comparing TEIs"): `tei://<domain-name>/<type>/<id>`;
  no port; type `[a-z][a-z0-9]*`; id one path segment, no `/` literally or as `%2F`; a client
  uses a TEI exactly as received and does not rewrite it. Registry at the pin (`tei-types/`):
  `purl` (Base64URL of the PURL, unpadded), `hash` (`<checksum-type>:<lowercase hex>`, e.g.
  `SHA-256:...`, not `SHA256`), `uuid`, `eanupc`, `gtin`, `asin`, `udi`. The registry is
  open: new types are added by PR.
- **Discovery** (`discovery/readme.md` "Selecting an API base", "Discovery response",
  "Unresolved identifiers", "Failover"): well-known at `https://<domain>/.well-known/tea`;
  pick endpoints with a mutually supported version, prefer the greatest such version (SemVer,
  build metadata ignored), then highest `priority` (absent = 1); call
  `<url>/v<version exactly as advertised>/discovery?tei=<query-encoded TEI>`. Success is a
  non-empty array, order = priority. Later calls go to a `servers[].rootUrl` chosen by the same
  rules, not to the well-known base. A `404` with a TEA error body stops discovery (no
  failover); DNS/TLS failure, `5xx`, a non-TEA `404`, unusable or non-conforming JSON are
  failed attempts that fail over, with a total attempt bound and exponential backoff. `401` and
  `403` never fail over. `http` bases are refused unless the client is configured to allow that
  specific base; "the allowance is per configured base URL, never a global setting".
- **Auth** (`/token` description, `bearerAuth`): a server needing auth implements `/token`
  with `client_credentials` and HTTP Basic, id and secret each form-urlencoded before joining
  (RFC 6749 2.3.1); clients never probe `/token`; a `401` with a Bearer challenge is the signal
  to get a token; on `error="invalid_token"` re-obtain and retry once. A server with no auth
  ignores a presented Bearer and answers `/token` with `404`.
- **Demo** (curl, 2026-11-09): `https://demo.rearmhq.com/.well-known/tea` is
  `{"schemaVersion":1,"endpoints":[{"url":"https://demo.rearmhq.com/tea","versions":["1.0.0"],"priority":1}]}`.
  `/tea/v1.0.0/discovery?tei=tei%3A%2F%2Fdemo.rearmhq.com%2Fuuid%2Fb9f115fb-6688-4a3f-a262-3acc04e1fc9e`
  returns one result, `productReleaseUuid` `b9f115fb-...`, server
  `https://demo.rearmhq.com/tea`, `["1.0.0"]`. `/tea/v1/discovery` and `/tea/discovery` are
  `404`. The product release has 10 component refs, all pinned; its latest collection holds
  one BOM, hosted (no `url`), SHA-256 only, 14,300,708 bytes; the downloaded bytes match the
  published SHA-256; the response carries `Repr-Digest`, a strong `ETag` and
  `Cache-Control: public, max-age=86400, immutable`. An unknown product is
  `404 {"error":"OBJECT_UNKNOWN"}`. The legacy form `urn:tei:uuid:demo.rearmhq.com:<uuid>` is
  refused: `400 {"error":"INVALID_REQUEST","message":"a TEI starts with tei://"}`.
- **Oolong server** (`oolong/oolong-server`, verified by running it): Node 24 is installed
  locally (`v24.14.1`); `npm ci` and `npm run build` succeed and produce `dist/main.js`.
  Started as `node dist/main.js` with `PORT`, `SERVER_HOST=http://127.0.0.1:<port>`,
  `CONTENT_DIR=<copy of oolong/fixtures/tea-1.0.0>`, `PAGE_TOKEN_SECRET=<random>`,
  `NODE_ENV=production`, it serves `/.well-known/tea` (`url` = `SERVER_HOST`, `503` until
  content is ready, `src/tea/well-known.controller.ts`), `/healthz` (liveness only) and the
  API under `/v1.0.0`. Discovery of `tei://localhost/uuid/22222222-2222-4222-8222-222222222222`
  returns one result whose `rootUrl` is `SERVER_HOST`. Unmounted paths return the Nest body
  `{"message":...,"error":"Not Found","statusCode":404}` (not a TEA error); CLE and `/token`
  return `404 {"error":"NOT_IMPLEMENTED"}`; a bad uuid is `400 INVALID_REQUEST`; revision 1 of
  artifact `55555555-...-555555555555` downloads as `302` to `https://example.invalid/...`.
  Fixture ids, bytes and checksums are in oolong `docs/tea-upgrade/fixture-reference.md`.
  `oolong/docker-compose.tea-interop.yml` runs the same server at `127.0.0.1:3005` (Docker is
  not available on this host; the design does not need it).
- **Go.** Go 1.26.0 is installed locally (`/usr/local/go/bin`); rearm-cli's `go.mod` says
  `go 1.26.0`. `crypto/sha3` is in the standard library.
- **Generator trial** (scratch directory, not committed): `oapi-codegen` v2.8.2
  (`-generate types,client`) consumes the pinned 3.1.1 document without error; the output builds
  and passes `go vet`; `allOf` responses come out flat (`PaginatedProductResponse` has
  `HasNext`, `NextPageToken`, `Results`; `ProductReleaseWithCollection` has
  `LatestCollection`); `date-time` is `time.Time`, `uuid` is `openapi_types.UUID`; enums are
  named string types with constants and a `Valid()` method; the low-level client takes an
  `HttpRequestDoer`; `ClientWithResponses` reads bodies with an unbounded `io.ReadAll` (28
  sites); download operations return `Body []byte`. Runtime deps: `oapi-codegen/runtime`
  v1.8.0, `google/uuid`. `ogen` v1.24.0 also consumes it and builds, but its generated code
  imports `go.opentelemetry.io/otel` and `go-faster/jx`, and its `ErrorResponse` decoder fails
  on an unknown field (`unexpected field`).
- **What the CLI does today** (`cmd/tea.go`): `discovery` and `full_tea_flow` commands with
  `--tei`, `--usehttp`, `--useport` (port for the well-known fetch); accepts only
  `urn:tei:<type>:<domain>:<id>` with types uuid, purl, hash, swid, asin, gs1; resolves DNS
  first; picks the well-known endpoint by priority only and its `versions[0]`; uses that base
  for every later call (not `servers[]`); prompts on stdin when discovery returns several
  results; for an unpinned component takes `releases[0]` of the default listing (sorted by
  `createdDate` ascending, i.e. the oldest); prints product, CLE, component releases, and for
  each artifact format its description, media type, URL and signature URL.
- **rearm-client-go's shape**: package `rearm` at module root; `generate.go` with
  `//go:generate go run github.com/Khan/genqlient`; `tools.go` pins the generator; generated
  code checked in; `New(baseURL, ..., opts ...Option)` with `WithHTTPClient`, `WithUserAgent`;
  response reads capped with `io.LimitReader`; README with a usage example; CI workflow
  `rearm-submit.yaml`: a `test` job (`go test ./...`, actions pinned by SHA) and a
  `record-release` job using `relizaio/rearm-actions` (`441a3590...`, v1.9.0) with
  `rearm_auth: github-oidc` and no stored secret. Its `LICENSE` is MIT, "Copyright (c)
  2019-2026 Reliza Incorporated".
- **Licence of the spec** (read at the pin): the repository's `LICENSE` at
  `962a0228...` is the Apache License 2.0 with "Copyright OWASP Foundation" (git blob
  `0808751c23d827d483dc3673d6e3f49514e49d1f`); `spec/LICENSE`, `discovery/LICENSE` and
  `NOTICE` do not exist at that commit (`404` from `raw.githubusercontent.com`);
  `spec/openapi.yaml` declares `info.license` `Apache-2.0`.
- **Go `net/http` on redirects** (prototype, Go 1.26.0): a `POST` carrying
  `Authorization: Basic ...` set on the request, answered `302` or `307` with a target on
  the same host at another port, or on a subdomain of the origin host, reaches that target
  with the `Authorization` header intact. With `CheckRedirect` returning
  `http.ErrUseLastResponse` no request reaches the target and the `3xx` response is
  returned to the caller.
- **Round 2 prototypes** (scratch, not committed), Go 1.26.0:
  - the redirect case above (`httptest` servers, a dialer mapping `*.localhost` to
    `127.0.0.1`);
  - a server that sends headers and one body byte then stalls: `io.ReadAll` under a
    `context.WithTimeout` of 300 ms returns `context deadline exceeded` after 300 ms;
  - a module with `go.mod` and a separate `tools.mod`: `go get -modfile=tools.mod -tool
    github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen@v2.8.2`, then from `teaapi/`
    `go tool -modfile=../tools.mod oapi-codegen -config oapi-codegen.yaml ../spec/openapi.yaml`
    generates `tea.gen.go` (442 KB); `go mod tidy` leaves `go.mod` with only
    `github.com/oapi-codegen/runtime v1.8.0` and the indirect `github.com/apapsch/go-jsonmerge/v2
    v2.0.0` and `github.com/google/uuid v1.6.0`; `go build ./...` passes. That module's
    graph has 80 modules (`go list -m all`): `runtime` v1.8.0's own `go.mod` requires gin,
    echo, iris and `golang.org/x/crypto` v0.46.0, `x/net` v0.48.0, `x/sys` v0.39.0,
    `x/text` v0.32.0, `gopkg.in/yaml.v3` v3.0.1 among others;
  - decoding `teaapi.DiscoveryInfo`: `productReleaseUuid` values in uppercase, `{...}`,
    `urn:uuid:` and 32-hex-without-hyphens forms decode (to the same lowercase value);
    `"not-a-uuid"`, `""` and a 35-character value fail to decode;
  - `debug.ReadBuildInfo()` inside the module's own `go test`: `Main.Path` is the module,
    `Main.Version` is `(devel)`, `Deps` is empty.
- **rearm-cli dependencies** (`go.mod` @ `82f908b`): `go 1.26.0`; it already requires
  `golang.org/x/sys` v0.48.0, `x/text` v0.42.0, `x/mod` v0.41.0, `google/uuid` v1.6.0,
  `gopkg.in/yaml.v3` v3.0.1, and neither `x/crypto` nor `x/net`.
- **What `--usehttp` does today** (`cmd/tea.go` `queryWellKnown`, `createHTTPClient`): it
  fetches `http://<domain>:<useport>/.well-known/tea`; every later URL is the endpoint
  `url` from that document, used as written (so `http` when the document says `http`); one
  `http.Client` with `Timeout: 30 * time.Second` per call. With `debug=true` it prints the
  domain, the resolved hosts, the well-known URL and the discovery URL (TEI query
  included) before requesting them.

### 3.1 Layout of the module

```
go.mod, go.sum              module github.com/oolong-project/tea-client-go; go 1.26.0 (no tool directive)
tools.mod, tools.sum        same module path; go 1.26.0;
                            tool github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen (v2.8.2)
spec/openapi.yaml           byte copy of the pinned upstream file (never edited)
spec/tea-well-known.schema.json  byte copy, same commit
spec/LICENSE                byte copy of the upstream LICENSE at the same commit (Apache-2.0)
spec/SPEC_PIN.md            repository, paths, commit, blob ids, retrieved date, licence of spec/
teaapi/                     package teaapi: GENERATED models and low-level client
  doc.go                    hand-written package doc: generated from ../spec/openapi.yaml; licence note (3.11)
  generate.go               //go:generate go tool -modfile=../tools.mod oapi-codegen -config oapi-codegen.yaml ../spec/openapi.yaml
  oapi-codegen.yaml         package teaapi, output tea.gen.go, generate models + client
  tea.gen.go                generated, checked in, never edited
tei/                        package tei: TEI parsing, legacy URN conversion (no deps)
doc.go client.go transport.go base.go discovery.go api.go pagination.go
errors.go auth.go download.go checksum.go flow.go      package tea (hand-written)
internal/oolongtest/        starts the oolong server for integration tests (4.2)
*_test.go                   unit tests (httptest)
integration_test.go         //go:build integration
live_test.go                //go:build live
example_test.go             runnable examples, quoted by README.md
README.md
.github/workflows/ci.yaml   (3.11)
```

The root package is named `tea` (import path ends in `tea-client-go`, as rearm-client-go's
root package is `rearm`); the package doc and README show `import tea "github.com/oolong-project/tea-client-go"`.

**Dependencies (`go.mod`).** Direct requirements: `github.com/oapi-codegen/runtime` v1.8.0
(imported by the generated code: `runtime` and `runtime/types`), `github.com/google/uuid`
v1.6.0 (in the `tea` package's signatures, and under `openapi_types.UUID`),
`golang.org/x/crypto` (`blake2b`) and `golang.org/x/mod` (`semver`), both at their newest
release when the coder starts, named in the PR. Indirect: `github.com/apapsch/go-jsonmerge/v2`
v2.0.0 (required by `runtime`). Nothing else is imported. What this means for a consumer
such as the CLI: module graph pruning puts every requirement listed in `runtime` v1.8.0's
`go.mod` into the consumer's module graph (gin, echo, iris and their requirements, and
minimums such as `x/crypto` v0.46.0, `x/net` v0.48.0, `x/sys` v0.39.0, `x/text` v0.32.0,
3.0). They take part in minimal version selection, so they can raise a consumer's version of
a module it shares, but packages a consumer does not import are not compiled into it. For
the CLI today that is no raise for `x/sys`, `x/text`, `x/mod`, `uuid` or `yaml.v3` (it is
already newer or equal) and two new graph entries, `x/crypto` and `x/net`. This is accepted:
oapi-codegen's client has no form without `runtime`, and it is the price of the generated,
compile-time-checked layer the task asks for. The generator itself is **not** in `go.mod`:
it lives in `tools.mod` (D1), so its graph (kin-openapi, `x/tools`, yaml, ...) never
reaches a consumer.

### 3.2 Decisions

- **D1, generator: oapi-codegen v2.8.2, models and client, into `teaapi`.** It reads the
  pinned 3.1.1 document as is (3.0), yields flat `allOf` types, typed enums with `Valid()`,
  `time.Time` timestamps and a client over an injectable `HttpRequestDoer`, with two small
  runtime dependencies. A spec change regenerates `teaapi` and breaks consumers at compile
  time, as genqlient does for rearm-client-go. Rejected: **ogen** (pulls OpenTelemetry into
  every consumer, and its strict decoders reject unknown fields, which `discovery/readme.md`
  "Unresolved identifiers" tells clients to ignore, so a later TEA minor would break it);
  **openapi-generator** `go` (needs a Java runtime, so no plain `go generate`, and much larger
  output). The generator is pinned with a `tool` directive in a separate modfile,
  `tools.mod` (with `tools.sum`), and run as `go tool -modfile=../tools.mod oapi-codegen`
  from `teaapi/generate.go` (prototyped, 3.0), so `go generate ./...` needs nothing
  installed and the generator's requirement graph stays out of `go.mod` and out of every
  consumer's build (F-9 of round 1; rearm-client-go's `tools.go` puts genqlient in its
  `go.mod`, which this design deliberately does not copy). A generator bump is
  `go get -modfile=tools.mod -tool ...@<version>` followed by a regenerate. CI checks the
  checked-in output is current (3.11). "Generated code is not edited by hand" applies to `teaapi/tea.gen.go`: a regenerate
  is its own commit with nothing else in it. Where the generated code is unsuitable (the
  unbounded reads and `[]byte` downloads above), the hand-written layer does not use that part
  (D3, 3.8), it does not patch it.
- **D2, two layers.** `teaapi` is exported, so consumers can name every spec type
  (`teaapi.Product`, `teaapi.Collection`, ...). The hand-written `tea` package is the API people
  use: it returns `teaapi` types (no aliases, no copies: one source of truth for shapes) and
  owns everything the spec leaves to the client: URL policy, limits, auth, errors, discovery,
  verification. `API.Generated()` returns the `*teaapi.ClientWithResponses` built on the same
  bounded, auth-aware doer for anything not wrapped.
- **D3, every response is bounded in bytes and in time** ("The Go client treats every server
  as untrusted": bound what it reads; a time bound is how a byte bound holds against a server
  that sends slowly). The doer the generated client
  uses wraps each response body in a reader that fails with `ErrResponseTooLarge` past the
  byte limit for that kind of call (3.4), and refuses at once when `Content-Length` is
  larger. The limit applies after the transport's own gzip decoding, so a compressed bomb is
  caught too. The same doer applies the time bound for the kind (3.4: `AttemptTimeout`,
  `CallTimeout` or `IdleTimeout`), through the request's context, so it holds whatever
  `http.Client` the caller passed. **How the doer knows the kind:** every hand-written
  wrapper sets it on the request context (an unexported context key carrying a closed
  `callKind`: `kindWellKnown`, `kindDiscovery`, `kindJSON`, `kindToken`, `kindDownload`,
  `kindSignature`). A request with no kind, which is any call made through `Generated()`,
  is classified by its path relative to the API base, against the spec's templates:
  `/artifact/{uuid}/latest/download` and `/artifact/{uuid}/{v}/download` are
  `kindDownload` (`DownloadBytes`, `IdleTimeout`); `/artifact/{uuid}/latest/signature` and
  `/artifact/{uuid}/{v}/signature` are `kindSignature` (`SignatureBytes`, `IdleTimeout`);
  `/discovery` is `kindDiscovery` (`DiscoveryBytes`, `AttemptTimeout`); everything else is
  `kindJSON` (`JSONBytes`, `CallTimeout`). A non-2xx body is always read up to
  `ErrorBodyBytes`, whatever the kind. The generated download operations read the whole body
  into `[]byte`, so through `Generated()` a download can hold up to `DownloadBytes` in
  memory; the doc comment of `Generated()` says so and points to `DownloadArtifact`.
- **D4, URL policy: https only, plus per-base http allowances that cover what is under the
  base.** Every URL the client requests (well-known, API bases, redirects, external artifact
  URLs) must be `https`, or `http` and **under** a base the caller passed with
  `WithInsecureHTTPBase`. There is no global switch, and nothing a server sends can add an
  allowance. **"Under"** (used here and by D5) is defined on parsed URLs: same scheme
  (lowercase), same host (compared lowercased, IDNA not applied: a non-ASCII host is refused),
  same port (an absent port means the scheme's default, so `http://h` and `http://h:80` are
  the same); and the URL's escaped path equals the base's path, or starts with the base's
  path followed by `/`. A base path is stored without a trailing `/` (validation refuses one,
  as the spec's patterns do), so the empty path of an origin `http://host:port` covers every
  path on that origin, and `/tea` covers `/tea` and `/tea/v1.0.0/...` but not `/teapot`. A
  URL whose path has a dot segment (`.` or `..`, also as `%2e`/`%2E`, in any segment) is
  refused before any allowance or credential decision. A base learned from a server
  (well-known `url`, `servers[].rootUrl`, a redirect) is usable over http only when it is
  under a configured allowance; it never becomes an allowance itself. This is a stated
  reading of the spec's "per configured base URL, never a global setting, so it cannot apply
  to a base the client learned from discovery" (`discovery/readme.md` "Selecting an API
  base"): the allowance is attached to a string the caller configured, and a hostile
  document can choose at most a path on an origin the caller has already declared plain-http
  for local testing; it can never reach another scheme, host or port over http. The reason
  for covering paths under the base, and not only byte-equal bases as round 1 did, is the
  CLI's `--usehttp` (3.10): a local ReARM serves its well-known at the origin and lists
  `<origin>/tea`, which the CLI cannot know before fetching the document. Named as a
  departure in section 5. No URL with userinfo is followed. TLS verification is the
  `http.Client`'s; the library never sets `InsecureSkipVerify` and exports no option to.
- **D5, credentials are bound to one API root and added by the transport.** Credentials are
  configured per `rootUrl` (`WithBearerToken(rootURL, token)`, `WithAPIKey(rootURL, id,
  secret)`). The auth transport is the only place an `Authorization` header is decided, on
  every hop, redirects included: it first **removes** any `Authorization` header from a
  request that is not under (D4's definition) a configured credential root, then adds the
  root's credential to a request under it. The one exemption: a `kindToken` request is left
  exactly as the token code built it (its Basic header is neither removed nor replaced),
  because it follows no redirect and so has one hop only (3.7). So a `302` to another origin, to another port or
  to a subdomain, an external `url`, or a `servers[]` entry on another host never sees the
  token, whatever Go's own redirect header rules would do (Go 1.26 forwards `Authorization`
  set on a request to another port on the same host and to subdomains, 3.0), and a header a
  caller sets through a `Generated()` request editor is dropped off-root too. The token
  request is additionally made with no redirects followed (3.7): Basic credentials go to
  exactly one URL. Credentials never appear in a URL, an error, a trace event or a log line
  ("Secrets and credentials").
- **D6, TEI parsing keeps the TEI exactly as written**; unknown but well-formed types are
  accepted and marked unknown (the registry is open). Legacy `urn:tei:` strings are not TEIs:
  `tei.Parse` refuses them, and a separate, explicit `tei.ParseLegacyURN` converts them
  (3.3). Discovery only ever sends `tei://` strings, which is what both oolong and the demo
  accept.
- **D7, discovery returns all results and never prompts.** `Discover` returns the ordered
  result array; the flow helper takes a caller-supplied selector or returns `*AmbiguousError`
  carrying the results (3.9).
- **D8, unknown JSON properties are ignored; unknown enum values are kept, and refused only
  where the client acts on them.** `encoding/json` ignores unknown fields (forward compatible,
  as the spec asks for `error-response`); an unknown `checksum-type` makes that checksum
  unusable (3.8), an unknown error code maps to `ErrorCodeUnknown`. This is a stated
  departure from "a client that accepts a response shape the spec does not define is a bug":
  the spec itself asks clients to tolerate extensions, and the generator does not validate
  JSON schemas. What the client relies on to build URLs or to trust bytes is validated
  strictly (3.5, 3.6, 3.8): well-known and discovery documents in full, `rootUrl`, versions,
  pagination tokens, checksums; uuids must decode, and every path is rebuilt from the decoded
  value in canonical form (3.5).
- **D9, no automatic retries of API calls**, except discovery's bounded failover (3.6) and
  the single token recovery on `401` (3.7). The caller's `context.Context` bounds every call,
  and the library's own time bounds (3.4) apply inside it: the earlier deadline wins.
- **D10, versions.** The client supports exactly `1.0.0` (`tea.SupportedVersions`).
  `golang.org/x/mod/semver` compares versions after a strict SemVer 2.0.0 check with the
  spec's pattern (x/mod accepts shorthand forms the spec does not); build metadata is ignored
  for matching and the advertised string is used verbatim in the path.

### 3.3 Package `tei`

```go
type Type string                        // closed set of known types, open for unknown
const (TypeUUID Type = "uuid"; TypePURL = "purl"; TypeHash = "hash"; TypeEANUPC = "eanupc"
       TypeGTIN = "gtin"; TypeASIN = "asin"; TypeUDI = "udi")
func (t Type) Known() bool

type TEI struct{ raw, domain string; typ Type; id string }
func Parse(s string) (TEI, error)
func (t TEI) String() string            // exactly the input
func (t TEI) Domain() string            // as written
func (t TEI) Type() Type
func (t TEI) ID() string                // as written (still percent-encoded)
func (t TEI) PURL() (string, error)     // TypePURL only: Base64URL (unpadded) decode
func ParseLegacyURN(s string) (TEI, error)
func ParseCompat(s string) (t TEI, legacy bool, err error) // tei:// first, then urn:tei:
var ErrSyntax, ErrLegacyForm, ErrLegacyTypeUnsupported error
```

`Parse` rules, in order, each failure an `*SyntaxError` wrapping `ErrSyntax` that names the
rule and never echoes more than 64 bytes of input:

1. Length at most 2048 bytes; ASCII only, no control characters or spaces.
2. Prefix exactly `tei://` (lowercase; an input starting `urn:tei:` gets `ErrLegacyForm`,
   whose message says to use `tei://` or `ParseLegacyURN`).
3. Exactly three parts split on `/` after the prefix: domain, type, id. A fourth part, `?` or
   `#` anywhere is refused.
4. Domain: a DNS name: labels of `[A-Za-z0-9-]`, 1 to 63 chars, not starting or ending with
   `-`, total at most 253, no trailing dot; no `:` (port), `@`, `[`; not an IPv4 literal (all
   labels numeric). `localhost` is accepted (oolong's fixtures use it). Case is kept; the
   well-known URL uses the lowercased domain (DNS is case-insensitive, `discovery/readme.md`
   "Comparing TEIs").
5. Type: `^[a-z][a-z0-9]*$`, at most 32 chars. Unknown types parse (`Known()` false).
6. Id: non-empty, RFC 3986 `pchar` only (unreserved, sub-delims, `:`, `@`, `%HH`), every `%`
   followed by two hex digits, and no `%2F` or `%2f`. Not decoded, not normalised.
   Type-specific canonical forms (lowercase uuid, unpadded Base64URL) are not enforced: a TEI
   is used as received, and a non-canonical one is the publisher's problem, resolved or not by
   their server.

`ParseLegacyURN` converts `urn:tei:<type>:<domain>:<id>` (the form `cmd/tea.go` accepts, with
`<id>` the rest after the second colon, colons included) into `tei://<domain>/<type>/<id'>` and
then runs `Parse` on the result:

| legacy type | conversion of `<id>` |
| --- | --- |
| `uuid`, `eanupc`, `gtin`, `asin`, `udi` | unchanged |
| `purl` | the raw PURL as written, Base64URL-encoded without padding (no canonicalisation) |
| `hash` | `<alg>:<hex>`: `<alg>` mapped to `checksum-type` spelling when it is one of `SHA1`, `SHA256`, `SHA384`, `SHA512` (to `SHA-1` ...) or already a `checksum-type` value; else refused. Hex unchanged |
| `swid`, `gs1`, anything else | `ErrLegacyTypeUnsupported` (no v1.0.0 registry entry) |

The conversion is a migration aid for the CLI, not identity: the result may differ from what
the publisher prints (a non-canonical PURL, uppercase hex), in which case discovery answers
`404 OBJECT_UNKNOWN`, which is the correct outcome. Nothing in the library calls it
implicitly; `ParseCompat` exists so the CLI can accept both forms and warn on `legacy`.

### 3.4 Client, options and limits (`client.go`, `transport.go`)

```go
func New(opts ...Option) (*Client, error)
type Option func(*options)
func WithHTTPClient(hc *http.Client) Option      // copied; the library sets CheckRedirect on its copy
func WithUserAgent(ua string) Option             // default "tea-client-go/<module version>" (below)
func WithInsecureHTTPBase(rawURL string) Option  // D4; repeatable; absolute http URL, no userinfo,
                                                 // query, fragment, trailing "/" or dot segment
func WithWellKnownLocation(domain, baseURL string) Option // fetch <baseURL>/.well-known/tea for domain
func WithBearerToken(rootURL, token string) Option        // D5
func WithAPIKey(rootURL, keyID, secret string) Option     // D5, 3.7
func WithLimits(l Limits) Option
func WithDiscoveryAttempts(n int, firstBackoff time.Duration) Option
func WithTrace(f func(TraceEvent)) Option        // observer for debug output; never blocks a call
type TraceEvent struct {
    Kind   TraceKind     // TraceRequest, TraceResponse, TraceAttemptFailed, TraceBackoff
    Stage  Stage         // StageWellKnown, StageAPIDiscovery, StageAPI, StageDownload, StageToken
    Method string
    URL    string        // scheme, host, port and path only: never query, fragment or userinfo
    Status int           // TraceResponse
    Reason string        // TraceAttemptFailed: same text as the Attempt in DiscoveryError
    Wait   time.Duration // TraceBackoff
}
func (c *Client) API(base APIBase) (*API, error)
func ParseAPIBase(rootURL, version string) (APIBase, error)
type APIBase struct{ /* rootURL *url.URL; version string */ }
func (b APIBase) RootURL() string; func (b APIBase) Version() string; func (b APIBase) String() string // root + "/v" + version
```

`WithWellKnownLocation` replaces `--usehttp`/`--useport` for the CLI and is how tests point
`tei://localhost/...` at a local server; an `http` location also needs
`WithInsecureHTTPBase` covering it (3.10 for the CLI mapping). `New` refuses an option value
it cannot parse (`ErrInvalidOption`), and an http root given to `WithBearerToken`/`WithAPIKey`
that is not under an insecure base.

`WithTrace` is how the CLI keeps its `debug=true` output (3.10): the function is called
synchronously, in request order, for every request the client makes (well-known, discovery,
API, token, download, redirects included) and for each failed discovery attempt and backoff.
An event never carries a header, a body, a query string (presigned artifact URLs carry
secrets there) or a credential; the TEI is not in the URL, so a caller prints it itself, as
the CLI does today. A panic in the function is not recovered (it is the caller's code).

`Limits` (zero value = default; for the three durations, a negative value disables that
bound and leaves only the caller's context):

| field | default | applies to |
| --- | --- | --- |
| `WellKnownBytes` | 64 KiB | `.well-known/tea` |
| `DiscoveryBytes` | 1 MiB | `/discovery` |
| `DiscoveryResults` | 32 | results in one `/discovery` answer, and results given to `Candidates` (3.6, 3.9) |
| `JSONBytes` | 16 MiB | every other JSON response |
| `ErrorBodyBytes` | 64 KiB | non-2xx bodies (read to classify, then discarded) |
| `DownloadBytes` | 512 MiB | artifact content (per call override in `DownloadOptions`) |
| `SignatureBytes` | 16 MiB | signature content |
| `Redirects` | 5 | per request |
| `Pages` | 1000 | per iterator (3.5) |
| `FlowComponents` | 1000 | component refs walked by the flow helper |
| `AttemptTimeout` | 15 s | each discovery attempt: one well-known GET or one `/discovery` GET, headers and body (3.6) |
| `CallTimeout` | 30 s | each non-download request: every typed JSON call, `/token`, and `Generated()` calls of `kindJSON`; headers and body |
| `IdleTimeout` | 30 s | downloads and signatures (wrapped and through `Generated()`): longest wait for the response headers or for the next body byte |

The demo's one BOM is 14.3 MB and oolong refuses artifact bodies over 32,000,000 bytes, so
512 MiB leaves room without letting a server fill a disk. `CallTimeout` matches the CLI's
`Timeout: 30 * time.Second` today (3.0); `AttemptTimeout` with the default 4 attempts and
backoff bounds a whole `Discover` at about 62 s.

**How the time bounds work** (D3). They are applied by the bounded doer through the request
context, never through `http.Client.Timeout` or the transport, so they hold for a caller's
`http.Client` as well:

- `AttemptTimeout` and `CallTimeout`: `context.WithTimeoutCause(ctx, d, ErrTimeout)` around
  the request; the cancel function runs when the body is closed or fully read. When the
  caller's context has an earlier deadline, that one fires first (the earlier wins, as
  `context` does by itself).
- `IdleTimeout`: a `context.WithCancelCause` around the request and a `time.AfterFunc(d)`
  that cancels it with `ErrTimeout`, started when the request is sent and reset after each
  `Read` that returns bytes. A download that keeps sending bytes is never cut off; one that
  stops for `d` is.
- The error: when the library's bound fired (`context.Cause` is `ErrTimeout`) and the
  caller's context is still live, the call returns an error wrapping `ErrTimeout` (for
  discovery: a failed attempt, 3.6). When the caller's context ended, its `ctx.Err()` is
  returned wrapped, not `ErrTimeout`.

The default transport (when the caller passes no `http.Client`) is a clone of
`http.DefaultTransport` with `TLSHandshakeTimeout` 10 s and `ResponseHeaderTimeout` 30 s, as a
second line under the context bounds; there is no whole-request `http.Client.Timeout`
(downloads are long; `IdleTimeout` and the context bound them).

**User agent.** The default is `tea-client-go/<v>`. `<v>` comes from
`debug.ReadBuildInfo()`: the `Deps` entry whose `Path` is
`github.com/oolong-project/tea-client-go` (its `Replace.Version` when replaced with a
version, else its `Version`), or `Main.Version` when this module is the main module; an
empty value, `(devel)` or no build info gives `devel`. In the module's own tests it is
therefore `tea-client-go/devel` (3.0). Computed once.

`CheckRedirect` on the library's copy of the `http.Client`: at most `Limits.Redirects`; every
target passes D4; the auth decision is the transport's (D5). A refused redirect returns
`*URLPolicyError` (wraps `ErrInsecureURL` or `ErrRedirectLimit`). A request of `kindToken`
follows no redirect at all (3.7): `CheckRedirect` returns `http.ErrUseLastResponse` for it,
and the token code turns the `3xx` into an error.

`Client` and `API` are safe for concurrent use; the token cache (3.7) is guarded by a mutex.
Every exported call that does I/O takes a `context.Context` first.

### 3.5 Typed calls (`api.go`, `pagination.go`)

`API` is bound to one `APIBase`. One method per operation of 3.0, returning `teaapi` types:

```go
func (a *API) Base() APIBase
func (a *API) GetProduct(ctx, id uuid.UUID) (*teaapi.Product, error)
func (a *API) QueryProducts(ctx, q ProductQuery) (*teaapi.PaginatedProductResponse, error)
func (a *API) ListProductReleases(ctx, id uuid.UUID, p ReleasePage) (*teaapi.PaginatedProductReleaseResponse, error)
func (a *API) GetProductCLE(ctx, id uuid.UUID) (*teaapi.Cle, error)
func (a *API) QueryProductReleases(ctx, q ReleaseQuery) (...)
func (a *API) GetProductRelease(ctx, id uuid.UUID) (*teaapi.ProductReleaseWithCollection, error)
func (a *API) GetProductReleaseCLE(...); ListProductReleaseCollections(ctx, id, CollectionPage)
func (a *API) GetLatestProductReleaseCollection(ctx, id) (*teaapi.Collection, error)
func (a *API) GetProductReleaseCollection(ctx, id uuid.UUID, version int) (*teaapi.Collection, error)
// the same eight for components and component releases
func (a *API) GetLatestArtifact(ctx, id) (*teaapi.Artifact, error)
func (a *API) GetArtifact(ctx, id uuid.UUID, version int) (*teaapi.Artifact, error)
func (a *API) DownloadArtifact(...), DownloadArtifactToFile(...), DownloadSignature(...)   // 3.8
func (a *API) RequestToken(ctx, keyID, secret string) (*teaapi.TokenResponse, error) // 3.7
func (a *API) Generated() *teaapi.ClientWithResponses
// iterators (Go 1.23 range-over-func), bounded by Limits.Pages:
func (a *API) AllProducts(ctx, q ProductQuery) iter.Seq2[teaapi.Product, error]
// ... AllProductReleases, AllComponents, AllComponentReleases, and the per-parent lists
```

`ProductQuery`, `ReleaseQuery`, `ReleasePage`, `CollectionPage` carry `PageSize`,
`PageToken`, `SortOrder` (`SortAsc`/`SortDesc`), the resource's `SortField` as a named type
with the spec's constants, and for the query operations `IDType` (`teaapi.IdentifierType`) and
`IDValue`. They are checked before any request: page size 0 (= server default) or 1..100,
enum values `Valid()`, `IDValue` without `IDType` refused (`ErrInvalidArgument`; the spec
leaves `idValue` alone undefined and oolong answers `400`).

Each method: build the request through the generated client (path params are typed uuids and
ints, so a value cannot inject a path), send it through the bounded doer, then map the result:
2xx with a decoded JSON body of the declared type is returned; 2xx with a missing or
undecodable body is `*UnexpectedResponseError` (never a nil-pointer panic); anything else goes
through 3.6's error mapping. **Uuids in responses:** a uuid field is accepted when
`google/uuid` decodes it (the `openapi_types.UUID` the generated types use; it also accepts
uppercase, `{...}`, `urn:uuid:` and 32-hex forms, 3.0), and refused when it does not (the
body is then undecodable: `*UnexpectedResponseError`, or a failed attempt in discovery).
The original string is not kept and not re-checked against the spec's lowercase pattern:
every path the library builds from a uuid is rebuilt from the decoded value with
`uuid.UUID.String()`, which is canonical lowercase, so a non-canonical spelling from a server
cannot reach a path. This is part of D8's stated departure (a server that sends uppercase
uuids is non-conforming, and that is the server's test suite's concern). A response is also
checked for what callers rely on: pagination with `hasNext: true` and no `nextPageToken` is
`*UnexpectedResponseError`. Iterators stop at the
first error, at `hasNext: false`, or with `ErrPageLimit` after `Limits.Pages` pages; a page
token is passed back verbatim, never parsed (it is opaque).

`downloadLatestArtifact` and `downloadLatestArtifactSignature` are not wrapped as their own
methods: verification needs the checksums of the revision whose bytes are fetched, and the
latest endpoint can move between the metadata read and the download. `DownloadLatest...`
helpers therefore read `GetLatestArtifact` and download that `version` through the versioned
endpoint. The two operations remain reachable through `Generated()`.

### 3.6 Errors (`errors.go`) and discovery (`discovery.go`)

Errors (all exported, all wrap with context, `fmt.Errorf("getting product release %s: %w", id, err)`):

```go
type ErrorCode string // closed: the six spec codes + ErrorCodeUnknown ("") for others
type APIError struct {
    Operation string; Status int
    TEA bool              // body was a TEA error-response (application/json, object, string "error")
    Code ErrorCode        // when TEA
    RawCode string        // the code as received when unknown, truncated to 64 bytes
    Message string        // sanitised: control chars dropped, at most 200 bytes
    Challenge *Challenge  // parsed WWW-Authenticate on 401 (scheme, error param)
}
func (e *APIError) Is(target error) bool // matches the sentinels below
var ErrObjectUnknown, ErrNotImplemented, ErrSignatureNotFound, ErrNoAcceptableFormat,
    ErrInvalidRequest, ErrInvalidPageToken, ErrUnauthorized, ErrForbidden, ErrServer error
type UnexpectedResponseError struct{ Operation string; Status int; Reason string }
type URLPolicyError struct{ URL string /* redacted: no query */; Reason string } // ErrInsecureURL, ErrRedirectLimit
var ErrResponseTooLarge, ErrInvalidArgument, ErrInvalidOption, ErrPageLimit, ErrTooManyComponents,
    ErrTooManyResults, ErrTimeout error
```

A body is a TEA error only when `Content-Type` is `application/json` (parameters allowed) and
the body is a JSON object with a string `error` (`discovery/readme.md` "Unresolved
identifiers"); unknown properties and unknown codes are tolerated. A response body never goes
into an error beyond the sanitised `message`. The library never panics on server input; the
fuzz tests in 4.1 hold it to that.

Discovery:

```go
func (c *Client) Discover(ctx, t tei.TEI) ([]DiscoveryResult, error)
func (c *Client) DiscoverPURL(ctx, base APIBase, purl string) ([]DiscoveryResult, error)
type DiscoveryResult struct {
    ProductReleaseUUID uuid.UUID
    Servers []teaapi.ServerInfo   // as received, validated
    DiscoveredAt APIBase          // the base that answered /discovery (informational)
}
func (c *Client) SelectServer(r DiscoveryResult) (APIBase, error) // same rules as well-known
type NotResolvedError struct{ Base APIBase; Code ErrorCode; RawCode string } // TEA 404: stop, no failover
type DiscoveryError struct{ Stage Stage; Attempts []Attempt } // StageWellKnown, StageAPIDiscovery
type Attempt struct{ Stage Stage; URL string /* no query */; Reason string; Err error }
func (e *DiscoveryError) Unwrap() []error // each Attempt.Err, so errors.Is(err, ErrTimeout) etc. work
```

Steps of `Discover`:

1. Well-known URL: `https://<lowercased domain>/.well-known/tea`, or the
   `WithWellKnownLocation` base for that domain. GET with `Accept: application/json`, bounded
   by `WellKnownBytes`.
2. Validate the document against `tea-well-known.schema.json` by hand: `schemaVersion` is the
   integer 1; `endpoints` non-empty; each `url` matches the schema's pattern
   `^https?://[^/?#@]+(/[^?#\s]*[^/?#\s])?$` and passes D4; each `versions` non-empty and every
   item matches the SemVer pattern; `priority`, when present, a number in [0, 1]. Unknown
   properties are ignored (D8). A document failing any of these is a failed attempt.
3. Candidates: endpoints with at least one version in `SupportedVersions` (build metadata
   ignored); ordered by greatest mutually supported version, then `priority` (absent = 1)
   descending, then document order. None: `*DiscoveryError` naming the stage, no request made.
4. For each candidate in order: GET
   `<url>/v<version as advertised>/discovery?tei=<url.QueryEscape(t.String())>` (query
   encoding applied once; `+` in a version is a literal path character). Bounded by
   `DiscoveryBytes`.
5. Classify: `200` with a valid body (JSON array, at least one item and at most
   `Limits.DiscoveryResults` items, each item's `productReleaseUuid` decoding as a uuid
   (3.5's rule), `servers` non-empty, each `rootUrl` matching the `server-info` pattern and
   each version the SemVer pattern) returns the results in array order. A TEA `404` returns
   `*NotResolvedError` at once, no failover. `400`, `401`, `403` return `*APIError` at once,
   no failover. DNS or connection failure, TLS failure, the attempt's `AttemptTimeout`
   expiring (error wraps `ErrTimeout`; the caller's own context ending is not a failed
   attempt, it ends `Discover` with `ctx.Err()`), `5xx`, a non-TEA `404`, an unusable or
   invalid body, a body over `DiscoveryBytes` (`ErrResponseTooLarge`) or with more than
   `DiscoveryResults` items (`ErrTooManyResults`; refused whole, never truncated): a failed
   attempt; move to the next candidate. The same classification and `AttemptTimeout` apply
   to the well-known fetch of step 1 (a body over `WellKnownBytes` is a failed attempt).
6. Attempt bound: `WithDiscoveryAttempts` (default 4 attempts in total across both stages);
   the well-known fetch itself is retried within the bound on failed attempts. Backoff before
   each retry: 250 ms doubling, capped at 4 s, interrupted by the context. Exhausted:
   `*DiscoveryError` with the stage and each attempt's URL (without query) and reason.

`SelectServer` applies step 3's ordering to `Servers` and D4 to the chosen `rootUrl`; an
`http` root is usable only if it is under a configured insecure base (D4's definition). The `/discovery`
base is not reused unless it is listed in `servers` ("Discovery response"). No DNS
pre-resolution step (the CLI's `resolveDNS`): the HTTPS request resolves, and a DNS failure is
a failed attempt.

### 3.7 Authentication (`auth.go`)

Anonymous by default: no `Authorization` header is ever sent without a configured credential.

- `WithBearerToken(root, token)`: the transport adds `Authorization: Bearer <token>` to
  requests under `root` (D5). A `401` is returned as `*APIError` with the challenge.
- **The token request** (one internal function, used by `API.RequestToken(ctx, keyID,
  secret)` and by `WithAPIKey`'s recovery): `POST <API base>/token`, i.e.
  `<root>/v<version>/token` of the `API` it is called on, with `grant_type=client_credentials`
  (form-encoded) and `Authorization: Basic base64(formurlencode(id) ":" formurlencode(secret))`.
  The request carries `kindToken`, so it **follows no redirects** (3.4): any `3xx` answer is
  returned as `*UnexpectedResponseError` (`Reason` "redirect from /token refused", status
  kept, `Location` not echoed), and no request reaches the `Location`. The Basic header
  therefore reaches exactly one URL, the one built from the `API`'s own base. For
  `RequestToken` that base is whatever the caller built the `API` from (the caller passes raw
  credentials and chooses the base; the base must still pass D4). The Basic header is set on
  the request by the token code, and the auth transport leaves a `kindToken` request as it is
  (D5). For `WithAPIKey` the base is additionally required to be under
  the configured root, else `ErrInvalidArgument` and no request. Bounded by `CallTimeout`.
- `WithAPIKey(root, id, secret)`: no token up front (clients do not probe `/token`). On a `401`
  from a request under `root` whose `WWW-Authenticate` is a Bearer challenge, the client makes
  the token request above, reads the
  `token-response` (bounded by `ErrorBodyBytes`; `token_type` must be `Bearer`;
  `access_token` non-empty), caches it with its `expires_in` (if given, minus 30 s), and
  retries the original request once. A `401` with `error="invalid_token"` on a request that
  carried a cached token drops it, obtains one fresh token and retries once; never more than
  one recovery per request. A `/token` `404` is `ErrNotImplemented`; a token error
  (`token-error-response`) is `*TokenError{Code}` without the description echoed beyond 200
  sanitised bytes. A `403` never triggers recovery. Requests with a body are retried only for
  `GET` (all resource operations are `GET`).
- Neither credential is sent to a well-known location, to `/discovery` on a base other than
  `root`, to an external artifact URL, or across a redirect off `root` (D5); the Basic
  credentials are sent across no redirect at all.

### 3.8 Download and checksum verification (`download.go`, `checksum.go`)

```go
type ArtifactRef struct{ UUID uuid.UUID; Version int }
type DownloadOptions struct{ MaxBytes int64; AllowSHA1Only bool }
type DownloadResult struct{ Bytes int64; MediaType string; Source DownloadSource; Verified []teaapi.ChecksumType }
func (a *API) DownloadArtifact(ctx, ref ArtifactRef, f teaapi.ArtifactFormat, w io.Writer, o DownloadOptions) (DownloadResult, error)
func (a *API) DownloadArtifactToFile(ctx, ref ArtifactRef, f teaapi.ArtifactFormat, path string, o DownloadOptions) (DownloadResult, error)
func (a *API) DownloadLatestArtifact(ctx, id uuid.UUID, mediaType string, w io.Writer, o DownloadOptions) (DownloadResult, error)
func (a *API) DownloadSignature(ctx, ref ArtifactRef, f teaapi.ArtifactFormat, w io.Writer, o DownloadOptions) (int64, error)
func VerifyChecksums(r io.Reader, sums []teaapi.Checksum, o DownloadOptions) ([]teaapi.ChecksumType, error)
type ChecksumError struct{ Alg teaapi.ChecksumType; Expected, Actual string }  // wraps ErrChecksumMismatch
var ErrChecksumMismatch, ErrNoUsableChecksum, ErrUnsupportedContentEncoding error
```

1. **Source.** If `f.Url` is set: GET that URL (`DownloadSourceExternal`), which must pass D4;
   no credential is attached (D5). Else GET `<base>/artifact/<uuid>/<version>/download?mediaType=<f.MediaType>`
   (`DownloadSourceServer`), the versioned endpoint, so the bytes are those of the revision whose
   checksums the caller holds. Redirects (`302`) follow 3.4's rules. `Content-Disposition`
   is ignored: the library never derives a file name or path from a server.
2. **Status.** `200` only; `406` maps to `ErrNoAcceptableFormat`, `404` to the 3.6 mapping;
   anything else to `*APIError`/`*UnexpectedResponseError`. A `Content-Length` above the limit
   is refused before reading.
3. **Content coding.** The transport's transparent gzip handling is kept (it decodes only what
   it asked for, giving the bytes before content coding, which is what checksums cover). Any
   other `Content-Encoding` than empty or `identity` that the transport did not remove is
   `ErrUnsupportedContentEncoding`. No other transformation of the bytes.
4. **Hashing while streaming.** Bytes go through `io.MultiWriter(w, hashers...)` with a reader
   capped at `MaxBytes` (default `Limits.DownloadBytes`); past the cap: `ErrResponseTooLarge`.
   The request is `kindDownload` (`kindSignature` in step 7), so `IdleTimeout` applies (3.4):
   no response headers, or no further body byte, for that long fails the download with an
   error wrapping `ErrTimeout`; a slow but moving download is never cut off. There is no
   whole-download time limit beyond the caller's context.
5. **Which checksums.** Supported: SHA-256/384/512 and SHA3-256/384/512 (stdlib), BLAKE2b-256/384/512
   (`x/crypto/blake2b`), SHA-1 (weak). BLAKE3 has no implementation in the standard library or
   `x/crypto` and is unsupported (see 5). Each `algValue` must be lowercase hex of exactly the
   algorithm's length (40/64/96/128) or the metadata is refused (`*UnexpectedResponseError`).
   Every supported checksum listed must match; a single mismatch is `*ChecksumError`, whatever
   else matched. At least one supported non-SHA-1 checksum must be present, else
   `ErrNoUsableChecksum` (unless `AllowSHA1Only`, where one matching SHA-1 suffices). A format
   with no checksums (allowed by the spec for hosted content) is `ErrNoUsableChecksum`: the
   download helper fails closed; a caller who wants the bytes regardless uses `Generated()`.
   `Repr-Digest` is not used (the spec says it does not replace the published checksums).
6. **Failure handling.** `DownloadArtifact` has already written bytes to `w` when it fails; its
   doc comment says the caller must discard them. `DownloadArtifactToFile` writes to a temporary
   file created with `os.CreateTemp` in the target's directory, mode 0600, verifies, `Sync`s,
   then `os.Rename`s over `path`; on any error the temporary file is removed and `path` is not
   touched.
7. **Signatures.** `DownloadSignature` uses `f.SignatureUrl` when set, else the versioned
   signature endpoint; same URL and auth rules; capped at `SignatureBytes`; `404
   SIGNATURE_NOT_FOUND` maps to `ErrSignatureNotFound`. The bytes are not verified (no
   technology defined).

### 3.9 Flow helper (`flow.go`)

```go
type FlowOptions struct {
    Select func(ctx context.Context, cands []Candidate) (int, error) // nil: ambiguity is an error
    SkipCLE bool
}
type Candidate struct{ Result DiscoveryResult; API *API; Release *teaapi.ProductReleaseWithCollection }
type AmbiguousError struct{ Results []DiscoveryResult }
type FlowResult struct {
    API *API
    Release *teaapi.ProductReleaseWithCollection
    ReleaseCLE *teaapi.Cle                 // nil when NOT_IMPLEMENTED or SkipCLE
    Components []FlowComponent             // in the release's order
}
type FlowComponent struct {
    Ref teaapi.ComponentRef
    Pinned bool                            // Ref.Release was set
    Component *teaapi.Component            // fetched when not pinned
    Release *teaapi.ComponentReleaseWithCollection
    CLE *teaapi.Cle
    Err error                              // this component failed; the others continue
}
func (c *Client) FullFlow(ctx, t tei.TEI, o FlowOptions) (*FlowResult, error)
func (c *Client) Candidates(ctx, results []DiscoveryResult) ([]Candidate, error)
func (c *Client) FlowFromRelease(ctx, a *API, release uuid.UUID, o FlowOptions) (*FlowResult, error)
```

`FullFlow`: `Discover`; one result: `SelectServer`, `FlowFromRelease`. Several results: with
`Select` nil return `*AmbiguousError` (the CLI later loads `Candidates` and asks the user, as
`disambiguateProductRelease` does today, outside the library); otherwise call
`Candidates` (each result's selected server and its product release) and `Select`, and
continue with the chosen one. `Candidates` refuses more than `Limits.DiscoveryResults`
results with `ErrTooManyResults` before making any request (a caller may pass results it
did not get from `Discover`); otherwise it makes one product-release request per result, so
one `FullFlow` makes at most `DiscoveryResults` such requests. A `Select` index out of range
is `ErrInvalidArgument`. `FlowFromRelease`: get the product release (its
`latestCollection` is part of the response); its CLE unless skipped (`ErrNotImplemented` and
`ErrObjectUnknown` on CLE give `nil`, other errors are returned); for each component ref, up to
`Limits.FlowComponents` (more refs than that: `ErrTooManyComponents`, no silent truncation): pinned, get that component release; unpinned, get the component and
`ListComponentReleases(sortField=createdDate, sortOrder=desc, pageSize=1)` and take the first
(the newest; the CLI's `releases[0]` of the ascending default is the oldest, which this
corrects); then its CLE. A per-component failure is recorded in `FlowComponent.Err` and the
walk continues (the CLI prints warnings and goes on); a failure on the product release is
returned. The flow does not download artifacts; the result carries every collection's
artifacts and formats, which is all `full_tea_flow` prints.

### 3.10 What the later CLI switch needs (and gets here)

| CLI today | library |
| --- | --- |
| `urn:tei:` input | `tei.ParseCompat` + deprecation warning on `legacy` |
| `--usehttp`, `--useport` | with `O = "http://<domain>:<useport>"`: `WithWellKnownLocation(domain, O)` + `WithInsecureHTTPBase(O)` (below) |
| `--useport` without `--usehttp` | `WithWellKnownLocation(domain, "https://<domain>:<useport>")`, no allowance |
| `resolveTEI` / `discovery` | `Client.Discover` |
| stdin disambiguation | `*AmbiguousError` + `Candidates`, prompt stays in the CLI |
| `full_tea_flow` | `Client.FullFlow`, printing stays in the CLI |
| `getCle` / `printCLE` | `Get...CLE` (typed `teaapi.Cle`) |
| `debug=true` prints of the URLs it requests | `WithTrace`, the CLI prints each `TraceRequest`'s `URL` (and the TEI itself) |
| `http.Client{Timeout: 30s}` per call | the defaults of `CallTimeout` (30 s), `AttemptTimeout`, `IdleTimeout` |

**`--usehttp` under D4.** `O` is the origin the CLI already builds for the well-known fetch.
Because an allowance covers what is under it, `O` also covers any endpoint `url` and any
`servers[].rootUrl` on that same origin, whatever their path: a local ReARM that serves
`http://localhost:8086/.well-known/tea` listing `http://localhost:8086/tea` works with
`--usehttp --useport 8086` exactly as today (test 9b, and 27b for the `servers[]` root). Two
behaviours are deliberately stricter than today, and both are what "safe by default" asks
for: an `http` URL on any other host or port (a well-known document listing
`http://other:9000`) is refused, where today the CLI would follow it; and without
`--usehttp` nothing is ever fetched over http. If a user needs a second plain-http origin,
the CLI can pass it as one more `WithInsecureHTTPBase` (a later flag, the CLI task's
decision). No new CLI flag is needed for today's behaviour.

The CLI's `swid` and `gs1` legacy types have no v1.0.0 registry entry; `ParseLegacyURN`
refuses them with a message the CLI can show.

### 3.11 README, examples, CI

- `README.md`: what the module is; the two packages; install; the example below; options and
  limits and time bounds; the URL and credential policy (D4 with the "under" rule, D5), and
  that a caller transport passed with `WithHTTPClient` sits below the auth transport and so
  sees `Authorization` headers (use `WithTrace` for debug output, not a header-dumping
  transport); the legacy URN note; how to regenerate (`go generate ./...`, never edit
  `teaapi/tea.gen.go`; generator bump in `tools.mod`); how to run unit, integration (4.2) and
  live (4.3) tests; **Versioning** and **Licence** (below). Its code is copied from
  `example_test.go`, which compiles in `go test`:

```go
c, err := tea.New()
t, err := tei.Parse("tei://demo.rearmhq.com/uuid/b9f115fb-6688-4a3f-a262-3acc04e1fc9e")
res, err := c.FullFlow(ctx, t, tea.FlowOptions{})
fmt.Println(*res.Release.ProductName, res.Release.Version)
```

  (examples have no `// Output:` line, so `go test` compiles but does not run them: no network.)
- **Licence of `spec/` and of what is generated from it.** tea-client-go is MIT (its existing
  `LICENSE`). `spec/openapi.yaml` and `spec/tea-well-known.schema.json` are Apache-2.0
  (3.0). Apache-2.0 section 4(a) asks for a copy of the licence with redistributed files, so
  `spec/LICENSE` is a byte copy of the upstream `LICENSE` at the pinned commit (blob
  `0808751c23d827d483dc3673d6e3f49514e49d1f`, checked by test 1); there is no upstream
  `NOTICE` file at that commit, so none is carried (4(d) does not apply). `spec/SPEC_PIN.md`
  names the licence of `spec/` and the blob of `spec/LICENSE`. `teaapi/tea.gen.go` carries the
  spec's descriptions as doc comments, so it is a derivative of Apache-2.0 material:
  `teaapi/doc.go` (hand-written; the generated file is not edited) says that the package is
  generated from `spec/openapi.yaml` of CycloneDX/transparency-exchange-api at the pinned
  commit, that its comments are taken from that document, licensed under Apache-2.0
  (`spec/LICENSE`), and that the generator transformed it (4(b)). README's **Licence**
  section says: the module is MIT; `spec/` is Apache-2.0; `teaapi` includes text from the
  Apache-2.0 spec. **Nothing is copied from rearm-client-go**: its layout, option pattern,
  `generate.go` idea and CI workflow are models only, and the coder writes each file anew. If
  a coder does copy a file or a substantial part of one verbatim (rearm-client-go is MIT,
  "Copyright (c) 2019-2026 Reliza Incorporated"), the copied file keeps that copyright and
  permission notice in a header comment and the PR says so. Referencing the public
  `relizaio/rearm-actions` by commit SHA in a workflow is use, not copying.
- **Versioning.** This task tags nothing; tags are the repository maintainers' decision. The
  policy README states, which the later CLI switch relies on: tags are `v0.x.y` until the CLI
  has switched and the API has settled. Every exported identifier, `teaapi`'s included, is
  public API ("The Go client treats every server as untrusted"). In `v0`, a change that
  alters or removes an exported identifier (a spec re-pin or a generator bump that changes
  `teaapi`'s output, or a change to `tea`) is a new minor version (`v0.N+1.0`) and README's
  "Changes" list names what broke; a regenerate that changes no exported identifier, and any
  fix that keeps the API, is a patch. From `v1.0.0`, such a breaking change needs a new major
  version (module path `/v2`), so a re-pin to a TEA version with an incompatible schema is a
  major release; an additive regenerate is a minor. The coder adds the "Changes" list empty,
  with this policy above it.
- `.github/workflows/ci.yaml`, modelled on rearm-client-go's `rearm-submit.yaml` (top-level
  `permissions: {}`, each job asks for `contents: read`, actions pinned by full commit SHA,
  `persist-credentials: false`), on `push` and `pull_request`:
  - `test`: `actions/setup-go` with `go-version-file: go.mod`;
    `test -z "$(gofmt -l .)"`; `go vet ./...`; `go test ./...`; generator drift:
    `go generate ./... && git diff --exit-code`; spec pin: the 4.1 test that hashes
    `spec/openapi.yaml` runs in `go test`; ASCII check of the coding principles over
    `*.go *.yaml *.md` excluding `spec/` (copied upstream text) and `teaapi/tea.gen.go`
    (generated from it).
  - `integration`: also checks out `oolong-project/oolong` at ref `2026-11-oolong` into
    `oolong/` (`actions/checkout` with `repository:`), prints its `git rev-parse HEAD`,
    `actions/setup-node` with Node 24, `npm ci` and `npm run build` in
    `oolong/oolong/oolong-server`, then
    `OOLONG_SERVER_DIR=$GITHUB_WORKSPACE/oolong/oolong/oolong-server TEA_INTEGRATION_REQUIRED=1 go test -tags integration -run Integration ./...`.
  - `live`: only on `workflow_dispatch`, `TEA_LIVE=1 go test -tags live -run Live ./...`.
    Never on push or PR (it depends on an external service and data).
  - `record-release`: the ReARM release job of rearm-client-go
    (`relizaio/rearm-actions` setup-cli / initialize / sbom-sign-scan / finalize at the same
    pinned SHA, `rearm_auth: github-oidc`, `rearm_api_url: https://demo.rearmhq.com`,
    `create_component: 'true'`, permissions `contents: read`, `id-token: write`), `needs:
    [test, integration]`, runs only on push to `2026-11-oolong` or `main`, and only when the
    repository variable `REARM_RECORD_RELEASE` is `'true'`. No secret is added. Until the
    operator creates the OIDC trust rule on demo and sets the variable, the job is skipped
    (not failed). This is the repository's first reporting to ReARM.

## 4. Tests the coder must add

All Go tests: `gofmt -l .` empty, `go vet ./...`, `go test ./...` offline. Unit tests use
`httptest.NewServer` / `httptest.NewTLSServer` (with the test server's client, so TLS is real)
and made-up data (`example.com`, `example.invalid`, generated uuids). Timestamps in fixtures are
fixed. Each topic has its own file (`tei/tei_test.go`, `discovery_test.go`, ...).

### 4.1 Unit tests

**Spec pin and generation (`spec_test.go`)**
1. The git blob id of `spec/openapi.yaml` (`sha1("blob <len>\0" + bytes)`) is
   `c96baa3e102c195895643fab81ae468ee54b8182`; of `spec/tea-well-known.schema.json`,
   `b0804f1ae79c8ec72adea6d5c12e8f75f6036729`; of `spec/LICENSE`,
   `0808751c23d827d483dc3673d6e3f49514e49d1f`. `go.mod` has no `tool` line (the generator is
   in `tools.mod`), read and checked as text.
2. Every one of the 28 operation ids of 3.0 has a method on `teaapi.ClientInterface`
   (compile-time assertions in a `_test.go`), and the `tea.API` wrapper set of 3.5 exists.

**TEI (`tei/tei_test.go`)**, table-driven:
3. Accepted, `String()` byte-equal to input: each registry example of 3.0
   (`uuid`, `purl` `cGtnOm1hdmVuL2NvbW1vbnMtaW8vY29tbW9ucy1pb0AyLjIyLjA`, `hash`
   `SHA-256:fd44...`, `eanupc`, `gtin`, `asin`, `udi`), the demo TEI, `tei://localhost/uuid/...`,
   an unknown type `tei://example.com/foo2/x` (`Known()` false), `Example.com` domain (kept),
   uppercase uuid id (kept, not rewritten), id with `%3A` (kept).
4. Refused with `ErrSyntax` (and the rule named): `TEI://...`, `tei:/x`, empty domain, port
   `tei://example.com:443/uuid/x`, userinfo, IPv4 literal, `[::1]`, label of 64 chars, domain
   of 254, trailing dot, `_` in domain, non-ASCII domain, type `Uuid`, type `1abc`, missing id,
   fourth segment, id with `%2F` and `%2f`, `%G1`, trailing `%`, `?`, `#`, space, control
   char, 2049 bytes. The error message never contains more than 64 input bytes.
5. `urn:tei:uuid:example.com:x` via `Parse`: `ErrLegacyForm`.
6. `ParseLegacyURN`: the three examples in `cmd/tea.go`'s help (`uuid` kept;
   `purl:cyclonedx.org:pkg:pypi/cyclonedx-python-lib@8.4.0` gives
   `tei://cyclonedx.org/purl/<base64url unpadded of "pkg:pypi/cyclonedx-python-lib@8.4.0">` and
   `PURL()` returns the original; `hash:localhost:SHA256:<hex>` gives `.../hash/SHA-256:<hex>`);
   `swid` and `gs1` give `ErrLegacyTypeUnsupported`; `hash` with `MD5:` refused;
   `ParseCompat` reports `legacy` correctly for both forms.
7. Fuzz `FuzzParse` and `FuzzParseLegacyURN`: no panic; any accepted value round-trips
   `String()` and re-parses equal.

**Options and URL policy (`client_test.go`, `transport_test.go`)**
8. `New` refuses: `WithInsecureHTTPBase("https://...")`, a relative URL, one with userinfo,
   query, fragment, a trailing `/` or a dot segment; `WithBearerToken` on an http root not
   under an allowance.
9. Allowance covers what is under its base (D4), checked on API requests (`API` built with
   `ParseAPIBase` directly, no discovery):
   - **9a** With `WithInsecureHTTPBase("http://127.0.0.1:PORT/tea")`: requests to
     `http://127.0.0.1:PORT/tea/v1.0.0/product/<uuid>` and to the base itself are allowed;
     `http://127.0.0.1:PORT/teapot/v1.0.0/...`, `http://127.0.0.1:PORT/other`,
     `http://127.0.0.1:OTHER/tea/...` and `http://localhost:PORT/tea/...` are refused with
     `ErrInsecureURL` and no request reaches the server (handler counts). A root
     `http://127.0.0.1:PORT/tea/../admin`, `.../tea/%2e%2e/admin` or
     `https://example.com/a/./b` given to `ParseAPIBase` is refused with `*URLPolicyError`
     wrapping `ErrInsecureURL` (Go cleans dot segments out of redirect `Location`s itself, so
     these reach the client only as learned bases or artifact URLs: case 23 covers the
     well-known form). With the origin
     form `WithInsecureHTTPBase("http://127.0.0.1:PORT")`, `/anything/v1.0.0/...` is allowed
     and `http://127.0.0.1:OTHER/...` refused. `http://H` and `http://H:80` compare equal (unit
     test of the `under` helper, no request).
   - **9b** (discovery): the same rule on learned bases is cases 21b and 27b.
10. Redirects: https to http refused; to userinfo URL refused; 6th redirect gives
    `ErrRedirectLimit`; an http redirect target is allowed only when under an allowance.
11. Limits, by kind:
    - **11a** A typed JSON response one byte over `JSONBytes` gives `ErrResponseTooLarge`
      (with and without `Content-Length`); a gzip body that expands over the limit gives the
      same; a non-2xx body over `ErrorBodyBytes` is cut and still classified.
    - **11b** (discovery) Well-known over `WellKnownBytes` and discovery over
      `DiscoveryBytes` are failed attempts whose `Attempt.Err` is `ErrResponseTooLarge`
      (`errors.Is` through `*DiscoveryError`).
    - **11c** `Generated()` is bounded: `Generated().GetProductWithResponse` against a body
      over `JSONBytes` returns an error wrapping `ErrResponseTooLarge`;
      `Generated().DownloadArtifactByVersionWithResponse` with a body over a small
      `DownloadBytes` (set to 1 KiB in the test) likewise, and the same body under the limit
      succeeds; the request kind is taken from the path (a body of 2 KiB on a JSON path with
      `JSONBytes` 1 KiB and `DownloadBytes` 4 KiB fails, on a download path succeeds).
    - **11d** Time bounds on JSON calls (`CallTimeout` set to 200 ms): a server that sends
      headers and then stalls the body, and one that never sends headers, both make
      `GetProduct` return an error wrapping `ErrTimeout` within 1 s; the same through
      `Generated().GetProductWithResponse`; with a caller `http.Client` (no transport
      timeouts) the result is the same; with a caller context whose deadline is earlier
      (100 ms), the error is `context.DeadlineExceeded`, not `ErrTimeout`; with
      `CallTimeout` negative and a caller deadline of 300 ms, the call ends at the caller's
      deadline. Idle bound in the doer (`IdleTimeout` 200 ms):
      `Generated().DownloadArtifactByVersionWithResponse` against a body that stalls after
      1 KiB fails with an error wrapping `ErrTimeout` within 1 s, and one that sends a byte
      every 100 ms for 1.5 s succeeds (the wrapped downloads are case 31b).

**Credentials (`auth_test.go`)**
12. Anonymous: no `Authorization` header on any request.
13. Bearer bound to root (`WithBearerToken`, root `http://localhost:PORT/tea`). In 13 and 14
    every test server's origin is allowed with `WithInsecureHTTPBase`, so D4 lets the request
    through and the header decision is what the test observes, and the `http.Client`'s dialer
    maps `*.localhost` to `127.0.0.1` (the shape of the 3.0 prototype; `httptest`'s TLS
    certificate does not cover subdomains):
    - **13a** Sent to `<root>/v1.0.0/...`; not sent to a sibling path outside the root
      (`/teapot/...`), to another port, to a subdomain (`sub.localhost:PORT2`), or to a `302`
      or `307` target on another port or a subdomain (the second
      server sees no `Authorization`); an `Authorization` header set by a
      `Generated()` request editor on a request outside the root is removed (the server sees
      none), and on a request under the root is replaced by the configured bearer.
    - **13b** (discovery) Not sent to the well-known location or to `/discovery` on a base
      other than the root, nor to a `servers[]` root on another host.
    - **13c** (download) Not sent to an external artifact `url` (case 30).
14. Token request and API key:
    - **14a** `API.RequestToken(ctx, "a:b@c", "p:q%")`: one `POST <base>/token`, body
      `grant_type=client_credentials`, Basic credentials form-encoded before joining; a
      `token-response` is returned; `token_type` other than `Bearer` refused; `/token` `404`
      gives `ErrNotImplemented`. Redirects: `/token` answering `307` to the same host on
      another port, and `302` to a subdomain test server: each gives
      `*UnexpectedResponseError` and the second server receives **no request at all** (its
      handler count is 0, so no `Authorization` reached it). With a bearer also configured for
      the same root, the token request carries Basic, not the bearer. Neither secret appears
      in any returned error string (`strings.Contains`).
    - **14b** `WithAPIKey`: first request `401` with `WWW-Authenticate: Bearer realm="x"`
      triggers exactly one `POST /token`, then one retry with the token; `invalid_token` on a
      cached token gives one refresh and one retry, a second `invalid_token` is returned as
      `ErrUnauthorized` (two token requests total, no loop); `403` gives no token request;
      the automatic token request answered `307` to another port fails the original call and
      the redirect target receives no request; neither secret nor token appears in any
      returned error string or `TraceEvent`.

**Errors (`errors_test.go`)**
15. Mapping: each of the six codes with `application/json; charset=utf-8` to its sentinel via
    `errors.Is`; an unknown code `FUTURE_CODE` gives `ErrorCodeUnknown` with `RawCode`; extra
    properties ignored; `{"message":"Not Found"}`, a JSON array, `text/html` and an empty body on
    `404` give `TEA: false`; `401` without body keeps the parsed challenge; `5xx` gives
    `ErrServer`; a `message` of 10 KiB with control characters is cut to 200 bytes without them.
16. Unexpected JSON: for every typed call, a 200 with invalid JSON, `null`, a wrong type, and a
    missing body gives `*UnexpectedResponseError` and no panic (one table over all methods).
    Uuids (3.5): a product whose `uuid` is `"not-a-uuid"` gives `*UnexpectedResponseError`; one
    whose `uuid` is uppercase `ABCDEF00-2222-4222-8222-222222222222` decodes, and a following
    `ListProductReleases` with that value requests the lowercase path.
17. Fuzz `FuzzDecodeDiscovery` and `FuzzDecodeWellKnown`: no panic.

**Typed calls and pagination (`api_test.go`, `pagination_test.go`)**
18. Each method sends the path, method and query of the spec (pageSize, pageToken, sortField,
    sortOrder, idType, idValue as given, omitted when zero); a uuid path parameter is lowercase.
19. Local refusals before any request: pageSize 101 and -1, an invalid sort field, `IDValue`
    without `IDType`.
20. Iterator: three pages followed to the end, the token passed back verbatim (including
    characters that need query encoding); `hasNext: true` without token gives
    `*UnexpectedResponseError`; a server that always has a next page stops with `ErrPageLimit`
    after `Limits.Pages` pages; context cancel stops it.

**Discovery (`discovery_test.go`)**, with `WithWellKnownLocation` to TLS test servers:
21. Request form: the discovery URL is `<url>/v1.0.0/discovery?tei=` + query-encoded TEI, and
    the server sees the TEI decoded once equal to the input; a version `1.0.0+build.5` is used
    verbatim in the path with a literal `+`.
    **21b** The local-ReARM shape of `--usehttp` (3.10), on a plain-http `httptest` server
    with origin `O`: options `WithWellKnownLocation("localhost", O)` +
    `WithInsecureHTTPBase(O)` only; `O/.well-known/tea` lists `O + "/tea"`; `/tea/v1.0.0/discovery`
    answers one result whose `servers[0].rootUrl` is `O + "/tea"`. `Discover` succeeds,
    `SelectServer` gives `O/tea` and `1.0.0`, and `GetProductRelease` on that base requests
    `/tea/v1.0.0/productRelease/<uuid>`. The same document listing `http://127.0.0.1:OTHER/tea`
    instead is refused (no request to `OTHER`; no candidate left: `*DiscoveryError`).
22. Selection: endpoints `[{v 0.2.0-beta.2}, {1.0.0, priority 0.5}, {1.0.0, no priority}]`
    calls the third first; `["1.0.0-rc.1"]` only: no candidate, `*DiscoveryError` stage
    well-known, no discovery request; ties keep document order.
23. Well-known validation, each a failed attempt: `schemaVersion` 2 or `"1"`, empty
    `endpoints`, url with query, userinfo, trailing slash, a dot segment (`https://h/a/../b`) or
    `http` (not allowed), version `v1.0.0`, priority 1.5, not JSON. Unknown extra property:
    accepted.
24. Failover: first endpoint answers `503`, non-TEA `404`, invalid JSON, `[]`, or a result whose
    `productReleaseUuid` is `"not-a-uuid"` or the 35-character
    `"22222222-2222-4222-8222-22222222222"` (both fail to decode, 3.5), or an http rootUrl not
    under an allowance; the second endpoint is tried and its result returned. A result with an
    uppercase `productReleaseUuid` is accepted and `ProductReleaseUUID.String()` is lowercase.
    A TLS failure (a test server with an untrusted certificate) is a failed attempt.
25. No failover: TEA `404 OBJECT_UNKNOWN` gives `*NotResolvedError` with the code and no request to
    the second endpoint; same for `401` and `403` (as `*APIError`) and `400`.
26. Bound and backoff: four failing endpoints with attempts set to 3 make exactly 3 requests and
    return `*DiscoveryError` listing them; a cancelled context during backoff returns
    `context.Canceled` promptly.
    **26b** Attempt timeout (`AttemptTimeout` 200 ms, first backoff 10 ms): a well-known server
    that sends headers and then stalls the body on its first request is a failed attempt whose
    `Attempt.Err` wraps `ErrTimeout`; the well-known retry within the attempt bound (3.6 step
    6) is answered normally and `Discover` succeeds, within 1 s, with two well-known requests
    seen; a first `/discovery`
    endpoint that stalls the same way is a failed attempt and the second endpoint's result is
    returned within 1 s; the same with a caller `http.Client` that has no timeouts. A caller
    context that ends during a stalled attempt ends `Discover` with `ctx.Err()`, no further
    attempt.
    **26c** Results cap (`DiscoveryResults` 2): a first endpoint answering 3 valid results is a
    failed attempt (`ErrTooManyResults`) and the second endpoint's 2 results are returned;
    with one endpoint only and attempts exhausted, `errors.Is(err, ErrTooManyResults)` holds on
    the `*DiscoveryError`. A default-limit answer of 33 results is refused the same way.
27. Results: two results are returned in array order; `SelectServer` picks per 22's rules and
    refuses an http root not under an allowance; `DiscoverPURL` sends `purl=` and not `tei=`.
    **27b** With `WithInsecureHTTPBase("http://127.0.0.1:PORT")`, `SelectServer` accepts a
    `servers[]` root `http://127.0.0.1:PORT/tea` (under the origin) and refuses
    `http://127.0.0.1:PORT2/tea` and `http://localhost:PORT/tea`.

**Download and checksums (`download_test.go`, `checksum_test.go`)**
28. Known-answer vectors for every supported algorithm (bytes `"abc"` and the 137-byte fixture
    JSON of 3.0); BLAKE3-only gives `ErrNoUsableChecksum`.
29. Rules: one of two listed checksums wrong gives `*ChecksumError` naming it; SHA-1 only gives
    `ErrNoUsableChecksum`, and passes with `AllowSHA1Only`; no checksums gives
    `ErrNoUsableChecksum`; `algValue` uppercase or wrong length refused; unknown `algType`
    ignored when another supported one matches.
30. Source: a format with `url` is fetched from that URL only (the API server sees no request) and
    without `Authorization` even when a bearer is configured; without `url`, the versioned endpoint
    with `mediaType` is called; a `302` from the API to another origin is followed without the
    header; an `http` external URL is refused.
31. Bounds: `Content-Length` over `MaxBytes` refused before the body is read; a chunked body over it
    stops with `ErrResponseTooLarge`; gzip `Content-Encoding` (transport-negotiated) verifies against
    the uncompressed bytes; `Content-Encoding: br` gives `ErrUnsupportedContentEncoding`.
    **31b** Idle timeout (`IdleTimeout` 200 ms): a download that sends 1 KiB and then stops
    fails with an error wrapping `ErrTimeout` within 1 s, and `DownloadArtifactToFile` leaves
    no file; a download that sends one byte every 100 ms for 1.5 s (longer than
    `IdleTimeout` in total) completes and verifies; a server that never sends headers fails
    within 1 s; the same for `DownloadSignature`.
32. `DownloadArtifactToFile`: on a checksum mismatch, a pre-existing target file keeps its content
    and no temp file is left in the directory; on success the file has the bytes and mode 0600; a
    `Content-Disposition: attachment; filename="../../evil"` has no effect on where bytes go.
33. Signature: `SignatureUrl` used when set; else the versioned signature endpoint; `404
    SIGNATURE_NOT_FOUND` gives `ErrSignatureNotFound`; over `SignatureBytes` refused.
34. `406 NO_ACCEPTABLE_FORMAT` gives `ErrNoAcceptableFormat`. `DownloadLatestArtifact` reads
    `/latest` metadata and then downloads that version through the versioned endpoint.

**Flow (`flow_test.go`)**
35. One result: release, CLE, two components (one pinned, one unpinned); the unpinned one lists
    with `sortField=createdDate&sortOrder=desc&pageSize=1` and uses the first result.
36. CLE `404 NOT_IMPLEMENTED` gives `nil` CLE, not an error; a `500` on one component's release is
    recorded in its `Err` and the other component completes; a `404` on the product release fails
    the flow.
37. Two results, `Select` nil: `*AmbiguousError` with both results and no product-release request;
    with a `Select` returning 1, the second candidate's server is used.
38. `Limits.FlowComponents` = 2 with three refs gives `ErrTooManyComponents`, no
    silent truncation.
    **38b** `Candidates` with `DiscoveryResults` 2 and three results gives `ErrTooManyResults`
    and makes no request (handler count 0); with two results it makes exactly two
    product-release requests. A `Select` returning 5 for two candidates gives
    `ErrInvalidArgument`.

**Trace and user agent (`trace_test.go`, `client_test.go`)**
39. `WithTrace`:
    - **39a** One `QueryProducts` with a page token gives a `TraceRequest` (`StageAPI`, `GET`,
      the URL without query) and a `TraceResponse` with the status; a `302` produces a request
      event for the target; a `RequestToken` gives `StageToken` events. No event contains a
      `?`, the page token, the bearer token, the API key id or secret (a configured bearer and
      the `RequestToken` credentials are present in the test and searched for in every field).
    - **39b** (discovery) `Discover` against two endpoints where the first fails gives, in
      order, the well-known request and response, the first discovery request, a
      `TraceAttemptFailed` with the reason, a `TraceBackoff`, then the second discovery
      request and response; no event contains the TEI or a `?`.
40. User agent: by default every request carries `User-Agent: tea-client-go/devel` in the
    module's own tests (3.4, `Main.Version` is `(devel)` there); `WithUserAgent("x/1")`
    replaces it. The version lookup is a function over a `*debug.BuildInfo`, tested with
    built values: a `Deps` entry `v0.3.1` gives `tea-client-go/v0.3.1`, a replaced entry with
    `Replace.Version` `v0.3.2` gives that, `(devel)` and no build info give `devel`.

### 4.2 Integration tests against oolong (`integration_test.go`, `//go:build integration`)

Harness `internal/oolongtest.Start(t) *Server`:
- `OOLONG_BASE_URL` set (e.g. `http://127.0.0.1:3005` from `docker-compose.tea-interop.yml`): use it.
- Else `OOLONG_SERVER_DIR` (an `oolong/oolong-server` directory where `npm ci && npm run build`
  has run): copy `<dir>/../fixtures/tea-1.0.0` into `t.TempDir()` (tests own their data), pick a
  free port on 127.0.0.1, start `node <dir>/dist/main.js` (node from `PATH`, or `OOLONG_NODE`) with
  a clean environment plus `PORT`, `SERVER_HOST=http://127.0.0.1:<port>`, `CONTENT_DIR`,
  `NODE_ENV=production`, `API_VERSION=1.0.0` and `PAGE_TOKEN_SECRET` = 32 random bytes hex made
  per run (never committed); poll `GET /.well-known/tea` until `200` (it is `503` until content
  is ready), 30 s deadline; capture stdout/stderr and print them only on failure; kill the process
  (and wait) in `t.Cleanup`.
- Neither set: `t.Skip` with how to set them, unless `TEA_INTEGRATION_REQUIRED=1`, then fail.

The client under test: `WithInsecureHTTPBase(server URL)`,
`WithWellKnownLocation("localhost", server URL)`. Cases (ids and bytes from
`fixture-reference.md`):

1. Discover `tei://localhost/uuid/22222222-2222-4222-8222-222222222222`: one result,
   `ProductReleaseUUID` that uuid, `SelectServer` gives the server URL and `1.0.0`.
2. Discover an unknown TEI (`.../99999999-9999-4999-8999-999999999999`): `*NotResolvedError`,
   `OBJECT_UNKNOWN`. `DiscoverPURL("pkg:generic/oolong-fixture/widget@1.0.0")`: same release.
3. A base whose path is wrong (`WithWellKnownLocation` pointing at a document listing
   `<server>/nope`, served by a local `httptest` server, allowed): the Nest 404 is a failed
   attempt, not `NotResolved`.
4. FullFlow: release `1.0.0`, product name `Alpha Fixture`, one component pinned to
   `44444444-4444-4444-8444-444444444444`, latest collection version 2 holding artifact
   `55555555-5555-4555-8555-555555555555` version 2 with JSON and XML formats; CLE `nil` (oolong
   answers `NOT_IMPLEMENTED`).
5. Products iterate with page size 2: five products in name order (Alpha, Beta, Delta, Epsilon,
   Gamma); product releases of Alpha by `createdDate` asc: `0.9.0`, `1.0.0`, `1.1.0`;
   `QueryProductReleases(IDType TEI, IDValue the fixture TEI)`: exactly that release; a modified
   page token gives `ErrInvalidPageToken`.
6. Collections by version 1 and 2 for the product release and for the component release
   `44444444-...-444444444444`; latest collection of `22222222-...-222222222223` is the synthetic
   version 1 with no artifacts; unknown uuid gives `ErrObjectUnknown`; every `Get...CLE` gives
   `ErrNotImplemented`; `RequestToken` gives `ErrNotImplemented`.
7. Download revision 2 JSON: 137 bytes, SHA-256 and SHA-1 verified; XML: 57 bytes; JSON signature:
   30 bytes; XML signature and artifact `55555555-...-555555555557` signature:
   `ErrSignatureNotFound`; `DownloadArtifactToFile` into `t.TempDir()`.
8. Revision 1 (external `url` `https://example.invalid/fixture-widget-r1.json`): the test's
   `http.Client` transport answers that host with the fixture JSON bytes in-process and forwards
   `127.0.0.1` normally; the download verifies, the oolong server sees no download request, and
   with a bearer configured for the server the fake host sees no `Authorization`. Nothing leaves
   the machine.
9. A bearer configured for the server is accepted and ignored (oolong ignores Bearer).

### 4.3 Live test against the demo (`live_test.go`, `//go:build live`)

Runs only with the tag and `TEA_LIVE=1` (else `t.Skip`), so offline `go test ./...` never
reaches the network. This is the one departure from "Tests do not reach the network": opt-in,
never in the default run or on PRs, asked for by the task. TEI
`tei://demo.rearmhq.com/uuid/b9f115fb-6688-4a3f-a262-3acc04e1fc9e`, default options (https only,
anonymous). Assertions on structure only:

1. `Discover`: at least one result and at most `DiscoveryResults`; every result has a
   non-nil `ProductReleaseUUID` and at least one server; `SelectServer` on each gives an
   `https` base with version `1.0.0`. Nothing is asserted about which uuid is returned or how
   it relates to the TEI's id (the spec does not tie them; round 2 dropped that assertion).
2. `FullFlow` (with `Select` choosing 0 if several, and recording the candidate it chose; with
   one result, that result): the release's `Uuid` equals the `ProductReleaseUUID` of the
   discovery result it was fetched for (the contract of `GET /productRelease/{uuid}`, not
   data); the release has a non-empty `Version` and `LatestCollection.Version >= 1`; every
   component has `Release != nil` or a typed `Err`; no untyped error.
3. For the first artifact format found in the flow (product release first, then components) that
   has at least one supported checksum and size within a 64 MiB test cap: `DownloadArtifact` to
   `io.Discard` verifies. If none qualifies, the subtest skips with the reason.
4. An unknown product uuid gives `ErrObjectUnknown`.

### 4.4 Suites and checks the coder runs before hand-over

`gofmt -l .` (empty), `go vet ./...`, `go test ./...`, `go test -race ./...`,
`go generate ./... && git diff --exit-code` (which runs the generator from `tools.mod`),
`go mod tidy -diff` (empty), each fuzz target for 30 s
(`go test -fuzz=FuzzParse -fuzztime=30s ./tei` and the others), the integration suite against a
locally built oolong at the `2026-11-oolong` head (name the commit in the PR), and the live suite
once (name the date). The ASCII check of the coding principles over the hand-written files.

## 5. Risks and open points

- **Split recommended: three children, one repository, in order.** The surface is 28 wrapped
  operations, discovery with failover, auth recovery, verified downloads, a flow helper, an
  integration harness and CI; one coder pass would be well past a day and the review of it
  unwieldy. Each child is fully specified by this design and builds only the sections named;
  each test (sub-)case is assigned to the child that builds the code it exercises, and every
  case of section 4 is assigned exactly once (checked against the threat model's table,
  which now has a child column):
  1. **OOL-3a: tea-client-go foundation: pinned spec, generated teaapi, tei package, client,
     URL policy, limits and time bounds, errors, typed calls, token request, trace, oolong
     integration harness, CI** -- 3.1 (with `tools.mod`, `spec/LICENSE`, `teaapi/doc.go`),
     3.2, 3.3, 3.4 (all of it: options including `WithTrace`, `Limits` including
     `AttemptTimeout`, `CallTimeout`, `IdleTimeout`, `DiscoveryResults` as fields, the bounded
     doer with call kinds and `Generated()` path classification, the time-bound mechanism,
     `CheckRedirect` with the `kindToken` no-redirect rule, user agent), 3.5 (including
     `RequestToken` and `Generated()`), 3.6 (errors part: the error types, sentinels including
     `ErrTimeout` and `ErrTooManyResults`, mapping), 3.7 (`WithBearerToken`, the token request
     and `RequestToken`; D5's removal rule in the auth transport), 3.11 (`ci.yaml` jobs `test`,
     `integration`, `record-release`; a minimal README with the Licence and Versioning
     sections). Tests: 4.1 cases 1-8, 9a, 10, 11a, 11c, 11d, 12, 13a, 14a, 15, 16, 18-20, 39a, 40;
     4.2 harness and cases 5, 6, 9. Depends on nothing. **Design from parent: yes.**
  2. **OOL-3b: tea-client-go discovery: well-known, endpoint selection, failover with
     attempt timeouts, results cap, servers selection, PURL discovery** -- 3.6 (discovery
     part: `Discover`, `DiscoverPURL`, `SelectServer`, `DiscoveryError`/`Attempt`, steps 1-6
     including `AttemptTimeout` use and the `DiscoveryResults` cap), D4 on learned bases.
     Tests: 4.1 cases 9b (= 21b, 27b), 11b, 13b, 17, 21, 21b, 22-25, 26, 26b, 26c, 27, 27b, 39b;
     4.2 cases 1-3; 4.3 case 1; the `live` job of `ci.yaml`. Depends on OOL-3a.
     **Design from parent: yes.**
  3. **OOL-3c: tea-client-go downloads, checksum verification, flow helper, API-key auth,
     README** -- 3.7 (`WithAPIKey` recovery), 3.8 (including `IdleTimeout` on the wrapped
     downloads), 3.9 (including the `Candidates` cap), 3.10 (documented in README), README of
     3.11 in full. Tests: 4.1 cases 13c (= 30), 14b, 28, 29, 30, 31, 31b, 32-38, 38b; 4.2
     cases 4, 7, 8; 4.3 cases 2-4. Depends on OOL-3b. **Design from parent: yes.**
  Signed off without `--no-split`.
- **BLAKE3 is unsupported.** Neither the standard library nor `golang.org/x/crypto` implements
  it; adding a third-party crypto module for an algorithm neither oolong's fixtures nor the demo
  use is not worth the supply-chain cost now. A BLAKE3-only format fails closed
  (`ErrNoUsableChecksum`), never passes unverified. Revisit when a server publishes BLAKE3.
- **SHA-1-only formats fail closed by default.** The spec allows SHA-1; a caller can opt in
  with `AllowSHA1Only`. A choice the spec leaves open, named here.
- **Interpretation of the http allowance (D4), a stated departure from the letter of
  `discovery/readme.md`.** The spec says the allowance "is per configured base URL, never a
  global setting, so it cannot apply to a base the client learned from discovery". Round 1
  allowed a learned base over http only when byte-equal to a configured one; round 2 allows
  one that is **under** a configured base (path segments, no dot segments), so that the CLI's
  `--usehttp` keeps working for a server whose API root is a path on its origin (3.10). What
  this keeps from the spec's intent: the allowance is still per configured string and never
  global; nothing a server sends creates or widens one; a learned base can never reach a
  scheme, host or port over http that the caller did not name; and without an allowance
  nothing is ever fetched over http. What it gives up: a hostile or misconfigured document on
  an allowed origin can choose the path the client uses there. Since the caller has declared
  that origin plain-http (so anyone on the path can already change every byte), this adds no
  capability an attacker did not have. The comment at the `under` helper names the spec
  section and this departure ("Implement the pinned TEA specification"). Rejected
  alternatives: a separate CLI flag for the API base (`--tea-base`), which changes the CLI's
  surface and still cannot cover a `servers[].rootUrl` that differs from the well-known `url`;
  and a byte-equal rule, which breaks `--usehttp` for the local-ReARM shape.
- **Time bounds are defaults, not guarantees for a caller who disables them.** A negative
  `AttemptTimeout`, `CallTimeout` or `IdleTimeout` leaves only the caller's context; the doc
  comment of `Limits` says so. The 15 s / 30 s / 30 s defaults are chosen to match the CLI's
  30 s and to let a 4-attempt discovery finish in about a minute.
- **Through `Generated()`, a download is read into memory** up to `DownloadBytes` (512 MiB by
  default): that is the generated code's shape (`[]byte`), and the wrapped
  `DownloadArtifact` streams instead. Documented on `Generated()`.
- **Schema validation is partial (D8).** The client validates what it builds URLs from or trusts
  bytes with, not every response field against the schema; server conformance is oolong's job
  and its e2e suite checks responses against the pinned document.
- **SSRF for server-side consumers.** A TEI from an untrusted party makes the client fetch
  `https` URLs chosen by that TEI's domain and its documents, possibly internal addresses. The
  library does not filter addresses; README tells server-side consumers to pass an
  `http.Client` whose dialer refuses private ranges. Threat model T9.
- **The integration suite needs Node 24 and an oolong checkout.** CI provides both; locally the
  suite skips without them. It follows the moving `2026-11-oolong` head, as the task asks; a
  server change that breaks it shows as an integration failure naming the oolong commit.
- **ReARM release recording needs the operator**: an OIDC trust rule for
  `oolong-project/tea-client-go` on demo and the repository variable `REARM_RECORD_RELEASE`.
  Until then the job is skipped. Noted for the operator in the sign-off.
- **oapi-codegen output size** (about 440 KB) is checked in, as rearm-client-go checks in its
  generated file.
- **`runtime`'s module graph reaches consumers** (3.1): accepted, stated in README; a later
  `runtime` release with a smaller `go.mod` is picked up by a normal dependency bump.
- No open questions for the operator.
