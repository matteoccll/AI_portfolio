# Two Little Giraffes — Gelato & Café Website

Static multi-page site for Two Little Giraffes, an artisan gelato and café in
Battersea, London — covering the shop, menu, story, gelato bike, monthly art
exhibitions, and contact.

## Architecture

A single self-contained HTML file. All styling lives in one embedded stylesheet
and design tokens are extracted into CSS custom properties at `:root` — brand
colours, type scale, spacing, and cream surface tiers — so the visual language
is defined once and cascades automatically. Typography uses three variable
fonts (Fraunces serif with the `SOFT` axis raised for rounder display
lettering, DM Sans for body, and Caveat for handwritten accents), sized with
`clamp()` for continuous fluid scaling rather than breakpoint jumps. The six
pages are served from one document via lightweight hash-based routing in vanilla
JavaScript — no framework, no build step. The flavour board renders from an
inline JSON block, so the daily counter can be updated by editing data alone.

## Stack

- Vanilla HTML5 + CSS3 + a small amount of vanilla JavaScript — deliberate: zero
  build toolchain, instant deploy, no dependencies
- Fraunces (variable serif, SOFT axis) + DM Sans + Caveat — font pairing chosen
  to match brand identity
- CSS custom properties — design token layer for maintainability
- Data-driven flavour board — content separated from markup via inline JSON
- GitHub Pages — static hosting

## Accessibility

Semantic landmark elements, ARIA state on the navigation (`aria-current`,
`aria-expanded`, `aria-controls`, `aria-label`) for the page router and mobile
menu, descriptive labels on icon-only controls, and a `prefers-reduced-motion`
block disabling transitions.

## Status

Work in progress. Content and copy are placeholders pending sign-off from the
client; several elements are intentionally marked as placeholders in the design.
Published with the subject's knowledge and permission.

Deployed: https://matteoccll.github.io/twolittlegiraffes_website/
