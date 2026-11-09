# OOL-2 test run 1: Git pod clones private repositories over SSH with a deploy key and a required known_hosts

Task: OOL-2. Verdict: **PASSED**, nothing OPEN. No review items.

Inputs: `impl/OOL-2/notes-1.md` (DETAILED_DESIGN round 1), `design/OOL-2/architecture-2.md`,
`design/OOL-2/threat_model-2.md`, `review-items/OOL-2/design-reviewer/round-2.md`; coding
principles read at `145008cdcc23`.

## What was tested

| PR | Head tested | Base tip | Merge |
|---|---|---|---|
| https://github.com/oolong-project/oolong/pull/5 | `ac4d97bf1ab9be9da7fff6e8a2cecaab3790f7cc` | `2026-11-oolong` @ `0fac8166b6ff` (merge of OOL-1, PR 4) | the base tip is an ancestor of the head, so the scratch merge is a no-op and the tree tested is `ac4d97b`; GitHub reports the PR mergeable |

tea-client-go is not changed by OOL-2 (its `2026-11-oolong` @ `82d03da` holds only `LICENSE`), so
the Go steps are skipped as in OOL-1. CI on the head: "Build And Push Oolong Server Container
Image" and "Build And Push Oolong Helm Chart" both completed with success.

Tools: Node v24.14.1, Helm v4.2.3, host git 2.43.0, host OpenSSH client and `/usr/sbin/sshd`, k3s
single node. Docker is not usable on the host. The PR changes no server source (only chart,
scripts, tests and docs), so the live check uses the server image CI built for OOL-1 and the
chart's pinned `alpine/git:2.54.1` Git pod image.

## Results

