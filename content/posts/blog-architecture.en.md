---
title: "How This Blog Is Built — Running Hugo and Congo with Hugo Modules"
date: 2026-08-25T13:30:00+09:00
draft: false
categories: ["Tech"]
tags: ["Hugo", "Congo", "GitHub Actions", "Static Sites"]
---

For the first post, I want to talk about the blog itself.

What I picked, why, and how the whole thing runs today. The one part I especially want to cover is installing the theme as a Hugo Module — there still isn't much written about it in Japanese, and it's the kind of thing that saves you a lot of trouble later, so I'll walk through it with the actual config files as they are.

The first half is about the choices I made; the second half gets into the implementation.

## What I actually want from a personal site

When I boil it down, there are only three things I want from a personal site.

First, being able to focus on writing. I don't want to log into an admin panel every time I write a post, fight with the editor's quirks, and then find that the preview doesn't match production. What I want to produce is text, and the fewer steps around that, the better.

Second, it has to last. A personal site is worth more for being readable five years from now than for the moment I hit publish. I don't want to put it somewhere that disappears when the service shuts down, or somewhere that locks my data into a proprietary format.

And third, it has to be low-maintenance. I don't want to babysit a server, and I don't want to be up at night applying security patches. I'd also rather not commit to paying a monthly fee forever.

Line up those three, and the options narrow down fast. Keep the Markdown as files, generate static HTML, and put it somewhere. That's a static site generator.

## Why Hugo

There are plenty of static site generators, but my reason for choosing Hugo was simple.

The deciding factor was that everything fits in a single binary. Hugo is written in Go, and all you need to run it is the `hugo` command. It doesn't create a `node_modules`, and a package manager's mood swings won't break your build. This repository doesn't even have a `package.json`. Not having to "fix the dependencies first" when I come back to it after a couple of years is worth more than you'd think for a personal site.

The build speed mattered too. Building this entire site locally takes under a second, and the preview server reflects changes instantly. Not being made to wait while writing feeds directly into how much I actually write.

And the output is just HTML. The result is a pile of static files, so it can live anywhere. Right now it's on GitHub Pages, but if I change my mind I can move it elsewhere. Weak lock-in is a big deal if you plan to keep something for a decade.

There's one prerequisite. **Hugo comes in a standard and an extended build, and this site requires the extended build.** That's because the Congo theme (more on it below) needs Sass compilation, which the standard Hugo can't do. The `hugo` you get from Homebrew is the extended build, so I never think about it day to day, but in CI I explicitly pull the extended binary.

## Why Congo

I'm using the Congo theme. The fastest way to explain it is to work backwards from the features I actually use.

`config/_default/params.toml` doubles as a list of exactly what I'm asking the theme to do.

```toml
colorScheme = "ocean"
defaultAppearance = "dark"
enableSearch = true
enableCodeCopy = true

# User defined CSS
customCss = ["css/user.css"]

[header]
    logo = "images/logo.png"

[footer]
    showAppearanceSwitcher = true

[homepage]
    layout = "profile"

[article]
    showAuthor = false
    showBreadcrumbs = true
    showEdit = false
    editURL = "https://github.com/tocky/tocky.net/tree/main/content/"
    showReadingTime = false
    showTableOfContents = true
    showTaxonomies = true
    showComments = false

[list]
    showBreadcrumbs = true
    showTableOfContents = true
    showTaxonomies = true
```

Reading top to bottom, it's a list of the things I wanted.

- `homepage.layout = "profile"` — makes the homepage a profile page rather than a list of posts. For a personal site, I wanted the first thing you see to be "who's writing this"
- `enableSearch` — full-text search. On the `hugo.toml` side I add `JSON` to the `home` output so it emits an index
- `enableCodeCopy` — adds a copy button to code blocks. Non-negotiable if you're planning to write technical posts
- `showTableOfContents` — a table of contents. I intended to write long posts
- `defaultAppearance = "dark"` and `showAppearanceSwitcher` — dark by default, but readers can switch
- `colorScheme = "ocean"` — the color scheme

