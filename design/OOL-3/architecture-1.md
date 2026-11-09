# OOL-3 architecture, round 1: tea-client-go, a Go client library for TEA v1.0.0

Task: OOL-3. Coding principles: `coding-principles/coding_principles.md` at commit
`145008cdcc23`. Read at: oolong-project/oolong `2026-11-oolong` @ `0fac816`,
oolong-project/tea-client-go `2026-11-oolong` @ `82d03da` (LICENSE only),
github.com/relizaio/rearm-client-go @ `7dc78ac`, github.com/relizaio/rearm-cli @ `82f908b`
(`cmd/tea.go`), and the TEA specification at CycloneDX/transparency-exchange-api commit
`962a0228af1bfcc1232ac3f79239317ff33fdb04` (`spec/openapi.yaml`, `discovery/readme.md`,
`discovery/tea-well-known.schema.json`, `tei-types/*.md`). Every fact in 3.0 was checked
against those, or by running the command it names.

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
  `rearm_auth: github-oidc` and no stored secret.

### 3.1 Layout of the module

```
go.mod                      module github.com/oolong-project/tea-client-go; go 1.26.0
                            tool github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen (v2.8.2)
spec/openapi.yaml           byte copy of the pinned upstream file (never edited)
spec/tea-well-known.schema.json  byte copy, same commit
spec/SPEC_PIN.md            repository, paths, commit, blob ids, retrieved date
teaapi/                     package teaapi: GENERATED models and low-level client
  generate.go               //go:generate go tool oapi-codegen -config oapi-codegen.yaml ../spec/openapi.yaml
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
Dependencies: `github.com/oapi-codegen/runtime`, `github.com/google/uuid` (both from the
generated code), `golang.org/x/crypto` (`blake2b`), `golang.org/x/mod` (`semver`). No others.

### 3.2 Decisions

- **D1, generator: oapi-codegen v2.8.2, models and client, into `teaapi`.** It reads the
  pinned 3.1.1 document as is (3.0), yields flat `allOf` types, typed enums with `Valid()`,
  `time.Time` timestamps and a client over an injectable `HttpRequestDoer`, with two small
  runtime dependencies. A spec change regenerates `teaapi` and breaks consumers at compile
  time, as genqlient does for rearm-client-go. Rejected: **ogen** (pulls OpenTelemetry into
  every consumer, and its strict decoders reject unknown fields, which `discovery/readme.md`
  "Unresolved identifiers" tells clients to ignore, so a later TEA minor would break it);
  **openapi-generator** `go` (needs a Java runtime, so no plain `go generate`, and much larger
  output). The generator is pinned with a `tool` directive in `go.mod` (Go 1.24+), so
  `go generate ./...` needs nothing installed; CI checks the checked-in output is current
  (3.11). "Generated code is not edited by hand" applies to `teaapi/tea.gen.go`: a regenerate
  is its own commit with nothing else in it. Where the generated code is unsuitable (the
  unbounded reads and `[]byte` downloads above), the hand-written layer does not use that part
  (D3, 3.8), it does not patch it.
- **D2, two layers.** `teaapi` is exported, so consumers can name every spec type
  (`teaapi.Product`, `teaapi.Collection`, ...). The hand-written `tea` package is the API people
  use: it returns `teaapi` types (no aliases, no copies: one source of truth for shapes) and
  owns everything the spec leaves to the client: URL policy, limits, auth, errors, discovery,
  verification. `API.Generated()` returns the `*teaapi.ClientWithResponses` built on the same
  bounded, auth-aware doer for anything not wrapped.
- **D3, every response is bounded** ("The Go client treats every server as untrusted"). The doer
  the generated client uses wraps each response body in a reader that fails with
  `ErrResponseTooLarge` past the limit for that kind of call (3.4), and refuses at once when
  `Content-Length` is larger. The limit applies after the transport's own gzip decoding, so a
  compressed bomb is caught too.
- **D4, URL policy: https only, plus exact per-base http allowances.** Every URL the client
  requests (well-known, API bases, redirects, external artifact URLs) must be `https`, or `http`
  and exactly equal (scheme, host, port, path) to an API base or well-known location the
  caller passed with `WithInsecureHTTPBase`. A base learned from a server is allowed over http
  only if it equals one of those configured strings; there is no global switch. This is the
  spec's per-base allowance read literally: the allowance is attached to a configured string,
  so a hostile document cannot widen it. No URL with userinfo is followed. TLS verification is
  the `http.Client`'s; the library never sets `InsecureSkipVerify` and exports no option to.
- **D5, credentials are bound to one API root and added by the transport.** Credentials are
  configured per `rootUrl` (`WithBearerToken(rootURL, token)`, `WithAPIKey(rootURL, id,
  secret)`). The transport adds `Authorization` only when the request URL has the same scheme,
  host and port as that root and its path is the root's path or below it. It is evaluated on
  every request, redirects included, so a `302` to another origin, an external `url`, or a
  `servers[]` entry on another host never sees the token, whatever Go's own redirect header
  rules would do (Go forwards `Authorization` to subdomains). Credentials never appear in a
  URL, an error or a log line ("Secrets and credentials").
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
  strictly (3.5, 3.6, 3.8): well-known and discovery documents in full, uuids used in paths,
  `rootUrl`, versions, pagination tokens, checksums.
- **D9, no automatic retries of API calls**, except discovery's bounded failover (3.6) and
  the single token recovery on `401` (3.7). The caller's `context.Context` bounds every call.
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
func WithUserAgent(ua string) Option             // default "tea-client-go/<module version>"
func WithInsecureHTTPBase(rawURL string) Option  // D4; repeatable; validated as an absolute http URL
func WithWellKnownLocation(domain, baseURL string) Option // fetch <baseURL>/.well-known/tea for domain
func WithBearerToken(rootURL, token string) Option        // D5
func WithAPIKey(rootURL, keyID, secret string) Option     // D5, 3.7
func WithLimits(l Limits) Option
func WithDiscoveryAttempts(n int, firstBackoff time.Duration) Option
func (c *Client) API(base APIBase) (*API, error)
func ParseAPIBase(rootURL, version string) (APIBase, error)
type APIBase struct{ /* rootURL *url.URL; version string */ }
func (b APIBase) RootURL() string; func (b APIBase) Version() string; func (b APIBase) String() string // root + "/v" + version
```

