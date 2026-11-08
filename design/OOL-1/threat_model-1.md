# OOL-1 threat model, round 1: Git pod HTTPS token authentication

Task: OOL-1. Design: `design/OOL-1/architecture-1.md` (round 1). The change adds a credential
to the Git pod; the public read API, the server and the Go client do not change.

## 1. What is exposed

Assets:

- **A1, the Git token** (and its username): read access to the private content repository,
  more if the operator over-scoped it.
- **A2, served content**: revisions on the content PVC, served publicly by the server. It must
  never contain A1.
- **A3, sync availability**: the Git pod holds the publish `flock` while it syncs.

Trust boundaries the change crosses:

- **B1, Kubernetes Secret to Git pod**: the operator's Secret is mounted read-only at
  `/etc/oolong-git-auth` (files, not env).
- **B2, Git pod to Git host**: HTTPS; the host, and anything it redirects to, is outside the
  operator's control.
- **B3, values author to chart render**: `gitPod.auth` and `gitPod.env` shape the pod.
- **B4, Git pod to content PVC to public API**: unchanged; the publish step copies the work
  tree without `.git`.

Not in scope: hostile repository content (existing SEC-02 boundary, unchanged), Git-pod egress
control (existing NetworkPolicy guidance, unchanged), a hostile operator (controls the image
and the Secret already).

## 2. Threats and mitigations

| Id | Threat | Mitigation in the design |
| --- | --- | --- |
| T1 | A1 lands in the remote URL, `.git/config`, `FETCH_HEAD`, a command line, the pod spec or a log | Token is read only by the credential helper from a mounted file and printed with the shell builtin `printf` to git's stdin pipe (arch 3.5); URL userinfo stays refused (`validate_git_repo_url`, unchanged); the pod spec carries only the Secret name; logs print the auth type only (arch 3.3) |
| T2 | Another credential helper (system or global `store`/`cache`) persists A1 to disk | First `GIT_CONFIG_*` entry resets `credential.helper` to empty; prototyped: without it `store` wrote `~/.git-credentials` (arch 3.4, D3); our helper ignores `store`/`erase` |
| T3 | A1 is offered to another host: a redirect, a different host name, or a downgrade to `http://` | Helper scoped to `credential.<scheme>://<hostport>` of `gitPod.repoUrl` (arch 3.4, D4); `protocol.http.allow never` (existing) blocks plaintext redirects; libcurl does not resend auth across hosts |
| T4 | `gitPod.env` leaks A1 (`GIT_TRACE_CURL` with `GIT_TRACE_REDACT=0`, `GIT_CURL_VERBOSE`), overrides the wiring (`GIT_CONFIG_*`, `GIT_ASKPASS`, `GIT_ALLOW_PROTOCOL`) or sends A1 to an unverified peer (`GIT_SSL_NO_VERIFY`) | Reserved names and prefixes fail the render (arch 3.6, D5) |
| T5 | A1 reaches A2 | The Secret mount is not on the PVC; the token is never written to the work tree; publish skips `.git` (existing) (arch 3.5) |
| T6 | Missing or bad credentials make git wait for a prompt while holding the publish lock (A3) | `GIT_TERMINAL_PROMPT=0` for every type; askpass variables reserved (arch 3.4, 3.6) |
| T7 | A malformed Secret value (embedded newline) injects attributes into git's credential protocol | Helper and start-up check accept one line only, no CR (arch 3.4 step 4, 3.5) |
| T8 | A1 sits in the process environment of every git child and in `kubectl exec ... env`; a rotated token keeps the old value until restart | Secret mounted as files, whole-directory mount so rotation propagates (arch 3.3, D1); render asserts no `secretKeyRef` |
| T9 | A half-configured `gitPod.auth` silently falls back to anonymous, or an unknown type is accepted | Schema plus template guard refuse it, including with `--skip-schema-validation` (arch 3.2, 3.3) |
| T10 | Over-scoped token widens the impact of a leak | Docs: read-only, single-repository scope per provider (arch 3.7). Operator guidance, no test |

## 3. Mitigations that need a test

All are in the architecture's test list (section 4):

- T1: 4.2 case 3 (no token in work tree, `.git`, PVC, `$HOME`, env or exported config; remote
  URL unchanged); 4.1 case 2 (no token env in the render). That the token is never an argument
  of an executed program is by construction (builtins only, arch 3.5) and checked in review and
  by the image check in 4.3, not by a process-table test, which would be timing-dependent.
- T2: 4.2 case 4.
- T3: 4.2 cases 6 and 7.
- T4: 4.1 case 5.
- T5: 4.2 case 3 (revisions and pointer files).
- T6: 4.2 cases 1 and 5 (bounded by `timeout`).
- T7: 4.2 cases 8 and 11.
- T8: 4.1 case 2 (volume, no `subPath`, no `secretKeyRef`).
- T9: 4.1 cases 3 and 4; 4.2 case 9.
