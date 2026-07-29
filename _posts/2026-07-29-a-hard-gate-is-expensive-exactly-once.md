---
layout: post
title: "A Hard Gate Is Expensive Exactly Once"
date: 2026-07-29
---

I built a feature this week without opening an editor. A terminal and GitHub's PR review UI, nothing else. It was liberating, and I want to be careful about how I say that, because "I don't read the code in an editor" sounds like negligence.

It isn't, and the reason is a division of labour that took a whole session to see clearly. Structure is checked by dependency-cruiser and an architecture test. Types and lint are checked by their own tools. What's left for a human is *semantics*: does this assertion distinguish the two behaviours it needs to, is this the copy a user should actually see, is a bare `ZodError` a 400 or a 500. That residue is small enough to read in a PR diff — and it's the only part where my judgment was ever the bottleneck anyway.

But the residue is only small because a lot of it got moved. This is the story of moving it, which cost seven review rounds, and of why I think that's the correct price to have paid exactly once.

## The gate you can't skip

My build harness runs one phase per invocation, each ending in a pull request I have to merge before the next phase can start — the gated style I described in [Two Styles of Agentic SDLC Harness]({% link _posts/2026-07-24-agentic-harness-styles.md %}). Phase 1 writes tests. Phase 2 implements them. That ordering isn't a convention I'm trying to live up to; it's the shape of the tool.

Which quietly solves the thing TDD always lost on. TDD never failed on merit — it failed because nothing *stops* you from writing the tests afterward, so under deadline you don't. Here there's nothing to resist. The agent has no ego about test-first, and it cannot peek at an implementation that doesn't exist yet.

Better, the Phase 1 PR is *supposed* to be red. Mine opened at 25 failing and 44 green, with a single cause: `not implemented yet`. A green tests-phase PR would have meant the new assertions asserted nothing new. Red-by-design is the receipt — you can't fake the ordering.

The gate also gives me somewhere to stand. Over seven rounds I reversed five of my own earlier decisions and the agent retracted four of its claims. None of that survives a fluid harness, where there's no natural moment to say "not yet" without feeling like you're obstructing your own tooling.

## The ladder

Everything below came out of reading assertions in a diff. No tool found any of it.

**A mock that echoed its input.** Cleanup was asserted with `toHaveBeenCalledTimes(2)`. The storage mock echoed back the key it was handed — exactly like the real gateway does — so two genuinely different implementations produced identical calls: using the key that staging *returned*, versus re-deriving one locally from `randomUUID()`. The second leaves real orphaned files in storage forever. The count passed either way. Making the mock return `uploaded-${key}` and asserting that prefix separated them.

**The same test again, harder.** A prefix only proves the deleted keys were staged keys, not *which* ones. So: mock `randomUUID` to a counter, make every key predictable, assert literally.

```ts
expect(deletedKeys()).toEqual([stagedKeyFor(1), stagedKeyFor(3)]);
```

That immediately exposed a wrong assertion. In the staging-failure case `a.txt` and `c.txt` succeed while `b.txt` fails, so **two** keys need cleaning up — and the count-based assertion had expected one. It had sailed through because the suite was already red for an unrelated reason.

Sit with that one, because it's the sharpest thing in the session. **A red-by-design phase hides wrong assertions inside expected failure.** The property that makes the gate honest is the same property that makes a bad assertion invisible to everything except a human reading it.

Worth noticing too that the expected keys are 1 and 3, not 1 and 2 — the failing file still consumes a uuid, because the key is generated before storage is ever touched. That asymmetry now lives in the test instead of being buried in a number.

**Assertions no string could fail.** A helper called `expectSafeFailureBody` checked five regexes for what an error message *wasn't* — no model name, no storage key, no Prisma error code, no env var name — plus a non-empty check. Any string satisfied all six. It never said what a user should see. Replaced with the literal copy per status:

| Status | Copy |
|---|---|
| 409 | `That record already exists.` |
| 503 | `The service is temporarily unavailable. Please try again.` |
| 500 | `Something went wrong. Please try again.` |

