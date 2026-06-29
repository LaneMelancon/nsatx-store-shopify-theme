# NeuroSolution ATX Supplement Store — Shopify Theme

**Maintained By:** Lane Melancon — Onn Grid, LLC
**Client:** Dr. Brandon Crawford — NeuroSolution Center of Austin
**Last Updated:** 2026-06-29
**GitHub Repo:** `https://github.com/LaneMelancon/nsatx-store-shopify-theme`

---

## Project Direction

This is an exported default theme from Shopify called 'Tinker.' It serves as the boilerplate skeleton for building the Supplement Store using the Design System (`nsatx-store/design-system/CLAUDE.md`).

This repo is kept independent (as a submodule of `nsatx-store`) specifically to support Shopify's GitHub integration — the Shopify GitHub app requires a standalone repo to sync theme code directly to a store without manual imports.

## Shopify GitHub Integration

- **Dev store:** `atx-prod-test-01.myshopify.com` — connect via Shopify GitHub app (pending setup)
- **Production store:** `neurosolution-shop.myshopify.com` — connect after dev store is confirmed
- Once connected, pushing to `main` will sync theme changes to the store in real time

## Markdown Rules

- Keep this markdown file up to date with changes or additions related to the Shopify Theme
- Always develop against the dev store (`atx-prod-test-01`) before pushing to production
- Reference `nsatx-store/design-system/CLAUDE.md` for all design tokens, components, and styling decisions

---

## Shopify Theme Development Guidelines

The following sections provide frontend best practices and conventions for Claude Code when working on the Shopify theme project. Follow these guidelines unless project owner/maintainer explicitly overrides them.

### Project Structure

- Follow the standard Shopify theme directory structure: `assets/`, `config/`, `layout/`, `sections/`, `snippets/`, `templates/`, `locales/`.
- Never create directories outside this structure without explicit justification.
- Keep `layout/theme.liquid` lean — load scripts and styles via `{% render %}` snippets or section includes, not inline.
- Use `templates/` for JSON templates (Online Store 2.0). Avoid `.liquid` templates unless supporting a legacy theme.

### Liquid Templating

- Prefer `{% render %}` over `{% include %}`. `render` has a sandboxed scope; `include` leaks parent variables and is deprecated.
- Never use `{% include %}` in new code.
- Keep logic out of templates. Move conditional blocks and loops into snippets.
- Avoid deeply nested Liquid logic (>3 levels). Refactor into named snippets for readability.
- Use `liquid` tag blocks (`{% liquid %}`) to group multiple tags without extra whitespace output.
- Strip whitespace with `{%- -%}` and `{{- -}}` in performance-sensitive areas (loops, large partials).
- Always provide fallback values for variables: `{{ product.title | default: 'Untitled' }}`.
- Never output raw user-controlled data without appropriate filters (e.g., `escape`, `strip_html`).

### Sections & Blocks (Online Store 2.0)

- Every merchant-editable area must be a section or a block inside a section.
- Define schema settings (`{% schema %}`) for all configurable content — never hardcode merchant-facing copy.
- Use `presets` in section schemas so sections are available in the Theme Editor.
- Sections must be self-contained and re-renderable in isolation (required by the section rendering API).
- Use `blocks` for repeatable elements (e.g., slides, tabs, FAQ items). Set a reasonable `max_blocks` limit.
- Always define a `name` and `class` in the section schema for Editor discoverability.
- Validate that schema `type` values match Shopify's supported input types (text, richtext, image_picker, url, color, range, select, checkbox, radio, product, collection, etc.).
- App Blocks must be declared in section schemas under `"blocks": [{ "type": "@app" }]` to support app integrations cleanly.

### Metafields & Metaobjects (Read-Only in Theme)

- Themes can only **read** metafields — never attempt to write or update them from Liquid or client-side JS.
- Access metafields via `product.metafields.namespace.key` — never construct dynamic metafield keys at runtime.
- Always check for existence before outputting: `{% if product.metafields.custom.tagline != blank %}`.
- Metafield values have a **16 KB per-value cap**. If a value appears truncated, flag it — the fix is on the data/admin side, not in the theme.
- Document every custom metafield namespace and key the theme depends on inside `README.md`, so developers know what needs to be configured in the store.

### Performance

- Lazy-load all images below the fold using `loading="lazy"` and Shopify's `image_url` filter with explicit `width` and `height`.
- Always use the `image_url` filter with a `width` parameter — never output raw CDN URLs or hardcoded sizes.
- Use `srcset` for responsive images. Shopify's `image_tag` filter generates srcset automatically.
- Defer non-critical JavaScript with `defer` or `type="module"`.
- Never use `document.write()`.
- Avoid loading third-party scripts in `<head>` — use `async` or move to end of `<body>`.
- Limit the number of section-level `<script>` tags; consolidate theme JS into `assets/`.
- Use `preload` for critical fonts and above-the-fold images.
- Target a Lighthouse performance score ≥ 80 on mobile for all new features.

