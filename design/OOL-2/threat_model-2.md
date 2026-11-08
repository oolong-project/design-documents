# OOL-2 threat model, round 2: Git pod SSH deploy key authentication

## Changes in this round

Follows architecture round 2 (`design/OOL-2/architecture-2.md`), answering review items F-1 and
F-2 of `review-items/OOL-2/design-reviewer/round-1.md`; no threat or mitigation changes, only
which tests prove them. Section 3: S2 adds architecture 4.2 case 13 (ssh itself refuses an
unknown host, known_hosts unchanged, which fails under `accept-new`; F-1) and says case 3 tests
the start-up pre-check only; S5 names what 4.2 case 4 now proves (exactly one key offered, from
the server log, and `ssh -G` resolves one `identityfile`), replacing the default-identity claim
that could not fail (F-2). F-3 (state variable names, architecture 3.4) is a functional fix with
no threat change; its test is architecture 4.2 case 14.

Task: OOL-2. Design: `design/OOL-2/architecture-2.md` (round 2). The change adds a second kind of
credential to the Git pod and opens the ssh transport for it; the public read API, the server and
the Go client do not change. OOL-1's threats (`design/OOL-1/threat_model-2.md`) for type `token`
are unchanged and its tests stay.

## 1. What is exposed

Assets:

- **A1, the SSH private key**: read access to the private content repository (more if the
  operator used a key with write access or an account key instead of a deploy key).
- **A2, served content**: revisions on the content PVC, served publicly. It must never contain A1.
- **A3, sync availability**: the Git pod holds the publish `flock` while it syncs.
- **A4, content integrity**: content comes from the host the operator pinned; an impostor host
  must not be able to feed the catalog.

Trust boundaries crossed:

- **B1, Kubernetes Secret (and values) to Git pod**: key and known_hosts mounted read-only at
  `/etc/oolong-git-auth`, copied to the in-memory `/tmp/oolong-git-ssh`.
