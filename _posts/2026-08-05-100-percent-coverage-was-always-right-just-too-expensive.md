---
layout: post
title: "100% Coverage Was Always Right, Just Too Expensive"
date: 2026-08-05
---

Years before any of this, I worked on a project with a hard 100% coverage gate. Istanbul, thresholds at 100 across all four metrics, CI red at 99.4%. Exceptions were allowed, but only one way: an explicit ignore comment with a written reason next to it.

It worked. It found blind spots in our tests passively, week after week, in a way no review round ever did — not because we were disciplined, but because the tool didn't need us to be. And it was tedious enough that I would not have recommended it to anyone.

The tedium had a specific shape, and naming it matters, because it's the part that changed. The gate never left you wondering *what* to test — it printed the file and the line. The cost was that writing the fifteenth test for a `default:` branch nobody will ever hit costs exactly as much as writing an interesting one. Sustained tedium is how a standard dies: not repealed, just quietly lowered to 85 during a bad sprint.

Test code is now the cheapest thing in my pipeline. So the standard is worth another look, and I think it's better now than it was then — for a reason that has nothing to do with generating tests faster.

## The gate the model can't argue with

I keep coming back to the same want: assertions that hold without the AI's cooperation. Coverage instrumentation is nearly the ideal shape of one. It's a counter, not an opinion. It doesn't read the diff, has no taste, cannot be persuaded, and has no view on whether this particular exception is reasonable given the deadline. It reports which lines executed. The line ran or it didn't.

Compare that to the layers I actually rely on for semantics: a reviewer agent's findings, a rubric applied by a model, a doc that gets followed. All useful; all model-judged. Coverage is arithmetic. That makes it a poor quality bar and an excellent blind-spot detector — a distinction I'll come back to, because conflating the two is how 100% coverage earned its bad reputation.

## Why 100, and not 90

The number isn't ambition. 100 is the only threshold with no argument left inside it.

Any number below it has to be defended, and re-defended every time someone wants to land something. Nobody has ever won 85-versus-90 on merit; the winner is whoever is more tired. At 100 there's no debate to have, and — more to the point — nothing for an agent to negotiate with either.

Below 100, global and per-file thresholds are two different tools with two different failure modes, because a well-tested core carries an untested module. At 100 that distinction collapses: 100% global *is* 100% per file. Vitest even has a shorthand for the case where it stops mattering.

```ts
coverage: {
  thresholds: { perFile: { 100: true } }
}
```

And only at 100 is the check monotone in the direction you need. Any newly uncovered line fails, so the report is a diff. At 92, you can add a pile of uncovered code, stay at 92, and learn nothing.

Which is also why I'd skip the ratchet. Tools will offer to raise the threshold for you as coverage improves — `thresholds.autoUpdate` in vitest, the same idea by other names elsewhere. A ratchet beats a static 80. It's also a gate whose position is set by whatever last landed, which means the bar is an artifact of your history rather than a decision. 100 is the only value a ratchet converges to that nobody had to pick.

## Three exits, and you choose one in the diff

The gate's real effect isn't that it forces tests. It's that an uncovered line has exactly three exits, and the diff won't merge until you've taken one.

| Exit | What it means |
|---|---|
| Write the test | the branch is real and nobody was looking at it |
| Delete the code | the branch is unreachable, so it was decoration |
| Write the reason | it's real, not testable here, and now says so in the file |

The middle row is the one I underrated at the time. A hard gate *prices* defensive code, and a surprising amount of defensive code turns out not to be worth its price. Discovering that an `if (!x) throw` cannot be reached from any caller is a design finding, and it arrives as a coverage failure rather than as someone's opinion in review.

The third row is the whole design. Without it, a 100% gate is a lie factory — people write tests that execute lines for the sake of the number, and the gate reports green over a suite that asserts nothing. With it, the number stays honest and the exceptions become a short, greppable list you can actually read.

## The escape hatch is where all the judgment went

The hatch is a comment: `/* istanbul ignore next */`, and its siblings for `if`, `else`, and `file`. Istanbul tolerates trailing text, so the convention is to attach the reason to the hint itself:

```ts
/* istanbul ignore next -- process.exit(); the test runner would die with it */
```

Istanbul does not care whether you wrote one. So the policy that makes this design work is not a coverage setting at all: the coverage tool enforces the number, and something else has to enforce that the reason exists. That split is the same shape as [the config block that did nothing]({% link _posts/2026-07-29-your-quality-gate-isnt-configured-until-youve-watched-it-fire.md %}): a bare ignore and a justified ignore look equally deliberate in a diff, and only a second mechanism tells them apart.

