# Capella University Homepage (v2)

Vanilla HTML, CSS, and JavaScript implementation of the Capella University
homepage from Figma. No framework, no CSS preprocessor — Vite is used only as
the dev server and bundler.

This is the **second version** of the homepage. See
[`HANDOFF.md`](HANDOFF.md) for what changed from v1 and for the engineering
detail behind everything below, [`DEBUGGING.md`](DEBUGGING.md) when something
looks broken — it is a symptom-first runbook covering the traps this codebase
has already hit — and
[`ACCESSIBILITY-AUDIT.md`](ACCESSIBILITY-AUDIT.md) for the WCAG 2.1 AA pass on
the hero and RFI form, including how each measurement was taken and what to
re-test when the photo or the copy changes.

## Getting started

```bash
npm install
```

```bash
npm run dev
```

Open [http://localhost:5173](http://localhost:5173).

To build for production:

```bash
npm run build
```

```bash
npm run preview
```

## Project structure

```
index.html         # Page markup (single page)
css/
  tokens.css       # Design tokens — colors, type scale, spacing, motion,
                   #   nav interaction states
  styles.css       # All styles (mobile-first, @imports tokens.css)
js/
  main.js          # Carousel, program finder, and all scroll/reveal animations
public/assets/     # Committed image assets (SVG / PNG / JPG / WebP)
  videos/          # CTA band background clip — 3 encodes, each MP4 + WebM
HANDOFF.md         # Engineering handoff: gotchas, breakpoints, asset contracts
DEBUGGING.md       # Symptom-first runbook for traps this codebase has hit
ACCESSIBILITY-AUDIT.md  # WCAG 2.1 AA pass: findings, measurements, retest list
```

`js/main.js` is a set of small `init*` functions, all called on
`DOMContentLoaded`. The motion ones bail early on `prefers-reduced-motion`; the
navigation and form ones always run, since they're behaviour rather than
animation:

| Function | Responsibility |
| --- | --- |
| `initCarousel` | Featured-story carousel — dots, swipe, keyboard, card reveal |
| `initHeroRfi` | The hero's integrated 2-step RFI form — step swap, stepper state, the gated Degree→Area→Specialization chain, the conditional RN-licence / learning-format questions, per-field error/success/disabled states, the panel-height reservation at 1024+, and `fitTabletHero()` which makes the 768–1023 hero hug the form |
| `initProgramFinder` | Degree-level chips + dependent Area/Specialization selects |
| `initRevealAnimations` / `initTextReveal` | Fade-up on scroll; per-word masked heading reveal |
| `initCountUp` | Stat numbers counting up |
| `initParallax` / `initHeroParallax` / `initContentParallax` / `initCardScroll` | Scroll-driven motion (content band, hero photo, program-finder drift, carousel card slide-in). The hero *copy* no longer drifts — it holds the RFI form |
| `initNavScroll` / `initMobileNav` | Sticky-nav shrink; hamburger panel |
| `initMegaMenu` / `initMobileMenuTree` | Desktop dropdown positioning; the mobile panel's nested menus |
| `initCtaVideos` | Background video in the CTA band — encode tier, lazy-load, pause offscreen |
| `initFooterPartners` | Footer brand carousel — manual arrows, paging by a whole view |

## Assets

**All assets are committed under `public/assets/` and referenced as
`/assets/…`.** They are not fetched from Figma at runtime — an earlier version
of this project used Figma MCP asset URLs, which expire after ~7 days.

Two asset contracts are load-bearing and documented in `HANDOFF.md` §3 — read it
before replacing them:

- **`hero-rfi-desktop.webp` + `hero-rfi-mobile.webp`** — the hero photo, cut from
  one source into a per-breakpoint crop and picked by `<picture>`. The crop is
  baked into the asset on purpose; see HANDOFF §3a before re-exporting. The
  portrait crop covers **≤1023**, not just phones, and is **mirrored** at
  768–1023 so the subject clears the form panel (HANDOFF §5e) — re-exporting or
  re-cropping invalidates that and the contrast figures in
  `ACCESSIBILITY-AUDIT.md`.
- **`carousel-portrait-faculty.webp`** — a pre-cut transparent portrait sized to
  the carousel card, and intentionally *taller* than the card because the figure
  breaks out above its top edge.
- **`wnba-capella-lockup*.webp`** — the partnership card's artwork. The logos,
  the divider rule *and* the "official higher learning partner" line are all
  baked into one image, so the `alt` text is the only accessible copy of that
  line. Two crops (wide / stacked), picked by a `<picture>` `media` query.

## Design source

- [Figma — Hi-fi Review](https://www.figma.com/design/6tdLZrCAiMSii7sMAUrRDs/Hi-fi---Review?node-id=5647-5140)
  — the homepage overall.
- [Figma — Card Update](https://www.figma.com/design/vkdlGCLzDrSK1crS3ZtT5A/Card-Update?node-id=6-194)
  — the featured-story cards (v2 rebuilt these against this file). The WNBA
  partnership card is
  [`55-62`](https://www.figma.com/design/vkdlGCLzDrSK1crS3ZtT5A/Card-Update?node-id=55-62)
  (desktop + mobile in one frame).
- **Jake's page updates** —
  [Figma — UI Elements for Homepage Proto](https://www.figma.com/design/h3IvZdQj2uH5bm7JPUD89a/UI-Elements-for-Homepage-Proto?node-id=50-21184)
  — the nav dropdown and its activated state, button and chip states, the closing
  CTA, and the mobile bottom bar / chat launcher.

## Browser notes

- Layout is mobile-first; the wide carousel layout takes over at `≥1024px`. The
  hero has **three** compositions, not two — stacked below 768, two columns
  (photo left, floating form panel right) at 768–1023, and the full-bleed
  overlay from 1024. The `<picture>` query splits at 1023 to match. See
  HANDOFF §5e.
- Every animation has a `prefers-reduced-motion: reduce` fallback.
