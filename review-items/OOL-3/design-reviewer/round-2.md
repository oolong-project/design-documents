# OOL-3 design review, round 2

Task: OOL-3, tea-client-go: Go client library for TEA v1.0.0. Reviewed:
`design/OOL-3/architecture-2.md` and `design/OOL-3/threat_model-2.md` (round 2), starting
from their "Changes in this round", against round 1 of this review
(`review-items/OOL-3/design-reviewer/round-1.md`), oolong-project/oolong `2026-11-oolong`
@ `e1c9e6e` (the only change since `0fac816` is OOL-2's git SSH test; no TEA, fixture or
spec change), oolong-project/tea-client-go `2026-11-oolong` @ `82d03da` (LICENSE only), and
the coding principles at `145008cdcc23`.

Verdict: PASSED. Every round-1 item is settled by the new round; nothing is OPEN. One new
low item (F-10) is ACCEPTED with a reason. What I re-checked beyond the items: the split's
case lists cover every 4.1 case and sub-case exactly once (1-8, 9a, 10, 11a, 11c, 11d, 12,
13a, 14a, 15, 16, 18-20, 39a, 40 in OOL-3a; 9b, 11b, 13b, 17, 21-27 with 21b, 26b, 26c, 27b,
39b in OOL-3b; 13c, 14b, 28-38 with 31b, 38b in OOL-3c), and each case only exercises code
its child or an earlier child builds; the threat model's new child column agrees. The new
`WithTrace` hook (T13) carries no header, query, body or credential, and test 39 searches
every field for the configured secrets. The new uuid and "under" rules are named as stated
departures (D8, D4, section 5), as "Implement the pinned TEA specification" asks.

## F-1: Basic credentials for /token can follow a redirect off the root (RESOLVED, security, ARCHITECTURE D5, 3.4, 3.7; THREAT_MODEL T4)

The token request now carries an unexported `kindToken` on its context, `CheckRedirect`
returns `http.ErrUseLastResponse` for it, and any `3xx` from `/token` is an
`*UnexpectedResponseError` with `Location` not echoed (3.7), for `API.RequestToken` and for
`WithAPIKey`'s automatic request alike. The auth transport additionally removes any
`Authorization` header from a hop outside a configured credential root (D5), so a header a
caller sets through `Generated()` cannot leak either. Because the kind key is unexported,
nothing outside the library can claim the `kindToken` exemption. Test 14a redirects `/token`
with `307` to another port and `302` to a subdomain and asserts the target sees no request;
14b does the same for the automatic request; 13a covers the bearer and the editor header. T4
names the path. Settled.

## F-2: No time bound on a stalled server, and discovery's attempt timeout is undefined (RESOLVED, security, ARCHITECTURE D3, 3.4, 3.6, 3.8; THREAT_MODEL T1)

`AttemptTimeout` (15 s), `CallTimeout` (30 s) and `IdleTimeout` (30 s, reset on each read that
returns bytes) are defined in `Limits`, applied by the bounded doer through the request
context, so they hold with a caller's `http.Client` and for `Generated()` calls (by path
class). The error rule (`ErrTimeout` only when the library's bound fired and the caller's
context is live) is exact, and a stalled discovery attempt is a failed attempt that fails
over. Tests 11d, 26b and 31b cover headers-never-sent, stalled body, trickling body (must
succeed), a caller client without timeouts, and an earlier caller deadline. T1 names slow
responses. Settled.

## F-3: The exact http allowance does not cover what --usehttp does today, and "under a base" is undefined (RESOLVED, functional, ARCHITECTURE D4, 3.10, section 5)

"Under" is defined on parsed URLs: same scheme, host and port (default port normalised), and
the escaped path equal to the base's path or starting with it followed by `/`; dot segments,
also percent-encoded, are refused before any allowance or credential decision; D5 uses the
same rule. The CLI passes the origin it already builds for `--usehttp`, which covers a local
ReARM's `<origin>/tea` endpoint and `servers[]` root, and still refuses http on any other host
or port (test 21b, 27b; `/teapot` in 9a). Widening from byte-equal to "under" is argued in
section 5 against the spec's "per configured base URL": the allowance stays a caller-configured
string, and a document can only choose a path on an origin the caller already declared
plain-http. I accept that reading. Settled.

