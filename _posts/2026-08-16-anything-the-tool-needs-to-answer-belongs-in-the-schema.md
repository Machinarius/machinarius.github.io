---
layout: post
title: "Anything the Tool Needs to Answer Belongs in the Schema"
date: 2026-08-16
---

My harness had three records claiming to know where a feature stood: the root ticket's Linear status, four Linear child tickets — one per phase — and the repository handoff doc with its prose Dev Log. When they disagreed, and they did, nothing said which one won. The resume point was whichever one the agent happened to read first.

The failure that finally forced the rewrite is dull enough to be convincing. A ticket merged on the 14th. Its documentation phase completed correctly — the handoff was folded into the domain specs and deleted, source-ticket lineage intact. Its four phase children stayed open anyway: one sitting at `In Review`, one at `In Progress`, until I closed them by hand a day after the work merged and nine days after the phase they represented had actually finished.

The cause was a sentence in the skill's own text:

> Moving a child ticket (or the root) to "Done" once its PR is actually merged is a manual action the human does.

Four tickets per feature, each requiring a person to remember, none of which any automation read. Stale by construction. That's the same shape as the [ceremony nobody performs]({% link _posts/2026-08-15-too-much-ceremony-buys-you-none.md %}#the-ceremony-that-survived) — except here the un-performed step wasn't optional politeness, it was the state the next run would route from.

## The process couldn't run the ticket

Both versions of the harness were unavailable at once, and the bind is worth naming because it recurs.

The old one couldn't run it, because the work *was* editing it. Following `/corpus-work` would have created the four phase child tickets the ticket explicitly forbids and written a handoff doc in the exact template the ticket replaces — the process would have violated the deliverable. There was nothing for the phases to bite on either: no domain spec for the harness skills, no test layer for prose instructions, and an acceptance criterion demanding all eight reference files be rewritten in one atomic change. Every phase boundary the pipeline exists to enforce was degenerate.

The new one couldn't run it for the duller reason that it didn't exist yet. It was the deliverable.

So the ticket ran with no harness at all: read everything, put a plan on a review surface, one branch, one PR, and me standing in for every gate that would otherwise have fired on its own. That isn't the escape hatch — the escape hatch is a sanctioned cheap path for small work, and it still runs *through* the machinery. This is the condition a harness hits precisely when it's being changed, and it's the one moment none of what you've built is available to help.

## One record, and only one thing allowed to write it

The fix is a YAML frontmatter block on `docs/handoffs/<ROOT-ID>.md`, and it is the only machine-readable record of where implementation stands: schema version, Linear id, domain, surface, feature branch, phase, round, phase status, active branch, active PR, test baseline commit, final PR.

Routing reads that block and nothing else. Not the Dev Log. Not a branch-name convention. Not a Linear comment, a prose heading, or the Linear status. Linear keeps `Backlog/Todo → In Progress → In Review → Done`, and none of those map to a phase number — the phases live entirely in the repository now, and there are seven of them instead of four, because the old single "implementation" phase split into unit, integration and E2E.

That leaves three authorities with clean edges:

| Question | Ask |
| --- | --- |
| Why does this exist, and what is "done"? | The Linear root ticket |
| Where does implementation stand? | The handoff frontmatter |
| Did a branch, PR, or merge actually happen? | Git and GitHub |

And one rule the whole design rests on: **the checkpoint records intent and the last known position; GitHub decides whether that position is still true.** A recorded PR is never assumed merged.

## Wrap the mechanism in a skill

The plan went out as a review artifact with three open decisions on it. Two came back as I'd recommended. The one that mattered arrived as a freeform annotation:

> Can we ensure that the harness interacts with the handoff with a dedicated skill, and that skill in turn wraps over that mechanical tool?

That turned what had been section 4 of a plan — an "evidence rules" table the agent was supposed to read and apply — into an executable command with a skill in front of it. Three layers, each doing one job:

