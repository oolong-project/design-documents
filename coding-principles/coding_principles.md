# Coding principles for Oolong

The conventions reviewers push back on in oolong-project/oolong (the TEA server,
`oolong/oolong-server`, its container, Helm chart and docs) and
oolong-project/tea-client-go (the Go client). Read this before writing or reviewing
a non-trivial change. Most of these rules are not obvious from the surrounding code.

Each section gives the rule, why it exists, and what it means concretely. A change
that departs from a rule says so, with the reason, in its design or implementation
note.

This file is versioned like any other document: agents read it at the commit their
board pins and never edit it. A correction that keeps recurring is proposed to the
operator, who adds it here.

---

## Everything is public

**Why:** both code repositories and this documents repository are public,
MIT-licensed open source. Anything written into them is published for good: a
force-push does not take it back out of forks, caches and mirrors.

**Concretely:**

- Code, comments, commit messages, PR titles and bodies, fixtures, test output and
  documents name no secret, credential, token, internal hostname, private
  repository, private issue or other detail that is not already public.
- Describe a change in terms of these repositories only. If work elsewhere
  motivated it, the reason is stated in Oolong terms.
- Test fixtures use made-up data (example.com hosts, generated uuids, sample
  products). Never real customer data, never a real key, not even a revoked one.
- Logs pasted into a report are checked first for tokens, paths from a private
  machine and hostnames.

---

## Implement the pinned TEA specification; never change it

**Why:** Oolong is an implementation of the Transparency Exchange API. Clients are
written against the specification, not against Oolong. A server that drifts from
the spec breaks them quietly, and an edited copy of the spec hides the drift.

**Concretely:**

- `tea-spec/openapi.yaml` is a byte-for-byte copy of the upstream document at the
  commit `tea-spec/SPEC_PIN.md` names. Do not edit it. Moving to a newer upstream
  version is a task of its own (re-pin: copy the new file, update `SPEC_PIN.md`,
  then adapt the server).
- Paths, parameters, response shapes, status codes, headers and error codes follow
  the pinned spec. Where the spec leaves a choice open, the design says which
  choice was made and why.
- A deliberate departure from the spec (a known upstream defect, an extension) is
  named in the design and in a comment at the site, with the spec location.
- The Go client follows the same spec. A client that accepts a response shape the
  spec does not define is a bug, not leniency, unless the design says otherwise.

---

## Generated code is not edited by hand

**Why:** `oolong/oolong-server/src/generated-nest/` is OpenAPI Generator output
(`typescript-nestjs-server`, the version `openapitools.json` pins). A regenerate
replaces the whole directory, so a hand edit there is lost the next time, and
nobody notices until the behaviour goes with it.

**Concretely:**

- Implement behaviour in the handwritten code (`src/handlers/`, `src/controllers/`,
  `src/tea/`), never inside `src/generated-nest/`.
- If the generated types are wrong for the spec (the generator has historically
  mishandled `allOf` and response headers), write a handwritten type next to the
  handler and say why in a comment. Do not patch the generated file.
- A change that regenerates the directory is its own commit, made by the generator
  at the pinned version, with nothing else in it, so the diff can be reviewed as
  generated output.
- A green build after a regenerate is not evidence that the types match the spec.
  Responses are checked against the spec by tests.

---

## Every path built from input goes through the path guard

**Why:** the server serves files from a content directory, and request input
(identifiers, names, versions) ends up in file system paths. A path built without
the guard can read outside the content directory through `..`, an absolute path,
an encoded separator or a symlink. This is the main security boundary of the
server.

**Concretely:**

- Build every path that contains request input or content-supplied names through
  `PathGuard` and `isSafeSegment` in `src/tea/path-guard.ts`. Never join an
  unchecked segment with `path.join` or string concatenation.
- Resolve symlinks before the containment check, not after. The existing specs
  (`path-guard.spec.ts`, `content-symlink-traversal.spec.ts`) are the pattern for
  new cases.
- Validate identifiers by their declared format (a uuid is a uuid, a TEI matches
  its scheme) at the boundary, before they reach the content store.
