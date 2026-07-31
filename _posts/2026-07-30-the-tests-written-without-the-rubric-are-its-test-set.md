---
layout: post
title: "The Tests Written Without the Rubric Are Its Test Set"
date: 2026-07-30
---

This is the third thing I've written about the same two Markdown files. The first was about earning them: seven review rounds that produced [nine rules for what makes a test any good]({% link _posts/2026-07-29-a-hard-gate-is-expensive-exactly-once.md %}), none of which I could have authored up front. The second was about discovering that [nothing in the repo would ever apply them]({% link _posts/2026-07-29-your-quality-gate-isnt-configured-until-youve-watched-it-fire.md %}), and the four plausible mechanisms I tried before finding one that worked on a fresh clone.

This one is about the question I hadn't asked either time: **how would I know the rules are any good?**

It came up sideways. I was mid-review on an unrelated PR, found a mock I didn't like, and went looking for whether the rubrics already covered it. They didn't. So I drafted new rules, checked whether a ticket already owned that work, and found the one that owns the delivery mechanism — with a priority rationale I'd written myself a day earlier:

> the payoff window is the next test-writing pass; a rubric that lands a pass late gets judged by the tests written without it.

I quoted that back as an argument for urgency. The pass in question was the one I was standing in. The rubric was about to miss it. Sunk cost, get moving.

The reply I got was one sentence: *if we judge the rubric on known-bad items, that's a good regression test.*

That inverts the whole thing. The tests written without the rubric aren't evidence the rubric arrived late. They're **labelled data**. Lateness isn't the cost — it's the only reason there's anything to score the rubric against.

## Rule 9, aimed one level up

The rubric already contains the discipline that validates it. Rule 9, verbatim:

> **Prove the test can fail before trusting it.** Plant the violation, confirm the failure names it, remove it, confirm green. A check nobody has seen fail is not known to work.

Its stated trigger is "always for a new lint or dependency-cruiser rule; whenever an assertion has only ever been observed green." It points outward, at machinery. Turn it around and it reads: **a rubric nobody has seen catch anything is not known to work.**

That's not a cute symmetry. It's the same failure mode I'd just spent a session on in a different costume. A dead config block and a live one look identical in the diff; a rule that catches real defects and a rule that catches nothing look identical in the Markdown. Both are indistinguishable by reading, and both stay indistinguishable until something travels end to end.

The difference is that a config block needs a sentinel I have to invent. A rubric's sentinels already exist — I've been generating them for weeks and throwing them away.

## The corpus is mostly transcription — this time

Here's what made me take this seriously rather than file it as a nice-to-have.

The test file with the mock I objected to carries a header documenting two *rejected earlier revisions of itself*: first a hand-rolled `as unknown as Context` fake, then an app whose routes were `() => { throw x }`. Both were reviewed, argued about, and replaced. And `IntegrationTesting.md` Rule 3 — "use a real route table, not an inline throwing app" — was plainly induced from the second one. Same story for the unit rubric's Rule 1, whose bad example is a hand-constructed `PrismaClientKnownRequestError` with an assumed `P2002` code, which is a real thing someone tried.

So the specimens are sitting in the tree, with recorded verdicts, next to the rules they produced. Building the corpus is largely harvesting.

That's worth stating as a general property: **a review process that writes down *why* it rejected something is already producing labelled data, and most of us discard it.** The rejection lives in a PR comment thread, or a commit that got amended away, or — if you're lucky and slightly obsessive — a comment block at the top of the file. A rubric extracted from those incidents is a compression of the corpus. Keeping the corpus means you can check the compression.

And then the counter-point that undercuts my own optimism, which I'd rather state than dodge: **none of that debris is guaranteed to exist any more, and agents make it scarcer, not more plentiful.** Nothing about that file header was inevitable. The default shape of an agentic correction is silent repair — I say *fix the mock*, it gets fixed inside the turn, and the rejected revision doesn't reach a comment thread or even an amended commit, because it was never a commit. Human review is sloppy and leaves debris everywhere. An agent handed a correction leaves none unless something obliges it to.

