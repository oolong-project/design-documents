# OOL-2 implementation notes, round 1: Git pod clones private repositories over SSH with a deploy key

Task: OOL-2 (depends on OOL-1, completed). Built from `design/OOL-2/architecture-2.md` and
`design/OOL-2/threat_model-2.md` (newest rounds, read after assignment). Coding principles read at
`145008cdcc23`.

## Pull request

| PR | Repository | Base | Head commit | Base tip merged |
|---|---|---|---|---|
| https://github.com/oolong-project/oolong/pull/5 | oolong-project/oolong | `2026-11-oolong` | `ac4d97bf1ab9be9da7fff6e8a2cecaab3790f7cc` | `0fac8166b6ff` (the merge of OOL-1, PR 4; the branch was cut from it and the base had not moved) |

tea-client-go is not touched. Image build on the head: passed. Merge order: not applicable (one PR).

## What changed

- **Values and schema** (design 3.1, 3.2): `auth.type` gains `ssh`; new `auth.ssh` block
  (`privateKeyKey`, `knownHostsKey`, `knownHosts`, exactly one known_hosts source). `repoUrl` drops
  its blanket `@` refusal for a conditional one (`@` refused for `none` and `token`, a
  password-shaped `user:secret@` refused for `ssh`), written with not-patterns only so Helm never
  echoes a value. A private key pasted into `knownHosts` is refused the same way.
- **Templates** (3.3): `oolong.gitAuthType` accepts `ssh` and checks its block;
  `oolong.requireGitRepoUrl` is type-aware, validates both ssh forms with `regexMatch`, checks the
  port range, and checks `gitPod.allowedHosts` at render for every type (D9);
  `oolong.gitSshUid` / `oolong.gitSshGid` feed the rendered passwd; `GIT_SSH_VARIANT` is reserved.
  `git-deployment.yaml` adds the projected `git-auth` volume, the in-memory `git-ssh` volume and
  the `git-passwd` mount for `ssh` only; `git-configmap.yaml` adds the `<fullname>-git-ssh`
  ConfigMap (passwd, inline known_hosts) and the conditional `protocol.ssh.allow` block;
  `NOTES.txt` has an ssh branch.
- **Scripts** (3.4, 3.5): `parse_ssh_repo_url` and its small validators, an ssh branch in
  `validate_git_repo_url`, `configure_git_auth <helper> <auth-dir> [<ssh-dir>]` with an ssh half
  (`configure_ssh_auth`), `install_ssh_files` (copy at 0600, `ssh-keygen -y -P ''`, `ssh-keygen -F`,
  atomic `mv`), and a call to `install_ssh_files` at the start of `locked_sync_and_publish`.
  `GIT_SSH_COMMAND` is exactly the string of design 3.4 step 6.
- **Docs**: README (SSH deploy key section, repoUrl and allowlist wording, values list), security
  assessment (SEC-08 note and Secrets), `NOTES.txt`, values comments.

## Departures from the design

1. **A missing final newline is added to the copies.** `install_ssh_files` appends one when the
   source ends without it. OpenSSH rejects a private key without a final newline, and secret
   managers often strip it; without this the start-up check would refuse a key that is otherwise
   fine. The check, the file mode and the message are unchanged.
2. **4.1 case 3 is asserted on names.** The design says the none and token renders contain no
   `git-ssh` and no `/etc/passwd`. The rendered `git-sync.sh` names `/tmp/oolong-git-ssh` as the
   third argument of `configure_git_auth` for every type (design 3.3: ignored unless the type is
   ssh), and `install_ssh_files` has an error text with `/etc/passwd`. The test therefore asserts
   no `name: git-ssh`, `name: git-passwd`, `mountPath: /etc/passwd`, `mountPath: /tmp/oolong-git-ssh`,
   `projected`, or ConfigMap named `*-git-ssh` in those renders.
3. **4.1 case 6 (template guard)** covers the four cases whose rule is not a schema pattern:
   neither source, both, empty `privateKeyKey`, no `existingSecret`, plus type `SSH`. A
   `knownHostsKey` with a space is schema-only, as design 3.3 says of pattern rules.
4. **Image check without Docker** (4.4): Docker is not usable on this host, so the check ran in
   cluster pods with the pinned `alpine/git:2.54.1@sha256:6a1fdb96...` as uid 10001. Output below.
5. The test files `git-auth.integration.sh` test 10 changed as design 3.7 says (`SSH` and `Ssh` are
   refused, message `must be none, token or ssh`).

