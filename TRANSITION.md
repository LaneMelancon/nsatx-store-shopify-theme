# Design System → Shopify Theme Transition

**Project:** NeuroSolution ATX Supplement Store **Maintained By:** Lane Melancon — Onn Grid, LLC **Last Updated:** 2026-07-15

This file documents everything that needs to be considered, reconciled, or built when applying the design system to the Shopify theme (`nsatx-store-shopify-theme`). Use it as the source of truth for planning and performing the full implementation.

---

## Strategy

**Tinker's default components are being kept.** The visual quality of Tinker's built-in components (cards, buttons, nav, product grids, etc.) is the primary reason for choosing it, and replacing them wholesale would eliminate that advantage.

The approach is therefore:

1. **Override Tinker's token layer** with core colors, fonts, styling, and spacing — Tinker's components inherit the new values automatically
2. **Add custom components** following the new tokens and base styles from the design system only when instructed and where Tinker has no equivalent
3. **Add spacing utility classes (potentially)** as an additive layer — Tinker doesn't have them but they don't conflict

The design system's role is **brand token source of truth + stylistic preferences**, not a replacement for Tinker's component CSS.

---

## Theme Context

The Shopify theme is built on **Tinker** — an exported default Shopify theme. Tinker uses:

- CSS custom properties for all tokens, defined in `snippets/theme-styles-variables.liquid`
- Its own naming conventions for colors, fonts, and spacing that differ from the design system
- Web Components (custom elements) for interactive UI, following the Shopify Dawn pattern
- `{% render %}` snippets for icons and reusable partials
- Shopify's font picker and color scheme system for merchant customization

---

## Scope

This pass covers the **token layer only**: fonts, colors, spacing, and the theme settings that expose them. It does **not** touch individual section/block/snippet component files, and does not port the design system's page-level patterns — Tinker's own component CSS stays as-is per the Strategy above and inherits new values purely through the token cascade. Remaining visual customization happens by hand in the Theme Editor once this layer is live.

## 1. Font Loading

**Confirmed mechanism:** Tinker's font system is an indirection chain, not baked values. `config/settings_schema.json` exposes 4 `font_picker` settings — `type_body_font`, `type_subheading_font`, `type_heading_font`, `type_accent_font` — and `snippets/theme-styles-variables.liquid` turns each into a base-role variable set (`--font-body--family/style/weight`, and the same for `subheading`/`heading`/`accent`). Every other font variable in the theme — `--font-paragraph--family`, `--font-h1--family` through `--font-h6--family`, `--button-font-family-primary`, `--cart-secondary-font-family` — is generated as `var(--font-<role>--family)`. Because these are `var()` references, not copied literals, **overriding just the 4 base-role variables at `:root` cascades to every downstream component automatically** — zero component files need to change.

Font delivery today goes entirely through Shopify's hosted font-picker CDN. Objective and Trade Gothic LT aren't in Shopify's font library, so this pipeline is bypassed for them; `design-system/css/fonts.css` self-hosts both locally (`design-system/fonts/*.woff2`, 14 files), which is what the theme will mirror.

**Confirmed role mapping:**

| Tinker role variable set | New value |
| --- | --- |
| `--font-body--family/style/weight` | Objective, 400, normal |
| `--font-subheading--family/style/weight` | Objective, 400, normal (matches body/default text styling) |
| `--font-heading--family/style/weight` | Objective, 400, normal (h1–h6 all use the same weight as body) |
| `--font-accent--family/style/weight` | Trade Gothic LT, 700, normal |

Any merchant who sets a heading's font role to "accent" via the Theme Editor gets Trade Gothic LT — matching the design system's eyebrow/nav-link/footer-heading usage — without touching Tinker's per-heading role selector UI.

