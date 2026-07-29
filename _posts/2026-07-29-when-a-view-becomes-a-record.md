---
layout: post
title: "When a View Becomes a Record"
date: 2026-07-29
---

A skill in my harness ends by rendering an HTML artifact — a human-readable representation of what it just scoped — and then committing it into the repo. That commit is where the trouble started. The skill's instructions said to commit "on whatever branch is currently checked out," and a ticket-filing session has no idea what the working tree is doing. So a ticket's scoping artifact landed on an unrelated in-flight feature branch, and I had to rebase it back out.

The obvious fix is to pick a better branch. That's the fix I spent an hour not making, and I'm glad I didn't, because the branch was never the question.

## Three homes, none of them right

Once you accept the artifact must be committed, there are only three places it can go: `main` directly, a dedicated `docs/<id>-ticket-draft` branch, or the ticket's eventual feature branch. I rejected `main` on instinct. The AI built the docs-branch version. Then I rejected the premise underneath both remaining options: a skill whose entire stated job ends at *"the ticket is filed"* should not be creating branches at all. Cutting a feature branch is the first step of building a feature, not of scoping one.

Which surfaced the real tension. If the skill doesn't commit, the artifact sits untracked from the moment the ticket is filed until someone actually starts the work — possibly weeks, possibly never, one `git clean` from gone. The skill's own instructions defended committing for exactly that reason:

> It's a real record of what was scoped and approved, not scratch to end the session with — this skill has no Phase 4 of its own to fold it into later, so it just stays.

So: is an untracked artifact acceptable in exchange for the skill never touching git? Yes or no, and everything else follows.

## The repo had already answered

The answer was in the commit log. A commit on `main` from the day before, `chore: remove spent Lavish scoping artifacts`, had deleted two of these files with this reasoning:

> Per-ticket Lavish artifacts are **scoping surfaces, not durable docs** … Both of these have outlived their work and their content lives in `docs/`.

The repo had already ruled these things ephemeral. The skill's *"it just stays"* sentence had been contradicting a decision that was already made, and the draft commit I'd been reviewing argued the opposite of it — citing one of the very artifacts that had been deleted as precedent for artifacts living on `main`. It wasn't wrong about the branch. It was wrong on the merits.

That's the part worth generalizing. Two instructions in the same repo, both plausible in isolation, silently contradicting each other. Nothing catches that except reading the log before writing the fix.

## Naming the tiers

The rule I landed on isn't about artifacts at all:

- **Linear is the source of truth for planning.**
- **OpenSpec deltas are the archive of what actually got implemented.**
- **Lavish artifacts are human-friendly representations of those two, and own nothing.**

From which one line does all the work:

> A Lavish artifact persists exactly as long as the skill lifecycle that needs it as a review surface, and no longer.

Note what that buys over "don't commit ticket artifacts." My two skills behave *differently* — `corpus-ticket` never commits its artifact; `corpus-work` commits its Phase 0 plan artifact on the feature branch and deletes it in Phase 4. Stated as two rules, that reads as an inconsistency, and the next person to tidy up will fix the wrong one. Stated as one principle, both are derivable: the filing lifecycle ends at filing, so the artifact dies there. The build lifecycle spans five phases across multiple sessions, so committing is the only way the artifact survives *within* one lifecycle — that's continuity, not durability, and Phase 4's deletion is the lifecycle ending on schedule.

|  | ticket artifact | Phase 0 plan artifact |
|---|---|---|
| Committed? | no | yes, on the feature branch |
| Lifecycle | ends at filing | folded and deleted in Phase 4 |
| Touches git? | never | yes |

The principle also overrules something I'd have kept. The earlier cleanup commit had spared one artifact because its tickets were still in Backlog — "delete when spent" rather than "never durable." Under the stated rule that exception doesn't hold: the *filing* lifecycle ended regardless of what the tickets are doing. Worth writing the overrule down explicitly, so it doesn't read later as an oversight.

## The ignore pattern is load-bearing

One consequence I initially under-called. With the skill no longer committing, a leftover artifact shows up as untracked in `git status` — and I described that as "noise." It isn't noise. My build skill's Step 0 says:

> Run `git status --porcelain`; if it isn't empty, stop and ask.

An untracked artifact therefore **blocks the next run of that skill, for any ticket**, until someone deletes it by hand. The `.gitignore` entry isn't tidiness, it's what keeps the harness startable. And the pattern has to be surgical: it must not swallow the Phase 0 plan artifact, which has to stay committable. A subdirectory for ephemeral drafts beats a negation pattern with a growing list of exceptions — the negation pattern is always one edit away from ignoring something load-bearing.

There's a sharper version of the classification test, too. Not "is this a reference doc?" but *"was this rendered as the review surface for a skill invocation?"* My two skill explainers live in the same directory and look superficially identical to the ticket drafts. Under the weaker test they're an exception to the rule. Under the sharper one they simply fall outside it, because nobody invoked a skill to produce them. Rules with exceptions rot. Tests that partition cleanly don't.

## What this is really about

Every one of these artifacts is a *view*. The failure mode — the branch collision, the duplicated content, the contradicting instructions — came from letting a view quietly become a record. The commit I ultimately threw away was doing exactly that: arguing that what mattered was that the artifact "exists somewhere durable," when the thing it represented was already durable somewhere better.

The artifact was, by the skill's own instructions, required to match the Linear description **verbatim**. So the committed file was by construction a second copy of something Linear already held. That's not a principle, that's just a fact — and it's the cheapest possible argument for the rule. I'd spent an hour on the principled version first.
