---
layout: post
title: "Your Quality Gate Isn't Configured Until You've Watched It Fire"
date: 2026-07-29
---

A long review session left me with a written standard: two Markdown files spelling out what makes a good test in one particular repo, each rule earned from watching a specific assertion fail to distinguish a specific pair of behaviours. That session is [A Hard Gate Is Expensive Exactly Once]({% link _posts/2026-07-29-a-hard-gate-is-expensive-exactly-once.md %}); this post picks up at the sentence that one ends on.

Writing them down was the cheap part. Then a repo-wide grep for `rubrics` returned zero hits outside `docs/rubrics/` itself. No doc, no skill, no config, no code mentioned them. I had a standard that nothing in the system would ever apply.

So the configuration problem isn't "which linter." It's: **for every rule you want enforced, what mechanism applies it without anyone choosing to?** And then, immediately after: how do you know that mechanism is actually running? Because I got that second question wrong four separate times in one session, and every wrong answer looked like a working one.

## Rank the layers by what they don't trust

The available places to put a rule sort into a ladder — ordered not by how much they can express, but by how little they ask of anybody:

1. **Structural checks** — dependency-cruiser rules, an architecture test in the suite. They run in `pnpm test`. Nothing to remember, nothing to trigger.
2. **Hooks on the write itself** — `PreToolUse` on `Write|Edit`. Fires whether or not anyone wanted it to.
3. **Prompt files that are loaded in full every run** — my harness's phase references. A pointer sitting there is in the context window automatically; nobody has to fetch it.
4. **The agent-facing index** — `AGENTS.md`. Depends on the index being loaded, and then on the link being followed. More on this below, because I was wrong about the first half.
5. **A skill** — triggers on model-judged intent. Probabilistic, and the only layer that can load a whole document before the write.
6. **A flag you pass at invocation time** — technically the most reliable transport on the list, and last anyway. It attaches a standing responsibility to whoever is driving.

That last entry is what the whole ordering is really measuring. The question isn't how reliably a layer delivers its payload; it's **how much operator responsibility it attaches** to deliver it at all. A structural check attaches none — clone the repo, run the suite, it's there. A flag attaches a permanent one, on every invocation, forever, to every person and every automation that ever pushes.

The temptation is to place each rule at whichever layer is cheapest to write, which is almost always near the bottom. Adding a line to `AGENTS.md` costs nothing and feels like progress. The useful move is the opposite: put each rule at the highest layer that can express it, and treat everything below as backstop rather than mechanism.

## The hook's constraint chose the design

I assumed a `PreToolUse` hook could hand the agent a document before it wrote the file. It can't, quite, and finding out why reshaped what I built.

Three things about the API mattered:

- `matcher` is tool-name-only — `"Write|Edit"` — but each handler also takes an `if` field using permission-rule syntax, which matches arguments too. So `if: "Write(**/*.test.ts)"` gets real path filtering, and you can route `*.api.test.ts` to the integration rubric and everything else to the unit one.
- `additionalContext` is documented to land *next to the tool result*. That is, after the write. An allow-and-inject hook is a revise-after nudge, not a pre-write guide.
- Hook output is capped at 10,000 characters. My rubrics are 9,382 and 8,963 bytes. Neither inlines comfortably; both together are impossible.

The only shape that gets text in front of the agent before the file exists is `permissionDecision: "deny"` with the pointer in `permissionDecisionReason`. I specified it that way first, then reversed it — and the reversal is the part worth keeping.

A hook that denies the first test-file write in a session is a hook that can stall every test-file write in the repo when it's wrong. Allow-only removes that failure mode entirely; the worst case becomes an unhelpful nudge. And "after the write" turns out to be exactly the right moment for what the rubric actually needs to say: *now go verify what you just wrote against this checklist.* The constraint I'd been treating as a limitation was describing the correct design.

Two knock-ons. Once nothing blocks, firing once per session is needlessly stingy — once per test-file path per session is right, because a genuinely new test file deserves its own nudge. And the review depth I'd budgeted for building the hook dropped a full tier, because the risk it was priced against had evaporated.

The hook injects the path plus a short checklist, not the file. Discoverability is what a hook is for; content lives where content lives.

## The config block that did nothing

Here's the part I'd want someone to take away.

The repo runs an external validation pipeline on every push — review, tests, lint, docs, PR, CI. Its config file already carried a 110-line `document.instructions` block, added months earlier, pointing the docs step at a README and telling it what to read first. Adding a `review.instructions` block with a pointer to the rubrics was obviously the same move at a different step.

I convinced myself it was safe to specify without testing, on three grounds: precedent in the same file, an explicit statement in the pipeline's own prompt that repo-specific instructions "may narrow or clarify, never weaken" — so per-step injection is a designed feature — and, decisively, a string in the binary reading `contains unknown field %q`. A validator that rejects unknown keys means you cannot commit a block that quietly does nothing. I wrote that into the acceptance criteria as *self-verifying*.

All three grounds were true. The conclusion was wrong.

A sentinel probe settled it: throwaway repo, a unique marker inside `review.instructions`, and a diff adding exactly the file type the instruction described. The review agent's own account afterward:

> My review prompt contained only the generic review task text. It did not contain the marker, the rubric directive, or any other text from `review.instructions`. I only know the marker exists because I read the config off disk myself — not because the pipeline injected it.

Confirmed on two versions, with `document.instructions` running as a positive control in the same probe — that step *did* receive its marker, which is what makes the negative result mean something rather than just meaning the probe was broken. The `review` step simply doesn't consume the key. It accepts it, and silently discards it.