**Decided: hide the now-vestigial font-picker settings — via `visible_if`, not deletion.** Once the bridge overrides `--font-*--family`, `type_body_font`/`type_subheading_font`/`type_heading_font`/`type_accent_font` no longer have any visual effect — but Shopify would still fetch/`@font-face` whatever font a merchant picks there, wasting bytes. **Do not delete these settings outright** — both `snippets/fonts.liquid` (`{%- unless settings.type_body_font.system? -%}`) and `snippets/theme-styles-variables.liquid` (which generates `--font-body--family` etc. directly from `settings.type_body_font.family`) actively reference them; removing the setting definitions would make `settings.type_body_font` resolve to `nil` at runtime, risking broken `@font-face`/preload output. Instead, add a `visible_if` condition to each of the 4 settings (exact condition syntax to confirm during implementation) — this hides them from the Theme Editor while keeping the underlying `font` object valid for Liquid, per Shopify's own docs: the value assigned to a setting stays available in Liquid regardless of whether the input is visible.

**What to build:**

1. Copy the 14 `.woff2` files from `design-system/fonts/` into `shopify-theme/assets/`.
2. New snippet `snippets/brand-fonts.liquid` — a `{% style %}` block with 14 `@font-face` rules, each `src: url({{ 'objective-400.woff2' | asset_url }}) format('woff2')`, `font-display: swap` — using the `asset_url` filter instead of relative paths (Shopify fingerprints/hashes asset URLs, so static relative paths won't resolve).
3. Preload only the critical weights (Objective 400 for body/headings, Objective 500 for buttons, Trade Gothic LT 700 for accent/eyebrow — 3 files, per Shopify's performance guidance to preload sparingly).
4. Render `brand-fonts` in `layout/theme.liquid` `<head>`, then override the 4 base-role variables in the token-bridge snippet (§2) so it loads after `theme-styles-variables.liquid`.
5. Hide (don't delete) the 4 vestigial `font_picker` settings via `visible_if` in `settings_schema.json`.

---

## 2. Color Tokens

**Confirmed mechanism:** Tinker does not use `color_scheme_group`/`color_scheme` (Dawn-style). It uses the newer **`color_palette`** setting type — one flat, named swatch list per theme (2–20 colors, no per-key labels, fixed native Admin picker UI that theme code cannot restyle — confirmed against shopify.dev). ~20 individual UI-color settings (buttons, badges, borders, popovers) default to Liquid references like `{{ settings.color_palette.color1 }}`, resolved once into a `:root` block by `snippets/color-palette.liquid`. A separate scoped layer, `snippets/contrast-override.liquid`, rescopes color variables to `.color-custom-{section_id}` with its own WCAG contrast fallback and won't pick up a `:root`-level override.

**Verified directly against the schema files** (not assumed): `settings_schema.json`'s own per-setting defaults for every button/badge/border color already just reference `.foreground` — the `color1`/`color3`/`color4` references only exist as _overrides_ in `settings_data.json`'s `current` and `presets.Tinker` blocks, on exactly these settings:

| Setting                             | Current reference |
| ----------------------------------- | ----------------- |
| `palette_primary_button_background` | `color1`          |
| `popover_border_color`              | `color1`          |
| `badge_sale_background_color`       | `color3`          |
| `drawer_border_color`               | `color4`          |
| `palette_input_border`              | `color4`          |
| `palette_variant_border`            | `color4`          |

(`color2` has no existing references anywhere in the schema.) This means only `settings_data.json` needs the reference-string updates below — `settings_schema.json`'s per-setting defaults don't need touching.

**Decided: new 13-key `color_palette`**, replacing the old 6-key set (`background`, `foreground`, `color1`–`color4`) with a curated ramp of raw design-system primitives (well under the 20-color cap):

| Palette key  | Value     | Source                |
| ------------ | --------- | --------------------- |
| `background` | `#ffffff` | `--color-white`       |
| `foreground` | `#0e2415` | `--color-green-900`   |
| `green800`   | `#0d371b` | `--color-green-800`   |
| `green700`   | `#0c8234` | `--color-green-700`   |
| `green400`   | `#3ce174` | `--color-green-400`   |
| `green200`   | `#9df0b9` | `--color-green-200`   |
| `green100`   | `#cef8dc` | `--color-green-100`   |
| `green50`    | `#e7fbee` | `--color-green-050`   |
| `green25`    | `#f8fcf9` | `--color-green-025`   |
| `orange25`   | `#fffcf0` | `--color-orange-025`  |
| `neutral950` | `#1a2f21` | `--color-neutral-950` |
| `neutral900` | `#273a2d` | `--color-neutral-900` |
| `neutral800` | `#3e5044` | `--color-neutral-800` |

**Remap the existing `color1`–`color4` reference strings** in both `settings_data.json`'s `current` and `presets.Tinker` blocks to the new key names (verified list above — 6 reference strings total, `color2` has none to update):

| Setting                             | Old reference | New reference |
| ----------------------------------- | ------------- | ------------- |
| `palette_primary_button_background` | `color1`      | `green700`    |
| `popover_border_color`              | `color1`      | `green700`    |
| `badge_sale_background_color`       | `color3`      | `foreground`  |
| `drawer_border_color`               | `color4`      | `orange25`    |
| `palette_input_border`              | `color4`      | `orange25`    |
| `palette_variant_border`            | `color4`      | `orange25`    |

Update `color_palette.default` in `settings_schema.json` and `current.color_palette` + `presets.Tinker.color_palette` in `settings_data.json` to the new 13-key table.

**Separately, the `:root` token-bridge** (`snippets/token-bridge.liquid`, rendered last in `<head>`, after `color-palette.liquid`) — this is independent of the merchant-facing palette above; it re-declares Tinker's ~60 `--color-*` variables at `:root` against the _full_ semantic token set in `design-tokens.css` (a verbatim copy of `design-system/css/tokens.css`, loaded before this snippet). Confident mappings for the well-understood core variables:

| Tinker variable | DS token |
| --- | --- |
| `--color-background` | `var(--surface-brand-default)` |
| `--color-foreground` | `var(--text-brand-default)` |
| `--color-foreground-muted` | `var(--text-brand-body)` |
| `--color-foreground-subdued` | `var(--text-brand-caption)` |
| `--color-border` | `var(--border-brand-subtle)` |
| `--color-primary-button-background` | `var(--brand-primary)` |
| `--color-primary-button-text` | `var(--text-brand-alt)` |
| `--color-primary-button-border` | `var(--brand-primary)` |
| `--color-secondary-button-background` | `transparent` |
| `--color-secondary-button-text` | `var(--text-brand-accent)` |
| `--color-secondary-button-border` | `var(--border-brand-accent)` |
| `--color-instock` | `var(--icon-success-default)` |
| `--color-lowstock` | `var(--icon-warning-default)` |
| `--color-outofstock` | `var(--icon-error-default)` |
| `--color-error` | `var(--icon-error-default)` (only usage found is `fill` on `.icon-error` in `base.css`) |
| `--color-success` | `var(--icon-success-default)` (only usage found is `color` on `.icon-success` in `base.css`) |
| `--color-input-background` | `var(--surface-brand-default)` |
| `--color-input-border` | `var(--border-brand-subtle)` |
| `--color-input-text` | `var(--text-brand-body)` |
| `--color-selected-variant-background` | `var(--brand-primary)` |
| `--color-selected-variant-text` | `var(--text-brand-alt)` |

The remaining ~40 Tinker color variables are hover states, `-rgb` triplets, and per-component scoping. Two things to know before assigning them during implementation:

- **`-rgb` variables can't be `var()` aliases** — Tinker uses comma-separated RGB triplets for `rgba()` composition; these must be hardcoded literals derived from whichever DS hex they mirror, manually re-synced if that color ever changes.
- **Hover-state variables** don't have a dedicated DS "brand hover" alias token (only success/info/warning/error surface tokens have `-default-hover` variants) — derive these visually.

**No dark-theme mechanism.** The `[data-theme='dark']` alias-override block has been removed from `design-system/css/tokens.css` — there is no dark-mode token layer to bridge. Dark-background sections (nav dark variant, footer, dark hero/CTA pattern, etc.) are handled the same way as any other section: background colors set and adjusted by hand in the Theme Editor, using the palette in §2 like any other color choice.

---

## 3. Spacing Tokens

**Resolved (was a risk, now isn't):** the design system's `--space-*` tokens no longer use the experimental CSS `@function` syntax — they're now plain `calc(var(--space-base) * N)` expressions, fully supported everywhere. Tinker's spacing variables can be **directly aliased** to the DS custom properties (a live `var()` reference, same technique as the color bridge) rather than needing literal values copied in.

**Current DS scale** (confirmed): `--space-base: 0.25rem`; `--space-4xs` (0.0625rem) · `-3xs` (0.125) · `-2xs` (0.1875) · `-xs` (0.25) · `-sm` (0.5) · `-reg` (1) · `-md` (1.5) · `-lg` (2) · `-xl` (3) · `-2xl` (4) · `-3xl` (5) · `-4xl` (6.25).

**Tinker's scale** (`snippets/theme-styles-variables.liquid`, three parallel `--padding-*`/`--margin-*`/`--gap-*` sets, rem): 3xs `0.125` · 2xs `0.25–0.3` · xs `0.5` · sm `0.7` · md `0.8` · lg `1` · xl `1.25` · 2xl `1.5` · 3xl `1.75` · 4xl `2` · 5xl `3` · 6xl `4–5`.

**Approach:** in `token-bridge.liquid`, override each Tinker spacing variable to `var(--space-*)`, rounding to the nearest DS step where there's no exact match (Tinker's scale has more steps than DS's at a few points — e.g. `sm: 0.7rem` sits between DS `sm` (0.5) and `reg` (1)). These are live aliases, not copied literals, so a later DS scale tweak propagates automatically. Finalize the handful of ambiguous roundings with a quick visual pass.

---

## 4. `base.css` — two different files, two different plans

**Tinker's `assets/base.css`** (the theme's own file, 4,014 lines). Confirmed by direct grep: already almost entirely a **token consumer** — only one hardcoded hex value (`#000`) in the whole file, everything else reads `var(--color-*)`, `var(--padding/gap/margin-*)`, `var(--font-*)`. The `:root` bridge in §2/§3 cascades into the vast majority of it automatically, with **no edits needed** there.

The genuinely scoped edit is `base.css`'s own local `:root` block (lines ~14–24) — variables it defines itself rather than inheriting from `theme-styles-variables.liquid`:

```css
--hover-lift-amount: 4px;
--hover-scale-amount: 1.03;
--hover-subtle-zoom-amount: 1.015;
--hover-shadow-color: var(--color-shadow);
--hover-transition-duration: 0.25s;
--hover-transition-timing: ease-out;
--surface-transition-duration: 0.3s;
--surface-transition-timing: var(--ease-out-quad);
--submenu-animation-speed: 360ms;
--submenu-animation-easing: cubic-bezier(0.25, 0.1, 0.25, 1);
```

Align these to the design system's own transition/easing tokens (`--duration-fast` 150ms / `--duration-normal` 250ms / `--duration-slow` 400ms, `--ease-out` / `--ease-in-out`) where a reasonable match exists. Small, contained, ~10 lines.

**`design-system/css/base.css`** (reset + typography fundamentals, 239 lines) — **not being ported, none of it.** Checked it directly against Tinker's own `base.css` line by line; Tinker already has a more mature equivalent of everything in it:

- **Reset + tag defaults** (`body`, `h1`–`h6`, `a`, `button`, `input`/`select`/`textarea`, `img`/`svg`) — Tinker already defines all of these itself, already wired to the same token system the bridge retargets in §2/§3. Porting the design system's version would just add a second, competing set of global tag rules.
- **`prefers-reduced-motion`** — Tinker already has 6 separate scoped media-query blocks; the design system's is one single global block. Tinker's is the more complete implementation.
- **`:focus-visible`** — Tinker already has a global rule plus a `@supports not selector(:focus-visible)` fallback and several component-specific refinements the design system's version doesn't have.
- **Concrete conflict if ported anyway:** the design system's `body` rule sets `min-height: 100vh`; Tinker deliberately uses `min-height: 100svh` (the mobile-toolbar-safe unit). Porting would risk regressing that on iOS Safari if it won the cascade.
- The file's typography/color **utility classes** (`.text-lg`, `.font-bold`, `.text-brand-default`, `.accent`, etc.) are reference/demonstration code showing how the design system's own docs compose `tokens.css` — not a reusable class library for the theme. Nothing in Tinker's Liquid output would ever apply those class names; Theme Editor customization happens through Tinker's own schema-driven settings, not by hand-authoring CSS classes. Once the token remap in §2/§3 flows into Tinker's own `base.css` and component CSS, that's the complete picture — no separate utilities file needed.

---

## 5. Files to Create / Modify

**New:**

- `shopify-theme/assets/objective-{300,400,500,700,800}[-italic].woff2` (10 files)
- `shopify-theme/assets/trade-gothic-lt-{400,700}[-italic].woff2` (4 files)
- `shopify-theme/assets/design-tokens.css` (verbatim copy of `design-system/css/tokens.css`)
- `shopify-theme/snippets/brand-fonts.liquid`
- `shopify-theme/snippets/token-bridge.liquid` (fonts + colors + spacing `:root` overrides)

**Modified:**

- `shopify-theme/layout/theme.liquid` — add the new stylesheet/snippet renders in `<head>`, after the existing `theme-styles-variables`/`color-palette` renders
- `shopify-theme/config/settings_schema.json` — replace `color_palette.default`'s 6 keys with the new 13-key table; hide (via `visible_if`, not delete) the 4 vestigial `font_picker` settings
- `shopify-theme/config/settings_data.json` — same 13-key `color_palette` update in `current` + `presets.Tinker`, plus the 6 reference-string remaps listed in §2
- `shopify-theme/assets/base.css` — realign the ~10 local hover/transition/animation variables to DS duration/easing tokens

**Explicitly out of scope for this pass:** any `sections/`, `blocks/`, or per-component `snippets/` files (styling), and porting any design-system page-level patterns — those are done by hand in the Theme Editor once this token layer and core theme styling is live.

## Open Questions

- [ ] `snippets/contrast-override.liquid` computes its own scoped `--color-*` values for `.color-custom-{section_id}` (popovers, badges, drawers) independently of the `:root` token-bridge, using its own WCAG "smart contrast" logic driven by merchant-configured settings. It won't automatically inherit the new palette/bridge values — needs a dedicated check during implementation to confirm it still produces sensible results against the new colors, not just the settings it was designed around.
- [ ] Finalize the handful of ambiguous spacing roundings (§3) with a visual pass.
- [ ] Audit which Tinker interactive components already have JS (the ~80 files in `assets/`) before building any new Web Components — unrelated to this token pass, still open.

## Verification

Once implemented:

- `shopify theme dev` against the dev store; confirm the 13 new palette swatches appear correctly in the Theme Editor (Colors settings + any section/block color picker), and that the 6 remapped settings (primary button, popover border, badge sale, drawer/input/variant borders) render the intended new colors.
- `shopify theme check` — must pass clean.
- Confirm font role override cascades to `--font-paragraph`, `--font-h1`–`--font-h6`, and buttons without editing any component file.
- Confirm the 4 font-picker settings are hidden from the Theme Editor's Typography group (via `visible_if`) with no console/schema errors, and that `snippets/fonts.liquid`/`theme-styles-variables.liquid` still render without errors now that those settings are Editor-hidden.
- Spot-check `base.css`-driven hover/transition motion (buttons, cards, submenu) after the local variable realignment.
- Cross-browser per `AGENTS.md`: Chrome/Firefox/Safari desktop+iOS, Chrome Android.

Formal WCAG contrast auditing is not a gate here — contrast on the newer palette entries (`green400`, `green200`, `neutral800`, etc.) will vary by context (background vs. text use), and that's expected; treat it as ordinary visual judgment during hand-customization, not a pass/fail check.
