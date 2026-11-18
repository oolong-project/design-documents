# OOL-7 architecture, round 1: TEA /token and Bearer authentication (all-or-nothing), API keys in a YAML file, interop tests in tea-client-go

Task: OOL-7. Coding principles: `coding-principles/coding_principles.md` at commit
`145008cdcc23`. Code read at oolong-project/oolong `2026-11-oolong` @ `77a17b3` and
oolong-project/tea-client-go `2026-11-oolong` @ `e848589`. Spec: the pinned TEA spec at
`962a0228af1bfcc1232ac3f79239317ff33fdb04` (`tea-spec/openapi.yaml`, byte-identical to
tea-client-go `spec/openapi.yaml`; `discovery/readme.md` "Authentication and authorization";
`auth/readme.md` at the same commit).

The operator's decisions recorded in the task description are requirements here; this design
does not reopen them. Where it adds a decision of its own it is numbered (D1...) with the reason.

## 1. Problem

Oolong serves every TEA endpoint to anyone who can reach it; a publisher with non-public
supply-chain data must put an ingress in front and invent its own authentication, which no TEA
client speaks. TEA 1.0 defines one interoperable baseline: `POST /token` exchanges an API key
(HTTP Basic, `client_credentials`) for an opaque access token, resource endpoints take
`Authorization: Bearer` and answer `401` with a Bearer challenge. This task adds that baseline
to oolong as an all-or-nothing switch (`auth.mode none | required`), with API keys in a
versioned YAML file mounted from a Kubernetes Secret or ConfigMap, and proves it against the
existing client side of tea-client-go (`WithAPIKey`) with interop tests.

Out of scope:

- Per-product (or any per-object) authorization. v1: any valid key reads everything. The
  design leaves one decision point and a versioned key file so it can come later without
  changing tokens or `/token` (3.5).
- Mixed public/protected endpoints, optional grant types (SAML, JWT assertions, mTLS, token
  exchange), credentials in the request body (`client_id`/`client_secret` form parameters,
  "may" in the spec), refresh tokens ("should not"), a revocation endpoint, RFC 9728 metadata.
- Rate limiting of `/token` (risk in 5; the endpoint is cheap by construction, 3.4).
- Password-style (operator-chosen) secrets and a slow hash (argon2id): the `sha256:` prefix
  leaves room (3.2).
- Any change to the Go client library's code or public API (3.8): its client side is done.
- TLS termination in oolong: TLS stays at the ingress, as today (3.6, 3.7).

## 2. Repositories and branches

- oolong-project/oolong, branch `2026-11-oolong` (PR against it):
  - server, new: `oolong/oolong-server/src/tea/auth/` (`auth-config.ts`, `api-keys.ts`,
    `credentials.ts`, `access-token.ts`, `access-policy.ts`, `auth.guard.ts`,
    `token-endpoint.ts`, `auth-log.ts`, `keygen.ts`, and a spec file for each, 4.1)
  - server, changed: `src/app.module.ts`, `src/tea/create-app.ts`, `src/tea/tea.controller.ts`,
    `src/tea/well-known.controller.ts`, `src/tea/health.controller.ts`, `src/tea/errors.ts`,
    `src/tea/representation.ts`, `src/tea/tea.service.ts`, `package.json` (one script)
  - server tests: new `test/auth.e2e-spec.ts`, `test/auth-none.e2e-spec.ts`; existing suites
    unchanged (they are the `none` regression)
  - chart: `helm-chart/oolong/values.yaml`, `values.schema.json`, `templates/deployment.yaml`,
    `templates/_helpers.tpl`, `templates/NOTES.txt`, new `templates/auth-token-secret.yaml`,
    `scripts/chart-render.integration.sh`
  - docs: `README.md`, `oolong/oolong-server/IMPLEMENTATION_NOTES.md` (3.9 lists every place)
- oolong-project/tea-client-go, branch `2026-11-oolong` (PR against it):
  - `internal/oolongtest/oolongtest.go` (auth-enabled server start), new `export_test.go`
    (test clock seam), new `integration_auth_test.go` (build tag `integration`). No change to
    any non-test file outside `internal/oolongtest`.

`docs/tea-upgrade/` named by the role prompt no longer exists on `2026-11-oolong`: commit
`3b6e456` folded the audit and the security assessment into
`oolong/oolong-server/IMPLEMENTATION_NOTES.md`. The security assessment for this feature goes
there (3.9).

## 3. Design

### 3.1 Modes and what each endpoint does

`AUTH_MODE` (environment) is `none` (default when unset) or `required`; anything else stops
startup. The chart sets it from `auth.mode`.

| Request | `none` (exactly today) | `required` |
|---|---|---|
| `GET`/`HEAD /.well-known/tea`, `GET /healthz` | as today | as today; never 401; a presented `Authorization` is ignored |
| `POST /v1.0.0/token` | `404 {"error":"NOT_IMPLEMENTED"}` whatever the headers and body; never 401 | token endpoint (3.4) |
| every other mounted TEA route, `GET` and `HEAD`, `/discovery` and all `.../download` and `.../signature/download` included | as today; `Authorization` ignored, never parsed | authentication runs first (guard, 3.5): no or bad credential is `401` before any parameter validation, page-token check, snapshot read or object lookup; with a valid token the request runs exactly as today |
| unmatched paths (`GET /v1.0.0/nope`, `OPTIONS` on any path) | Nest 404 as today | Nest 404 as today (no route, so no guard; the route table is the public spec, so the 404 leaks nothing) |

Resource `401` responses (`required` only):

- No `Authorization` header, or a scheme other than `Bearer` (an API key sent as `Basic` to a
  resource endpoint included): `401`, `WWW-Authenticate: Bearer realm="tea"` (no `error`
  attribute, RFC 6750 section 3.1: the request carried no bearer credential).
- `Bearer` scheme with a token that is malformed, has a bad MAC, is expired, or whose key is no
  longer present, enabled and unexpired; or more than one `Authorization` header (D10):
  `401`, `WWW-Authenticate: Bearer realm="tea", error="invalid_token"`.
- Both: no body (`Content-Length: 0`), `Cache-Control: no-store`. D1: the spec's
  `401-unauthorized` body is optional and, when present, must be an `error-response` whose
  `error` is from `unknown-error-type`; no value of that enum means "unauthorized", so the only
  spec-conforming body is none. Same for `403` (3.5). No `error_description` (D2: it would tell
  a token holder whether its key was revoked, disabled or expired; the client needs only the
  `invalid_token` code).

Express routes case-insensitively and with an optional trailing slash (prototyped:
`/v1.0.0/PRODUCT/x` and `/v1.0.0/token/` reach the same handlers). Authentication is therefore
a Nest global guard keyed on the matched handler, not a path-prefix middleware (D3): whatever
URL spelling reaches a handler, the guard runs for it. New routes are protected by default; a
route is public only by an explicit `@PublicRoute()` marker (3.5).

`/token` answers `200`, not Nest's `@Post` default `201` (prototyped: the current controller
would answer `201`). The handler writes its own response (3.4).

### 3.2 API keys: format, key file, keygen

**Key id.** `^[A-Za-z0-9][A-Za-z0-9._-]{0,63}$` (`KEY_ID_PATTERN`). Not secret. These
characters are not changed by `application/x-www-form-urlencoded` encoding (Go's
`url.QueryEscape` in tea-client-go `auth.go` leaves them as they are), so ids interoperate
whether or not a client encodes them.

**Secret.** `oolong_sk_` followed by exactly 43 base64url characters
(`^oolong_sk_[A-Za-z0-9_-]{43}$`, `SECRET_PATTERN`), the unpadded base64url of 32 bytes from
`crypto.randomBytes`. The fixed prefix lets secret scanners find leaked keys. `/token` refuses
anything else before hashing (3.4).

