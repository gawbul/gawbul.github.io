# AGENTS.md

This file provides guidance for AI agents and coding assistants when working with code in this repository.

## Commands

```bash
hugo                        # Build site into public/
hugo server                 # Serve locally with live reload (http://localhost:1313)
hugo server -D              # Serve including draft posts
pre-commit run --all-files  # Run all linting hooks manually
```

CI builds with `hugo --gc --minify` — the local `hugo` command does not minify.

## Architecture

This is a Hugo static site deployed to GitHub Pages via `.github/workflows/hugo.yaml`. Pushes to `main` trigger an automated build and deploy.

- **`hugo.toml`** — site configuration: base URL, theme, social links, menus, analytics (Google), comments (Disqus)
- **`content/about/index.md`** — the About page (TOML front matter, `+++`)
- **`content/posts/*.md`** — blog posts (YAML front matter, `---`)
- **`themes/hugo-goa/`** — theme, managed as a git submodule; do not edit directly
- **`public/`** — Hugo build output committed to the repo; regenerate with `hugo` after any content change
- **`static/`** — files copied verbatim into `public/` (images, PGP key, etc.)

## Content conventions

Blog post front matter fields: `title`, `author`, `date`, `lastmod`, `description`, `subtitle`, `image`, `images`, `tags`, `categories`, `draft`, `showpagemeta`, `showcomments`.

The About page uses minimal TOML front matter (`date`, `draft`, `title`) with free-form Markdown body. Link non-obvious terms to their Wikipedia articles.

## Pre-commit hooks

`.pre-commit-config.yaml` enforces: trailing whitespace removal, LF line endings, single trailing newline, YAML/TOML validity, no merge conflict markers, and **no direct commits to `main`**. Always work on a feature branch.