- **B2, Git pod to SSH host**: a new transport (TCP to the URL's host and port); the host is
  outside the operator's control and is trusted only through known_hosts.
- **B3, values author to chart render**: `gitPod.repoUrl`, `gitPod.auth.ssh`, `gitPod.env`,
  `securityContext` shape the pod and, through the URL, what ssh is asked to do.
- **B4, Git pod to content PVC to public API**: unchanged; publish copies the work tree without
  `.git`.

Not in scope: hostile repository content (SEC-02, unchanged), Git-pod egress control (existing
NetworkPolicy guidance, now also naming the ssh port), a hostile operator.

## 2. Threats and mitigations

| Id | Threat | Mitigation in the design |
| --- | --- | --- |
| S1 | A1 lands in a log, the pod spec, the Helm release record, an env var, a command line, `.git/config`, node disk, the PVC or a published revision | Values carry only Secret key names (arch 3.1); a private key pasted into `knownHosts` is refused by a schema not-pattern and the template guard without being printed (arch 3.2, 3.3); the key reaches the pod only as a projected Secret file and an in-memory 0600 copy outside `CONTENT_DIR` (arch 3.3, D3); `GIT_SSH_COMMAND` names a path, never content (3.4); `ssh-keygen` output is discarded and every error names a file, not its content (3.4) |
| S2 | Impostor host (DNS spoofing, hijacked route, wrong host in the URL) receives the connection and serves content (A4) or learns which key is used | `StrictHostKeyChecking=yes` against the operator's known_hosts only (`UserKnownHostsFile` = the copy, `GlobalKnownHostsFile=/dev/null`, `-F /dev/null` drops system and user config), never `no` or `accept-new`, `UpdateHostKeys=no`; known_hosts required at render (arch D6, D7). SSH verifies the host key during key exchange, before user authentication, so the key is never offered to an unverified host (arch 4.2 case 2 asserts it from the server log) |
| S3 | Option or command injection through the URL: `ssh://-oProxyCommand=...`, a user or host starting with `-`, `ext::`/`fd::` transports, shell metacharacters | One narrow grammar implemented in the template, the schema guard and the script (arch 3.5, D8): no leading `-`, no `::`, no whitespace or quote, conservative character sets; the scp form only without `://`; `protocol.ext.allow never` stays; `GIT_SSH_COMMAND` holds only fixed, charset-checked paths (arch 3.4) |
| S4 | A password or token written into an ssh URL is echoed by Helm's schema error or stored in the release record | Schema uses only not-patterns on `repoUrl` (a `user:secret@` shape is refused without echo); the template guard's messages are fixed text (arch 3.2, 3.3) |
| S5 | `gitPod.env` overrides or weakens the ssh wiring (`GIT_SSH`, `GIT_SSH_COMMAND`, `GIT_SSH_VARIANT`, `core.sshCommand` through `GIT_CONFIG_*`), or an agent socket supplies another identity | Reserved names fail the render (arch 3.3, OOL-1 D5); `IdentityAgent=none`, `IdentitiesOnly=yes`, `-F /dev/null` (arch 3.4) |
| S6 | A prompt (passphrase, password, keyboard-interactive, unknown host) or a stalled server blocks while the publish lock is held (A3) | `BatchMode=yes`, password and keyboard-interactive off, `ConnectTimeout`, `ServerAliveInterval`/`CountMax`; an encrypted key is refused at start-up without a prompt (`ssh-keygen -y -P ''`) (arch 3.4) |
| S7 | Opening the ssh transport widens what the pod may fetch for other types | `protocol.ssh.allow always` only for type `ssh`; `none` and `token` keep `never` and keep refusing ssh URLs (arch D10, 3.5) |
| S8 | Allowlist bypass: the host is parsed differently by the check and by ssh/git | The host for the allowlist is the same parsed field that ssh connects to, from one grammar; checked at render and at start (arch 3.5, D9); egress control remains the backstop (docs) |
| S9 | The chart-rendered `/etc/passwd` adds an account or a login shell | Two fixed lines (root, the Git uid) with `/sbin/nologin`, from integer-validated values, read-only mount, type `ssh` only; no change of uid, capability or privilege (arch D4) |
| S10 | A revoked or rotated key keeps being used | Key and known_hosts re-copied and re-validated before every sync from the refreshed Secret mount (arch 3.4); a bad rotation fails the sync and the previous revision keeps serving |
| S11 | Over-privileged key (account key, write access) widens the impact of a leak | Docs: read-only deploy key for the one repository, no passphrase, provider notes (arch 3.7). Operator guidance, no test |

## 3. Mitigations that need a test

All are in the architecture's test list (section 4):

- S1: 4.2 case 7 (key not in PVC, work tree, `.git/config`, env or output); 4.2 case 8 (start-up
  errors do not print file content); 4.2 case 5 (copy is 0600 in the ssh dir); 4.1 case 1 (no
  `secretKeyRef`, no key env, emptyDir `medium: Memory`); 4.1 case 7 (private key in
  `knownHosts` refused without echo).
- S2: 4.2 case 13 (unknown host refused by ssh itself with the pre-check bypassed: connection
  reached sshd, no key offered, known_hosts byte-identical, nothing published; this is the case
  that fails under `accept-new`); 4.2 case 2 (mismatched host key: connection reached sshd, no
  key offered, known_hosts unchanged, nothing published); 4.2 case 3 (no entry: the start-up
  pre-check refuses before connecting; an aid, not the control); 4.2 case 6 (exact options, no
  `accept-new`/`no`); 4.1 cases 5 and 6 (known_hosts required).
- S3: 4.1 case 10 and 4.2 case 9 (shared refused vectors, including `-oProxyCommand`, `ext::`,
  command substitution not executed); 4.2 case 12 (path checks before export).
- S4: 4.1 case 8.
- S5: 4.1 case 13; 4.2 case 4 (an agent holding the authorized key is not used and exactly one
  key, the configured one, is offered, proven from the server log; `ssh -G` with the command
  resolves exactly one `identityfile` and `identityagent none`). No test places a key under
  `$HOME/.ssh`: ssh reads default identities from the passwd home, not `$HOME`, so such a file
  proves nothing (architecture 3.0); in the pod the two are the same directory.
- S6: 4.2 case 8 (encrypted key refused, bounded by the timeout); 4.2 case 6 (`BatchMode=yes` in
  the command).
- S7: 4.1 cases 1 and 3 (conditional protocol block; token and none renders unchanged); 4.1 case
  11 and 4.2 case 9 (ssh URLs refused for none and token).
- S8: 4.1 case 12; 4.2 case 9.
- S9: 4.1 cases 1 and 4 (exact two lines, uid from values, read-only mount, only for ssh) and 4.1
  case 3 (absent for other types).
- S10: 4.2 case 11.
- S11: none (documentation).