The first rule I'd carry over unchanged: **a bare ignore fails the build** — it's an unexplained hole in the number, with no record of who decided it was fine.

The second one I had as *`istanbul ignore file` is banned outright*, and that's too strong. Debug harnesses, CI fixtures, and modules whose only job is to be driven by an integration test are all real, and demanding unit coverage of them manufactures exactly the assertion-free tests this gate is supposed to catch. A file that is covered somewhere else is not the same object as a file nobody tested.

What I'd defend instead is narrower, and it's about *where the decision lives*: **a whole-file exemption is the only exception whose scope grows by itself.** Every other ignore covers a fixed branch. This one covers code nobody has written yet — including the module that quietly grows inside the fixture six months from now. So the rule isn't "never"; it's that the exemption has to be visible from outside the file it exempts, and it has to say where the coverage actually comes from. Three moves, in the order I'd try them:

1. **Merge the coverage.** If an integration suite genuinely exercises the file, the honest fix is to make the number say so rather than to carve out an exception — blob reports plus `vitest run --merge-reports`, or `nyc merge` on the istanbul CLI. No exemption at all, and the day the integration test stops touching that file, the gate tells you. That's the one option where the claim "it's covered by the integration test" stays *true* rather than becoming folklore.
2. **Exempt in config, not in the file.** `coverage.exclude` for a fixture, or a per-glob threshold when you want a lower bar rather than none: `'src/fixtures/**': { lines: 0 }`. One list, in one place, in the diff — the same enumerate-don't-pattern-match move that keeps [structural rules from dying]({% link _posts/2026-07-29-a-hard-gate-is-expensive-exactly-once.md %}). Keep the glob tight enough that a new module can't drift into it by accident, because a broad exclude is the failure mode wearing a policy's clothes.
3. **`ignore file` with a reason naming the covering test**, when neither of the above is worth the work. The check below treats it like any other hint — a reason is required, nothing more — which is the permissive reading. Rejecting it outright is defensible too. The part that isn't optional is that *something* records why.

The distinction that survives all three: an exemption is fine, an *invisible* exemption isn't. A comment at the top of a file is the least visible place to put a policy, which is the whole objection — not the exemption itself.

### What actually enforces it

A grep, in CI. That's the whole mechanism:

```sh
# fail on any coverage hint that doesn't carry a reason
if grep -rPno --include='*.ts' --include='*.tsx' --include='*.js' --include='*.jsx' \
     '(?:istanbul|c8)\s+ignore\s+(?:next|if|else|file)\b(?!\s+--\s+\w)' src; then
  exit 1
fi
```

Two things about it are policy rather than plumbing. **The reason has to sit on the same line as the hint** — partly because a check that looks for a hint and for dashes as two separate facts can be satisfied by unrelated code on the same line, and partly because a reason short enough to fit on the line is a reason that stays scannable. And it's restricted to source extensions, or else the document describing the policy fails the policy, which is how a check gets deleted.

**Wrap it in `if ...; then exit 1; fi`, never `grep | grep -v`.** An inverted pipeline exits 1 when nothing matches, which is the *success* case. Get the polarity backwards and you ship a check that's either permanently red or permanently green, and permanently green looks exactly like working.

That's run, not sketched — a fixture of eleven awkward comments, including hints sharing a line with code, dashes with nothing after them, and reasons split across lines. It fails the seven that should fail.

One thing to get right rather than infer: **the pattern has to recognise everything istanbul recognises.** Istanbul anchors the directive at the start of the comment and does nothing with whatever follows, which is exactly why `-- reason` is a convention and not a feature. Anything narrower than istanbul leaves a comment that suppresses coverage without tripping the check; anything broader is merely noisy. Reading the two patterns won't tell you which you've built — plant a bare ignore and watch CI go red, then plant a justified one and confirm coverage really does stop counting that line.

ESLint can carry the same rule if you want it at edit time rather than at push time, but not as a config line: comments aren't AST nodes, so no core rule reaches them and you end up writing the rule yourself. I'd still start with the grep. One line, one place, no parser, no config resolution, and it sees every file type rather than only the ones your lint config parses. The editor loop is the only thing the other version buys.

Worth naming the layer above both, since neither can do it: checking that a reason *exists* is deterministic, but judging whether the reason is any good is a model-judged CI step. Different tier, and a different post.

Then there's the part that's genuinely new since the old project: a model will write you a *beautiful* reason. Fluent, plausible, well-formatted, and produced in the same second as the ignore hint it justifies. The gate is deterministic; the hatch is prose, and prose on demand is precisely the thing on the other side of the table.

