# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Local dev server (live reload)
hugo server

# Production build
hugo --gc --minify

```

Hugo requires the extended version. The project uses `mise` for tool management (`mise.toml` pins `hugo-extended = "latest"`).

The theme is a git submodule; first-time clone needs `git submodule update --init --recursive`.

## Architecture

Hugo static site with the PaperMod theme (git submodule at `themes/PaperMod`). Built output goes to `public/` (not committed). Hosted on GitHub Pages.

**Content** lives in `content/` and is bilingual (English + Italian). English posts are `filename.md`; Italian translations are `filename.it.md`. Content sections: `posts/`, `projects/`, `about`, `archives`, `search`.

**Layouts** in `layouts/` override PaperMod defaults. Custom CSS is in `assets/css/extended/custom.css` — the `extended/` subdir appends to PaperMod's base CSS rather than replacing it. The custom `layouts/partials/header.html` adds scroll-position restoration on language switch; `layouts/partials/templates/schema_json.html` overrides PaperMod's JSON-LD schema partial.

**Config** is `hugo.yaml` — single file for all Hugo config including both language menus, PaperMod params, and output formats. The `search` page uses client-side Fuse.js against the JSON output format (`outputs.home: [HTML, RSS, JSON]`); tune matching via `params.fuseOpts`.

**CI/CD** (`.github/workflows/deploy.yml`) checks out submodules recursively, builds with `hugo --gc --minify`, and deploys to GitHub Pages on every push to `main` (ignoring README/LICENSE-only changes).

## Bilingual content

New posts need both an English file and an `.it.md` translation. Pages without a translation fall back to the default language automatically.

New posts can be scaffolded with:

```bash
hugo new posts/my-post.md          # English
hugo new posts/my-post.it.md       # Italian
```

Both start as `draft: true`. Remove that front matter key (or set it to `false`) before committing if the post should publish.

Common optional front matter fields: `tags`, `description`, `ShowToc: true`.

Because `defaultContentLanguageInSubdir: true`, all URLs are prefixed by language code (`/en/posts/...`, `/it/posts/...`). Use Hugo's `relLangURL` / `absLangURL` for internal links rather than hardcoded paths.

## Deployment

Push to `main` triggers the GitHub Actions workflow which builds the site and deploys it to GitHub Pages at `https://thewalterman.github.io/`.