Then there's multilingual support. Congo lets you cleanly split settings and menus per language. I planned to run this in both Japanese and English, so a theme that doesn't fall apart here was a requirement.

I set `showEdit = false` but still filled in `editURL` so that I can turn on an "Edit on GitHub" link whenever I want — it's just disabled for now.

## Why install the theme as a Hugo Module instead of a git submodule

This is the part I most wanted to write about.

For a long time, the standard way to install a Hugo theme was to drop it under `themes/` as a git submodule. The docs and tutorials mostly describe it that way. But this site has used **Hugo Modules** for the theme from the start.

Hugo Modules is a mechanism that uses Go's module system directly, treating the theme as a dependency. The config is just two lines in `config/_default/module.toml`.

```toml
[[imports]]
path = "github.com/jpanther/congo/v2"
```

That's it. The version lives in `go.mod`.

```
module github.com/tocky/tocky.net

go 1.23

require github.com/jpanther/congo/v2 v2.14.0 // indirect
```

And here's what's inside the `themes/` directory.

```
themes/
└── .gitkeep
```

**Not a single theme file is in the repository.** The directory is empty except for a `.gitkeep`, which exists only because Git won't track an empty directory — there's nothing else. This repository has never once had a `.gitmodules`.

This approach has advantages that a submodule doesn't.

**The version is pinned explicitly.** A submodule pins to a commit hash too, but `v2.14.0` written in `go.mod` tells you at a glance what you're using. And because `go.sum` holds the hashes, it also verifies that what you fetched hasn't been tampered with.

**Cloning is straightforward.** With a submodule, forget `git clone --recursive` and you're left with an empty directory and a build that fails with a cryptic error. With Hugo Modules the build fetches everything automatically, so the clone instructions don't need any caveats.

**Updating is two commands.** Bringing the theme up to date is just this:

```sh
hugo mod get -u   # update modules to the latest
hugo mod tidy     # tidy up go.mod / go.sum
```

The diff shows up in a few lines of `go.mod` and `go.sum` and nowhere else. It's an easier change to review than bumping a submodule pointer.

There's a cost to weigh, though. Hugo Modules **requires the Go toolchain**. Both local and CI need Go, and this repository uses Go 1.23. You give up the "all you need is Hugo" simplicity. Even so, I decided that not carrying the theme around as my own code pays off in the long run.

## Why I keep local overrides to a minimum

Hugo prefers files you place in the project's `layouts/`, `assets/`, and `static/` over the theme's files at the same path. It's a handy way to swap out just a piece of the theme, and this site uses it.

But override too much and you're guaranteed to get stuck.

An overridden file stays frozen even when the theme moves on. Let copies that no longer track the theme pile up locally, and the moment you bump the version, the layout breaks. And the last thing you'll suspect is that the culprit is "a file I copied a couple of years ago."

This site went through exactly that. At one point I had a full 118-line copy of the theme's template sitting in `layouts/partials/head.html`. I think I copied it to make one small change, but when the theme updated, that one file got left behind. In the end I deleted the whole thing and went back to the theme's own implementation.

So now I override only the three things that genuinely need it.

```
layouts/partials/comments.html   # enable Disqus
assets/images/logo.png           # header logo
assets/images/author.jpg         # profile image
static/css/user.css              # tweaks for Japanese
```

The contents of `comments.html` are a single line.

```go-html-template
{{ template "_internal/disqus.html" . }}
```

Rather than reading the theme's template, understanding it, and replacing it, **I put the smallest possible snippet into a hook the theme already provides.** As long as I keep that distance, theme updates don't cause me trouble.

## Why I split the config

Hugo lets you put every setting in a single `hugo.toml`, or split it across `config/_default/`. This site splits it.

