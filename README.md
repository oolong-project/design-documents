# design-documents

Design documents for the Oolong repositories
([oolong](https://github.com/oolong-project/oolong) and
[tea-client-go](https://github.com/oolong-project/tea-client-go)), handed between
the roles of the **OOLONG** agent task board on [ReARM](https://github.com/relizaio/rearm).
Each file is published to the board with `rearm agent doc publish`, which records it as
a release pinned to the commit it was read at. Rounds are history: never edit a
published round, publish the next one.

This repository is public, like the code it describes. Nothing in it may name a
secret, a credential, an internal hostname or any other non-public detail.

## The board

| Order | Role | Reads | Writes |
|---|---|---|---|
| 10 | `architect` | the task | `ARCHITECTURE` + `THREAT_MODEL` (both on the first round, then only what changed) |
| 20 | `design_reviewer` | `ARCHITECTURE`, `THREAT_MODEL` | `BOARD_REVIEW_ITEMS`: functional and security review in one pass |
| 30 | `coder` | `ARCHITECTURE`, `THREAT_MODEL` | `DETAILED_DESIGN`, PRs |
| 40 | `tester` | `DETAILED_DESIGN` | `BOARD_TEST_REPORT`; test and fixture PRs when coverage is missing |

The coordinator (the board's singleton seat) takes intake, splits and orders tasks,
and merges PRs (`PR_MERGE`) into the integration branches. The design reviewer and
the tester must be a different agent from the hop before them. Task keys are `OOL-n`.

- **Code:** integration branch `2026-11-oolong` in oolong and tea-client-go (cut from
  `main` on 2026-11-08). Integration branch to `main` is the operator's call.
- **Configuration:** [`board/board.yaml`](board/board.yaml), a board file
  (`kind: BOARD`). Change the board by editing it and applying it:
  `rearm agent board apply --file board/board.yaml --dry-run`, then without
  `--dry-run`, from a checkout of this repository so the apply records the file's
  commit. The file sets the design checkpoint on the coder (`coordinatorCheckpoint:
  true`), which only a person may set, so the file is applied by a person (the UI or
  a personal login), not by an agent key.

## Coding principles

[`coding-principles/coding_principles.md`](coding-principles/coding_principles.md)
holds the conventions for oolong and tea-client-go. The board's prompts pin it to one
commit of this repository (`git show <commit>:coding-principles/coding_principles.md`),
so an agent cannot change the rules it works under. Agents never edit anything under
`coding-principles/`. To change a principle, the operator commits the new version,
then moves the pinned commit in every prompt in `board/board.yaml` and applies it.

## Layout

| Type | Written by | Path |
|---|---|---|
| `ARCHITECTURE` | architect | `design/<key>/architecture-<round>.md` |
| `THREAT_MODEL` | architect | `design/<key>/threat_model-<round>.md` |
| `BOARD_REVIEW_ITEMS` | design reviewer | `review-items/<key>/design-reviewer/round-<round>.md` |
| `DETAILED_DESIGN` | coder (implementation note) | `impl/<key>/notes-<round>.md` |
| `BOARD_TEST_REPORT` | tester | `tests/<key>/run-<round>.md` + `.json` index |
| `BOARD_INVESTIGATION_REPORT` | coder or tester | `investigations/<key>/report-<round>.md` |
| `BOARD_QUESTIONS` | any role | `questions/<key>/round-<round>.md` |

`<key>` is the task key (`OOL-42`), shown on every card and by `task show`. The path
comes from the board (`rearm agent doc publish` asks it), relative to the repository
root.

## Rules

Never rewrite this repository's history: a published document pins the commit it was
published from, so never force-push, never `--amend` or rebase a pushed commit. If your
push is rejected because someone else pushed first, `git pull --no-rebase` and push
again; a merge commit with your trailers is fine. Fix a bad round by publishing the
next round.
