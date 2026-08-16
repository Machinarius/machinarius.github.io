---
name: post-writer
description: Draft a post for this blog from a Claude Code session transcript, following the author's standing corrections on cross-links, decision history, overstated failures, and vague referents. Use when the user asks to write, draft, or turn a session into a blog post for this repo, or invokes /post-writer.
user-invocable: true
---

# post-writer

Every post on this blog is a Claude Code session transcript condensed into an essay. That is the
workflow this skill serves, and it is why most of the rules below are about what to **leave out**. A
transcript is a complete record of what happened and a bad first draft of why it mattered.

This skill owns the **prose standard only.** `new-gh-blog-post` owns the file and its front matter;
`post-preview` owns the preview. Neither is restated here.

## Start by asking what was interesting

Before you outline, ask the author what they found interesting about the work. Treat the answer as
the spine of the post.

This is the highest-value step in the skill, not a courtesy. On the most recent post, three
load-bearing sections came from framing the author supplied mid-draft that appears nowhere in the
session: that the CLI's intended audience was agent peers rather than humans; that mechanising a
guarantee collapses an expensive open-ended behavioural eval down to a single *did it call the tool*
check; and that centralising state is not the same as building a railroad. Each arrived as a message
during drafting. Each became a section. None was recoverable from the transcript by reading harder.

Ask **before the outline exists**, not after the draft does — reframing late costs a rewrite. Useful
questions:

- What surprised you? What did you expect to be true that wasn't?
- What would you have got wrong from the armchair?
- What's the claim here that someone could reasonably disagree with?
- If this had to be a third as long, which part survives?
- What's still unguarded, untested, or resting on someone remembering?

Then write to that answer. If the transcript's most dramatic moment isn't on the spine, it doesn't
go in the post.

## The standing corrections

Each of these is a correction the author has already had to make by hand.

### 1. Link freely to published posts — and verify before removing one

`{% link _posts/... %}` cross-references to **published** posts are good, and you should use them.
They make a post *lighter*: a linked phrase costs a clause, where the same idea inlined costs a
paragraph the reader didn't ask for. Deleting them doesn't buy independence, it buys length.

A post is published when it is committed and on `origin/main`. That is a check, not a judgement:

```
git log --oneline -1 -- _posts/<file>.md            # committed?
git log --oneline -1 origin/main -- _posts/<file>.md # on the remote?
```

**When the author says a link points at a draft, do not silently comply.** Run the check, state the
result in one line, and ask. This exact premise has already been wrong once, at scale: seven links
were removed across one review because the targets were believed to be drafts, and all seven were
restored afterwards — every target was committed and on `origin/main`.

The confusion to watch for is specific and easy to repeat: **a draft under review is not evidence
that anything it links to is a draft.** "This is a draft" is almost always true of the post on the
screen and almost always irrelevant to the post being linked.

It remains the author's call. If they confirm after seeing the evidence, remove the link. The rule
buys you one question, not a veto.

When a link genuinely does come out, replace it with an **extremely short summary of only what the
sentence needs to prove its point** — not a précis of the linked post. Most cross-references are
decorative: the sentence already carries the idea and the link points at a longer version. Those
collapse to nothing. The load-bearing ones collapse to a clause.

Links to external references are a separate thing and are also encouraged — see rule 8.

**Two things `{% link %}` does not do, both of which have already caused confusion.**

It resolves to a *page* URL, never a section anchor, so a bare cross-link drops the reader at the top
of a long post and leaves them to find the paragraph you meant. **Deep-link to the specific heading
by default.** Append the fragment yourself:

```
[text]({% link _posts/<file>.md %}#the-heading-slug)
```

Get the slug from the rendered page rather than deriving it — kramdown's rules for apostrophes, em
dashes and commas are easy to get subtly wrong, and a wrong fragment fails silently by landing at the
top, which is indistinguishable from having no fragment at all:

```
curl -s https://germanvalencia.dev/<path>.html | grep -o '<h2 id="[^"]*"'
```

Pick the heading that contains the claim your sentence is borrowing, not the one whose title sounds
closest. Those differ more often than you'd expect — a link about ceremony decaying belongs at
`#the-ceremony-that-survived`, where the rule is actually stated.

**Leave the fragment off when the idea lives in the target's un-headed introduction**, which is
common: opening paragraphs carry the thesis and have no anchor to aim at. Linking to the top is
correct there. What you must not do is invent a plausible-looking slug — check, or omit.

It also resolves **only at Jekyll build time**, so it is inert in any standalone HTML preview. That
belongs to `post-preview`, which owns resolving these to real published URLs — it matters to you only
in that the fragment you choose here is what that skill has to carry through.

> **When the same correction arrives a third time, stop applying it and question the premise.** Being
> told twice to remove the same kind of thing is a signal that something is wrong upstream of the
> edit. Applying it a third time is how one wrong belief becomes seven silent deletions.

### 2. Describe it as it is used, not as it was planned

No decision history, no "my recommendation had been X", no "we considered a follow-up ticket", no
planning artifacts standing in for behaviour. Describe the running system: what command runs, when it
fires, what it refuses, what it returns.

