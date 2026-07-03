---
name: new-post
description: Scaffold a new Chirpy blog post from draft notes (often exported from Bear/Obsidian). Creates the dated _posts file with correct front matter, converts note-app image syntax to Jekyll /assets/img paths, and moves referenced images into assets/img/<slug>/. Use when the user wants to publish/add a blog post or turn draft notes into a post.
---

# Scaffold a new blog post

`$ARGUMENTS` is the draft: a path to a note file/folder, pasted content, or a topic. Ask for the draft or title if unclear.

## Steps

1. **Slug & filename.** Derive a URL slug from the title (keep Korean as-is; replace spaces with `-`). Create `_posts/YYYY-MM-DD-<slug>.md` using today's date (or a date the user specifies). Confirm the filename before writing.

2. **Front matter.** Write exactly this shape (`---` on its own line — this is the #1 import bug):
   ```yaml
   ---
   title: "<title>"
   date: <YYYY-MM-DD HH:MM:SS +0900>
   categories: [<Category>]
   tags: [<tag>, ...]
   ---
   ```
   Omit `layout`, `comments`, `toc` (Chirpy defaults). Add `render_with_liquid: false` only if the body has literal `{{ }}` / `{% %}`.

3. **Images.** For every image the draft references:
   - Copy the file into `assets/img/<slug>/` (create the dir).
   - Rewrite the Markdown to `![](/assets/img/<slug>/<filename>)` — absolute path with leading `/`.
   - Strip note-app cruft: `<!-- {"width":N} -->` trailing comments, URL-encoded names (`image%20N.png` → `image N.png`), and Bear/Obsidian embed syntax.

4. **Body cleanup.** Remove `.textbundle` wrappers or export artifacts. Preserve trailing whitespace in the body (Markdown hard breaks; `.editorconfig` intentionally keeps it for `.md`).

5. **Verify.** Suggest `bundle exec jekyll b && bundle exec htmlproofer _site --disable-external` to confirm images/links resolve before the user pushes. **Do not commit or push** unless the user asks (push to `main` auto-deploys the live site).
