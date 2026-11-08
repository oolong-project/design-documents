# OOL-1 test run 1: Git pod clones private repositories over HTTPS with a token

Task: OOL-1. Verdict: **REJECTED**, one review item OPEN (T-1, priority 2, about
DETAILED_DESIGN).

Inputs: `impl/OOL-1/notes-1.md` (DETAILED_DESIGN round 1), `design/OOL-1/architecture-2.md`,
`design/OOL-1/threat_model-2.md`, coding principles read at `145008cdcc23`.

## What was tested

| PR | Head tested | Base tip | Merge |
|---|---|---|---|
| https://github.com/oolong-project/oolong/pull/4 | `9c03937d31c1b0ef20b2fd111ff03bfada169a3f` | `2026-11-oolong` @ `f216f6867c98f4f69826027bacd68bef258e30b0` | base tip is an ancestor of the head; the scratch merge is a no-op, so the tree tested is `9c03937` |

tea-client-go is not changed by OOL-1 (its `2026-11-oolong` @ `82d03dac` holds only `LICENSE`).

CI on the head: "Build And Push Oolong Server Container Image" and "Build And Push Oolong Helm
Chart" both completed with success (GitHub check runs on `9c03937`).

Tools: Node v24.14.1, Helm v4.2.3, git 2.43.0 and python3 on the test host; k3s single node
with Traefik for the live check. Docker was not usable on the host (no socket access), so the
server was not built locally; the live check uses the server image CI built from the PR branch
(see Live checks). Go is not installed on the host; the Go steps ran in a `golang:1.25` pod.

## Results

| # | Check | Result |
|---|---|---|
| 1 | `npm ci` | passed |
| 2 | `npm run lint` | exit 0; its `--fix` rewrites `src/tea/pagination.spec.ts` and `src/tea/representation.spec.ts`, neither touched by the PR (same on the base, as the coder noted) |
| 3 | `npm run build` | passed |
| 4 | `npm test` | 13 suites, 57 tests passed on this host; but see T-1: the chart-render suite passes here only because five refusal cases pass vacuously, and it fails when they run |
| 5 | `npm run test:e2e` | 4 suites passed, 1 skipped; 17 tests passed, 1 skipped |
| 6 | `npm run format` | changes the same two non-PR spec files; no PR file changes |
| 7 | `sh test/git-auth.integration.sh` (design 4.2 cases 1 to 12) | 12 of 12 passed |
| 8 | `sh test/git-publish.integration.sh` (unchanged suite) | 10 of 10 passed |
| 9 | `sh helm-chart/oolong/scripts/chart-render.integration.sh` | exit 0 on this host, vacuously in part; **fails** at its first case when its fixed `/tmp` outputs are writable (T-1). With only those paths moved to a private directory and the first case skipped, every other case passes, including the new 4.1 cases 1 to 9, the checksum probe and the lint cases |
| 10 | `helm lint` with `auth.type=none` and with `auth.type=token,auth.existingSecret=x` | both passed |
| 11 | Plain-ASCII sweep over the 14 changed files | no non-ASCII byte in any added line; `docs/tea-upgrade/security-assessment.md` keeps pre-existing non-ASCII lines from the base |
| 12 | No edits in `src/generated-nest/` or `tea-spec/` | passed |
| 13 | Path guard, timestamps, error bodies | not applicable: no server code changes |
| 14 | `gofmt -l`, `go build`, `go vet`, `go test` in tea-client-go | skipped: the base holds no Go code (`./...` matches no packages) |
| 15 | Render refusal matrix (below) | 38 of 38 passed |
| 16 | Live check in the cluster (below) | 24 of 24 passed |

