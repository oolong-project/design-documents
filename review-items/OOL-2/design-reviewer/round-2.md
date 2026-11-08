# OOL-2 design review, round 2

Task: OOL-2, Git pod: clone private repositories over SSH with a deploy key and a required
known_hosts. Reviewed: `design/OOL-2/architecture-2.md` and `design/OOL-2/threat_model-2.md`
(round 2), starting from their "Changes in this round" sections, against oolong-project/oolong
`2026-11-oolong` @ `0fac816` (`git-publish.inc.sh` lines 71-112 and 186-268) and the coding
principles at `145008cdcc23`. Everything outside the changed sections was passed in round 1 and
is unchanged (checked by diff against round 1); it is not reopened here.

Verdict: PASSED. All three round-1 items are settled; nothing is OPEN and no new item.

## F-1: No test made ssh itself refuse an unknown host (RESOLVED, security, ARCHITECTURE section 4.2 case 13)

New 4.2 case 13 calls `sync_worktree` directly in a subshell (so neither the `ssh-keygen -F`
pre-check nor `install_ssh_files`, which `locked_sync_and_publish` runs first, can mask or
undo the setup), after overwriting the copied `$SSH_DIR/known_hosts` with a file that has no
entry for the host. It asserts a non-zero return, `Host key verification failed`, a connection
reaching sshd with no `publickey` line and no `SHA256:` fingerprint, and a byte-identical
known_hosts by `cmp`. 3.0 records the prototype: under `accept-new` ssh appends the host and
authenticates, so the `cmp` and the failed return are what separate `yes` from `accept-new`, and
the coder runs the `accept-new` mutation once and reports it. Against the current code this
holds: `sync_worktree` (lines 222-268) runs `git clone`/`git fetch` with the exported
`GIT_SSH_COMMAND` and lets their stderr through, so the message assertion can be met. Case 3 is
relabelled the start-up aid, and threat model S2 maps case 13 as the control.

## F-2: 4.2 case 4 relied on a default identity in `$HOME/.ssh` (RESOLVED, security, ARCHITECTURE section 4.2 case 4)

Case 4 no longer plants a key under `$HOME/.ssh` and says why (3.0: ssh resolves `~` from the
passwd home, prototyped). It now asserts, from the `LogLevel VERBOSE` sshd log, at least one
`Failed publickey` line with the `wrong` fingerprint and that every new line carrying a `SHA256:`
fingerprint names only the `wrong` key, which fails if the agent's `good` key or any default
identity of the test user's real home is offered; and `ssh -G` with the exported command resolves
exactly one `identityfile` and `identityagent none`. Both halves can fail for the right reason.
3.0 notes that in the pod `HOME` and the D4 passwd home are both `/tmp/git-home`. Threat model S5
is updated to match.

## F-3: ssh state variables outside the `OOLONG_GIT_` prefix (RESOLVED, functional, ARCHITECTURE section 3.4)

3.4 renames them `OOLONG_GIT_SSH_AUTH_DIR`, `OOLONG_GIT_SSH_DIR` and `OOLONG_GIT_SSH_KNOWN_HOST`,
so the existing reserved prefix already refuses them in `gitPod.env` (new 4.1 case 13 vector).
`configure_git_auth` unsets them for every type right after line 81, before the path checks that
can return early, and again when `install_ssh_files` fails; `install_ssh_files` keys on
`OOLONG_GIT_AUTH_TYPE` and fails closed for `ssh` when the state is unset. New 4.2 case 14 covers
the none/token unset, the fail-closed path (no sshd connection) and the failed-configure unset.
