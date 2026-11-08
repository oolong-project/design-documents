# OOL-1 architecture, round 2: Git pod clones private repositories over HTTPS with a username and token

Task: OOL-1. Coding principles: `coding-principles/coding_principles.md` at commit
`145008cdcc23`. Code read at oolong-project/oolong `2026-11-oolong` @ `f216f68`.

## Changes in this round

Answers the design review of round 1 (`review-items/OOL-1/design-reviewer/round-1.md`):

- **F-1** (redirect test never holds the token): `test/git-http-auth-server.py` gets an
  explicit `--mode` (`backend`, `redirect`, `auth-redirect`); new 4.2 case 8 makes the
  configured host take the token and then redirect the authenticated request to another host,
  and asserts the token reached the first host and not the second; cases 6, 7 and 8 first
  assert that the server named received at least one request from that clone. Prototyped
  (section 3.4, D4). Round 1's 4.2 cases 8 to 11 are now 9 to 12. Sections changed: 3.4
  (D4), 4.2, 5. The threat model's T3 changes too.
- **F-2** (token in `gitPod.repoUrl` logged and stored before the refusal): new decision D6,
  `@` in `gitPod.repoUrl` fails the render in `oolong.requireGitRepoUrl` and in a new schema
  entry written so Helm's error does not echo the value; `git-sync.sh` validates the URL before
  it prints the configuration block; new 4.1 cases 8 and 9. Sections changed: 3.2, 3.3, 3.5,
  3.7, 4.1, 5. The threat model's T1 changes too (`threat_model-2.md`).
- **F-3** (README line 157 still says public): added to the docs list. Section changed: 3.7.

## 1. Problem

The chart's Git pod (`helm-chart/oolong`, `gitPod.*`) clones `gitPod.repoUrl` with no
credentials, so the content repository must be public. Operators need to sync a private
repository over HTTPS with a username and token (GitHub fine-grained PAT, GitLab deploy or
project token, Bitbucket app password) held in a Kubernetes Secret, without the token reaching
a URL, a command line, a log, a file on the content PVC or a published revision.

Out of scope:

- SSH deploy keys. That is OOL-2; this design shapes `gitPod.auth` so OOL-2 adds `type: ssh`
  without reshaping it (section 3.1, section 5).
- A chart-created Secret holding the token (decision D2).
- Server code (`oolong/oolong-server/src`): only the Git pod clones; the server keeps reading
  published revisions off the shared volume. The TEA API, the Go client and the content format
  do not change.
- Egress restriction for the Git pod (NetworkPolicy guidance already in README and NOTES.txt
  stays as it is).

## 2. Repositories and branches

- oolong-project/oolong, branch `2026-11-oolong` (PR against it):
  - `helm-chart/oolong/values.yaml`, `values.schema.json`
  - `helm-chart/oolong/templates/git-deployment.yaml`, `git-configmap.yaml`, `_helpers.tpl`,
    `NOTES.txt`
  - `helm-chart/oolong/scripts/git-publish.inc.sh`, new `scripts/git-credential-oolong.sh`,
    `scripts/chart-render.integration.sh`
  - tests: new `oolong/oolong-server/test/git-auth.integration.sh`, new
    `oolong/oolong-server/test/git-http-auth-server.py`, new jest wrapper
    `oolong/oolong-server/src/tea/git-auth.integration.spec.ts`
  - docs: `README.md` (Git Pod Configuration), `docs/tea-upgrade/security-assessment.md`
- oolong-project/tea-client-go: not touched.

## 3. Design

### 3.1 Values: `gitPod.auth` with a type discriminator

Added to `values.yaml` under `gitPod` (after `env`):

```yaml
  # Credentials for a private repository. type is one of:
  #   none  - anonymous clone (default)
  #   token - HTTPS username + token read from an existing Secret
  # (ssh is reserved for SSH deploy keys, OOL-2.)
  auth:
    type: none
    # Name of an existing Secret in the release namespace. Required when type
    # is not none; must be empty when type is none. The chart never creates it.
    existingSecret: ""
    token:
      # Keys in existingSecret holding the HTTPS username and the token.
      usernameKey: username
      tokenKey: token
```

The `repoUrl` comment (line 72, "Public https:// Git URL") becomes
`https:// Git URL without credentials (required when gitPod.enabled is true); for a private
repository set gitPod.auth`.

