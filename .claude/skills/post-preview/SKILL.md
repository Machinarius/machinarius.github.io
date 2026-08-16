---
name: post-preview
description: Turn a Jekyll post's markdown into a Lavish review artifact for this blog, resolving Liquid tags and rendering it in the site's own Forest Synth theme so the preview reads like the published page. Use when the user asks to preview, review, or visually check a post or a writing plan for this repo, or invokes /post-preview.
user-invocable: true
---

# post-preview

Wraps the `lavish` skill for this repo. Lavish handles the session, the annotation surface and the
feedback loop. This skill owns the two things Lavish can't know: **how the markdown becomes HTML**,
and **how the artifact should look**, so a preview reads like the site rather than like a generic
artifact.

It does not write or edit prose. Drafting is `post-writer`'s job, and so is every word the author
asks to change — see [Feedback goes to `post-writer`](#feedback-goes-to-post-writer).

## When to use it

- Previewing a draft post before it lands in `_posts/`.
- Previewing a research pack, outline, or plan **about** a post, so the review happens on a surface
  that matches where the writing is going.
- Any Lavish artifact produced inside this repo.

Don't use it for artifacts about a *different* project. Lavish's own rule is to match the design
system of the subject the artifact represents — if the artifact is about another repo's product, use
that repo's system instead.

## From Jekyll markdown to a standalone artifact

The artifact has to render with no Jekyll and no server, so everything Jekyll would have done at
build time you do by hand. Three of those are easy to get silently wrong.

**Front matter is not content.** Strip the `---` block. `title:` becomes the `<h1>`, `date:` becomes
the mono metadata line under it. Add a banner saying it is an uncommitted draft preview and naming
the source path, so a reader can never mistake the artifact for the published page.

**Liquid tags do not resolve — and `href="#"` is the trap.** `{% link _posts/<file>.md %}` is inert
outside a build. Rendering it as `href="#"` produces something that looks like a link and silently
scrolls to the top of the preview, which reads as a broken post rather than a preview artifact.
Resolve each one to its real published URL instead. `_config.yml` sets no `permalink`, so Jekyll's
default applies:

```
_posts/2026-07-29-a-hard-gate-is-expensive-exactly-once.md
  → https://germanvalencia.dev/2026/07/29/a-hard-gate-is-expensive-exactly-once.html
```

Carry any `#fragment` through unchanged, and spot-check a couple before handing the preview over:

```
curl -s -o /dev/null -w '%{http_code}\n' <url>
```

A 404 means the target isn't deployed yet — say so rather than shipping a dead link. External links
are already absolute; leave them alone.

**Convert to semantic HTML, not a `<pre>` dump.** Headings, paragraphs, lists, blockquotes,
`<code>` for inline backticks, `<pre><code>` for fenced blocks with `<` and `&` escaped, real
`<table>` markup for pipe tables. Wide blocks — tables, long code — go inside their own
`overflow-x: auto` container so the reading column never scrolls sideways.

## The design source, and why it is not written down here

Lavish resolves design direction in a strict priority order, and for this repo step (2) always
resolves the same way: **the site's own theme wins.** `assets/main.scss` is Forest Synth layered over
the `minima` gem — a committed, dark, single-theme design system with a named palette.

**Read the tokens from `assets/main.scss` at preview time. Never copy their values into this file.**

That is a deliberate call, not laziness. Copied values are a cache: the moment someone retunes the
palette, this skill becomes a confident, authoritative-looking, wrong second copy — and the copy
nobody re-reads is the one that goes stale. The variables are one file read away. Read them.

The same rule applies to anything else this skill could be tempted to restate: `_config.yml`'s theme,
minima's own rules, and the Lavish CLI's commands are all owned elsewhere. Point at them; don't
transcribe them.

## Steps

1. **Read `assets/main.scss`.** Take the `$fs-*` palette variables, the minima variable hooks
   (`$base-font-family`, `$base-font-size`, `$base-line-height`, `$content-width`), and the mono
   stack. These are the artifact's tokens.
2. **Open the Lavish playbooks that match the content** — `lavish-axi playbook <id>`. A draft post is
   usually none of them; a plan, comparison, table or decision surface each have one, and an artifact
   that asks the user to choose something must open `input`.
3. **Write the artifact** to `.lavish/<slug>.html`, applying the tokens from step 1 as CSS custom
   properties. Do **not** add the Tailwind/DaisyUI CDN snippet — that is Lavish's fallback for when a
   project has no design system, and this project has one.
4. **Open the session:** `lavish-axi .lavish/<slug>.html`.
5. **Poll for feedback:** `lavish-axi poll .lavish/<slug>.html`. Follow the `lavish` skill's rules
   about foreground polling; this skill changes nothing about them.
6. **Clean up when the work lands.** `lavish-axi end .lavish/<slug>.html`, then delete the file. A
   preview is a review surface, not a record — anything durable belongs in the post or the commit.

   `.lavish/` is gitignored, so a leftover preview can't reach a commit. It will still confuse the
   next session that opens one, and the directory is **shared** — check whether another session has a
   preview open before deleting anything in there.

## What the artifact has to reproduce

Match what a reader actually sees on the published page. Each of these is set in `assets/main.scss`;
go there for the values.

| Element | What the site does |
| --- | --- |
| Page ground | `$fs-background`, with `color-scheme: dark` on `html` |
| Body text | the sans stack at `$base-font-size` / `$base-line-height`, `$fs-foreground` |
| Metadata, labels, data | the mono stack at 13px — the site uses mono for anything that is *not* prose |
| Reading column | `$content-width`; let tables and wide blocks exceed it, nothing else |
| `h2` | bottom hairline in `$fs-border`, negative letter-spacing |
| Blockquote | 2px left rule in `$fs-primary`, `$fs-card` ground, **not italic** |
| `code` / `pre` | `$fs-card` ground, `$fs-border` hairline, mono |
| Tables | `$fs-accent` header ground with `$fs-primary` mono header text, zebra on even rows |
| Links | 1px underline, 2px offset; on hover `$fs-primary` plus the micro-glow text-shadow |
| Focus | 2px `$fs-primary` outline, 2px offset |

Three rules that are easy to get wrong:

- **Paint the background explicitly.** Artifacts stay portable and open outside Lavish; a transparent
  body borrows whatever ground the host paints.
- **This theme is committed to dark.** Don't add a light mode or a `prefers-color-scheme` block — the
  site has neither, and a preview that flips to light stops being a preview.
- **Never use colour as the only signal.** Status needs a word as well as a hue, or it dies in a
  grayscale read.

## Feedback goes to `post-writer`

A poll returns the author's annotations. **Do not rewrite prose in response to them.** You own the
render; `post-writer` owns the words, and its rules only work if they see what the author actually
said.

The loop, per annotation:

1. **Forward it verbatim** — the exact wording, plus the anchor text or selector it was attached to.
   Don't summarise, tidy, or pre-interpret it. `post-writer` keys on phrasing: whether this is a
   standing rule or a one-sentence fix, whether the same correction has now arrived twice, and
   whether a premise is worth pushing back on. Paraphrasing destroys all three signals.
2. **Let `post-writer` edit `_posts/<file>.md`.** The markdown is the record; the artifact is a view
   of it. A change that lands only in the HTML is a change that doesn't exist.
3. **Re-render the affected part** from the amended markdown. Lavish live-reloads on save, so there
   is no need to re-open the session.
4. **Reply with `--agent-reply`**, naming what changed. Feedback applied without a reply looks
   identical to feedback ignored.

Two operational notes, both learned the hard way:

- **Pass `lavish-axi` absolute paths.** A shell whose working directory moved earlier in the session
  resolves a relative path against the wrong root and the poll dies with `ENOENT` — the edits still
  land, but the reply never reaches the browser and the author is left waiting on a session that
  looks alive.
- **Queued feedback survives a dead poll.** If one dies, re-run it; nothing is lost.

## Relationship to `new-gh-blog-post`

`new-gh-blog-post` step 6 already does a minimal preview, and it deliberately specifies *"lavish's own
default styling."* That is the cheaper path and it is fine for confirming a file was written
correctly.

This skill is the themed alternative, for when the point is to judge how the writing **reads**. They
overlap; they are not duplicates. If a themed preview is wanted during post creation, invoke this
skill instead of that step rather than running both.
