# OOL-5 questions, round 1: discovery (child 2 of OOL-3)

Task: OOL-5, built from the design carried from OOL-3: `design/OOL-3/architecture-2.md` and
`design/OOL-3/threat_model-2.md` (round 2 of each). Coding principles read at `145008cdcc23`.
These questions are about the ARCHITECTURE round 2 release `28dce0e1-36f5-4840-b2cd-09b798a457ec`.

The discovery part is built and tested on https://github.com/oolong-project/tea-client-go/pull/2
(head `16ee9fa`, base `2026-11-oolong` at `1c85712`). Q-1 is a conflict between the design and the
pinned specification that the coder cannot settle alone; Q-2 to Q-4 are readings the coder made where
the design is silent or terse, asked now so that one design round settles all of them. For each,
the PR already implements the reading stated under "Built", so an answer of "confirmed" needs no
code change.

## Q-1: oolong's framework 404 is a TEA error response by the spec's definition; design case 4.2-3 expects a failed attempt

Design 3.0 ("Oolong server") says unmounted paths return the Nest body
`{"message":...,"error":"Not Found","statusCode":404}` and calls it "not a TEA error"; 4.2 case 3
then expects a well-known document listing `<server>/nope` to give a failed attempt there, "not
NotResolved". Running oolong `2026-11-oolong` at `e1c9e6e` confirms the body:
`404`, `Content-Type: application/json; charset=utf-8`,
`{"message":"Cannot GET /nope/v1.0.0/discovery?tei=x","error":"Not Found","statusCode":404}`.

That body is an `application/json` JSON object with a string `error` property. The pinned spec
(`discovery/readme.md` "Unresolved identifiers", commit `962a0228`) defines exactly that as a TEA
error response and adds: "Clients ... shall not reject the response for an `error` value they do
not know, so that a later TEA version can extend `error-response` without turning its `404`
responses into failover triggers." Design 3.6 states the same rule ("unknown properties and unknown
codes are tolerated") and step 5 makes a TEA 404 stop discovery with `*NotResolvedError`. So the
design's own rules give `*NotResolvedError` (unknown code `Not Found`) for case 3, and its test
expects the opposite. Satisfying case 3 as written would need a rule the spec forbids (refusing an
unknown `error` value, or recognising Nest's body by its `statusCode` property).

Built: the spec's rule. Case 3 (`TestIntegrationDiscoverWrongPath`) asserts `*NotResolvedError` at
`<server>/nope` with `RawCode` `Not Found` and no request to the second endpoint. A wrong path whose
404 is not a TEA error (`{"message":"Not Found"}`, `text/html`, an empty body, a JSON array) fails
over, as case 24 tests.

Asked: confirm the spec's rule and amend 3.0 and 4.2 case 3 to match (or say which other rule
applies and why the spec allows it).

## Q-2: does a successful well-known fetch count against the attempt bound?

3.6 step 6 says the bound is "4 attempts in total across both stages"; 3.4 says the default 4
attempts of `AttemptTimeout` plus backoff bound a whole `Discover` at "about 62 s" (4 x 15 s plus
0.25 + 0.5 + 1 s), which only holds when every GET counts, the successful well-known fetch
included. Case 26 says "four failing endpoints with attempts set to 3 make exactly 3 requests and
return `*DiscoveryError` listing them", which can also be read as three `/discovery` requests.

Built: every GET counts. With attempts 3 the client makes the well-known request and two
`/discovery` requests (3 requests), and the `*DiscoveryError` lists the 2 failed ones. A
consequence: `WithDiscoveryAttempts(1, ...)` lets `Discover` make only the well-known request
(it always ends in `*DiscoveryError`), while `DiscoverPURL` (no well-known stage) can still use its
one attempt; the option's doc comment says so.

Asked: confirm, or say that only failed attempts (or only `/discovery` requests) count.

## Q-3: when every candidate has failed and attempts remain, are the candidates tried again?

3.6 step 4 says "for each candidate in order", step 6 bounds the attempts, and case 26c says "with
one endpoint only and attempts exhausted, `errors.Is(err, ErrTooManyResults)` holds", which reads
as the single endpoint being asked until the bound is used up.

Built: the candidates are tried round-robin until the bound is used up (one endpoint and the default
4 attempts: one well-known request and three `/discovery` requests to that endpoint, with backoff).

Asked: confirm, or say that discovery ends with `*DiscoveryError` once each candidate was tried once.

## Q-4: statuses and answers step 5 does not name

Built, each a reading of "unusable" in step 5:

- a `/discovery` status other than 200, 400, 401, 403, 404 and 5xx (for example 204, 405, 429, or
  a 3xx the client did not follow) is a failed attempt;
- at the well-known stage, a TEA-shaped 404 is a failed attempt (there is no API base to report in
  `*NotResolvedError`, and "Unresolved identifiers" is about `/discovery`); 400, 401 and 403 there
  stop at once with `*APIError`, as in step 5;
- a `/discovery` answer is refused whole (a failed attempt) when any `servers[].rootUrl` in it is
  refused by the URL policy, even if another server in the same result is usable (case 24's "an
  http rootUrl not under an allowance"); a server `priority` outside [0, 1] is refused the same way
  (the `server-info` schema's bounds; step 5 names only rootUrl and versions);
- `SelectServer` applies D4 to the server its ordering puts first and returns `*URLPolicyError` if
  that one is refused; it does not fall back to the next server.

Asked: confirm, or say which of these should behave otherwise.