Shape rules, chosen so OOL-2 is additive:

- `auth.type` is the closed set `none | token` (coding principles, "Prefer closed types over
  magic strings and booleans": a third state, `ssh`, is already known). OOL-2 adds `ssh` to the
  enum and an `auth.ssh` sub-block with its own key names (for example `privateKeyKey`,
  `knownHostsKey`).
- `auth.existingSecret` sits at the `auth` level and is shared by every non-`none` type. Each
  type keeps its key names in a sub-block named after the type (`auth.token`, later
  `auth.ssh`), so no field changes meaning when a type is added.
- The Secret reaches the pod as files in one read-only mount (section 3.3), which is also how
  OOL-2 must deliver a private key and known_hosts. OOL-2 adds items to the same volume.

### 3.2 Schema (`values.schema.json`)

Add `gitPod.properties.auth` (draft-07, which the file already declares). Verified with Helm
v4.2.3 on a scratch copy of the chart:

```json
"auth": {
  "type": "object",
  "additionalProperties": false,
  "required": ["type"],
  "properties": {
    "type": { "type": "string", "enum": ["none", "token"] },
    "existingSecret": {
      "type": "string",
      "maxLength": 253,
      "pattern": "^([a-z0-9]([-a-z0-9.]*[a-z0-9])?)?$"
    },
    "token": {
      "type": "object",
      "additionalProperties": false,
      "required": ["usernameKey", "tokenKey"],
      "properties": {
        "usernameKey": { "type": "string", "minLength": 1, "maxLength": 253, "pattern": "^[-._a-zA-Z0-9]+$" },
        "tokenKey":    { "type": "string", "minLength": 1, "maxLength": 253, "pattern": "^[-._a-zA-Z0-9]+$" }
      }
    }
  },
  "allOf": [
    { "if": { "properties": { "type": { "const": "token" } } },
      "then": { "required": ["existingSecret", "token"],
                "properties": { "existingSecret": { "minLength": 1 } } } },
    { "if": { "properties": { "type": { "const": "none" } } },
      "then": { "properties": { "existingSecret": { "maxLength": 0 } } } }
  ]
}
```

This refuses every half-configured block: `type: token` without a Secret name, a Secret name
with `type: none` (an operator who forgot `type` would otherwise get a silent anonymous clone),
an empty or malformed key name, a misspelt field (`additionalProperties: false`), and `type:
ssh` until OOL-2. Key-name and Secret-name patterns are the Kubernetes rules (Secret data key;
DNS-1123 subdomain).

Also add `gitPod.properties.repoUrl`, which has no schema entry today (coding principles, "The
container and the chart stay locked down": new chart values come with a schema entry):

```json
"repoUrl": {
  "type": "string",
  "maxLength": 2048,
  "not": { "pattern": "@" }
}
```

Decision **D6, `@` in `gitPod.repoUrl` fails the render, and the schema never echoes the
value.** Now that private repositories are supported, the likeliest misconfiguration is a token
written into the URL (`https://user:<token>@host/org/repo.git`). Today that renders (only the
`https://` prefix is checked), lands in the Deployment's `GIT_REPO_URL` env and the Helm release
record, and is printed by `git-sync.sh` before `validate_git_repo_url` refuses it. Two choices
in the rule, both prototyped with Helm v4.2.3 on a scratch copy of the chart:

- **The rule is "no `@` anywhere in the URL", not "no `@` in the authority".** It is exactly the
  rule `validate_git_repo_url` already applies at runtime (`case "$_rest" in *@*)` over
  everything after `https://`), so the render and the pod refuse the same set and no URL renders
  only to crash-loop. It also needs no authority parsing (`?` and `#` delimiters). A URL with `@`
  in its path is refused at runtime today, so refusing it at render moves an existing failure
  earlier and breaks nothing that works.
- **The schema uses `not: {pattern}`, and does not check the `https://` prefix.** Helm's
  error for a failed positive `pattern` prints the value (`at '/gitPod/repoUrl':
  'https://tester:...@example.invalid/repo.git' does not match pattern ...`), which would put
  the token into CI logs; for a failed `not` it prints only `at '/gitPod/repoUrl': 'not'
  failed`. The `https://` prefix stays in the template guard only (section 3.3), whose message
  is fixed text. An `anyOf` of `""` and an `https://` pattern was tried and also echoes the
  value. The default `repoUrl: ""` passes the entry.

### 3.3 Template changes

`_helpers.tpl`, new `oolong.gitAuthType` (called from `git-deployment.yaml` next to
`oolong.requireGitRepoUrl`): returns the validated type and fails the render when the type is
not `none`/`token`, when `type: token` has an empty `existingSecret`, `token.usernameKey` or
`token.tokenKey`, or when `type: none` has a non-empty `existingSecret`. It repeats the schema
rules because `helm template --skip-schema-validation` skips the schema (coding principles,
"The container and the chart stay locked down"). Messages name the value path, e.g.
`gitPod.auth.existingSecret is required when gitPod.auth.type is token`.

`_helpers.tpl`, `oolong.requireGitRepoUrl` (lines 74-81): after the emptiness check and before
the `https://` prefix check, add (D6):

```
{{- if contains "@" .Values.gitPod.repoUrl }}
{{- fail "gitPod.repoUrl must not contain @ (no user name or token in the URL); set gitPod.auth for a private repository" }}
{{- end }}
```

The message names the value path and the rule and never includes the value. It runs before the
prefix check so `http://user:<token>@host/...` gets the userinfo message. It is the guard that
holds with `--skip-schema-validation`. Prototyped: `helm template` with
`gitPod.repoUrl=https://tester:not-a-real-token-0001@example.invalid/repo.git` fails with and
without `--skip-schema-validation`, and neither output contains the token.

`git-deployment.yaml`:

- Always add env `OOLONG_GIT_AUTH_TYPE` with the validated type (quoted). It is the only
  auth-related env var.
- When the type is `token`, add a volume and a mount:

  ```yaml
  volumes:
    - name: git-auth
      secret:
        secretName: {{ .Values.gitPod.auth.existingSecret | quote }}
        defaultMode: 0440
        optional: false
        items:
          - key: {{ .Values.gitPod.auth.token.usernameKey | quote }}
            path: username
          - key: {{ .Values.gitPod.auth.token.tokenKey | quote }}
            path: token
  volumeMounts:
    - name: git-auth
      mountPath: /etc/oolong-git-auth
      readOnly: true
  ```

  The mount is the whole directory, never `subPath`, so kubelet propagates a rotated Secret.
  The files are group-readable through the pod's `fsGroup` (default `podSecurityContext.fsGroup:
  10001`), as the existing `git-script` ConfigMap already is.
- The `gitPod.env` loop (line 47) no longer filters silently; reserved names fail the render
  first (section 3.6), so the loop emits every entry it is given.

`git-configmap.yaml`: add `git-credential-oolong.sh: |` from `.Files.Get
"scripts/git-credential-oolong.sh"`, like `git-publish.inc.sh` (lines 80-81). The
`checksum/git-script` annotation then also rolls the pod when the helper changes.

In `git-sync.sh`, validate before anything is printed (D6): move `. /scripts/git-publish.inc.sh`
and `validate_git_repo_url || exit 1` (lines 64-66) up to just before `echo "Configuration:"`
(line 47), after the `CONTENT_DIR` / `WORK_DIR` / ... defaults and the retention checks (the
include derives `ACTIVE_POINTER_FILE` from `CONTENT_DIR` when sourced, so it cannot move above
the defaults). Directly after it add
`configure_git_auth /scripts/git-credential-oolong.sh /etc/oolong-git-auth || exit 1`. The
`git config --global protocol.*` lines and `mkdir -p "$REVISIONS_DIR"` stay where they are,
after the configuration block. Print `  Auth: $OOLONG_GIT_AUTH_TYPE` in the configuration
block. So `  Repository: $GIT_REPO_URL` is printed only for a URL that passed the userinfo
refusal; nothing else is printed about credentials. `validate_git_repo_url` prints a fixed
message, never the URL, on refusal (unchanged).

`NOTES.txt` (line 23 block): one sentence stating the auth type and, for `token`, that Secret
`<existingSecret>` must exist in the release namespace with the two keys, that a rotated
Secret is picked up without a restart, and that the token should be read-only and scoped to
the one repository.

### 3.4 Wiring git: `configure_git_auth` in `git-publish.inc.sh`

New function `configure_git_auth <helper-script> <auth-dir>`, in `git-publish.inc.sh` so the
shell tests can source and call it (as they do `sync_worktree`):

1. `export GIT_TERMINAL_PROMPT=0` for every type, so missing or bad credentials fail at once
   instead of waiting for a terminal while the publish `flock` is held.
2. Refuse `<helper-script>` and `<auth-dir>` unless they match `^[A-Za-z0-9/._-]+$`. They are
   chart constants, but they end up in a git config value that git runs through a shell, so the
   function enforces that they carry no shell syntax ("Values are never interpolated into shell
   text").
3. `case "$OOLONG_GIT_AUTH_TYPE"`: `none` returns 0; `token` continues; anything else (empty,
   `ssh`, `Token`) prints `ERROR: OOLONG_GIT_AUTH_TYPE must be none or token` and returns 1.
   OOL-2 adds its `ssh)` branch here.
4. For `token`: check `<auth-dir>/username` and `<auth-dir>/token` with the same reader the
   helper uses (section 3.5): each must be readable and hold exactly one non-empty line, with
   or without a trailing newline, and no carriage return. On failure print
   `ERROR: <auth-dir>/token is missing, empty or not a single line` (the path, never content)
   and return 1, so a broken Secret shows as a crash loop with a clear log line at start-up.
5. Derive the credential origin from `GIT_REPO_URL`: `<scheme>://<hostport>` where hostport is
   the text between `://` and the first `/`. Refuse an empty hostport or one containing `@`.
   In the pod this runs after `validate_git_repo_url`, so the scheme is `https` and there is no
   userinfo; the tests use `http://127.0.0.1:<port>` against a local server.
6. Export the git configuration through the environment (git 2.31+; the pinned image is
   `alpine/git:2.54.1`), so no config file is written for it:

   ```sh
   GIT_CONFIG_COUNT=2
   GIT_CONFIG_KEY_0=credential.helper
   GIT_CONFIG_VALUE_0=              # empty: resets every helper from system/global config
   GIT_CONFIG_KEY_1="credential.${origin}.helper"
   GIT_CONFIG_VALUE_1="/bin/sh ${helper} ${authdir}"
   ```

Decisions in this step:

- **D1, files not env for the secret.** The task proposed `secretKeyRef` env vars. This design
  mounts the Secret as files instead: (a) kubelet updates a mounted Secret in place, so a
  rotated token is used on the next sync without a pod restart, while env from `secretKeyRef`
  is frozen until the pod restarts; (b) the token is not in the environment that every git,
  curl and shell child inherits, nor in `kubectl exec ... env` output; (c) OOL-2's private key
  and known_hosts must be files anyway, so one mechanism serves both types.
- **D3, the empty `credential.helper` entry is required, not cosmetic.** Prototyped with git
  2.43: with a `credential.helper store` in global config and only the scoped entry set, git
  wrote `~/.git-credentials` holding the token; with the reset entry first it did not. The
  environment entries are read after all config files, so the reset clears anything the image
  or `$HOME/.gitconfig` configures.
- **D4, host scope.** The helper is configured only for `credential.<scheme>://<hostport>`.
  Prototyped: a clone of the same repository by another host name, and an initial-request
  redirect to another host, both failed with `terminal prompts disabled`, and the other host
  never received an `Authorization` header. Round 2 also prototyped the case where the token is
  in hand at the redirect (git 2.43, libcurl 8.5, helper scoped to `http://localhost:<port2>`):
  the configured host logged `info/refs` once without and once with `Authorization`, answered
  the authenticated request with a 302 to `http://127.0.0.1:<port>`, git followed it, the
  second host logged one request without `Authorization` and answered 401, and the clone failed
  with `Authentication failed` because no helper matches the second host. libcurl drops the
  header across hosts (git never sets `CURLOPT_UNRESTRICTED_AUTH`) and the scope keeps git from
  filling it again; 4.2 case 8 pins both. A redirect to `http://` is refused by the existing
  `protocol.http.allow never`. `http.followRedirects` stays at git's default (`initial`), which
  same-host redirects (for example `/repo` to `/repo.git`) need.
- `credential.useHttpPath` stays off: the scope is the host, which is also what providers scope
  tokens by; the operator scopes the token to the repository (docs, section 3.7).

### 3.5 The credential helper `scripts/git-credential-oolong.sh`

Invoked by git as `/bin/sh /scripts/git-credential-oolong.sh /etc/oolong-git-auth <action>`.

- For any action other than `get` (`store`, `erase`) it exits 0 and does nothing, so git can
  never make it persist or forget anything.
- For `get` it reads and discards git's request on stdin up to the blank line, reads the first
  line of `<dir>/username` and `<dir>/token` with `IFS= read -r` (a trailing newline, common
  with `kubectl create secret --from-file`, is tolerated), and refuses (message to stderr naming
  the file, exit 1, nothing on stdout) when a value is empty, contains a carriage return, or
  the file has a second line. Refusing multi-line values keeps a malformed Secret from injecting
  extra credential attributes (`url=`, `host=`) into git's helper protocol.
- It prints `username=<u>` and `password=<t>` with `printf`, a shell builtin in busybox ash and
  dash, so the token is never an argument of an executed program. The script uses only
  builtins (`read`, `printf`, `case`, `[`); no `cat`, `echo` of the value, `set -x` or `eval`.
- It is plain ASCII and POSIX sh.

The token is therefore never in `GIT_REPO_URL`, the remote URL, `.git/config`, `FETCH_HEAD`, a
command line, the pod spec or an env var. A token written into `gitPod.repoUrl` by mistake is
refused at render (D6) before it reaches the pod spec, the release record or a log;
`validate_git_repo_url` keeps refusing userinfo at runtime (unchanged, now before the
configuration block is printed), and `copy_worktree_except_git` keeps skipping `.git` when
publishing.

### 3.6 `gitPod.env` reserved names fail the render

Today `git-deployment.yaml` line 47 drops seven reserved names silently. Decision **D5**: a
reserved name now fails the render, for the old and the new names alike, with
`gitPod.env.<NAME> is set by the chart and cannot be overridden`. Reason: a silently dropped
`GIT_SSL_NO_VERIFY` or `GIT_CONFIG_COUNT` leaves the operator believing a setting is active;
failing says so at install time. This is a behaviour change for values that set one of the
seven old names (they never had an effect); README states it. The check is a new helper
`oolong.requireGitPodEnvNotReserved` in `_helpers.tpl`, called from `git-deployment.yaml`; Go
templates range over map keys in sorted order, so the first offending key is reported
deterministically.

Reserved exactly:

- set by the chart: `GIT_REPO_URL`, `GIT_BRANCH`, `GIT_REPO_HOST_ALLOWLIST`,
  `REVISION_RETENTION_COUNT`, `CONTENT_DIR`, `SYNC_INTERVAL`, `HOME`
- would print credentials: `GIT_CURL_VERBOSE`
- would override or bypass the credential and protocol wiring: `GIT_CONFIG`,
  `GIT_CONFIG_COUNT`, `GIT_CONFIG_PARAMETERS`, `GIT_CONFIG_GLOBAL`, `GIT_CONFIG_SYSTEM`,
  `GIT_CONFIG_NOSYSTEM`, `GIT_ASKPASS`, `SSH_ASKPASS`, `SSH_ASKPASS_REQUIRE`,
  `GIT_TERMINAL_PROMPT`, `GIT_ALLOW_PROTOCOL` (overrides `protocol.*.allow`, so it could
  re-enable `http://`)
- would send the token to an unverified peer: `GIT_SSL_NO_VERIFY` (a private CA is supported
  with `GIT_SSL_CAINFO`, which stays allowed)
- reserved now for OOL-2 so it does not have to change the list: `GIT_SSH`, `GIT_SSH_COMMAND`

Reserved by prefix: `GIT_TRACE` (covers `GIT_TRACE`, `GIT_TRACE_CURL`, `GIT_TRACE_PACKET`,
`GIT_TRACE_REDACT`, `GIT_TRACE2*`; `GIT_TRACE_REDACT=0` would turn off the redaction that
`GIT_TRACE_CURL` applies to the `Authorization` header), `GIT_CONFIG_KEY_`,
`GIT_CONFIG_VALUE_`, `OOLONG_GIT_` (the chart's own namespace).

Everything else (`GIT_SSL_CAINFO`, `GIT_HTTP_LOW_SPEED_LIMIT`, `HTTPS_PROXY`, ...) passes
through as today. The reserved list protects against accidental leaks and overrides by the
values author; it is not a boundary against a hostile operator, who controls the image and the
Secret anyway.

### 3.7 Docs

- `README.md`: every place that says the Git pod syncs a public repository: drop "(currently,
  only public git repositories are supported)" (line 57), change "from a public HTTPS Git
  repository" to "from an HTTPS Git repository, public or private (see Private repositories)"
  in the Git Pod Configuration intro (line 157), and change "Public `https://` Git URL" to
  "`https://` Git URL without credentials" (line 176). Line 115 (the TEA API is a public
  catalog) is about the server, not the Git pod, and stays. In Git Pod Configuration, add a
  "Private repositories" paragraph: create the Secret with
  the two keys (from files or a secret manager, not from a literal on the command line), set
  `gitPod.auth.type=token` and `gitPod.auth.existingSecret`; per provider which username goes
  with the token (GitHub fine-grained PAT: the account's user name, token with read-only
  Contents on the one repository; GitLab deploy token: its generated username,
  `read_repository`; Bitbucket app password: the Bitbucket user name, repository read);
  rotation is picked up on a later sync without a restart; the Secret files rely on the pod's
  `fsGroup`; reserved `gitPod.env` names now fail the render; a `gitPod.repoUrl` containing `@`
  fails the render (never put the token in the URL). Values list gains `gitPod.auth.type`, `gitPod.auth.existingSecret`,
  `gitPod.auth.token.usernameKey`, `gitPod.auth.token.tokenKey`.
- `docs/tea-upgrade/security-assessment.md` SEC-08 note (line 85): the Git pod clones a public
  URL or, with `gitPod.auth.type: token`, a private one with a token read from a mounted Secret
  through a host-scoped credential helper; the token is never in the URL, `.git/config`, the
  PVC or a revision. Add one line to the Secrets section saying the same.
- `values.yaml` comments as in section 3.1.

## 4. Tests the coder must add

All new shell is POSIX sh and plain ASCII (`LC_ALL=C /usr/bin/grep -nP "[^\x00-\x7F]"` prints
nothing for every touched file). Fixtures use made-up values (`example.invalid`,
`tester` / `not-a-real-token-0001`).

### 4.1 `helm-chart/oolong/scripts/chart-render.integration.sh` (run by
`src/tea/chart-render.integration.spec.ts` in `npm test`)