### JavaScript

- Write vanilla JS unless the project already uses a framework. Do not introduce React, Vue, or Alpine.js without team approval.
- Use Web Components or custom elements for encapsulated interactive UI (carousels, modals, accordions). Shopify Dawn uses this pattern.
- Avoid jQuery. It is not included in modern Shopify themes.
- Namespace all global variables and custom events under a project-specific prefix (e.g., `window.ThemeName`).
- Use `CustomEvent` for cross-component communication. Dispatch events on `document` and listen from any component.
- Always use `addEventListener` — never inline `onclick` attributes.
- Handle errors gracefully; wrap Fetch calls and Cart API calls in try/catch with user-visible fallback messaging.

### Cart (Client-Side)

- Use the **Cart API** (`/cart/add.js`, `/cart/update.js`, `/cart/change.js`) for all cart mutations — never rely on full-page form submissions where a dynamic cart experience is expected.
- Dispatch Shopify's standard cart events (`cart:refresh`, `cart:updated`) after mutations so other components (mini-cart, header count, etc.) can react without tight coupling.
- Always optimistically update the UI and roll back on API error with a visible message.
- Never redirect to `/checkout` directly from JS — use `window.location` only after a confirmed cart state, or use the standard checkout button.

### CSS & Styling

- Use CSS custom properties (variables) for all design tokens: colors, spacing, typography, border-radius.
- Define theme color and font settings in `config/settings_schema.json` and expose them as CSS variables in `layout/theme.liquid`.
- Avoid `!important`. If needed, it indicates a specificity problem — fix the selector instead.
- Do not use inline styles in Liquid templates except for dynamically injected CSS custom properties.
- Scope component styles using a BEM-like convention or web component selectors — avoid broad tag selectors.
- Minify production CSS. Use Shopify CLI's build pipeline or a documented build step.

### Localization & Accessibility

- All customer-facing strings must use `{{ 'key' | t }}` translation keys defined in `locales/`.
- Never hardcode English strings in templates — even for single-locale stores.
- Always provide `alt` text for images: use the product/image alt attribute, a metafield setting, or a schema setting fallback. Never leave `alt` empty on meaningful images.
- All interactive elements (buttons, links, modals) must be keyboard accessible and have visible focus styles.
- Use semantic HTML: `<nav>`, `<main>`, `<header>`, `<footer>`, `<article>`, `<section>`, `<button>`.
- Every form input must have an associated `<label>` (visible or visually hidden via `.visually-hidden`).
- Modals and drawers must trap focus and support `Escape` to close.
- Target WCAG 2.1 AA compliance as a minimum baseline.

### Shopify CLI

- `shopify theme dev` — starts a local dev server that mirrors file changes to the dev store in real time. Use this before committing to preview changes in a real browser.
- `shopify theme check` — lints the theme for errors and warnings. Run before every push.
- `shopify theme push` — redundant in this project; the GitHub integration handles deployment to the dev store automatically on push to `main`.

### Git

- Commit directly to `main`. This is a solo project and the GitHub → Shopify integration deploys on every push.
- Keep commits small and focused — one logical change per commit.
- Write commit messages in the imperative mood: `Add sticky header behaviour`, `Fix quantity input on mobile cart`.
- Never commit secrets, tokens, or credentials. Use environment variables or Shopify CLI's authenticated sessions.
- Keep the following in `.gitignore`: `node_modules/`, `.env`, `.DS_Store`, and any build artefact directories.
- Run `shopify theme check` before pushing and resolve all errors.

### Testing & Quality

- Run `shopify theme check` before every commit and resolve all errors. Treat warnings as errors for new code.
- Validate all schema JSON — malformed schema silently breaks the Theme Editor.
- Test on real Shopify preview URLs, not just `localhost` — cart, redirects, and some Liquid objects behave differently locally.
- Test across: Chrome, Firefox, Safari (desktop + iOS), and Chrome Android.
- Test with a screen reader (VoiceOver on macOS/iOS, NVDA on Windows) for accessibility-critical flows.
- Test Theme Editor functionality: every section must be addable, removable, and reorderable without JS errors.

### Documentation

- Every section must have a comment block at the top describing its purpose and any non-obvious schema settings.
- Document all metafield namespaces and keys the theme reads in `README.md`.
- Add inline `{% comment %}` blocks for non-obvious Liquid workarounds explaining the reason.
- Maintain a `CHANGELOG.md` using [Keep a Changelog](https://keepachangelog.com/) format.
