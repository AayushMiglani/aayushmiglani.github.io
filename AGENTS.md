# AGENTS.md

Guidance for Codex when working in this repository.

## What this repo is

A personal engineering-writing site — **plain static HTML, no build step, no framework, no JS**. Served by GitHub Pages at https://aayushmiglani.github.io/. Every page is hand-written HTML you can open directly in a browser. There is nothing to compile, bundle, or transpile; what you write is what ships.

Do not introduce a static-site generator, a CSS framework, a package.json, or a build pipeline. The whole point of this site is that it stays dependency-free and instantly editable. If a change can be made in plain HTML, make it in plain HTML.

## Layout

```
.
├── index.html          — homepage; contains the post index (<ul class="posts">)
├── 404.html            — custom 404
├── README.md           — human-facing "how to add a post" notes
├── resume/             — résumé (LaTeX source + output); unrelated to posts
└── posts/
    └── <slug>/
        ├── index.html          — the article (self-contained, inline CSS)
        └── crosspost/devto.md  — optional dev.to cross-post (Markdown)
```

Each post lives at its own clean URL (`/posts/<slug>/`) by being an `index.html` inside a folder. Slugs are lowercase, hyphenated, and descriptive (e.g. `correlation-id-spring-boot`, `backend-for-frontend-aggregation`).

## Adding a new post — checklist

1. Create `posts/<slug>/index.html` from the template below.
2. Set a unique `<title>`, `<meta name="description">`, and all the Open Graph / Twitter meta tags.
3. Add `<link rel="canonical" href="https://aayushmiglani.github.io/posts/<slug>/" />` in the `<head>`.
4. Prepend a `<li>` entry to `<ul class="posts">` in the root `index.html` (newest post first).
5. If cross-posting, add `posts/<slug>/crosspost/devto.md` (see Cross-posting below).
6. Commit and push to `main` (see Git / deployment — note the push gotcha).

## The post template

Every post is **fully self-contained**: it carries its own complete inline `<style>` block. There is no shared stylesheet — this is deliberate, so each post renders standalone and can never be broken by an unrelated change. **When creating a new post, copy the entire `<style>` block verbatim from an existing post** (e.g. `posts/correlation-id-spring-boot/index.html`). Keep the design system identical across all posts; do not invent per-post styles.

Structural skeleton inside `<body>`:

```html
<div class="container">

  <nav style="margin-bottom: 32px; font-size: 0.9rem;">
    <a href="/" style="color: var(--muted); text-decoration: none;">← All posts</a>
  </nav>

  <header class="article-header">
    <h1>Title</h1>
    <p class="lede">One- or two-sentence italic summary of the whole piece.</p>
    <div class="byline">
      <strong>N min read</strong> · Tag · Tag · Tag
    </div>
  </header>

  <section>
    <h2>...</h2>
    <p>...</p>
  </section>

  <hr/>   <!-- renders as "· · ·" section break -->

  <section>
    <h2>Recap</h2>
    ...
  </section>

  <footer>
    <p>Related links...</p>
    <p><strong>Author:</strong> <a href="https://github.com/AayushMiglani">Aayush Miglani</a> · <a href="/">aayushmiglani.github.io</a></p>
  </footer>

</div>
```

### Design-system reference (the CSS tokens already in the template)

Use these existing classes and CSS variables — don't add new ones unless a post genuinely needs something the system can't express.

- **Colors** (`:root` vars): `--fg`, `--muted`, `--accent`, `--accent-soft`, `--border`, `--code-bg`, plus callout colors `--callout-bg/border` (yellow) and `--warn-bg/border` (red).
- **Body**: serif (`Charter, Georgia, …`), 19px, line-height 1.7, `.container` capped at 720px.
- **`<h2>`** = major sections; **`<h3>`** = subsections.
- **`code` / `pre`**: monospace; `pre` is dark (GitHub-dark palette). Escape HTML entities inside `<pre><code>` (`&lt;`, `&gt;`, `&amp;`).
- **`blockquote`**: blue, accent-bordered — use it for the **"What you'll build"** framing box near the top and for key asides.
- **`.callout`**: yellow box for tips/notes. **`.callout.warn`**: red box for traps and footguns. First `<strong>` becomes the callout's bold lead line.
- **`.flow`**: bordered light box with `white-space: pre` — use for ASCII architecture/sequence diagrams.
- **`hr`**: renders as a centered `· · ·` divider (via `::before`); use sparingly to separate the body from the recap.