Using the existing `COMMON_ARGS` pattern and grep/awk/python text checks as the file does:

1. Default render: `OOLONG_GIT_AUTH_TYPE` is `"none"`; no `git-auth` volume, no
   `/etc/oolong-git-auth` mount, no `secretName`.
2. `auth.type=token`, `existingSecret=oolong-git-auth`, `usernameKey=user`, `tokenKey=pat`:
   `OOLONG_GIT_AUTH_TYPE` is `"token"`; volume `git-auth` with `secretName: "oolong-git-auth"`,
   `defaultMode: 0440`, `optional: false`, items `user -> username` and `pat -> token`; mount
   at `/etc/oolong-git-auth` with `readOnly: true` and no `subPath`; the render contains no
   `secretKeyRef` and no env named `OOLONG_GIT_TOKEN` or `OOLONG_GIT_USERNAME`.
3. Schema refusals (each `helm template` must fail): `type=token` without `existingSecret`;
   `type=none` with `existingSecret=x`; `type=ssh`; `type=Token`; unknown field
   `auth.existingsecret`; `token.tokenKey=` (empty); `token.usernameKey='a b'`;
   `existingSecret=Bad_Name`.
4. Template guard with `--skip-schema-validation`: `type=token` without `existingSecret` fails
   with the message naming `gitPod.auth.existingSecret`; `type=bogus` fails.
