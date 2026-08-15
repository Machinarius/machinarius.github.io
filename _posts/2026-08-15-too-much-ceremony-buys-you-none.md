---
layout: post
title: "Too Much Ceremony Buys You None"
date: 2026-08-15
---

Everything I've written here for the last three weeks argues one direction: gate it, write the rule down, make the mechanism apply itself. This is the counterweight, and it's the failure I actually hit rather than one I'm warning about hypothetically.

There are changes in my repos with no ticket, no scoping artifact, and no phase PR. Not because the harness broke — it works fine. Because I didn't invoke it.

The part worth writing about isn't that I skipped it. It's that the skipping is *binary*. I don't run an abbreviated version — a thin ticket, three of the eight sections, a scoping phase whose artifact I drop. I run none of it, and then I run all of it again on the next thing that clears the bar. The usual diagnosis for that pattern is discipline, and I think the usual diagnosis is wrong.

I also thought I knew which work was falling through, and I was wrong about that too. I counted, about two thirds of the way through writing this, and the count is in the middle of the post rather than at the end because it changed what the rest of it says.

## A process with one mode is a step function

Ceremony cost is roughly fixed. Task value varies over three or four orders of magnitude. Put a fixed cost against a variable payoff and you get a crossover point, and below the crossover the rational move is not to pay less — there is no *less* — it's to pay nothing.

That alone explains a gap. It doesn't explain the shape of the gap, and the shape is the interesting part.

A pure cost calculation predicts a gradient, and my harness is built for one. It has [a named escape hatch]({% link _posts/2026-07-24-corpus-ticket-and-corpus-work-under-the-hood.md %}): a bug or a DX-only improvement with no new user-observable surface skips the four phases and goes straight to a single PR, after an explicit propose-and-confirm. There was a two-tier inspection dial next to it, until I deleted it — which turns out to be the best single piece of evidence in this post, so I'll come back to it.

The hatch is real and it gets used. And most of what I commit still goes around it entirely. That's the more useful version of the observation, because it rules out the obvious fix: **the tier existed and I bypassed it anyway.** Three things explain that:

**The discount is on the wrong axis.** The escape hatch cuts four phases to one, and phases are mostly the agent's work. What it doesn't cut is the cost of *entering*: a ticket has to exist, the scope has to be written, the propose-and-confirm is an exchange I have to have, and there's still a handoff doc and a PR at the end. That's a large discount on the cheap half and almost none on the expensive half. The crossover is set by entry cost, and entry cost barely moved.

**Nothing marks which parts are load-bearing.** Every section of a ticket template looks deliberate, because it is — each one got added by someone solving a real problem. Which of them earn their place on *this* ticket is a different question, the answer changes from ticket to ticket, and nobody could have marked it in advance. So the trimming has to happen at write time, by the person least inclined to do it. "Just fill in the useful parts" isn't an option you can execute; it's an option you'd have to design.

**Partial compliance feels like a lie. Total avoidance feels like a decision.** This is the asymmetry that makes the edge sharp. A ticket with half its sections empty is a visible failure to do the thing properly, sitting in a tracker with my name on it. No ticket at all is just a small change that happened. One of those is embarrassing and the other is invisible, and they're both the same underlying choice. Nothing about the system makes it easier to be honest at low effort than to be absent at zero.

So the sanctioned cheap path is cheaper than the full process and still dearer than nothing, the operator opts out whole, and the process reports full compliance on everything that reached it.

## Cheap generation moved the crossover the wrong way

Here's the part that's specific to now, and it's why my own harness got harder to justify without changing a line.

