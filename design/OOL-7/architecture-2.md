# OOL-7 architecture, round 2: TEA /token and Bearer authentication (all-or-nothing), API keys in a YAML file, interop tests in tea-client-go

Task: OOL-7. Coding principles: `coding-principles/coding_principles.md` at commit
`145008cdcc23`. Code read at oolong-project/oolong `2026-11-oolong` @ `77a17b3` (unchanged tip
for round 2) and oolong-project/tea-client-go `2026-11-oolong` @ `e848589`. Spec: the pinned TEA
spec at `962a0228af1bfcc1232ac3f79239317ff33fdb04` (`tea-spec/openapi.yaml`, byte-identical to
tea-client-go `spec/openapi.yaml`; `discovery/readme.md` "Authentication and authorization";
`auth/readme.md` at the same commit).

The operator's decisions recorded in the task description are requirements here; this design
does not reopen them. Where it adds a decision of its own it is numbered (D1...) with the reason.

## Changes in this round

Answers the design review round 1 (`review-items/OOL-7/design-reviewer/round-1.md`, F-1 to F-5).
The question pinned to this round (release `3fc05bef`) is that review-items release itself; it
holds no other question.

- **F-1** (startup failure silent, E11 impossible): the key file is now read and validated in
  `createApp()` before `NestFactory.create`, and the result is handed to
  `AppModule.forRoot(startup)` (D12); `NestFactory.create` also gets `abortOnError: false`, and
  `main.ts` catches the rejection and prints one line to stderr before exiting 1 (D13). Changed:
  2, 3.2 "Initial load" and "Reload", 3.4 "Body parsing", 3.6 (D12, D13), 3.9 (README), 4.1
  R1, 4.2 E11 and new E12 (`test/startup.e2e-spec.ts`), 5.
- **F-2** (HEALTHCHECK unhealthy under `required`): new 3.10 defines readiness under `required`
  and changes `docker/tea-ready.js`; README and IMPLEMENTATION_NOTES text in 3.9; tests H1-H4
  in 4.2 (`test/tea-ready.e2e-spec.ts`). Changed: 2, 3.9, 3.10, 4.2, 5.
- **F-3** (redirected artifacts not protected): no design change; the docs say it (3.7 NOTES
  text; 3.9 README and IMPLEMENTATION_NOTES security assessment) and 5 names it as a residual
  risk. No new test, as the item says (E1 already shows the redirecting routes need a token).
  The threat model's A1 and residual risks change in `threat_model-2.md`.
- **F-4** (per-request log line for refused known ids): refusals for known ids are counted per
  id and logged at most once a minute per id, like D9 (D9 extended, `AuthLog`). Changed: 3.6,
  4.1 new L1-L2, 4.2 E10.
- **F-5** (key file read whole before the size check): bounded read of at most 65537 bytes from
  one open file descriptor, refusing non-regular files (D14). Changed: 3.2, 4.1 K2 and new K5.

