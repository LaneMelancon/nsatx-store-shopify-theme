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

## Rules
- Keep this markdown file up to date with changes or additions related to the Shopify Theme
- Always develop against the dev store (`atx-prod-test-01`) before pushing to production
- Reference `nsatx-store/design-system/CLAUDE.md` for all design tokens, components, and styling decisions