Ceremony is a *human*-executed cost. Deciding what the thing is, writing the scope, reviewing what came back. It sits entirely in the bucket that [didn't get cheaper]({% link _posts/2026-08-05-100-percent-coverage-was-always-right-just-too-expensive.md %}) — the reading-and-deciding bucket, the one that got *worse* because there's more output competing for the same attention. Implementation sits in the bucket that collapsed.

Hold ceremony cost fixed at *C* and let build time *B* fall. Ceremony's share of the task is *C / (C + B)*, which does this:

| Build time, as a multiple of ceremony | Ceremony share |
|---|---|
| 10× | 9% |
| 1× | 50% |
| ¼× | 80% |

That table is arithmetic, not a measurement. I haven't timed my own tickets, and I'd rather say so than dress an identity up as data.

The direction doesn't need the data, though. Nothing in the last two years reduced the cost of deciding what a thing is and reviewing what came back; a great deal reduced the cost of building it. So every harness that was proportionate at the top row has been sliding down the table ever since.

Every row is the same harness. **My process didn't get heavier; the work got lighter, which is arithmetically the same thing.** And there's no event for it. No commit moves a design across a row, so nothing ever prompts the re-read.

And the collapse isn't uniform across kinds of work. A feature still has a build phase long enough to keep ceremony's share respectable. Editing a Markdown rule doesn't — the build time for "add a rule to the rubric" is close to zero, so ceremony is essentially the entire cost of doing it properly. Anything whose deliverable is a paragraph sits at the bottom row of that table permanently.

I wrote that as a prediction and then went and checked it, which is the next section, and it's the point where the post stopped being the one I set out to write.

## A process can only measure the work that agreed to be measured

This is the line I'd keep from the whole post.

I review my tickets and my scoping artifacts and I conclude the harness is working. It is — on its sample. The sample is "tasks that cleared the crossover," which is the definition of the tasks the ceremony was cheap relative to. Every metric drawn from a process's own artifacts is computed over the subset that opted in.

That makes the record worse than no record, for the same reason [a rendered view becomes dangerous once it's committed]({% link _posts/2026-07-29-when-a-view-becomes-a-record.md %}): people trust it. A tracker with nothing in it is obviously incomplete. A tracker with forty well-scoped tickets reads as the history of the project, and there's no field in it for the eighty changes that didn't come through.

You can't detect this by asking, including asking yourself — the whole point is that the skipped work didn't feel like skipping. You have to count from the side that can't be avoided, which is the repository. Two queries, and I was confident the first one would be the informative one:

1. **Merged PRs with no ticket reference.** The classic hole.
2. **Mainline commits that never came from a PR at all.**

## So I ran them

The window is 2026-07-11 to 2026-08-15 — the day after the harness skills first landed, to the day I'm writing this. 39 first-parent commits on `main`.

**The first query came back empty.** 37 merged PRs, 36 of them naming a `COR-` ticket. The one exception is a turbo version pin. The metric I'd have led with detects nothing here, and that's worth recording rather than quietly dropping, because [a null result is ambiguous]({% link _posts/2026-07-29-a-hard-gate-is-expensive-exactly-once.md %}) until you know *why* it's null. It's null because PR discipline isn't where this process leaks.

The second query is where everything was.

| Mainline commits, 2026-07-11 → 2026-08-15 | Count |
|---|---|
| Landed through a pull request | 14 |
| Landed straight on `main` — no PR, no ticket, no review | 25 |

Then the split that actually matters, which is *what those commits changed*:

| Area | Via PR | Direct to `main` |
|---|---|---|
| Product code | 4 | 0 |
| The harness itself — skills, rubrics, specs, ticket drafts | 5 | 25 |

Product code has a spotless record. Every feature went through the four phases or the hatch, every PR named its ticket, and not one line of application code has reached `main` unreviewed since the harness existed. The single product file in the direct column is a test file that rode along inside a harness commit. The gate works exactly as advertised.

Everything that went around it is the harness. Ten of those commits edit `.claude/skills/` — the files that *define what the phases are*. Three edit `docs/rubrics/`, including the commit that first added the [testing rubrics]({% link _posts/2026-07-29-a-hard-gate-is-expensive-exactly-once.md %}) I'd just spent three posts on.

**The process governs the product completely, and does not govern itself.**

I had this backwards for the entire first draft. I assumed the leak was small work, and that the fix was a cheaper tier. Small work isn't the category. *Rules* are the category.

### The receipt

The sharpest one isn't an addition. On 2026-08-09, direct to `main`:

> `chore(skills): drop inspection mode from corpus-work and corpus-ticket`
>
> Standard/Exhaustive added a why-trail requirement at every hard gate, all of which fire before the PR is reviewable — ceremony that cost context without changing what got reviewed.

That commit deletes a governance tier out of both skill files. It's why the inspection dial I described in the harness posts isn't in the list of tiers above: it's gone. I still think the reasoning is correct — it's the argument this whole post is making, in a commit body. But a change that *removes a review requirement* landed with no ticket, no PR, and no reviewer, and the justification I just quoted approvingly is one I wrote and nobody else has read until now.

Which is precisely the failure the [hard-gate post]({% link _posts/2026-07-29-a-hard-gate-is-expensive-exactly-once.md %}) named and that I was most confident I'd avoid: **a wrong rule amortizes a wrong standard.** That post's own closing move was that rules need the same gate the tests got. Five weeks of commits say they never got it — and not because I changed my mind. Because the only gate on offer was priced for a feature.

## The fix that makes it worse

The instinct on seeing that ratio is to close the hole. A commit hook that rejects a message with no ticket ID. Branch protection requiring a linked issue. Required fields that won't let you save.

That converts avoidance into fake compliance, and fake compliance is strictly worse.

Pre-2023 it was at least *weakly* self-limiting: filling in eight sections of boilerplate to land a typo fix cost enough that most people would either do it properly or give up loudly. The cost was a bad proxy for care, but it was a proxy. That proxy is now gone. A model will produce a fluent, well-structured, entirely plausible eight-section ticket for a one-line change in about four seconds, and it will do it every time without ever getting annoyed enough to tell you the format is wrong.

Same shape as [the beautiful ignore-comment reason]({% link _posts/2026-08-05-100-percent-coverage-was-always-right-just-too-expensive.md %}), one level up. So:

> **Anything you mandate that a model can produce, you have mandated nothing.** You've mandated the string, and the string was never the point.

And you've paid for it by destroying the one signal you had. Bypass leaves a hole you can see. A mandate leaves a filled field you'll believe — present, readable, well-formatted, and doing nothing, which is the [inert-is-worse-than-broken]({% link _posts/2026-07-29-a-hard-gate-is-expensive-exactly-once.md %}) failure with a compliance rate attached. The metric goes to 100% and the information content goes to zero, and those two things happen in the same commit.

## Who executes the step, and which budget does it draw on

The useful audit question isn't "is this ceremony valuable." Everything in a template is valuable to someone. It's:

| Executed by | Marginal cost | Real constraint |
|---|---|---|
| A machine, on an event | ~zero | correctness of the rule |
| An agent, in-loop | near zero | the [context tax]({% link _posts/2026-07-29-a-hard-gate-is-expensive-exactly-once.md %}) — re-read every run, forever |
| A human, on their own initiative | high | attention, which is the scarce one |

Harnesses grow downward through that table. Adding a field to a human-facing template is the single easiest change to make and the only one that requires no implementation, so it's where accumulation lands by default. Each addition is individually defensible and individually small. The sum of them is the crossover point, and no single commit ever moved it.

The corollary is more optimistic than it sounds: a lot of what's in my human-facing ceremony is there because, when I wrote it, a human was the only thing that could execute it. That's a stale assumption in most cases now. Moving a step from row three to row two isn't a reduction in rigour, it's a change of payer — and it's the only kind of ceremony reduction that doesn't cost you the ceremony's function.

## Make the tier the artifact

The wrong response to "the step is too high" is to lower it uniformly. That just relocates the crossover and loses the rigour on the work that warranted it.

More steps is the right shape, and I want to be careful here, because I already had a second step and it didn't work. So the shape isn't the hard part. Tiers only work under one constraint, and it's the one my own escape hatch misses:

**The cheapest sanctioned path has to be cheaper than bypassing it.** If bypass is `git commit -m`, the small tier is one line in the commit message — not a short form, not a "lightweight" template with four fields. Anything heavier than the bypass is a tier that will never be used, and you've added documentation instead of a path.

Measured that way, my escape hatch isn't a small tier at all. It's the full entry cost with the phases removed, which prices it against the four-phase path rather than against `git commit` — **a discount computed relative to the wrong alternative.** That's the mistake I'd expect to be common, because from inside a process the interesting variation is all at the top end. The tier a process actually needs is the one immediately above doing nothing, and it's the one nobody designs.

Then borrow the move that makes the coverage gate honest — [three exits, and the diff won't merge until you've taken one]({% link _posts/2026-08-05-100-percent-coverage-was-always-right-just-too-expensive.md %}). The tier choice itself becomes the recorded decision. Not "did you follow the process," which is unanswerable and, once mandated, dishonest — but "which mode did you pick," which is one greppable word that someone can disagree with six weeks later. A wrong tier is reviewable. An absent process isn't there to review.

The honest catch: tier boundaries drift downward. Everything becomes small, and you've built bypass with a permission slip. The countermeasure is that **the distribution is the metric, not the compliance rate.** My own numbers make the case: 36 of 37 PRs cited a ticket, which is a compliance rate of 97% and tells you nothing, while the distribution says two thirds of my commits never entered the system at all.

And the count says something about *which* tier to build, which I would have got wrong from the armchair. I'd have designed a tier for small changes, sorted by size. The work falling through isn't sorted by size — it's sorted by kind, and the kind is rule changes. A rubric line and a feature are not the same object and shouldn't share a pipeline: one needs acceptance tests and a running-feature review, the other needs someone to argue with the rule before it starts applying itself to everything downstream. What a rule change actually wants is [the thing that made the rules trustworthy in the first place]({% link _posts/2026-07-29-a-hard-gate-is-expensive-exactly-once.md %}) — planting the violation and watching it fire — which is cheap, mechanical, and has nothing to do with four phases.

## The ceremony that survived

Sorting my own harness by what I still actually do, the split isn't by value. Some of what I skip is more valuable than some of what I keep. It's by *who initiates*.

The things that survived — dependency-cruiser, the architecture test, types, lint, the coverage threshold, the phase PR that won't let phase 2 start — all fire on their own, at an event, whether or not I remembered them. The things that eroded are the ones where the first step is me deciding to begin. That's the same three-tier table from [the hard-gate post]({% link _posts/2026-07-29-a-hard-gate-is-expensive-exactly-once.md %}), read from the failure side: prose written for humans "needs discipline," and discipline turns out to be a budget I've spent by lunchtime.

So the rule I'd put next to the other ones:

> **Ceremony you have to remember to perform will be performed exactly as long as the work is expensive enough to justify it, and not one task longer.** The only durable ceremony is the kind that happens to you.

Which reframes what [configuring a quality gate]({% link _posts/2026-07-29-your-quality-gate-isnt-configured-until-youve-watched-it-fire.md %}) is even for. I'd been treating "find a mechanism that applies this without anyone choosing to" as an anti-forgetting measure. It isn't. It's an anti-*economics* measure. Forgetting is random; opting out is systematic, it concentrates on one category, and it leaves a record that looks complete.

There's a structural reason the harness ended up as the uncovered category, and it isn't only price. Every gate I have is bound to an event in the product's lifecycle — a commit, a push, a PR, a merge. Changing a rule isn't an event in that lifecycle. It's an event in the lifecycle of the thing that watches the lifecycle, and I never built a clock for that one. **The layer that governs is the layer with no events in it**, which is why it ends up ungoverned by default rather than by decision.

## Limits

I'm one person, and ceremony has a function for teams that it doesn't have for me — consent, coordination, and giving people who weren't in the room a way to object. Bypassing in a team also has a different cost: someone else finds out from the diff. Read the tiering advice above with that in mind; the crossover point moves when the record's audience isn't just you.

There are also contexts where the ceremony *is* the deliverable — regulated work, anything with an audit obligation — and "nobody reads it" isn't a defect in the process, it's the process functioning. None of this applies there.

The count is one repo, one person, five weeks, and 39 commits. It's enough to falsify my guess about *which* work leaks — that part is unambiguous — and nowhere near enough to establish that rule changes are the leaking category anywhere but here.

And the obvious one: this is a post in which I describe my own avoidance and then explain why it was rational, which is precisely what someone being lazy would write. The check that would settle it is whether the un-ceremonied work caused problems the process would have caught. The one piece of evidence I have points the wrong way for me — the [three consecutive commits that each reversed the rule in the commit before]({% link _posts/2026-07-29-a-hard-gate-is-expensive-exactly-once.md %}) were exactly this category, landing exactly this way. Three reversals is a small sample and I found them by reading, not by measuring, so I'd rather call it a warning than a result. What I can't do is claim nothing went wrong, because I never had anything watching.

So the fix I'd defend is narrower than the essay and it's specific: whatever governs your work needs a gate of its own, priced for a paragraph rather than a feature, fired by an event that exists in the rules' lifecycle rather than the product's. Everything above is the argument for why nobody builds that one — a process you can't afford isn't a lighter process, it's a process you don't have plus the belief that you do, and the belief is the expensive half. Mine cost me five weeks of unreviewed rules while I wrote three posts about how well the gates were working.
