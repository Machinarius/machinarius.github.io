---
layout: post
title: "Two Styles of Agentic SDLC Harness: Gated vs. Fluid"
date: 2026-07-24
---

I've spent the last few weeks iterating on a homegrown SDLC harness for an agent
(`corpus-ticket` + `corpus-work`, a pair of Claude Code skills) and comparing notes
against [OpenSpec](https://github.com/Fission-AI/OpenSpec), a maintained,
portable spec-driven-development tool. They solve the same underlying problem —
"AI coding without an agreed plan is chat-history archaeology" — but they land on
opposite answers for how much to enforce.

## The gated style

My harness splits a feature into four sequential phases — acceptance tests,
implementation, human revision, docs — each ending in its own GitHub pull
request. The reason is mundane and specific: the repo has no CI and no branch
protection, so a human actually reading a diff is the *only* thing standing
between "committed" and "merged." Racing ahead to the next phase before that
review happens defeats the point of having phases at all — phase 2 needs
phase 1's tests to be the reviewed version sitting in the branch, not a version
the session merely remembers writing.

A few mechanisms fall out of that one constraint:

- **Tests-first as a real contract.** Acceptance tests are their own reviewed PR,
  red until implementation exists. Driving them to green is what "done" means,
  not a checklist someone can talk themselves out of.
- **A phase no diff can replace.** One phase exists purely to put a human in
  front of the *running* feature — copy quality, interaction feel, edge cases
  nobody wrote a test for. Code review can't see any of that.
- **An audit trail that survives reverted attempts.** A running "Dev Log" and an
  append-only handoff doc record not just what shipped, but what was tried and
  walked back, and why — the kind of thing that normally evaporates the moment
  you `git revert` it.
- **Ceremony that scales down on purpose.** A named escape hatch for
  diagnose-and-fix work, and a two-tier inspection mode, so a one-line bug fix
  doesn't pay the same toll as a new feature.

The cost is exactly what you'd expect: this only works because it's fitted to
one team's stack (issue tracker, PR flow, workspace tooling, one repo). It isn't
going anywhere else without a rewrite.

## The fluid style

OpenSpec's own words for its philosophy: **"enablers, not gates."** A change
folder builds `proposal → specs → design → tasks`, but that arrow describes
what becomes *possible* next, not what you're forced to do next. Discover the
design was wrong mid-implementation? Edit `design.md` and keep going. Nothing
locks. The human review is front-loaded — read the whole artifact bundle once,
before `/opsx:apply` — rather than gating every subsequent diff.

Its standout idea is **delta specs**: a change doesn't rewrite a domain's spec,
it writes `ADDED`/`MODIFIED`/`REMOVED` blocks against it, and those deltas merge
into a permanent, per-domain spec on archive. That's a real answer to the
brownfield problem my harness doesn't have — a living, queryable "what does
this system guarantee today" that a future planning session can read without
archaeology through old tickets. In exchange, OpenSpec is a generic,
installable CLI that works across 25+ editors and any stack — it trades
per-repo depth for portability.

## The actual axis

It's tempting to call one of these "better," but the difference isn't quality —
it's where each one spends its trust. The gated style doesn't trust an unreviewed
diff, so it forces a human checkpoint at every phase boundary and treats
"agreed but not yet reviewed" as an unfinished state. The fluid style trusts the
human to keep steering after one upfront agreement, and optimizes for how
cheaply that agreement can be revised. Neither is wrong; they're just answers
to different failure modes — a bad merge with no safety net, versus a stale
plan nobody bothers to update.

## Stealing the good idea without the philosophy

The interesting move isn't picking a side — it's grafting OpenSpec's one
structurally superior piece onto the gated harness without diluting the gates
themselves. Concretely: keep every PR gate, keep the tests-first contract, keep
the human-exercises-it phase — and add a `docs/specs/<domain>/spec.md` that
acceptance tests write deltas against instead of one-off narrative docs. The
docs-handoff phase — which already folds a feature's history into permanent
docs and deletes the scratch copy — is a natural place to also merge that
delta into the living spec, the same way OpenSpec's archive step does. Same
audit discipline, same enforcement, one more source of truth that outlives a
single feature.

## Where I've landed

I'm keeping the homegrown harness where it lives — a real project, with real
review gates, is where its maintenance cost actually pays for itself, and
building it has taught me more about which mechanisms matter than adopting a
tool would have. But I'm not planning to rebuild it from scratch on the next
project either. The plan is to run both, on different projects, on purpose:
the homegrown one where control and depth matter, OpenSpec (or something
thinner) where portability or a teammate's onboarding cost matters more — and
to deliberately compare notes between them, the way this post did, rather than
let two separate muscle memories form in isolation.