## Tests

Added or changed:

- `helm-chart/oolong/scripts/chart-render.integration.sh`: design 4.1 cases 1 to 15 (the OOL-1
  `schema_refuses "type ssh"` line is removed; the reserved-name loop gains `GIT_SSH_VARIANT` and
  `OOLONG_GIT_SSH_DIR`). Refused URLs were also run by hand to confirm each fails with a
  `gitPod.repoUrl` message (port range, URL grammar), not for an unrelated reason.
- `oolong/oolong-server/test/git-ssh-auth.integration.sh` with the jest wrapper
  `src/tea/git-ssh-auth.integration.spec.ts`: design 4.2 cases 1 to 14 (the file prints `test 1` to
  `test 14`; case 7 runs inside case 1's block because it inspects that run's files). It starts an unprivileged
  `sshd` on 127.0.0.1 and needs the OpenSSH server binary, `ssh`, `ssh-keygen`, `ssh-agent`,
  `ssh-add`, `git` and `python3`.
- `oolong/oolong-server/test/git-auth.integration.sh`: test 10 as above.

Run (Node 24, `helm`, OpenSSH client and server):

```bash
cd oolong/oolong-server
npm ci && npm run lint && npm run build && npm test && npm run test:e2e
sh test/git-ssh-auth.integration.sh
sh ../../helm-chart/oolong/scripts/chart-render.integration.sh
for t in none token ssh; do helm lint ../../helm-chart/oolong --set image=oolong-server:local --set gitPod.auth.type=$t ...; done
```

Results on the head: `npm test` 14 suites, 58 tests passed; `npm run test:e2e` 4 suites passed (1
skipped, as before); lint, build and `helm lint` for `none`, `token` and `ssh` clean; the plain
ASCII check over every changed line prints nothing. `npm run lint` rewrites the same two unrelated
spec files as before; that rewrite is not part of the PR and was reverted.

**Non-vacuity of case 13 (design 4.2, reported once, not committed).** With
`StrictHostKeyChecking=accept-new` substituted for `yes` in `GIT_SSH_COMMAND`: case 6 (the exact
command) fails first. With case 6 removed from a throwaway copy of the suite, case 13 fails with
`a sync against an unknown host succeeded`. The library was restored afterwards.

## Image check and live check (cluster, no Docker)

Image, uid 10001, no changes: `ssh -F /dev/null -G example.invalid` prints `No user exists for uid
10001` and `rc=255`. With the two-line passwd of design 3.3 mounted over `/etc/passwd`: `rc=0`,
`ssh-keygen -y -P ''` on a fresh key works (`keygen_rc=0`), `printf` and `read` are shell builtins,
`tail` is `/usr/bin/tail` (the copy step uses `tail -c 1` and ran in the pod), `git version 2.54.0`.

Chart installed with `gitPod.auth.type=ssh` against an sshd pod in the cluster (same pinned image,
service DNS name as the host, a throwaway deploy key, no public exposure):

- Clone, publish and repeated syncs succeed; the pod does not restart. sshd logs
  `Accepted publickey` for the deploy key.
- `/tmp/oolong-git-ssh` is a 1Mi tmpfs; `identity` and `known_hosts` are mode 600, owned by uid
  10001. `/etc/passwd` is the two fixed lines.
- The private key's content is in no file under `/content` or the git home, no process
  environment, the pod spec or the pod log. `origin` is the plain ssh URL.
- A commit pushed to the server was published within one sync interval.
- known_hosts replaced with a wrong host key: syncs fail with `Host key ... has changed and you
  have requested strict checking` and `Host key verification failed`, sshd records no accepted key
  in that window, and restoring known_hosts recovered the sync (about 75 seconds, kubelet
  propagation) with zero restarts.

## For the tester

- Everything in Tests runs locally with an OpenSSH server binary; no cluster is needed. The cases
  that touch the host's real home (default identities) cannot: ssh reads them from the passwd
  entry, so case 4 proves single-key offering from the sshd log and `ssh -G` instead (design 3.0).
- The 4.2 suite starts `sshd` as the test user with `StrictModes no` and `UsePAM no`; a host whose
  user has no valid login shell will not accept the connection.
- Not changed: the OOL-1 follow-up the tester noted (no way to mount a private CA bundle) is
  outside this task. The ssh path has no CA concern: host keys come from known_hosts.
- Nothing is OPEN from the design review.
- models: claude-sonnet-5-5 only (no sub-agents).
