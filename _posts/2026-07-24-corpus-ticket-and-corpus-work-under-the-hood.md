---
layout: post
title: "Under the Hood: corpus-ticket and corpus-work"
date: 2026-07-24
---

[Last post]({% post_url 2026-07-24-agentic-harness-styles %}) compared my homegrown
harness against OpenSpec at the philosophy level — gated vs. fluid. This one skips
the comparison and just opens the thing up. It's two Claude Code skills,
`corpus-ticket` and `corpus-work`, built for and living inside **Corpus**, a
project I'm building solo (private repo, for now — the harness is the interesting
part, not the product it's scoping).

## corpus-ticket: an advisor that can say no

`corpus-ticket` turns a rough idea into a Linear issue for Corpus's team — but its
actual job is refusing to write a bad ticket, not writing tickets fast. The system
prompt is explicit about posture: *"You are my advisor, not my assistant. Never
open with agreement."* In practice that means every conversation runs through a
fixed set of checkpoints before anything gets drafted:

1. React to the idea — push back if it's vague, don't open with "Great idea!"
2. Search Linear for duplicates/overlap. Find one → branch into "is this the same
   thing, related-but-distinct, or genuinely separate" instead of drafting.
3. Interrogate whatever's unscoped: problem statement, acceptance criteria, explicit
   out-of-scope, priority-with-a-reason, labels, dependencies.
4. Right-size it — one ticket, a linked split, or an epic — judged by number of
   distinct *outcomes*, not effort. A single feature that touches a migration, two
   endpoints, and some UI is still one (big) ticket; several independently-shippable
   pieces get split and linked (`blocks`/`relatedTo`).
5. Place it in an existing Linear project or propose a new one — no ticket ships
   project-less.
6. Draft it, show it, get an explicit yes, create it.

Seven hard gates sit behind step 6 — duplicate search actually ran, every
interrogation item has a real answer, an explicit yes came *after* seeing the
draft — and none of them are satisfied by "the conversation implied it." The draft
itself gets rendered as a reviewable HTML artifact (via a separate `lavish` skill)
rather than pasted as markdown, because a ticket the user is about to bless is a
sign-off, not a status update.

## corpus-work: status *is* the resume point

`corpus-work` takes a scoped ticket and drives it to a mergeable PR, one phase per
invocation:

```
Todo/Backlog          → Phase 0: Planning
Testing Scenarios      → Phase 1: Acceptance tests (Playwright or integration)
Initial Implementation → Phase 2: Implementation, drive suite to green
Human Revision          → Phase 3: Human QA + fixes
Impl. Documents         → Phase 4: Fold into permanent docs
In Review               → feature-branch → main PR, awaiting merge
```

The root ticket's Linear status *is* the save file — every invocation reads it,
finds the child ticket in flight, and picks up exactly where the last run left
off. That's what makes "let's work on COR-5" and "continue COR-5" the same
command.

The one-phase-per-run rule exists for a boring, load-bearing reason: Corpus has no
CI and no branch protection, so a human reading a diff is the only gate between
"committed" and "in the feature branch." Phase 2 needs phase 1's tests to be the
*reviewed* version sitting in the branch, not a version the session merely
remembers writing — so every phase stops at an explicit hard gate before pushing,
and racing ahead defeats the entire premise.

A few mechanisms carry that rule through the details:

- **Two-tier logging.** A `## Dev Log` in `docs/handoffs/<TICKET>.md` records every
  branch created, in full detail — feature branch, base, why — because a later
  run parses *that* to know what's in flight. A single Linear comment on the root
  ticket gets rewritten (never reposted) with one short line per phase, deliberately
  lighter, for a human skimming Linear. Same history, two resolutions, on purpose.
- **An escape hatch.** Not every ticket earns four phases — a Bug or a
  DX-only Improvement with no new user-observable surface can skip straight to a
  single PR, but only after an explicit propose-and-confirm, never an inferred
  guess.
- **Inspection mode.** Standard by default; Exhaustive (one sentence of *why* at
  every gate, not just a bare yes) for anything touching auth, money, data
  destruction, or a ticket that's already looped back once.
- **Loop-back, not silent patching.** If driving tests to green in Phase 2 reveals
  the Phase 1 contract itself was wrong, or a human in Phase 3 finds a real gap,
  the root ticket goes back to "Testing Scenarios" for another round — logged as
  a round, not folded quietly into whatever phase found the gap.
- **A PR comment monitor** triages an automated review bot's comments right after
  opening a PR, so that feedback is addressed before the human is asked to look.
- **Docs that outlive the ticket.** Phase 4 hands the whole handoff doc — the "why"
  behind every decision, not just the diff — to a separate `no-mistakes` skill,
  which folds it into two permanent `docs/features/<Feature>-business.md` /
  `-technical.md` files and deletes the scratch handoff. Corpus's own
  `DocumentUpload-technical.md` is what that produces: architecture, an interface
  contract table (every `data-testid` and endpoint shape a test can target), failure
  modes, and a "Durable Decisions" section recording things like *why* a
  `RAGDocument` model got renamed to `RagDocument` mid-feature, tagged back to its
  source ticket (COR-5).

## Where this bites

None of this is portable, and I'm not pretending otherwise. It's fitted to one
repo's specific gap — no CI, no branch protection — and to a team of one where
"the human" and "the person invoking the skill" are the same person. The ceremony
(four child tickets, a handoff doc, a Dev Log at two resolutions) is a real tax,
which is exactly why the escape hatch and inspection-mode dial exist: most of the
overhead is supposed to scale down when the ticket doesn't need it, not apply
uniformly.

The most recent iteration (`e484887`) trimmed the Linear-facing Dev Log down to
one line per phase instead of one per branch, after the fuller version turned out
to be noise nobody actually read on that side. That's the harness harness — I keep
finding rules that were solving a problem the *previous* rule created, and cutting
until only the load-bearing part is left.
