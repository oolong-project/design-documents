# OOL-2 design review, round 1

Task: OOL-2, Git pod: clone private repositories over SSH with a deploy key and a required
known_hosts. Reviewed: `design/OOL-2/architecture-1.md` and `design/OOL-2/threat_model-1.md`
(round 1) against oolong-project/oolong `2026-11-oolong` @ `0fac816` (OOL-1 merged) and the
coding principles at `145008cdcc23`, for consistency with OOL-1's reviewed design.

Verdict: REJECTED, on three small test and naming items. The design itself is sound and I
checked it against the code:

- Function: `ssh` joins the closed `gitPod.auth.type` enum (one type at a time, so `token` and
  `ssh` are mutually exclusive by construction), the schema `if`/`then`/`else` keeps OOL-1 D6
  exactly for `none` and `token`, the template guard and script switch on the type, and every
  file and line the design cites (`_helpers.tpl` 70-131, `git-publish.inc.sh` 25-141 and 186-195,
  `git-configmap.yaml` 47-66, `git-deployment.yaml` 22 and 76-102, `values.yaml` 16-17 and 86-108,
  `chart-render.integration.sh` 236 and 261-266) matches. `fsGroup: 10001` makes the 0440
  projected files readable by uid 10001, as for OOL-1's token volume. The `/etc/passwd` finding
  (D4) and the image check in 4.4 are the right way to settle the image question.
- Security: known_hosts is required at render (schema `oneOf` and template guard); the ssh
  command uses `-F /dev/null`, `StrictHostKeyChecking=yes`, `UserKnownHostsFile` on the copy,
  `GlobalKnownHostsFile=/dev/null`, `UpdateHostKeys=no`, `IdentitiesOnly=yes`,
  `IdentityAgent=none`, `BatchMode=yes`, and never `no` or `accept-new`; the key is copied 0600
  into a `medium: Memory` emptyDir, off the PVC and node disk, and never printed; `GIT_SSH`,
  `GIT_SSH_COMMAND` are already reserved and `GIT_SSH_VARIANT` is added, `core.sshCommand` is
  covered by the `GIT_CONFIG_*` prefixes; the URL grammar refuses leading `-`, `::`, whitespace,
  `ext::`/`file://` and other schemes, with shared vectors for template, schema and script;
  `protocol.ssh.allow` opens only for type `ssh`; password-shaped userinfo and a private key in
  `knownHosts` are refused by not-patterns with no echo (the OOL-1 lesson), tested with and
  without `--skip-schema-validation`. The threat model fits the exposure and maps each
  mitigation to a test.

## F-1: No test makes ssh itself refuse an unknown host (OPEN, security, ARCHITECTURE section 4.2)

Architecture 4.2 cases 2 and 3, 3.4 `install_ssh_files` step 4, threat model S2. The task asks
for a script-level test that fails "on an unknown host key". As designed:

- Case 3 (no entry for the host) is refused by the `ssh-keygen -F` pre-check in
  `install_ssh_files`, before ssh runs. The design says so itself: the pre-check "is not the
  control: `StrictHostKeyChecking=yes` is".
- Case 2 (mismatched key) is refused by ssh, but `StrictHostKeyChecking=accept-new` refuses a
  changed key too and adds nothing to known_hosts, so every assertion of case 2 also passes
  under `accept-new`.

So the one setting that separates `yes` from `accept-new` (trust on first use of an unknown
host) is guarded only by the string compare of case 6. Because `install_ssh_files` runs at the
start of every `locked_sync_and_publish`, a real sync can never reach ssh without an entry, and
no case shows ssh's own policy refusing it.

The design must add a case to 4.2: after a successful `configure_git_auth`, overwrite
`$SSH_DIR/known_hosts` with a file that holds only `other.example.test <key>` and call
`sync_worktree` directly (bypassing the pre-check). Assert that it fails with
`Host key verification failed`; that the new sshd log lines show `Connection from` and no
`publickey` line and no key fingerprint; and that `$SSH_DIR/known_hosts` is byte-identical
afterwards. The last assertion is the one that fails under `accept-new` (ssh would add the host
and connect). Name the case under S2 in threat model section 3.

## F-2: 4.2 case 4's default identity sits where ssh never looks (OPEN, security, ARCHITECTURE section 4.2)

Architecture 4.2 case 4, threat model S5. The case puts the `good` key at
`$HOME/.ssh/id_ed25519` and asserts its fingerprint appears nowhere, to show default identities
are not used. OpenSSH's client resolves `~` and its default identity files from the passwd
entry (`getpwuid`, `pw_dir`), not from `$HOME`; the suite sets an isolated `HOME`, so ssh would
not read that file even without `IdentitiesOnly=yes` and an explicit `IdentityFile`. That half
of the assertion passes for the wrong reason. (The agent half is real: ssh does honour
`SSH_AUTH_SOCK`.)

In the pod the two agree (`HOME=/tmp/git-home` and the D4 passwd home is `/tmp/git-home`), so the
design is fine; only the test claim needs to be concrete. Either drop the `$HOME/.ssh` step and
say why, or replace it with an assertion that holds against whatever default identities the test
user's real home has: in the new sshd log lines, every `publickey` line names the `wrong`
fingerprint and no other fingerprint appears (exactly one key was offered).

## F-3: `OOLONG_SSH_*` state is outside the reserved prefix (OPEN, functional, ARCHITECTURE section 3.4)

Architecture 3.4 step 5 and `install_ssh_files`. The script keeps its state in
`OOLONG_SSH_AUTH_DIR`, `OOLONG_SSH_DIR` and `OOLONG_SSH_KNOWN_HOST`, and `install_ssh_files`
is a no-op only because `OOLONG_SSH_DIR` is empty. These names are not covered by the reserved
`OOLONG_GIT_` prefix (`_helpers.tpl` line 119), so `gitPod.env.OOLONG_SSH_DIR` renders, reaches
the shell as a variable, and for type `none` or `token` turns `install_ssh_files` on in every
sync (each then fails on a missing `/identity`). It is an operator misconfiguration, not an
attack, but it is exactly what OOL-1 D5 made the render refuse.

The design must say one of: name the variables `OOLONG_GIT_SSH_AUTH_DIR`, `OOLONG_GIT_SSH_DIR`,
`OOLONG_GIT_SSH_KNOWN_HOST` (already reserved by the prefix, so no list change); and/or have
`install_ssh_files` return 0 unless `OOLONG_GIT_AUTH_TYPE` is `ssh`, and have
`configure_git_auth` clear the three variables for `none` and `token`. Low priority; it blocks
only because every open item blocks on this board.
