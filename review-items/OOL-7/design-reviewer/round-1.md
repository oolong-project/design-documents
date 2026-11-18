# OOL-7 design review, round 1

Task: OOL-7, TEA /token and Bearer auth in oolong (all-or-nothing), API keys in a YAML file,
interop tests in tea-client-go. Reviewed: `design/OOL-7/architecture-1.md` (round 1) and
`design/OOL-7/threat_model-1.md` (round 1), against oolong-project/oolong `2026-11-oolong` @
`77a17b3`, oolong-project/tea-client-go `2026-11-oolong` @ `e848589`, the pinned TEA spec
(`tea-spec/openapi.yaml`, `962a0228`) and the coding principles at `145008cdcc23`.

Verdict: REJECTED. Five items are open. F-1 is priority 1: as designed, a server that cannot
read its key file exits without saying why, and E11 cannot pass. F-2 and F-3 are priority 2.
F-4 and F-5 are small. All five are filed about ARCHITECTURE. F-3 also asks for a change to the
threat model; the item says which. Apart from these items the design is thorough, matches the
operator's decisions in the task, and fits the code.

## What holds

- Scope matches the task and the operator's decisions; the out-of-scope list is honest. The
  one departure from the task's test list (no interop case for 403, because v1 never answers
  403) has a reason that holds, and the existing unit tests `TestDiscoverNoFailover` and
  `TestDiscoverAPIKeyNoRecovery` cover "403 does not fail over".
- A global guard keyed on the matched handler (D3) is the right choice. Express routes are
  case-insensitive and accept a trailing slash, so a path-prefix middleware could be bypassed.
  The guard runs before pipes, `@Param`/`@Query`, page tokens and snapshot reads, which gives
  auth before lookup, and the `@Res()` download handlers are covered too. E1 covers all 27
  `GET` routes and their `HEAD` forms in `tea.controller.ts`, with fixture, unknown and
  malformed ids.
- The `/token` contract matches the pinned spec: `client_credentials` with Basic, RFC 6749
  section 2.3.1 form-decoding (compatible with tea-client-go `auth.go`'s `url.QueryEscape`),
  `401 invalid_client` + `WWW-Authenticate: Basic`, `400 invalid_request` /
  `unsupported_grant_type`, `Cache-Control: no-store`, and `200`, not Nest's `201`. D1 (no
  body on resource 401/403) and D2 (no `error_description`) are correct readings of
  `401-unauthorized` and `unknown-error-type`.
- Unknown ids and wrong secrets cost the same (one SHA-256, one `timingSafeEqual` against a
  dummy digest) and get byte-identical responses, and C3 and E7 test it. SHA-256 is justified
  by the fixed-format 256-bit secrets that keygen mints, and `/token` refuses any other format
  before hashing.
- Tokens: domain-separated HMAC, a separate key that may not equal `PAGE_TOKEN_SECRET`, a strict
  payload, a skew bound, and a per-request check that the key is active, so revocation reaches
  live tokens (P4, E9). The residual risk of rotating under the same id is named and documented.
- Polling for key-file reloads instead of `fs.watch` (D5), with whole-volume mounts and no
  `subPath`, is correct for the kubelet `..data` symlink swap. R4 exercises it, and reloads are
  all or nothing as the principle requires.
- D7 (`bodyParser: false`) is a real bound on every request. Its one `none`-mode change (500
  becomes 404 `NOT_IMPLEMENTED` on `/token`) moves toward the operator's stated behaviour and is
  named for review. Accepted.
- D8 (`private` instead of `public` on versioned downloads under `required`) closes the
  shared-cache leak, and JSON responses need nothing more (RFC 9111 section 3.5).
- The chart: one schema and template guard per rule, enforced again with
  `--skip-schema-validation`, `AUTH_*` reserved in `env` (otherwise `env.AUTH_MODE=required`
  would bypass every check), no `value` on `tokenSecret` (D11), a one-item read-only mount, and
  `defaultMode: 0440` readable through the existing `fsGroup: 10001`. Probes use
  `/healthz` and `/.well-known/tea`, both of which stay public.
- The interop tests build on tea-client-go behaviour that exists: on-challenge recovery
  (`transport.go` `Do`: one token request, one retry, GET only), the 30 s cache leeway (which
  is why I5 freezes the client clock and needs a 60 s TTL), `ErrNoAccessToken` stopping
  discovery without failover, and `Client.now` at `client.go` line 384. `export_test.go` does
  not exist yet, so adding it is not a conflict.

## F-1: A key-file failure at startup exits silently, and E11 cannot pass as written

Functional (and operational security: "fail closed" must not mean "fail without a reason").

