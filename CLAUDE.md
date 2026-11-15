# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
pnpm run build         # regenerate index.html, styles.css, script.js from content.yaml
pnpm run lint          # lint HTML, CSS, and JS
pnpm test              # run Playwright E2E tests (headless Chromium)
pnpm exec serve . -l 3000  # local dev server on port 3000
```

CI (`.github/workflows/ci.yml`) runs build → lint → tests on every push/PR to `main`. On push to `main`, a second `deploy` job (gated on the first passing) assembles `index.html`, `styles.css`, `script.js`, `CNAME`, `assets/` and `fonts/` into `_site/` and publishes it via GitHub Actions Pages (Settings → Pages → Source must be "GitHub Actions").

## Architecture

Single-page static site — a bakery order form. Content is driven by **`content.yaml`** (the master config); `build.js` renders that through Handlebars templates in `templates/` into the files that ship.

- `content.yaml` — **source of truth** for all site content (menu, holiday specials, theme colors, pickup schedule, payment info, copy). Alyssa edits only this file.
- `templates/index.html.hbs`, `templates/styles.css.hbs`, `templates/script.js.hbs` — Handlebars templates. Structure lives here; content comes from `content.yaml`.
- `build.js` — validates `content.yaml` against `schema.js`, renders templates, writes `index.html` / `styles.css` / `script.js`, and stamps them with a content-hashed cache-bust version.
- `schema.js` — Ajv JSON Schema for `content.yaml`. Build fails with a human-readable error if the config doesn't match.
- `index.html`, `styles.css`, `script.js` — **generated artifacts**. Gitignored and not committed — CI builds and deploys them. Never hand-edit; the next build overwrites them. Run `pnpm run build` locally to produce them for dev/tests. Each file starts with an `AUTO-GENERATED` header.
- `tests/order-form.spec.js` — Playwright E2E tests against `http://localhost:3000`. Reads `content.yaml` so it adapts when the menu changes.

Payment QR codes (`venmo-qr.png`, `zelle-qr.png`) are embedded directly in the repo; their filenames are referenced from `content.yaml` under `payment`.

Linting is configured via `eslint.config.mjs` (JS), `.stylelintrc.json` (CSS), and `.htmlhintrc` (HTML). `build.js` and `schema.js` are ignored by ESLint (they're Node scripts, not shipped to the browser).

CSS rules must never have multiple declarations on a single line — stylelint enforces `declaration-block-single-line-max-declarations: 1`.

## Workflow

**To change site content (holiday specials, prices, menu, colors, etc.):**

1. Edit `content.yaml`.
2. Commit and push to `main`. The GitHub Action builds, tests, and deploys the site to GitHub Pages.

If the YAML is malformed or missing required fields, the build fails, the deploy job never runs, and the previous deployment stays live — the site never goes into a broken state.

**To change site structure (form fields, layout, CSS rules):** edit the relevant `templates/*.hbs`, then run `pnpm run build` locally to verify.

## Cache Busting

`build.js` computes a short sha256 hash of each generated file's contents and injects it as `?v=<hash>` in the `<link>` / `<script>` tags of `index.html`. No manual version bumps — the hash changes automatically when content changes.

## Deployment

The site is published to GitHub Pages from `muddymoose/muddymoose.github.io` via the `deploy` job in CI (Pages source: GitHub Actions). Only files copied into `_site/` by the "Assemble site" step are served — add new static assets there.
