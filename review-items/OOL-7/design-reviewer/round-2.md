# OOL-7 design review, round 2

Task: OOL-7, TEA /token and Bearer auth in oolong (all-or-nothing), API keys in a YAML file,
interop tests in tea-client-go. Reviewed: `design/OOL-7/architecture-2.md` and
`design/OOL-7/threat_model-2.md` (round 2, docs commit `03e4307`), starting from their
"Changes in this round" sections, against oolong-project/oolong `2026-11-oolong` @ `77a17b3`,
oolong-project/tea-client-go `2026-11-oolong` @ `e848589` and the coding principles at
`145008cdcc23`.

Verdict: PASSED. F-1 to F-5 are RESOLVED and nothing is open. The split by repository
recommended in section 5 (child 1 oolong, now including 3.10; child 2 tea-client-go interop
tests, after child 1) still fits the design.

## F-1: RESOLVED

D12 loads `AUTH_*` and the key file in `createApp()` before `NestFactory.create` and hands
one `AuthStartup` value to `AppModule.forRoot`. The registry constructor reads nothing, so no
key-file error can occur inside Nest's dependency injection, and `process.env` is read once.
D13 adds `abortOnError: false` for anything else and `bootstrap().catch(reportStartupFailure)`,
which prints one `startup failed: ...` line and exits 1. E11 now asserts what that path
guarantees: a rejection with a `KeyFileError`/`AuthConfigError` message and no file content.
E12 spawns `src/main.ts` through `ts-node` (already a devDependency) and checks the stderr
line and exit status, which covers the operator-visible half.

## F-2: RESOLVED

3.10 defines readiness under `required`. `/.well-known/tea` must answer 200: it is 503 until
content loads, so it covers `CONTENT_AWAIT_GIT_REVISION`. Then `/v1.0.0/products` must answer
401 with a Bearer challenge. The script holds no key, and a `required` container that serves
without a token is reported unhealthy. Reading `PORT` instead of hard-coding 3000 keeps the
image's behaviour (default 3000) and makes H1-H4 possible. Docker runs the health command with
the container's environment, so `AUTH_MODE` is visible to the script. H1-H4 cover both modes,
the mismatch cases, and the not-yet-ready case.

## F-3: RESOLVED

The threat model's A1 now states the exception, and the residual risks include it. The README
"Authentication" section, the README redirect paragraph, the IMPLEMENTATION_NOTES security
assessment and the `required` NOTES text all tell operators to keep private bytes in the
content directory. No new test, as asked.

## F-4: RESOLVED

Refusals for known ids are counted per id and reason, and `AuthLog.flush()` writes at most one
line per id per minute. The counters are keyed only by ids in the current `KeySet`, so their
memory is bounded. T4 names the threat, and L1, L2 and the E10 burst case test it.
`token issued` stays one line per token, which is fine: only a holder of the secret can cause
it, and it is the audit trail.

## F-5: RESOLVED

D14 reads from one descriptor into a single 65537-byte buffer and refuses when the buffer
fills. `O_NONBLOCK` plus `fstat().isFile()` refuses a FIFO or directory without blocking. The
same read serves startup and reload. K5 covers the boundary, a 10 MB file (spy on the bytes
read), a FIFO, a directory, ENOENT and the `..data` chain.

## Nothing new

The round-2 changes introduce no new finding. The startup line prints no variable or file
value (A3, K4). The `err.stack` form is used only for errors that are not configuration errors,
which is no worse than today's unhandled-rejection report. 3.10 does not touch the chart's
probes.