```
config/_default/
├── hugo.toml            # outputs, permalinks, language declarations
├── params.toml          # Congo theme settings
├── module.toml          # where the theme comes from
├── taxonomies.toml      # category / tag definitions
├── languages.ja.toml    # Japanese site info / author info
├── languages.en.toml    # English version
├── menus.ja.toml        # Japanese navigation
└── menus.en.toml        # English version
```

Splitting it paid off mainly in two ways.

One is that **I never wonder where the thing I want to change lives.** Want to tweak the theme's appearance? That's `params.toml`. Want to add a nav item? That's `menus.*.toml`. When it's all in one file, you're scanning a 200-line file from the top.

The other is the real win: **the multilingual files pair up.** `languages.ja.toml` and `languages.en.toml`, `menus.ja.toml` and `menus.en.toml` sit side by side with the same structure, so it's easy to spot when I've edited one and left the two out of sync. Mix language-specific sections together in a single file and that gets a lot harder.

## Why GitHub Pages + Actions

Hosting is GitHub Pages, and the build runs on GitHub Actions.

The main reason is that it costs nothing. I couldn't get myself to keep paying a monthly bill to run a personal site. If you plan to keep it going for ten years, zero fixed cost is worth a lot.

The operating rule is simple too: whatever lands on `main` becomes production. There's no staging environment. With exactly one rule, I never have to wonder "does this go live?" even when I come back after a long break.

Running only the build on PRs is something I added later. Before anything hits `main`, I can confirm that at least the build passes.

---

That's the reasoning behind the choices. From here, we get into the implementation.

## The directory layout at a glance

Here's the repository as a whole.

```
tocky.net/
├── .devcontainer/
│   └── devcontainer.json     # Go 1.23 + Hugo extended 0.165.0
├── .github/workflows/
│   └── hugo.yml              # build and deploy to GitHub Pages
├── archetypes/
│   └── default.md            # template for hugo new
├── assets/images/            # logo and profile image
├── config/_default/          # the split config (above)
├── content/
│   ├── about/
│   │   ├── _index.ja.md
│   │   └── _index.en.md
│   └── posts/                # posts
├── data/
├── layouts/partials/
│   └── comments.html         # the only template override
├── static/
│   ├── css/user.css
│   └── favicon files
├── themes/
│   └── .gitkeep              # empty. the theme isn't here
├── go.mod                    # the theme's version lives here
└── go.sum
```

As you can see, **the theme itself is nowhere to be found.** The `github.com/jpanther/congo/v2 v2.14.0` written in `go.mod` gets loaded from Go's module cache at build time. Clone the repository, run `hugo`, and everything you need shows up automatically.

## Inside the config files

### hugo.toml

This defines the site's overall skeleton.

```toml
defaultContentLanguage = "ja"
disqusShortname = "tocky-net"

[outputs]
    home = ["HTML", "RSS", "JSON"]

[permalinks]
    posts = "/:year/:month/:slugorfilename/"

[languages]
    [languages.en]
        [languages.en.params]
            displayName = "🇺🇸"
            description = "Technology, software, development, cloud, and everything else I'm interested in"
    [languages.ja]
        [languages.ja.params]
            displayName = "🇯🇵"
            description = "テクノロジー、ソフトウェア、開発、クラウドなど私が興味のあることすべて"
```

The `JSON` in `outputs` is for the search index. Congo's full-text search reads it, so if you're going to set `enableSearch = true`, you need this as a pair.

`permalinks` is the URL rule for posts, and I've set it to `/:year/:month/:slugorfilename/`. For this post that becomes `/2026/08/blog-architecture/`. Putting the year and month in the path makes collisions less likely if I ever want to restructure things later.

The two-level `[languages.*.params]` is the result of following a Hugo spec change. It used to be that you could write `displayName` and `description` directly under the language, but now they go under `params`.

### languages.ja.toml

```toml
locale = "ja"
label = "日本語"
title = "tocky.net"
copyright = "&copy; tocky.net"

[params.author]
    name = "Shin Tokiwa"
    image = "images/author.jpg"
    headline = "A developer who loves technology"
    bio = "..."
    links = [
        { github = "https://github.com/tocky" },
    ]
```