3.2 has the `ApiKeyRegistry` constructor read and parse the key file in `required` mode, and
"any error stops startup". The constructor runs inside `NestFactory.create(AppModule, {
logger: false, ... })`. In `@nestjs/core` 11 (oolong's lock file has 11.2.8), an exception
during `NestFactory.create` goes to `ExceptionsZone.asyncRun`, whose default teardown is
`process.exit(1)`. It only rethrows when `abortOnError: false` is passed, and 3.4 D7 does not
pass it. The exception is first handed to `ExceptionHandler`, which logs it through Nest's
`Logger`, and `logger: false` silences that logger (3.6 notes this itself). Two consequences:

- An operator whose key file is missing, unreadable or invalid sees the pod exit with status 1
  and no output at all. 3.6's premise that "operators must see key-file rejections" fails for
  the startup case, the one they are most likely to hit (first install, a typo in the file).
- E11 ("`createApp()` rejects, the error names the file problem") cannot hold: `createApp()`
  never rejects; the process exits, inside the jest worker.

The design must say how a startup failure surfaces. Either of these works:

- Read and validate the key file in `createApp()` before `NestFactory.create`, next to
  `loadAuthConfig`, so the error is an ordinary rejection that `main.ts` reports, and hand the
  resulting config and initial `KeySet` to the module (for example `AppModule.forRoot(config)`
  as a dynamic module). This also settles a smaller ambiguity: 3.6 calls
  `loadAuthConfig(process.env)` once in `createApp()` and again in the `AUTH_CONFIG` factory,
  so the "later `process.env` changes have no effect" claim holds only if the factory uses the
  first result.
- Or pass `abortOnError: false` and say what then prints the reason (`main.ts` does not catch
  the rejection of `bootstrap()` today; an unhandled rejection prints the stack, which is
  acceptable only if no key-file content is in the message, as K4 ensures).

E11 then asserts what the chosen path guarantees: `createApp()` rejects with the
`KeyFileError` message (rule and location, no content). Also add one check that the reason
reaches stderr when `dist/main.js` is started with a bad key file. tea-client-go's `StartAuth`
harness already runs the built server, and so does a small e2e that spawns `node dist/main.js`.

## F-2: The image's HEALTHCHECK is permanently unhealthy under `AUTH_MODE=required`

Functional.

`oolong/Dockerfile` sets `HEALTHCHECK ... CMD ["node", "tea-ready.js"]`, and both
`docker-compose.yml` and `docker-compose.tea-interop.yml` use the same script.
`docker/tea-ready.js` requires `GET /v1.0.0/products` to answer `200`. When
`CONTENT_AWAIT_GIT_REVISION=true` it also reads `results` from that body. Under `required` that
request has no token and gets `401`, so a container started as the design's new README `docker
run` example (3.9) stays `unhealthy` forever, and anything gated on health (compose
`depends_on: condition: service_healthy`, orchestrators that honour the Docker health status)
never starts. The design does not list `docker/tea-ready.js` among the changed files (section 2).

The design must say what readiness means under `required` and change the script to match. One
option: when `AUTH_MODE=required`, accept `401` with a `WWW-Authenticate: Bearer` challenge on
`/v1.0.0/products` as "the API is up", after `/.well-known/tea` has answered `200`. The
`CONTENT_AWAIT_GIT_REVISION` results check cannot run without a token; the design should state
that `/.well-known/tea` (503 until a snapshot is active) is the readiness signal in that case.
The script must not be given an API key. Add a test: the script against an app in `required`
mode exits 0, and against one that is not ready exits 1. The script is plain Node, so a jest
spec can spawn it against a `createApp()` listening on a port. Also mention the behaviour in
the README paragraph for the container image.

## F-3: Artifacts served by redirect are not protected by `required`; neither document says so

Security. The threat model and the docs must change; nothing new needs to be built.

`TeaService.download` answers a format that has no local body with `302 Location:
<format.url>`, and a signature with no local file with `302` to `signatureUrl`
(`tea.service.ts` lines 331-358, `representation.ts` `sendRedirect`). Under `required` the
`302` itself is behind the guard, but the bytes are at the publisher's URL, which oolong
neither controls nor protects. The URL is also in the artifact JSON every authenticated caller
reads. The threat model's A1 lists "artifact bytes and signatures" as protected content without
this exception, and 3.9's README and security-assessment text would lead an operator to believe
`auth.mode=required` makes their artifacts private.

- THREAT_MODEL: A1 states that `required` protects what oolong serves. Artifact formats and
  signatures stored as publisher URLs are only as private as those URLs. Add this to the
  residual risks.
- ARCHITECTURE 3.9: the README "Authentication" section, the IMPLEMENTATION_NOTES security
  assessment, and the `required` NOTES text say the same in one sentence: keep bytes that must
  stay private in the content directory, not behind a `url`. The existing NOTES line about
  redirect targets can carry it.

No new test is needed. E1 already shows that the redirecting routes need a token.

## F-4: Refusals for a known key id are logged once per request, unbounded

Security (resource use, log flooding), low.

D9 aggregates refusals for unknown ids and malformed requests into one line a minute, but
`auth: token refused for key <id>: wrong secret` is written for every request that names a
known id. Key ids are not secret, and operator-chosen ids like the design's own `ci-pipeline`
example are guessable. So an unauthenticated client that knows or guesses one id can make the
server write one stderr line per request, at whatever rate it can send. Bound it like D9: count
refusals per known id and reason, and log at most one line per id per minute (`auth: <n> token
requests refused for key <id> (<a> wrong secret, <b> disabled, <c> expired) in the last
minute`). The first refusal may still be logged at once if the design wants it to be prompt.
Adjust E10's expected line to match.

## F-5: The key file is read into memory whole before its size is checked

Functional (coding principles, "Bound the work and memory of every request": "File reads are
bounded in size. A file larger than the limit the design states is refused or streamed, never
read into memory whole."), low.

3.2 reads the file with `fs.readFileSync` at startup and on every poll, then refuses it if it
is over 65536 bytes. A wrong file mounted by mistake, or a large one, is then read whole every
`reloadIntervalMs`. The file is operator-controlled, so this is not an attack path, but the
design breaks the principle without saying why. Read at most 65537 bytes (open the path with
symlinks followed, then read into a bounded buffer) and refuse when the limit is reached. K2's
65537-byte case then shows the refusal; the size check stays as it is.
