---
layout: post
title: "Exploring Delta Specs Inside a Gated Harness"
date: 2026-07-24
---

A follow-up to the [gated-vs-fluid post]({% post_url 2026-07-24-agentic-harness-styles %}):
this one is the actual design sketch for borrowing OpenSpec's one standout
idea — delta specs — into a PR-gated harness (`corpus-ticket` + `corpus-work`)
without touching the gates themselves. Nothing here is built yet; it's the
exploration, written down before any skill file changes.

## The gap it closes

Right now the harness's only lasting documentation is a one-shot narrative doc
per feature, written once at the final phase and never revisited except to
append a new source ticket. There's no persistent, queryable "what does the
`documents` domain guarantee today" that a future planning session can read —
just archaeology through old tickets. OpenSpec's delta specs solve exactly
this: a change never rewrites a domain's spec, it writes `ADDED` /
`MODIFIED` / `REMOVED` blocks against it, and those deltas merge into a
permanent, per-domain spec when the change is archived.

## The design constraint: zero new artifact types

The tempting move is to import OpenSpec's convention wholesale — a separate
delta file per change, living in its own folder until archive merges it. I'm
deliberately not doing that. The harness already has an artifact with exactly
the right lifecycle: a running handoff doc, created early, appended every
phase, folded once into permanent docs, then deleted. A delta doesn't need a
new file format and its own creation/deletion tracking — it needs one more
subsection in a document that already has that lifecycle solved.

## Where it plugs in, phase by phase

**`corpus-ticket`** — read-only, one addition. Its duplicate-search step
currently only checks the issue tracker. Add a check against
`docs/specs/<domain>/spec.md` for the relevant domain — not for duplicate
tickets, but for *contradicted requirements*. "Let uploads exceed the size cap
for admins" should trip on an existing `Per-file size cap` requirement and
force the question: deliberate `MODIFIED Requirement`, or did the ask just
forget the constraint exists? corpus-ticket never writes to the spec — only
the docs-handoff phase does that.

**Phase 0 (planning)** — gains a spec-recon step alongside its existing
codebase recon: read the relevant domain's spec, if it exists. The plan names
the target domain(s) explicitly in the handoff doc header, next to the fields
that already live there.

**Phase 1 (acceptance tests)** — alongside the interface contract it already
produces, add a `### Spec Delta` subsection: `ADDED Requirement` /
`MODIFIED Requirement` blocks in Given/When/Then form, one per acceptance
criterion, tagged with the target spec path. Staged in the handoff doc, same
PR, same review gate. Not yet true — the same "red until implementation"
discipline the tests already have.

**Phase 2 and loop-back** — no new mechanism. The existing loop-back trigger
("the phase-1 contract itself was wrong") already covers this; it just now
also means fixing the Spec Delta section, not only the interface contract and
the tests.

**Phase 3 (human revision)** — same extension. A real behavioral gap a human
catches during manual QA gets a delta addendum alongside the new test that
covers it, following the existing loop-back rule.

**Phase 4 (docs handoff)** — this is where the actual merge happens, and it's
the closest analog to OpenSpec's archive step. The fold into permanent docs
gains one more target: merge the accumulated Spec Delta section(s) into
`docs/specs/<domain>/spec.md` (creating the file if the domain is new), verify
the merge, *then* delete the handoff doc as it already does. Same
verify-then-delete discipline already applied to the handoff doc and the plan
artifact — one more thing folded, not a new discipline invented.

**Escape hatch** — explicitly exempt. An escape-hatch ticket by definition adds
no new user-observable surface, so there's nothing to add or modify. Worth
stating outright so it doesn't get force-fit onto a plain bug fix.

**Validation, before it ever reaches Phase 4** — waiting until the docs-handoff
merge to discover a malformed Spec Delta block is too late; by then it's the
last thing standing between the branch and the final PR. A pre-commit hook on
the handoff doc should validate each `### Spec Delta` section's structure the
moment it's committed — same spirit as `openspec validate`, which checks specs
and changes for structural issues on demand (and via `--all --json` in CI).
The harness doesn't need the full command, just the same instinct: catch a
malformed `ADDED`/`MODIFIED` block, a delta that targets a domain that doesn't
exist, or a requirement with no scenario, at commit time — not at fold time.

## What's still open

- Domain boundaries have to be decided somewhere, and they won't always
  cleanly match a project's existing project/label taxonomy — same problem
  OpenSpec itself doesn't solve for free.
- This is strictly more bookkeeping, not less. The docs-handoff phase does two
  folds instead of one, and the whole idea only pays for itself if the
  archaeology problem is actually being felt — a second or third ticket
  needing to know what an earlier one already promised.

Next step, if this survives more thinking: draft it as an actual change to the
`corpus-work` reference files and try it on a real ticket before deciding
whether it earns a permanent place in the harness.
