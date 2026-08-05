# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
zola serve        # local dev server with live reload
zola build        # build into public/
zola check        # check links and templates
```

## Architecture

Static site built with [Zola](https://www.getzola.org/). Default language is **Russian**; English is a secondary language at `/en/`.

**Content files:**
- `content/_index.md` — Russian homepage: intro + list of all posts (root section)
- `content/_index.en.md` — English homepage: intro + list of all posts
- Posts: `content/slug.md` (Russian) + `content/slug.en.md` (English)

**Templates** (`templates/`): `base.html` → layout with lang switcher; `index.html` → homepage intro + post list (renders `section.pages`); `page.html` → post.

**Styles:** `sass/main.scss` compiled automatically by Zola.

**Deployment:** push to `main` → GitHub Actions uploads `public/` to GitHub Pages. The `public/` directory must be committed — CI does not rebuild, it only deploys what's already there.

## Multilingual conventions

- Russian content: `filename.md`, accessed at `/slug/`
- English content: `filename.en.md`, accessed at `/en/slug/`
- The `lang` variable is available in templates (`{% if lang == "en" %}`)
- Root sections (`content/_index.md` / `_index.en.md`) have `sort_by = "date"` so posts list in reverse-chronological order