So the failure mode I'd argued was impossible is the one that was actually there. **A config surface that tolerates unknown keys is the worst thing a gate can be, because a dead block and a live block look identical in the diff, in review, and in the file forever after.** And no amount of reading the binary, the docs, or the precedent distinguishes them. Only a marker travelling end to end does.

The generalisation I'd draw: a gate is not configured when the config is written. It's configured when you've watched a sentinel come out the other side, with a control proving your probe can detect a hit at all.

## It has to work out of the box, or it's just a checklist

The obvious replacement worked on the first try. The pipeline takes an `--intent` string, and it reaches the reviewer verbatim — the probe quoted its marker back. Better, a bare pointer was enough: given only a path, the reviewer went and read the rubric, then filed an error-severity finding citing a rule *by name*, identifying the exact trigger in the diff and quoting the rubric's own checklist. Nothing needed inlining.

It also had a property I hadn't asked for and now want everywhere: in a repo with no `docs/rubrics/` at all, it didn't skip the check — it hunted, found a copy elsewhere on disk, used it, *and filed a separate warning that the substitution was unverified*. A gate that tells you its inputs were missing beats a gate that quietly grades you against nothing.

And I dropped the layer anyway, because `--intent` is a flag.

Not because I'd forget it — though I would. Because it converts a tooling problem into a human responsibility, and that's the specific trade I'm unwilling to make. Every mechanism I add to this harness should work the moment the repo is cloned, with nobody told anything. A flag inverts that: the gate now depends on the operator knowing that a thing needs doing, remembering what it is, and getting it right on every push — including the pushes made by automation that was written before the rule existed, and by the next person, who was never in the conversation where we decided this mattered.

That's not a weaker version of automation. It's the thing automation was supposed to replace. Anything re-supplied per invocation isn't a gate, it's a habit — and habits failing is the entire reason I was doing this. The standard already existed; what didn't exist was anything that applied it without me. Solving that with a flag I have to remember would have been the same bug one level up.

So: prefer the layer that lives in the repo, even when it's weaker on paper. Repo-resident and 80% reliable beats invocation-bound and perfect, because the 20% is a known gap you can design a backstop for, while the operator burden is an unbounded one you can only apologise for later. And it's worth being blunt that "we'll document that you have to pass `--intent`" is not a mitigation — it's the failure mode wearing a mitigation's clothes. A doc telling someone to remember a flag has exactly the reliability of the doc nobody read, which is where this whole exercise started.

## Check whether the layer is already free

Then the probe went one step further, and this is the cheapest lesson in the whole exercise: I stripped the config block *and* deleted `AGENTS.md` from the repo entirely, then pushed a diff with a weak assertion in it. The review agent still located the rubric on its own and filed error-severity findings citing two rules by name, the exact trigger, and the rubric's headline test.

The backstop was free. It came from the rubric existing at that path, and nothing else. That layer went from an acceptance-criteria block to zero work — and it is, by construction, the out-of-the-box behaviour I'd just spent two dead ends trying to build: no flag, no config, no instruction, nothing for anyone to remember. Clone the repo and it happens.

The honest caveat is in the ticket too: three of three runs is not a guarantee. This is model discretion, not enforcement, and it's scoped as a backstop with the hook remaining the mechanical layer. But the sequencing lesson stands — before building a mechanism to feed a rule into a gate, check whether the gate already finds it. I nearly shipped configuration for a behaviour I already had.

## The index nobody loads

The layer I'd called weakest turned out to be weaker still, and for a reason that has nothing to do with agents ignoring instructions.

`AGENTS.md` in that repo is 20 KB of conventions with ten `> **Reference:**` pointers in it. It is not auto-loaded into a session. There's no `CLAUDE.md`, so session context carries global rules and nothing from the repo's own guide — the file is read only if something opens it. The probe agent could see this directly in its own injected context block: my global rules were there, the repo's guide wasn't.

So all ten existing reference lines are weaker than they look, and the testing pointer I was about to add would have inherited that. The fix is a committed `CLAUDE.md` symlink, which fixes every session at once and requires nothing of anyone starting one — with precedent, since the repo already commits eight symlinks at mode `120000` for exactly this fresh-clone reason. That property is why a symlink beats the obvious alternative of telling everyone to read `AGENTS.md`: one is a file in the repo, the other is a responsibility handed to a person.

One assumption got written down rather than left silent: on a filesystem without symlink support, a committed symlink materialises as a text file containing a path. The layer becomes *inert* rather than *broken*. That's the worse of the two failure shapes, and it's the same shape as the dead config block — which is why it's named in the ticket instead of discovered later.

## Four plausible mechanisms

That's the tally: a deny-shaped hook that would have blocked every test-file write in the repo, a config block accepted and silently discarded, a flag that outsources the work to whoever pushes next, and an index that never loads. Every one was a reasonable design. Two failed silently. One worked perfectly and still had to go, because a gate that needs a person to remember it is just that person, with extra steps.

None of them were distinguishable from working by reading — not the docs, not the binary, not the precedent in the same config file I was editing. The only thing that separated the live layers from the dead ones was a marker travelling end to end, on a repo stripped down until nothing but the mechanism could explain the result.

Which is the whole thing, really. You don't get to reason your way to a working gate, and you don't get to delegate the last mile of it back to yourself. You get to watch one fire, on a clone nobody prepared, with nobody told anything.
