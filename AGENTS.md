# AGENTS.md

## What this is

Hugo static site for `www.svenshen.com` (deployed to GitHub Pages). Theme is PaperMod, a git submodule at `themes/PaperMod`. The repo root is the site source.

## Key paths

- `config.yaml` — main Hugo config (baseURL, params, menu, languages, markup)
- `content/posts/` — blog posts (Markdown + YAML front matter)
- `content/about.md`, `content/archives.md`, `content/search.md` — special pages (use `layout: "archives"` / `layout: "search"`)
- `archetypes/default.md` — template for `hugo new posts/xxx.md`
- `static/` — copied as-is to `public/`
- `public/` — build output, gitignored, treat as disposable
- `themes/PaperMod` — upstream submodule, do not edit
- `.github/workflows/hugo.yml` — CI build + deploy

## Bilingual content

English and Chinese posts coexist as sibling files. Chinese versions use the same filename with `.zh.md` suffix (e.g. `totp-explainer.md` + `totp-explainer.zh.md`). All pages have both language variants. The `config.yaml` has `en` and `zh` languages; menus and profileMode are duplicated per language section.

## Commands

```bash
# init submodules after clone
git submodule update --init --recursive

# preview (include drafts)
hugo server -D

# production build (validate all changes)
hugo --gc --minify

# create a new post
hugo new posts/my-post.md
```

## CI and toolchain

- CI uses Hugo extended `0.157.0` on `ubuntu-latest`
- Builds on push to `main` via `hugo --gc --minify --baseURL <pages-url>/`
- No `package.json`, `Makefile`, or any app runner
- No linter, no typechecker, no test suite
- Validation = `hugo --gc --minify`
- CI runs `npm ci` only if a lockfile exists (none does)

## Content conventions

- Front matter keys use lowercase (exception: PaperMod theme keys like `ShowPostNavLinks`, `UseHugoToc`)
- `date` format: ISO 8601 with timezone offset (e.g. `2026-03-09T12:00:00+08:00`)
- ATX headings, fenced code blocks with language tags, Hugo shortcodes (`{{< youtube ... >}}`, `{{< gist ... >}}`)
- New posts: lowercase hyphen-separated filenames in `content/posts/`

## Editing rules

- Never edit `themes/PaperMod` — use site-level overrides before touching the submodule
- Never hand-edit `public/` — it's a build artifact
- Never move content files casually — it changes URLs
- `resources/_gen/` is Hugo cache, also a build artifact
- Content is managed via Codex (per `readme.md`), but direct edits are fine
