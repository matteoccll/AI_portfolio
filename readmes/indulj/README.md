# Indulj — Mobile Massage Therapy Website

Static single-page site for Indulj, a mobile massage therapy service targeting London
corporate offices and care home residents.

## Architecture

Single HTML file with all styling in an embedded stylesheet. Design tokens are extracted
into CSS custom properties at `:root` — colours, type scale, spacing, shadows — so the
visual language is defined once and cascades automatically. Typography uses two variable
fonts (Fraunces serif with `SOFT=60` axis for optical softening, and Mulish sans-serif),
both sized with `clamp()` for continuous fluid scaling rather than breakpoint jumps. No
JavaScript framework; no build step.

## Stack

- Vanilla HTML5 + CSS3 — deliberate: zero build toolchain, instant deploy, no dependencies
- Fraunces (variable serif) + Mulish — font pairing chosen to match brand identity
- CSS custom properties — design token layer for maintainability
- GitHub Pages — static hosting

## Accessibility

Skip navigation link, semantic landmark elements, focus-visible outlines on all
interactive elements, `prefers-reduced-motion` block disabling all transitions.

## Status

Work in progress — approximately 3 hours in, with no prior web-development experience.
Content and copy are placeholders pending sign-off from the client.
Published with the subject's knowledge and permission.

Deployed: https://matteoccll.github.io/indulj_website/