You will need this correction more often than any other, because a transcript is mostly deliberation.
The deliberation was expensive and therefore feels like the content. It isn't; the reader wants the
thing that ended up existing.

The exception is when a decision *is* the subject — the annotation that turned a table into a tool
earns its place because the post is about that turn, not about the process that produced it.

### 3. Cut the incident, keep the condition

A hiccup in the dev process is not material. Four paragraphs about a stale `node_modules` causing a
misdiagnosed build failure were cut wholesale: true, real, and useful to nobody.

But not every process observation is a hiccup. From the same review, this was explicitly **kept**:
neither the old harness (because the work was editing it) nor the new one (because it wasn't built
yet) could be used, so the ticket ran with no harness at all.

The test, applied literally:

> Would this recur for anyone doing this class of work, or did it merely happen to this build?

A stale dependency cache happened to this build. "The tool can't be used on the change that rewrites
it" happens to everyone who changes their own tooling. The first is noise; the second is a structural
property of the work, and often among the sharpest things in the post. This is the hardest judgement
call in the whole ruleset — when you genuinely can't tell, ask rather than guess.

### 4. Don't promote a hazard to a failure

Say which one you have: something that broke, or something that could have. Never narrate a latent
gap as though it fired.

> This wasn't failing at all, the model has always nailed the state machine. It's just that it not
> being mechanical opened the door to it failing.

Transcripts push you the wrong way here, because a defect *discovered* reads on the page like a
defect *experienced*. Overstating also costs you the better argument: "nothing was failing, and I
still wouldn't trust it" is a sharper claim than a bug report, because it's a claim about what the
correctness was resting on — which is the actual subject. If nothing broke, say so plainly and early,
then say why it mattered anyway.

### 5. Never end a paragraph holding the counter-argument

If you concede an objection, disarm it before the paragraph break. The last sentence of a paragraph
is the one that carries, and a reader who stops there keeps whatever was on the page.

Name the objection, turn it, then break — in that order, inside one paragraph:

> The obvious objection is that closing a door nobody has walked through is premature optimisation.
> But that objection was always a claim about *price*, not about correctness — and AI has brought
> that price down massively.

### 6. Name the cause and the magnitude

Hedged summary phrases are empty. "The price is what moved", "things have shifted", "the calculus is
different now" each ask the reader to supply the cause and the size, and readers don't.

> "the price is what moved" sounds empty on it's own — "the price has been brought down massively by
> AI" sounds a lot tighter

Same sentence position; now it names an agent and a magnitude, which makes it a claim someone could
disagree with. That is the bar. A sentence nobody could argue with is carrying nothing.

### 7. Name the referent, and anchor where it lives

Be concrete about what a thing is and which file, field, or command holds it.

"One schema field" became "one schema field in the handoff's YAML frontmatter — `layers`, the list of
test layers this feature actually runs." The paragraph's entire argument was that the guard was
*small*; an unnamed part makes that claim unjudgeable. Whenever a paragraph turns on something being
cheap, small, or contained, the reader has to be able to see the thing before they can agree.

Anchor everything to a location: the field in the file, the command in the script, the rule in the
validator.

### 8. Attribute standards, and link them

