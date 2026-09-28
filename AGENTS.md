# Repository guidance

## Precedence

- Follow user instructions first, then this file, then the scoped rules in
  `.agents/rules/`.
- More specific rules override general rules when both apply.
- Product documentation defines intended behavior. Agent rules govern how work
  is performed; they must not invent product behavior.
- If documents conflict, identify the conflict and ask which behavior is
  authoritative before implementing it.

## Authorization and assumptions

- When the user asks a question or requests a review, do not make code changes
  unless they explicitly request them.
- Bug reports, logs, reproduction steps, and statements that something appears
  broken are requests for diagnosis, not implicit authorization to edit.
- When the user explicitly asks to change, fix, or implement something, carry
  the work through implementation and verification.
- Do not guess at missing product behavior, authorization boundaries, data
  lifecycle rules, or security requirements. Ask concise, grouped questions
  when answers are required to proceed safely.
- If requested, constraints cannot all hold, stop and explain the exact conflict.
  Do not silently relax a constraint or implement an approximation.
- If an instruction appears mistaken or conflicts with established behavior,
  state that directly instead of silently working around it.
- Do not add speculative abstractions or compatibility layers. Add indirection
  only for a current, concrete need.
- Reuse an existing type when it represents the same concept and contract. Use
  separate transport, domain, or persistence types when representation,
  ownership, serialization, or invariants genuinely differ.
- Keep one source of truth. Do not duplicate facts in counters, metadata, or
  indexes unless the product requires a separately maintained projection.
- If the user refers to the latest screenshot, use the newest file in
  `./tmp/screenshots/`.

## Product documentation

- `README.md` is the public product overview.
- `docs/README.md` is the audience-based documentation index.
- `docs/product-requirements.md` is the contract for confirmed product behavior.
- Other files in `docs/` contain current domain and operational drafts.
- The current documents are incomplete. Do not interpret silence as a product
  decision.
- Programming and architecture decisions must be documented separately from
  product behavior and agent conduct.
- When changing behavior, update every directly affected human-readable contract
  in the same task.

## Product scope and vocabulary

- **cicadad** means the daemon that discovers job configurations and executes them
- **cicada** means a cli used the regular users to create and validate the job configurations.
- **job** means a process executed at a given time
- **daemon log** during job execution, exceptions or errors may occur that cannot be logged to the job log file.

## General coding priorities

- Prefer simple, readable code and the smallest coherent local change.
- Keep changes scoped, but make touched code internally consistent. Do not leave
  half-converted patterns in the same responsibility.
- Follow established local style when it remains compatible with current rules.
- Do not rewrite useful user-authored comments unless the change makes them
  inaccurate.
- Prefer clear control flow over blanket rules about early returns, nesting,
  `break`, or `continue`.
- Extract a helper when it creates a meaningful abstraction, is reused, or
  substantially improves readability. Do not add forwarding helpers that merely
  rename one call.
- Make I/O and other side effects visible at appropriate boundaries.
- Do not force zero duplication. Remove duplication when one abstraction has a
  clear responsibility and improves maintenance.

## Errors and invariants

- Distinguish normal conditions, invalid external input, expected concurrent or
  stale state, dependency failures, persisted-data problems, and true internal
  invariant violations.
- Return or translate errors for invalid input, authorization failures,
  dependency failures, storage failures, persisted-data inconsistencies, and
  other conditions a server must handle without terminating.
- Panic only for a genuine internal invariant violation where continuing would
  indicate a programming error. Do not use panic for unresolved requirements,
  request-derived state, provider responses, or recoverable job failures.
- Do not add silent fallback or no-op behavior for unspecified states.
- Do not swallow errors with blank identifiers, empty results, or debug-only
  logging.
- Preserve error identity when callers need programmatic handling and add useful
  context when wrapping errors.
- Never commit `panic("TODO")` or equivalent placeholders for unresolved
  behavior. Ask for the missing decision or leave that work unimplemented.

## Go guidance

- Follow `.agents/rules/golang.md` and `.agents/rules/techstack.md` where they do
  not conflict with this file or current product decisions.
- Use `gofmt` and idiomatic package and identifier names.
- Keep `main` focused on configuration and dependency wiring.
- Pass `context.Context` through request, database, job, and provider flows.
- Avoid mutable global state beyond deliberate process-level wiring.
- Close HTTP response bodies, database rows, files, transactions, and other
  owned resources on every path.