- A new way of reaching the file system (a new content kind, a download, a git
  operation) comes with a test that a hostile path is refused.

---

## Bound the work and memory of every request

**Why:** Oolong is a public, unauthenticated API. A request that makes the server
walk the whole content tree, load every file or build an unbounded response costs
nothing to send and everything to serve. Nothing shows at test scale; it shows as a
slow or crashed server on a large content set.

**Concretely:**

- Look things up by index or by key. Do not scan the whole content set to answer a
  question about one object.
- Paginated endpoints enforce the spec's page size limits on the server, whatever
  the client asks for. Page tokens are verified and bounded; a forged or stale
  token is refused with the spec's error, not trusted.
- Responses that can grow (lists, collections) are paginated as the spec defines,
  never returned whole.
- File reads are bounded in size. A file larger than the limit the design states is
  refused or streamed, never read into memory whole.
- Regular expressions run on input are linear (no nested quantifiers on
  attacker-controlled text).

---

## Content reloads are all or nothing

**Why:** content changes under a running server (git sync, reloads). A request that
reads half of the old snapshot and half of the new one serves a release that never
existed, with checksums that match neither.

**Concretely:**

- A reload builds and validates the new snapshot completely, then switches to it in
  one step. Requests in flight finish on the snapshot they started with.
- A snapshot that fails validation is not served. The server keeps the last good
  one and logs why the new one was refused.
- Checksums, signatures and sizes served for an artifact always come from the same
  snapshot as the artifact itself.

---

## Prefer closed types over magic strings and booleans

**Why:** strings and booleans collapse a closed set of meanings into untyped values.
A typo or a rename silently turns a comparison into a no-op, and a second value
joining the set turns every boolean reader into a nest of conditions. The TEA spec
is full of closed sets (identifier types, artifact types, checksum algorithms,
error codes).

**Concretely:**

- In TypeScript, model a closed set as a string-literal union or an enum
  (`TeaErrorCode` in `src/tea/errors.ts` is the pattern), and type fields and
  parameters with it, not with `string`.
- In Go, declare a named type with typed constants (`type ArtifactType string` and
  `const ArtifactTypeBOM ArtifactType = "BOM"`), and switch on the constants.
- Parse the inbound value once, at the boundary, into the closed type. Unknown
  values are refused there (or mapped to an explicit unknown value where the spec
  says to tolerate them), and the code behind the boundary handles only known
  values.
- Use a closed type instead of a boolean whenever the domain has, or may grow, a
  third state.

---

## Timestamps are UTC instants in RFC 3339

**Why:** the TEA spec defines timestamps as RFC 3339 `date-time`. Strict clients
reject anything else, and stored or compared timestamps in mixed forms do not sort
or compare correctly.

**Concretely:**

- In TypeScript, write timestamps with `Date.prototype.toISOString()` (UTC, trailing
  `Z`). Never `toString()`, `toLocaleString()` or a hand-built format.
- In Go, format with `t.UTC().Format(time.RFC3339Nano)` (or `time.RFC3339` where the
  spec wants second precision), and parse with `time.Parse(time.RFC3339, ...)`.
- When reading timestamps from content files, accept any valid RFC 3339 value
  (offsets included) and normalize to UTC before comparing or serving.
- Timestamps in fixtures are fixed values, not the current time, so tests are
  repeatable.

---

## Errors: what the client sees, what the log gets

**Why:** an exception message can carry a file system path, a stack trace, part of
a content file or a configuration value. The client must never see it; the operator
must be able to find it.

**Concretely:**

- Client errors use the spec's error responses and codes (`TeaException` and
  `TeaErrorCode` in `src/tea/errors.ts`). Their messages name the parameter or the
  rule that failed and nothing else.
- Unexpected failures reach the client as the generic `Internal server error`
  response and are logged once at error level, with the error object, so the stack
  trace is in the log.
- Do not catch an unexpected error just to return its message.
- "Does not exist" and "is not served" look the same to the client, so a caller
  cannot probe which files or objects exist in the content directory.
- In Go, wrap errors with context (`fmt.Errorf("fetching release %s: %w", id, err)`)
  and never put a credential or a full response body into an error message.