5. Reserved env names: a loop over every exact name in section 3.6 plus `GIT_TRACE_CURL`,
   `GIT_TRACE2_EVENT`, `GIT_TRACE_REDACT`, `GIT_CONFIG_KEY_0`, `GIT_CONFIG_VALUE_0`,
   `OOLONG_GIT_TOKEN`: `--set gitPod.env.<NAME>=x` fails and the output names `<NAME>`.
   `gitPod.env.GIT_SSL_CAINFO=/etc/ssl/certs/ca.pem` and `gitPod.env.GIT_HTTP_LOW_SPEED_LIMIT=1`
   render and appear as env entries.
6. Checksum: extend the existing probe (lines 163-173) so appending a comment to
   `scripts/git-credential-oolong.sh` also changes `checksum/git-script`.
7. The existing render refusal of an `http://` `gitPod.repoUrl` stays green (today the only
   URL refusal at render; the runtime userinfo refusal is in `test/git-publish.integration.sh`,
   section 4.2).
8. Userinfo refused at render (D6): with
   `gitPod.repoUrl=https://tester:not-a-real-token-0001@example.invalid/repo.git`,
   `helm template` fails, once with schema validation and once with `--skip-schema-validation`;
   neither combined stdout+stderr contains `not-a-real-token-0001`; the second names
   `gitPod.repoUrl`. The same two runs with
   `http://tester:not-a-real-token-0001@example.invalid/repo.git` fail without the token in
   the output. `gitPod.repoUrl=https://example.invalid/r@x.git` also fails (no `@` anywhere).
   The default values (`repoUrl: ""`, Git pod off) still render.