## F-4: The split's test assignment does not match the sections each child builds (RESOLVED, functional, ARCHITECTURE section 5; THREAT_MODEL section 3)

Cases 9, 11, 13 and 14 are split into sub-cases, each assigned to the child that builds the
code, case 17 moves to OOL-3b, and the threat model's table gains a child column. Checked
above: every case is assigned exactly once and is buildable in its child. Settled.

## F-5: Validation and limits the generated layer cannot see are not specified (RESOLVED, functional, ARCHITECTURE D3, D8, 3.5)

Uuids are accepted when `google/uuid` decodes them, and every path is rebuilt from the decoded
value in canonical lowercase, stated as part of D8's departure; test 24 now uses values that
rule really refuses (`"not-a-uuid"`, 35 characters), and test 16 checks an uppercase uuid is
requested lowercase. The call kind reaches the doer as an unexported context value, and
`Generated()` requests are classified by path against the spec's templates (downloads and
signatures get their byte limit and `IdleTimeout`, everything else `JSONBytes` and
`CallTimeout`); test 11c checks both classes. Settled.

## F-6: Candidates walks every discovery result with no cap (RESOLVED, security, ARCHITECTURE 3.6, 3.9; THREAT_MODEL T1, T7)

`Limits.DiscoveryResults` (32) refuses a larger `/discovery` answer whole (a failed attempt)
and refuses more results in `Candidates` before any request, so one `FullFlow` makes at most
that many product-release requests. Tests 26c and 38b. Settled.

## F-7: The live test asserts on data, not structure (RESOLVED, functional, ARCHITECTURE section 4.3)

The TEI-uuid assertion is gone; case 2 now asserts that the fetched release's `uuid` equals the
discovery result it was fetched for, which is the contract of `GET /productRelease/{uuid}`.
Settled.

## F-8: Licensing of the copied spec files and of anything taken from rearm-client-go is not stated (RESOLVED, functional, ARCHITECTURE 3.0, 3.1, 3.11)

`spec/LICENSE` is a byte copy of the upstream Apache-2.0 `LICENSE` at the pin, checked by
blob id in test 1; no upstream `NOTICE` exists at that commit; `teaapi/doc.go`, `SPEC_PIN.md`
and README state the licence of `spec/` and that `teaapi` carries text from it. Nothing is
copied from rearm-client-go, with the rule for a coder who does copy. Settled.

## F-9: Module dependencies and versioning are not stated accurately (RESOLVED, functional, ARCHITECTURE 3.1, D1, 3.4, 3.11)

The generator moves to a separate `tools.mod` (prototyped, test 1 checks `go.mod` has no
`tool` line), so its graph stays out of consumers. The real `go.mod` requirements are stated,
including that `oapi-codegen/runtime`'s own requirements enter a consumer's module graph as
minimums, with the concrete effect on the CLI (no raise, two new entries), and accepted.
The v0 release policy and what a regenerate means for the version are stated, and the user
agent's version comes from `debug.ReadBuildInfo` with `devel` in the module's own tests
(test 40). Settled.

## F-10: A token request made through Generated() cannot carry Basic credentials (ACCEPTED, functional, ARCHITECTURE D3, D5)

New, low. Only wrapped requests carry a kind; a `Generated().RequestTokenWithResponse` call is
classified by path as `kindJSON`, so the auth transport treats its `Authorization` like any
other: removed when the base is not under a configured credential root, replaced by the
configured bearer when it is. A Basic header a caller sets through a request editor therefore
never reaches `/token`. Accepted, not held: it fails safe (credentials go nowhere), and the
wrapped `API.RequestToken` is the supported path with the no-redirect rule. The coder should
say it in the doc comment of `Generated()` beside the in-memory download note, so a caller is
not left guessing why a hand-built token request is answered `401`.
