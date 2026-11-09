# OOL-5 architecture, round 1: tea-client-go discovery (child 2 of OOL-3)

Task: OOL-5. Coding principles: `coding-principles/coding_principles.md` at commit
`145008cdcc23`. This round is the design carried from OOL-3 (ARCHITECTURE round 2, release
`28dce0e1-36f5-4840-b2cd-09b798a457ec`, `design/OOL-3/architecture-2.md`; THREAT_MODEL round 2,
release `a5ff2086-78a9-452e-8ccc-e3fc22ba4b89`, `design/OOL-3/threat_model-2.md`), cut down to
what this child builds and amended to answer the coder's questions. Read at: the TEA
specification at CycloneDX/transparency-exchange-api commit
`962a0228af1bfcc1232ac3f79239317ff33fdb04` (`discovery/readme.md`), oolong-project/oolong
`2026-11-oolong` @ `e1c9e6e` (built and run for Q-1), and oolong-project/tea-client-go PR #2
head `16ee9fa` (base `2026-11-oolong` @ `1c85712`). The threat model is unchanged and stays
OOL-3's round 2: no answer below changes a threat or a mitigation (section 5).

## Scope of this child

OOL-5 builds, on top of OOL-4 (OOL-3a, merged at `1c85712`), the discovery part of OOL-3's
design: 3.6 discovery (`Discover`, `DiscoverPURL`, `SelectServer`, `DiscoveryError`,
`Attempt`, steps 1-6 including `AttemptTimeout` use and the `DiscoveryResults` cap), D4 on
learned bases, and the `live` job of `ci.yaml` (3.11). Tests: 4.1 cases 9b (= 21b, 27b), 11b,
13b, 17, 21, 21b, 22-25, 26, 26b, 26c, 27, 27b, 39b; 4.2 cases 1-3; 4.3 case 1. Section and
case numbers are OOL-3's, kept so that references stay valid; sections this child does not
build (3.1, 3.3, 3.5, 3.7-3.10, most of 3.4 and 3.11) are OOL-3's round 2 as passed, and are
not repeated here. OOL-6 (OOL-3c) still builds from OOL-3's round 2; this round changes
nothing it builds (section 5).

## Changes in this round

