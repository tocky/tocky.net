# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

`tocky.net` is a personal blog built with [Hugo](https://gohugo.io/) (extended edition). It is a bilingual site — Japanese (`ja`, the default content language) and English (`en`) — using the [Congo](https://github.com/jpanther/congo) theme, which is pulled in as a **Hugo Module** (Go module), not a git submodule.

## Commands

Requires **Hugo extended** and **Go** (for module management). CI pins Hugo `0.165.0` and Go `1.23`. Congo v2.14+ requires Hugo extended `0.158.0`+, so keep local and CI Hugo in sync.

```sh
hugo server -D          # local dev server with drafts, live reload
hugo server             # local dev server, published content only
hugo --minify           # production build into ./public (what CI runs)
hugo new posts/my-post.md   # scaffold a new post from archetypes/default.md (draft: true)
hugo mod get -u         # update the Congo theme module to the latest
hugo mod tidy           # clean up go.mod / go.sum after module changes
```

There is no test suite, linter, or Node build step in this repo.

## Architecture

- **Theme via Hugo Modules.** Congo is imported through `config/_default/module.toml` and declared in `go.mod`. `themes/` is intentionally empty (`.gitkeep` only) — do **not** vendor theme files there. Theme behavior is driven entirely by config and by local overrides.
- **Local overrides win over the theme.** Files placed under `layouts/`, `assets/`, or `static/` shadow the theme's equivalents. Current overrides:
  - `layouts/partials/comments.html` — wires up Disqus (shortname `tocky-net`, set in `config.toml`).
  - `assets/images/logo.png`, `assets/images/author.jpg` — site logo and author avatar referenced from params/languages config.
  - `static/css/user.css` — custom CSS, loaded via `customCss` in `params.toml`.
- **Split config.** Config lives in `config/_default/` split by concern: `hugo.toml` (outputs, permalinks, language declarations), `params.toml` (Congo theme options — color scheme `ocean`, dark default, search, homepage `profile` layout), `languages.{en,ja}.toml` (title, copyright, and the per-language `[params.author]` block), `menus.{en,ja}.toml` (nav/footer menus per language), `taxonomies.toml`, `module.toml`. Edits to any of these change site-wide behavior; keep the `en`/`ja` pairs in sync. Note Congo-specific conventions: the author block lives under `[params.author]` (not top-level `[author]`), and languages use `locale`/`label` (not the deprecated `languageCode`/`languageName`).
- **Bilingual content.** Content files carry a language suffix (e.g. `content/about/_index.ja.md` and `_index.en.md`). When adding or editing content, provide both language variants. `ja` is the default and served at the root.
- **Post permalinks** are `/:year/:month/:slugorfilename/` (see `config.toml`); blog posts go under `content/posts/`.

## Deployment

Pushes to `main` trigger `.github/workflows/hugo.yml`, which builds with `hugo --minify` and deploys to **GitHub Pages**. There is no staging environment — merging to `main` publishes. The workflow checks out submodules recursively and runs `npm ci` only if a lockfile exists (neither is currently used).
