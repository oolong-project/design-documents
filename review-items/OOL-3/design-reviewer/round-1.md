# OOL-3 design review, round 1

Task: OOL-3, tea-client-go: Go client library for TEA v1.0.0. Reviewed:
`design/OOL-3/architecture-1.md` and `design/OOL-3/threat_model-1.md` (round 1) against
oolong-project/oolong `2026-11-oolong` @ `0fac816`, oolong-project/tea-client-go
`2026-11-oolong` (LICENSE only), the TEA specification at
CycloneDX/transparency-exchange-api `962a0228af1bfcc1232ac3f79239317ff33fdb04`
(`spec/openapi.yaml`, blob `c96baa3e...` as pinned in oolong `tea-spec/SPEC_PIN.md`;
`discovery/readme.md`), relizaio/rearm-cli @ `82f908b` (`cmd/tea.go`), relizaio/rearm-client-go
@ `7dc78ac`, and the coding principles at `145008cdcc23`.

Verdict: REJECTED. The design is thorough and mostly right; the items below are four
medium ones (one credential path D5 does not yet close, missing time bounds, the http
allowance not covering what the CLI's `--usehttp` needs, and a split whose test assignment
does not match the sections each child builds) and five low ones. What I checked and accept:

- Function. The 28 operation ids of 3.0 match `tea-spec/openapi.yaml` exactly, as do the
  closed sets, the checksum rules, `artifact-format.url` semantics, the `/token` profile and
  the `server-info`/`discovery-info` patterns. The flow covers everything `cmd/tea.go` does
  (well-known, endpoint selection, discovery, disambiguation, product release, CLE, pinned and
  unpinned components) and moves the stdin prompt out of the library (D7). The legacy
  `urn:tei:` decision (explicit `ParseLegacyURN`/`ParseCompat`, never implicit) is right and
  answers the task's question. Generator choice (oapi-codegen, generated `teaapi` checked in,
  spec pinned by blob id with a test, drift check in CI) meets the task. Oolong facts in 3.0
  (env vars `PORT`, `SERVER_HOST`, `CONTENT_DIR`, `PAGE_TOKEN_SECRET`, `NODE_ENV`,
  `API_VERSION`; `oolong/fixtures/tea-1.0.0`; fixture ids, names and sizes in
  `fixture-reference.md`) match the code. The integration harness owns its data
  (`t.TempDir()` copy), and `TEA_INTEGRATION_REQUIRED=1` stops CI from passing on a skip.
  The live test is opt-in by tag and env, and CI runs it only on `workflow_dispatch`.
- Security. Bounded reads after transport decoding with `Content-Length` pre-check (D3),
  redirect cap, page cap, discovery attempt bound, component cap; https only with no
  `InsecureSkipVerify` option (D4); credentials bound to one root and evaluated per hop (D5);
  strong-checksum-required streaming verification that fails closed, versioned endpoint so
  bytes and checksums belong to one revision, `Repr-Digest` not trusted, other content codings
  refused, temp-file-and-rename for `DownloadArtifactToFile`, no path from
  `Content-Disposition` (3.8); sanitised and truncated server text in errors (3.6). The
  threat model fits the exposure, and its T1 to T12 map to tests.