Prototyped for this round (Node 24.14.1, `@nestjs/core` from oolong's lock file, on `77a17b3`):

- A provider constructor that throws inside `NestFactory.create(M, { logger: false })`: the
  process exits with status 1 and prints nothing. With `abortOnError: false` the rejection
  reaches the caller; with `bootstrap().catch(...)` printing `err.message` the output is the one
  line. Today `dist/main.js` without `PAGE_TOKEN_SECRET` dies of an unhandled rejection that
  prints the throwing source line and the stack.
- The bounded read below against an 11-byte file through a `..data` symlink chain, a 65536-byte
  file (accepted), 65537 bytes and 200 MB (refused after reading 65537 bytes, under 0.1 ms, no
  memory growth), a FIFO and a directory (refused as not a regular file, the FIFO open does not
  block), and a missing path (`ENOENT`).
- `docker/tea-ready.js` (port changed for the run) against the built server in `none`: exit 0;
  against a stub that answers `/.well-known/tea` 200 and everything else `401` with a Bearer
  challenge: exit 1 today. The 3.10 logic: exit 0 against the stub with `AUTH_MODE=required`,
  exit 1 with `AUTH_MODE` unset, exit 1 when `/.well-known/tea` is 503, exit 0 against the `none`
  server with `AUTH_MODE` unset and exit 1 with `AUTH_MODE=required`.
- `node -r ts-node/register/transpile-only src/main.ts` starts (and fails) the server from
  source in under a second, so an e2e spec can check startup output without a prior build.

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
  - server, changed: `src/main.ts`, `src/app.module.ts`, `src/tea/create-app.ts`,
    `src/tea/tea.controller.ts`, `src/tea/well-known.controller.ts`,
    `src/tea/health.controller.ts`, `src/tea/errors.ts`, `src/tea/representation.ts`,
    `src/tea/tea.service.ts`, `package.json` (one script)
  - image health check: `oolong/docker/tea-ready.js` (3.10); `Dockerfile` and the compose files
    unchanged
  - server tests: new `test/auth.e2e-spec.ts`, `test/auth-none.e2e-spec.ts`,
    `test/startup.e2e-spec.ts`, `test/tea-ready.e2e-spec.ts`; existing suites unchanged (they
    are the `none` regression)
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
  refused. The size limit is enforced while reading, never after (D14, below): `parseKeyFile`
  keeps its own `bytes.length > 65536` check for callers that pass a buffer, and the file read
  never yields more than 65537 bytes.
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

**Reading the file** (D14, `readKeyFileBytes(path): Buffer` in `api-keys.ts`; coding
principles "Bound the work and memory of every request": "File reads are bounded in size"):

1. `fs.openSync(path, fs.constants.O_RDONLY | fs.constants.O_NONBLOCK)`. `open` follows
   symlinks (the Kubernetes `..data` chain); `O_NONBLOCK` keeps a FIFO mounted by mistake from
   blocking the open and has no effect on a regular file. An open error is
   `KeyFileError('cannot open (<code>)')` (the `errno` code, for example `ENOENT`, `EACCES`).
2. `fs.fstatSync(fd).isFile()` false (directory, FIFO, device): `not a regular file`.
3. Read into one `Buffer.alloc(65537)` with `fs.readSync` in a loop until it returns 0. When the
   buffer is full (65537 bytes read) stop and refuse: `larger than 65536 bytes`. The size from
   `fstat` is not trusted (it can change between `fstat` and the reads).
4. `fs.closeSync(fd)` in `finally`. Return the bytes read.

Memory per read is one 65537-byte buffer whatever the file's size; a 200 MB file is refused
after reading 65537 bytes (prototyped).

**Initial load** (D12, F-1). In `required` mode `createApp()` reads and parses the file
(`loadKeyFile(path): LoadedKeyFile` = `readKeyFileBytes` then `parseKeyFile`, returning the
`KeySet` and the SHA-256 of the bytes) before `NestFactory.create`, right after
`loadAuthConfig`. Any error rejects `createApp()` (fail closed: a server that requires auth and
cannot read its keys does not start), and `main.ts` prints the reason (3.6). The `KeyFileError`
message is prefixed with the configured path: `key file /etc/oolong-api-keys/api-keys.yaml:
keys[2].hash: expected sha256:<64 lowercase hex>` (the path is configuration, not content).

**Reload** (`ApiKeyRegistry`, `@Injectable`, in `api-keys.ts`). Its constructor reads nothing:
it takes the already loaded `KeySet` and bytes hash from the `AUTH_STARTUP` value (3.6), so
nothing in Nest's dependency injection can fail on the key file. In `onModuleInit` it starts
polling every `AUTH_KEYS_RELOAD_INTERVAL_MS` (default 10000) with `setInterval` (cleared in
`onModuleDestroy`), reading the file with `readKeyFileBytes` and comparing the SHA-256 of the
bytes with the last accepted file's:

- unchanged: nothing;
- changed and valid: the new `KeySet` replaces the old one in one assignment (requests read
  `current()` once per check);
- changed and invalid, unreadable, missing, too large or not a regular file: the last good
  `KeySet` stays, one log line with the reason (3.6), logged once per distinct failure (the
  bytes' hash, or the `KeyFileError` message for a read failure) so a bad file does not log
  every interval.

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

**Body parsing.** D7: `NestFactory.create(AppModule.forRoot(startup), { logger: false,
bodyParser: false, abortOnError: false })` (`forRoot` and `abortOnError`: 3.6, D12, D13).
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

Environment (`auth-config.ts`, `loadAuthConfig(env): AuthConfig`), and startup order in
`createApp()` (D12, F-1): `assertAdvertisedVersion()`, `assertPageTokenSecret()`, then
`const startup = loadAuthStartup(process.env)` (in `auth-config.ts`: `loadAuthConfig`, then in
`required` mode `loadKeyFile(config.keysFile)`, 3.2), then
`NestFactory.create(AppModule.forRoot(startup), ...)`. Everything that can refuse the
configuration runs before Nest, so a refusal is an ordinary rejection of `createApp()`.
`process.env` is read once, here; later changes have no effect.

```ts
export type AuthConfig =
  | { mode: 'none' }
  | { mode: 'required'; keysFile: string; tokenSecret: Buffer; tokenTtlSeconds: number;
      keysReloadIntervalMs: number };
export type AuthStartup =
  | { config: { mode: 'none' } }
  | { config: Extract<AuthConfig, { mode: 'required' }>; keys: KeySet; keysDigest: string };
```

D12, `AppModule.forRoot(startup: AuthStartup): DynamicModule` (the reviewer's first option).
`AppModule` keeps its controllers and providers in the `@Module` decorator and `forRoot` adds
`{ provide: AUTH_STARTUP, useValue: startup }` and
`{ provide: AUTH_CONFIG, useValue: startup.config }`. `createApp()` is the only place that
imports `AppModule` (every e2e suite goes through `createApp()`), so no other caller changes.
Reason: Nest's module factories run inside `NestFactory.create`, where an exception is either
a silent `process.exit(1)` (`abortOnError` default, `logger: false`) or a rejection that has
already passed through Nest's own logging; loading before Nest keeps the error in our hands and
lets one value feed both the guard and the registry (no second `loadAuthConfig` call).

D13, startup errors are printed, never silent:

- `NestFactory.create(..., { abortOnError: false })`: any other exception during module
  initialization (a bug, not configuration) rejects `createApp()` instead of exiting silently.
- `main.ts`: `void bootstrap()` becomes `bootstrap().catch(reportStartupFailure)`.
  `reportStartupFailure(err)` (in `main.ts`) writes to stderr and calls `process.exit(1)`. For a
  `StartupConfigError` (new base class in `errors.ts`, so that `auth-config.ts` and
  `api-keys.ts` need not import each other for it; extended by `AuthConfigError` and
  `KeyFileError`) it writes exactly one line, `startup failed: <message>`; for any other error
  `startup failed: ` followed by `err.stack` (what Node prints today for an unhandled rejection,
  without the source line). `process.exit(1)` and not `process.exitCode`: a failure
  after the app exists (for example `listen` with `EADDRINUSE`) would otherwise leave the
  reload timers holding the process open. stderr writes to a pipe or file are synchronous in
  Node on Linux, so the line is written before the exit.
- The existing `PAGE_TOKEN_SECRET` error (a plain `Error`) is then printed as
  `startup failed: Error: PAGE_TOKEN_SECRET must be set...` with its stack, exit status 1 as
  today. No value of any variable or file reaches these messages (A3, K4).

| Variable | Rule | Default |
|---|---|---|
| `AUTH_MODE` | `none` or `required`, exact | `none` |
| `AUTH_KEYS_FILE` | required in `required`; absolute path | none |
| `AUTH_TOKEN_SECRET` | required in `required`; at least 32 characters; not equal to `PAGE_TOKEN_SECRET` | none, no development fallback (coding principles "Secrets and credentials") |
| `AUTH_TOKEN_TTL_SECONDS` | integer `^[1-9]\d*$`, 60 to 86400 | 3600 |
| `AUTH_KEYS_RELOAD_INTERVAL_MS` | integer, 1000 to 300000 | 10000 |

In `none` mode every other `AUTH_*` variable is ignored (not validated) and no file is read, so
`none` stays today's behaviour. A violation throws an `AuthConfigError` naming the variable and
the rule, never the value.

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
- startup failure (any mode): `startup failed: <message>` from `main.ts` (D13), then exit 1.
- `/token` success, known id (the id comes from the operator's file, not from the request):
  `auth: token issued for key <id>`, one line per issued token. Only a caller holding the
  key's secret can cause it, and it is the audit trail of who obtained tokens.
- `/token` refusals for a known id (F-4): counted per key id and reason, and at most once per
  60 s per id that had refusals
  `auth: <n> token requests refused for key <id> (<a> wrong secret, <b> disabled, <c> expired) in the last minute`.
  Key ids are not secret and can be guessed, so a per-request line would let an
  unauthenticated client write to the log at its own rate. The counters are keyed only by ids
  present in the current `KeySet` (at most 1000), so their memory is bounded.
- `/token`, everything else (missing or malformed Basic, unknown id, bad body): counted, and at
  most once per 60 s `auth: <n> token requests refused (<a> malformed, <b> unknown key, <c> bad request) in the last minute`.
  D9: no attacker-supplied text (an unknown id may be a secret pasted in the wrong field)
  reaches the log, and a flood costs one line a minute (plus at most one line a minute per
  known id, above).
- Never: a secret, a hash, an `Authorization` value, a token, a request body, a YAML snippet.
  Resource `401`s are not logged.

`AuthLog` (`@Injectable`, `auth-log.ts`) holds both kinds of counters. A 60 s `setInterval`
(`unref()`, cleared in `onModuleDestroy`; started only in `required` mode) calls `flush()`,
which writes the non-zero lines (the aggregate line, then one line per known id in id order)
and resets the counters; nothing is written for a minute without refusals. `flush()` is public
and marked "visible for tests". The first refusal is not logged ahead of the minute: a client
misconfigured with a wrong secret shows up within a minute, which is enough for diagnosis.

`AppModule` providers: `AUTH_STARTUP` and `AUTH_CONFIG` (values from `forRoot`, D12),
`ApiKeyRegistry`, `AccessTokens`, `AccessPolicy`, `TokenEndpoint`, `AuthLog`,
`{ provide: APP_GUARD, useClass: AuthGuard }`. In `none` mode `ApiKeyRegistry` and `AuthLog`
read nothing and start no timer.

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
  expose the Service or pod port directly. Artifacts and signatures given as a url in the
  content are served by redirect and are only as private as that url: keep bytes that must stay
  private in the content directory.`, and when `useTraefikDevelopment` is true:
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
  stays, with one sentence added after it (F-3): `With auth.mode=required the redirect itself
  needs a token, but the bytes at the redirect target are only as private as that URL: keep
  artifacts and signatures that must stay private in the content directory, not behind a url.`
- new section `### Authentication` under `## Helm Chart Configuration`, after the `#### SSH
  deploy key` subsection and before `## Generation from TEA OpenAPI Specification` (line 290):
  what `required` protects and what stays public, including the redirect exception (F-3: an
  artifact format or signature whose bytes are not in the content directory is answered with a
  `302` to the publisher's URL; the `302` needs a token, the target does not, and the URL is
  in the artifact JSON every key holder reads; keep private bytes in the content directory);
  what a failed start prints (`startup failed: key file <path>: <rule>`, 3.6); minting a key
  with keygen (container command
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
  mount the directory to be safe), and one sentence on the image's health check (F-2, 3.10):
  with `AUTH_MODE=required` the container reports healthy once `/.well-known/tea` answers and
  the API answers `401` with a Bearer challenge; the health check holds no API key and does not
  look at content beyond `/.well-known/tea` (which answers `503` until content is loaded).

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
  boundaries and residual risks of the threat model (`design/OOL-7/threat_model-2.md` items),
  in particular: key-file source write access equals access control; revocation latency; no rate
  limit on `/token` (use the ingress); tokens are bearer tokens (TLS only); SHA-256 is safe only
  because secrets are 256-bit random from keygen; artifacts and signatures served by redirect
  are only as private as their URL (F-3, the same sentence as the README).
- `## Content Snapshots and Git Sync`, the **Readiness** bullet (line 148, which already names
  `docker/tea-ready.js`): one sentence on the health check under `AUTH_MODE=required` (3.10).
- `## Testing`: the new specs and e2e suites.

`helm-chart/oolong/values.yaml` (3.7) and `templates/NOTES.txt` (3.7, including the redirect
sentence). No change to `SECURITY.md`, the compose files or `.env.example` (they stay `none`).

### 3.10 Container health check under `required` (`oolong/docker/tea-ready.js`)

The image's `HEALTHCHECK` and both compose files run `node tea-ready.js`, which today requires
`GET /v1.0.0/products` to answer `200`. Under `required` that request has no token and gets
`401`, so the container would stay `unhealthy` (F-2). The chart is not affected (its probes use
`/healthz` and `/.well-known/tea`, both public).

Readiness under `required` means: the server answers, it has an active content snapshot, and
the authentication gate is in front of the API. The script (Docker runs the health command with
the container's environment, so `AUTH_MODE` and `PORT` are visible to it):

1. Base URL `http://127.0.0.1:<PORT>`, `PORT` from the environment, default `3000` (as
   `main.ts`; today the script hard-codes 3000, which a test cannot use and which breaks a
   container started with another `PORT`).
2. `GET /.well-known/tea` must be `200` and list version `1.0.0` (unchanged). It answers `503`
   until a content snapshot is active, so this is the content readiness signal in every mode.
3. `GET /v1.0.0/products`:
   - `AUTH_MODE` exactly `required`: ready when the status is `401` and `WWW-Authenticate`
     starts with `Bearer` (case-insensitive); anything else, `200` included (a `required`
     container serving without a token is not healthy), is exit 1. The
     `CONTENT_AWAIT_GIT_REVISION` check of `results` is skipped: it cannot run without a token,
     and step 2 already waits for the first revision (`/.well-known/tea` is `503` until then).
   - otherwise: unchanged (`200`, and with `CONTENT_AWAIT_GIT_REVISION=true` a non-empty
     `results`).
4. Any error or unparsable body: exit 1 (unchanged; Docker's `--timeout` bounds the run).

The script is never given an API key and sends no `Authorization`. It keeps reading
`AUTH_MODE` itself rather than trusting the server to say which mode it runs in, so a
container configured `required` whose server serves without a token is reported unhealthy.
D15: the `401` check is a readiness signal, not an authentication test; tests for the gate are
E1 to E3.

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
  a 65537-byte buffer passed to `parseKeyFile`; invalid UTF-8; YAML duplicate mapping key; YAML
  syntax error.
- K3 `expiresAt: 2027-01-01T02:00:00+02:00` (unquoted) is the instant `2027-01-01T00:00:00Z`.
- K5 `readKeyFileBytes` (D14, temporary directories): a file of exactly 65536 bytes is returned
  whole; 65537 bytes and a 10 MB file are refused `larger than 65536 bytes`, and for the 10 MB
  file `fs.readSync` was called with at most 65537 bytes in total (spy); a FIFO (`mkfifo`) and a
  directory are refused `not a regular file` and the call returns (the FIFO open does not
  block); a missing path is `cannot open (ENOENT)`; a file reached through the Kubernetes
  `..data` symlink chain of R4 is read.
- K4 validate before logging: a refused file that contains a well-formed secret
  (`oolong_sk_` + 43 characters) in an unknown field, in a bad `hash`, and on a line with a YAML
  syntax error yields messages that contain neither that string nor the line's text.

`api-keys.spec.ts` (`ApiKeyRegistry`, temporary directories, injected clock):
- R1 `loadAuthStartup` in `required` with a missing file, an unreadable file (mode 000), an
  invalid file and a 65537-byte file throws a `KeyFileError` whose message starts with
  `key file <path>: ` and names the rule; in `none` with the same `AUTH_KEYS_FILE` it reads
  nothing (spy on `readKeyFileBytes`). `ApiKeyRegistry` constructed from an `AuthStartup`
  value reads no file until `poll()` (spy).
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

`auth-log.spec.ts` (F-4, captured `console.error`):
- L1 three wrong-secret refusals and one disabled refusal for `k1`, two expired for `k2`, two
  unknown ids and one malformed header, then `flush()`: exactly three lines,
  `auth: 3 token requests refused (1 malformed, 2 unknown key, 0 bad request) in the last minute`,
  `auth: 4 token requests refused for key k1 (3 wrong secret, 1 disabled, 0 expired) in the last minute`,
  `auth: 2 token requests refused for key k2 (0 wrong secret, 0 disabled, 2 expired) in the last minute`;
  nothing is written before `flush()`; a second `flush()` writes nothing.
- L2 1000 refusals for one known id followed by `flush()` write one line for that id.

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

### 4.2 oolong e2e (`test/auth.e2e-spec.ts`, `test/auth-none.e2e-spec.ts`, `test/startup.e2e-spec.ts`, `test/tea-ready.e2e-spec.ts`, `npm run test:e2e`)

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
- E10 clean logs (captured stderr of the whole file, `app.get(AuthLog).flush()` called before
  the assertions): contains no secret used in the file, no issued access token, no Basic
  token68 value, no unknown id sent in E7, no `sha256:`; contains `auth: token issued for key
  k-active`, the startup line with the key count, and a line starting
  `auth: ` and containing `token requests refused for key k-disabled (` with a disabled count of
  at least 1; contains no line `auth: token refused for key` (the round-1 per-request form, F-4).
  A burst of 50 wrong-secret requests for `k-active` after a `flush()` adds no stderr line until
  the next `flush()`, which adds exactly one line for `k-active` with `50 wrong secret`.
- E11 startup refusals (F-1): `AUTH_MODE=required` with a missing key file, with an invalid one
  (a refused file that contains a well-formed secret in an unknown field) and with a 65537-byte
  one: `createApp()` rejects with a `KeyFileError` whose message starts with
  `key file <path>: ` and names the rule, and contains neither the secret nor any line of the
  file; no timer is left running (jest exits without `--forceExit`, as the existing suites).
  `AUTH_MODE=required` without `AUTH_TOKEN_SECRET`: rejects with an `AuthConfigError` naming
  `AUTH_TOKEN_SECRET`.

`startup.e2e-spec.ts` (F-1; spawns `node -r ts-node/register/transpile-only src/main.ts` from
`oolong/oolong-server` with `child_process.spawn`, a minimal environment built by the test
(`PATH`, `NODE_ENV=production`, `PORT=0`, `SERVER_HOST=http://127.0.0.1`, a temp
`CONTENT_DIR`, a random `PAGE_TOKEN_SECRET`, plus the case's `AUTH_*`), stdout and stderr captured, 20 s timeout per case):
- E12 `AUTH_MODE=required`, a valid `AUTH_TOKEN_SECRET` and `AUTH_KEYS_FILE` pointing at (a) a
  missing file, (b) a file with `version: 2`: the process exits with status 1 within the
  timeout, stderr has exactly one line, `startup failed: key file <path>: <rule>` (for (a)
  `cannot open (ENOENT)`, for (b) naming `version`), stdout is empty. (c) `AUTH_MODE=bogus`:
  exit 1, one stderr line `startup failed: ` naming `AUTH_MODE` and not containing `bogus`.

`tea-ready.e2e-spec.ts` (F-2; spawns `node ../../docker/tea-ready.js` relative to the test
directory with an environment of `PATH`, `PORT=<the app's port>` and the variables named per
case; the app is `createApp()` listening on an ephemeral port on `127.0.0.1`, fixtures copied as
in `tea.e2e-spec.ts`):
- H1 app in `required` (key file as in `auth.e2e-spec.ts`), script with `AUTH_MODE=required`:
  exit 0.
- H2 the same app, script without `AUTH_MODE`: exit 1 (a `401` is not ready in `none`).
- H3 app in `none`, script with `AUTH_MODE=required`: exit 1 (serving without a token is not
  healthy under `required`); script without `AUTH_MODE`: exit 0 (unchanged behaviour).
- H4 app in `required` with `CONTENT_AWAIT_GIT_REVISION=true` and no revision published
  (`/.well-known/tea` is `503`, as in `content-readiness.e2e-spec.ts`), script with
  `AUTH_MODE=required` and `CONTENT_AWAIT_GIT_REVISION=true`: exit 1.

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
  1. **OOL-7 oolong**: server auth, startup reporting, image health check, chart, docs.
     Sections 3.1-3.7, 3.9 and 3.10; tests 4.1, 4.2, 4.3. Design from parent: **yes**.
  2. **OOL-7 tea-client-go interop tests**: sections 3.8; tests 4.4. Depends on child 1 (its
     tests run against child 1's built server). Design from parent: **yes**.
  Signed off without `--no-split`.
- **`none`-mode change (D7):** `POST /token` with a malformed or oversized body moves from `500`
  to `404 NOT_IMPLEMENTED`, and request bodies on other routes are no longer parsed (none is
  used). Named here so the reviewer can judge it against "behaviour exactly as today"; it is the
  only `none` difference in responses, and it makes `/token` match the stated "404 NOT_IMPLEMENTED".
- **Startup output and health check, every mode (D13, 3.10):** a failed start now prints one
  `startup failed: ...` line (with the stack for errors that are not configuration errors)
  instead of Node's unhandled-rejection report; the exit status stays 1. `tea-ready.js` reads
  `PORT` (default 3000, so the image is unchanged). Both are operator-visible, not API, changes.
- **Artifacts served by redirect (F-3)** are not protected by `required`: oolong protects the
  `302`, not the publisher's URL it points to, and that URL is in the artifact JSON. Documented
  in README, IMPLEMENTATION_NOTES and NOTES (3.7, 3.9) and in the threat model (A1, residual
  risks); keeping private bytes in the content directory is the operator's remedy. Not built
  around: proxying or refusing redirects would change what the content format means.
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
  3.3 linear regexes, D14 bounded key-file read, F-4 bounded log lines), "Content reloads are all or nothing" (key file, 3.2), "Prefer closed
  types" (3.5), "Timestamps" (`expiresAt`), "Errors" and "Secrets and credentials" (3.6, D9,
  D11), "The container and the chart stay locked down" (read-only mount, schema entries, no new
  port or writable path), "The Go client treats every server as untrusted" (no client change;
  the tests check it does not send credentials elsewhere, I4), "Keep source files plain ASCII".
  The role prompt's `docs/tea-upgrade/` no longer exists on the branch (2).
- No open question for the operator.