So the velocity that produces specimens faster also destroys their labels faster, and the two effects don't cancel — you end up with more defects fixed and less evidence about which rule would have caught them. The specimens I just called free exist only because a habit older than the agent happened to catch them on the way past.

Which flips the conclusion rather than weakening it. If the corpus is what makes a rubric checkable, recording the verdict cannot be a byproduct I hope to harvest later; it has to be something the review step is obliged to emit, for the same reason the rubric needed a delivery mechanism instead of my memory. Otherwise the eval set is whatever survived by accident — and "whatever survived by accident" is a description of a biased sample, not a test set. The specimens that get documented are the ones someone found interesting enough to write up, which is exactly the correlation you don't want between your corpus and your existing rules.

## Two failures that look identical from outside

The immediate payoff isn't scoring. It's a distinction I'd been collapsing.

When a defect gets past a rubric, exactly one of two things happened:

**Coverage gap** — no rule covers it. The rubric had nothing to say. My `jest.mock("@corpus/database")` is this one, and I checked carefully: nothing in either document addresses stubbing a *first-party* package, and Rule 1 forbids *constructing* a dependency's error class while saying nothing about *substituting* the class object. Worse, the unit rubric's `infrastructure` section says "Prisma's own class/code branches are **not** hermetic — Rule 1," which is correct and which is probably what made the mock feel sanctioned. The rubric told the author not to test those branches hermetically; nothing told them that stubbing the classes wholesale to reach the *other* branches drains every `instanceof` of meaning. The gap isn't an absent tenth rule, it's a sentence that opens a door and doesn't close it.

**Compliance gap** — a rule existed and wasn't applied. This is the category the delivery mechanism addresses, and the one I'd already measured: a repo-wide grep for `rubrics` returned zero hits outside the rubrics' own directory. Nothing referenced them. Any rule in there could only fire if someone happened to name the path.

These want opposite fixes. A coverage gap needs *content* — a new rule, or an edit to the sentence that misled. A compliance gap needs *delivery* — a hook, an auto-loaded file, something that puts the existing rule in front of the writer. And the failure is symmetric: **conflating them is how you ship a hook when you needed a rule, or a rule when you needed a hook.** I have now done both. The hook I built delivers a rubric that would have said nothing about the specific defect that motivated me to build it; the rules I drafted would have sat in a document that nothing loaded.

You cannot tell the two apart by looking at the defect. You can only tell by asking, of a specimen with a known verdict: was there a rule, and did the mechanism deliver it? That's a two-cell answer per specimen, and it's the whole diagnostic.

## It's scorable, which surprised me

I expected "evaluate the rubric" to bottom out in taste. It doesn't, and the reason is that the output format already exists.

The one mechanism that worked out of the box was the external review step finding the rubric unaided. What it produced, when I probed it on a stripped-down repo:

> findings citing Rules 3 and 4 **by name**, naming the exact `toBeGreaterThan(0)` trigger in the diff, and quoting the rubric's own headline test

That is eval output. For each specimen, the question is mechanical: **did the reviewer cite the rule this specimen is labelled with?** Not "did it seem to review thoughtfully." Precision and recall over a labelled corpus.

What makes that work is a property of the rubric I'd filed under style. Every rule carries a mechanical trigger — *"always for a new lint or dependency-cruiser rule; whenever an assertion has only ever been observed green"* — and the document says so in its own header: check the rule against the code you are about to write, not against your intent. I'd read that as good technical writing. It's actually the precondition for labelling anything. You cannot mark a specimen as violating *write clear tests*, so you cannot score that rule, so you can never discover it isn't working. **A rule without a trigger isn't merely weak, it's unfalsifiable** — and a rubric made of them stays unmeasurable no matter how good the corpus is.

Which forces one addition the session didn't get to. A corpus of only known-bad items measures recall — how many real defects the rubric catches. It says nothing about precision, and a rule that fires on everything scores a perfect recall. **You need clean specimens too**: tests that were reviewed and accepted, labelled "no finding." Without them there's no penalty for a rule that flags every mock in the repo, and vague rules are exactly the ones that drift that way. The accepted revisions are as much data as the rejected ones, and they're in the same file headers.

