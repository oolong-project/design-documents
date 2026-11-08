# OOL-1 test run 2: Git pod clones private repositories over HTTPS with a token

Task: OOL-1. Verdict: **PASSED**, nothing OPEN. T-1 from run 1 is RESOLVED.

Inputs: `impl/OOL-1/notes-2.md` (DETAILED_DESIGN round 2, answering T-1), `impl/OOL-1/notes-1.md`
for the rest of the change, `design/OOL-1/architecture-2.md`, `design/OOL-1/threat_model-2.md`,
coding principles read at `145008cdcc23`.

## What was tested

| PR | Head tested | Base tip | Merge |
|---|---|---|---|
| https://github.com/oolong-project/oolong/pull/4 | `ae0fbf560988b2bd56860a776978493c0f834393` | `2026-11-oolong` @ `f216f6867c98f4f69826027bacd68bef258e30b0` | base tip is an ancestor of the head; the scratch merge is a no-op, so the tree tested is `ae0fbf5` |

Run 1 tested `9c03937`, an ancestor of the new head; the only change since is
`helm-chart/oolong/scripts/chart-render.integration.sh` (6 lines added, 5 removed). Nothing was
rebased. tea-client-go is not changed by OOL-1 (its `2026-11-oolong` @ `82d03da` holds only
`LICENSE`).

CI on the head: "Build And Push Oolong Server Container Image" and "Build And Push Oolong Helm
Chart" both completed with success (GitHub check runs on `ae0fbf5`).

Tools: Node v24.14.1, Helm v4.2.3, host git 2.43.0, k3s single node with Traefik. Docker is not
usable on the host, so the server image is the one CI built from the PR branch (server code is
unchanged by the PR). Go steps are skipped as in run 1: the tea-client-go base has no Go code.

## Results

| # | Check | Result |
|---|---|---|
| 1 | `npm ci` | passed |
| 2 | `npm run lint` | exit 0; its `--fix` rewrites `src/tea/pagination.spec.ts` and `src/tea/representation.spec.ts`, neither touched by the PR (unchanged from run 1, same on the base) |
| 3 | `npm run build` | passed |
| 4 | `npm test` | 13 suites, 57 tests passed |
| 5 | `npm run test:e2e` | 4 suites passed, 1 skipped; 17 tests passed, 1 skipped |
| 6 | `npm run format` | changes only the same two non-PR spec files |
| 7 | `sh test/git-auth.integration.sh` (design 4.2 cases 1 to 12) | 12 of 12 passed |
| 8 | `sh test/git-publish.integration.sh` | 10 of 10 passed |
| 9 | `sh helm-chart/oolong/scripts/chart-render.integration.sh` with a private `TMPDIR` | 20 of 20 cases, exit 0, `all chart render integration tests passed`, no `Permission denied` |
| 10 | Same script with the default shared temp directory | 20 of 20, exit 0 |
| 11 | Same script under `strace -f -e trace=openat,creat,open` with a private `TMPDIR` | 0 opens for writing under `/tmp` outside the private directory (74 inside it); `grep -n '/tmp'` over the script prints nothing; the listing of `oolong*` entries in `/tmp` is identical before and after |
| 12 | The five older refusal cases fail for the reason they name (run by hand on the head) | empty `image=`: `image is required`; 25.10.6: `image 25.10.6 is incompatible`; `$(...)` retention and retention 0: schema refusal; `http://` URL: `gitPod.repoUrl must be an https:// URL` (5 checks) |
| 13 | `helm lint` with `auth.type=none` and with `auth.type=token,auth.existingSecret=x` | both passed |
| 14 | Plain-ASCII sweep over the 14 changed files | no non-ASCII byte in any added line; `docs/tea-upgrade/security-assessment.md` keeps pre-existing non-ASCII lines from the base |
| 15 | No edits in `src/generated-nest/` or `tea-spec/` | passed |
| 16 | Path guard, timestamps, error bodies | not applicable: no server code changes |
| 17 | `gofmt -l`, `go build`, `go vet`, `go test` in tea-client-go | skipped: the base holds no Go code |
| 18 | Live check in the cluster (below) | 32 of 32 passed |