So the sentence I'd put on the wall: **a deterministic gate with a prose escape hatch relocates all of the judgment into the escape hatch.** Which is fine — as long as you know that's where it went, and you read it. The consolation is that it's a bounded amount of reading, in one grep, and the count going up is itself the signal. Vague coverage anxiety becomes a specific list of lines someone argued their way out of.

## What cheap code generation actually bought

Not what I expected. The tedium was in writing the boring tests, and yes, that's now nearly free — the same inversion as [noticing being the job while fixing is free]({% link _posts/2026-07-29-a-hard-gate-is-expensive-exactly-once.md %}). The gate stops trading iteration speed for blind-spot detection, which was the trade that made it unrecommendable.

But the cost didn't vanish; it moved, and it moved toward the scarcer resource. A 100% gate generates a *lot* of test code, and every line of it lands in a diff I'm supposed to review. Volume is the raw material of a [rubber stamp]({% link _posts/2026-07-26-steering-the-ai-without-steering-yourself-into-a-rubber-stamp.md %}). Iteration speed I had to spare; review attention I don't.

That's the honest version of "AI compensates for the tedium." It converts a throughput cost into an attention cost. Worth it — but only if something other than my attention is checking whether those tests mean anything. Which is the ceiling.

## Coverage measures execution, not assertion

Istanbul counts which lines ran. It has nothing to say about whether anything was checked.

```ts
it("formats the record", () => {
  format(record); // 100% of format(), zero claims about it
});
```

That test takes a function to full coverage and asserts nothing at all. It's the exact failure that [rubric question]({% link _posts/2026-07-29-a-hard-gate-is-expensive-exactly-once.md %}) exists for — *would this test fail if the behaviour it names were wrong?* — and coverage is structurally incapable of asking it. That question also has a name and a tool, which I'll come back to in a moment.

Which sharpens the claim I started with. The gate an agent can't weasel out of is one-dimensional: it must make the line execute, and no amount of arguing changes the counter. It can weasel completely on whether executing proved anything, and a green coverage threshold is a direct incentive to do exactly that. **100% coverage without an assertion standard is a machine for manufacturing tests that run code.**

Paired, though, they cover each other's blind spot precisely: coverage is deterministic about *where nobody looked*, the rubric is judgmental about *whether looking meant anything*. Neither substitutes for the other, and I'd be nervous shipping the first without the second now in a way I wasn't years ago, because back then nothing was generating tests at volume.

The name is **mutation testing**, and it's the honest next rung: Stryker and friends perturb the code and check that the suite notices, which is *would this test fail if the behaviour were wrong* asked mechanically rather than by a reviewer. So the rubric is the hand-rolled approximation of a check that already exists as a tool — which is a slightly deflating thing to discover, and also the best argument for eventually running the tool.

I haven't. It costs a multiple of your suite runtime on every run, and cheap code generation does nothing about that: it's compute, not typing. Which makes it nightly rather than per-PR, and a thing I want to try properly rather than assert about here. I name it because the temptation, having found one deterministic gate, is to assume it covers more than it does.

## In a tests-first cycle, the gate grades the plan

There's a second reading of the same number, and it only exists if the ordering is enforced rather than intended. My harness [runs one phase per pull request]({% link _posts/2026-07-24-agentic-harness-styles.md %}): Phase 1 writes tests, red by design; Phase 2 makes them green. So picture Phase 2 landing with every test passing and coverage at 96%. Nothing is broken. Every assertion anyone wrote is satisfied. And yet there are lines in the implementation that no test written the day before anticipated.

That isn't a missing test. It's a measurement of the distance between the plan and the build, and it has exactly two explanations:

- **The implementation built something nobody asked for.** An extra branch, a defensive path, a case the ticket never mentioned. The plan was fine; the code is over-specified, and the fix is a deletion.
- **The plan asked for it and the tests-phase rubric didn't produce a test for it.** The code is right; the rubric has a hole, and that hole recurs on every ticket until someone writes it down.

Both are findings about *upstream* artifacts, which is what makes this worth more than the raw percentage. Every other check I run grades the diff. This one grades the ticket — and it grades it automatically, on every cycle, without anyone deciding to ask whether the planning was thorough. That question is normally unanswerable except retroactively, after the bug.