## Writing voice and structure (house style)

The existing posts share a consistent, deliberate style. Match it.

- **Generic and teachable, never company-specific.** Posts explain a *pattern* readers can apply anywhere. Do not reference any employer, internal service name, product name, real endpoint, table name, or proprietary domain term. Invent neutral names (`ScreenResponse`, `SectionBuilder`, `Campaign`) and, when asked, use a fully abstract domain. Code is illustrative pseudo-real Java/Spring, not copied from any private repo.
- **Open with the problem, not the solution.** First section motivates *why this matters* with a concrete scenario before any code appears.
- **A "What you'll build" blockquote** near the top sets expectations for the whole piece.
- **Numbered steps or clearly-named `<h2>` phases** carry the body. Each section teaches one idea.
- **Callouts earn their place**: yellow `.callout` for a useful tip, red `.callout.warn` for "the trap most teams hit." Don't overuse them.
- **Close with two sections**: a forward-looking **"Where to take this next"** (2–3 numbered next steps) and a **"Recap"** (numbered recipe summarizing the whole post).
- **Footer** links related posts and credits the author.
- **Tone**: precise, confident, conversational-but-technical. Short paragraphs. Real numbers and trade-offs over hand-waving. Every sentence should teach something.
- **Read time** in the byline should roughly match length (~200 wpm).

The cleanest reference for all of the above is `posts/correlation-id-spring-boot/index.html`.

## Cross-posting (dev.to / Medium)

The canonical `<link>` in the HTML points back to this site so SEO weight accrues here, not the platform. When a post is cross-posted:

- Add `posts/<slug>/crosspost/devto.md` — the same article in Markdown with dev.to front matter:
  ```yaml
  ---
  title: "..."
  published: false
  description: ...
  tags: tag1, tag2, tag3, tag4   # max 4, no spaces inside a tag
  canonical_url: https://aayushmiglani.github.io/posts/<slug>/
  cover_image:
  ---
  ```
- Start the body with a `>` blockquote linking back to the blog ("This article is also available on my blog: …").
- Convert HTML to Markdown faithfully: `<pre><code>` → fenced code blocks (unescape the HTML entities back to real `<`, `>`, `&`), `.flow` diagrams → plain fenced blocks, `blockquote`/`.callout` → `>` blockquotes (lead with a bold phrase).
- **Medium**: use "Import a story" with the GitHub Pages URL — it preserves the canonical reference automatically. The dev.to Markdown can also be pasted into Medium.

## Git / deployment

- GitHub Pages serves `main` directly — **pushing to `main` is deploying.** Updates go live within a minute or two. There is no staging branch and no CI gate, so review rendering before you push (open the HTML file locally).
- **Push credential gotcha**: the repo is owned by GitHub user **`AayushMiglani`**, but this machine's default HTTPS credential is a different (work) account that gets a **403** on push. SSH authenticates correctly as `AayushMiglani`. If an HTTPS push is denied, push over SSH:
  ```bash
  git push git@github.com:AayushMiglani/aayushmiglani.github.io.git main
  ```
  (Or permanently: `git remote set-url origin git@github.com:AayushMiglani/aayushmiglani.github.io.git`.)
- **Commit hygiene**: only stage files related to the change at hand. The working tree may carry unrelated in-progress edits (e.g. `resume/resume.tex`) — never sweep those into a post commit. Stage paths explicitly (`git add index.html README.md posts/<slug>/`).
- Commit message style: conventional commits (`feat:`, `fix:`, `docs:`), short imperative subject + brief body.

## When the user asks for "a new doc/post/article"

Default workflow:
1. Confirm the topic angle and how abstract/generic it should be (these materially change the output) before writing.
2. Copy the template + `<style>` from an existing post; write the article in the house voice.
3. Add the homepage `<li>` entry and update `README.md`'s structure tree if it lists posts.
4. Offer to produce the dev.to crosspost.
5. Commit only the relevant files; push (mind the SSH gotcha).