`WithWellKnownLocation` replaces `--usehttp`/`--useport` for the CLI and is how tests point
`tei://localhost/...` at a local server; an `http` location also needs
`WithInsecureHTTPBase`. `New` refuses an option value it cannot parse (`ErrInvalidOption`),
and an http base given to `WithBearerToken`/`WithAPIKey` that is not also an insecure base.

`Limits` (zero value = default):

| field | default | applies to |
| --- | --- | --- |
| `WellKnownBytes` | 64 KiB | `.well-known/tea` |
| `DiscoveryBytes` | 1 MiB | `/discovery` |
| `JSONBytes` | 16 MiB | every other JSON response |
| `ErrorBodyBytes` | 64 KiB | non-2xx bodies (read to classify, then discarded) |
| `DownloadBytes` | 512 MiB | artifact content (per call override in `DownloadOptions`) |
| `SignatureBytes` | 16 MiB | signature content |
| `Redirects` | 5 | per request |
| `Pages` | 1000 | per iterator (3.5) |
| `FlowComponents` | 1000 | component refs walked by the flow helper |

The demo's one BOM is 14.3 MB and oolong refuses artifact bodies over 32,000,000 bytes, so
512 MiB leaves room without letting a server fill a disk. The default transport (when the
caller passes no `http.Client`) is a clone of `http.DefaultTransport` with
`TLSHandshakeTimeout` 10 s and `ResponseHeaderTimeout` 30 s; there is no whole-request
timeout (downloads are long; the context bounds them).

