# OOL-2 architecture, round 1: Git pod clones private repositories over SSH with a deploy key and a required known_hosts

Task: OOL-2 (depends on OOL-1, completed). Coding principles: `coding-principles/coding_principles.md`
at commit `145008cdcc23`. Code read at oolong-project/oolong `2026-11-oolong` @ `0fac816` (the
merge of OOL-1, PR #4). OOL-1's design (`design/OOL-1/architecture-2.md`, `threat_model-2.md`)
was read for intent; every fact below about the chart and scripts was checked against the
merged code, with file and line where it matters.

## 1. Problem

The Git pod can clone a public repository or, since OOL-1, a private one over HTTPS with a token
(`gitPod.auth.type: token`). Many operators would rather give it a read-only SSH deploy key. This
task adds `gitPod.auth.type: ssh`: the private key and a known_hosts file come from an existing
Secret (known_hosts may also be given inline in values), host keys are always verified, and the
operator picks exactly one of `none`, `token` or `ssh`.

Out of scope: GitHub App installation tokens; SSH certificates for the client; SSH agent
forwarding; jump hosts or proxies; a chart-created Secret (same reason as OOL-1 D2); any change
to the server, the TEA API or tea-client-go.

## 2. Repositories and branches

- oolong-project/oolong, branch `2026-11-oolong` (PR against it): `helm-chart/oolong` (values,
  schema, templates, scripts), `oolong/oolong-server/test` and `src/tea` (shell suites and jest
  wrappers), `README.md`, `docs/tea-upgrade/security-assessment.md`.
- oolong-project/tea-client-go: not touched.

## 3. Design

### 3.0 Facts this design rests on (checked)

- **The pinned image ships an SSH client.** `gitPod.image` is
  `alpine/git:2.54.1@sha256:6a1fdb96...` (`values.yaml` line 59). Without Docker, the image was
  inspected through the public Docker Hub registry API: the index digest resolves to the
  linux/amd64 manifest `sha256:cf7d0aa63721...`, config `sha256:f44022d5665f...`. Its history is
  `ADD alpine-minirootfs-3.24.4` then `apk add git git-lfs tig gpg less openssh patch perl`. The
  apk database in the layer lists `openssh 10.3_p1-r2` with `openssh-client-default`,
  `openssh-keygen` and `openssh-server`, and the layer contains `usr/bin/ssh`,
  `usr/bin/ssh-keygen` (version string `OpenSSH_10.3p1`) and `etc/ssh/ssh_config` (which
  `Include`s `/etc/ssh/ssh_config.d/*.conf`). It also lists `git 2.54.0-r0` (the binary reports
  2.54.0 although the tag says 2.54.1; nothing here depends on the patch level). Only the amd64
  image was inspected; the coder confirms in the image (4.4).
- **ssh cannot run as uid 10001 in that image without help.** The image's `/etc/passwd` has
  root, the system users and `nobody` (65534), no uid 10001, and the Git pod runs as 10001
  (`values.yaml` lines 86-93). OpenSSH's `ssh` and `ssh-keygen` call `getpwuid(getuid())` at
  start and exit 255 with `No user exists for uid %lu` when it fails; that string is in both
  binaries of the pinned layer. git over HTTPS does not need a passwd entry, which is why OOL-1
  never hit this. Decision D4 supplies the entry.
- ssh checks the mode of a private key only when the key file is **owned by the running user**
  (OpenSSH `sshkey_perm_ok`); prototyped: a user-owned key with mode 0640 is refused with
  `UNPROTECTED PRIVATE KEY FILE!`. Kubernetes Secret volume files are owned by root with group
  `fsGroup`, so the mounted key would probably be accepted, but the result depends on how the
  volume is provided (and in the shell tests the fixture files belong to the test user). D3
  copies the key to a file the pod owns with mode 0600, so ssh's behaviour does not depend on the
  volume.
- Today `git-sync.sh` sets `protocol.ssh.allow never` (`templates/git-configmap.yaml` line 64),
  `oolong.requireGitRepoUrl` refuses `@` and anything but `https://`
  (`templates/_helpers.tpl` lines 74-84), `validate_git_repo_url` accepts only `https://`
  without userinfo (`scripts/git-publish.inc.sh` lines 25-69), the schema refuses `@` in
  `repoUrl` with `not: {pattern: "@"}` (`values.schema.json` line 24) and the type enum is
  `["none", "token"]` (line 31). `GIT_SSH` and `GIT_SSH_COMMAND` are already reserved in
  `gitPod.env` (`_helpers.tpl` line 118); `GIT_SSH_VARIANT` is not.
- Helm v4.2.3 (prototyped on a copy of the chart with the 3.2 schema): the nested
  `if`/`then`/`else` on `gitPod.auth.type` works, and each failure prints only the path and
  `'not' failed` or `'oneOf' failed, none matched`, never the value.
- A local `sshd` runs unprivileged on 127.0.0.1 with its own host key, `UsePAM no` and
  `StrictModes no`, and `git clone` over `ssh://<user>@127.0.0.1:<port>/<abs-path>` with the
  3.4 ssh command works; a wrong key, a mismatched host key and the right key all leave
  distinguishable lines in its `LogLevel VERBOSE` log (prototyped, OpenSSH 9.6 on the test
  host). This is what the 4.2 suite uses.

### 3.1 Values (`values.yaml`)

`gitPod.auth` gets a third type and an `ssh` block; nothing in `token` changes (OOL-1's shape
rule: a type adds its own sub-block, no field changes meaning).

```yaml
  # Credentials for a private repository. type is one of:
  #   none  - anonymous clone (default)
  #   token - HTTPS username + token read from an existing Secret
  #   ssh   - SSH deploy key read from an existing Secret; host keys always verified
  auth:
    type: none
    # Name of an existing Secret in the release namespace. Required when type
    # is token or ssh; must be empty when type is none. The chart never creates it.
    existingSecret: ""
    token:
      usernameKey: username
      tokenKey: token
    ssh:
      # Key in existingSecret holding the unencrypted private key (OpenSSH or PEM).
      # ssh-privatekey matches the kubernetes.io/ssh-auth Secret type.
      privateKeyKey: ssh-privatekey
      # known_hosts for the Git host is required: set exactly one of these two.
      # knownHostsKey: key in existingSecret holding known_hosts lines.
      knownHostsKey: ""
      # knownHosts: known_hosts lines inline (public data; use --set-file or a values file).
      knownHosts: ""
```

The `repoUrl` comment (lines 72-73) becomes: `https:// Git URL without credentials, or for
auth.type ssh an ssh://[user@]host[:port]/path or [user@]host:path URL (required when
gitPod.enabled is true). Never put a password or token in the URL.` The `allowedHosts` comment
(line 75) becomes `Empty allows any host.` and adds `For ssh URLs the host is the part after
user@ and before :port or :path.`

Why both known_hosts sources: host keys are public, so many operators want them reviewed in a
values file (`knownHosts`); others keep everything for one repository in one Secret
(`knownHostsKey`, the Flux convention). Neither has a default, so a render without known_hosts
fails (the task's requirement) and the operator chooses explicitly.

### 3.2 Schema (`values.schema.json`)

1. `gitPod.properties.repoUrl` loses its `"not": {"pattern": "@"}`; keeps `type` and
   `maxLength: 2048`.
2. `gitPod` gets an `allOf` entry (`if` auth.type is `ssh`, `then`/`else`). Only not-patterns
   are used on `repoUrl`, because Helm v4.2.3 echoes a value that fails a positive `pattern`
   (OOL-1 round 2 note); the full shape is checked by the template guard (3.3), whose messages
   are fixed text:

```json
"allOf": [
  {
    "if": { "required": ["auth"],
            "properties": { "auth": { "required": ["type"],
                                      "properties": { "type": { "const": "ssh" } } } } },
    "then": { "properties": { "repoUrl": {
      "not": { "pattern": "^([A-Za-z][A-Za-z0-9+.-]*://)?[^/@]*:[^/@]*@" } } } },
    "else": { "properties": { "repoUrl": { "not": { "pattern": "@" } } } }
  }
]
```

   `else` keeps OOL-1 D6 exactly for `none` and `token`. `then` refuses a password-shaped
   userinfo (`user:secret@`) in any scheme or in scp form, so the one value in an ssh-type
   `repoUrl` that could be a secret is refused without being printed.
3. `auth.properties.type.enum` becomes `["none", "token", "ssh"]`.
4. `auth.properties.ssh`:

```json
"ssh": {
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "privateKeyKey": { "type": "string", "maxLength": 253, "pattern": "^([-._a-zA-Z0-9]+)?$" },
    "knownHostsKey": { "type": "string", "maxLength": 253, "pattern": "^([-._a-zA-Z0-9]+)?$" },
    "knownHosts":    { "type": "string", "maxLength": 65536, "not": { "pattern": "PRIVATE KEY" } }
  }
}
```

   Key names are not secret, so their positive patterns may echo; `knownHosts` uses a
   not-pattern so a private key pasted into the wrong field is refused without being printed.
5. `auth.allOf` gets a third entry:

```json
{
  "if": { "properties": { "type": { "const": "ssh" } } },
  "then": {
    "required": ["existingSecret", "ssh"],
    "properties": {
      "existingSecret": { "minLength": 1 },
      "ssh": {
        "required": ["privateKeyKey"],
        "properties": { "privateKeyKey": { "minLength": 1 } },
        "oneOf": [
          { "required": ["knownHostsKey"],
            "properties": { "knownHostsKey": { "minLength": 1 }, "knownHosts": { "maxLength": 0 } } },
          { "required": ["knownHosts"],
            "properties": { "knownHosts": { "minLength": 1 }, "knownHostsKey": { "maxLength": 0 } } }
        ]
      }
    }
  }
}
```

   The existing `none` entry (existingSecret must be empty) and `token` entry are unchanged.
   (Prototyped as written with Helm v4.2.3; see 3.0.)

### 3.3 Templates

**`_helpers.tpl`**

- `oolong.gitAuthType` (lines 86-111): accepts `none`, `token`, `ssh`; the message becomes
  `gitPod.auth.type must be none, token or ssh`. New `ssh` branch, each a `fail` naming the path
  and never a value: `gitPod.auth.existingSecret is required when gitPod.auth.type is ssh`;
  `gitPod.auth.ssh.privateKeyKey is required when gitPod.auth.type is ssh`; `set exactly one of
  gitPod.auth.ssh.knownHostsKey and gitPod.auth.ssh.knownHosts when gitPod.auth.type is ssh`
  (neither or both); `gitPod.auth.ssh.knownHosts must hold known_hosts lines, not a private
  key` (contains `PRIVATE KEY`). The comment at lines 86-89 names the three types.
- New `oolong.gitSshUid` / `oolong.gitSshGid` (used only for `ssh`): uid is
  `gitPod.securityContext.runAsUser`, else `podSecurityContext.runAsUser`; it must be an integer
  from 1 to 2147483647, else `fail "gitPod.securityContext.runAsUser must be set to a non-root
  integer uid when gitPod.auth.type is ssh"`. gid is `gitPod.securityContext.runAsGroup`, else
  `podSecurityContext.runAsGroup`, else `0` (Kubernetes' primary group when `runAsGroup` is
  unset); same integer check. Values arrive as float64 from YAML and int64 from `--set`; both are
  accepted when integral.
- `oolong.requireGitRepoUrl` (lines 70-84) becomes type-aware. It obtains the type with
  `include "oolong.gitAuthType" .` (so type errors are reported first), then:
  - `none` / `token`: unchanged (`@` refused, `https://` required); the https message drops
    the `ssh://` from its list of disabled schemes and adds `ssh:// needs gitPod.auth.type ssh`.
  - `ssh`: the URL must match one of the two forms of 3.5 (Sprig `regexMatch`), else
    `fail "gitPod.repoUrl must be ssh://[user@]host[:port]/path or [user@]host:path when
    gitPod.auth.type is ssh"`; a port outside 1-65535 fails with `gitPod.repoUrl port must be
    1-65535`.
  - All types (new): when `gitPod.allowedHosts` is non-empty, the host parsed from the URL
    (https: between `https://` and the first `/` or `:`; ssh: 3.5) must be in it, else
    `fail "gitPod.repoUrl host is not in gitPod.allowedHosts"`. Until now a mismatched host
    rendered and then crash-looped (runtime check, `git-publish.inc.sh` lines 51-67, kept);
    failing at render is the same direction as OOL-1 D6. Exact, case-sensitive match as at
    runtime.
  - The comment at lines 70-73 describes both forms.
- `oolong.requireGitPodEnvNotReserved` (line 118): add `GIT_SSH_VARIANT` to `$exact`.
  `GIT_SSH` and `GIT_SSH_COMMAND` are already there; `core.sshCommand` through
  `GIT_CONFIG_KEY_*` is already covered by the prefixes. `SSH_AUTH_SOCK` is not reserved: the ssh
  command disables the agent (`IdentityAgent=none`, 3.4) and 4.2 case 4 proves it.

**`git-deployment.yaml`** (type `ssh` only; `none` and `token` render exactly as today)

- `OOLONG_GIT_AUTH_TYPE` is `"ssh"`. No `GIT_SSH_COMMAND` or other ssh variable in the pod env:
  `configure_git_auth` sets them (D5).
- Volume `git-auth` is a **projected** volume, `defaultMode: 0440`, mounted read-only at
  `/etc/oolong-git-auth` without `subPath` (so Secret updates propagate), with sources:
  - `secret`: `name: existingSecret`, `optional: false`, `items`:
    `{key: <privateKeyKey>, path: identity}` and, when `knownHostsKey` is set,
    `{key: <knownHostsKey>, path: known_hosts}`;
  - when `knownHosts` is set: `configMap`: `name: <fullname>-git-ssh`, `items`:
    `{key: known_hosts, path: known_hosts}`.

  Either way the script finds `/etc/oolong-git-auth/identity` and
  `/etc/oolong-git-auth/known_hosts` (D2).
- Volume `git-ssh`: `emptyDir: {medium: Memory, sizeLimit: 1Mi}`, mounted at
  `/tmp/oolong-git-ssh` (writable). It holds the 0600 copies (D3), in memory, never on node
  disk, never on the content PVC.
- Volume `git-passwd`: `configMap` `<fullname>-git-ssh`, `items: [{key: passwd, path: passwd}]`,
  `defaultMode: 0444`, mounted at `/etc/passwd` with `subPath: passwd`, `readOnly: true` (D4).

**`git-configmap.yaml`**

- `git-sync.sh` (`git-configmap.yaml` lines 47-66): the call becomes
  `configure_git_auth /scripts/git-credential-oolong.sh /etc/oolong-git-auth /tmp/oolong-git-ssh || exit 1`
  (still after `validate_git_repo_url` and before the configuration block is printed). The
  protocol line becomes conditional, replacing `git config --global protocol.ssh.allow never`:

```sh
    if [ "$OOLONG_GIT_AUTH_TYPE" = ssh ]; then
      git config --global protocol.ssh.allow always
    else
      git config --global protocol.ssh.allow never
    fi
```

  `file`, `git`, `ext` and `http` stay `never` for every type. `https` stays allowed for `ssh`
  (nothing reaches it: the URL is pinned to an ssh form and submodules are never fetched).
- A second document in the same file, rendered only for type `ssh`: ConfigMap
  `<fullname>-git-ssh`, labels as the script ConfigMap, data:
  - `passwd`: exactly two lines, `root:x:0:0:root:/root:/sbin/nologin` and
    `oolong-git:x:<uid>:<gid>:oolong git sync:/tmp/git-home:/sbin/nologin` (uid/gid from 3.3);
  - `known_hosts` (only when `knownHosts` is set): `{{ .Values.gitPod.auth.ssh.knownHosts | toJson }}`
    (a JSON string is valid YAML for any content).

  Because it is in `git-configmap.yaml`, the `checksum/git-script` annotation (which hashes this
  whole file, `git-deployment.yaml` line 22) rolls the pod when the inline known_hosts or the uid
  changes.

**`NOTES.txt`** line 23 becomes `Git sync accepts https:// remotes, and ssh remotes when
gitPod.auth.type is ssh.` (rest unchanged). After the `token` block (lines 24-26) add an `ssh`
branch: `Git auth is ssh: Secret <existingSecret> must exist in this namespace with the key
<privateKeyKey>` plus, when `knownHostsKey` is set, `and <knownHostsKey>`; `Host keys are always
verified against the known_hosts you supplied. Use a read-only deploy key for the one
repository. A rotated Secret is picked up on a later sync without a restart.`

### 3.4 `configure_git_auth` and the ssh files (`git-publish.inc.sh`)

Signature: `configure_git_auth <helper-script> <auth-dir> [<ssh-dir>]`. The third argument is
required for `ssh` and ignored otherwise; it gets the same `[A-Za-z0-9/._-]+` check as the two
existing paths (lines 83-95), before anything is exported, because it ends up in
`GIT_SSH_COMMAND`, which git runs through a shell. The type switch (lines 97-107) accepts
`ssh` and its message becomes `ERROR: OOLONG_GIT_AUTH_TYPE must be none, token or ssh`. The
header comment (lines 71-74) describes all three. `token` is unchanged.

`ssh)` branch, in order; each failure prints one `ERROR:` line to stderr and returns 1; no line
ever contains file content:

1. `ssh-dir` missing or failing the path check: `ERROR: ssh directory path must match [A-Za-z0-9/._-]+`.
2. `command -v ssh` and `command -v ssh-keygen`, else `ERROR: ssh and ssh-keygen are required for auth type ssh`.
3. `ssh -F /dev/null -G oolong.invalid </dev/null >/dev/null 2>&1`, else
   `ERROR: ssh cannot run as uid <id -u>: it needs an /etc/passwd entry`. (`-G` only prints the
   resolved configuration; it makes no connection and no DNS lookup.) This turns the getpwuid
   failure of 3.0 into a clear start-up message if the passwd mount is missing.
4. `parse_ssh_repo_url` (3.5) on `GIT_REPO_URL` (already validated by `validate_git_repo_url`,
   so this cannot fail in the pod; it is still checked). It sets `_ssh_known_host`: `host` when
   there is no port or the port is 22, else `[host]:port` (the name ssh looks up).
5. Store the three values in shell variables (not exported): `OOLONG_SSH_AUTH_DIR`,
   `OOLONG_SSH_DIR`, `OOLONG_SSH_KNOWN_HOST`; then `install_ssh_files || return 1`.
6. Export, and only then return 0:
   - `GIT_SSH_VARIANT=ssh`
   - `GIT_SSH_COMMAND="ssh -F /dev/null -o BatchMode=yes -o IdentitiesOnly=yes -o IdentityAgent=none -o IdentityFile=<ssh-dir>/identity -o StrictHostKeyChecking=yes -o UserKnownHostsFile=<ssh-dir>/known_hosts -o GlobalKnownHostsFile=/dev/null -o UpdateHostKeys=no -o PasswordAuthentication=no -o KbdInteractiveAuthentication=no -o ConnectTimeout=30 -o ServerAliveInterval=15 -o ServerAliveCountMax=4"`
   - `GIT_TERMINAL_PROMPT=0` (already exported for every type, line 81).

   No `GIT_CONFIG_*` entries are exported for `ssh`.

`install_ssh_files` (new, no arguments, uses the three variables; returns 0 at once when
`OOLONG_SSH_DIR` is empty, so it is a no-op for `none` and `token`):

1. `$OOLONG_SSH_AUTH_DIR/identity` and `$OOLONG_SSH_AUTH_DIR/known_hosts` must each be a
   readable non-empty file, else `ERROR: <path> is missing or empty`.
2. In a subshell with `umask 077`: `cat <auth>/identity > <ssh>/identity.new`,
   `chmod 600 <ssh>/identity.new`; same for `known_hosts.new`. `cat` and redirection, not
   `cp -p`, so the copy is owned by the pod's uid with mode 0600 whatever the source's owner and
   mode.
3. `ssh-keygen -y -P '' -f <ssh>/identity.new </dev/null >/dev/null 2>&1`, else remove both
   `.new` files and print `ERROR: <auth>/identity is not an unencrypted SSH private key`. This
   refuses an encrypted key (no prompt: `-P ''`), a public key, and garbage at start-up instead of
   at the first fetch. Its stdout (the public key) and stderr are discarded.
4. `ssh-keygen -F "$OOLONG_SSH_KNOWN_HOST" -f <ssh>/known_hosts.new >/dev/null 2>&1`, else remove
   both `.new` files and print `ERROR: <auth>/known_hosts has no entry for <known-host>` (the
   host name is from the validated URL, not a secret). Works for plain, hashed and
   `@cert-authority` lines (prototyped). It also matches a `@revoked` line; that is fine, the
   check is a start-up aid and ssh itself still refuses the host. It is not the control:
   `StrictHostKeyChecking=yes` is.
5. `mv -f` each `.new` over its final name (`identity`, `known_hosts`): a reader never sees a
   half-written file.

Per-sync refresh: `locked_sync_and_publish` (lines 186-195) calls `install_ssh_files` first,
before `acquire_publish_lock`; on failure it returns 1 (the loop logs `Sync failed, will retry`,
the previous revision keeps serving). So a rotated key or known_hosts in the Secret is used at
the next sync without a restart, as for OOL-1's token (kubelet refreshes the projected volume,
typically within about a minute; inline known_hosts changes roll the pod through the checksum).

Why each ssh option (D6): `-F /dev/null` skips `~/.ssh/config` and the system `ssh_config` with
its `Include` directory, so nothing in the image or HOME adds a `ProxyCommand`, another
identity or a weaker host-key policy; `BatchMode=yes` makes any prompt (passphrase, password,
unknown host) a failure instead of a wait while the publish lock is held; `IdentitiesOnly=yes`
and `IdentityAgent=none` offer only the configured key; `StrictHostKeyChecking=yes` with
`UserKnownHostsFile` pointing at the copy and `GlobalKnownHostsFile=/dev/null` verifies against
the operator's known_hosts only, never adding a key (never `no`, never `accept-new`);
`UpdateHostKeys=no` keeps ssh from rewriting it; password and keyboard-interactive are off so a
server cannot fall back to them; the timeouts bound a stalled server.

### 3.5 URL grammar and `validate_git_repo_url` (`git-publish.inc.sh`)

`validate_git_repo_url` (lines 25-69) switches on `OOLONG_GIT_AUTH_TYPE`: `ssh` uses
`parse_ssh_repo_url`; anything else (including unset, as in today's tests) keeps the current
https code unchanged. The allowlist loop (lines 51-67) runs on the parsed host for both. New
messages: `ERROR: GIT_REPO_URL must be ssh://[user@]host[:port]/path or [user@]host:path for auth
type ssh`; for the https branch the existing `must use https://` message stays.

`parse_ssh_repo_url` (POSIX `case` patterns and parameter expansion only; nothing evaluated)
accepts exactly:

- **URL form** `ssh://[user@]host[:port]/path`: the text after `ssh://` up to the first `/` is
  the authority, the rest (with its leading `/`) is the path.
- **scp form** `[user@]host:path`: only when the URL contains no `://`; the text before the
  first `:` is `[user@]host`.

Pieces (same in the template's `regexMatch`; shared vectors in 4.1 case 12 and 4.2 case 9):

- `user` (optional): `[A-Za-z0-9_][A-Za-z0-9._-]*`. At most one `@`. No `:` (so no password).
- `host`: `[A-Za-z0-9][A-Za-z0-9.-]*` (the `gitPod.allowedHosts` item pattern; no IPv6 literal,
  no leading `-`, which would make ssh read it as an option).
- `port` (URL form only): 1 to 5 digits, value 1-65535.
- `path`: URL form `/` then `[A-Za-z0-9._~+/-]+`; scp form `[A-Za-z0-9._~+][A-Za-z0-9._~+/-]*`
  or `/` followed by one of `[A-Za-z0-9._~+-]` and more of the same set with `/` (that is, not
  empty, not starting with `-`, not starting with `//`). No `:`, so `ext::...`, `fd::...` and any
  `<transport>::` form are refused, and no whitespace or quote.

Refused for type `ssh`: `https://`, `http://`, `git://`, `file://`, `git+ssh://`, `ssh+git://`,
`ext::`, any `::`, a local path, a password, an empty user, two `@`, a port outside 1-65535, a
host or path starting with `-`, a missing path. For `none` and `token` every ssh form is refused
as today. The known_hosts name is `host` or `[host]:port` (3.4 step 4). The scp form has no
port; a non-standard port needs the URL form (documented).

### 3.6 Decisions

- **D1, reuse `gitPod.auth` with a closed type** ("Prefer closed types over magic strings and
  booleans"): `ssh` joins the enum, the schema and the template guard; `configure_git_auth`
  switches on it. No boolean `sshEnabled`, no new top-level block.
- **D2, one mount directory, projected volume, for `ssh`.** The two possible known_hosts sources
  land at the same path next to the key, so the script has one layout; the volume is not
  `subPath`-mounted, so Secret rotation propagates. The `token` volume stays a plain `secret`
  volume (OOL-1's render assertions still hold).
- **D3, copy the key and known_hosts to an in-memory emptyDir with mode 0600 at start and before
  every sync.** ssh's mode check then always passes, whoever owns the mounted files; the copy is
  validated before use; it never touches node disk or the PVC; rotation works. Departure from the
  task's sketch (copy into the git-home emptyDir): a separate `medium: Memory` emptyDir keeps the
  key off node disk, and the refresh keeps OOL-1's rotation behaviour.
- **D4, a chart-rendered `/etc/passwd` for type `ssh`.** Required by the 3.0 finding: without an
  entry for the pod's uid, ssh exits before connecting. Rejected: running the Git pod as a uid
  the image knows (65534) would change file ownership on the content PVC and break publishing
  over existing revisions; building our own image is out of scope. The file is read-only, holds
  only root and the Git uid with `/sbin/nologin`, uses integers from validated values (nothing is
  interpolated into a format that could add a line), and is mounted only for `ssh`. Some
  runtimes (CRI-O) add a passwd entry themselves; the mount makes the result the same everywhere.
- **D5, `GIT_SSH_COMMAND` is set by the chart's script, not as pod env.** `configure_git_auth`
  builds it from the validated paths and exports it, so the 4.2 suite exercises the exact command
  the pod runs; `gitPod.env` cannot set it, `GIT_SSH` or `GIT_SSH_VARIANT` (render fails, OOL-1
  D5); `GIT_SSH_VARIANT=ssh` pins OpenSSH argument handling.
- **D6, the ssh options of 3.4.** Host keys are never trusted on first use
  ("Secrets and credentials": a credential must not be offered to an unverified peer).
- **D7, known_hosts is required, from the Secret or inline, exactly one.** No default, so a
  render without it fails, schema and template guard both.
- **D8, URL grammar narrower than git's.** Only the two forms above, with conservative
  character sets, so the template, the schema and the script agree and nothing reaches ssh that
  it could parse as an option. An exotic but valid path (for example with a space) is refused;
  documented.
- **D9, allowlist checked at render too, for every type.** Earlier, clearer failure; the
  runtime check stays.
- **D10, `protocol.ssh.allow always` only for type `ssh`.** Every other type keeps `never`.

"The container and the chart stay locked down": this change adds one writable path (the
in-memory `/tmp/oolong-git-ssh`, 1Mi) and one read-only file mount (`/etc/passwd`), both only for
`ssh`; no port and no capability. New outbound destination: the SSH host and port of
`gitPod.repoUrl` (TCP, usually 22) when type is `ssh`; the existing guidance to restrict Git-pod
egress with a NetworkPolicy applies, and the docs say to allow that port. New values have schema
entries and safe defaults (`type: none`); no value is interpolated into shell text
(`git-sync.sh` still reads only env vars, and the ssh command contains only fixed paths).

### 3.7 Docs (every line that describes the old behaviour)

- `helm-chart/oolong/values.yaml` lines 72-73, 75, 96-99 (3.1) and the new `ssh` block comments.
- `helm-chart/oolong/templates/_helpers.tpl` comments at lines 70-73 and 86-89, messages at lines
  79, 82, 94 (3.3).
- `helm-chart/oolong/scripts/git-publish.inc.sh` comment lines 71-74, message line 104 (3.4).
- `helm-chart/oolong/templates/git-configmap.yaml` line 64 (3.3).
- `helm-chart/oolong/templates/NOTES.txt` lines 23-26 (3.3).
- `README.md`:
  - line 157: `from an HTTPS Git repository, public or private` becomes `from a Git repository
    over HTTPS (public, or private with a token) or SSH (private, with a deploy key)`.
  - line 159: `gitPod.repoUrl` is `https://` with no userinfo, or with `gitPod.auth.type=ssh` an
    `ssh://[user@]host[:port]/path` or `[user@]host:path` URL; the Git pod disables `git://`,
    `http://`, `file://` and `ext::` always, and `ssh` unless the type is `ssh`; the allowlist
    applies to both and is checked at render and at start.
  - lines 163-182, "Private repositories": split into "HTTPS token" (the current text) and a new
    "SSH deploy key" subsection: create a read-only deploy key for the one repository (GitHub and
    GitLab deploy keys are read-only by default; Bitbucket access keys are read-only), without a
    passphrase; a Secret from files, for example
    `kubectl -n oolong create secret generic oolong-git-ssh --from-file=ssh-privatekey=./deploy_key --from-file=known_hosts=./known_hosts`;
    set `gitPod.auth.type=ssh`, `gitPod.auth.existingSecret`, and `gitPod.auth.ssh.knownHostsKey=known_hosts`
    or `--set-file gitPod.auth.ssh.knownHosts=./known_hosts`; get known_hosts lines from the
    provider's published host keys, or from `ssh-keyscan` only after comparing
    `ssh-keygen -lf` fingerprints with the provider's published ones; host keys are always
    verified, ssh never adds one; the key is copied into an in-memory volume with mode 0600, and
    a rotated Secret is used on a later sync; a non-standard port needs the `ssh://` form; the
    Git pod needs egress to that port; the chart mounts a two-line `/etc/passwd` because ssh
    needs an entry for the pod's uid.
  - line 182, the `@` refusal: true for `none` and `token`; with `ssh` a user name is allowed and a
    password-shaped `user:secret@` is refused; reserved names now include `GIT_SSH_VARIANT`.
  - lines 197-199 (options list): `repoUrl` as in line 159; `auth.type` adds `ssh`;
    `existingSecret` "(required for `token` and `ssh`, empty for `none`)"; new rows
    `gitPod.auth.ssh.privateKeyKey` (default `ssh-privatekey`), `gitPod.auth.ssh.knownHostsKey`
    and `gitPod.auth.ssh.knownHosts` (exactly one required for `ssh`).
- `docs/tea-upgrade/security-assessment.md` line 85 (Git pod note): add `or with
  gitPod.auth.type: ssh a private one with a deploy key from a mounted Secret, host keys verified
  against a required known_hosts (StrictHostKeyChecking=yes)`; line 97 (Secrets): add that the
  SSH key is copied only into an in-memory volume with mode 0600 and is never in values, a URL, an
  environment variable, the content volume or a log.
- `oolong/oolong-server/test/git-auth.integration.sh` lines 360-366 (test 10): `ssh` leaves the
  refused list (add `SSH` and `Ssh` instead) and the expected message becomes `must be none, token
  or ssh`.

## 4. Tests the coder must add

Fixtures are generated per run (`ssh-keygen -N ''` in a temp directory); no key is committed. Host
names in render-only cases use `example.invalid` or `git.example.test`.

### 4.1 `helm-chart/oolong/scripts/chart-render.integration.sh`

`SSH_ARGS` = `COMMON_ARGS` with `gitPod.repoUrl=ssh://git@git.example.test/org/repo.git`,
`auth.type=ssh`, `auth.existingSecret=oolong-git-ssh`.

1. **ssh wiring, Secret known_hosts** (`SSH_ARGS`, `auth.ssh.knownHostsKey=known_hosts`): in the
   git Deployment, `OOLONG_GIT_AUTH_TYPE` is `"ssh"`; `git-auth` is projected with
   `defaultMode: 0440`, a secret source `name: "oolong-git-ssh"`, `optional: false`, items
   `ssh-privatekey -> identity` and `known_hosts -> known_hosts`, no configMap source; its mount
   is `/etc/oolong-git-auth`, `readOnly: true`, no `subPath`; `git-ssh` is an emptyDir with
   `medium: Memory` mounted at `/tmp/oolong-git-ssh`; `git-passwd` mounts `/etc/passwd` with
   `subPath: passwd`, `readOnly: true`, from ConfigMap `oolong-git-ssh`; that ConfigMap's
   `passwd` is exactly the two lines of 3.3 with `10001:0`; no `secretKeyRef`, no
   `GIT_SSH_COMMAND` in the pod env; the script calls `configure_git_auth` with
   `/tmp/oolong-git-ssh`; the rendered `git-sync.sh` holds the conditional
   `protocol.ssh.allow always` / `never` block.
2. **ssh wiring, inline known_hosts** (`SSH_ARGS`, `--set-file auth.ssh.knownHosts=<generated
   file>`): the projected volume has the secret source with only `identity` and a configMap
   source `known_hosts -> known_hosts`; the ConfigMap's `known_hosts` equals the file; the
   `checksum/git-script` differs from a render with a different `knownHosts`.
3. **token and none unchanged**: in `RENDER_A` and `RENDER_TOKEN` there is no `git-ssh`, no
   `git-passwd`, no `/etc/passwd`, no `projected`, no ConfigMap `oolong-git-ssh`, and the
   rendered script, for these types, still reaches `protocol.ssh.allow never` (the conditional's
   else branch; the existing token case still passes untouched).
4. **uid in passwd**: `SSH_ARGS` + `gitPod.securityContext.runAsUser=12345` gives
   `oolong-git:x:12345:0:`; with `gitPod.securityContext.runAsUser=null` and no
   `podSecurityContext.runAsUser` the render fails naming `gitPod.securityContext.runAsUser`.
5. **known_hosts required, schema**: `schema_refuses` (extended to take the URL) for: ssh with
   neither known_hosts source; both set; empty `privateKeyKey`; no `existingSecret`;
   `knownHostsKey` with a space; type `SSH`. The OOL-1 line `schema_refuses "type ssh"` (line
   236) is removed.
6. **known_hosts required, template guard**: the same five auth cases with
   `--skip-schema-validation` each fail and the output names the path
   (`gitPod.auth.ssh.knownHostsKey and gitPod.auth.ssh.knownHosts`,
   `gitPod.auth.ssh.privateKeyKey`, `gitPod.auth.existingSecret`).
7. **a private key in knownHosts is refused without echo**: `knownHosts` set to a file holding
   `-----BEGIN OPENSSH PRIVATE KEY-----`, a line `not-a-real-key-0001` and the END line: the
   render fails with and without `--skip-schema-validation`, the output does not contain
   `not-a-real-key-0001`, and (template run) names `gitPod.auth.ssh.knownHosts`.
8. **password in an ssh URL is refused without echo**: for
   `ssh://git:not-a-real-token-0001@git.example.test/org/repo.git` and
   `git:not-a-real-token-0001@git.example.test:org/repo.git` with type ssh, render fails with and
   without `--skip-schema-validation`; output never contains `not-a-real-token-0001` and names
   `gitPod.repoUrl`.
9. **URL forms accepted** with type ssh (render succeeds, `GIT_REPO_URL` value equals the input):
   `ssh://git@git.example.test/org/repo.git`, `ssh://git.example.test:2222/org/repo.git`,
   `git@git.example.test:org/repo.git`, `git.example.test:org/repo.git`,
   `git@git.example.test:/srv/git/repo.git`.
10. **URL forms refused** with type ssh, each with and without `--skip-schema-validation`:
    `https://git.example.test/org/repo.git`, `file:///tmp/repo.git`, `git://git.example.test/r`,
    `git+ssh://git@git.example.test/r`, `ext::sh -c true`, `ssh://-oProxyCommand=true/r`,
    `ssh://git@-host/r`, `ssh://git@@git.example.test/r`, `ssh://git@git.example.test:0/r`,
    `ssh://git@git.example.test:70000/r`, `ssh://git@git.example.test`, `git@git.example.test:`,
    `git@git.example.test:-x/r`, `git@git.example.test://r`, `ssh://git@[::1]/r`,
    `ssh://git@git.example.test/r x`.
11. **ssh forms refused for none and token**: `ssh://git@git.example.test/org/repo.git` and
    `git@git.example.test:org/repo.git` with type none and with token both fail (schema `@`
    rule, and with `--skip-schema-validation` the template guard), unchanged OOL-1 behaviour;
    `ssh://git.example.test/org/repo.git` (no `@`) with type none fails with the https message.
12. **allowlist on ssh URLs at render**: with `allowedHosts={git.example.test}` the URL form and
    the scp form render; with `allowedHosts={other.example.test}` both fail with
    `gitPod.repoUrl host is not in gitPod.allowedHosts`; same pair for an `https://` URL with type
    none (D9); the rendered `GIT_REPO_HOST_ALLOWLIST` is passed through as before.
13. **reserved names**: add `GIT_SSH_VARIANT` to the loop at lines 261-266 (`GIT_SSH`,
    `GIT_SSH_COMMAND` are there).
14. **lint**: `helm lint` accepts type ssh with `existingSecret=x` and `ssh.knownHostsKey=k`.
15. **order**: for an ssh render, `validate_git_repo_url || exit 1` and the `configure_git_auth`
    line come before `Repository: $GIT_REPO_URL` (extend the existing check at lines 319-324).

### 4.2 New `oolong/oolong-server/test/git-ssh-auth.integration.sh` (+ jest wrapper `src/tea/git-ssh-auth.integration.spec.ts`, same shape as `git-auth.integration.spec.ts`)

Setup, modelled on `git-auth.integration.sh` lines 1-80 (isolated `HOME`, `GIT_CONFIG_GLOBAL`,
`GIT_CONFIG_NOSYSTEM=1`, the `flock` shim, every git call under a 30 s timeout): requires `git`,
`ssh`, `ssh-keygen`, `ssh-agent`, `sshd` (by absolute path, `command -v sshd` or `/usr/sbin/sshd`)
and `python3`, and fails naming the missing tool. Generates in a temp dir: an sshd host key, a
second "impostor" host key, client keys `good`, `good2`, `wrong` (all `-N ''`) and one
passphrase-protected key. Starts `sshd -D -e -f <cfg> -E <log>` with `ListenAddress 127.0.0.1`,
a free port picked by python, `HostKey`, `AuthorizedKeysFile` (the `good` public key with the
`restrict` option), `PidFile`, `UsePAM no`, `StrictModes no`, `PasswordAuthentication no`,
`KbdInteractiveAuthentication no`, `LogLevel VERBOSE`; killed in the EXIT trap. A bare repo with
`products/` on `main` lives in the temp dir; `GIT_REPO_URL=ssh://<id -un>@127.0.0.1:<port><abs
path>`. The auth dir mimics the mount: `identity` and `known_hosts` with mode 0440;
`known_hosts` holds `[127.0.0.1]:<port> <host pub key>`. `OOLONG_GIT_AUTH_TYPE=ssh`, then
`validate_git_repo_url`, `configure_git_auth "$HELPER" "$AUTH_DIR" "$SSH_DIR"` and
`locked_sync_and_publish` as `git-sync.sh` does, with `git config --global protocol.ssh.allow
always` in the isolated global config.

Every negative case records the sshd log's line count first and asserts on the new lines only,
and first asserts that the attempt reached sshd (a new `Connection from 127.0.0.1` line), so a
"nothing was accepted" assertion cannot pass because nothing connected.

1. **clone, publish, fetch**: first sync publishes revision `<sha>` with `products/`; a second
   commit is pushed; the next sync publishes it. New log lines contain `Accepted publickey` with
   the `good` key's SHA256 fingerprint (`ssh-keygen -lf`), and no other key's fingerprint.
2. **mismatched host key**: `known_hosts` names `[127.0.0.1]:<port>` with the impostor key;
   `configure_git_auth` succeeds (an entry exists) and the sync fails; output contains
   `Host key verification failed` or `REMOTE HOST IDENTIFICATION HAS CHANGED`; the new log lines
   have `Connection from` and contain neither `Accepted publickey` nor `Failed publickey` nor
   any key fingerprint (the identity is never offered to an unverified host); the copied
   `known_hosts` is byte-identical to the source afterwards (ssh added nothing); no new revision
   and the pointer file is unchanged.
3. **no known_hosts entry**: `known_hosts` holds only `other.example.test <key>`;
   `configure_git_auth` returns 1 with `has no entry for [127.0.0.1]:<port>`; no new sshd log
   line. Non-vacuous because case 1 connected with the same setup.
4. **wrong key, agent and default identities not used**: start `ssh-agent`, `ssh-add` the `good`
   key, export `SSH_AUTH_SOCK`; put the `good` key also at `$HOME/.ssh/id_ed25519`; configure with
   the `wrong` key. The sync fails with `Permission denied (publickey)`; new log lines show
   `Connection from` and `Failed publickey` with the `wrong` fingerprint, no `Accepted publickey`,
   and the `good` fingerprint nowhere. Kill the agent.
5. **copy and mode**: after `configure_git_auth`, `$SSH_DIR/identity` and `$SSH_DIR/known_hosts`
   exist with mode 0600 (`stat -c %a`, or `ls -l` parsing), owned by the test uid, contents equal
   to the 0440 sources; no `.new` file is left.
6. **the exact ssh command**: `GIT_SSH_VARIANT` is `ssh`; `GIT_SSH_COMMAND` equals the 3.4 string
   with this run's `$SSH_DIR`; it contains `StrictHostKeyChecking=yes` and `IdentitiesOnly=yes`
   and does not contain `accept-new`, `StrictHostKeyChecking=no` or the key's content; no
   `GIT_CONFIG_*` variable is exported by the ssh branch.
7. **key never leaks**: after case 1, a distinctive line of the `good` private key (its second
   line) appears in no file under `CONTENT_DIR` (revisions, pointers, `_work` including
   `.git/config`), not in `env` output, not in the combined output of every step; `remote
   get-url origin` equals `GIT_REPO_URL`.
8. **start-up validation, nothing printed from the files**: `configure_git_auth` returns 1 and
   names the file for: missing `identity`; empty `identity`; the passphrase-protected key
   (`is not an unencrypted SSH private key`, and no prompt: bounded by the timeout); a public key
   as `identity`; missing or empty `known_hosts`. For each, the output does not contain the
   fixture's second line, and nothing is left in `$SSH_DIR` except earlier valid files.
9. **URL validation table, no network** (`validate_git_repo_url`, `OOLONG_GIT_AUTH_TYPE=ssh`):
   the accepted and refused vectors of 4.1 cases 9 and 10 give the same verdicts; the scp form
   with an allowlist that holds its host passes and with one that does not fails; the URL form
   likewise; `$(touch <mark>)` inside an ssh URL is refused and the mark is not created. With
   `OOLONG_GIT_AUTH_TYPE=token` and unset, both ssh forms are refused and `https://` still passes
   (the existing `git-publish.integration.sh` cases keep passing unchanged).
10. **known_hosts name**: with a scp-form URL `git@git.example.test:org/repo.git` and a
    known_hosts holding only `git.example.test <key>`, `configure_git_auth` succeeds (no
    connection is made); with only `[git.example.test]:2222 <key>` it fails; with
    `ssh://git@git.example.test:2222/org/repo.git` the reverse.
11. **rotation without reconfiguring**: replace `$AUTH_DIR/identity` with `good2`, put only
    `good2` in `authorized_keys`, push a commit; the next `locked_sync_and_publish` (no new
    `configure_git_auth`) publishes it and the new log lines show `good2`'s fingerprint. Then
    replace `identity` with garbage: the next sync fails naming the file and the published pointer
    is unchanged.
12. **argument checks before anything is exported**: type ssh with no third argument, and with
    `$SSH_DIR` containing a space, return 1 with the path message, and `GIT_SSH_COMMAND` stays
    unset (run in a subshell and print it).

### 4.3 Updated suites

- `oolong/oolong-server/test/git-auth.integration.sh` test 10, as in 3.7.
- `oolong/oolong-server/test/git-publish.integration.sh` lines 380-404 stay as they are and must
  pass (unset type keeps the https rules).

### 4.4 Suites and checks

- In `oolong/oolong-server` with Node 24: `npm ci`, `npm run lint`, `npm run build`, `npm test`
  (runs all shell suites through their jest wrappers; needs `helm`, `git`, `python3`, OpenSSH
  client and server), `npm run test:e2e` as usual.
- `helm lint helm-chart/oolong --set image=oolong-server:local` for each of the three types.
- ASCII check of every changed file ("Keep source files plain ASCII").
- **Image check (explicit step; report the output in the implementation note).** With Docker,
  against the pinned `gitPod.image`:
  `docker run --rm --user 10001 --entrypoint /bin/sh <image> -c 'command -v ssh ssh-keygen; ssh -V; ssh -F /dev/null -G example.invalid >/dev/null; echo rc=$?'`
  must show both tools and `rc=255` with `No user exists for uid 10001`; the same with
  `-v <rendered passwd file>:/etc/passwd:ro` must show `rc=0`, and
  `ssh-keygen -y -P '' -f <generated key>` must work. If Docker is not available, say so; the
  layer inspection in 3.0 is the evidence. Optional cluster check (kind or similar) with a
  Secret of type `kubernetes.io/ssh-auth` and a reachable test sshd.

## 5. Risks and open points

- **Fits one coder pass.** One repository; chart, one shell library, two test files and docs,
  about a day as estimated. No split; signed off with `--no-split`.
- **D4 is a new kind of mount** (an image file shadowed by a ConfigMap). It is limited to type
  `ssh`, read-only, and the content is two fixed-format lines from validated integers. If the
  image is later replaced by one with a real entry for the uid, the mount still works.
- **Behaviour change (D9):** a `gitPod.allowedHosts` that does not contain the `repoUrl` host now
  fails the render for every type instead of crash-looping the Git pod. Nothing that synced
  before stops working.
- **Behaviour change:** `gitPod.env.GIT_SSH_VARIANT` now fails the render (pre-1.0 chart, README
  says so).
- **URL grammar (D8) is stricter than git**: a repository path with characters outside the set,
  an IPv6 literal host or a scp form with a port is refused; the URL form covers ports.
  Documented; can be widened later with the same three implementations and vectors.
- **Image facts are from the amd64 layer**; the 4.4 image check confirms them. The image's git
  is 2.54.0 (tag 2.54.1); every git feature used (`GIT_SSH_COMMAND`, `GIT_SSH_VARIANT`,
  `protocol.ssh.allow`) is many releases older.
- **Test host requirement:** the 4.2 suite needs an OpenSSH server binary on the machine that runs
  `npm test` (it runs unprivileged on 127.0.0.1 and is killed at exit). The CI image build does
  not run tests ("Tests own their data and run locally").
- No open questions for the operator.