An exact match subsumes every leak regex — a body equal to one of those three strings cannot contain a Prisma code. Two things fell out of this that I'd keep as rules anywhere.

First: the literals are written into each test file rather than imported from the middleware, because importing them makes every assertion agree with whatever the code says, including copy someone changed by accident. **A test that can't disagree with the implementation isn't testing it.**

Second: strengthening these took the suite from 26 red to 24. The leak-pattern assertions and a separate response-shape test all collapsed into one `toEqual` on the whole body. Raising the bar *deleted* test code. That surprised me, and it's a useful counter to the assumption that a stricter standard means more to maintain.

**Fabricated premises.** This one took me two attempts to articulate, and the sentence I landed on is the one I'd keep:

> Even though the error does have a public constructor, building it manually bakes in a presumption that X failure from PG becomes Y code from Prisma. That's what fabricated means.

A hand-constructed `PrismaClientKnownRequestError` doesn't test that we map Prisma's behaviour. It tests that we map *my belief about* Prisma's behaviour, and it will keep passing after that belief goes stale. Same flaw in different clothing: `as unknown as Context` in the middleware tests — a fake that silently encodes which `Context` members the handler happens to touch today.

The fix for that one is instructive because the obvious destination was wrong. Moving those cases to the HTTP-level test file would have made them real, but that tier is excluded from `pnpm test` and needs a listening service, so it **never runs in CI**. Real-but-never-executed is the worse trade. Hono's `app.request()` was the third option: real dispatch, real `Context`, real `onError`, real `Response`, in-process, no port, still hermetic.

The confirmation was accidental and pleasing. After the switch the failure message reads `(reached from GET /api/health)` — values that now come from Hono's own dispatch rather than a literal I typed into a fake.

Same principle produced the CI fixture. Instead of an inline `throw` — a throw wearing a route's clothes, where the error never travels the path a real one would — there's a real route table calling the real env loader and a real Zod `.parse()`, never mounted anywhere. Nothing in a running service can reach it, which makes it strictly safer than a flag-guarded debug endpoint somebody can misconfigure.

And the `ZodError` case earned its own place, because `.message` is a JSON dump carrying field paths and received values — `["collections", 0, "name"]`, `expected string, received number`. A handler passing that through leaks the shape of internal data structures *and* returns a blob where a sentence belongs. Two more rules came out of building it: use a real failed parse rather than a constructed error, and make the fixture guard itself — it asserts that the `ZodError`'s own message still contains the field path, so if zod stops putting paths there the test fails loudly instead of passing while proving nothing.