**Hash.** `sha256:` followed by 64 lowercase hex characters, the SHA-256 of the UTF-8 secret
string (prefix included). SHA-256 and not a password hash, as decided: the secret is 256 random
bits minted by keygen, so an offline guess is infeasible and a slow hash would only turn
`/token` into a CPU and memory lever. The `sha256:` prefix is the hash type; `version: 1`
accepts only `sha256`. A later `argon2id:` type (operator-chosen secrets) would have to refuse a
ConfigMap source (5).

**Keygen** (`src/tea/auth/keygen.ts`, compiled to `dist/tea/auth/keygen.js`; npm script
`"keygen": "node dist/tea/auth/keygen.js"`): the only supported way to mint a key.

```
node dist/tea/auth/keygen.js [--id <key id>]
```

- `--id` must match `KEY_ID_PATTERN`, else exit 2 with `keygen: --id must match <pattern>` (the
  pattern, not the value). Without `--id` the id is `key-` plus 16 lowercase hex characters
  from `crypto.randomBytes(8)`. Any other argument: exit 2 with a usage line.
- stdout, exactly three lines, nothing else:

  ```
  id: key-1a2b3c4d5e6f7a8b
  secret: oolong_sk_<43 base64url characters>
  hash: sha256:<64 lowercase hex>
  ```

  (The `id:` and `hash:` lines are valid key-file YAML for one entry's fields.) stderr, one line:
  `keygen: store the secret now; it is not stored anywhere and is not shown again. Add the id
  and hash lines to the key file.` The secret appears only on stdout, once.
- Keygen reads no environment, no file and no network; it imports only `crypto` and the
  format constants from `credentials.ts` (it must not import `create-app.ts`, so it runs without
  `PAGE_TOKEN_SECRET`). In the container: `node dist/tea/auth/keygen.js` in the image's
  working directory `/app`.

**Key file, version 1.** UTF-8 YAML, at most 64 KiB, at the path `AUTH_KEYS_FILE`:

```yaml
version: 1
keys:
  - id: ci-pipeline
    hash: sha256:3f1d0c5e...            # 64 lowercase hex
    description: CI pipeline of the example product   # optional
    expiresAt: "2027-06-30T00:00:00Z"   # optional, RFC 3339
    disabled: false                      # optional, default false
```

Parsing and validation (`parseKeyFile(bytes: Buffer): KeySet` in `api-keys.ts`), all or
nothing (coding principles "Content reloads are all or nothing" applied to the key file): any
violation refuses the whole file.

- Size over 65536 bytes, or not valid UTF-8 (`new TextDecoder('utf-8', { fatal: true })`):
  refused.
- `yaml.load(text, { schema: yaml.CORE_SCHEMA })` (js-yaml 4, already a dependency). Prototyped
  with the installed js-yaml: `CORE_SCHEMA` keeps `2027-06-30T00:00:00Z` a string (the default
  schema turns it into a `Date`), keeps `yes` a string, has no `<<` merge key (it stays an
  ordinary key and is then refused as unknown), and duplicate mapping keys throw.
- Top level: a mapping with exactly the keys `version` and `keys`. `version` is the number `1`
  (a string `"1"` or `2` is refused: a v1 server must not half-apply a newer schema's rules).
  `keys` is a sequence of at most 1000 mappings; an empty sequence is valid (no key works).
- Each entry: only `id`, `hash`, `description`, `expiresAt`, `disabled` (an unknown field, for
  example `secret`, `scope` or `products`, is refused). `id` (string, `KEY_ID_PATTERN`, unique
  in the file); `hash` (string, `^sha256:[0-9a-f]{64}$`; another type prefix is refused as
  `unsupported hash type`); `description` (string, at most 200 characters, no control
  characters); `expiresAt` (string, RFC 3339 `date-time` with seconds and `Z` or an offset,
  calendar-valid: `2026-02-30T00:00:00Z` is refused; normalized to a UTC instant, coding
  principles "Timestamps are UTC instants in RFC 3339"); `disabled` (boolean).
- `KeySet` is `ReadonlyMap<string, ApiKeyRecord>` with
  `ApiKeyRecord = { id: string; digest: Buffer /* 32 bytes */; expiresAt?: number /* ms */;
  disabled: boolean; description?: string }`. The hex is decoded once at load.

Errors are a `KeyFileError` whose message names the rule and the location only, for example
`keys[2].hash: expected sha256:<64 lowercase hex>` or `yaml: duplicated mapping key at line 7`.
D4: a js-yaml `YAMLException.message` contains a snippet of the offending lines (prototyped), so
only its `reason` and `mark.line + 1` are used. No value from the file ever reaches a message,
because the message is what gets logged (3.6) and a mistaken file may hold a secret.

**Reload** (`ApiKeyRegistry`, `@Injectable`, in `api-keys.ts`). In `required` mode the
constructor reads and parses the file; any error stops startup (fail closed: a server that
requires auth and cannot read its keys does not start). It then polls every
`AUTH_KEYS_RELOAD_INTERVAL_MS` (default 10000) with `setInterval` (cleared in
`onModuleDestroy`), reading the whole file (`fs.readFileSync`, which follows symlinks) and
comparing the SHA-256 of the bytes with the last accepted file's:

- unchanged: nothing;
- changed and valid: the new `KeySet` replaces the old one in one assignment (requests read
  `current()` once per check);
- changed and invalid, unreadable or missing: the last good `KeySet` stays, one log line with
  the reason (3.6), logged once per distinct failure (the bytes' hash, or the error code for a
  read failure) so a bad file does not log every interval.

D5, polling and not `fs.watch`: a Kubernetes Secret or ConfigMap volume publishes an update by
writing a new timestamped directory and renaming a `..data` symlink over the old one; the file
name is a symlink through `..data`. Prototyped with that layout in Node 24: an `fs.watch` on the
file fires for the first swap and then never again (the watch follows the replaced inode); a
directory watch sees the renames but needs debouncing and re-reading anyway. Reading a file of
at most 64 KiB every 10 s costs nothing, works the same for Docker bind mounts and editors that
replace files, and matches how `ContentSnapshotRegistry` already polls. `poll()` is public and
marked "visible for tests", as in `content-snapshot.ts`.

The chart mounts the whole volume (no `subPath`, which kubelet never updates); revocation
latency is the kubelet sync delay (typically about a minute, longer on some clusters) plus at
most one poll interval. For an immediate cut-off, restart the pods (3.7, 3.9).

### 3.3 Access tokens

Stateless, HMAC-signed, as decided. Format (opaque to clients, spec `auth/readme.md` "The
access token is opaque"):

```
oat1.<payload>.<mac>
payload = base64url(JSON {"v":1,"kid":"<key id>","iat":<unix seconds>,"exp":<unix seconds>})
mac     = base64url(HMAC-SHA256(AUTH_TOKEN_SECRET, "oolong-access-token-v1." + payload))   (43 chars)
```

- Issue (`AccessTokens.issue(keyId, now)` in `access-token.ts`): `iat = floor(now / 1000)`,
  `exp = iat + AUTH_TOKEN_TTL_SECONDS`. Two tokens issued for one key in the same second are
  identical; that is harmless.
- D6, domain separation: the MAC input starts with `oolong-access-token-v1.`, and the key is a
  separate secret that startup refuses when equal to `PAGE_TOKEN_SECRET` (3.6). A page token
  (`pagination.ts` signs `body` alone) can never verify as an access token or the reverse,
  even with one key.
- Verify (`AccessTokens.verify(token, now): string /* key id */ | undefined`), in this order,
  `undefined` on the first failure: length at most 512; matches
  `^oat1\.[A-Za-z0-9_-]{1,400}\.[A-Za-z0-9_-]{43}$`; recomputed MAC equal under
  `crypto.timingSafeEqual` (both 32 bytes after decoding); payload decodes to a JSON object
  with exactly the keys `v`, `kid`, `iat`, `exp`; `v === 1`; `kid` matches `KEY_ID_PATTERN`;
  `iat` and `exp` are safe integers; `exp > iat`; `exp - iat <= 86400`;
  `iat <= nowSeconds + 60` (clock skew between replicas); `nowSeconds < exp`. The regexes are
  linear (coding principles "Bound the work...").
- The token carries only the key id (as decided). Every authenticated request then requires the
  key to be in the current `KeySet`, not `disabled`, and `expiresAt` unset or later than now
  (3.5). Disabling, removing or expiring a key therefore ends its live tokens at the next
  reload. Replacing a key's `hash` under the same id does not end tokens already issued for it
  (they stay valid up to their `exp`): rotation is "mint a new id, then disable or remove the
  old one", and the docs say so (3.9, threat model T6).
- The clock is injected (`now: () => number`, default `Date.now`) into `AccessTokens`,
  `ApiKeyRegistry` checks and `TokenEndpoint`, so unit tests can expire tokens and keys without
  waiting.

### 3.4 `POST /v1.0.0/token`

`TeaController.token` (line 253 today) becomes
`@PublicRoute() @Post('token') token(@Req() req, @Res() res): Promise<void>` delegating to
`TokenEndpoint.handle(req, res)` (`token-endpoint.ts`). In `none` mode `handle` throws
`new TeaException(404, 'NOT_IMPLEMENTED')` before looking at headers or body, exactly the
response of today.

**Body parsing.** D7: `NestFactory.create(AppModule, { logger: false, bodyParser: false })`.
No route other than `/token` takes a body, and Nest's global parsers would otherwise read up to
100 KB of any request before the handler. Prototyped on the current branch: a malformed JSON
body or a 200 KB urlencoded body posted to `/token` is answered `500 Internal server error`
today (the body-parser error is not an `HttpException` the filter passes through). With
`bodyParser: false`, in `none` mode those requests get the `404 NOT_IMPLEMENTED` the operator
asked for; this is the only response change in `none` mode, and it is the "never anything but
404 NOT_IMPLEMENTED" behaviour the description states. `/token` reads its own body
(`readTokenForm(req)` in `token-endpoint.ts`), so no path matching couples a middleware to the
route:

- `Content-Type` must be `application/x-www-form-urlencoded`, optionally with
  `charset=utf-8` (case-insensitive); otherwise `invalid_request`.
- `Content-Encoding` absent or `identity`; otherwise `invalid_request`.
- `Content-Length` over 1024: `invalid_request` without reading, with `Connection: close`.
  Without a length (chunked), stop at 1025 bytes read: `invalid_request`, `Connection: close`.
- A read that has not ended 10 s after it started: `invalid_request`, `Connection: close` (a
  slow body must not hold a socket for Node's 300 s `requestTimeout`).
- Bytes must be valid UTF-8; then `new URLSearchParams(text)`; at most 8 parameters; any name
  that occurs twice (RFC 6749 section 3.2) is `invalid_request`.

**Order of checks and responses.** Every response from `/token` in `required` mode carries
`Cache-Control: no-store` and `Content-Type: application/json; charset=utf-8`, and is written
with `sendJson` (`errors.ts`). Bodies are the spec's `token-response` or
`token-error-response`, never a TEA `error-response`.

1. `Authorization` (exactly one header, D10): scheme `Basic` (case-insensitive), one or more
   spaces, then a token68 matching `^[A-Za-z0-9+/]+={0,2}$` whose length is a multiple of 4;
   whole header at most 1024 characters. Base64-decode, require valid UTF-8, split at the first
   `:` (none: fail). Form-decode each part per RFC 6749 section 2.3.1 (`+` to space, then
   `decodeURIComponent`; a `URIError` fails). Then the id must match `KEY_ID_PATTERN` and the
   secret `SECRET_PATTERN`. Any failure here, a missing header, or another scheme:
   **`401 {"error":"invalid_client"}`, `WWW-Authenticate: Basic realm="tea"`**, with the body
   not read. Nothing is hashed for a request that fails here.
2. Read and parse the body (above): failure is **`400 {"error":"invalid_request"}`**.
3. `client_id` or `client_secret` present in the body: `400 invalid_request` (RFC 6749 section
   2.3.1: one authentication method per request; body credentials are not supported, out of
   scope in 1).
4. `grant_type` missing: `400 invalid_request`; present and not exactly `client_credentials`:
   **`400 {"error":"unsupported_grant_type"}`**. `scope` and any other parameter are ignored
   (RFC 6749 section 3.2), and the response has no `scope`.
5. Verify the credential (`verifyApiKey(keys, id, secret, now)` in `credentials.ts`), uniform
   cost for unknown ids: `record = keys.get(id)`; `digest = sha256(secret)` (always computed);
   `expected = record ? record.digest : DUMMY_DIGEST` (32 random bytes made once at startup);
   `match = timingSafeEqual(digest, expected)`; then
   `ok = match && record !== undefined && !record.disabled && !(record.expiresAt <= now)`,
   evaluating every term (no early return between hashing and the decision). Not `ok`:
   `401 invalid_client` + `WWW-Authenticate: Basic realm="tea"`, the same bytes for unknown id,
   wrong secret, disabled and expired keys.
6. **`200 {"access_token":"oat1...","token_type":"Bearer","expires_in":<AUTH_TOKEN_TTL_SECONDS>}`**.

No `error_description` on any token error (D2). Unexpected exceptions propagate to
`TeaExceptionFilter` (generic `500`), as for every route.

### 3.5 One decision point and the guard

`access-policy.ts`:

```ts
export type Caller =
  | { kind: 'anonymous' }                 // no Authorization, or a scheme other than Bearer
  | { kind: 'invalid-token' }             // Bearer that does not verify, or its key is not active
  | { kind: 'key'; key: ApiKeyRecord };   // active key from the current KeySet
export type AccessTarget = { kind: 'api' };  // v1: the TEA API as a whole
export type AccessDecision = 'allow' | 'unauthenticated' | 'invalid-token' | 'forbidden' | 'conceal';
export class AccessPolicy {
  decide(caller: Caller, target: AccessTarget): AccessDecision;
}
```

v1 `decide`: `anonymous` is `unauthenticated`, `invalid-token` is `invalid-token`, `key` is
`allow`. `forbidden` and `conceal` are part of the closed type now (coding principles "Prefer
closed types") so the guard maps them once; v1 never returns them. Per-product rules later add
fields to a `version: 2` key file (read per request from the current `KeySet`, carried in
`ApiKeyRecord`), add `AccessTarget` kinds (`product`, ...) checked after the object is resolved,
and return `forbidden`/`conceal`; tokens and `/token` do not change, because the token names
only the key id and the permissions are read from the current file per request.

`AuthGuard` (`auth.guard.ts`, registered as `APP_GUARD` in `AppModule`; `@PublicRoute()` is
`SetMetadata('teaPublicRoute', true)`, read with `Reflector.getAllAndOverride` over handler and
class):

1. `none` mode: `return true` without reading any header (Bearer ignored, spec discovery
   readme "shall ignore, rather than reject").
2. Public route (`WellKnownController.tea`, `HealthController.healthz`, `TeaController.token`):
   `return true`.
3. `caller = authenticate(req)`: more than one `Authorization` in `req.rawHeaders` is
   `invalid-token` (D10: Node keeps only the first, prototyped; a proxy might use another);
   none, or a scheme other than `Bearer`, is `anonymous`; otherwise `AccessTokens.verify`, then
   the key lookup and active check against `ApiKeyRegistry.current()` and the clock.
4. `decide(caller, { kind: 'api' })` and map: `allow`: set `req.teaCaller = caller` and return
   true; `unauthenticated`/`invalid-token`: throw `AuthChallenge` (3.1 headers); `forbidden`:
   throw `AuthChallenge` with status 403 and no challenge; `conceal`: throw
   `new TeaException(404, 'OBJECT_UNKNOWN')`.

`AuthChallenge` (new class in `errors.ts`: `status: 401 | 403`, `challenge?: string`).
`TeaExceptionFilter` handles it before the other branches: sets `WWW-Authenticate` when given,
`Cache-Control: no-store`, `Content-Length: 0`, ends the response (HEAD alike).

Nest runs guards after middleware and before pipes and handlers, so the guard precedes every
`@Param`/`@Query` read, `singleQuery`, `readToken` and snapshot access. Prototyped on the
current branch with a stub global guard: `GET` and `HEAD /v1.0.0/product/<fixture id>`,
`/v1.0.0/product/not-a-uuid` (400 today), `/v1.0.0/PRODUCT/not-a-uuid` and the `@Res()`
download route all answered the guard's 401; `/.well-known/tea`, `/healthz` marked public
answered 200; unmatched paths and `OPTIONS` stayed Nest 404.

D8, shared caches: with `required`, versioned artifact downloads must not say `public`
(`representation.ts` `cacheControl` returns `public, max-age=31536000, immutable` today, and
`public` lets a shared cache store a response to an authenticated request, RFC 9111 section
3.5, and serve it to anyone). `cacheControl(latest, shared)` returns
`private, max-age=31536000, immutable` when `shared` is false; `TeaService` passes
`shared = (authConfig.mode === 'none')`. `latest` stays `no-cache`. JSON responses carry no
`Cache-Control` today and need none: without `public` a shared cache must not store a response
to a request with `Authorization`.

### 3.6 Configuration, startup, logging

Environment (`auth-config.ts`, `loadAuthConfig(env): AuthConfig`, called once by
`assertAuthConfig()` in `createApp` next to `assertPageTokenSecret()`, before
`NestFactory.create`; the result is provided as `AUTH_CONFIG` to the module, so later
`process.env` changes have no effect):

```ts
export type AuthConfig =
  | { mode: 'none' }
  | { mode: 'required'; keysFile: string; tokenSecret: Buffer; tokenTtlSeconds: number;
      keysReloadIntervalMs: number };
```

| Variable | Rule | Default |
|---|---|---|
| `AUTH_MODE` | `none` or `required`, exact | `none` |
| `AUTH_KEYS_FILE` | required in `required`; absolute path | none |
| `AUTH_TOKEN_SECRET` | required in `required`; at least 32 characters; not equal to `PAGE_TOKEN_SECRET` | none, no development fallback (coding principles "Secrets and credentials") |
| `AUTH_TOKEN_TTL_SECONDS` | integer `^[1-9]\d*$`, 60 to 86400 | 3600 |
| `AUTH_KEYS_RELOAD_INTERVAL_MS` | integer, 1000 to 300000 | 10000 |

In `none` mode every other `AUTH_*` variable is ignored (not validated), so `none` stays today's
behaviour. A violation throws an `Error` naming the variable and the rule, never the value.

Logging (`auth-log.ts`). The app is created with `logger: false`, which silences Nest's
`Logger` (observed: the `content snapshot active` lines of `ContentSnapshotRegistry` never
appear in the output of a local run; only `main.ts`'s `console.log` does). Operators must see
key-file rejections, so `auth-log.ts` writes single lines to stderr with `console.error`,
prefixed `auth: `. Lines, and nothing else:

- startup (`required`): `auth: mode required, key file <AUTH_KEYS_FILE>, <n> keys, token ttl <s>s`;
  `none`: nothing.
- startup warning: `SERVER_HOST` starts with `http://` and its host is not `localhost`,
  `127.0.0.1` or `[::1]`: `auth: warning: SERVER_HOST is not https; API keys and tokens
  must only travel over TLS`.
- reload: `auth: key file reloaded, <n> keys`; `auth: key file rejected, keeping <n> keys: <KeyFileError message>`.
- `/token`, known id only (the id comes from the operator's file, not from the request):
  `auth: token issued for key <id>`; `auth: token refused for key <id>: <wrong secret | disabled | expired>`.
- `/token`, everything else (missing or malformed Basic, unknown id, bad body): counted, and at
  most once per 60 s `auth: <n> token requests refused (<a> malformed, <b> unknown key, <c> bad request) in the last minute`.
  D9: no attacker-supplied text (an unknown id may be a secret pasted in the wrong field)
  reaches the log, and a flood costs one line a minute.
- Never: a secret, a hash, an `Authorization` value, a token, a request body, a YAML snippet.
  Resource `401`s are not logged.

`AppModule` providers: `AUTH_CONFIG` (`useFactory: () => loadAuthConfig(process.env)`),
`ApiKeyRegistry`, `AccessTokens`, `AccessPolicy`, `TokenEndpoint`,
`{ provide: APP_GUARD, useClass: AuthGuard }`. In `none` mode `ApiKeyRegistry` reads nothing and
starts no timer.

### 3.7 Helm chart

`values.yaml`, new block after `pageTokenSecret` (comments as shown, plain ASCII):

```yaml
# TEA authentication (TEA 1.0 /token with API keys, Bearer tokens on every other endpoint).
#   none     - no authentication; /token answers 404 NOT_IMPLEMENTED (default)
#   required - every TEA endpoint needs a Bearer token; only /.well-known/tea,
#              /healthz and POST /token are public. Serve the API over TLS only.
auth:
  mode: none
  apiKeys:
    # The key file (see README "Authentication"). Set exactly one of these when mode is
    # required; both must be empty when mode is none. The chart never creates the key file.
    # Whoever can write this Secret or ConfigMap controls who can read the API.
    existingSecret: ""
    existingConfigMap: ""
    # Key in the Secret or ConfigMap holding the key file.
    key: api-keys.yaml
    # How often the server re-reads the mounted key file (milliseconds, 1000-300000).
    reloadIntervalMs: 10000
  # Lifetime of issued access tokens (seconds, 60-86400), returned as expires_in.
  tokenTtlSeconds: 3600
  # HMAC key for access tokens. Default: the chart creates <release-fullname>-auth-token
  # once (kept on upgrade, never rotated automatically). Set existingSecret to bring your own.
  tokenSecret:
    create: true
    existingSecret: ""
    secretKey: AUTH_TOKEN_SECRET
```

D11: unlike `pageTokenSecret`, `tokenSecret` has no `value` field: a token signing key in a
values file or `--set` argument would sit in the Helm release record (coding principles
"Secrets and credentials").

`values.schema.json`, new `auth` property, `additionalProperties: false` at every level:
`mode` enum `["none","required"]`; `apiKeys.existingSecret` and `existingConfigMap` string,
maxLength 253, pattern `^([a-z0-9]([-a-z0-9.]*[a-z0-9])?)?$` (as `gitPod.auth.existingSecret`);
`apiKeys.key` minLength 1, maxLength 253, pattern `^[-._a-zA-Z0-9]+$`; `apiKeys.reloadIntervalMs`
integer 1000..300000; `tokenTtlSeconds` integer 60..86400; `tokenSecret.create` boolean,
`existingSecret` as above, `secretKey` as `apiKeys.key`. `allOf`: `mode: required` then exactly
one of the two sources has minLength 1 and the other maxLength 0 (`oneOf`, as
`gitPod.auth.ssh` does for known_hosts); `mode: none` then both maxLength 0. No value here is a
secret, so the positive patterns may echo (OOL-1 lesson applies to secret-adjacent values only).

`_helpers.tpl`:

- `oolong.requireAuth`: the same rules as the schema, enforced again in the template so
  `--skip-schema-validation` cannot bypass them (pattern of `oolong.gitAuthType`): mode not
  `none`/`required`; `required` without exactly one source; `none` with a source; empty `key`;
  `tokenSecret.create` false without `tokenSecret.existingSecret`. Each `fail` names the value
  path (`auth.apiKeys.existingSecret`), not the value.
- `oolong.requireServerEnvNotReserved`: any `.Values.env` key starting with `AUTH_` fails the
  render (`env.AUTH_MODE is set by the chart from auth.*`), like `oolong.requireGitPodEnvNotReserved`.
  Without it `env.AUTH_MODE=required` would bypass every check above.
- `oolong.authTokenSecretName`: `tokenSecret.existingSecret` or `<fullname>-auth-token`.

New `templates/auth-token-secret.yaml`: rendered only when `auth.mode` is `required`,
`tokenSecret.create` is true and `existingSecret` is empty; a copy of
`page-token-secret.yaml` (`lookup` to keep the existing value, `fail` when the key is missing
from an existing Secret, `helm.sh/resource-policy: keep`, `randAlphaNum 64` on first install)
without the `value` input.

`templates/deployment.yaml` (server container), only when `auth.mode` is `required`; in `none`
the render is unchanged (4.3 C1):

- env: `AUTH_MODE=required`, `AUTH_KEYS_FILE=/etc/oolong-api-keys/api-keys.yaml`,
  `AUTH_TOKEN_TTL_SECONDS`, `AUTH_KEYS_RELOAD_INTERVAL_MS` (quoted), and `AUTH_TOKEN_SECRET`
  from `secretKeyRef` (`oolong.authTokenSecretName`, `tokenSecret.secretKey`).
- volume `api-keys`: `secret: { secretName, items: [{ key: <apiKeys.key>, path: api-keys.yaml }],
  defaultMode: 0440, optional: false }`, or the same under `configMap: { name, ... }`. The item
  mapping exposes only the one key of the source in the pod and fixes the file name.
- mount: `name: api-keys`, `mountPath: /etc/oolong-api-keys`, `readOnly: true`, no `subPath`
  (kubelet does not update `subPath` mounts). Files are readable by uid 10001 through the
  existing `podSecurityContext.fsGroup: 10001` (as the Git pod's `git-auth` volume).
- `volumeMounts:`/`volumes:` are today emitted only under `persistence.enabled`; they become
  `if or persistence.enabled (eq auth.mode "required")` with each entry under its own condition.

`templates/NOTES.txt` line 6 ("The TEA API is unauthenticated...") becomes conditional:

- `none`: the current sentence, plus `Set auth.mode=required to require API keys (TEA /token).`
- `required`: `TEA authentication is required: every TEA endpoint needs a Bearer token from POST
  /token; /.well-known/tea and /healthz stay public. API keys are read from <Secret|ConfigMap>
  <name> (key <key>); whoever can write it controls access. A change reaches the pod after the
  kubelet sync delay (about a minute); restart the pods for an immediate cut-off. API keys and
  tokens travel in headers: expose the API only over TLS (terminate TLS at the ingress) and never
  expose the Service or pod port directly.`, and when `useTraefikDevelopment` is true:
  `WARNING: useTraefikDevelopment exposes the API over plain HTTP; API keys and tokens would cross
  the network in the clear.` The chart does not refuse that combination (local clusters).

### 3.8 tea-client-go: interop harness and tests (no library change)

The client side exists (`WithAPIKey`, `WithPreemptiveAPIKey`, `RequestToken`, the 401
recovery in `transport.go` `Do`). The child changes only test code:

- `internal/oolongtest/oolongtest.go`:
  - `Server` gains `Output() string` (the process's combined stdout/stderr, from the existing
    `lockedBuffer`; empty for `OOLONG_BASE_URL`).
  - `type Key struct { ID, Secret, Hash string }`,
    `type KeyEntry struct { Key Key; Disabled bool; ExpiresAt string }`,
    `type AuthOptions struct { KeyIDs []string; TokenTTLSeconds int /* 0: 3600 */ }`,
    `type AuthServer struct { *Server; Keys map[string]Key; KeysFile string }`.
  - `StartAuth(t, AuthOptions) *AuthServer`: needs `OOLONG_SERVER_DIR` (with only
    `OOLONG_BASE_URL` set it skips, or fails under `TEA_INTEGRATION_REQUIRED=1`, as `Start`
    does without either). Mints each key by running `node dist/tea/auth/keygen.js --id <id>`
    in that directory and parsing the three stdout lines (so the interop suite exercises the real
    keygen), writes the key file (`version: 1`) in `t.TempDir()`, and starts `dist/main.js` as
    `startNode` does plus `AUTH_MODE=required`, `AUTH_KEYS_FILE`, `AUTH_TOKEN_SECRET` (64 random
    hex characters), `AUTH_TOKEN_TTL_SECONDS`, `AUTH_KEYS_RELOAD_INTERVAL_MS=1000`.
  - `(*AuthServer).WriteKeys(t, []KeyEntry)`: rewrites the key file atomically (temp file in the
    same directory, then `os.Rename`).
- New `export_test.go` (package `tea`):
  `func SetTestClock(c *Client, now func() time.Time) { c.now = now }`, exposing the existing
  `Client.now` seam (`client.go` line 384) to the external test package. Test-only, not public
  API.
- New `integration_auth_test.go` (`//go:build integration`, package `tea_test`), cases in 4.4.

### 3.9 Docs (every place touched)

oolong `README.md`:

- line 104 ("Chart notes"): unchanged.
- line 106 ("The TEA API is unauthenticated by design..."): first two sentences replaced by:
  `By default (auth.mode=none) the TEA API is unauthenticated and intended for a public catalog:
  anyone who can reach the Service can read the mounted content. Set auth.mode=required to
  require TEA API keys (see Authentication below).` The rest of the paragraph (302 redirects)
  stays.
- new section `### Authentication` under `## Helm Chart Configuration`, after the `#### SSH
  deploy key` subsection and before `## Generation from TEA OpenAPI Specification` (line 290):
  what `required` protects and what stays public; minting a key with keygen (container command
  and the three output lines; the secret is shown once); the key file format with an example
  that uses obviously fake values (`hash: sha256:` followed by 64 `0` characters, never a real
  hash of a real secret); creating the Secret (`kubectl create secret generic oolong-api-keys
  --from-file=api-keys.yaml`) or ConfigMap and `--set auth.mode=required
  --set auth.apiKeys.existingSecret=oolong-api-keys`; **who can write the source controls
  access** (a ConfigMap is often writable by more people than a Secret; prefer a Secret and
  restrict write access with RBAC); revocation (`disabled: true`, removal or `expiresAt`) ends
  live tokens after the kubelet sync delay plus the reload interval, `kubectl rollout restart`
  for an immediate cut-off; rotation is "new id, then disable the old one" (replacing a hash
  under the same id does not end tokens already issued, up to `tokenTtlSeconds`); never put the
  key file in the content repository (it would be served as content and published in Git
  history); TLS only (terminate at the ingress, never expose the Service); the client side is
  any TEA 1.0 client, for example tea-client-go `WithAPIKey`.
- `### Run the official container image` (line 110): one paragraph and a `docker run` example
  for auth: `-e AUTH_MODE=required -e AUTH_KEYS_FILE=/run/oolong/api-keys.yaml -e
  AUTH_TOKEN_SECRET="$(openssl rand -base64 48)" -v "$PWD/api-keys.yaml:/run/oolong/api-keys.yaml:ro"`
  (with a note that a single-file bind mount follows edits only when the editor rewrites in place;
  mount the directory to be safe).

`oolong/oolong-server/IMPLEMENTATION_NOTES.md`:

- line 21 (`create-app.ts` row): add "checks the `AUTH_*` configuration, disables Nest's body
  parsers".
- Code Layout table: rows for `auth/auth-config.ts`, `auth/api-keys.ts`, `auth/credentials.ts`,
  `auth/access-token.ts`, `auth/access-policy.ts`, `auth/auth.guard.ts`,
  `auth/token-endpoint.ts`, `auth/auth-log.ts`, `auth/keygen.ts`, one line each.
- lines 55-56 ("Authentication is not implemented..."): replaced by a pointer to the new
  section: `Authentication is optional (AUTH_MODE); see [Authentication](#authentication).`
- new `### Authentication` at the end of `## API Surface` (before `## Content Format`): the
  table of 3.1, the `/token` rules of 3.4, the token format of 3.3 marked as internal (clients
  must treat it as opaque), the decision point of 3.5.
- `### Environment Variables`: the five `AUTH_*` variables of 3.6, after `PAGE_TOKEN_DEV_FALLBACK`.
- new `## Security assessment: authentication` before `## Build and Run`: the assets, trust
  boundaries and residual risks of the threat model (`design/OOL-7/threat_model-1.md` items),
  in particular: key-file source write access equals access control; revocation latency; no rate
  limit on `/token` (use the ingress); tokens are bearer tokens (TLS only); SHA-256 is safe only
  because secrets are 256-bit random from keygen.
- `## Testing`: the new specs and e2e suites.

`helm-chart/oolong/values.yaml` (3.7) and `templates/NOTES.txt` (3.7). No change to
`SECURITY.md`, the compose files or `.env.example` (they stay `none`).

## 4. Tests the coder must add

Run every touched suite (coding principles "Tests own their data and run locally"): in
`oolong/oolong-server` with Node 24 `npm ci`, `npm run lint`, `npm run build`, `npm test`
(includes the chart render spec, needs helm), `npm run test:e2e`; in tea-client-go
`gofmt -l .` (empty), `go vet ./...`, `go vet -tags integration ./...`, `go test ./...`, and the
integration suite with `OOLONG_SERVER_DIR` pointing at the built oolong of the oolong child and
`TEA_INTEGRATION_REQUIRED=1`. Every test that changes `process.env` restores it in `afterAll`
(the e2e config runs all files in one worker, `maxWorkers: 1`). Fixtures use made-up ids and
secrets minted in the test; no key value is committed except the obviously fake documentation
example.

### 4.1 oolong unit specs (`src/tea/auth/*.spec.ts`, `npm test`)

`auth-config.spec.ts`:
- A1 unset `AUTH_MODE` is `none`; `none` ignores a missing or bogus `AUTH_KEYS_FILE`,
  `AUTH_TOKEN_TTL_SECONDS=abc`.
- A2 `AUTH_MODE` `Required`, `off`, `''` (set but empty), `required ` refuse, naming `AUTH_MODE`.
- A3 `required` refuses: no `AUTH_KEYS_FILE`; relative path; no `AUTH_TOKEN_SECRET`; a 31-character
  secret; secret equal to `PAGE_TOKEN_SECRET`; TTL `59`, `86401`, `3600.5`, `0x10`, `1e3`; reload
  `999`, `300001`. Accepts TTL `60` and `86400`; default 3600 and 10000. Every thrown message
  does not contain the secret value used in the test.

`api-keys.spec.ts` (`parseKeyFile`):
- K1 a valid file with all optional fields; `keys: []` is valid and empty.
- K2 refused, each with the expected location in the message: `version` missing, `"1"`, `2`;
  unknown top-level key; unknown entry key (`secret`, `scope`); `<<` merge key; duplicate id;
  id `-x`, `a b`, 65 characters; hash uppercase hex, 63 hex, `argon2id:...` (`unsupported hash
  type`); `expiresAt` `2026-02-30T00:00:00Z`, `2026-01-01`, `tomorrow`, a YAML number;
  `disabled: yes`; description of 201 characters or with a control character; 1001 entries;
  65537 bytes; invalid UTF-8; YAML duplicate mapping key; YAML syntax error.
- K3 `expiresAt: 2027-01-01T02:00:00+02:00` (unquoted) is the instant `2027-01-01T00:00:00Z`.
- K4 validate before logging: a refused file that contains a well-formed secret
  (`oolong_sk_` + 43 characters) in an unknown field, in a bad `hash`, and on a line with a YAML
  syntax error yields messages that contain neither that string nor the line's text.

`api-keys.spec.ts` (`ApiKeyRegistry`, temporary directories, injected clock):
- R1 startup in `required` with a missing file, an unreadable file and an invalid file throws.
- R2 `poll()` after replacing the file: removed key gone, new key present, one `reloaded` line.
- R3 `poll()` after an invalid file: previous keys kept, one `rejected` line with the reason;
  a second `poll()` with the same bad bytes logs nothing more; file deleted: previous keys kept,
  one line.
- R4 Kubernetes layout: `api-keys.yaml -> ..data/api-keys.yaml`, `..data -> ..v1`; publish `..v2`
  by renaming a `..data_tmp` symlink over `..data` and deleting `..v1` (as the prototype); two
  successive swaps are both picked up by `poll()`.
- R5 unchanged bytes: no reload and no log line.

`credentials.spec.ts`:
- C1 `parseBasic`: valid; scheme `basic` and `BASIC`; several spaces; missing header; `Bearer x`;
  `Basic` with characters outside base64; length not a multiple of 4; no `:`; invalid UTF-8
  after decoding; header of 1025 characters; percent-encoded id (`key%2D1` is `key-1`); `+` in
  the id decodes to a space and then fails the id format; `%zz` fails.
- C2 secret format: 43 characters accepted; 42 and 44, wrong prefix, `=` padding, a character
  outside base64url refused; and for each refused one the hash function is not called (spy).
- C3 uniform cost: for an unknown id, a known id with the wrong secret, a disabled key and an
  expired key, `verifyApiKey` calls SHA-256 exactly once and `timingSafeEqual` exactly once
  (spies) and returns the same refusal; for the right secret it returns the record.

`access-token.spec.ts` (injected clock):
- T1 issue then verify returns the key id; `exp - iat` equals the TTL.
- T2 refused: one character of payload or MAC changed; MAC under another secret; a token whose
  MAC is computed without the `oolong-access-token-v1.` prefix (page-token construction) with the
  same secret; length 513 (and the HMAC is not computed, spy); `oat2.` prefix; payload with an
  extra key, a missing key, `v: 2`, `kid` failing the pattern, non-integer `iat`; `exp - iat` of
  86401; `iat` 61 s in the future; at `exp` exactly and after it.

`access-policy.spec.ts` and `auth.guard.spec.ts`:
- P1 `decide` matrix: anonymous, invalid-token, key.
- P2 guard in `none` mode returns true for a request with `Authorization: garbage` and does not
  call `authenticate` (spy).
- P3 guard on a public route in `required` returns true without a header.
- P4 revocation applies to live tokens: a token for key k1 is `allow`; after the registry's
  `KeySet` has k1 `disabled`, removed, or `expiresAt` in the past (clock), the same token is
  `invalid-token`.
- P5 two `Authorization` headers (in `rawHeaders`) with a valid token first: `invalid-token`.
- P6 mapping with a stub `AccessPolicy` returning `forbidden`: 403, no `WWW-Authenticate`, empty
  body; `conceal`: `404 {"error":"OBJECT_UNKNOWN"}`.

`keygen.spec.ts` (`keygen.ts` exports `mintKey(id?)` and
`runKeygen(argv, stdout, stderr): number`, and calls `runKeygen` only under
`require.main === module`; the spec calls them with captured streams):
- G1 stdout is exactly three lines `id: `, `secret: `, `hash: `; the secret matches
  `SECRET_PATTERN`; the hash is `sha256:` + hex SHA-256 of the secret; two runs give different
  ids and secrets; stderr does not contain the secret.
- G2 `--id ci-pipeline` is used; `--id 'a b'` and an unknown flag exit 2 and print nothing on
  stdout.
- G3 the minted id, secret and hash, written to a key file, are accepted by `parseKeyFile` and
  `verifyApiKey`.

`representation.spec.ts`: `cacheControl(false, false)` is
`private, max-age=31536000, immutable`; `cacheControl(false, true)` unchanged; `latest`
`no-cache` either way.

### 4.2 oolong e2e (`test/auth.e2e-spec.ts`, `test/auth-none.e2e-spec.ts`, `npm run test:e2e`)

`auth.e2e-spec.ts`: `createApp()` with the fixtures copied to a temp dir (as `tea.e2e-spec.ts`),
`AUTH_MODE=required`, a temp key file with keys `k-active`, `k-disabled` (`disabled: true`),
`k-expired` (`expiresAt` in the past), minted with keygen's `mintKey()`, a random
`AUTH_TOKEN_SECRET`; a stderr capture (`jest.spyOn(console, 'error')`) for the whole file.

- E1 auth before lookup: a table of every route in `tea.controller.ts` (all 27 `GET` routes,
  and `HEAD` for each), each with a fixture id, an unknown uuid and `not-a-uuid`, plus
  `/discovery?tei=<fixture TEI>`, `/discovery` without `tei`, `/products?pageToken=forged`,
  `/artifact/<id>/1/download?mediaType=bogus`: without `Authorization` every one is `401`,
  `WWW-Authenticate: Bearer realm="tea"` (exact), `Cache-Control: no-store`, empty body. None is
  400, 404 or 406.
- E2 with `Authorization: Bearer garbage`, a tampered token, a token signed with another secret,
  and with two `Authorization` headers (raw `http.request` with a header array): `401`,
  `WWW-Authenticate: Bearer realm="tea", error="invalid_token"` (exact).
- E3 the `k-active` key's own Basic credentials sent to `/v1.0.0/products`: `401` with the
  no-error challenge; not served.
- E4 with a valid token: `/products`, `/product/<id>`, `/discovery?tei=...`, a paginated listing
  with its `nextPageToken`, an artifact download and its signature are `200` with bodies and
  headers equal to the same requests against a `none`-mode app, except the versioned download's
  `Cache-Control`, which is `private, max-age=31536000, immutable` (D8). An unknown uuid with a
  valid token is `404 OBJECT_UNKNOWN` (authorization does not conceal in v1).
- E5 public: `/.well-known/tea` and `/healthz` are `200` without a token and with
  `Bearer garbage`.
- E6 `/token` success: Basic built as tea-client-go does (each part `encodeURIComponent`-style
  form-encoded, joined with `:`, base64): `200` (not 201), body exactly
  `{access_token, token_type: "Bearer", expires_in: 3600}`, valid against
  `#/components/schemas/token-response` of `../../../tea-spec/openapi.yaml` with Ajv (as the
  existing schema checks), `Cache-Control: no-store`, `Content-Type: application/json`. A
  `scope=x` parameter is ignored and not echoed. The token works on E4.
- E7 `/token` `401`: no `Authorization`; `Bearer x`; Basic with an unknown id; with `k-active`
  and a wrong but well-formed secret; with a malformed secret; `k-disabled`; `k-expired`; two
  `Authorization` headers (raw `http.request`, valid Basic first): each
  `401`, `WWW-Authenticate: Basic realm="tea"`, `Cache-Control: no-store`, body exactly
  `{"error":"invalid_client"}` valid against `token-error-response`; the unknown-id and
  wrong-secret responses are byte-identical (status, headers except `Date`, body).
- E8 `/token` `400`: no `grant_type`, `grant_type=password` (`unsupported_grant_type`), two
  `grant_type`, `Content-Type: application/json`, `Content-Encoding: gzip`, 1025-byte body,
  `Content-Length: 1000000` with nothing sent (answered without waiting, `Connection: close`),
  9 parameters, invalid UTF-8 body, `client_secret=x` in the body with valid Basic: each `400`
  with the expected `error`, `Cache-Control: no-store`, valid against `token-error-response`.
- E9 revocation and reload through the running app: issue a token for `k-active`; rewrite the
  key file with `k-active` `disabled: true`; `app.get(ApiKeyRegistry).poll()`; the token is now
  `401 invalid_token` on `/products` and `/token` with the key is `401 invalid_client`; rewrite
  with the key enabled, `poll()`, a new token works; remove the key from the file, `poll()`, the
  token is `401 invalid_token`; write an invalid file, `poll()`, the last good keys still work.
- E10 clean logs (captured stderr of the whole file): contains no secret used in the file, no
  issued access token, no Basic token68 value, no unknown id sent in E7, no `sha256:`; contains
  `auth: token refused for key k-disabled: disabled` and `auth: token issued for key k-active`
  and the startup line with the key count.
- E11 startup refusals: `required` with a missing key file and with an invalid one: `createApp()`
  rejects, the error names the file problem and not its content.

`auth-none.e2e-spec.ts` (`AUTH_MODE` unset, then `none`; the existing e2e suites also stay green
unchanged, they are the main `none` regression):
- N1 `POST /v1.0.0/token` with valid-format Basic, with `Content-Type: application/json` and a
  malformed body, with a 200 KB urlencoded body, and with nothing: each
  `404 {"error":"NOT_IMPLEMENTED"}` (the two body cases were `500` before, 3.4 D7); never 401.
- N2 `GET /v1.0.0/products` with `Bearer garbage`, with `Basic ...` and with two
  `Authorization` headers: `200` (ignored); no response carries `WWW-Authenticate`.
- N3 versioned download `Cache-Control` is still `public, max-age=31536000, immutable`.
- N4 `AUTH_MODE=none` with `AUTH_KEYS_FILE` naming a missing file starts and serves.

### 4.3 Chart render tests (`helm-chart/oolong/scripts/chart-render.integration.sh`, run by `chart-render.integration.spec.ts`)

- C1 default values: the server Deployment has no `AUTH_` env, no `api-keys` volume or mount;
  no `-auth-token` Secret is rendered; NOTES contains the unauthenticated sentence.
- C2 `auth.mode=required`, `auth.apiKeys.existingSecret=oolong-api-keys`: env `AUTH_MODE`
  `"required"`, `AUTH_KEYS_FILE` `"/etc/oolong-api-keys/api-keys.yaml"`, `AUTH_TOKEN_TTL_SECONDS`
  `"3600"`, `AUTH_KEYS_RELOAD_INTERVAL_MS` `"10000"`, `AUTH_TOKEN_SECRET` by `secretKeyRef` to
  `<release>-oolong-auth-token` key `AUTH_TOKEN_SECRET`; volume `api-keys` with
  `secretName: "oolong-api-keys"`, item `key: "api-keys.yaml"` `path: api-keys.yaml`,
  `defaultMode: 0440`, `optional: false`; mount `/etc/oolong-api-keys` `readOnly: true`; the
  server Deployment contains no `subPath`; the auth-token Secret is rendered with
  `helm.sh/resource-policy: keep`. With `persistence.enabled=false` the same, and the content
  volume absent.
- C3 `existingConfigMap=oolong-api-keys` and `key=keys.yaml`: `configMap` volume with `name`,
  item `key: "keys.yaml"` `path: api-keys.yaml`.
- C4 `tokenSecret.existingSecret=my-auth`: no auth-token Secret rendered, `secretKeyRef` names
  `my-auth`.
- C5 refusals, each with schema validation and again with `--skip-schema-validation` (template
  guard), output checked for the value path: `required` with no source; both sources; `none`
  with `existingSecret`; `mode=Required`; `mode=off`; `key=` (empty); `key=a/b`;
  `existingSecret=Bad_Name`; `tokenTtlSeconds=59` and `86401`; `reloadIntervalMs=999`;
  `auth.apiKeys.subPath=x` (unknown field, schema only); `tokenSecret.create=false` without
  `existingSecret`; `env.AUTH_MODE=required` and `env.AUTH_TOKEN_SECRET=x` (reserved, template).
- C6 NOTES with `required`: contains the TLS sentence and the "whoever can write" sentence and
  not the unauthenticated sentence; with `useTraefikDevelopment=true` also the plain-HTTP
  WARNING.

### 4.4 tea-client-go interop tests (`integration_auth_test.go`, `-tags integration`)

All against `oolongtest.StartAuth` unless named otherwise, with a request-recording
`RoundTripper` (method, path, whether `Authorization` was Bearer/Basic/absent, status,
`WWW-Authenticate`), as `headerSpy` in `integration_test.go`.

- I1 token exchange: `a.RequestToken(ctx, id, secret)` returns `token_type` Bearer,
  `expires_in` 3600, a non-empty token; wrong secret, unknown id and a malformed secret
  (`not-a-key`) each return `*tea.TokenError` with `Status` 401 and `Code` `invalid_client`.
- I2 on-challenge flow: `WithAPIKey(srv.URL, id, secret)`; `GetProductRelease` succeeds;
  recorded: GET without auth 401 (challenge without `error`), POST `/v1.0.0/token` 200, GET with
  Bearer 200; a second call is one GET with Bearer 200 (cached), one token request in total.
- I3 private discovery: `discoveryClient` (existing helper) plus `WithAPIKey(srv.URL, ...)`:
  `Discover(fixtureTEI)` succeeds; `/.well-known/tea` was fetched without a token and answered
  200; `/discovery` 401, `/token` 200, `/discovery` with Bearer 200. Without the key: the error
  matches `tea.ErrUnauthorized`.
- I4 401 does not fail over: a test-owned well-known document (httptest) lists the oolong server
  (priority 1) and a decoy httptest server (lower priority); without a key, `Discover` returns an
  error matching `tea.ErrUnauthorized` and the decoy saw no request; with a key whose token
  request is refused (wrong secret), the error matches `tea.ErrNoAccessToken` and the decoy saw
  no request.
- I5 expired token, one refresh and one retry: `StartAuth` with `TokenTTLSeconds: 60`; the
  client's clock frozen with `tea.SetTestClock` (the client then keeps the cached token, server
  time moves on); first call obtains and caches a token; wait until 62 s after it was issued;
  second call: recorded GET with the old Bearer 401 with `error="invalid_token"`, exactly one
  POST `/token`, one GET with a different Bearer 200, and the call succeeds. `t.Parallel()`,
  skipped under `-short` (it waits about a minute; a 60 s TTL is the server's minimum, and the
  client caches only tokens with `expires_in` above its 30 s leeway, so a shorter wait cannot
  reach the expiry path).
- I6 revoked key: a call caches a token; `WriteKeys` with the key `disabled: true`; poll
  `RequestToken` until it returns `invalid_client` (deadline 15 s); the next API call fails with
  an error matching `tea.ErrNoAccessToken` that unwraps to `*tea.TokenError` `invalid_client`;
  recorded: GET with Bearer 401 `invalid_token`, POST `/token` 401, no further GET. Repeat with
  the key removed from the file and with `ExpiresAt` in the past. Then re-enable it: the next
  call succeeds.
- I7 wrong key: `WithAPIKey` with a wrong secret: the call fails with `tea.ErrNoAccessToken`
  (cause `invalid_client`); recorded GET 401, POST 401, no retry GET.
- I8 protected download: `DownloadArtifact` (the fixture of `TestIntegrationDownload`) with the key
  succeeds and verifies its checksum; the artifact GET was preceded by a 401 only if no token was
  cached; without a key the download fails with `tea.ErrUnauthorized`.
- I9 auth none unchanged: `oolongtest.Start` (none) with `WithAPIKey(srv.URL, ...)`:
  `GetProductRelease` succeeds, no POST `/token` was sent and no request carried
  `Authorization`; `a.RequestToken` returns an error matching `tea.ErrNotImplemented`.
  `TestIntegrationBearerIgnored` stays as it is.
- I10 clean logs: after I1, I6 and I7 on one `AuthServer`, `Output()` contains none of the
  secrets, none of the access tokens the client received, and no Basic token68 value.
- 403: oolong v1 never answers 403 (any valid key reads everything), so there is no interop case;
  "403 does not fail over" stays covered by the existing unit tests `TestDiscoverNoFailover`
  (row `403`) and `TestDiscoverAPIKeyNoRecovery` (T-7). This departs from the description's list
  for the reason stated; the case is added when oolong gains per-product rules.

## 5. Risks and open points

- **Split recommended, by repository** (as the description expects): the whole contract is
  designed here, but the two halves are separate coder passes in separate repositories, and the
  interop tests need the oolong change built.
  1. **OOL-7 oolong**: server auth, chart, docs. Sections 3.1-3.7 and 3.9; tests 4.1, 4.2, 4.3.
     Design from parent: **yes**.
  2. **OOL-7 tea-client-go interop tests**: sections 3.8; tests 4.4. Depends on child 1 (its
     tests run against child 1's built server). Design from parent: **yes**.
  Signed off without `--no-split`.
- **`none`-mode change (D7):** `POST /token` with a malformed or oversized body moves from `500`
  to `404 NOT_IMPLEMENTED`, and request bodies on other routes are no longer parsed (none is
  used). Named here so the reviewer can judge it against "behaviour exactly as today"; it is the
  only `none` difference, and it makes `/token` match the stated "404 NOT_IMPLEMENTED".
- **No rate limit on `/token`.** Each attempt costs one header parse, at most 1 KiB of body and
  one SHA-256; guessing a 256-bit secret is infeasible. Floods are an ingress concern (docs).
- **Rotation under the same id** does not end live tokens (tokens carry only the key id, as
  decided); docs make "new id, disable old" the rotation procedure. A per-key fingerprint in the
  token would close it, but would change what the token carries; not done.
- **Revocation latency** is kubelet sync plus up to `reloadIntervalMs`; documented, with pod
  restart for an immediate cut-off.
- **Future argon2id:** a hash type for operator-chosen secrets must refuse a ConfigMap source
  (the chart would need to know the source type at render; the server cannot) and bounds its
  cost per request; not part of v1.
- **Interop test duration:** I5 waits about a minute (reason in I5); it runs in parallel and is
  skipped with `-short`.
- **Coding principles:** no departure. Relevant sections: "Implement the pinned TEA
  specification" (3.1, 3.4, D1, D2), "Bound the work and memory of every request" (3.4 limits,
  3.3 linear regexes), "Content reloads are all or nothing" (key file, 3.2), "Prefer closed
  types" (3.5), "Timestamps" (`expiresAt`), "Errors" and "Secrets and credentials" (3.6, D9,
  D11), "The container and the chart stay locked down" (read-only mount, schema entries, no new
  port or writable path), "The Go client treats every server as untrusted" (no client change;
  the tests check it does not send credentials elsewhere, I4), "Keep source files plain ASCII".
  The role prompt's `docs/tea-upgrade/` no longer exists on the branch (2).
- No open question for the operator.
