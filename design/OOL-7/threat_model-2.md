# OOL-7 threat model, round 2: TEA /token and Bearer authentication

Task: OOL-7. Design: `design/OOL-7/architecture-2.md` (round 2). The change turns oolong from
a public read API into one that can require TEA API keys: it adds a credential-accepting
endpoint, a token signing key, a key file read at runtime, and a gate in front of every TEA
route. In `auth.mode none` nothing is protected and nothing new is exposed (one response change,
architecture D7).

## Changes in this round

Answers the design review round 1 (`review-items/OOL-7/design-reviewer/round-1.md`):

- **F-3**: A1 now says that `required` protects what oolong serves, and that artifact formats
  and signatures stored as publisher URLs are only as private as those URLs; added to the
  residual risks (architecture 3.7, 3.9, 5).
- **F-4**: T4 names log flooding through refusals for a known key id and its mitigation (one
  line per id per minute); its tests L1, L2 and E10 are listed in section 3.
- **F-1**: T8's tests now include E11 (as rewritten) and E12 (a refused start prints its reason).
  F-2 and F-5 change no threat (the health check sends no credential; the key file is
  operator-controlled), so nothing else here changed.

## 1. What is exposed

Assets:

- **A1, protected content**: everything the TEA API serves from its content (objects,
  discovery results, and artifact bytes and signatures stored in the content directory) when
  `auth.mode=required`. Its confidentiality is the point of the feature. **Exception:** an
  artifact format or signature that the content gives only as a URL is answered with a `302` to
  that URL (`TeaService.download`, `representation.ts` `sendRedirect`). `required` protects the
  `302`, not the target: the bytes are at the publisher's URL, which oolong neither controls nor
  protects, and the URL itself is in the artifact JSON every key holder reads. Those bytes are
  only as private as their URL.
- **A2, API key secrets**: 256-bit random values minted by keygen, held by clients. The server
  holds only their SHA-256 (A4).
- **A3, the token signing key** (`AUTH_TOKEN_SECRET`): whoever holds it can mint a valid token
  for any key id in the file.
- **A4, the key file**: ids, hashes, expiry and disabled flags. Not secret by itself (hashes of
  256-bit random secrets), but **write access to it is control over who can read A1**.
- **A5, access tokens**: bearer credentials, valid up to `tokenTtlSeconds` (default 1 h), while
  their key stays active.
- **A6, availability** of `/token` and of the API.

Trust boundaries:

- **B1, any network client to `POST /token`**: untrusted headers and body.
- **B2, any network client to the resource routes**: untrusted `Authorization` and parameters.
- **B3, operator's Secret or ConfigMap to the pod**: the key file and the signing key, through
  kubelet volume and env.
- **B4, server to its log**: stderr is read by whoever can read pod logs.
- **B5, server to shared caches** (CDN or proxy in front of the ingress).
- **B6, tea-client-go to any server**: unchanged library; the new tests check its credential
  handling against the real server.

Not in scope: TLS itself (terminated at the ingress, as today; the design warns and documents),
a hostile operator or cluster admin (controls the image and the Secrets already), per-object
authorization (v1 has none; any valid key reads everything, as decided).

## 2. Threats and mitigations