Once it's precision and recall over a labelled set, the obvious question is whether the existing LLM-eval harnesses just do this — a diff is text, a finding is text, and scoring predictions against labels is what those tools are for. I suspect the friction is that the unit under evaluation is a repository state rather than a prompt, which may want a thin adapter or may want something purpose-built. I haven't tried it, so that's a separate post rather than a claim here.

## The trap, stated plainly

The specimens a rule was induced from are its training set.

A corpus built only from them tells you the rules encode their own originating cases, which you already knew, because you wrote them from those cases. It measures nothing about generalisation. And I'd flagged the overfitting risk in the first post — "mock returns must be distinguishable from their inputs" generalises, "prefix staged keys with `uploaded-`" does not — without noticing that the risk was measurable rather than merely acknowledgeable.

So the corpus needs held-out items: defects found *after* the rubric was written.

I have exactly one. The mock survived review, shipped into an open PR, and the rubric was sitting in the tree the entire time. Nobody was prevented from applying it — it just wasn't. And the verdict on it came from a human read during review, which is what makes it a label rather than just a diff.

One held-out specimen, and it's a coverage gap. Data point one, in the actionable category.

## Which reorders the work

The practical consequence is that the corpus is a better first ticket than the new rules it was supposed to justify.

I had drafted two content rules — *never stub a value whose identity is the contract*, and *identity that crosses a module boundary gets a pinning test in the tier CI actually runs*. Both feel right. Neither is demonstrably necessary. With a corpus, the argument changes shape entirely: instead of asserting the new rule would have caught this, I can show that the current rubric provably misses a specimen and the new rule catches it. That's the difference between a rule I'm confident about and a rule with a receipt.

There's a second thing the scores would settle that I'd been deciding by instinct. The rules are not equally mechanizable. *A mock that fixes a load failure is an import-graph bug* has a tell precise enough to lint for — the suite failed to **load**, not to assert — whereas *never stub a value whose identity is the contract* cannot be machine-checked in the general case. The best that one reduces to is a question a reviewer has to actually ask: *if this stub were the wrong object, would any assertion in this file fail?* If the answer is no, the test agrees with itself. I'd been sorting rules onto [the enforcement ladder]({% link _posts/2026-07-29-your-quality-gate-isnt-configured-until-youve-watched-it-fire.md %}) by feel. A score sorts them by evidence: a rule the reviewer reliably cites is fine left as prose, and a rule it keeps missing is either badly written or needs to stop being prose.

It also, usefully, stays out of the way. The delivery ticket's scope explicitly excludes rubric *content* and machine-enforcement — it changes only what loads the rubrics. Validating a rubric is neither of those. Four ticket-sized pieces that don't overlap: does the standard exist, does anything apply it, does anything record the verdict when it's applied, and does applying it catch anything. I had been treating the third as free.

## So when should you introduce a rubric?

The answer I started the session with was "as early as possible, and the window is closing." That's not wrong, but it's answering a different question than the one that matters.

You write the rules when you have incidents, because — as far as I can tell — you cannot author them up front; every line I have was earned by watching a specific assertion fail to distinguish a specific pair of behaviours. You deliver them when a mechanism exists that doesn't depend on anyone remembering. But you can only **know** they work after a pass has run without them, because that pass is what produces the labelled specimens.

Which means the uncomfortable framing is the correct one: a rubric introduced before there's work to judge it against isn't early, it's *unmeasured*. Lateness isn't the failure mode. Lateness is the eval set arriving.

Honest about where this stands: none of it is built. The corpus is a design and a ticket I hadn't filed when the session ended, with one held-out specimen in it, and the reviewer behaviour I want to score was observed three times out of three, which is a signal and not a guarantee. The interesting part isn't the artifact. It's that "is this rubric good?" turned out not to be a matter of taste, and that the thing making it answerable was the pass I'd been treating as a missed opportunity.