Counts in the index: 166 passed (items 1 to 3, 6, 10 to 12 once each; jest tests, shell cases,
matrix runs and live checks one each: 57 + 17 + 12 + 10 + 38 + 24), 1 failed (T-1, item 9), 5
skipped (the four Go steps and the e2e suite's one skipped test).

## T-1: chart-render.integration.sh fails on a clean host; fixed /tmp outputs hide it

Priority 2, about DETAILED_DESIGN. `helm-chart/oolong/scripts/chart-render.integration.sh`
(touched by this PR) still writes five older refusal cases to fixed paths
(`/tmp/oolong-helm-empty.out`, `-old.out`, `-inject.out`, `-range.out`, `-url.out`). On this
host those files belong to another user, the redirect fails with `Permission denied`, and the
`if helm template ... >file; then fail` pattern reads the failed redirect as a refusal, so all
five pass without running helm. The coder's note flags this.

When they do run (the same script with only `/tmp/` replaced by a private directory, nothing
else changed), the first case fails, on the PR head and on the base alike:

```
test helm template requires a compatible image
ASSERT FAILED: helm template without image should fail
```

`values.yaml` on the base now sets a compatible default `image:` (since `f57be27`), so a render
without `--set image` succeeds. Because the script stops at its first failure, on any host where
`/tmp` is writable (a clean CI runner or developer machine) `npm test` fails in
`chart-render.integration.spec.ts`, and none of the PR's new render cases (design 4.1 cases 1 to
9, including 4.1 case 7, "the existing render refusal of an `http://` `gitPod.repoUrl` stays
green") is reached. The failure predates the PR, but the PR changes this script and its new
cases depend on it, and the green `npm test` reported for the head holds only on a host where the
redirect fails. Coding principles, "Tests own their data and run locally": a test creates the
content it reads in a temporary directory; these five do not.

Expected: the five older cases write to `mktemp` files like the new ones, and the first case
asserts what the chart refuses today (for example an empty `--set image=`), so the whole script
runs and passes on a clean host.

## Render refusal matrix

Run on the PR head with `--set image=x:1 --set gitPod.enabled=true --set
gitPod.repoUrl=https://example.invalid/r.git`, each value once with schema validation and once
with `--skip-schema-validation` (results: rc and the first error line).

| Values | Schema | `--skip-schema-validation` |
|---|---|---|
| `auth.type=token`, no `existingSecret` | refused | refused: `gitPod.auth.existingSecret is required when gitPod.auth.type is token` |
| `type=none` with `existingSecret=x` | refused | refused: `... must be empty when gitPod.auth.type is none` |
| `type=ssh`, `type=Token`, `type=` (empty), `type=null` | refused (null: `missing property 'type'`) | refused: `gitPod.auth.type must be none or token` |
| `auth=null` | refused by the template guard | refused |
| `token.tokenKey=` / `token.usernameKey=` empty | refused | refused, naming the key |
| `auth.token=null` with `type=token` | refused | refused |
| `token.usernameKey='a b'` | refused | renders (design: pattern rules are schema-only) |
| `existingSecret=Bad_Name` | refused | renders (same) |
| misspelt `auth.existingsecret` | refused | refused (the real `existingSecret` is then empty) |
| `type=token` with `existingSecret=x` | renders | renders |
| defaults | renders | renders |
| `repoUrl=https://tester:not-a-real-token-0001@example.invalid/r.git` | refused, output has no token | refused naming `gitPod.repoUrl`, output has no token |
| `gitPod.env.GIT_TRACE_CURL=1` | refused naming it | refused naming it |
| `gitPod.env.GIT_SSL_CAINFO=/x.pem` | renders | renders |

`helm lint` with `type=token` and no Secret name, and with a Secret name and `type: none`, both
fail with `at '/gitPod/auth': 'allOf' failed`.

## Live checks

All in the local k3s cluster; nothing ran against a shared deployment. Throwaway credentials were
generated for this run and are not reproduced here.

**Private git server.** Namespace `oolong-tester-gitserver`: Gitea 1.24.6 (rootless, SQLite,
registration off, sign-in required to view), exposed through Traefik at
`https://oolong-tester-git.psclaude.rearmhq.com/`. Traefik's Let's Encrypt resolver issued a
publicly trusted certificate for that name (issuer Let's Encrypt, `CN =
oolong-tester-git.psclaude.rearmhq.com`), so the chart's default trust store applies and no
private CA was needed. Private repository `oolong-admin/tea-fixtures` holds a copy of
`oolong/fixtures/tea-1.0.0` from the PR head. User `oolong-reader` is a read-only collaborator;
its token has the single scope `read:repository`.

Server-side checks from the test host (git 2.43, no helper but the one under test):

| # | Request | Result |
|---|---|---|
| L1 | anonymous clone | refused: `terminal prompts disabled` |
| L2 | clone with the right user and a wrong token | refused: `Authentication failed` |
| L3 | clone with the read-only token | succeeds |
| L4 | push with the read-only token | refused: HTTP 403 |

**Chart.** Installed from the PR head checkout (`helm upgrade --install ool1 helm-chart/oolong`)
into `oolong-tester-ool-1` with `gitPod.enabled=true`, `repoUrl` the Gitea URL,
`allowedHosts` that host, `syncInterval=15`, `auth.type=token`,
`auth.existingSecret=oolong-git-auth` (created with `--from-file` for `username` and `token`).
Server image `registry.rearmhq.com/library/oolong-server@sha256:3952e68e...` (the
`ool-1-git-https-token-auth` branch tag CI pushed; the server code is unchanged by the PR). The
Git pod runs the chart's pinned `alpine/git:2.54.1@sha256:6a1fdb96...`.

| # | Check | Result |
|---|---|---|
| L5 | Git pod clones, publishes revision `41c5f88`, repeats syncs | passed; log shows `Auth: token` and the plain repository URL |
| L6 | Server serves the fixtures from the PVC | `/.well-known/tea` 200; `/v1.0.0/products` 200 with the five fixture products; 18 of 19 crawled requests 200, the one 404 is a deliberate unknown TEI in `/discovery` |
| L7 | Token absent from: Git pod log, server log, pod specs, Deployments and ConfigMaps, `helm get all`, the Helm release Secret (decoded), namespace events | 0 hits each |
| L8 | Token absent from `/content` (revisions, pointer files, `_work` with `.git`) and `$HOME` in the Git pod | 0 files; no `.git-credentials` |
| L9 | `.git/config` and `git remote get-url origin` | plain HTTPS URL, no user name or token |
| L10 | Token absent from every process environment in the Git pod | 0 hits; the only auth env name is `OOLONG_GIT_AUTH_TYPE` |
| L11 | Token absent from process command lines | 15394 command lines sampled over 50 s of syncs (git fetch and `git-remote-https` caught), 0 with the token |
| L12 | Token absent from served content (all bytes of the crawl) | 0 hits; the user name is not served either |
| L13 | Secret mount | `/etc/oolong-git-auth` read-only tmpfs, files `0440` group `10001` |
| L14 | New commit pushed | published within one sync interval |
| L15 | Secret replaced with a wrong token (rotation to a bad value) | syncs fail with `Authentication failed` 62 s later (kubelet propagation), no token in the log |
| L16 | While failing | the server keeps serving the last revision (200); a commit pushed meanwhile is not published |
| L17 | Real rotation: new read-only token in the Secret, old token revoked in Gitea | recovered and published the pending commit 15 s later |
| L18 | Restarts across L14 to L17 | 0, same pod: rotation is picked up without a restart, as the design (D1) and NOTES.txt claim |
| L19 | No token from either value in `/content` or `$HOME` after rotation | 0 files |
| L20 | A Secret without the `token` key (second release, removed afterwards) | Git pod stays `ContainerCreating`; event `references non-existent secret key: token` (design 4.3 optional check) |
| L21 | A Secret whose token has two lines (second release, removed afterwards) | crash loop with `ERROR: /etc/oolong-git-auth/token is missing, empty or not a single line`; content not logged |
| L22 | Pinned image: `type printf; type read; type echo` | all shell builtins (busybox ash) |
| L23 | Pinned image: `git --version` | `git version 2.54.0`: the image tagged `2.54.1` ships git 2.54.0, not 2.54.1 as the design says; no effect on this change, which needs git 2.31+ for `GIT_CONFIG_COUNT` (recorded, not filed) |
| L24 | Gitea access refused for an anonymous API call on the private repository | 403 |

## Notes

- **Private CA.** Not needed here, because the public certificate worked. On reading the chart:
  design 3.6 says "a private CA is supported with `GIT_SSL_CAINFO`, which stays allowed", but
  the Git pod has no way to mount a CA file: its only volumes are the content PVC, the
  script ConfigMap, the `git-home` emptyDir and the `git-auth` Secret with exactly the
  `username` and `token` items. A self-hosted Git server behind a private CA therefore cannot
  be trusted without a custom image. This is outside OOL-1's task (the providers it names use
  public CAs), so it is recorded here and not filed; it is a candidate follow-up task (for
  example a `gitPod.caBundle` from a ConfigMap or Secret), and the design's sentence overstates
  what the chart offers today.
- The redirect cases (4.2 cases 6 to 8) ran against the host's git 2.43, as the design notes.
- Models: the suites (items 1 to 12, 14 first attempt) were run by a sonnet sub-agent and its
  logs checked here; the review of the diff and tests, the live check and the verdict are this
  hop's own.
- The cluster resources (`oolong-tester-gitserver`, release `ool1` in `oolong-tester-ool-1`)
  are left in place for reuse.
