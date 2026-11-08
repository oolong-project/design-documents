# OOL-1 design review, round 2

Task: OOL-1, Git pod: clone private repositories over HTTPS with a username and token from a
Secret. Reviewed: `design/OOL-1/architecture-2.md` (round 2) and
`design/OOL-1/threat_model-2.md` (round 2), starting from their "Changes in this round"
sections, against oolong-project/oolong `2026-11-oolong` @ `f216f68` and the coding principles
at `145008cdcc23`. What round 1 passed and round 2 did not change (D1 to D5, the schema for
`gitPod.auth`, the helper, the reserved names) is not reopened.

Verdict: PASSED. F-1, F-2 and F-3 are RESOLVED; no new items.

## F-1: The redirect test never has the token in hand when the redirect happens (RESOLVED)

Architecture 3.4 (D4) and 4.2, threat model T3. The test server gets an explicit `--mode`, and
the new `auth-redirect` mode answers unauthenticated requests with 401 and authenticated ones
with a 302 to the other host. New 4.2 case 8 asserts that the configured host's log holds an
`auth=yes` request (so the token was really sent and the redirect followed an authenticated
request), that the target's log holds at least one request from that clone and none with
`auth=yes`, and that no token is left in `$HOME` or the work directory. Cases 6 to 8 first
assert a non-empty log, and each log is truncated right before its clone, so no assertion runs
over an empty log or over another case's requests. The prototype's outcome (second host gets
no `Authorization`, 401, `Authentication failed`) matches git's behaviour: a 401 while
credentials are already held is rejected rather than refilled for the new host, and libcurl
drops the header across the host change. T3 now names the tested mechanism. That case 8 runs
against the test host's git, not the image's, is stated in section 5 and acceptable.

## F-2: A token in `gitPod.repoUrl` is logged and stored before the userinfo refusal fires (RESOLVED)

Architecture 3.2 (D6), 3.3, 4.1, threat model T1. `oolong.requireGitRepoUrl` now fails on `@`
anywhere in the URL, before the prefix check, with fixed text naming `gitPod.repoUrl`; that
matches `validate_git_repo_url` exactly (`case "$_rest" in *@*)` in `git-publish.inc.sh`
line 35), so render and pod refuse the same set, which is better than the authority-only rule
round 1 suggested. The new schema entry uses `not: {pattern: "@"}` with the reason (a positive
pattern's error echoes the value) recorded; 4.1 case 8 asserts the token is absent from the
output with and without schema validation, so it holds whichever Helm version runs it.
`git-sync.sh` now sources the include and validates before `echo "Configuration:"`; the move is
safe (the include only derives `ACTIVE_POINTER_FILE` from `CONTENT_DIR` at source time, and the
defaults it needs are set above), and 4.1 case 9 pins the order (the string
`validate_git_repo_url || exit 1` occurs only once in the rendered ConfigMap, so the `grep -n`
comparison is unambiguous). The behaviour change is listed in section 5.

## F-3: README line 157 still says the Git pod syncs a public repository (RESOLVED)

Architecture 3.7 now covers README lines 57, 157 and 176, and correctly leaves line 115 (the TEA
API's public catalog) alone.
