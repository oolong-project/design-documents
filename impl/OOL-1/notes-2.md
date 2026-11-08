# OOL-1 implementation notes, round 2: tester item T-1

Task: OOL-1. Answers `tests/OOL-1/run-1.md` (BOARD_TEST_REPORT round 1, REJECTED, T-1 OPEN).
Round 1 (`impl/OOL-1/notes-1.md`) still describes the rest of the change.

## Pull request

| PR | Repository | Base | Head commit | Base tip merged |
|---|---|---|---|---|
| https://github.com/oolong-project/oolong/pull/4 | oolong-project/oolong | `2026-11-oolong` | `ae0fbf560988b2bd56860a776978493c0f834393` | `f216f6867c98f4f69826027bacd68bef258e30b0` (unchanged since round 1, so already contained) |

The tested head `9c03937` is an ancestor of the new head; nothing was rebased or force-pushed.
Merge order: not applicable (one PR).

## Review items

- **T-1 fixed.** Both parts of the item, in `helm-chart/oolong/scripts/chart-render.integration.sh`
  only:
  1. The five older refusal cases (`empty`, `old`, `inject`, `range`, `url`) no longer write to
     fixed paths in the shared temp directory; they write to one `mktemp` file (`CASE_OUT`).
     A redirect that cannot be opened can no longer pass a case without running helm. No fixed
     temp path is left in the script (`grep -n '/tmp/'` prints nothing).
  2. The first case ("requires a compatible image") rendered with no `--set image` and expected
     a refusal, which stopped holding when the base gained a default image. It now renders with
     `--set image=` (empty), which the chart still refuses with `image is required`.

## Tests

Run on the new head: `sh helm-chart/oolong/scripts/chart-render.integration.sh` prints 20 `test`
lines, no `Permission denied`, exits 0 and ends with `all chart render integration tests passed`;
the files under the shared temp directory that another user owns are not touched.
`npx jest src/tea/chart-render.integration.spec.ts src/tea/git-auth.integration.spec.ts`: 2 passed.
The other suites and checks from round 1 are unaffected (this round changes one shell test).

## For the tester

- Re-run `chart-render.integration.sh` on a host where the old fixed outputs are not writable
  and on one where they are; both now run every case.
- Not changed, per the report: the private-CA sentence in design 3.6 (the Git pod has no way to
  mount a CA bundle) is outside this task and recorded as a follow-up candidate; the git version
  in the pinned image has no effect on this change.
- models: claude-sonnet-5-5 only (no sub-agents).
