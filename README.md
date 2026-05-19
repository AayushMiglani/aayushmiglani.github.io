# aayushmiglani.github.io

Personal site for engineering writing — built as plain static HTML, served by GitHub Pages.

🌐 Live at: **https://aayushmiglani.github.io/**

## Structure

```
.
├── index.html                              — homepage / post index
├── 404.html                                — custom 404
└── posts/
    └── correlation-id-spring-boot/
        └── index.html                      — first post
```

Each post lives at its own clean URL (no `.html` suffix) by being an `index.html` inside a folder.

## Adding a new post

1. Create `posts/<slug>/index.html` with the article HTML.
2. Add a `<link rel="canonical" href="https://aayushmiglani.github.io/posts/<slug>/" />` in the `<head>`.
3. Add an entry to the `<ul class="posts">` list in the root `index.html`.
4. Commit and push to `main`. GitHub Pages serves the update within a minute or two.

## Cross-posting (dev.to, Medium)

The canonical `<link>` tag points back to this site. When cross-posting:

- **dev.to** — set `canonical_url:` in the post's front matter to the GitHub Pages URL.
- **Medium** — use the "Import a story" feature with the GitHub Pages URL, which preserves the canonical reference automatically.

SEO weight then accrues to this domain rather than the platform.

## License

Content and code are MIT-licensed. See [`LICENSE`](LICENSE).