`CheckRedirect` on the library's copy of the `http.Client`: at most `Limits.Redirects`; every
target passes D4; the auth decision is the transport's (D5). A refused redirect returns
`*URLPolicyError` (wraps `ErrInsecureURL` or `ErrRedirectLimit`).

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
through 3.6's error mapping. A response is also checked for what callers rely on: a uuid
field used later in a path matches the spec's lowercase pattern; pagination with
`hasNext: true` and no `nextPageToken` is `*UnexpectedResponseError`. Iterators stop at the
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
var ErrResponseTooLarge, ErrInvalidArgument, ErrInvalidOption, ErrPageLimit, ErrTooManyComponents error
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
5. Classify: `200` with a valid body (JSON array, at least one item, each item's
   `productReleaseUuid` matching the uuid pattern, `servers` non-empty, each `rootUrl` matching
   the `server-info` pattern and each version the SemVer pattern) returns the results in array
   order. A TEA `404` returns `*NotResolvedError` at once, no failover. `400`, `401`, `403`
   return `*APIError` at once, no failover. DNS or connection failure, TLS failure, timeout of
   the attempt (not of the caller's context), `5xx`, a non-TEA `404`, an unusable or invalid
   body: a failed attempt; move to the next candidate.
6. Attempt bound: `WithDiscoveryAttempts` (default 4 attempts in total across both stages);
   the well-known fetch itself is retried within the bound on failed attempts. Backoff before
   each retry: 250 ms doubling, capped at 4 s, interrupted by the context. Exhausted:
   `*DiscoveryError` with the stage and each attempt's URL (without query) and reason.

`SelectServer` applies step 3's ordering to `Servers` and D4 to the chosen `rootUrl`; an
`http` root is usable only if it exactly equals a configured insecure base. The `/discovery`
base is not reused unless it is listed in `servers` ("Discovery response"). No DNS
pre-resolution step (the CLI's `resolveDNS`): the HTTPS request resolves, and a DNS failure is
a failed attempt.

### 3.7 Authentication (`auth.go`)

Anonymous by default: no `Authorization` header is ever sent without a configured credential.

- `WithBearerToken(root, token)`: the transport adds `Authorization: Bearer <token>` to
  requests under `root` (D5). A `401` is returned as `*APIError` with the challenge.
- `WithAPIKey(root, id, secret)`: no token up front (clients do not probe `/token`). On a `401`
  from a request under `root` whose `WWW-Authenticate` is a Bearer challenge, the client posts
  `grant_type=client_credentials` (form-encoded) to `<root>/v<version>/token` with
  `Authorization: Basic base64(formurlencode(id) ":" formurlencode(secret))`, reads the
  `token-response` (bounded by `ErrorBodyBytes`; `token_type` must be `Bearer`;
  `access_token` non-empty), caches it with its `expires_in` (if given, minus 30 s), and
  retries the original request once. A `401` with `error="invalid_token"` on a request that
  carried a cached token drops it, obtains one fresh token and retries once; never more than
  one recovery per request. A `/token` `404` is `ErrNotImplemented`; a token error
  (`token-error-response`) is `*TokenError{Code}` without the description echoed beyond 200
  sanitised bytes. A `403` never triggers recovery. Requests with a body are retried only for
  `GET` (all resource operations are `GET`).
- Neither credential is sent to a well-known location, to `/discovery` on a base other than
  `root`, to an external artifact URL, or across a redirect off `root` (D5).

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
continue with the chosen one. `FlowFromRelease`: get the product release (its
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
| `--usehttp`, `--useport` | `WithWellKnownLocation(domain, "http://host:port")` + `WithInsecureHTTPBase` |
| `resolveTEI` / `discovery` | `Client.Discover` |
| stdin disambiguation | `*AmbiguousError` + `Candidates`, prompt stays in the CLI |
| `full_tea_flow` | `Client.FullFlow`, printing stays in the CLI |
| `getCle` / `printCLE` | `Get...CLE` (typed `teaapi.Cle`) |

The CLI's `swid` and `gs1` legacy types have no v1.0.0 registry entry; `ParseLegacyURN`
refuses them with a message the CLI can show.

### 3.11 README, examples, CI

- `README.md`: what the module is; the two packages; install; the example below; options and
  limits; the URL and credential policy (D4, D5); the legacy URN note; how to regenerate
  (`go generate ./...`, never edit `teaapi/tea.gen.go`); how to run unit, integration (4.2) and
  live (4.3) tests. Its code is copied from `example_test.go`, which compiles in `go test`:

```go
c, err := tea.New()
t, err := tei.Parse("tei://demo.rearmhq.com/uuid/b9f115fb-6688-4a3f-a262-3acc04e1fc9e")
res, err := c.FullFlow(ctx, t, tea.FlowOptions{})
fmt.Println(*res.Release.ProductName, res.Release.Version)
```

  (examples have no `// Output:` line, so `go test` compiles but does not run them: no network.)
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
   `b0804f1ae79c8ec72adea6d5c12e8f75f6036729`.
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
8. `New` refuses: `WithInsecureHTTPBase("https://...")`, a relative URL, one with userinfo or
   query; `WithBearerToken` on an http root not also allowed.
9. Allowance is exact: with `WithInsecureHTTPBase("http://127.0.0.1:PORT")`, a request to
   `http://127.0.0.1:PORT/v1.0.0/...` under an API base equal to it is allowed;
   `http://127.0.0.1:OTHER`, `http://localhost:PORT` and a discovered `http` root that differs
   by path are refused with `ErrInsecureURL` and no request reaches the server (handler counts).
10. Redirects: https to http refused; to userinfo URL refused; 6th redirect gives
    `ErrRedirectLimit`.
11. Limits: a JSON response one byte over `JSONBytes` gives `ErrResponseTooLarge` (with and
    without `Content-Length`); a gzip body that expands over the limit gives the same; well-known
    over 64 KiB and discovery over 1 MiB likewise.

**Credentials (`auth_test.go`)**
12. Anonymous: no `Authorization` header on any request.
13. Bearer bound to root: sent to `<root>/v1.0.0/...`; not sent to a sibling path outside the
    root, another port, a subdomain, a `302` target on another origin (the second server sees no
    header), an external artifact `url`, the well-known location, or a `servers[]` root on
    another host.
14. API key: first request `401` with `WWW-Authenticate: Bearer realm="x"` triggers exactly one
    `POST /token` with `grant_type=client_credentials` and Basic credentials form-encoded
    (id `a:b@c`, secret `p:q%`), then one retry with the token; `invalid_token` on a cached token
    gives one refresh and one retry, a second `invalid_token` is returned as `ErrUnauthorized`
    (two token requests total, no loop); `403` gives no token request; `/token` `404` gives
    `ErrNotImplemented`; `token_type` other than `Bearer` refused; neither secret nor token
    appears in any returned error string (assert with `strings.Contains`).

**Errors (`errors_test.go`)**
15. Mapping: each of the six codes with `application/json; charset=utf-8` to its sentinel via
    `errors.Is`; an unknown code `FUTURE_CODE` gives `ErrorCodeUnknown` with `RawCode`; extra
    properties ignored; `{"message":"Not Found"}`, a JSON array, `text/html` and an empty body on
    `404` give `TEA: false`; `401` without body keeps the parsed challenge; `5xx` gives
    `ErrServer`; a `message` of 10 KiB with control characters is cut to 200 bytes without them.
16. Unexpected JSON: for every typed call, a 200 with invalid JSON, `null`, a wrong type, and a
    missing body gives `*UnexpectedResponseError` and no panic (one table over all methods).
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
22. Selection: endpoints `[{v 0.2.0-beta.2}, {1.0.0, priority 0.5}, {1.0.0, no priority}]`
    calls the third first; `["1.0.0-rc.1"]` only: no candidate, `*DiscoveryError` stage
    well-known, no discovery request; ties keep document order.
23. Well-known validation, each a failed attempt: `schemaVersion` 2 or `"1"`, empty
    `endpoints`, url with query, userinfo, trailing slash or `http` (not allowed), version `v1.0.0`,
    priority 1.5, not JSON. Unknown extra property: accepted.
24. Failover: first endpoint answers `503`, non-TEA `404`, invalid JSON, `[]`, or a result with a
    bad uuid / http rootUrl; the second endpoint is tried and its result returned. A TLS failure
    (a test server with an untrusted certificate) is a failed attempt.
25. No failover: TEA `404 OBJECT_UNKNOWN` gives `*NotResolvedError` with the code and no request to
    the second endpoint; same for `401` and `403` (as `*APIError`) and `400`.
26. Bound and backoff: four failing endpoints with attempts set to 3 make exactly 3 requests and
    return `*DiscoveryError` listing them; a cancelled context during backoff returns
    `context.Canceled` promptly.
27. Results: two results are returned in array order; `SelectServer` picks per 22's rules and
    refuses an http root unless exactly allowed; `DiscoverPURL` sends `purl=` and not `tei=`.

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

1. `Discover`: at least one result; every result has a valid uuid and at least one server whose
   selected base is `https` with version `1.0.0`; the TEI's uuid is among the results (it is the
   identifier itself).
2. `FullFlow` (with `Select` choosing 0 if several): the release has non-empty `Version`, a uuid,
   `LatestCollection.Version >= 1`; every component has `Release != nil` or a typed `Err`; no
   untyped error.
3. For the first artifact format found in the flow (product release first, then components) that
   has at least one supported checksum and size within a 64 MiB test cap: `DownloadArtifact` to
   `io.Discard` verifies. If none qualifies, the subtest skips with the reason.
4. An unknown product uuid gives `ErrObjectUnknown`.

### 4.4 Suites and checks the coder runs before hand-over

`gofmt -l .` (empty), `go vet ./...`, `go test ./...`, `go test -race ./...`,
`go generate ./... && git diff --exit-code`, each fuzz target for 30 s
(`go test -fuzz=FuzzParse -fuzztime=30s ./tei` and the others), the integration suite against a
locally built oolong at the `2026-11-oolong` head (name the commit in the PR), and the live suite
once (name the date). The ASCII check of the coding principles over the hand-written files.

## 5. Risks and open points

- **Split recommended: three children, one repository, in order.** The surface is 28 wrapped
  operations, discovery with failover, auth recovery, verified downloads, a flow helper, an
  integration harness and CI; one coder pass would be well past a day and the review of it
  unwieldy. Each child is fully specified by this design and builds only the sections named:
  1. **OOL-3a: tea-client-go foundation: pinned spec, generated teaapi, tei package, client,
     URL policy, limits, errors, typed calls, oolong integration harness, CI** -- 3.1, 3.2,
     3.3, 3.4, 3.5, 3.6 (errors part), 3.7 (`WithBearerToken` only), 3.11 (`ci.yaml` jobs
     `test`, `integration`, `record-release`; a minimal README); tests 4.1 cases 1-13 and 15-20,
     4.2 harness and cases 5, 6, 9. Depends on nothing.
     **Design from parent: yes.**
  2. **OOL-3b: tea-client-go discovery: well-known, endpoint selection, failover, servers
     selection, PURL discovery** -- 3.6 (discovery part); tests 4.1 cases 21-27, 4.2 cases 1-3,
     4.3 case 1, and the `live` job of `ci.yaml`. Depends on OOL-3a. **Design from parent: yes.**
  3. **OOL-3c: tea-client-go downloads, checksum verification, flow helper, API-key auth,
     README** -- 3.7 (`WithAPIKey`), 3.8, 3.9, 3.10 (documented in README), README of 3.11;
     tests 4.1 cases 14, 28-38, 4.2 cases 4, 7, 8, 4.3 cases 2-4. Depends on OOL-3b.
     **Design from parent: yes.**
  Signed off without `--no-split`.
- **BLAKE3 is unsupported.** Neither the standard library nor `golang.org/x/crypto` implements
  it; adding a third-party crypto module for an algorithm neither oolong's fixtures nor the demo
  use is not worth the supply-chain cost now. A BLAKE3-only format fails closed
  (`ErrNoUsableChecksum`), never passes unverified. Revisit when a server publishes BLAKE3.
- **SHA-1-only formats fail closed by default.** The spec allows SHA-1; a caller can opt in
  with `AllowSHA1Only`. A choice the spec leaves open, named here.
- **Interpretation of the http allowance (D4).** The spec forbids applying the allowance to a
  base learned from discovery; this design lets a learned base use http only when it is
  byte-equal to a configured one, which is what local testing against oolong needs (its
  `servers[].rootUrl` is `SERVER_HOST`). If the reviewer reads the spec as forbidding even that,
  the alternative is to require the caller to build the `API` from the configured base directly
  for local servers; the tests then change in 4.1 case 9 and 4.2 case 1 only.
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
- No open questions for the operator.
