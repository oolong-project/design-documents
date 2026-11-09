# OOL-5 design review, round 1

Task: OOL-5, OOL-3b tea-client-go discovery (child 2 of OOL-3). Reviewed: the child's own
`design/OOL-5/architecture-1.md` (ARCHITECTURE round 1, release `d09e5d6e`), which carries
OOL-3's ARCHITECTURE round 2 (`28dce0e1`) for this child and amends it to answer the coder's
`questions/OOL-5/round-1.md` (Q-1 to Q-4); THREAT_MODEL stays OOL-3's round 2 (`a5ff2086`).
Compared against `design/OOL-3/architecture-2.md` (section 5, child 2) and
`design/OOL-3/threat_model-2.md`; the pinned spec's `discovery/readme.md` at `962a0228`;
oolong-project/tea-client-go `2026-11-oolong` @ `1c85712` (OOL-4 merged) for what this child
builds on; the coding principles at `145008cdcc23`. As a split child, OOL-3's design is not
reviewed again; what is reviewed is that the child matches it and the four amendments.

Verdict: PASSED. No items.

## Scope and carried content

- Scope matches OOL-3 section 5 child 2 exactly: 3.6 discovery part (`Discover`,
  `DiscoverPURL`, `SelectServer`, `DiscoveryError`/`Attempt`, steps 1-6 with
  `AttemptTimeout` and the `DiscoveryResults` cap), D4 on learned bases, the `live` job of
  `ci.yaml`. Out of scope (OOL-6, oolong, CLI) is the same.
- Decisions D4, D7, D8, D10 are carried verbatim; D3, D5, D9 are referenced as built in
  OOL-4 and used unchanged. D6 (TEI parsing) is OOL-4's and not needed here. What the child
  relies on from OOL-4 exists at `1c85712`: `Limits.DiscoveryResults` and `AttemptTimeout`,
  `WithDiscoveryAttempts`, `StageWellKnown`, `kindWellKnown`/`kindDiscovery`,
  `ErrTooManyResults`, the shape-only TEA error check.
- Test cases: 9b, 11b, 13b, 17, 21, 21b, 22, 25, 26b, 27b, 39b; 4.2 cases 1-2; 4.3 case 1;
  the `live` job are carried unchanged. 23, 24, 26, 26c, 27 and 4.2 case 3 change only as
  the Changes section says. No case of another child is added, and every case only uses
  code this child or OOL-4 builds (21b's `GetProductRelease`, 13b's `WithBearerToken`, 26b's
  `WithHTTPClient` are OOL-4's). Every threat-model test the table assigns to 3b (T1 11b,
  26, 26b, 26c; T2 17; T4 13b; T5 21b, 23, 24, 27, 27b; T6 21, 23, 24; T7 24, 26, 26c, 27;
  T13 39b) is still in the list.

## The amendments

- **Q-1** (oolong's Nest `404` is a TEA error response): checked against `discovery/readme.md`
  at the pin, "Unresolved identifiers": the definition is by shape only and clients "shall
  not reject the response for an `error` value they do not know". OOL-3's 3.0 and 4.2 case 3
  were inconsistent with OOL-3's own 3.6; the child now follows 3.6 and the spec. The
  non-TEA `404` failover that 4.2 case 3 used to exercise against oolong is still tested by
  unit case 24 (`{"message":"Not Found"}` and others; case 15's shapes), so no failover test
  is lost. The oolong consequence (a wrong base path stops instead of failing over) is named
  in section 5 for the operator; it is a server matter outside this task.
- **Q-2** (every discovery GET counts): this is what makes 3.4's 62 s bound and T1's time
  bound hold; case 26 now asserts it concretely (3 requests: well-known plus two
  `/discovery`, 2 listed attempts), and case 23's new sub-case (two well-known requests at
  attempts 2) pins it at the well-known stage.
- **Q-3** (round-robin until the bound): the bound and backoff are unchanged, so T1 and T7
  hold; case 26 (one endpoint, 3 `/discovery` requests) and 26c pin it.
- **Q-4** (other statuses fail over; every well-known `404` fails over; an answer with any
  D4-refused root or bad `priority` is refused whole; `SelectServer` does not fall back):
  each refuses more, never less, and is a choice the spec leaves open, named as one in 3.6
  and section 5 ("Implement the pinned TEA specification": the design says which). The
  refusals that guard a boundary have hostile-path cases: 24 (mixed https and refused http
  roots refused whole, distinguishable because the second endpoint's result is the one
  returned), 23 (TEA-shaped well-known `404`), 27 (no fallback past a refused first server).
- `SelectServer`'s re-check of each server's `rootUrl` pattern, versions and `priority`
  (`ErrInvalidArgument`) is new text not listed under Changes; it is stricter than OOL-3's
  "same rules as well-known", fails closed, and only applies to results a caller built, so
  it drops nothing and needs no item.

THREAT_MODEL: unchanged, and the claim holds: no answer adds an input, asset or boundary, and
no mitigation is weakened (Q-1 lets a server stop discovery at its own authority, which
`OBJECT_UNKNOWN` already allowed, T7).

## Routing notes

The task reached this role after the coder had already built and passed PR #2 (head
`50265c3`), because the architect's new round for the child ended the carried design-review
pass. The task's pinned ARCHITECTURE input is still OOL-3's `28dce0e1`; the round reviewed
here is the child's `d09e5d6e`, which the design checkpoint names.
