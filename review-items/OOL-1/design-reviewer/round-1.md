# OOL-1 design review, round 1

Task: OOL-1, Git pod: clone private repositories over HTTPS with a username and token from a
Secret. Reviewed: `design/OOL-1/architecture-1.md` (round 1) and
`design/OOL-1/threat_model-1.md` (round 1), against oolong-project/oolong `2026-11-oolong` @
`f216f68` and the coding principles at `145008cdcc23`.

Verdict: REJECTED. Three items are open, all about ARCHITECTURE (F-2 also asks for a line in
the threat model). The design is otherwise sound and close to buildable as written; the items
are small.

## What holds

- Scope matches the task. Server code, the TEA API, the Go client and the content format are
  untouched, and the out-of-scope list (SSH for OOL-2, no chart-created Secret, egress) is
  honest. D2 (no chart-created Secret) follows "Secrets and credentials".
- D1 (Secret mounted as files instead of `secretKeyRef` env) departs from the task's intended
  shape with reasons that hold: rotation without a restart, the token stays out of every child
  process environment, and OOL-2 needs files anyway. Accepted.
- D3 (empty `credential.helper` entry first) is correct: git applies credential config to every
  matching entry in order and an empty value clears the list, and `GIT_CONFIG_*` entries are
  read after all config files. Test 4.2 case 4 exercises it.
- The helper value `/bin/sh <helper> <dir>` starts with `/`, so git runs it as given and appends
  the action; the path check in 3.4 step 2 keeps shell syntax out of it, as "Values are never
  interpolated into shell text" requires.
- The schema and the template guard together refuse every half-configured block, including
  with `--skip-schema-validation`, and the reserved-name list (D5, fail the render) covers the
  trace, askpass, config and protocol overrides the task named plus `GIT_SSL_NO_VERIFY`,
  `GIT_ALLOW_PROTOCOL` and `GIT_TRACE_REDACT`. Failing the render instead of dropping silently
  is the better choice and the behaviour change is documented.
- The threat model fits the exposure: the change adds a credential to a pod and nothing to the
  public read API. The server serves only catalogued artifacts through the path guard, so the
  rest of a private repository's work tree on the PVC is not reachable through the API.

## F-1: The redirect test never has the token in hand when the redirect happens

Security finding, about ARCHITECTURE (section 4.2) and THREAT_MODEL T3.

Test 4.2 case 7 puts a server on `localhost:<port2>` that answers every request, including the
first unauthenticated one, with a 302 to `127.0.0.1:<port>`. Git only fills credentials after a
401, so at the moment of that redirect it holds no credentials for anyone: the case checks that
the helper is not offered for the second host, which case 6 already covers, but it would pass
just the same if git or libcurl forwarded an already-filled token across a redirect. That is
the case T3 relies on "libcurl does not resend auth across hosts" for, and it is the one where
the token can actually leave: the configured host asks for credentials, receives them, and then
redirects the authenticated request elsewhere. It is a real boundary (B2, a Git host the
operator does not control), so it needs a case that exercises the refusal.

The design must add, in 4.2:

- a mode of `test/git-http-auth-server.py` that requires HTTP Basic auth for its own requests
  and answers each authenticated request with a 302 to a given base URL;
- a case: `GIT_REPO_URL` on `localhost:<port2>` in that mode, helper configured for it with the
  correct credentials, redirect to `http://127.0.0.1:<port>`; `sync_worktree` fails within the
  timeout, the redirecting server's log shows a request with an `Authorization` header (so the
  token was really sent to it and the redirect really followed an authenticated request), and
  the `127.0.0.1` server's log shows at least one request and none with an `Authorization`
  header.

Also make the "no `Authorization` header" assertions in cases 6 and 7 non-vacuous: each must
first assert that the server named received at least one request from that clone, so an
assertion over an empty log cannot pass.

## F-2: A token in `gitPod.repoUrl` is logged and stored before the userinfo refusal fires

Security finding, about ARCHITECTURE (sections 3.3 and 4.1) and THREAT_MODEL T1.

The design keeps the userinfo refusal where it is today, in `validate_git_repo_url` at
runtime, and T1 counts it as a mitigation. Now that the chart supports private repositories,
the most likely misconfiguration is an operator writing the token into the URL
(`https://user:<token>@host/org/repo.git`), a common habit. Today that value:

- renders, because `oolong.requireGitRepoUrl` in `_helpers.tpl` checks only the `https://`
  prefix and `values.schema.json` has no entry for `gitPod.repoUrl`, so the token lands in the
  Deployment's `GIT_REPO_URL` env and in the Helm release record;
- is printed to the pod log by `git-sync.sh` (`echo "  Repository: $GIT_REPO_URL"`,
  `git-configmap.yaml` line 48) before `validate_git_repo_url` runs at line 66.

So the refusal comes after the token is already in a log, the pod spec and the release record,
which is what section 1 of the design says must not happen ("Never log or return a secret",
coding principles "Secrets and credentials").

The design must:

- refuse userinfo at render time: `oolong.requireGitRepoUrl` fails when the authority part of
  `gitPod.repoUrl` (between `https://` and the first `/`) contains `@`, with a message that names
  `gitPod.repoUrl` and the rule and does not echo the value; and give `gitPod.repoUrl` a schema
  entry (string, `https://` prefix, no `@` in the authority) as "New chart values come with a
  schema entry" asks;
- in `git-sync.sh`, run `validate_git_repo_url` before the configuration block is printed;
- add to 4.1 a render case with `gitPod.repoUrl=https://tester:not-a-real-token-0001@example.invalid/repo.git`
  that fails (with and without `--skip-schema-validation`) and whose output does not contain
  `not-a-real-token-0001`;
- update T1 to name the render-time refusal and the reordered log line.

## F-3: README line 157 still says the Git pod syncs a public repository

Functional finding (documentation), about ARCHITECTURE (section 3.7).

Section 3.7 lists README lines 57 and 176 for the "public" wording, but line 157 (Git Pod
Configuration intro) also says "populate content from a public HTTPS Git repository". The docs
list must name it too, so the README does not contradict the new "Private repositories"
paragraph.
