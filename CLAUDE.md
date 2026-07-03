# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

A personal Jekyll blog ("Ggulp's dev blog") built on the **Chirpy** theme (`jekyll-theme-chirpy`, a bundled gem). Content is written in Korean (`lang: ko-KR`).

## Commands

- Serve locally: `bundle exec jekyll serve` (also in `scripts/start_blog.sh`)
- Build: `bundle exec jekyll b`
- CI mirror (catch broken links/images before pushing): `bundle exec jekyll b && bundle exec htmlproofer _site --disable-external`
- First-time asset render needs the submodule: `git submodule update --init`

## Deploy

Push to `main` auto-deploys via GitHub Actions (`.github/workflows/pages-deploy.yml`) — build + html-proofer, then GitHub Pages. **Commit/push only when explicitly asked** (a bad push publishes to the live site). Commit messages: one short line, no signature/trailer.

## Writing posts

- File: `_posts/YYYY-MM-DD-title.md`. Chirpy applies `layout: post`, `comments: true`, `toc: true` by default — omit them.
- Minimal front matter:
  ```yaml
  ---
  title: "..."
  date: 2026-01-05 04:12:41 +0900   # include timezone offset
  categories: [AWS]
  tags: [foo]
  ---
  ```
- Add `render_with_liquid: false` if the body contains literal `{{ }}` / `{% %}` syntax.
- Post images live at `/assets/img/<post-slug>/<image>.png` — absolute paths starting with `/`.

### Import gotchas (posts drafted in Bear/Obsidian)

- The opening `---` must be on its **own line**. Note-app exports sometimes merge it as `---title:` — split it. (This was the `8e1c083` "edited front matter syntax" fix.)
- Convert note-app image embeds to Jekyll: strip `<!-- {"width":N} -->` comments and rewrite `assets/image%20N.png` / spaced filenames to `/assets/img/<slug>/...`, then move the images into `assets/img/<slug>/`. (See the `/new-post` skill.)
- Delete leftover `*.textbundle/` artifacts from `_posts/`.

## Conventions

- `.editorconfig` governs formatting: 2-space indent, LF, UTF-8, final newline. **Do not trim trailing whitespace in `*.md`** (Markdown hard breaks rely on it). Double quotes in YAML, single quotes in JS/CSS/SCSS.
- `Gemfile.lock` and `_site/` are git-ignored (unusual — don't try to commit them).
- Don't change post `permalink` config (Chirpy warns against it in `_config.yml`).
