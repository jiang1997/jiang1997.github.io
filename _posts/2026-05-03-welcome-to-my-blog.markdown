---
layout: post
title: "How to write a new post"
date: 2026-05-03 00:00:00 +0800
categories: meta
tags: [jekyll, workflow]
---

The standard loop for adding a post to a Jekyll + Minima blog.

## The 4-step loop

```bash
# 1. Create the file. Name = date + URL slug.
$EDITOR _posts/2026-05-04-thoughts-on-rust.markdown
```

```yaml
# 2. Top of the file — YAML "frontmatter" between two --- lines:
---
layout: post
title:  "Thoughts on Rust"
date:   2026-05-04 14:30:00 +0800
categories: programming
tags: [rust, language-design]
---

The post body in **Markdown** starts here.
```

```bash
# 3. Save. With `jekyll serve --livereload` running on :4000,
#    the site rebuilds in ~200 ms and the browser auto-refreshes.

# 4. Ship it.
git add _posts/2026-05-04-thoughts-on-rust.markdown
git commit -m "Add post: Thoughts on Rust"
git push
```

GitHub Pages picks up the push and rebuilds in 30–90 s.

> **Things worth knowing**
>
> - **The filename is load-bearing.** `YYYY-MM-DD-slug.markdown` isn't a convention — Jekyll *parses it*. The date becomes `page.date` (overridable via the frontmatter `date:` field), the slug becomes `page.slug` and the URL fragment. A file named `notes.md` in `_posts/` is silently skipped, no error.
> - **Frontmatter is YAML, with all of YAML's quirks.** Quote titles containing `:` or starting with `[`, `&`, `*`, `?`. The `date:` value needs the timezone offset, set globally via `_config.yml`'s `timezone:`. If parsing fails, Jekyll prints a quiet warning and skips the post.
> - **`layout: post`** wraps the body in Minima's `_layouts/post.html` (title header, date, prev/next nav). Without it, you get raw content with no styling.

## Drafts (work in progress)

Posts not yet ready to publish go in `_drafts/` instead of `_posts/`, and don't need a date in the filename:

```bash
mkdir -p _drafts
$EDITOR _drafts/half-baked-idea.markdown
```

Preview drafts locally — restart the server with `--drafts`:

```bash
bundle exec jekyll serve --livereload --drafts
```

When ready, move and rename:

```bash
mv _drafts/half-baked-idea.markdown _posts/2026-05-10-half-baked-idea.markdown
```

> Drafts get `page.date` set to the file's mtime, so they appear at the top of the local index until renamed. Without `--drafts`, Jekyll ignores `_drafts/` entirely — both locally and on GitHub Pages — so it's safe to commit drafts as backups, though most people gitignore `_drafts/`.

## Images and other assets

Drop them in any **non-underscored** directory. The convention is `assets/img/`:

```bash
mkdir -p assets/img
cp ~/Pictures/diagram.png assets/img/
```

Reference in a post:

{% raw %}
```markdown
![Architecture diagram]({{ "/assets/img/diagram.png" | relative_url }})
```
{% endraw %}

> The `relative_url` Liquid filter prepends `site.baseurl`. For a *user site* the baseurl is empty, so `/assets/img/diagram.png` resolves directly. But using the filter anyway means the same code works if the post is copied to a *project site* (where `baseurl` is `/repo-name`). Cheap insurance.
>
> **Underscored directories are reserved by Jekyll** (`_posts`, `_drafts`, `_layouts`, `_includes`, `_data`, `_site`). Don't create your own — they'll be either consumed or excluded.

## Categories vs. tags — the practical difference

| | Affects URL? | Generates archive page? | Typical use |
|--|--|--|--|
| `categories` | **Yes** — prepended to permalink | No (in Minima) | High-level groupings: *programming*, *journal*, *meta* |
| `tags` | No | No (in Minima) | Cross-cutting topics: *rust*, *kubernetes*, *book-review* |

Because categories show in the URL, **changing them later breaks links**. Tags are cheap to revise.

To get clean URLs without category prefixes, override the permalink globally in `_config.yml`:

```yaml
defaults:
  - scope: { type: posts }
    values:
      permalink: /:year/:month/:day/:title/
```

Or per-post via frontmatter `permalink: /thoughts-on-rust/`.

## Common pitfalls

| Symptom | Cause |
|--|--|
| Post doesn't appear | Filename missing `YYYY-MM-DD-` prefix, or date is in the future |
| "Future" post hidden | Add `--future` to `jekyll serve`, or move the date back |
| Page renders without styling | Missing `layout: post` |
| Build error: `did not find expected key` | YAML frontmatter has an unquoted `:` or stray indentation |
| Site live but local doesn't update | LiveReload connection dropped — hard-refresh once |
| Push succeeds but site stale | Bad Liquid syntax silently fails the build; check the latest Pages build status with `gh api repos/<owner>/<repo>/pages/builds/latest` |

## A helper, if posts come often

Drop this in `~/.zshrc` (or `~/.bashrc`):

```bash
newpost() {
  local slug="${1:?usage: newpost slug-words [title]}"
  local title="${2:-${slug//-/ }}"
  local date="$(date +%Y-%m-%d)"
  local file="_posts/${date}-${slug}.markdown"
  cat > "$file" <<EOF
---
layout: post
title: "${title}"
date: $(date '+%Y-%m-%d %H:%M:%S %z')
categories: meta
---

EOF
  ${EDITOR:-vi} "$file"
}
```

Then `newpost thoughts-on-rust "Thoughts on Rust"` scaffolds the file with the right date and timezone, and opens it in your editor.