---

## Secrets and credentials

**Why:** a secret written to a log, a response, a URL or a commit cannot be taken
back.

**Concretely:**

- Secrets (the page-token secret, git sync credentials, any token a deployment
  adds) come from the environment or the chart's Secrets, never from a values file
  default, a fixture or code.
- Never log or return a secret, not even truncated. Log which secret is in use by
  its name, not its value.
- Credentials travel in headers, never in a URL query string (proxies and access
  logs keep URLs). This applies to the Go client as much as to the server.
- A development fallback for a secret (such as `PAGE_TOKEN_DEV_FALLBACK`) is off by
  default and refused in production configuration.

---

## The container and the chart stay locked down

**Why:** Oolong is deployed by people who trust its defaults. A default that
loosens the runtime (root, writable mounts, extra ports, open egress) becomes their
exposure.

**Concretely:**

- The image runs as the unprivileged user the Dockerfile creates. Base images are
  pinned by digest.
- The server only reads the content directory; only the git publisher writes to
  it. A new mount the server does not need to write is mounted read-only.
- New chart values come with a schema entry in `values.schema.json` and a safe
  default. Values are never interpolated into shell text.
- A change that needs a new port, capability, writable path or outbound
  destination says so in its design, with the reason.

---

## The Go client treats every server as untrusted

**Why:** tea-client-go talks to any TEA server a user points it at, including
hostile ones.

**Concretely:**

- Bound what it reads: response sizes, page counts, redirects followed.
- Verify what the spec lets it verify (checksums of downloaded artifacts) and fail
  closed when verification fails.
- Follow only URLs whose scheme the design allows (normally `https`). Never send
  credentials across a redirect to another host.
- Every exported identifier is part of the public API: document it, and do not
  change or remove it without a design that says how existing callers are handled.
- Accept a `context.Context` on every call that does I/O.

---

## Keep source files plain ASCII

**Why:** non-ASCII punctuation (em-dashes, curly quotes, non-breaking spaces,
ellipsis characters) looks like its ASCII counterpart in most editors, but it breaks
parsers that expect ASCII or a specific encoding, diffs noisily, and defeats grep.
It usually arrives by pasting from a document or a chat.

**Concretely:**

- In code, comments, configuration, Helm templates, shell scripts and fixtures, use
  ASCII: `--` or `-` for dashes, straight quotes, plain spaces, `...`.
- The exceptions are fixtures that test Unicode handling on purpose, and quoted
  text that must stay exact (a spec reference such as a section sign may stay if
  it is copied from the source it cites, but prefer `RFC 9110 section 12.5.1`).
- Check with GNU grep by absolute path: `LC_ALL=C /usr/bin/grep -nP "[^\x00-\x7F]" <files>`
  must print nothing. Do not rely on a bare `grep`: some environments alias it to a
  tool that decodes UTF-8 and misses most smart punctuation. When in doubt, count
  bytes above 127 with a script; the count must be 0.

---

## Tests own their data and run locally

**Why:** the CI image build compiles the server but does not run its tests, so a
green build on a PR says nothing about them. And tests that share state pass or
fail depending on run order.

**Concretely:**

- Run the tests yourself before handing a change over. Server, in
  `oolong/oolong-server` with Node 24: `npm ci`, `npm run lint`, `npm run build`,
  `npm test`, `npm run test:e2e`. Go client: `gofmt -l .` (empty), `go vet ./...`,
  `go test ./...`.
- Every change ships with tests for the behaviour it changes, including the
  refusals the design names.
- A test creates the content it reads (a temporary directory or a fixture copied
  into one), never edits a shared fixture in place, and never depends on another
  test having run first.
- A new test goes in the spec file for its topic, or a new file named after the
  feature, not at the end of an unrelated file.
- Tests do not reach the network. A test that needs a TEA server starts one
  locally (or uses `httptest` in Go).

---

## More principles land here as they come up

When the same correction is made on two changes, that is the signal it belongs
here. Propose it to the operator with the two cases; do not edit this file from a
task.