- The http allowance (D4, section 5's open point). `discovery/readme.md` says the allowance
  "cannot apply to a base the client learned from discovery". The design allows a learned
  base over http only when it is byte-equal to a base the caller configured. A hostile
  document cannot widen that, and it is what testing against oolong needs, so I accept the
  reading. The design names it as a departure in section 5; the coding principles also want a
  comment at the site with the spec location, which the coder adds. F-3 is about the CLI
  mapping, not about this reading.

## F-1: Basic credentials for /token can follow a redirect off the root (OPEN, security, ARCHITECTURE section 3.7 and THREAT_MODEL T4)

Security. D5 puts the bearer token on each hop from the transport, so Go's redirect rules
never see it. 3.7 does not say the same for the token request. It says the client "posts ... to
`<root>/v<version>/token` with `Authorization: Basic ...`". If that header is set on the
request itself, as the generated `RequestToken` and a plain `http.NewRequest` would do, Go's
`http.Client` copies it to every redirect target on the same host (any path, any port) or on a
subdomain. A `/token` that answers `307` (or `302`/`303`, which Go then sends as a `GET`) to
`https://<root host>:8443/x` or to a subdomain receives the API key id and secret. The root
rule of D5 is meant to stop exactly this. Test 14 checks the token flow but never redirects
`/token`, and T4 lists no such case.

The design must say one of these:

- the Basic header is added by the same root-scoped transport rule as the bearer, and an
  `Authorization` header already on a hop outside the root is removed; or
- the token request follows no redirects, so any `3xx` from `/token` is an error.

Also name what applies to `API.RequestToken(ctx, keyID, secret)`, the exported call that
takes raw credentials. Add a case to 4.1 test 14: `/token` answers `307` to another port on
the same host and `302` to a subdomain test server, and the second server sees no
`Authorization` header. Add that case to T4.

## F-2: No time bound on a stalled server, and discovery's attempt timeout is undefined (OPEN, security, ARCHITECTURE sections 3.4 and 3.6, THREAT_MODEL T1)

Security (resource use). The coding principles section "The Go client treats every server as
untrusted" asks the client to bound what it reads. Byte limits do not bound time. The default
transport sets only `TLSHandshakeTimeout` and `ResponseHeaderTimeout`, and 3.4 removes any
whole-request timeout. So a hostile well-known, discovery or API server that sends headers and
then one byte a minute holds the call forever, unless the caller's context has a deadline.
The CLI's contexts do not have one today; its own client uses `http.Client.Timeout = 30s` on
every call. With `WithHTTPClient`, even the header timeout is whatever the caller's transport
has. 3.6 step 5 counts "timeout of the attempt" as a failed attempt, but no attempt timeout is
defined anywhere (not in `Limits`, not in `WithDiscoveryAttempts`), so a coder has to invent
it. One stalled endpoint then blocks failover to the next one, which is what the attempt bound
exists for.

The design must define:

- a per-attempt timeout for discovery (well-known and `/discovery`), with a default and an
  option;
- a default deadline for every non-download JSON call (for example a `Limits.CallTimeout`),
  applied with `context.WithTimeout` when the caller's context has no earlier deadline;
- for downloads, an idle timeout (no bytes for N seconds), not a whole-request timeout;
- whether these also apply when the caller supplies its own `http.Client`. They should,
  because they are context-based.

Add tests: a well-known server that sends headers and then stalls the body is a failed attempt
within the attempt timeout, and the next candidate is tried. A JSON call against a stalled body
returns a deadline error within the default. A download that stops sending bytes fails after
the idle timeout. T1 should name slow responses.

## F-3: The exact http allowance does not cover what --usehttp does today, and "under a base" is undefined (OPEN, functional, ARCHITECTURE sections 3.10 and D4)

Functional. In 3.10, `--usehttp`/`--useport` becomes
`WithWellKnownLocation(domain, "http://host:port")` + `WithInsecureHTTPBase`. Today
`--usehttp` makes the CLI use http for the well-known fetch and for every URL it learns after
it. A local ReARM serves `.well-known/tea` at `http://localhost:8086` and lists its endpoint as
`http://localhost:8086/tea`. Under D4's exact match, that endpoint, and the `servers[].rootUrl`
discovery returns for it, are refused unless the caller also configured
`http://localhost:8086/tea`. The CLI cannot know that before it fetches the well-known
document. The later CLI switch would then break `--usehttp` for every server whose API root
is not its origin. Oolong works only because `SERVER_HOST` is both the well-known base and
the root.

The design must say how the CLI keeps `--usehttp` working within D4. Options: the CLI takes the
API base as a flag (`--tea-base`) and passes both strings; or a documented allowance form for
an origin (`http://host:port`, which allows exactly that origin and no other), with the
trade-off stated against the spec's "per configured base URL". Whichever it picks, 3.10 and
README say it, and a test covers the local-ReARM shape (well-known at the origin, endpoint at
`/tea`).

Also, D4 says a URL must be "exactly equal" to a configured base, while test 9 allows requests
"under" it. Define "under" as the base's path followed by `/` or the end of the path (path
segments, so `/tea` does not allow `/teapot`), the same as D5's root rule. Add the `/teapot`
case to test 9.

## F-4: The split's test assignment does not match the sections each child builds (OPEN, functional, ARCHITECTURE section 5)

Functional (buildability of the split). OOL-3a builds 3.1 to 3.5, the error part of 3.6, and
`WithBearerToken`. It is assigned 4.1 cases 1 to 13, but several of those cases exercise code
that only OOL-3b or OOL-3c builds:

- case 9: "a discovered `http` root that differs by path" (discovery, OOL-3b);
- case 11: "well-known over 64 KiB and discovery over 1 MiB" (OOL-3b);
- case 13: the bearer not sent to "an external artifact `url`" (download, OOL-3c), "the
  well-known location" or "a `servers[]` root on another host" (OOL-3b).