- **The data.** Typed frontmatter, one file per feature. No second state file beside it.
- **The tool.** `pnpm handoff` — arithmetic over that data plus a GitHub check. `show` returns a `resolution.action` from a closed set: `plan`, `start`, `resume`, `report-and-stop`, `advance`, `blocked`, `stop-ask`. It also returns the exact reference file to open and the exact `set` command to run next.
- **The skill.** `corpus-handoff`, the only agent-facing interface. `corpus-work` never touches the file, and never touches the tool's flags.

The skill is the interesting layer, and it's the one I wouldn't have built unprompted. Its content is almost entirely about *obligation* — the bridge between a typed answer and what a model is allowed to conclude from it. The action table is that bridge in miniature: a mechanical enum on the left, a sentence about judgment on the right.

| `action` | What you must do |
| --- | --- |
| `resume` | Check out that branch and continue. **Never cut a second branch for the same phase.** |
| `report-and-stop` | Report the URL and stop the run. Do not start the next phase, however mergeable the diff looks. |
| `advance` | Run the `set` in `help`, delete the merged branch, go on. |
| `blocked` | Stop and ask. An abandoned PR is always a human decision. |

Plus the line that keeps the layer from decaying into advice: **`show` is not optional and its result is not advisory.** Skipping it and reading the frontmatter yourself reintroduces exactly the assumed-it-merged bug the whole thing replaced.

