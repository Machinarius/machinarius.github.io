---
layout: post
title: "Steering the AI Without Steering Yourself Into a Rubber Stamp"
date: 2026-07-26
---

Scoping out features with an agentic harness is a fundamentally different process than doing it artisanally. The appealing part is delegation: I can hand the AI an open-ended investigation — go compare these approaches, go summarize these tradeoffs — and get back a report tailored to what I actually need, in a fraction of the time it'd take to do it by hand.

The dangerous part is the same property viewed from the other side. The AI answers the question you asked. Ask a question shaped like a request for approval and you get approval. Ask for execution and you get execution, however under-specified the request was. Nothing about the harness pushes back on a badly framed ask — which means the work that used to go into doing the thing now goes into specifying it, and every failure mode below is a version of that same shift.

## Steer by asking, not asserting

The first mitigation is to steer toward a direction by asking questions rather than asking the AI to evaluate your choice directly. "What are the tradeoffs of X vs. Y" invites a real comparison. "Is X good?" invites agreement. The framing matters more than it seems like it should.

But this comes with a tension: the goal isn't to smuggle your preferred answer in through a longer path. If the questioning is just a more elaborate route to the same rubber stamp, you haven't gained anything — you still want the AI to genuinely challenge your design, not confirm it with extra steps.

That cuts both ways. I once asserted flatly that a design doc was no longer relevant and should just be deleted. Instead of agreeing, the AI pushed back with a concrete reason it was still load-bearing elsewhere in the codebase — and it was right. Steering-by-question protects against my confirmation bias; the AI declining to rubber-stamp *my* confident claim is the same discipline running the other direction.

## Dissent has to be manufactured on purpose

If agreement is the default, disagreement is something you have to go build. Three things reliably produce it, and none of them happen by accident.

**Give a sub-agent one hostile job and then leave it alone.** A failure mode specific to multi-agent setups: spinning off a sub-agent to research some ancillary concern, then having the parent intervene before it gets to explore independently. If you spun it off specifically so it could investigate without your assumptions in the loop, stepping in early defeats the entire point. I once gave a sub-agent one mandate — attack this design, find what's wrong with it — and it found the bug that mattered most: a success path that never marked its own work as done, so the next pass would silently redo it forever. Two ordinary review passes hadn't caught it. The one whose only job was to find fault did.

**Hand it something that already works.** Reviewing a design in isolation is weaker than holding it next to a sibling that's in production. I pointed an agent at another implementation of the same pattern and asked it to compare, not to review harder — and it found a real bug: the two designs kept their deduplication keys at different granularities, so one of them would silently drop legitimate work under load. Staring longer at the same file wouldn't have caught that. A second working example did.

**Ask explicitly for blind spots.** The AI catches some incidentally, just by working through a task, but it's far better at it when sent looking on purpose. Blind-spot detection isn't a side effect you can rely on — it's a mode you have to invoke.

## Planning didn't get superseded

There's a temptation, once the agent is fast enough, to skip straight to execution. Why write a plan when you can just say "do it" and watch code appear? The plan feels like the ceremony you were finally allowed to drop.

It isn't. Going straight into execution mostly moves the planning to a worse place: you discover the scope in pieces, mid-implementation, one surprise at a time. Every discovery arrives as an interruption to work that's already half-done, and each one either widens the change or forces you to unwind part of it. Scope extensions ad infinitum, found in the most expensive order possible.

And it wears on you. There's a specific frustration in watching a task you thought was nearly done sprout another offshoot, then another — the finish line moving every time you get close to it. At that point it's not a productivity problem, it's a morale one. Discovering scope up front is tedious; discovering it one interruption at a time, forever, is corrosive.

The plan is where scope gets bounded *before* anything is built on top of it. That job didn't get automated away — if anything the AI made it more valuable, because the AI will happily execute an under-specified request all the way to a large, confident, wrong diff.

And no persona saves you here. You can open with "you are my advisor, push back on me" all you like — if the actual request is *go do this*, it goes and does it. The framing doesn't outrank the instruction. Wanting to be advised and then asking to be obeyed just gets you obeyed. If you want the plan, you have to ask for the plan.

## The judgment doesn't delegate

The AI is a very powerful token predictor, but it's still just that. It doesn't hold the whole system in its head unless you make it, and it won't surface the edge case you never thought to ask about. Thinking at the systems level — what happens when these pieces are assembled, what breaks at the boundary, what the unhappy path looks like — is still the engineer's job.

Two places where that shows up concretely.

**Scope boundaries.** Left to its own devices, the AI will jam a piece of logic into whatever place it roughly belongs rather than drawing a hard line. It isn't being sloppy; it just has no strong prior about where a boundary *should* sit, because that's a judgment call, not a prediction. I haven't found a prompt that reliably fixes this. But the difference shows immediately when you ask for scope explicitly instead of hoping it holds: a rename once touched six places across a repo, and asked to scope it properly, the agent split that out into its own enumerated checklist rather than folding it silently into the change that triggered it — specifically so nothing quietly went un-updated. The hard line held because I asked for it.

**Prior art.** Always check what already exists in the repo first. Skipping this is how you end up with duplicate implementations, reinvented utilities, or a "new" design that quietly contradicts a pattern living three directories over. I've watched a ticket reopen a design question that had already been settled — the answer was sitting in the architecture docs the whole time, just not in the specific place the check had been told to look. Knowing where to look is the part that doesn't delegate.

## The upside, once you're actually steering

Do all of that and you get something genuinely new in return: throwaway scenarios and what-ifs have become stupid cheap. Spinning up an agent to explore a branch you're 90% sure you'll discard used to not be worth the time. Now it's close enough to free that you explore dead ends on purpose, just to confirm they're actually dead rather than assuming it — which is exactly the kind of cheap evidence a plan wants.

The same cheapness applies to illustration, not just investigation. [lavish-axi](https://github.com/kunchenguid/lavish-axi) has been invaluable for letting the AI explain complex concepts beyond what plain Markdown can carry — diagrams, side-by-side comparisons, interactive artifacts you can annotate directly instead of reading a wall of prose.

## Is prompting the new coding?

I'm not sure, but it feels like it somehow. Everything above — asking instead of asserting, giving sub-agents room to fail, asking explicitly for blind spots, refusing to let "do it" stand in for a plan — is a skill, and it's specific to working with these systems rather than a general engineering skill wearing a new hat. Whether it's a durable discipline or just this generation's syntax, still figuring out.

It points at something bigger, too. Learning concepts in the AI age is a fundamentally different exercise: you can learn faster than before, or skip the learning altogether and let the agent apply its own judgment. Either way the old-school route — building understanding entirely by hand — isn't competitive anymore. If you want the learning intact, the adversarial move works here as well as it does on designs: task agents with evaluating your comprehension and your plans directly, so the AI is a skeptic to argue with rather than an oracle to defer to.

That still leaves the question I haven't resolved. How do you avoid becoming dependent on AI to learn, when the market is pushing everyone to lean on it as hard as possible just to stay relevant? There's real tension between "use it constantly or fall behind" and "using it constantly is exactly what erodes the underlying skill." Arguing with a skeptic is the closest thing to an answer I've got — and it trades away some of the speed that made delegation attractive in the first place.