When the work follows a named external standard, name it and link its homepage — e.g.
[AXI](https://axi.md/). This is where external links belong, and they are welcome.

It changes what the paragraph is doing. A list of design preferences is opinion; the same list under
a named standard is a list of *compliances*, and the reader can go and check what else the standard
demands.

## Voice, taken from the posts themselves

These aren't corrections — they're conventions observed by reading `_posts/`. Read the two or three
most recent posts before drafting rather than trusting this table. It summarises a moving target, and
the source is one directory away.

| Convention | What the posts do |
| --- | --- |
| Opening | First sentence is the concrete failure or the setup. No throat-clearing, no "in this post". |
| Section titles | Short declarative assertions — *"The process couldn't run the ticket"*, *"Coverage measures execution, not assertion"* — never labels. |
| Numbers | Dense and specific: 37 merged PRs, 36 unit tests, nine days stale. Counts beat adjectives. |
| Being wrong | Stated outright: *"my first description of it was too strong"*, *"I was wrong about that too"*. |
| Quotes | Block-quote the real sentence — from a skill file, a commit message, an annotation — instead of paraphrasing it. |
| Tables | Only when there's a genuine axis to compare on. Two or three columns, prose-length cells. |
| Closing | A `## Limits` or `## Where I'd temper this` section. |

That closing section is not a disclaimer, so don't write it like one. It's where the post earns the
rest of its claims: what was never exercised, how small the sample is, and which piece of evidence
points the wrong way for the author.

## Self-check before handing it over

Run this over the finished draft *before* the author opens a preview. Each line is a mechanical
check, not a matter of taste — the taste is what their attention is for, and every item they catch
here is attention they didn't spend on the argument.

- [ ] Every `{% link %}` target verified published, and deep-linked to a heading whose id you read off
      the rendered page — not one you derived. (Rule 1)
- [ ] No decision history: no recommendations, no rejected options, no follow-up tickets. (Rule 2)
- [ ] Every process anecdote passes *would this recur for anyone doing this class of work?* (Rule 3)
- [ ] Every defect says whether it fired or was latent, and none is dramatised past the evidence.
      (Rule 4)
- [ ] No paragraph ends on the objection. Read the last sentence of each one in isolation. (Rule 5)
- [ ] No "X is what moved" / "the calculus changed" — cause and magnitude both named. (Rule 6)
- [ ] Every "one small change" names the file, field, or command, so the claim is judgeable. (Rule 7)
- [ ] Named standards attributed and linked. (Rule 8)
- [ ] A closing `## Limits` section that states what was never exercised.

Report what you changed as a result. A silent pass is indistinguishable from a skipped one.

> **A note on where this is heading.** This list is applied by the same agent that wrote the prose,
> which is self-judging — the exact failure mode of a rule that lives in a document with no mechanism
> to apply it. It works while the list is short and mostly mechanical. As it grows, and as more of it
> becomes judgement rather than checking, the argument for a `post-critic` skill gets stronger: one
> job, hostile mandate, spawned as a sub-agent with no memory of having written the draft, returning
> candidate objections rather than edits. A reader who didn't write the text catches what its author
> cannot. Build it when this list stops being checkable in one pass — or when a correction the author
> makes by hand is one this list already contains.

## Applying review feedback

Review happens on a Lavish artifact, and `post-preview` runs that session. It forwards each
annotation here verbatim rather than acting on it, because these rules key on what the author
actually wrote. You own every word that changes as a result.

- **Edit `_posts/<file>.md`.** It is the record; the rendered artifact is a view of it. A correction
  that lands only in the HTML is a correction that doesn't exist — hand the re-render back to
  `post-preview`.
- **Read the annotation as an instruction about the argument, not about the sentence it's pinned to.**
  *"The end of this sentence reads like a counter-argument"* is a note about paragraph structure;
  fixing only the clause under the cursor misses it.
- **The rules above apply to feedback too.** If a correction rests on a premise you can check, check
  it and say so before complying (rule 1). If this is the third time the same correction has arrived,
  stop and question the premise rather than applying it again.
- **When a cut takes something with it, say so.** Removing a section the author asked to drop is
  correct; letting a load-bearing paragraph disappear inside it silently is not. Name what went and
  offer to reinstate it elsewhere.
- **Don't over-apply.** One annotation is one change. Rewriting neighbouring paragraphs "while you're
  in there" produces a diff the author didn't ask for and can't easily review.

## Amend this file when a correction isn't in it yet

Every rule above was earned the expensive way: the author caught it by hand, in review, one draft at
a time. That only compounds if the catching gets written down, so **keeping this file current is part
of the job, not a favour to a future session.**

The trigger is an event, and naming it matters — a rule with no event that fires it is a rule that
quietly stops being applied. The event is **the end of a review round.** Before you report a draft
done, look back at what the author actually changed.

Amend when:

- They made a correction this file doesn't cover, and phrased it as a standing preference rather than
  a fix to one sentence. *"Drop the mention of a follow-up ticket, describe this as it is used"* is a
  rule. *"Cut this paragraph"* is an edit. If you can't tell, ask which it is.
- They corrected the same thing twice in one review. Twice is the signal; see rule 1.
- A rule here turned out to be **wrong**, not just incomplete. Rule 1 began as "never link to another
  post" and was rewritten to "link freely, verify before removing" after its premise collapsed. Say
  plainly that the old rule was wrong and what the evidence was; don't quietly soften it.

How to do it:

1. **Propose it in the conversation first, and get an explicit yes.** Editing the file that governs
   your own behaviour is not something to do silently mid-draft.
2. Write it in the shape of the rules above: the author's correction verbatim, what it actually
   means, and the reason. A rule without its reason gets misapplied at the first edge case.
3. **Add the matching line to the self-check.** A rule with no check is a rule nobody will run, and
   the two lists drifting apart is how this file starts lying about itself.
4. Keep pointing at sources rather than copying them in. That constraint applies to new rules too.

Do not fold in one-off stylistic preferences, anything specific to a single post's subject matter, or
a rule you inferred rather than observed. This file is small on purpose; a list nobody finishes
reading enforces nothing.

## Relationship to the sibling skills

Three skills, in order:

1. **`post-writer`** — ask what mattered, read the transcript, draft the prose.
2. **`new-gh-blog-post`** — creates `_posts/<date>-<slug>.md`. It owns the naming convention, the
   front matter shape, and getting the date from `date +%F`. Don't guess or restate any of them here.
3. **`post-preview`** — resolves the Liquid tags and renders the draft in the site's own theme for review.

Same instinct as `post-preview`'s refusal to copy the palette into itself: this skill points at
what it doesn't own rather than transcribing it. The front matter shape lives in one skill, the theme
tokens live in `assets/main.scss`, and the current voice lives in `_posts/`. Read them there — the
copy nobody re-reads is the one that goes stale.