Which is why the *direction* of the repair matters more than the number does. The tempting fix — write a test for the uncovered line — is the one move that's almost always wrong here, because a test authored after the implementation in order to satisfy a threshold is derived from the code rather than from the requirement. That's the [fabricated premise]({% link _posts/2026-07-29-a-hard-gate-is-expensive-exactly-once.md %}) in its purest form: it will pass forever, and what it asserts is that the code does what the code does. In a tests-first cycle the legitimate paths back to 100% are deleting the code or amending the plan and re-running the tests phase. Topping up coverage at the end quietly converts an implementation detail into a requirement, and the number goes green while the signal is destroyed.

So the three exits from earlier acquire an order in this setting. Delete first. Amend the plan second. The written reason last, and only when the first two genuinely don't apply.

And it closes the loop on what makes these gates worth paying for at all: the deliverable of an expensive gate is a rule that makes the next one cheap. A coverage failure diagnosed as "the rubric never asks for an error-path test on external calls" produces a rubric line, and the next ticket's tests phase writes that test before any implementation exists. The number stays at 100 because the plan got better, not because someone chased it.

The honest limit is the same one that makes red-by-design work: this reading requires that the tests came first *and* that you can prove it. Where the ordering is a convention rather than a gate, "the test anticipated the code" and "the test was written to cover the code" are indistinguishable after the fact, and a coverage gap stops being evidence about the plan.

Once you're in that position, fabricating the premise is close to unavoidable — the only test available to you is one derived from code that already exists. Which is why the durable response to an uncovered line is not the test. It's **the rubric**: write down the rule that would have made the tests phase anticipate that code, and the next cycle starts from a stronger spec regardless of how honest this one's ordering was. A test written after the fact fixes one line and teaches nothing. A rubric line changes what gets written *before* the implementation exists, which is the only place the fix can actually land.

## The backlog of things we knew were right

The coverage gate is one instance of a larger category, and the category is the interesting part: **practices that were abandoned on labour cost, not on merit.** Not the ideas we tried and disliked — the ones we agreed were correct, scheduled, and then never got to, because there were always more tasks than people. Every team I've been on has that list, and nobody writes it down, because a backlog item whose only blocker is "this is a lot of typing nobody will fund" doesn't feel like a decision. It feels like weather.

Cheap code generation is a re-litigation of that entire list. Which is more interesting than it being a faster way to do the things we already do, and it deserves a filter rather than enthusiasm, because only part of the list actually comes back.

The filter is the same one that just showed up twice above: **ask what the practice was actually paying for.**

- Cost was *writing code* — the fifteenth `default:` branch test, the fixture that takes an afternoon, the error path nobody wanted to simulate, the property-based generators, the doc for the second consumer of an interface. These are now nearly free, and every one of them is a candidate to reopen.
- Cost was *reading code* — a second reviewer on every PR, a design doc per ticket, manual QA passes. These got *worse*, not better, because there's now more output competing for the same attention. Volume is the constraint, and generation adds to it.
- Cost was *compute or wall-clock* — mutation testing, exhaustive fuzzing, full-matrix CI. Unchanged. Nothing about cheap generation makes your test suite finish faster.

Three buckets, and the whole win lives in the first one. So the question worth asking of an old abandoned standard isn't "could an agent do this now" — it's "was the thing that killed it typing?" If yes, it's probably back on the table at close to zero cost. If it died because someone had to *read* the output, cheap generation has moved it further out of reach, not closer.

100% coverage happens to sit almost perfectly in the first bucket, which is why it's the one I reached for first. That's also the reason it isn't a general endorsement: the same reasoning that revives it rules out most of what's next to it on the list.

## The setup

- Thresholds at 100 on statements, branches, functions, and lines. At 100 the per-file question answers itself.
- `/* istanbul ignore <hint> -- reason */` as the only exception, plus a separate check that rejects a hint with no reason.
- Whole-file exemptions in the coverage config where they're visible, not as a comment inside the exempted file — and merged coverage instead, wherever an integration suite already covers it.
- Then plant an uncovered line, watch CI go red, and remove it — because a threshold nobody has [seen fire]({% link _posts/2026-07-29-your-quality-gate-isnt-configured-until-youve-watched-it-fire.md %}) is indistinguishable from a threshold set to zero, and the coverage config that silently excludes half the source tree is a very common object.

The gate's output was never really a number. It's a list of places nobody looked, regenerated on every push by something with no opinions and no stake in the deadline. That part was always worth having. What was missing was any way to act on the list cheaply, and that's the part that arrived.

Which is why I'd rather people take the filter than the gate. The gate is one revived standard and it might not be yours. The filter is the reusable move: go find the practice your team agreed was correct and dropped anyway, and check whether what killed it was typing.