Counts in the index: 164 passed (items 1 to 3, 6, 13 to 15: 8; jest and e2e tests 57 + 17; shell
cases 12 + 10 + 20; T-1 checks 10, 11 and the five of 12: 7; live checks 32), 0 failed, 5
skipped (the four Go steps and the e2e suite's one skipped test). The render refusal matrix of
run 1 was not repeated: the chart templates, schema and values are byte-identical between
`9c03937` and `ae0fbf5` (the Helm upgrade below renders the same manifests).

## T-1: chart-render.integration.sh fails on a clean host; fixed /tmp outputs hide it (RESOLVED)

Both parts are fixed on `ae0fbf5`. The five older refusal cases write to one `mktemp` file
(`CASE_OUT`), which follows `TMPDIR`, so a write that cannot be opened no longer passes a case
without running helm; no fixed path under `/tmp` is left (items 10 and 11). The first case now
renders with `--set image=` and the chart refuses it with `image is required`, so it holds with
the default image in `values.yaml` (item 12). The script runs all 20 cases and passes both with a
private `TMPDIR` and with the shared one, and `npm test`'s `chart-render.integration.spec.ts`
passes for the right reason.

Observed, not filed: the script's `mktemp` files are not removed on exit (no `trap`), so each run
leaves small files in the temp directory. This does not affect a result.

## Live checks

All in the local k3s cluster; nothing ran against a shared deployment. Credentials are throwaway
values generated for these runs and are not reproduced here.

**Setup.** The private Gitea from run 1 (namespace `oolong-tester-gitserver`,
`https://oolong-tester-git.psclaude.rearmhq.com/`, publicly trusted certificate), private
repository `oolong-admin/tea-fixtures`, read-only user `oolong-reader` with a token scoped
`read:repository`. Release `ool1` in `oolong-tester-ool-1` upgraded with
`helm upgrade --reuse-values` from a `git archive` of `ae0fbf5` (`helm-chart/oolong`): revision 2,
`auth.type=token`, `auth.existingSecret=oolong-git-auth`, `allowedHosts` the Gitea host,
`syncInterval=15`. The rendered manifests equal revision 1's apart from the chart-managed page
Secret, which the chart re-renders from its existing value. Both Deployments were then restarted
so every check ran in fresh pods. Server image: the `ool-1-git-https-token-auth` CI build (server
code unchanged by the PR); Git pod image: the chart's pinned `alpine/git:2.54.1`.

| # | Check | Result |
|---|---|---|
| L1 | Host: anonymous clone | refused: `terminal prompts disabled` |
| L2 | Host: clone with the right user and a wrong token | refused: `Authentication failed` |
| L3 | Host: clone with the read-only token | succeeds |
| L4 | Host: push with the read-only token | refused: `Forbidden` |
| L5 | Fresh Git pod clones with the token from the mounted Secret and syncs | passed; log shows `Auth: token` and the plain repository URL |
| L6 | New commit `044b6e2` pushed | published within one sync interval |
| L7 | Server `/.well-known/tea`, `/v1.0.0/products`, `/healthz` | 200 each; products lists the five fixture products |
| L8 | Token absent from served content | 0 hits |
| L9 | Token absent from: Git pod log, server log, pod specs, Deployments and ConfigMaps, `helm get all`, the decoded Helm release Secrets, namespace events | 0 hits each (7 checks) |
| L10 | Token absent from `/content` (revisions, `_work` with `.git`) and `$HOME`; no `.git-credentials` anywhere in the pod | 0 files each (2 checks) |
| L11 | `git remote get-url origin` and `.git/config` | plain HTTPS URL; neither the token nor the user name in `.git/config` (2 checks) |
| L12 | Token absent from every process environment, and from 34682 process command lines sampled over 30 s of syncs | 0 hits each (2 checks) |
| L13 | Secret mount | `/etc/oolong-git-auth` read-only tmpfs, files `0440` group `10001` |
| L14 | Secret replaced with a wrong token | syncs fail with `Authentication failed` after kubelet propagation (seen twice, at about 60 s); the wrong value is not logged |
| L15 | While failing | the server keeps serving the last revision (200) |
| L16 | Commit `168fe44` pushed while the pod was failing | not published while failing; published on the first successful sync after the good token propagated |
| L17 | Real rotation: a new read-only token created, put in the Secret, the old token revoked in Gitea | old token refused (`ls-remote` exit 128), new token accepted; the pod keeps syncing successfully after the revocation (2 checks) |
| L18 | Restarts across L14 to L17 | 0, same pod: rotation is picked up without a restart (design D1) |
| L19 | Neither the old nor the new token in the Git pod log, pod specs, Deployments, ConfigMaps, `/content` or `$HOME` after rotation | 0 hits (2 checks) |

The first attempt at L14 to L17 also ran host-side git steps that picked up the test host's own
global credential helper and so did not use the intended credentials; those steps were discarded
and re-run with an isolated `HOME`. The results above are from the re-run and the Git pod log.

## Notes

- **Private CA (unchanged from run 1, not filed).** Design 3.6 says a private CA is supported with
  `GIT_SSL_CAINFO`, but the Git pod has no volume to mount a CA file from; round 2 did not change
  this. A candidate follow-up task (for example a `gitPod.caBundle`), outside OOL-1's scope.
- The image tagged `alpine/git:2.54.1` ships git 2.54.0 (run 1, L23); no effect on this change.
- Models: a sonnet sub-agent ran items 1 to 11 and 13 to 15 and its output was checked here;
  item 12, the review of the round-2 diff, the live check and the verdict are this hop's own
  (opus).
- The cluster resources (`oolong-tester-gitserver`, release `ool1` in `oolong-tester-ool-1`) are
  left in place for reuse.