9. Validation before logging (D6): in the rendered `git-sync.sh` of the default `COMMON_ARGS`
   render, the line `validate_git_repo_url || exit 1` comes before the line containing
   `Repository: $GIT_REPO_URL` (compare the two line numbers with `grep -n`).

### 4.2 New `oolong/oolong-server/test/git-auth.integration.sh` (+ jest wrapper
`src/tea/git-auth.integration.spec.ts`, same pattern as `git-publish.integration.spec.ts`)

Setup: own `ROOT=$(mktemp -d)` with trap cleanup; `HOME=$ROOT/home`,
`GIT_CONFIG_GLOBAL=$ROOT/gitconfig`, `GIT_CONFIG_NOSYSTEM=1`; a bare repository with a
`products/` commit; `test/git-http-auth-server.py` (python3, stdlib only, run with
`python3 -I`), bound to `127.0.0.1` on an ephemeral port written to a file, logging one line
per request to a log file named on its command line: method, path and `auth=yes` or `auth=no`
(whether an `Authorization` header was present; never the header's value). It takes
`--mode`:

- `backend`: requires HTTP Basic auth `tester:not-a-real-token-0001` (401 with
  `WWW-Authenticate: Basic` otherwise) and serves the bare repository through
  `git http-backend`;
- `redirect --target <base>`: answers every request, authenticated or not, with a 302 to
  `<base><path>`;
- `auth-redirect --target <base>`: answers a request without the correct Basic credentials
  with 401 and `WWW-Authenticate: Basic`, and each request with them with a 302 to
  `<base><path>`.

Each case that inspects a log truncates it right before its clone. Auth files under `$ROOT/auth` (`username`, `token`, written with a
trailing newline). Sources `git-publish.inc.sh` and calls `configure_git_auth` with the chart's
`scripts/git-credential-oolong.sh` and `$ROOT/auth`, then `sync_worktree` / `publish_revision`
directly. Every git call in the test is bounded with `timeout 30` (or the python fallback if
`timeout` is absent) so a regression to prompting fails instead of hanging.

1. `OOLONG_GIT_AUTH_TYPE=none` against the protected server: `sync_worktree` fails; stderr
   contains `terminal prompts disabled`.
2. `token` with correct files: `sync_worktree` clones and `publish_revision` publishes; a
   second commit is pushed to the bare repo and a second `sync_worktree` fetches it.
3. After 2: `grep -rF not-a-real-token-0001` over `$WORK_DIR` (including `.git`),
   `$CONTENT_DIR` (revisions and pointer files) and `$HOME` finds nothing;
   `git -C "$WORK_DIR" remote get-url origin` equals `GIT_REPO_URL`; `env` and every
   `GIT_CONFIG_VALUE_n` exported by `configure_git_auth` do not contain the token.
4. Reset: `credential.helper store` in `$GIT_CONFIG_GLOBAL`, clone succeeds,
   `$HOME/.git-credentials` does not exist.
5. Wrong token file: `sync_worktree` fails with `Authentication failed`, bounded by the
   timeout.
In cases 6, 7 and 8 every "no `Authorization` header" assertion is non-vacuous: the case first
asserts that the log of the server named holds at least one line from that clone, then that
none of those lines has `auth=yes`.

6. Host scope, other name: the `backend` server on `<port>`; `GIT_REPO_URL` uses
   `localhost:<port>`, the helper is configured for it, and a clone of
   `http://127.0.0.1:<port>/...` fails within the timeout; the server log holds at least one
   request from that clone and none with `auth=yes`.
7. Host scope, unauthenticated redirect: a second instance in `redirect` mode on `<port2>`
   with target `http://127.0.0.1:<port>`; with `GIT_REPO_URL` on `localhost:<port2>` and the
   helper configured for it, `sync_worktree` fails within the timeout; the `backend` log holds
   at least one request from that clone and none with `auth=yes`.
8. Host scope, redirect after authentication (F-1): a second instance in `auth-redirect` mode
   on `<port2>` with target `http://127.0.0.1:<port>`; `GIT_REPO_URL` on `localhost:<port2>`,
   the helper configured for it with the correct auth files. `sync_worktree` fails within the
   timeout. The `auth-redirect` log holds a request with `auth=yes` (the token was really sent
   to the configured host, and the redirect followed an authenticated request). The `backend`
   log holds at least one request from that clone and none with `auth=yes`. `$HOME` and the
   work directory do not contain `not-a-real-token-0001` afterwards.
9. File validation in `configure_git_auth` (returns 1, stderr names the file and does not
   contain the file's content): missing auth dir; empty `token`; `token` with two lines;
   `token` with a CRLF ending; empty `username`.
10. Type validation: `OOLONG_GIT_AUTH_TYPE` empty, `ssh`, `Token` each return 1.
11. Path validation: a helper path containing a space, and an auth dir containing `;`, are
    refused before any config is exported.
12. Helper actions: `git-credential-oolong.sh <dir> store` and `erase` print nothing and exit
    0; `get` with a two-line token prints nothing on stdout and exits 1.

The existing `test/git-publish.integration.sh` (including the userinfo refusal of
`validate_git_repo_url`) stays unchanged and green.

### 4.3 Suites and checks

- In `oolong/oolong-server` with Node 24: `npm ci`, `npm run lint`, `npm run build`, `npm test`
  (runs both shell suites through their jest wrappers; needs `helm`, `git` with
  `git-http-backend`, `python3`). `npm run test:e2e` is unaffected by this change but is run
  as usual.
- `helm lint helm-chart/oolong --set image=oolong-server:local` with `auth.type=none` and with
  `auth.type=token,auth.existingSecret=x`.
- Live check of the pinned image when Docker is available: `docker run --rm --entrypoint
  /bin/sh alpine/git:2.54.1@sha256:6a1f... -c 'type printf; type read; git --version'` reports
  `printf` and `read` as shell builtins and git 2.54.1. Optional cluster check (kind or
  similar): install with `type: token` and a Secret missing the `token` key; the Git pod does
  not start and the event names the missing key.

## 5. Risks and open points

- **Fits one coder pass.** Chart and shell changes plus two test files and docs, about half a
  day as estimated. No split; signed off with `--no-split`.
- **Behaviour change (D5):** values that set one of the seven previously dropped names now fail
  the render. Chart is pre-1.0 (0.0.9); README calls it out.
- **Behaviour change (D6):** a `gitPod.repoUrl` containing `@` used to render and then
  crash-loop the Git pod; it now fails the render. Nothing that synced before stops working.
- **Redirect behaviour is git's and libcurl's (D4).** 4.2 case 8 runs against the git on the
  test host, not the pinned image's git 2.54.1; the image's git is newer than the prototyped
  2.43 and the behaviour (no `CURLOPT_UNRESTRICTED_AUTH`, credentials matched per URL) is long
  standing. The case is not repeated inside the image (it has no python3 for the test server).
- **D2, no chart-created Secret.** The task allowed one as optional; it is left out because the
  token would then live in a values file or a `--set` argument and in the Helm release record
  ("Secrets and credentials": secrets come from the environment or Secrets, never a values
  default). It can be added later as an additive `auth.token.create` without reshaping.
- **fsGroup dependency:** the Secret files are readable by uid 10001 through `fsGroup`. An
  operator who removes `podSecurityContext.fsGroup` gets a start-up failure from
  `configure_git_auth` naming the unreadable file (fails closed). README notes it.
- **Rotation latency:** kubelet propagates a changed Secret to the mount after its sync period
  (typically about a minute); syncs in that window use the old token and may fail and retry,
  while the previous revision keeps serving.
- **For OOL-2:** add `ssh` to the enum in the schema, `oolong.gitAuthType` and
  `configure_git_auth`; add `auth.ssh.{privateKeyKey,knownHostsKey}` and items in the same
  `git-auth` volume; relax the https-only checks (`oolong.requireGitRepoUrl`,
  `validate_git_repo_url`, `protocol.ssh.allow`) only when the type is `ssh`. `GIT_SSH` and
  `GIT_SSH_COMMAND` are already reserved.
- No open questions for the operator.