This is the same ladder as [the enforcement layers]({% link _posts/2026-07-29-your-quality-gate-isnt-configured-until-youve-watched-it-fire.md %}#rank-the-layers-by-what-they-dont-trust), read from the other end. There, the question was which layer can apply a rule without anyone choosing to. Here it's which layer can *answer a question* without anyone choosing to — and the skill exists precisely because the answer still has to be handed to something that reasons.

## The challenge that found the defect

Then the design got pushed on: is the split really "what's next" mechanical, "commit to it" judgment?

Where it held: `show` is pure computation with no side effects and can run any number of times; `set` is the only mutation; the tool never calls `set` itself. A recommendation can sit in front of you indefinitely without the checkpoint moving. The code already enforced that — I just hadn't stated it.

Where my implementation failed its own claim: `next_phase` was **not** mechanical. Phases 2–4 each implement one test layer and are conditional, and which layers a feature runs was decided in phase 0 — as prose, in the Plan section, which the tool cannot read. So for a feature with no integration layer, `show` would have confidently returned `phase: 3` and the skill would have had to overrule it.

Worth being precise about what kind of defect that is, because my first description of it was too strong. **Nothing was failing.** The model has always got this state machine right — reading the plan, noticing a layer wasn't in scope, routing past it. What I found wasn't a bug caught in the act. It was that the correctness was resting on the model reading prose carefully, every run, indefinitely — and I was in the middle of building a tool whose entire premise is that nothing load-bearing should rest on that.

Which is the more useful version of the point anyway. A mechanical answer derived from incomplete data isn't a neutral non-answer — it's a confident wrong one that a human record then has to contradict, which is exactly the two-records-disagree shape I was removing. The door was open. Nobody had walked through it yet, and "not yet" is not a property you can schedule around.

Worse, it doesn't get to pick its moment. A latent gap like that fires in the middle of some larger piece of work — an interruption to something already half-built, which is the most expensive order in which to discover anything. The obvious objection is that closing a door nobody has walked through is premature optimisation. But that objection was always a claim about *price*, not about correctness — and AI has brought that price down massively.

The guard here was one schema field in the handoff's YAML frontmatter — `layers`, the list of test layers this feature actually runs, defaulted at `init`, revisable mid-flight when a layer turns out to be unnecessary — plus the branch in the resolution function that routes past the phases those layers don't cover, a validator rule rejecting any checkpoint parked in a phase its feature never runs, and the tests around all three. Small, but exactly the kind of small that got deferred anyway when it had to compete for the same hands. When building the guard gets cheap enough, "we'll deal with it if it happens" stops being prudence about effort and becomes a bet that the interruption will land somewhere convenient. It's the same shift that [made a 100% coverage gate affordable]({% link _posts/2026-08-05-100-percent-coverage-was-always-right-just-too-expensive.md %}#what-cheap-code-generation-actually-bought): the standard didn't get better, the tax on holding it collapsed.

The fix was to make the layer decision data. `layers` is now a checkpoint field. Routing skips absent layers and says so — `skipping 4 (e2e) — not in this feature's layers` — rather than looking like a miscount, and the validator rejects a checkpoint parked in a phase its feature never runs.

The rule that came out of it went into the skill verbatim:

> Anything the tool needs in order to answer correctly belongs in the schema; anything requiring judgment stays with you.

With the second half enumerated, so it can't quietly migrate: whether to start the phase at all, whether a failure is a wrong test or a wrong contract, whether a layer turned out to be unnecessary, and every human gate.

## What the tool refuses

The schema isn't held up by good intentions. `pnpm handoff validate` runs on every commit through the pre-commit hook, so a malformed checkpoint fails at the commit that introduces it rather than several phases later, when it would be the last thing standing between the branch and the final PR.

The same rules run inside `set`. The tool validates the transition before performing it, so a rejected change leaves the file exactly as it was rather than half-applied:

```
error: that change would leave the checkpoint invalid — nothing was written
issues[1]:
  `phase: 2` with no `test_baseline_commit` — implementation is measured
  against the merged phase-1 specification, so the baseline is pinned
  before phase 2 starts
```

```
  `active_branch` is still set while `phase_status: complete` — clear
  evidence when a phase ends, or a later run resumes a branch that
  already merged
```

Both errors name the rule rather than the field, which is what makes them usable by the thing reading them. And the skill closes the obvious escape route: hand-editing costs you the validator and gains nothing — if a transition seems impossible through the tool, that's a signal the transition is incoherent, not that the tool is in the way.

## Write the tool for the agent; the human reads it anyway

The steer I had to keep repeating while the tool was being built: **its audience is a peer, not a person.** Left alone, an agent writing a CLI writes it for a human — progress lines in stdout, help text where content should be, a nicely aligned table, an error phrased as an apology. Those are the conventions in every CLI it has ever read, so they're the default, and saying "this is agent-facing" once doesn't hold it. It has to be said again at the next design decision.

What changes when the reader is an agent is specific enough to have a standard, and the tool follows it: [AXI](https://axi.md/), the Agent eXperience Interface. Output is TOON rather than JSON — the standard puts that at roughly 40% fewer tokens for the same content. Errors go to *stdout*, in the same shape as success, because an agent doesn't read stderr and an error it can't see is an error it retries blindly. Exit codes mean something: 0 including no-ops, 1 error, 2 usage. Nothing prompts interactively. And every response carries its own next step:

```
resolution:
  action: advance
  phase: 1
  reference: .agents/skills/corpus-work/references/phase-1-tests.md
  reason: phase 0 is complete — route to phase 1
help[1]:
  Run `pnpm handoff set COR-999 --phase 1 --status not_started`
```

That `help` line is the entire ergonomic argument in one row. The expensive token cost is rarely a longer response — it's the follow-up call. So the command the agent is about to have to assemble gets handed over already assembled.

Then the asymmetry that makes this a free choice rather than a trade: **I can read that too.** It's English tokens in a shape a person parses fine on sight, arguably better than the aligned table I'd have asked for, since nothing is hidden behind a column width. The reverse does not hold. An agent handed human-shaped output reads `Fetching data…` as data, can't distinguish a truncated list from a complete one, and never sees the error on stderr at all.

So the two audiences aren't symmetric, and the tie goes to the agent every time. Optimising for the human costs the agent real capability; optimising for the agent costs the human some polish. That isn't a close call — it's just a call the default pulls against, which is why it has to be made out loud and then made again.

## Prose was already working. That's the argument, not the objection

The thing I want to be careful about claiming: the harness was already behaving well. Tickets in Linear with real acceptance criteria, phase instructions written as prose, [a gate that costs you a review round rather than a merge]({% link _posts/2026-07-29-a-hard-gate-is-expensive-exactly-once.md %}#the-gate-you-cant-skip) — that combination has been producing work I'd defend for weeks. This isn't a rescue.

It's the same designs, moved down a layer. Every rule in that frontmatter schema was already written somewhere in the skill's prose: don't start phase 2 without a merged phase-1 baseline, don't leave a branch recorded once a phase completes, don't cut a second branch for a phase that already has one, don't assume a recorded PR merged. Prose stated them. A model followed them, mostly, and I couldn't tell the difference between "followed" and "happened not to be tested" from the outside. Now the transition that would violate one of them exits non-zero and writes nothing.

Two properties change when a rule moves from a paragraph to a validator. It stops being negotiable — there's no reading of `phase: 2` with a null baseline that gets through, and no deadline under which it becomes reasonable. And it stops being *distributed*: eight reference files each restating a fragment of the state machine becomes one schema, one resolution function, and one skill that says what each answer obliges. The failure I opened this post with was a coordination failure between three records. Centralizing was never about tidiness — a single record is the only kind that can't disagree with itself.

There's a third property, and it's the one that decides whether any of this was worth building. To *trust* a guarantee written in prose, I'd have to evaluate whether the model follows it — across phrasings, across context lengths, across the runs where it's mid-task and reasoning about something else. That's an open-ended behavioural eval. It's expensive to build, expensive to run, and it goes stale the moment either the prose or the model changes.

Putting the guarantee in a validator collapses that surface to a single question: **does the agent actually call the skill and the tool?** Everything downstream of the invocation is arithmetic, and arithmetic is covered by 36 unit tests with no model in them at all. The remaining eval is binary, cheap, and observable in a transcript rather than inferred from the quality of the output.

That's also the honest statement of what's still unguarded, and it lands exactly where [the enforcement ladder]({% link _posts/2026-07-29-your-quality-gate-isnt-configured-until-youve-watched-it-fire.md %}#rank-the-layers-by-what-they-dont-trust) already put skills: triggering on model-judged intent, the probabilistic layer. I haven't removed the probability. I've concentrated all of it into one event I can watch, which is a strictly better place for it than distributed across every rule in eight reference files.

What I'd resist calling it is a railroad, at least not yet. The tool can refuse an incoherent transition; it can't make anyone run `show` in the first place, and it has no view on whether the work in the branch is any good. It narrows the space of states the agent can leave behind. Everything the model does *inside* a phase is still governed by prose, review, and the same [semantic residue]({% link _posts/2026-07-29-a-hard-gate-is-expensive-exactly-once.md %}) a human has to read. The gate got harder; the judgment didn't move.

## Where I'd temper this

Nothing here has been exercised. The change can't dogfood itself — running the pipeline on it would have created the tickets it forbids — so the substitute was walking the finished instructions against ten forward-test scenarios, six of them driven through the real tool and four confirmed only as instructions. That's a walkthrough, not a run. The first real exercise is the next ticket, and the [rubric argument]({% link _posts/2026-07-30-the-tests-written-without-the-rubric-are-its-test-set.md %}#rule-9-aimed-one-level-up) applies unchanged: a mechanism nobody has seen fire is not known to work.

The one real gap in this whole session wasn't found by the validator, the 36 tests, or the four green gates — none of which would have gone red, since nothing was actually failing. It was found by someone reading a plan and asking whether the boundary I'd drawn was the one I'd actually built. A gate can catch a rule being broken. It cannot tell you a rule was never mechanised in the first place.

The tool tells you what's next. It still can't tell you that what's next was computed from a field you forgot to give it.