There are two easy things to get wrong here.

One is putting the author info under `[params.author]` — not a top-level `[author]`. Get this wrong and Congo emits a warning, and depending on the version, the build itself stops.

The other is using `locale` and `label` to specify the language. The old `languageCode` / `languageName` are deprecated. Copy an old post or an old theme example verbatim and you'll get a warning at build time.

### menus.ja.toml

```toml
[[main]]
    name = "Blog"
    pageRef = "posts"
    weight = 10

[[main]]
    name = "About"
    pageRef = "about"
    weight = 30

[[footer]]
    name = "Categories"
    pageRef = "categories"
    weight = 10

[[footer]]
    name = "Tags"
    pageRef = "tags"
    weight = 20
```

The key point is writing `pageRef` instead of `url` — that way Hugo resolves the per-language paths (`/posts/` and `/en/posts/`) for you. I spread `weight` out to 10 and 30 so I can slot something in between later.

### taxonomies.toml

```toml
category = "categories"
tag = "tags"
```

Just two taxonomies: categories and tags. You can add more, but adding classification axes to a personal blog is more than I can keep up with, so I've stuck with these two.

## How the content is organized

Japanese is the default language, and `defaultContentLanguage = "ja"` puts it at the root (`/`). The English version lives under `/en/`.

Content files split by language using a **language suffix**.

```
content/about/_index.ja.md   → /about/
content/about/_index.en.md   → /en/about/
```

Put `.ja.md` and `.en.md` in the same directory and Hugo recognizes them as translations of the same page, wiring up the language-switch link between them. For a page that only exists in one language — like this post — switching languages just takes you back to the homepage.

The post template lives in `archetypes/default.md`.

```markdown
---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
draft: true
---
```

`hugo new posts/my-post.md` generates a draft in this shape. Since it defaults to `draft: true`, a work-in-progress never gets published by accident.

## Tweaks for Japanese

The `static/css/user.css` loaded via `customCss` in `params.toml` is essentially just this.

```css
@import url('https://fonts.googleapis.com/css2?family=Ubuntu&display=swap');

html {
    font-family: "Ubuntu", "Helvetica Neue", "Helvetica", "Hiragino Sans", "Hiragino Kaku Gothic ProN", "Arial", "Yu Gothic Medium", "Meiryo", sans-serif;
    font-feature-settings: "palt";
}
```

It does two simple things.

**Use different fonts for Latin and Japanese text.** Ubuntu goes first for Latin characters, followed by a list of Japanese fonts. The browser walks the list front to back looking for a font that has each character, so Latin lands on Ubuntu and Japanese lands on Hiragino or Yu Gothic — the split happens on its own.

**`font-feature-settings: "palt"` tightens up the punctuation.** By default, Japanese punctuation and brackets carry too much whitespace on either side and the text looks loose. Enable `palt` (proportional metrics) and each character snaps to a proper width. A single line noticeably changes how Japanese looks, so it's well worth having on any site that handles Japanese text.

## The key points in the GitHub Actions workflow

From `.github/workflows/hugo.yml`, just the parts worth knowing.

### Pin every version

```yaml
env:
  DART_SASS_VERSION: 1.102.0
  GO_VERSION: "1.23"
  HUGO_VERSION: 0.165.0
  TZ: Asia/Tokyo
```

Hugo, Go, and Dart Sass all get a pinned version rather than `latest`. And since they're all gathered in one `env` block, bumping them means looking in just one place.

On top of that, **I keep the Hugo version aligned across local, devcontainer, and CI.** `.devcontainer/devcontainer.json` pins the same 0.165.0. That kills the dullest failure of all — it works on my machine but breaks in CI.

### Set TZ explicitly

```yaml
TZ: Asia/Tokyo
```