| Id | Threat | Mitigation (architecture section) |
|---|---|---|
| T1 | A protected route served without a valid token: a route left out of the gate, a URL spelling that bypasses a path prefix (case, trailing slash), `HEAD`, a `@Res()` download handler, or a check that runs after the lookup and leaks existence (`404` vs `401`, `400` for a malformed id). | Global Nest guard keyed on the matched handler, protected by default, explicit `@PublicRoute()` on exactly three handlers; runs before params, pipes, page tokens and snapshot reads (3.1, 3.5; prototyped). |
| T2 | Forged or replayed-after-revocation tokens: MAC forgery, token from another HMAC context (page tokens), expired token, token of a disabled, removed or expired key. | HMAC-SHA256 with a dedicated key and domain prefix, startup refuses key equal to `PAGE_TOKEN_SECRET`, constant-time MAC compare, strict payload, `exp` and skew checks; every request re-checks the key in the current key file (3.3, 3.5). |
| T3 | Online guessing or enumeration at `/token`: telling unknown ids from wrong secrets by response or timing; guessing secrets. | Same status, headers and body for every credential failure; one SHA-256 and one `timingSafeEqual` against a dummy digest for unknown ids; format refused before hashing (format is public); 256-bit secrets make guessing infeasible (3.4). |
| T4 | Resource exhaustion through `/token`: large or slow bodies, many parameters, compressed bodies, expensive hashing; log flooding by unauthenticated refusals, including for a known or guessed key id. | Body read by the handler only: 1 KiB cap (early refusal on `Content-Length`), 10 s read deadline, at most 8 parameters, no content coding, `bodyParser: false` globally; SHA-256 not a password hash (3.2, 3.4). Refusals are counted and logged at most one aggregate line a minute plus one line a minute per known id with refusals, counters keyed only by ids in the key file (3.6, D9). Rate limiting is left to the ingress (residual, 5). |
| T5 | Credential leakage: secrets, Basic values, tokens, or a secret pasted into the key file reaching logs, error messages or responses. | Log lines are a fixed set; ids logged only when they come from the key file; unknown or malformed requests only counted; key-file errors name rule and location, never a value or a YAML snippet; no `error_description`; config errors name the variable, not the value (3.2, 3.6). |
| T6 | Revocation that does not take effect: key file updates not reaching the pod (`subPath`), a watcher that stops after the first update, a bad file that silently drops all keys or keeps an old one without notice, rotation under the same id. | Whole-volume mount, polling read of the file with hash compare (prototyped against the `..data` symlink swap), all-or-nothing reload that keeps the last good set and logs the rejection; docs: rotation is new id then disable, restart for an immediate cut-off (3.2, 3.7, 3.9). Residual: kubelet sync delay. |
| T7 | Access control taken over through the key file source: a ConfigMap writable by many, `env.AUTH_*` overriding the chart, a key file shipped in the content repository (served publicly, kept in Git history). | Docs state that write access is access control and prefer a Secret with RBAC; the chart reserves `AUTH_*` in `env`; read-only mount of one item; docs forbid the content repository (3.7, 3.9). |
| T8 | Misconfiguration that fails open: `required` without keys, unknown `AUTH_MODE` treated as `none`, schema bypassed with `--skip-schema-validation`, both sources set. | Server refuses to start on any invalid `AUTH_*` in `required` or an unknown mode; chart schema plus template guards for every rule (3.6, 3.7). |
| T9 | Shared cache serves an authenticated response to anyone: versioned downloads are `Cache-Control: public` today. | `private` instead of `public` in `required` mode; JSON responses carry no `Cache-Control` and a shared cache must not store them for requests with `Authorization` (3.5, D8). |
| T10 | Ambiguous credentials: two `Authorization` headers read differently by a proxy and by Node (Node keeps the first, prototyped). | Two headers are refused: `invalid_token` on resources, `invalid_client` on `/token` (3.4, 3.5). |
| T11 | Tokens or keys in transit over plain HTTP. | TLS at the ingress, documented in README and NOTES, NOTES warning for `useTraefikDevelopment`, startup warning for a non-loopback `http://` `SERVER_HOST`; credentials only in headers (3.6, 3.7). |
| T12 | `none` mode regresses: a 401, a parsed Bearer, or `/token` answering anything but `404 NOT_IMPLEMENTED`. | Guard returns before reading headers in `none`; existing e2e suites unchanged plus `auth-none.e2e-spec.ts` (3.1, 3.4). |

## 3. Mitigations that need a test

All are in the architecture's test list (section 4):

- T1: E1 (every route, `GET` and `HEAD`, fixture, unknown and malformed ids, forged page token),
  E3, E5, P3; I3, I8.
- T2: T1, T2 (token unit cases including the page-token construction), P4, E2, E9; I5, I6.
- T3: C2, C3 (one hash and one compare, spies), E7 (byte-identical unknown-id and wrong-secret
  responses); I1, I7.
- T4: C1, E8 (size, early `Content-Length` refusal, parameters, encoding, content type); L1, L2,
  E10 (refusals aggregated per minute and per known id, no per-request line).
- T5: K4, A3 (messages without values), G1 (secret only on stdout), E10, I10.
- T6: R2, R3, R4 (Kubernetes symlink swap), R5, E9; C2 and C3 of the chart (no `subPath`,
  whole-volume item mount); I6.
- T7: chart C5 (`env.AUTH_*` reserved), C2/C3 (one item, read-only).
- T8: A2, A3, R1, E11, E12 (a refused start exits 1 and prints its reason, not silently);
  chart C5 (with and without schema validation).
- T9: E4 (`private` under `required`), N3 (`public` under `none`), the `representation.spec.ts`
  case.
- T10: P5, E2 (two headers on resources), E7 (two headers on `/token`).
- T11: chart C6 (NOTES TLS sentence and plain-HTTP warning). The `SERVER_HOST` startup warning
  is advisory output only and gets no test.
- T12: N1, N2, N3, N4, existing e2e suites, P2; I9.

Residual risks (accepted, documented): no rate limit on `/token` (ingress); revocation latency of
the kubelet sync delay plus the reload interval; rotation under the same id leaves issued tokens
valid until they expire; bearer tokens are usable by whoever holds them until expiry or
revocation; artifacts and signatures served by redirect to a publisher URL are not protected by
`required` (A1 exception; README, IMPLEMENTATION_NOTES and NOTES tell operators to keep private
bytes in the content directory, architecture 3.7, 3.9).
