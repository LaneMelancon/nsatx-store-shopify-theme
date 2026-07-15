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

## 1. Font Loading — Highest Risk

**The problem:** Tinker loads fonts through Shopify's font picker (`settings.type_body_font`, `settings.type_heading_font`, etc.) and generates CSS variables like `--font-paragraph--family`, `--font-h1--family`. The design system fonts — Objective and Trade Gothic LT — are custom fonts hosted locally in the project and delivered throught the store's theme conection to the Github repo. They do not exist in Shopify's font library, so the Theme Editor font picker is irrelevant.

**What needs to happen:**

- Add custom `<link rel="preload">` tags for the jsDelivr font files directly in `snippets/fonts.liquid` (or a new dedicated snippet)
- Bridge Tinker's font variable names to the design system's font stack:
  - `--font-paragraph--family` → Objective
  - `--font-h1--family` through `--font-h6--family` → Objective
  - Define a variable for Trade Gothic LT for eyebrow roles
- This ensures all of Tinker's existing component rules inherit the correct fonts without rewriting every rule

**Design system font variables:**

| DS Variable/Token  | Font            | Role                        |
| ------------------ | --------------- | --------------------------- |
| `--font-primary`   | Objective       | Default, headings, body, UI |
| `--font-secondary` | Trade Gothic LT | Eyebrow, nav links          |

---

## 2. Color Tokens

**The problem:** Tinker uses semantic color variables generated from the Theme Editor's color scheme settings — `--color-background`, `--color-foreground`, `--color-primary-button-background`, etc. The design system uses `--color-green-700`, `--text-brand-default`, `--color-neutral-800`, etc. These are completely different naming conventions.

**What needs to happen:**

- After loading `tokens.css`, set Tinker's semantic vars to point at design system tokens
- Manage one source of truth (the design system tokens); Tinker's vars become aliases

**Mapping to establish (partial — expand during planning or implementation):**

| Tinker Variable      | Design System Token            |
| -------------------- | ------------------------------ |
| `--color-background` | `var(--surface-brand-default)` |
| ...                  | ...                            |

---

## 3. Spacing Tokens

**The problem:** Tinker uses predefined semantic spacing/utility variables from the theme. The design system uses a named numeric scale (`--space-4xs` through `--space-4xl`) utilizing a CSS function rule called `--space` that relies on the base spacing incremenet token `--space-base` (0.25rem) to take in a number or percentage as a argument (`--multiplyer`) and return the product of the base spacing increment and the multiplyer. Tinker uses named semantic tokens (`--padding-xs`, `--padding-sm`, `--padding-md`, `--padding-lg`, `--gap-sm`, `--gap-md`, `--margin-xs`, `--margin-md`, `--margin-lg`).

**What needs to happen:**

- Do not add spacing utility classes — Tinker does not use them and they won't carry over
- After loading `tokens.css`, map Tinker's spacing vars to the design system scale
- Do not hardcode pixel values in Tinker's vars — point them at `--space-*` tokens or use the `--space(--multiplyer)` CSS function rule

**Important:** Tinker's spacing tokens are defined in `snippets/theme-styles-variables.liquid` using `rem` values on a non-standard scale (`--padding-md: 0.8rem`, `--gap-md: 0.9rem`, etc.). They do not align cleanly with the design system's scale. Rather than trying to alias them precisely, the recommended approach is to **override Tinker's spacing vars entirely** to match the design system scale during the bridge step.

**Suggested overrides (set in the token bridge file):**

| Tinker Variable | Override Value | DS Token Reference |
| --------------- | -------------- | ------------------ |
| ...             | ...            | ...                |

**In design system CSS, always use `--space-*` tokens directly — never reference Tinker's vars.** The bridge is one-directional: Tinker's vars point at DS tokens, not the reverse.

---

## Open Questions

- [ ] How should dark-background sections (`.nav--dark`, `green-900` footers) integrate with Tinker's color scheme system?
- [ ] Will the jsDelivr font CDN be used in production, or should font files be self-hosted in `assets/`?
- [ ] Which Tinker interactive components already have JS (audit needed before building new Web Components)?