For a site using date-based permalinks, this quietly matters. GitHub Actions runners default to UTC, so a post written before 9 a.m. Japan time is treated as the previous day in CI. As long as `/:year/:month/` is in the URL, that means the URL changes.

### Build on PRs, deploy only on main

```yaml
on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]
```

```yaml
deploy:
  if: github.event_name != 'pull_request'
```

The build job runs on PRs too, but the `deploy` job doesn't run on `pull_request` events. "I want to confirm the build passes before merging, but I don't want to publish" — expressed in a single condition.

### Carry the build cache across runs

```yaml
- name: Build with Hugo
  run: |
    hugo build \
      --gc \
      --minify \
      --baseURL "${{ steps.pages.outputs.base_url }}/" \
      --cacheDir "${{ runner.temp }}/.cache/hugo"
```

I set `--cacheDir` explicitly and put it on `actions/cache`. Things like image-processing results get reused, which affects build time once there are a lot of posts. `--gc` cleans up cache entries that are no longer used.

I pass `steps.pages.outputs.base_url` to `--baseURL` to tell the build the correct public URL that GitHub Pages holds. Get this wrong and every link inside the generated HTML ends up broken.

## Gotchas

Finally, a few things I actually got tripped up by.

### Waiting forever on `.hugo_build.lock`

I hit this one for real while writing this very post.

During a build, Hugo creates a file called `.hugo_build.lock` at the project root and takes an exclusive lock. Normally it's a mechanism you never have to think about, but **if `hugo server` freezes partway through its initial build for some reason, the process stays around still holding the lock.**

Run another build in that state, and you get this.

```
hugo: collected modules in 819 ms
Start building sites … 
hugo v0.165.0+extended+withdeploy darwin/arm64
```

It goes no further. No error. No timeout. It just sits there, silently waiting.

The nasty part is that the cause lies outside the command you're currently running. Rerun it and it stops at the same place every time, so you go off investigating in the wrong direction entirely — suspecting the config, clearing the cache.

The answer was to look at the processes.

```sh
ps aux | grep hugo
```

There was a `hugo server` that had been started hours earlier and was stuck. I killed it, and the next build finished in a second.

**When a build stops silently, suspect a leftover process first.** Just knowing this means you can get out of it fast the next time you hit it.

### Congo requires Hugo extended

Congo declares this itself.

```toml
[hugoVersion]
  extended = true
  min = "0.87.0"
```

The standard Hugo can't build it. Homebrew's `hugo` is the extended build, so you won't notice day to day, but when you fetch the binary yourself in CI, you have to pick the one whose filename starts with `hugo_extended_`.

### Don't take the theme's declared minimum version at face value

Look at that declaration again. It says `min = "0.87.0"`.

But Congo v2.14.0's CHANGELOG says this:

> ⚠️ Required Hugo version is now 0.158.0 or later

**The declared value and the version actually required don't match.** Look at the theme's `module.toml`, conclude "so it works on 0.87.0," and it won't actually work.

When you bump a theme, read the CHANGELOG rather than the declaration. Congo's CHANGELOG in particular is easy to follow, since breaking changes are marked with ⚠️. The v2.14.0 entry lists, alongside that Hugo version requirement, "fixed an issue where the author-settings warning would halt the site build" — which ties back to the `[params.author]` point from earlier.

## Rebuilt with AI

Worth noting: I've been doing this overhaul together with AI.

The bulk dependency updates, the migration off deprecated settings, and the research that this post is built on — I did all of it with Claude Code. Look at the git history and you'll find commits carrying `Co-Authored-By: Claude` still there. Work where the research load dominates over the judgment calls — "read the CHANGELOG, list the deprecated items, fix them one by one" — is especially fast to hand off.

That's a different subject, so I'll save it for another post. I've also been pushing AI deep into real work on the job, so I've got plenty to write about.

## What's coming

So that's how this blog runs.

What I want to write next is how I fold AI into real work. And then, some thoughts from running a software consultancy for 18 years. I'll write about whatever catches my interest, from both the technical and the business side.
