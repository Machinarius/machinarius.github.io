---
name: new-gh-blog-post
description: Create a new Jekyll blog post (front matter + _posts file) in a GitHub Pages Jekyll repo. Use when the user asks to write/add/create a new blog post, or invokes /new-gh-blog-post.
user-invocable: true
---

# new-gh-blog-post

Creates one file in `_posts/` following Jekyll's `YYYY-MM-DD-title.md` naming convention.

## Steps

1. Confirm the current repo is a Jekyll site (`_config.yml` and `_posts/` exist). If `_posts/` is missing, create it.
2. Take the title from the skill args. If none given, ask for one.
3. Slugify the title: lowercase, spaces/punctuation to `-`.
4. Get today's date with `date +%F` (don't guess it).
5. Write `_posts/<date>-<slug>.md`:
   ```
   ---
   layout: post
   title: "<Title>"
   date: <date>
   ---

   <body text if the user gave any, otherwise leave empty>
   ```
6. If `lavish-axi` is on PATH (`command -v lavish-axi`), preview the post before reporting: write `.lavish/<slug>.html` with the title in an `<h1>` and the body converted to normal semantic HTML (`<p>`, `<h2>`/`<h3>`, `<ul>`/`<ol>`, `<code>`, etc.) — not a `<pre>` dump — so it renders with lavish's own default styling like any other artifact, then run `lavish-axi .lavish/<slug>.html`. Skip silently if `lavish-axi` isn't available.
7. Report the file path. Do not `git add`/commit/push unless the user separately asks.
8. Once the post is committed (in this invocation or a later request), clean up the preview: if `.lavish/<slug>.html` exists, run `lavish-axi end .lavish/<slug>.html` (ignore errors if no session is open) and delete the file.