- Use parameterized SQL. Do not build queries by concatenating untrusted values.
- Keep package responsibilities clear; do not create `util` or `misc` dumping
  grounds.
- Use a single-field struct when it establishes useful semantics, ownership,
  synchronization, lifecycle, or a method-bearing abstraction. Do not add one
  merely to rename a value.
- Document concurrency guarantees for exported types when concurrent use is
  relevant.
- Follow `.agents/rules/echo.md` for handlers, middleware, server operation, and
  HTTP tests.

## Testing

- Test observable behavior, domain invariants, regressions, and security
  boundaries. Do not add tests that merely mirror constants or freeze incidental
  implementation details.
- Choose table-driven tests, subtests, assertion libraries, and parallelism when
  they improve the test. They are not mandatory patterns.
- Do not call `t.Parallel()` when tests share SQLite databases, environment
  variables, ports, fixture directories, global configuration, or other mutable
  state unless isolation is proven.
- Use real migrations in SQLite integration tests.
- Test authorization across role, course, student, action, and ownership scope.
- Test provider timeouts, cancellation, malformed responses, throttling, and
  outages through local fakes or `httptest` servers.
- Add upload, traversal, SSRF, and input-limit regression tests where relevant.
- Test background-job retry, idempotency, duplicate delivery, and crash recovery
  when those behaviors are introduced.

## Workflow and verification

- Work from the repository root unless a command requires a narrower working
  directory.
- Inspect existing files and nearby conventions before editing.
- Do not revert or overwrite unrelated user changes.
- Do not hide warnings with `nolint`, `lint:ignore`, blank assignments, or
  similar suppression unless the user explicitly approves it.
- Fix new findings in touched code. Report unrelated existing findings rather
  than broadening the task without authorization.
- Clean up temporary byproducts created during the task. Preserve intended
  generated artifacts and mention them in the result.
- Follow the 120-character Markdown limit configured in
  `.markdownlint.json`; prefer shorter lines when they remain readable.
  Remember: Markdown inside `_bmad`, `_bmad-output`, `.agents/skills`, `.opencode`, `.cache`, and `vendor` must not be
  validated. Files are accepted as they are.
- Follow `.agents/rules/markdown.md` for Markdown and
  `.agents/rules/toml.md` for TOML.

Once Go code exists, run the applicable checks while iterating on a change:

1. `gofmt` on changed Go files.
2. `go test ./...`.
3. `go vet ./...`.
4. `golangci-lint run ./...`.
5. `go test -race ./...` when concurrency-sensitive behavior changes and the
   environment supports it.

Do not run overlapping Go build, test, vet, or lint commands against the same
module. If a required tool or module does not exist yet, state that verification
was not applicable rather than inventing project setup.

### Full project check

`./run-all-tests.sh` is the complete test suite and the authoritative gate. The
Go checks above are the fast subset for iterating; they are not sufficient on
their own, because the script also enforces checks that no Go command covers.
Run the full script before reporting work complete whenever a change touches Go
sources, Markdown, the OpenAPI description, the shell scripts, or dependencies.
The repository has no CI, so nothing else runs these checks. Skipping the script
is how the duplication gate stayed red across four consecutive stories.

The script runs these components in order:

1. `gofmt -l` over tracked and untracked Go files.
2. `go test ./...`.
3. `go vet ./...`.
4. `golangci-lint run ./...`.
5. `go test -race ./...`.
6. A duplication marker balance check over Go files. An unbalanced
   `jscpd:ignore-start` suppresses duplication detection to the end of that
   file while still exiting successfully, so the imbalance has to fail before
   the scan reports a clean result.
7. JSCPD duplication detection over Go and markup sources at `--threshold 0`.
   Follow `.agents/rules/duplication.md` when it reports a clone.
8. `govulncheck ./...`, only when that tool is installed.
9. `trivy fs .` for dependency vulnerabilities and committed secrets.
10. Redocly lint of `api-doc/openapi.yaml`.
11. `markdownlint` over tracked Markdown outside `_bmad`, `_bmad-output`,
    `.agents/skills`, `.opencode`, `.cache`, and `vendor`.

The script includes the individual Go checks, so do not repeat them afterwards.
Several components download tooling through `npx` and need network access. When
a component cannot run, name it and say it was skipped instead of implying the
full suite passed.

## More rules

Read and implement all rules from `.agents/rules/*.md`
