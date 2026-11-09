# OOL-4 implementation notes, round 2: CI deferred by the operator

Task: OOL-4 (child 1 of OOL-3). Round 1 (`impl/OOL-4/notes-1.md`) describes the build, the tests and
the departures; this round changes one thing about scope and nothing in the code.

## Pull request

| PR | Repository | Base | Head commit | Base tip merged |
|---|---|---|---|---|
| https://github.com/oolong-project/tea-client-go/pull/1 | oolong-project/tea-client-go | `2026-11-oolong` | `4d16b0c7fa04196bd3cca07bd01c0b622e96b420` (unchanged from round 1) | `82d03dac23f9` (the base had not moved) |

Merge order: not applicable (one PR). The repository has no CI that reports a result for this PR.

## Scope decision: no CI workflow in this round

Round 1 stopped because GitHub refused the push of `.github/workflows/ci.yaml` (the credential on the
coder host has no permission to write workflow files). The operator answered on the task: do not
build CI in this round; CI is set up after this round. So:

- The PR contains no `.github/` directory. It does not add `ci.yaml`, and it is not expected to.
- Design 3.11 is built except for `ci.yaml`: the README with its Licence and Versioning sections is
  in the PR. The jobs of 3.11 (`test`, `integration`, `record-release`, the check that the
  generated code is current) are not built in this round and are not claimed as done.
- The workflow written for round 1 is kept outside the repository, uncommitted, for the follow-up
  that sets CI up. Two facts for that follow-up: the integration job must check oolong out into a
  directory that is not named `oolong/` inside this module (oolong-server's `node_modules` holds a
  `.go` file that `./...` would otherwise treat as a package of this module; round 1 used
  `_oolong/`), and it needs Node 24 and `npm ci && npm run build` in `oolong-server` before the
  tests that start the server.
- The unpushed local commit and branch of round 1 (`ool-4-ci-pending`) no longer exist.

Round 1's sentence that this round "cannot be signed off PASSED" is superseded by this decision.

## Tests

Nothing changed. Re-run on the head `4d16b0c`: `gofmt -l .` prints nothing; `go vet ./...` and
`go test -count=1 ./...` pass (root package and `tei`). The integration tests, the fuzzers and the
mutation results are as reported in round 1.

## For the tester

- Check that the PR has no `.github/` content and that nothing in the README or the code refers
  to a workflow that is not there.
- The integration suite needs the oolong server built locally, as round 1 explains.
- models: round 2 by claude-sonnet-5-5 (a note only); the code of round 1 was built on claude-opus-5-5.
