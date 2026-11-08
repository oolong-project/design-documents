# OOL-1 implementation notes, round 1: Git pod clones private repositories over HTTPS with a token

Task: OOL-1. Built from `design/OOL-1/architecture-2.md` and `design/OOL-1/threat_model-2.md`
(newest rounds, read after assignment). Coding principles read at `145008cdcc23`.

## Pull request

| PR | Repository | Base | Head commit | Base tip merged |
|---|---|---|---|---|
| https://github.com/oolong-project/oolong/pull/4 | oolong-project/oolong | `2026-11-oolong` | `9c03937d31c1b0ef20b2fd111ff03bfada169a3f` | `f216f6867c98f4f69826027bacd68bef258e30b0` (the base had not moved, so the branch already contains it) |

tea-client-go is not touched. Image build on the head: passed. Merge order: not applicable (one PR).

## What changed

- **Values and schema.** `gitPod.auth` with `type` (`none` | `token`), `existingSecret`,
  `token.usernameKey` / `token.tokenKey`, exactly as design 3.1 and 3.2. `gitPod.repoUrl` gets a
  schema entry whose `not: {pattern: "@"}` makes Helm print no value on refusal (D6).
- **Templates.** `oolong.gitAuthType` (validates the type and the block, so it also holds with
  `--skip-schema-validation`), `oolong.requireGitPodEnvNotReserved` (D5), the `@` guard in
  `oolong.requireGitRepoUrl` (D6). `git-deployment.yaml` always sets `OOLONG_GIT_AUTH_TYPE` and,
  for `token`, mounts the Secret as the `git-auth` volume at `/etc/oolong-git-auth` (whole
  directory, read-only, `defaultMode: 0440`, `optional: false`, items to `username` and `token`).
  The `gitPod.env` loop emits every entry it is given. `git-configmap.yaml` carries the new
  helper and `git-sync.sh` now validates the URL and calls `configure_git_auth` before it prints
  the configuration block, which gains `Auth: <type>`.
- **Scripts.** New `scripts/git-credential-oolong.sh` (POSIX sh, builtins only, `get` reads one
  line from each file, other actions do nothing). New `configure_git_auth` in
  `scripts/git-publish.inc.sh` (prompt off, path checks, type check, file check, origin from
  `GIT_REPO_URL`, `GIT_CONFIG_*` with the empty `credential.helper` reset first).
- **Docs.** README (public wording removed, Private repositories section, values list),
  security assessment (SEC-08 note and Secrets section), `NOTES.txt`, values comments.

## Departures from the design

1. **One reader for the credential files.** Design 3.4 step 4 says `configure_git_auth` checks the
   files "with the same reader the helper uses". It does so by running the helper itself with
   `get`, stdin from `/dev/null` and stdout discarded, so there is a single implementation and the
   token is never an argument. The helper's stderr message (the path, never content) is what the
   operator sees. Behaviour is as designed.
2. **Test server options.** `test/git-http-auth-server.py` also takes `--bind` and `--port`, and
   reads `TEST_GIT_USER` / `TEST_GIT_TOKEN` when set. Defaults are the designed ones (`127.0.0.1`,
   ephemeral port, the made-up credentials). They exist so the same server could run in a cluster
   for the live check below, with throwaway credentials.
3. Case 4 of the shell suite adds a control: the same clone with only the scoped helper entry
   (no reset entry) does write `$HOME/.git-credentials`, so the assertion that the reset prevents it
   is not vacuous. Case 9 also asserts that a token file without a trailing newline is accepted.

## Tests

Added:

- `helm-chart/oolong/scripts/chart-render.integration.sh` (run by `src/tea/chart-render.integration.spec.ts`):
  design 4.1 cases 1 to 9. New cases write their output to `mktemp` files.
- `oolong/oolong-server/test/git-auth.integration.sh` with `test/git-http-auth-server.py` and the
  jest wrapper `src/tea/git-auth.integration.spec.ts`: design 4.2 cases 1 to 12 (the file prints
  `test 1` to `test 12`).

Run (Node 24, `helm`, `git` with `http-backend`, `python3`):

```bash
cd oolong/oolong-server
npm ci && npm run lint && npm run build && npm test && npm run test:e2e
sh test/git-auth.integration.sh
sh ../../helm-chart/oolong/scripts/chart-render.integration.sh
helm lint ../../helm-chart/oolong --set image=oolong-server:local --set gitPod.auth.type=token --set gitPod.auth.existingSecret=x
```

Results on the head: `npm test` 13 suites, 57 tests passed; `npm run test:e2e` 4 suites passed
(1 skipped, as before); lint, build and `helm lint` (both auth types) clean; plain ASCII check on
every touched file prints nothing. `npm run lint` rewrites two unrelated spec files
(`pagination.spec.ts`, `representation.spec.ts`) through `--fix`; that rewrite is not part of the PR.

## Live check in a cluster (design 4.3, optional)

Chart installed with `gitPod.auth.type=token` against a private HTTPS git server (a pod running
`test/git-http-auth-server.py` behind a publicly trusted certificate, throwaway credentials):

- Clone, publish and repeated syncs succeed; the pod does not restart.
- The token is in none of: `/content`, the pod's home directory, the process environment, the pod
  spec, the pod log. No `.git-credentials` file. `git remote get-url origin` is the plain URL.
- The Secret mounts read-only at `/etc/oolong-git-auth` on a tmpfs.
- Rotation: replacing the Secret with a wrong token made syncs fail with `Authentication failed`
  within about a minute; restoring it recovered within about 15 seconds, with zero restarts.
- A Secret without the configured key: the pod stays in `ContainerCreating` and the event reads
  `references non-existent secret key: <key>`.

## For the tester

- Everything above reproduces locally with the commands under Tests; no cluster is needed.
- **Existing refusal cases in `chart-render.integration.sh` write to fixed `/tmp/oolong-helm-*.out`
  paths** (not changed here). On a host where another user already owns one of those files the
  redirect fails, and `if helm template ...; then` reads that as a refusal, so the case passes
  without running helm. My new cases use `mktemp`. Run the script as a user that owns, or can
  remove, those files to exercise the older cases.
- The redirect cases (shell cases 7 and 8) run against the git on the test host, not the pinned
  image's git, as the design notes.
- Nothing is OPEN from the design review.