Answers the coder's questions `questions/OOL-5/round-1.md` (release
`ee20b0ab-e931-4ea3-a3c3-6e84c75946b0`) about OOL-3's ARCHITECTURE round 2. The pull request
already implements each answer below: **PR #2 at `16ee9fa` needs no code change** for any of
them (the coder's doc comments already state the rules). It needs **three test sub-cases**
that pin the Q-4 rules the PR builds but does not yet test (cases 23 and 24); nothing else.

- **Q-1 (oolong's framework 404 is a TEA error response): the coder is right; the spec's rule
  applies.** Decided rule: whether a `404` stops discovery is decided by the shape of the
  response only (`application/json`, a JSON object, a string `error`), never by the value of
  `error` and never by other properties such as `statusCode`. Oolong's unmounted-path body
  `{"message":"Cannot GET ...","error":"Not Found","statusCode":404}` (re-checked by running
  oolong at `e1c9e6e`, 3.0) has that shape, so at `/discovery` it gives `*NotResolvedError`
  with `Code` `ErrorCodeUnknown` and `RawCode` `Not Found`, and no failover. The spec says so
  in as many words ("shall not reject the response for an `error` value they do not know, so
  that a later TEA version can extend `error-response` without turning its `404` responses
  into failover triggers"), and OOL-3's own 3.6 already adopted that rule; OOL-3's 3.0 calling
  the body "not a TEA error" and 4.2 case 3 expecting a failed attempt were wrong. Recognising
  Nest's body (by `statusCode`, or by refusing an `error` value that is not one of the six
  codes) would break the spec's forward-compatibility rule. Sections amended: 3.0 (Oolong
  server; new bullet "Discovery rules quoted"), 3.6 (TEA error paragraph), 4.2 case 3. PR
  change: none (`TestIntegrationDiscoverWrongPath` asserts exactly the amended case 3). The
  consequence for oolong deployments (a well-known `url` with a wrong path stops discovery
  instead of failing over) is a server matter, noted for the operator in section 5.
- **Q-2 (does a successful well-known fetch count against the bound?): confirmed, every GET
  counts.** Decided rule: each discovery GET is one attempt against the
  `WithDiscoveryAttempts` bound, whether it fails or succeeds, the well-known fetch included;
  `DiscoveryError.Attempts` lists only the failed ones. This is what makes 3.4's "about 62 s"
  bound true, and it is the plain reading of the spec's "limit the total number of attempts
  across both stages". Case 26's "exactly 3 requests" means 3 requests in total: the
  well-known fetch and two `/discovery` requests, with 2 attempts listed. `WithDiscoveryAttempts(1, ...)`
  is accepted (it is useful for `DiscoverPURL`) and makes `Discover` end in `*DiscoveryError`
  after the well-known fetch; its doc comment says so. Sections amended: 3.4 (the option's
  comment), 3.6 step 6, 4.1 case 26. PR change: none.
- **Q-3 (are candidates tried again once all have failed?): confirmed, round-robin until the
  bound.** Decided rule: while attempts remain, after the last candidate the client starts
  again from the first, in the same order, with the backoff before each retry; the bound, not
  the number of candidates, ends discovery. The spec's "select the next untried ... endpoint
  ..., if one is available" orders the attempts while untried candidates remain but does not
  end discovery when none remain; the spec's attempt bound and "retry with backoff" do, and
  step 6 already retries the single well-known fetch the same way. Sections amended: 3.6
  steps 4 and 6, 4.1 cases 26 and 26c. PR change: none.
- **Q-4 (statuses and answers step 5 does not name): all four readings confirmed.** Decided
  rules: (a) at `/discovery`, only `200` with a valid body succeeds and only a TEA `404`,
  `400`, `401` and `403` stop; every other status (another `2xx` such as `204`, a `3xx` the
  client did not follow, `405`, `429`, any other `4xx`) is a failed attempt, as is a redirect
  refused by D4 or over the redirect limit; (b) at the well-known stage there is no API base
  and "Unresolved identifiers" does not apply, so any `404`, TEA-shaped or not, is a failed
  attempt, while `400`, `401` and `403` stop with `*APIError` as at `/discovery`; (c) a
  `/discovery` answer with any `servers[].rootUrl` refused by D4, or any `priority` outside
  [0, 1], is invalid as a whole (a failed attempt), even if another server or result in it is
  usable: the spec says conforming deployments "shall advertise only lowercase `https` base
  URLs, in `.well-known/tea` `endpoints[].url` and in `/discovery` `servers[].rootUrl`
  alike", the `server-info` schema bounds `priority`, and step 2 already refuses a whole
  well-known document for one refused `url` (case 23); (d) `SelectServer` applies D4 to the
  server its ordering puts first and returns `*URLPolicyError` when that one is refused, with
  no fallback to the next: after (c), every result `Discover` returns has only roots this
  client accepts, so the check only fires on results a caller built or carried to a client
  with other allowances, and there failing closed is the rule. Sections amended: 3.6 step 5,
  the well-known paragraph after step 5, and `SelectServer`; 4.1 cases 23, 24 and 27. PR
  change: no code change; add the sub-cases now listed in case 23 (a TEA-shaped `404` at the
  well-known stage is a failed attempt) and case 24 (a `429` and a `204` fail over; a result
  listing an allowed `https` server next to an `http` root not under an allowance is refused
  whole and fails over). Case 27's `SelectServer` sub-cases are already built on
  `DiscoveryResult` values the test builds.

## 1. Problem

OOL-3 designs `github.com/oolong-project/tea-client-go`, a typed Go client for TEA v1.0.0.
OOL-4 built its foundation (pinned spec, generated `teaapi`, `tei`, client, URL policy,
limits and time bounds, errors, typed calls, trace, the oolong harness, CI). This child adds
discovery as `discovery/readme.md` defines it: the well-known document, endpoint selection,
`/discovery` with failover under an attempt bound and backoff, the results cap, server
selection from `servers[]`, and discovery by PURL.

Out of scope: downloads, checksum verification, the flow helper, API-key auth and the full
README (OOL-6); any server change in oolong-project/oolong; switching the CLI.

## 2. Repositories and branches

- oolong-project/tea-client-go, branch `2026-11-oolong` (PR #2 against it, never `main`):
  `discovery.go`, its tests, `integration_test.go` and `live_test.go` cases, the `live` job of
  `.github/workflows/ci.yaml`, README and example lines for discovery.
- oolong-project/oolong: not changed. Its `2026-11-oolong` head is the integration-test server
  (4.2).

## 3. Design

### 3.0 Facts this design rests on (checked)

- **Spec pin.** `tea-spec/SPEC_PIN.md` in oolong names repository
  `https://github.com/CycloneDX/transparency-exchange-api`, path `spec/openapi.yaml`, commit
  `962a0228af1bfcc1232ac3f79239317ff33fdb04`, git blob `c96baa3e102c195895643fab81ae468ee54b8182`,
  OpenAPI 3.1.1, `info.version` 1.0.0, prefix `/v1.0.0`. The file fetched from
  `raw.githubusercontent.com` at that commit has that blob id and is byte-identical to oolong's
  `tea-spec/openapi.yaml`. `discovery/tea-well-known.schema.json` at the same commit has blob
  `b0804f1ae79c8ec72adea6d5c12e8f75f6036729` (oolong does not carry it).
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
  returns one result whose `rootUrl` is `SERVER_HOST`. Unmounted paths return Nest's default
  404 body with `Content-Type: application/json; charset=utf-8`: run again at `2026-11-oolong`
  @ `e1c9e6e` for this round, `GET /nope/v1.0.0/discovery?tei=x` is `404`
  `{"message":"Cannot GET /nope/v1.0.0/discovery?tei=x","error":"Not Found","statusCode":404}`
  (`TeaExceptionFilter` in `src/tea/errors.ts` passes a Nest `HttpException` 404 payload
  through as JSON). By the spec's definition this **is** a TEA error response, with the
  unknown code `Not Found` (Q-1; OOL-3's round 2 called it "not a TEA error", which was wrong).
  An unknown TEI at the right base is `404 {"error":"OBJECT_UNKNOWN"}`. CLE and `/token`
  return `404 {"error":"NOT_IMPLEMENTED"}`; a bad uuid is `400 INVALID_REQUEST`; revision 1 of
  artifact `55555555-...-555555555555` downloads as `302` to `https://example.invalid/...`.
  Fixture ids, bytes and checksums are in oolong `docs/tea-upgrade/fixture-reference.md`.
  `oolong/docker-compose.tea-interop.yml` runs the same server at `127.0.0.1:3005` (Docker is
  not available on this host; the design does not need it).
- **Go.** Go 1.26.0 is installed locally (`/usr/local/go/bin`); rearm-cli's `go.mod` says
  `go 1.26.0`. `crypto/sha3` is in the standard library.
- **Discovery rules quoted** (`discovery/readme.md` at the pin, read for this round):
  - "Unresolved identifiers": "A response is a TEA error response only when its `Content-Type`
    is `application/json` (optionally with parameters such as `charset`) and its body is a
    JSON object with a string `error` property, typically `OBJECT_UNKNOWN`. Clients shall
    ignore properties they do not recognize and shall not reject the response for an `error`
    value they do not know, so that a later TEA version can extend `error-response` without
    turning its `404` responses into failover triggers. Other JSON `404` bodies, such as
    `{"message":"Not Found"}`, are not TEA error responses." The section and the "404 Not
    Found" list are about `/discovery`; nothing in the spec makes a well-known `404` a stop.
  - "Failover, invalid documents, and attempt bounds": the failed-attempt classes "whether
    they occur in the well-known stage or the API discovery stage" (DNS, TLS, `5xx`, a non-TEA
    `404`, unusable or non-conforming JSON); "the client shall select the next untried
    well-known endpoint that supports a compatible API version, if one is available";
    "Clients should limit the total number of attempts across both stages for a discovery
    operation. Additional attempts should use exponential backoff." "Client behavior": "retry
    with backoff", "fail closed if discovery cannot be validated".
  - "TLS Encryption": "Conforming deployments shall advertise only lowercase `https` base
    URLs, in `.well-known/tea` `endpoints[].url` and in `/discovery` `servers[].rootUrl`
    alike. A client shall reject an `http` base URL unless it has been explicitly configured
    to allow that specific base for local testing."
  - "Authentication and authorization": "Clients shall not fail over to another endpoint
    solely in response to `401` or `403`."
- **What `--usehttp` does today** (`cmd/tea.go` `queryWellKnown`, `createHTTPClient`): it
  fetches `http://<domain>:<useport>/.well-known/tea`; every later URL is the endpoint
  `url` from that document, used as written (so `http` when the document says `http`); one
  `http.Client` with `Timeout: 30 * time.Second` per call. With `debug=true` it prints the
  domain, the resolved hosts, the well-known URL and the discovery URL (TEI query
  included) before requesting them.

### 3.2 Decisions (carried from OOL-3; those this child relies on)

D3 (every response bounded in bytes and in time by the doer, per call kind: `kindWellKnown`,
`kindDiscovery`), D5 (credentials bound to a root, added and removed by the transport) and
D9 (no retries except discovery's bounded failover) are built in OOL-4 and used here as they
are; their text is OOL-3's round 2. The decisions discovery itself rests on:

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
- **D10, versions.** The client supports exactly `1.0.0` (`tea.SupportedVersions`).
  `golang.org/x/mod/semver` compares versions after a strict SemVer 2.0.0 check with the
  spec's pattern (x/mod accepts shorthand forms the spec does not); build metadata is ignored
  for matching and the advertised string is used verbatim in the path.


### 3.4 Options and limits this child uses (built in OOL-4, `client.go`, `transport.go`)

Built in OOL-4 and used here unchanged, except the doc comment of `WithDiscoveryAttempts`
(Q-2). The full section is OOL-3's round 2, 3.4.

```go
func WithInsecureHTTPBase(rawURL string) Option  // D4; repeatable
func WithWellKnownLocation(domain, baseURL string) Option // fetch <baseURL>/.well-known/tea for domain
func WithLimits(l Limits) Option
func WithDiscoveryAttempts(n int, firstBackoff time.Duration) Option // n >= 1; every discovery GET
    // counts, the well-known fetch included (3.6 step 6), so with n = 1 Discover makes only the
    // well-known request and ends in *DiscoveryError, while DiscoverPURL can use its one attempt
func WithTrace(f func(TraceEvent)) Option        // TraceRequest, TraceResponse, TraceAttemptFailed, TraceBackoff
func ParseAPIBase(rootURL, version string) (APIBase, error)
```

| field | default | applies to |
| --- | --- | --- |
| `WellKnownBytes` | 64 KiB | `.well-known/tea` |
| `DiscoveryBytes` | 1 MiB | `/discovery` |
| `DiscoveryResults` | 32 | results in one `/discovery` answer (and, in OOL-6, results given to `Candidates`) |
| `ErrorBodyBytes` | 64 KiB | non-2xx bodies (read to classify, then discarded) |
| `Redirects` | 5 | per request |
| `AttemptTimeout` | 15 s | each discovery attempt: one well-known GET or one `/discovery` GET, headers and body |

`AttemptTimeout` with the default 4 attempts and backoff bounds a whole `Discover` at about
62 s (4 x 15 s plus 0.25 + 0.5 + 1 s), because every GET counts against the bound (3.6 step
6). When the library's bound fired and the caller's context is still live, the attempt's
error wraps `ErrTimeout` (a failed attempt); when the caller's context ended, `Discover`
returns its `ctx.Err()` wrapped. `CheckRedirect` refuses a redirect target that fails D4 or
exceeds `Redirects` with `*URLPolicyError`.

### 3.6 Errors (`errors.go`) and discovery (`discovery.go`)

The error types are built in OOL-4 (errors part of 3.6) and listed here because discovery
returns them. Errors (all exported, all wrap with context, `fmt.Errorf("getting product release %s: %w", id, err)`):

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
identifiers"); unknown properties and unknown codes are tolerated. **The test is on the shape
only (Q-1):** the value of `error` is never checked against the six codes and no other property
(such as `statusCode`) is looked at, so a framework's default `404` body that carries a string
`error`, as oolong's Nest body does (3.0), is a TEA error; at `/discovery` it stops discovery
with `*NotResolvedError` whose `Code` is `ErrorCodeUnknown` and whose `RawCode` is the value
received (`Not Found`). Refusing such a body by its `error` value or by `statusCode` would be
the rule the spec forbids, since it would turn a later TEA version's `404` into a failover
trigger. A `404` whose body lacks that shape (`{"message":"Not Found"}`, `text/html`, an
empty body, a JSON array) is a non-TEA `404`. A response body never goes
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
4. For each candidate in order (and, while attempts remain, again from the first, step 6): GET
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
   attempt; move to the next candidate. **Also failed attempts (Q-4):**
   - any other status: a `2xx` other than `200` (such as `204`), a `3xx` the client did not
     follow (a `304`, a `3xx` without `Location`), `405`, `429`, any other `4xx`.
     Only `200` with a valid body is success, and only the four answers the spec or the request
     itself makes final stop (a TEA `404`; `401` and `403`, which the spec forbids failing over
     on; `400`, which another endpoint would give for the same request);
   - a redirect refused by D4 or over `Limits.Redirects` (`Attempt.Err` is the
     `*URLPolicyError`);
   - a valid-looking body with any `servers[].rootUrl` refused by D4 (an `http` root not under
     an allowance, a dot segment, a non-ASCII host) or any `priority` outside [0, 1]. The
     answer is refused **whole**, even when another server in the same result, or another
     result, would be usable: the spec requires conforming deployments to advertise only
     `https` bases in `servers[].rootUrl` and the `server-info` schema bounds `priority`, so
     such an answer is non-conforming, and step 2 treats one refused well-known `url` the same
     way. So every `DiscoveryResult` that `Discover` or `DiscoverPURL` returns lists only
     roots this client's D4 accepts.

   **The well-known stage (Q-4)** uses the same classification and `AttemptTimeout` (a body
   over `WellKnownBytes` is a failed attempt), with one difference: there is no API base and
   "Unresolved identifiers" is about `/discovery`, so **every** `404` at the well-known stage,
   a TEA-shaped one included, is a failed attempt, never `*NotResolvedError`. `400`, `401` and
   `403` there stop with `*APIError`, as at `/discovery`.
6. Attempt bound: `WithDiscoveryAttempts` (default 4 attempts in total across both stages).
   **Every discovery GET is one attempt, whether it fails or succeeds (Q-2):** the well-known
   fetch, also when it succeeds, and every `/discovery` request. This is what bounds a whole
   `Discover` at about 62 s by default (3.4). `DiscoveryError.Attempts` lists the failed ones
   only, so four failing endpoints with a bound of 3 give one successful well-known request,
   two `/discovery` requests and two listed attempts. With a bound of 1, `Discover` makes the
   well-known request only and ends in `*DiscoveryError`; `DiscoverPURL`, which has no
   well-known stage, makes its one `/discovery` request. The well-known fetch is retried within
   the bound on failed attempts. **When every candidate has failed and attempts remain, the
   candidates are tried again (Q-3)**, round-robin from the first, in step 3's order, until a
   result, a stop or the bound: one candidate and the default bound give one well-known
   request and three `/discovery` requests to it. The spec's "next untried ... endpoint ...,
   if one is available" orders the attempts while untried candidates remain; it does not end
   discovery when none remain, the attempt bound does ("retry with backoff"). A failure that
   would repeat (an invalid body, too many results) is retried too: telling transient from
   permanent failures is a distinction the spec does not make, and the bound and backoff cap
   the cost. Backoff before each retry: 250 ms doubling, capped at 4 s, interrupted by the
   context. Exhausted: `*DiscoveryError` with the stage and each failed attempt's URL (without
   query) and reason.

`SelectServer` applies step 3's ordering to `Servers` and D4 to the chosen `rootUrl`; an
`http` root is usable only if it is under a configured insecure base (D4's definition).
**The first server in that order is the one checked (Q-4):** if D4 refuses it,
`SelectServer` returns `*URLPolicyError` (wrapping `ErrInsecureURL`) and does not fall back
to the next server. After step 5, a result from `Discover` on the same `Client` never has a
refused root, so this fires only for a `DiscoveryResult` the caller built or carried to a
client with other allowances, and there it fails closed. Because a caller can build one,
`SelectServer` also re-checks each server's `rootUrl` pattern, versions and `priority`
(`ErrInvalidArgument`), and no server with a supported version is `ErrInvalidArgument`. The `/discovery`
base is not reused unless it is listed in `servers` ("Discovery response"). No DNS
pre-resolution step (the CLI's `resolveDNS`): the HTTPS request resolves, and a DNS failure is
a failed attempt.

### 3.11 CI (the part this child builds)

- `.github/workflows/ci.yaml` (built in OOL-4) gains the `live` job: only on
  `workflow_dispatch`, `TEA_LIVE=1 go test -tags live -run Live ./...`. Never on push or PR (it
  depends on an external service and data). Its permissions and action pins follow the
  workflow's existing jobs (top-level `permissions: {}`, `contents: read`, actions pinned by
  full commit SHA, `persist-credentials: false`).

## 4. Tests the coder must add

All Go tests: `gofmt -l .` empty, `go vet ./...`, `go test ./...` offline. Unit tests use
`httptest.NewServer` / `httptest.NewTLSServer` (with the test server's client, so TLS is real)
and made-up data (`example.com`, `example.invalid`, generated uuids). Timestamps in fixtures are
fixed. Each topic has its own file (`tei/tei_test.go`, `discovery_test.go`, ...).


### 4.1 Unit tests

- **9b** (discovery) Allowance covers what is under its base (D4) on learned bases: cases 21b
  and 27b.
- **11b** (discovery) Well-known over `WellKnownBytes` and discovery over `DiscoveryBytes` are
  failed attempts whose `Attempt.Err` is `ErrResponseTooLarge` (`errors.Is` through
  `*DiscoveryError`).
- **13b** (discovery) With `WithBearerToken` on a root (13's setup: every test server's origin
  allowed, `*.localhost` mapped to `127.0.0.1`), the bearer is not sent to the well-known
  location or to `/discovery` on a base other than the root, nor to a `servers[]` root on
  another host.
17. Fuzz `FuzzDecodeDiscovery` and `FuzzDecodeWellKnown`: no panic.


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
    accepted. **New in this round (Q-4):** a well-known `404` with a TEA-shaped body
    (`application/json`, `{"error":"OBJECT_UNKNOWN"}`) is a failed attempt, not
    `*NotResolvedError`: with attempts set to 2 the well-known location sees two requests and
    `Discover` returns `*DiscoveryError` at stage well-known.
24. Failover: first endpoint answers `503`, non-TEA `404`, invalid JSON, `[]`, or a result whose
    `productReleaseUuid` is `"not-a-uuid"` or the 35-character
    `"22222222-2222-4222-8222-22222222222"` (both fail to decode, 3.5), or an http rootUrl not
    under an allowance; the second endpoint is tried and its result returned. **New in this
    round (Q-4):** the same for a first endpoint answering `429` or `204`, and for a result
    whose `servers` list an allowed `https` root first and an `http` root not under an
    allowance second (the answer is refused whole; the attempt's reason names the URL policy).
    A result with an
    uppercase `productReleaseUuid` is accepted and `ProductReleaseUUID.String()` is lowercase.
    A TLS failure (a test server with an untrusted certificate) is a failed attempt.
25. No failover: TEA `404 OBJECT_UNKNOWN` gives `*NotResolvedError` with the code and no request to
    the second endpoint; same for `401` and `403` (as `*APIError`) and `400`.
26. Bound and backoff (Q-2, Q-3): four failing endpoints with attempts set to 3 make exactly 3
    requests in total, the successful well-known request and one `/discovery` request to each
    of the first two endpoints, and return `*DiscoveryError` (stage API discovery) listing those
    2 failed attempts, each with its URL without query and its reason; one failing endpoint
    with the default 4 attempts gets 3 `/discovery` requests (the candidates are tried again)
    and the `*DiscoveryError` lists 3 attempts; a failing well-known location with attempts
    set to 3 gets 3 requests, with backoffs doubling from the first; a cancelled context
    during backoff returns `context.Canceled` promptly.
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
    with one endpoint only and attempts exhausted (that endpoint asked again until the bound,
    step 6), `errors.Is(err, ErrTooManyResults)` holds on the `*DiscoveryError`. A default-limit answer of 33 results is refused the same way.
27. Results: two results are returned in array order; `SelectServer` picks per 22's rules and
    refuses an http root not under an allowance (on a `DiscoveryResult` the test builds, since
    `Discover` never returns one, 3.6 step 5), also when that root is ordered before an allowed
    `https` one (no fallback); `DiscoverPURL` sends `purl=` and not `tei=`.
    **27b** With `WithInsecureHTTPBase("http://127.0.0.1:PORT")`, `SelectServer` accepts a
    `servers[]` root `http://127.0.0.1:PORT/tea` (under the origin) and refuses
    `http://127.0.0.1:PORT2/tea` and `http://localhost:PORT/tea`.


    - **39b** (discovery) `Discover` against two endpoints where the first fails gives, in
      order, the well-known request and response, the first discovery request, a
      `TraceAttemptFailed` with the reason, a `TraceBackoff`, then the second discovery
      request and response; no event contains the TEI or a `?`.

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
3. A base whose path is wrong (`WithWellKnownLocation` pointing at a document, served by a
   local `httptest` server and allowed, that lists `<server>/nope` first and `<server>`
   second): oolong's Nest `404` body is a TEA error response by the spec's definition (3.0,
   3.6), so `Discover` returns `*NotResolvedError` with `Base` `<server>/nope`, `Code`
   `ErrorCodeUnknown` and `RawCode` `Not Found`, and makes no `/discovery` request to the
   second endpoint (Q-1; OOL-3's round 2 expected a failed attempt here, which the spec's rule
   does not allow). Non-TEA `404`s that do fail over are case 24.

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

### 4.4 Suites and checks the coder runs before hand-over

`gofmt -l .` (empty), `go vet ./...`, `go test ./...`, `go test -race ./...`,
`go generate ./... && git diff --exit-code` (which runs the generator from `tools.mod`),
`go mod tidy -diff` (empty), each fuzz target for 30 s
(`go test -fuzz=FuzzParse -fuzztime=30s ./tei` and the others), the integration suite against a
locally built oolong at the `2026-11-oolong` head (name the commit in the PR), and the live suite
once (name the date). The ASCII check of the coding principles over the hand-written files.

## 5. Risks and open points

- **No split: this round fits one coder pass.** PR #2 already builds the whole child; the
  answers need no code change and three test sub-cases (cases 23 and 24). Signed off with
  `--no-split`.
- **Oolong's framework 404 stops discovery (consequence of Q-1, a server matter, not this
  task's).** Because oolong answers every unmounted path with a JSON body carrying a string
  `error` (`Not Found`), a well-known document that lists an oolong base with a wrong path
  makes a conforming client stop with "not resolved" at that base instead of failing over to
  the next endpoint. The client cannot tell that body from a later TEA version's error
  without breaking the spec's forward-compatibility rule, so the fix, if one is wanted,
  belongs in oolong: answer unmatched routes with a non-TEA `404` (for example
  `{"message":"Not Found"}` without `error`, or no JSON body), so that a wrong path fails over
  as "Failover, invalid documents, and attempt bounds" describes ("a web server answering for
  an unmounted path"). Out of scope here (OOL-3 changes nothing in oolong); noted for the
  operator in the sign-off. Integration case 3 pins the current oolong behaviour and will
  fail, naming the oolong commit, if oolong changes it; it is then updated to expect a failed
  attempt and the second endpoint's result.
- **Repeating failures are retried (Q-3).** A candidate that answers with a body the client
  refuses is asked again while attempts remain. With the default bound that is at most three
  `/discovery` requests and under 2 s of backoff; the bound and backoff are what the spec asks
  for, and they cap a hostile document's cost as before (threat model T1, T7).
- **A partly unusable answer is refused whole (Q-4).** A deployment that lists an `http` root
  next to `https` ones is non-conforming; a client without an allowance for it fails over or
  ends in `*DiscoveryError` instead of using the `https` root. Chosen to match step 2 and to
  fail closed on non-conforming answers; the attempt's reason names the URL policy, so the
  cause is visible.
- **Interpretation of the http allowance (D4), a stated departure from the letter of
  `discovery/readme.md`.** Carried unchanged from OOL-3's round 2, section 5: an allowance
  covers URLs under its base (path segments, no dot segments), so that the CLI's `--usehttp`
  keeps working for a server whose API root is a path on its origin; nothing a server sends
  creates or widens one, and a learned base can never reach a scheme, host or port over http
  that the caller did not name. The comment at the `under` helper names the spec section and
  this departure.
- **Time bounds are defaults, not guarantees for a caller who disables them.** A negative
  `AttemptTimeout` leaves only the caller's context (OOL-3, section 5).
- **Threat model unchanged.** No answer adds an input, an asset or a trust boundary, or
  changes a mitigation: Q-1 makes one more server answer a stop, which a hostile endpoint
  could already cause with `OBJECT_UNKNOWN` (T7's trust model: a TEI's domain decides where
  its data comes from); Q-2 and Q-3 keep the attempt bound and backoff of T1 and T7 (Q-2 is
  what makes T1's time bound hold); Q-4 refuses more answers, never fewer, and `SelectServer`
  still applies D4 to the root it returns (T5). The threat model for this child stays OOL-3's
  round 2 (`design/OOL-3/threat_model-2.md`).
- **OOL-6 (OOL-3c) is not affected.** It builds from OOL-3's round 2 and nothing it builds
  changes: `FullFlow` returns `Discover`'s errors as they are (on a wrong oolong path, now
  `*NotResolvedError`), and `Candidates` calls `SelectServer` on results from `Discover`, whose
  roots all pass D4 after Q-4. Its test cases (4.1 13c, 14b, 28-38b; 4.2 cases 4, 7, 8; 4.3
  cases 2-4) are unchanged.
- No open questions for the operator.