| # | Check | Result |
|---|---|---|
| 1 | `npm ci` | passed |
| 2 | `npm run lint` | exit 0; its `--fix` rewrites only `src/tea/pagination.spec.ts` and `src/tea/representation.spec.ts`, not in the PR (same as OOL-1) |
| 3 | `npm run build` | passed |
| 4 | `npm test` | 14 suites, 58 tests passed; `git-ssh-auth`, `git-auth`, `git-publish` and `chart-render` wrappers each ran and passed (none skipped) |
| 5 | `npm run test:e2e` | 4 suites passed, 1 skipped; 17 tests passed, 1 skipped (`conference-git-live`, skipped as on the base) |
| 6 | `npm run format` | changes only the same two non-PR spec files |
| 7 | `sh test/git-ssh-auth.integration.sh` (design 4.2 cases 1 to 14; 7 runs inside 1) | 14 of 14, run twice with identical output; no sshd or ssh-agent left running afterwards |
| 8 | `sh test/git-auth.integration.sh` (OOL-1 suite, test 10 updated per design 3.7) | passed |
| 9 | `sh test/git-publish.integration.sh` (design 4.3: unchanged, must pass) | passed |
| 10 | `sh helm-chart/oolong/scripts/chart-render.integration.sh` with a private `TMPDIR` | 33 of 33 (20 from OOL-1 plus design 4.1 cases), `all chart render integration tests passed` |
| 11 | `helm lint` for type `none`, `token` (existingSecret x) and `ssh` (existingSecret x, knownHostsKey k) | 3 of 3 passed |
| 12 | Plain-ASCII sweep over the 13 changed files | no non-ASCII byte in any added line; `docs/tea-upgrade/security-assessment.md` keeps pre-existing non-ASCII lines from the base |
| 13 | No edits in `src/generated-nest/` or `tea-spec/` | passed |
| 14 | Path guard, timestamps, error bodies | not applicable: no server code changes |
| 15 | Mutation: `StrictHostKeyChecking=accept-new` in a throwaway copy of the library | case 6 fails; with case 6 neutralised, case 13 fails with `a sync against an unknown host succeeded` (confirms the coder's non-vacuity report) |
| 16 | Mutation: `IdentitiesOnly=yes` and `IdentityAgent=none` removed (library and expected string) | case 4 fails: `a sync with the wrong key succeeded` (the agent's good key was used) |
| 17 | Mutation: identity copy `chmod 644` | caught at case 1: `ssh-keygen -y` refuses the copy (`is not an unencrypted SSH private key`) |
| 18 | Render refusal matrix (below) | 27 cases, each with and without `--skip-schema-validation`: 54 of 54 as the design says |
| 19 | Live check in the cluster (below) | 59 of 59 passed |
| 20 | `gofmt -l`, `go build`, `go vet`, `go test` in tea-client-go | skipped: not changed, base has no Go code |

Tests exist for every case in design 4.1 (cases 1 to 15) and 4.2 (cases 1 to 14) and in the threat
model's section 3 mapping; spot-read cases 1, 2, 4, 7, 13 and 14 assert what the design names (new
sshd log lines only, `Connection from` first, fingerprints, `cmp` on known_hosts, pointer
unchanged), and items 15 to 17 show the key ones can fail.

Counts in the index: 271 passed (items 1 to 3, 6, 12, 13: 6; jest and e2e 58 + 17; shell cases
14 + 12 + 10 + 33; lint 3; CI 2; mutations 3; matrix 54; live 59), 0 failed, 5 skipped (four Go
steps and the e2e suite's one skipped test).

## Render refusal matrix (Helm v4.2.3, chart from `ac4d97b`)

Each row ran `helm template` twice, with the schema and with `--skip-schema-validation`. "leak" is
whether the output contained the planted `not-a-real-key-0001` / `not-a-real-token-0001`: 0 in
every run.

| Values | Schema | Template guard |
|---|---|---|
| ssh + `knownHostsKey`; ssh + inline `knownHosts` (`--set-file`); ssh scp URL + matching `allowedHosts` | renders | renders |
| ssh, no `existingSecret` | refused | `gitPod.auth.existingSecret is required when gitPod.auth.type is ssh` |
| ssh, neither known_hosts source; both sources | refused | `set exactly one of gitPod.auth.ssh.knownHostsKey and gitPod.auth.ssh.knownHosts ...` |
| ssh, `privateKeyKey` empty; null; `auth.ssh=null` | refused | `gitPod.auth.ssh.privateKeyKey is required ...` |
| ssh, private key pasted in `knownHosts` | refused, not echoed | `gitPod.auth.ssh.knownHosts must hold known_hosts lines, not a private key`, not echoed |
| type `SSH` | refused | `gitPod.auth.type must be none, token or ssh` |
| `ssh://git:<pw>@host/...` and `git:<pw>@host:...` with type ssh | refused, not echoed | URL-form message, not echoed |
| `https://` URL with type ssh; `ssh://-oProxyCommand=true/r`; `ext::sh -c true` | URL-form message | URL-form message |
| port 70000 | `gitPod.repoUrl port must be 1-65535` | same |
| ssh URL with type token; ssh URL without `@` with type none; type none with `existingSecret` | refused | the OOL-1 `@` message; `must be an https:// URL (ssh:// needs gitPod.auth.type ssh)`; `existingSecret must be empty` |
| `allowedHosts` without the URL host (D9) | `gitPod.repoUrl host is not in gitPod.allowedHosts` | same |
| `runAsUser` null; `runAsUser` 0 | `gitPod.securityContext.runAsUser must be set to a non-root integer uid ...` | same |
| `gitPod.env` `GIT_SSH_VARIANT`, `GIT_SSH_COMMAND`, `OOLONG_GIT_SSH_DIR` | `... is set by the chart and cannot be overridden` | same |
| `knownHostsKey` with a space; unknown field `auth.ssh.passphrase` | refused | renders: schema-only rules, as design 3.3 and notes departure 3 say |

## Live checks

All in the local k3s cluster; nothing ran against a shared deployment. Every key is a throwaway
ed25519 key generated for this run and is not reproduced here.

**Setup.** The Gitea from OOL-1 (namespace `oolong-tester-gitserver`, private repository
`oolong-admin/tea-fixtures`) already runs Gitea's built-in SSH server on port 2222 behind its
in-cluster Service; it was used as is, not exposed outside the cluster. A throwaway deploy key was
registered on the repository **read-only** through the Gitea API. known_hosts was built from the
server's own host public key file (read inside the Gitea pod, not by `ssh-keyscan`), as
`[<service DNS name>]:2222 ssh-rsa ...`. Namespace `oolong-tester-ool-2`: Secret `oolong-git-ssh`
of type `kubernetes.io/ssh-auth` with `ssh-privatekey` and `known_hosts`; release `ool2` installed
from `helm-chart/oolong` at `ac4d97b` with `gitPod.auth.type=ssh`,
`auth.existingSecret=oolong-git-ssh`, `auth.ssh.knownHostsKey=known_hosts`,
`repoUrl=ssh://git@<service DNS name>:2222/oolong-admin/tea-fixtures.git`, `allowedHosts` that
name, `syncInterval=15`.

| # | Check | Result |
|---|---|---|
| L1 | Install renders and both pods run | passed |
| L2 | Git pod clones over ssh; log shows `Auth: ssh` and the plain URL | passed; repeated syncs succeed |
| L3 | Server `/.well-known/tea`, `/healthz`, `/v1.0.0/products` | 200 each; products lists the five fixture products (3 checks) |
| L4 | New commit `f4e79e8` pushed | published within one sync interval |
| L5 | `id` and `/etc/passwd` in the Git pod | `uid=10001(oolong-git)`; passwd is exactly the two lines of design 3.3 (`10001:0`, `/sbin/nologin`) |
| L6 | Mounts | `/etc/oolong-git-auth` read-only tmpfs, files `0440` root:10001; `/tmp/oolong-git-ssh` tmpfs `size=1024k`; `/etc/passwd` read-only (3 checks) |
| L7 | Key copies | `identity` and `known_hosts` in `/tmp/oolong-git-ssh` mode `600`, owned by uid 10001, no `.new` left |
| L8 | `git remote get-url origin` and `.git/config` | plain ssh URL, nothing else (2 checks) |
| L9 | Pod spec | no `GIT_SSH_COMMAND` or other ssh variable in the pod env; process env holds `GIT_SSH_COMMAND` exactly as design 3.4 step 6 (paths only) and `GIT_SSH_VARIANT=ssh` |
| L10 | The ssh command line seen during a sync | the 3.4 options, `-p 2222 git@<host> git-upload-pack '<path>'`; no key material |
| L11 | Private key body (all lines, every key used in this run) absent from: Git pod log, server log, pods/Deployments/ReplicaSets/ConfigMaps/Services/PVC, events, `helm get all`, every decoded Helm release record (revisions 1 to 4), served `/v1.0.0/products`, files under `/content` (including `_work/.git`), files under `$HOME`, every process environment | 0 hits each (11 checks); positive control: the same pattern matches the copy in `/tmp/oolong-git-ssh` (5 of 5 lines) |
| L12 | Key absent from process command lines, 21727 samples over 35 s of syncs | 0 hits |
| L13 | Host side (isolated `HOME`, port-forward): deploy key with correct known_hosts | `ls-remote` succeeds |
| L14 | Host side: unregistered key | `Permission denied (publickey)` |
| L15 | Host side: known_hosts without the host, `StrictHostKeyChecking=yes` | `No RSA host key is known ... Host key verification failed`; known_hosts byte-identical afterwards (no TOFU) |
| L16 | Host side: push with the deploy key | refused: `Deploy Key ... is not authorized to write` (read-only key) |
| L17 | Secret known_hosts replaced by a wrong host key for the same name (no restart) | after kubelet propagation (about 75 s) syncs fail with `REMOTE HOST IDENTIFICATION HAS CHANGED`, `... has changed and you have requested strict checking`, `Host key verification failed` |
| L18 | While L17: was the key offered? | Gitea logs no public-key attempt and no deploy-key use in the window, only failed pre-auth connections (2 checks) |
| L19 | While L17: commit `d6b516a` pushed | not published; the copied known_hosts equals the wrong source (ssh added nothing); server keeps serving (200); 0 restarts (3 checks) |
| L20 | Correct known_hosts restored | first sync after propagation publishes `d6b516a` |
| L21 | Secret known_hosts empty (running pod) | each sync fails `ERROR: /etc/oolong-git-auth/known_hosts is missing or empty`; no ssh run |
| L22 | Same, fresh pod | exits at start-up with the same message (CrashLoop), nothing printed from the files |
| L23 | Secret known_hosts holding only another host, fresh pod | `ERROR: ... known_hosts has no entry for [<host>]:2222` |
| L24 | Secret without the `known_hosts` key, fresh pod | pod stays `ContainerCreating`: `FailedMount ... references non-existent secret key: known_hosts` (projected source `optional: false`) |
| L25 | Unregistered key in the Secret, fresh pod | `Permission denied (publickey)`; Gitea logs failed public-key attempts |
| L26 | Rotation: second read-only deploy key registered, Secret switched to it (same pod) | copy refreshed in about 60 s, next sync succeeds with the new key (Gitea logs that key's use) |
| L27 | First deploy key deleted in Gitea | host-side: old key `Permission denied`, new key works (2 checks) |
| L28 | Commit `3907c48` pushed after the revocation | published; 0 restarts across L26 to L28 |
| L29 | Secret key replaced by garbage containing a marker (same pod) | syncs fail `ERROR: /etc/oolong-git-auth/identity is not an unencrypted SSH private key`; marker not in the log (2 checks) |
| L30 | Good key restored | sync recovers; 0 restarts |
| L31 | Inline known_hosts: `helm upgrade` with `knownHostsKey=""` and `--set-file knownHosts` | ConfigMap `known_hosts` equals the file; projected volume `defaultMode` 0440 with the Secret `identity` item and the ConfigMap `known_hosts` source; `checksum/git-script` changed and the pod rolled; syncs succeed (3 checks) |
| L32 | Inline known_hosts changed to a wrong host key | pod rolls, new pod refuses with `Host key verification failed` |
| L33 | Back to Secret known_hosts | rolls, syncs succeed |
| L34 | Token regression: OOL-1 release `ool1` (`oolong-tester-ool-1`) upgraded to the chart at `ac4d97b` with `--reuse-values` | manifests differ only in the script ConfigMap and the Git Deployment's `checksum/git-script`; restarted pod logs `Auth: token`, syncs and publishes `3907c48` over HTTPS (2 checks) |
| L35 | Token pod | `protocol.ssh.allow` is `never`; no `/tmp/oolong-git-ssh`; no chart passwd entry (2 checks) |

59 checks in all (rows with several checks are counted as stated).

## Notes (not filed)

- OpenSSH 10.3 in the pinned image prints a three-line "connection is not using a post-quantum
  key exchange algorithm" warning on every fetch against this Gitea. It is noise in the Git pod
  log, not a fault of the change; a server with a post-quantum key exchange would not trigger it.
- Rotation of a Secret-held key or known_hosts took about 60 to 75 s (kubelet propagation), as the
  design says; inline known_hosts changes roll the pod through the checksum.
- After a failed refresh (empty or bad source) the previous valid copies stay in the in-memory
  directory but are not used: `locked_sync_and_publish` stops before ssh runs.
- Models: a sonnet sub-agent ran items 1 to 13 and 15 to 17, and its output was checked here; the
  diff review, the render matrix, the live check and the verdict are this hop's own (opus).
- Cluster resources are left in place: `oolong-tester-gitserver` (now with the second deploy key
  registered, the first deleted), release `ool1` in `oolong-tester-ool-1` (now on the `ac4d97b`
  chart), release `ool2` in `oolong-tester-ool-2`.