The interesting assertion there is 500, not 400. Request validation (the client's mistake) and parsing an upstream's response (not the client's mistake) produce the identical error class, so the boundary must not guess between them: a bare one falling through to a generic 500 is the correct conservative default, and a typed wrapper is what earns a 502.

## Noticing is the job; fixing is free

Here is the part that changed how I work. **I didn't write a single one of those assertions.** My inputs were one-sentence discomforts:

- "The mock should return `uploaded-${key}` so we can differentiate errors and logs in tests."
- "I'd rather have hard expectations of keys, not simply a count of regex matches."
- "We should assert hard user-friendly strings, as that is what the middleware is supposed to return."

Each came back implemented in minutes. And note that **not one of them named a bug.** The first was about log legibility. The second was aesthetic. The third was about intent. All three turned out to be covering real defects.

That inverts the economics of diligence. The expensive part of test review used to be *fixing* what you found, which is exactly why you learn to swallow the vague objections — not worth the detour. When noticing is the entire cost, following unarticulated taste is free, so I raised the complaints I'd previously have let go. They paid, every time.

It's also what [steering by asking rather than asserting]({% link _posts/2026-07-26-steering-the-ai-without-steering-yourself-into-a-rubber-stamp.md %}) turns into once the fixing is free. A one-sentence discomfort is the smallest possible version of that move: it points at something without prescribing the answer, so what comes back is an implementation I still have to judge rather than an agreement I asked for.

The skill this rewards is not "navigate the codebase fast." It's "notice precisely, and say it in one sentence." What I've replaced the editor with is prose.

## The bill

Seven rounds. That's not slow because gating is slow — it's slow because every catch above came out of my head at review time, which means the standard for "a good test in this repo" was unwritten and got re-derived by hand at each gate. It doesn't amortize. The next tests phase would start from zero and pay the same seven rounds.

The gate was doing double duty: the place the standard gets *discovered* rather than the place it gets *applied*.

But it wasn't only producing an approved PR. It was producing contract. The three copy strings, the 500-versus-502 boundary, the requirement that staging generate each key before its first await — all of it went into the handoff document, so Phase 2 has to match the tests rather than the tests being bent to fit the implementation. Those are constraints the next phase inherits instead of rediscovering.

That's the mechanism worth naming: careful test review doesn't just filter the next phase, it **specifies** it. The gate's real output is trust in work that hasn't happened yet.

Which suggested the fix for the seven rounds. If the gate already produces contract at ticket scope, do the same move at repo scope — write down what makes a test good here, once.

## Three tiers, not two

My first attempt at that conclusion was half right, and the half that was wrong is the interesting part.

The right half: structure belongs to tooling, and tooling is better at it than I am because it never gets tired at round six. dependency-cruiser and the repo's architecture test earned their keep twice this week. When I asked for a fixture file named `routes.ci.ts`, the architecture test rejected it and told me the *process*, not just the verdict:

> "The concern implies the file" — if this is a new concern, add it to the doc first; if it fits an existing one, rename it.

So the convention got documented before the allowlist widened. Without the check, the allowlist gets quietly widened and the convention is never written down at all. And it widened precisely rather than bluntly: a plain `helpers.ts` in that directory is still rejected. Worth saying, because the lazy response to a failing structural check is to loosen it until it passes, and that's how these rules die.

The wrong half was concluding that everything tooling *can't* express has to stay in my head, applied by hand, forever. dependency-cruiser cannot express *a mock's output must differ from its input*. So I assumed semantic rules were mine to remember.

What showed me otherwise was one sentence I typed while the rubrics were being drafted: **the intended audience isn't a human, it's the agent that writes the next tests.**

That changes the genre completely. Each rule now leads with a mechanical trigger — "you are about to write `toHaveLength`", "your mock returns its input unchanged" — followed by a bad/good code pair and one line of reason. The persuasion is gone. The narrative of how we found it is gone. An agent needs a decidable condition and just enough rationale to generalise, not convincing.

So there are three tiers, and the middle one is the one I didn't know existed when the session started:

| Tier | Applies itself? | Covers |
|---|---|---|
| Prose written for humans | no — needs discipline | anything |
| Agent-facing rules with decidable triggers | yes | semantics |
| Structural tooling — dep-cruiser, arch tests, types, lint | yes | structure |

Two rubrics exist now, one per testing domain, each organised around a single question: *would this test fail if the behaviour it names were wrong?* The nine unit rules are nine distinct ways of answering "no" while still looking thorough. Rule 9 is literally "prove the test can fail," so the verified-red standard is codified rather than remembered.

## Rules need the same gate the tests got

The necessary counterweight: **a wrong rule amortizes a wrong standard.** Tooling outliving my attention is the entire benefit and the entire risk, and in the last stretch of this session three consecutive commits each reversed something from the commit before — all of them rules.

Three lessons, and they're the same lessons one level up.

**An unfalsified rule proves nothing.** Every new dep-cruiser rule got verified by planting the violation, watching it fire, removing it, watching it pass. A rule never observed failing is indistinguishable from a rule that doesn't work. That's how I found the bug in my own: `ci-fixtures-are-test-only` rejected `.ci.ts` importing `.ci.ts`, so a fixture could only ever be a single file — directly contradicting the convention's premise, since a route table held to our standards delegates to a service rather than inlining logic. Reading the pattern didn't reveal that. Building the two-file case did.

**Enumerate, don't pattern-match.** My first rule exempted `*.ci.ts` from canonical naming, which meant `helpers.ci.ts` passed where `helpers.ts` is rejected — any file could opt out of the convention by renaming itself. A special-case predicate in a lint rule is one rename away from being a universal escape hatch. The fix was to stop having a predicate: `routes.ci` became an ordinary entry in the canonical list, so a misnamed fixture fails like any other misnamed file. The correct rule ended up tighter than both attempts before it, and review pressure moving in the restrictive direction is not the direction unreviewed tooling drifts.

**A null result is ambiguous.** Twice a check "looked clean" only because the check itself was broken — a grep tripping on escaped quotes, and a link-verification script that silently matched nothing. Absence of output means either no violation or your verification didn't execute, and you have to know which. The strongest version of this bit me hardest elsewhere in the same session: I argued a failure mode was *impossible* on the strength of a string in a binary, wrote that into acceptance criteria as self-verifying, and was exactly wrong. So the rule isn't only *prove the check can fail* — it's *prove the thing you're relying on to tell you it failed.*

One shape worth preferring, since two of the session's failures had it: **inert is worse than broken.** A thing that is present, readable, and does nothing looks configured in the diff, in review, and in the file forever after. Prefer the mechanism whose failure is loud; when you can't, write the quiet failure down before it happens rather than after.

And a small loop-closing detail I like. The agent had *documented* the naming loophole as an acceptable trade-off, with reasoning — and writing it down is precisely what made it reviewable, and what got it killed a turn later. The payoff of recording a decision isn't only future legibility. A written-down bad decision is far easier to spot than an unwritten one.

## The context tax

One more property of agent-facing docs, because it has no human analogue and it changed how I wrote them.

The ordinary reason not to restate another document is DRY: two copies is how the unread copy goes stale. Two of my integration rules were restating the existing testing-setup doc, which is both derivative *and* precisely the failure the same file warned about three paragraphs earlier. They got demoted to an "inherited policy, not restated here" pointer.

The second reason is specific to this audience. The phase reference files are re-read *in full* on every single run, so anything restated in them is paid for on every invocation, forever, in exchange for a sentence the agent could have followed a path to. A human skims past a redundant paragraph once. An agent buys it again every time.

That rule also caught a misapplication of itself, which is the most reassuring thing a rule can do. I'd rejected a proposed test-writing skill partly on the grounds that it would be a third copy of the same pointer — except a pointer isn't a copy. The objection was wrong on its own terms, and the rule I was citing is what made that visible.

## What the gate is for

I started out thinking the value of a hard gate is that it stops bad code. It isn't — bad code is what the *next* gate stops. The value is that the gate creates a moment where reviewing tests is the only thing on my desk, so I read the mock instead of the pass count.

Which gives me the rule I'd actually defend: **a hard gate is expensive exactly once per class of mistake, and the deliverable of each expensive gate is a rule that makes the next one cheap.** A gate that produces no rule is pure cost. The rubric accreting is what turns "was that slow phase waste?" from a hunch into evidence — six or seven lines earned so far, none of them authorable up front, every one of them from watching a specific assertion fail to distinguish a specific pair of behaviours.

Some honest limits. This was a tests phase, which is unusually readable by design; whether an implementation diff of the same size survives PR-UI-only review is untested, and I shouldn't pretend otherwise. A rubric extracted from a handful of incidents will overfit in places — "mock returns must be distinguishable from their inputs" generalises, "prefix staged keys with `uploaded-`" does not. The no-editor feeling is downstream of the checks being *right*, and the moment a rule is wrong I'm reading prose about code I can't see, which is exactly what those three reversed commits were. And this all runs on [the gap I've written about before]({% link _posts/2026-07-24-corpus-ticket-and-corpus-work-under-the-hood.md %}) — no CI, no branch protection, and this week the automated reviewer hit its quota two PRs running, so my read was the entire review process. A gate is not a second pair of eyes, and I shouldn't confuse the two.

There's also a gap I didn't see until the rubrics were written and committed: a repo-wide grep for them returned zero hits outside their own directory. Nothing in the system referenced them. I had a standard and no mechanism that would ever apply it — which turns out to be a completely separate problem, and a harder one: [Your Quality Gate Isn't Configured Until You've Watched It Fire]({% link _posts/2026-07-29-your-quality-gate-isnt-configured-until-youve-watched-it-fire.md %}).

