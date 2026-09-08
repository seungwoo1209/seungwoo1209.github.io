# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

A personal Jekyll blog ("sw's blog") built on the **Chirpy** theme (`jekyll-theme-chirpy`, a bundled gem). Content is written in Korean (`lang: ko-KR`). The theme's own files are not in this repo — see "Overriding the theme" below.

## Commands

- Serve locally: `bundle exec jekyll serve` (also in `scripts/start_blog.sh`)
- Build: `bundle exec jekyll b`
- CI mirror (catch broken links/images before pushing): `bundle exec jekyll b && bundle exec htmlproofer _site --disable-external`
- First-time asset render needs the submodule: `git submodule update --init`

## Deploy

Push to `main` auto-deploys via GitHub Actions (`.github/workflows/pages-deploy.yml`) — build + html-proofer, then GitHub Pages. **Commit/push only when explicitly asked** (a bad push publishes to the live site). Commit messages: one short line, no signature/trailer.

## Site structure — the root URL is remapped

The site root serves the **category index**, not the post list (`e4a4ecf`). Four files are load-bearing here; changing one alone breaks the others:

- `_tabs/categories.md` — `permalink: /` moves the category tab to the root.
- `posts/index.html` — the post list, displaced to `/posts/`. It is a plain page rather than a tab because `jekyll-paginate` only paginates an `index.html` in `site.pages`; `_config.yml` points `paginate_path` at it.
- `_includes/sidebar.html` — hand-inserts the "전체 글" link after the category tab, since the post list is no longer a tab.
- `_includes/topbar.html` — labels the root breadcrumb from `page.title`, and skips the duplicate `categories` crumb on category/tag pages.

Do not change the **post** `permalink` in `_config.yml` (Chirpy warns against it); the tab permalink above is the deliberate exception.

## Overriding the theme

The theme is a gem, so customization works by **shadowing its files** — a file at the same path in this repo wins. Current overrides:

- `_includes/sidebar.html`, `_includes/topbar.html` — see above.
- `_includes/metadata-hook.html` — Chirpy's own empty extension point at the end of `<head>`; loads the web fonts. Preferred over copying `_data/origin/cors.yml`, which would freeze the ~20 other CDN versions pinned in that file.
- `assets/css/jekyll-theme-chirpy.scss` — custom styles. Fonts are IBM Plex Sans KR (text) + JetBrains Mono (code) (`3a5f26b`); Chirpy's defaults carry no Hangul glyphs, so Korean text fell back to whatever font the visitor's OS shipped.

`$font-family-base` cannot be set through `@use 'main' with (...)` — `_sass/main.scss` never forwards `abstracts` — so that scss overrides the *compiled* declarations selector by selector. **After a theme upgrade, re-check that selector list**; the verification command is in the file's own comment.

PWA is enabled but its offline cache is off on purpose (`8b28903`): the service worker served cached pages cache-first without revalidating, so edits kept showing the stale version.

## Writing posts

**`/new-post` (`.claude/skills/new-post/`) is the source of truth** for turning a draft into a post — slug rules, image conversion, and note-app import quirks. Drafts land in `sources/`, a git-ignored drop folder, as either a Notion export zip or a Bear/Obsidian `.md` plus its image folder.

- File: `_posts/YYYY-MM-DD-<slug>.md`. Chirpy applies `layout: post`, `comments: true`, `toc: true` by default — omit them.
- Front matter:
  ```yaml
  ---
  title: "..."
  date: 2026-01-05 04:12:41 +0900   # include timezone offset
  categories: [Worklog]             # exactly one of Project / Worklog / Knowledge
  tags: [AWS, S3]                   # subject first
  ---
  ```
- **Categories are a single level, and exactly three exist** — `Project`, `Worklog`, `Knowledge` — chosen by the post's *nature*, not its subject (`9efecb7`). The subject goes in `tags`, as the first tag (`a91d449`): Jekyll categories are one flat global namespace, so a subject reused under two natures would collide into a single `/categories/<subject>/` page.
- Add `render_with_liquid: false` if the body contains literal `{{ }}` / `{% %}` syntax.
- Post images live at `/assets/img/<post-slug>/<image>.png` — absolute paths starting with `/`.

### Import gotchas

- The opening `---` must be on its **own line**. Note-app exports sometimes merge it as `---title:` — split it. (This was the `8e1c083` "edited front matter syntax" fix.)
- Strip `<!-- {"width":N} -->` comments, rewrite spaced or URL-encoded image paths, and move the images into `assets/img/<slug>/`.
- A Notion title containing parens — `Associate(SAA-C03)` — leaks literal `(` `)` into the URL-encoded image path, so `[^)]*` patterns break on it. Match the image ref line-wise instead.
- Remove the leading `# <title>` H1 (Chirpy renders the title from front matter) and any leftover `*.textbundle/` artifacts.

## Conventions

- `.editorconfig` governs formatting: 2-space indent, LF, UTF-8, final newline. **Do not trim trailing whitespace in `*.md`** (Markdown hard breaks rely on it). Double quotes in YAML, single quotes in JS/CSS/SCSS.
- `Gemfile.lock` and `_site/` are git-ignored (unusual — don't try to commit them).
