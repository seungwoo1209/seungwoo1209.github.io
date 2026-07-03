---
name: new-post
description: Scaffold a new Chirpy blog post from draft notes (usually a Notion export zip under sources-zip/, sometimes Bear/Obsidian). Creates the dated _posts file with correct front matter, converts note-app image syntax to Jekyll /assets/img paths, and moves referenced images into assets/img/<slug>/. Use when the user wants to publish/add a blog post or turn draft notes into a post.
---

# Scaffold a new blog post

**Default source:** unless the user says otherwise, the draft starts from a zip under `sources-zip/` (a Notion "Export block" archive). `$ARGUMENTS` may instead be a path to a note file/folder, pasted content, or a topic. If `sources-zip/` has no zip and no other draft is given, ask for the draft or title.

## Steps

0. **Extract the Notion zip** (when starting from `sources-zip/`).
   - The archive is a **zip-in-a-zip**: the outer zip contains an inner `...-Part-1.zip`. Extract the outer first, then the inner.
   - Korean filenames break `unzip` ("Illegal byte sequence") — use macOS `ditto` to preserve UTF-8 names:
     ```bash
     ditto -x -k "<inner>-Part-1.zip" <destdir>
     ```
   - The `.md` file inside is the draft; its sibling folder holds the images. If multiple zips exist, ask which one (or process the newest). Extract to the scratchpad, not the repo.

1. **Slug & filename.** Derive a URL slug from the title: **keep Korean as-is**, lowercase Latin letters, replace spaces with `-`, and drop punctuation (`+ : ? , . ( )` etc.). E.g. `"CloudWatch Logs에서 ... export하는 방법 + ... 권한설정"` → `cloudwatch-logs에서-...-export하는-방법-최소-권한-원칙으로-s3-권한설정`. Create `_posts/YYYY-MM-DD-<slug>.md` using today's date (or a date the user specifies). Confirm the filename before writing. (This slug also becomes the post URL `/posts/<slug>/` and the image folder — see step 3.)

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
   - Copy the file into `assets/img/<slug>/` (create the dir). **Rename to remove spaces** — `image 1.png` → `image-1.png` — so paths need no URL-encoding.
   - Rewrite the Markdown to `![](/assets/img/<slug>/<filename>)` — absolute path with leading `/`, empty alt text. Notion refs look like `![image.png](%EC%BB.../image%201.png)`; replace the whole thing with the clean absolute path.
   - Strip note-app cruft: `<!-- {"width":N} -->` trailing comments and Bear/Obsidian embed syntax.

4. **Body cleanup.** Remove the leading `# <title>` H1 — Chirpy renders the title from front matter, so a body H1 duplicates it. Remove `.textbundle` wrappers or export artifacts. Preserve trailing whitespace in the body (Markdown hard breaks; `.editorconfig` intentionally keeps it for `.md`).

5. **Verify.** Suggest `bundle exec jekyll b && bundle exec htmlproofer _site --disable-external` to confirm images/links resolve before the user pushes. **Do not commit or push** unless the user asks (push to `main` auto-deploys the live site).