The 3a coder cannot write these, and the 3b and 3c children have no instruction to add them.
They are the T4 and T5 refusals, so they must not fall between children. Split cases 9, 11
and 13 into sub-cases, and assign each to the child that builds the code it tests. Also check
the threat model's test table still holds per child.

## F-5: Validation and limits the generated layer cannot see are not specified (OPEN, functional, ARCHITECTURE sections D3, 3.5 and 3.6)

Functional (buildable without guessing). Two places assume information the chosen generated
types do not carry:

- 3.5 checks that "a uuid field used later in a path matches the spec's lowercase pattern", and
  3.6 step 5 and test 24 refuse a result with "a bad uuid". But `openapi_types.UUID` is
  `google/uuid`'s `UUID`, whose `UnmarshalText` accepts uppercase, `{...}` and `urn:uuid:`
  forms. After decoding, the original string is gone, and a uuid is always re-formatted
  lowercase. Say which you mean: validate the raw JSON string (a second decode of the fields
  concerned into strings), or accept anything `google/uuid` parses, since the path is rebuilt
  canonically. Then make test 24's "bad uuid" a value that the chosen rule actually refuses.
- D3 says the doer applies "the limit for that kind of call", but a doer sees only an
  `*http.Request`. Say how the kind reaches it (for example, a context value each wrapper
  sets) and which limit applies to calls made through `Generated()`. That includes the two
  unwrapped download operations, which read the body into `[]byte`; I would apply
  `DownloadBytes` to them and `JSONBytes` to the rest. Add a test that a `Generated()` call is
  bounded.

## F-6: Candidates walks every discovery result with no cap (OPEN, security, ARCHITECTURE section 3.9)

Security (bounded work). `Candidates` selects a server and fetches the product release for
each discovery result. The only bound on the number of results is `DiscoveryBytes` (1 MiB),
which allows several thousand. A hostile discovery answer can therefore make one `FullFlow`
with a `Select` callback issue thousands of requests, to `servers[]` hosts it chooses. The flow
caps components (`FlowComponents`) but not results. Add a `Limits` field for discovery
results, refused with an error (no silent truncation, as for components), and a test.

## F-7: The live test asserts on data, not structure (OPEN, functional, ARCHITECTURE section 4.3 case 1)

Functional (test list). 4.3 case 1 asserts "the TEI's uuid is among the results (it is the
identifier itself)". Nothing in the spec says a `uuid` TEI's id is the product release uuid. It
is how ReARM issues its TEIs today, so it is data that may change, which the task asks the
live test not to assert. Drop that assertion, or say why it is part of the TEA contract.

## F-8: Licensing of the copied spec files and of anything taken from rearm-client-go is not stated (OPEN, functional, ARCHITECTURE sections 3.1 and 3.11)

Functional (public repository). `spec/openapi.yaml` and `spec/tea-well-known.schema.json` are
Apache-2.0 (the upstream LICENSE; `info.license` in the document). The generated `teaapi`
carries their descriptions as comments. tea-client-go is MIT. Apache-2.0 section 4 asks for a
copy of the licence with redistributed files. Say that `spec/` carries the upstream LICENSE
(for example `spec/LICENSE`), with `SPEC_PIN.md` and the README naming the licence of
`spec/`. Also say whether anything is copied verbatim from rearm-client-go (for example the CI
workflow); if so, it keeps that project's MIT copyright notice. If it is only modelled on it,
say so.

## F-9: Module dependencies and versioning are not stated accurately (OPEN, functional, ARCHITECTURE sections 3.1 and 3.4)

Functional (public API stability). 3.1 says the dependencies are the two runtime modules,
`x/crypto` and `x/mod`, "No others". A `tool` directive in the main `go.mod` adds oapi-codegen
and its whole requirement graph (kin-openapi, yaml, x/tools, ...) to the module's requirements.
Through MVS, that graph can raise versions in every consumer's build, including the CLI's.
Either state that and accept it (rearm-client-go's `tools.go` does the same), or keep the
generator in a separate modfile (`go tool -modfile=tools/go.mod oapi-codegen` in
`generate.go`).

Also state the release policy. The coding principles section "The Go client treats every
server as untrusted" treats every exported identifier as public API. Since `teaapi` is
exported, a re-pin or a generator bump is a breaking change. Say: tags `v0.x` until the CLI
switch, and what a regenerate means for the version. Also say how
`WithUserAgent`'s default `<module version>` is obtained (`debug.ReadBuildInfo` reports
`(devel)` in the module's own tests).
