# Developer Handoff — Capella University Homepage (v2)

Engineering notes for anyone picking up this build. Covers the toolchain, the
animation system, responsive behavior, accessibility, asset handling, and the
edge cases / gotchas that aren't obvious from the code alone.

> See also: [`README.md`](README.md) for the quick-start, and
> [`DEBUGGING.md`](DEBUGGING.md) for symptom-first troubleshooting — start there
> when something *looks* broken; this file explains how things are *built*.

**This is the second version of the homepage**, living at
[`camila-go/CU-Homepage-Test-v2`](https://github.com/camila-go/CU-Homepage-Test-v2).
What changed from v1, and where the details are:

| Area | Change | §  |
| --- | --- | --- |
| Featured-story cards | Rebuilt to the **Card Update** Figma (`1440 × 600`, new copy and people, Figma-variable type scale, a portrait that breaks out above the card). Slide 2 is now the **WNBA partnership card**, not an alumni testimonial | §3, §6 |
| Closing CTA | A single full-bleed TV-spot clip in three encodes, picked by viewport, replacing the three gold-backdrop loops | §12 |
| Hero | Rebuilt to the **Phase 3 Hi-Fi** Figma: one pre-cropped photo (no more two-layer wall), left-aligned copy, and the **2-step RFI form integrated into the hero** — including step 1's conditional RN-licence / learning-format questions and the full Dropdown + Input field state matrices. The hero copy no longer floats on scroll — see §7 | §3a, §5, §5a, §5b, §7 |
| Carousel motion | Scroll-driven, ratcheted card slide-in (the card's text does not animate) | §7 |
| Nav | Rounded pill hover with full press / keyboard-focus states; scroll shrink now uses hysteresis | §5 |
| Assets | Carousel portraits re-cut as transparent WebP (2.5 MB of PNGs → 136 KB); hero is two pre-cropped WebPs, one per breakpoint | §3 |

---

## 1. Stack & tooling

| Thing | Detail |
| --- | --- |
| Build tool | [Vite 6](https://vitejs.dev) (`vite`, `vite build`, `vite preview`) |
| Language | Vanilla HTML + CSS + ES modules. **No framework, no CSS preprocessor.** |
| JS deps | None used at runtime. `vanilla-tilt` is still in `package.json` but **no longer imported** (the 3D tilt was removed — popular-program cards now use a CSS-only hover scale). Safe to `npm uninstall vanilla-tilt`. |
| Icons | Font Awesome Kit loaded via `<script src="https://kit.fontawesome.com/...">` in `<head>` |
| Fonts | Adobe Typekit (`acumin-pro-extra-condensed`, `acumin-pro`) + Google Fonts (`Inter`) |
| Dev server | `npm run dev` → http://localhost:5173 |

```bash
npm install
npm run dev      # local dev w/ HMR
npm run build    # production build → dist/
npm run preview  # serve the production build
```

`vite.config.js` is intentionally minimal (`root: '.'`). If this is ever
deployed under a sub-path (e.g. GitHub Pages project site), set `base` in
`vite.config.js` **and** note the absolute `/assets/...` paths below.

---

## 2. Project structure

```
index.html        # All page markup (single page)
css/
  tokens.css      # Design tokens (colors, type scale, spacing, easings) — imported first
  styles.css      # All component styles, mobile-first with desktop overrides
js/
  main.js         # All interactivity + animations (init* functions)
public/
  assets/         # Images + SVGs (served from /assets/... at runtime)
    videos/       # CTA band background clip, 3 encodes x (MP4 + WebM) — see §12
```

- `public/` is Vite's static dir, so files there are referenced with an
  **absolute path** (`/assets/hero.png`), not a relative one. Don't "fix" these
  to `./assets/...` — that will break the production build.
- `css/tokens.css` is `@import`-ed at the top of `styles.css`. All design
  primitives (color, type scale, radii, durations, easing curves) live there.
  Prefer adding/modifying tokens over hard-coded values.

---

## 3. Assets — read before touching images

### 3a. Hero is one photo, pre-cropped per breakpoint

The hero art is a single frame (subject on the right against a red wall), so
there is nothing to separate into layers:

- **`hero-rfi-desktop.webp`** (2560×1217, 153 KB) — landscape, used at 1024px+.
- **`hero-rfi-mobile.webp`** (750×1136, 78 KB) — portrait, used at **≤1023px**,
  via a `<picture>` source. Not just phones: at 768–1023 it is the hero's
  full-bleed background, **mirrored**, with the form panel floating on its right
  half (§5e). The landscape crop cannot do that job — in a 768×952 box `cover`
  zooms it 0.78× of 2560px, i.e. a 440px-wide head on a 768px screen. `fetchpriority="high"` on both (the hero is the LCP element), each with
  its own `media`-gated `<link rel=preload>` so a phone never pulls the desktop
  frame — ⚠️ **the preload `media` and the `<source>` `media` have to agree**, or
  the browser preloads one crop and then downloads the other.
- `.hero__background` keeps a `background-color` (wall red) so the hero never
  flashes black before the photo paints.

Both crops come from **one 4096×2322 source**, cut at exactly the boxes the
Figma frames crop to — desktop `28,664 → 2599,1886`, mobile
`1333,680 → 2417,2322`.

⚠️ **Cut the crop into the asset; don't port Figma's percentage offsets into
CSS.** Figma expresses these crops as independent width/height percentages on an
absolutely-positioned `<img>`. Those only hold at the one container aspect they
were authored for — at 1440×912 the desktop numbers resolve to a 1.32 aspect
against a 1.76 source and visibly stretch the subject. With the crop baked in,
`object-fit: cover` plus the anchor below is correct at every width.

⚠️ **`object-position` must stay anchored, NOT centred.** The subject is
off-centre in both crops — her head is in the right third of the desktop crop
(x 68–90%) and at the very top of the mobile one (y 3–34%) — so a centred crop
walks her face out of frame the moment the container's aspect stops matching the
image's. With `center center` her face was measurably **cut off entirely at
768px and 800px wide**, the top of her head was clipped from ~390px up, and she
was cropped vertically on ultrawide and short-window desktops (1920×700,
2560×1440, 3440×*). The anchors are:

| | value | what it protects |
| --- | --- | --- |
| mobile + tablet (≤1023) | `center top` | the crop goes width-bound as the width grows past 375 against the fixed 568px band, eating the top — where her head is. At 768–1023 the box is taller than it is wide, so the crop goes height-bound instead and eats the sides evenly, which `center` is the right answer for — her head is centred in this crop |
| desktop (≥1024) | `right top` | the crop goes height-bound below ~1270px and on short windows, eating the sides — where her head is; `top` also covers the vertical crop on ultrawide |

**At both design viewports (375×568 and 1920×912) the crop's aspect matches the
container's exactly**, so there is zero slack and the anchor is inert — the
composition Figma specified is untouched. It only engages off-design. Verified
across 39 container sizes: her face is fully in frame in all of them. The only
residual is the right edge of her *hair* below ~361px wide (≤76 source px, face
still clear by 43px), which an aspect-preserving band height under 375 would
remove if it ever matters.

⚠️ **Keep `.hero__bg-photo`'s scale as small as the parallax needs.** At the
design viewports the crops match the container's aspect, so `cover` leaves no
slack and that scale is the *only* source of parallax travel room — but every
bit of zoom crops tighter than the framing Figma composed. It is `scale(1.05)`
(2.38% per edge) against an **8px** shift cap in `initHeroParallax`. Because
`object-position` anchors the image's **top** edge, that 2.38% — ~13.5px on the
568px mobile band, the shortest container it runs on — is the whole budget above
the image; measured worst-case margin at full drift is 6.2px. Raising either
number without the other exposes the top edge (12px left only ~2px of margin).

Regenerating (if the source art changes): crop the two boxes above out of the
source with PIL and save as WebP at quality 82, sizes as listed. The prior
two-layer hero (`hero-red.webp` + `hero-people.webp`, a reconstructed wall and a
transparent people cutout) is **no longer referenced** — see §8 for the orphaned
files.

- **Cache-busting query strings:** Some `<img src>` values carry `?v=N`. These
  were bumped each time an asset on disk was replaced to defeat browser/Vite
  caching. If you replace one of these images, **bump the number**.
- **Carousel portraits are transparent WebP, cut at the exact card scale, and
  are TALLER than the card on purpose.** `carousel-portrait-faculty.webp`
  (786×**628**, Lisa Kraeger) was extracted from the Card Update Figma render
  at 1:1 with the 1440×600 card, with the panel background keyed out. The extra
  height is the part of the figure that **breaks out above the card's top
  edge** (28px), so the CSS positions it at a negative `top` and the card keeps
  `overflow: visible`. The phone on the student slide does the same (41px). Do
  not "fix" the overflow or re-crop these to 600.
  Because they are already card-scale, the desktop CSS drops them in at
  `left: 0` with `object-fit: fill` and **no** cropping or `object-position`
  tricks. They must stay truly transparent; a gray or black box behind a
  portrait means the asset was flattened on export, not a CSS bug.
  These are 1× extractions from the Figma render (the Figma MCP's
  `get_design_context`, which serves the original asset URLs, was erroring); for
  production, re-export the originals from Figma at 2× and keep the same pixel
  dimensions doubled.
  `carousel-portrait-alumni.webp` (Dr. Compton Moore) is left in the repo but
  **unreferenced** since the alumni testimonial became the WNBA card.
- **The WNBA partnership lockups are flattened brand art, not composed layers.**
  `wnba-capella-lockup.webp` (1984×815, side-by-side) and
  `wnba-capella-lockup-stacked.webp` (908×1352) each bake in the logos, the
  divider rule *and* the "official higher learning partner of the WNBA" line.
  They are the Figma source images recoloured to black over transparency and
  downscaled to 2× of their largest rendered size. The design **re-flows** the
  lockup rather than scaling one artwork, so a `<picture>` with
  `media="(min-width: 1024px)"` — matching the CSS breakpoint — picks one.
  Because the caption is pixels, the `alt` text is the only place that line
  exists for assistive tech and answer engines; keep it if you swap the asset.
- **Asset aspect ratios are tuned to their CSS slots.** The carousel portrait
  crops rely on each asset's ratio being close to its slot ratio (so
  `object-position: bottom` doesn't clip heads and `object-fit: fill` doesn't
  visibly warp). Swapping in an asset with a very different aspect ratio
  will reintroduce warping/clipping — re-check the carousel at all breakpoints.
- **Figma-sourced assets expire.** Original art was pulled via Figma MCP URLs
  that expire (~7 days). The committed copies in `public/assets/` are the source
  of truth now; don't expect the Figma URLs to still resolve.
- **CTA background video:** the closing "what are you waiting for?" section
  plays a single full-bleed TV-spot clip (`public/assets/videos/cta-tvspot*`)
  — see §12. The reduced-motion fallback is the clip's own poster frame, so no
  extra image is fetched. The previous three-clip set
  (`{leftLady,middleMan,rightLady}_loop.{webm,mp4}`) and its `cta-people.png` /
  `cta-mobile.jpg` stills have been **deleted** — recover from git history if
  that treatment comes back.

### 3d. Committed but unreferenced assets

Nothing in the page loads these — they're kept, not wired up. Listed so you
don't go hunting for the code that uses them:

| File(s) | Size | Note |
| --- | --- | --- |
| `hero-base.png` | 12.2 MB | Superseded, twice over — first by the two-layer WebP hero, then by the Phase 3 crops (§3a). |
| `hero.png` | 6.1 MB | Regeneration source for the **old** two-layer hero. No longer needed by anything the page loads. |
| `hero-red.webp`, `hero-people.webp` | ~346 KB | The old two-layer hero (reconstructed wall + people cutout), orphaned by the Phase 3 single-photo hero (§3a). |
| `footer-partner-{sei,strayer,jwmi}.svg` | ~30 KB | **In use** by the footer partner carousel (§13). |
| `footer-partner-devmountain.svg` | ~8 KB | **⚠️ Mislabelled — this is the SOPHIA wordmark, not Devmountain.** Don't wire it up by filename. |
| `footer-partner-sophia.svg` | ~1 KB | Sophia droplet **mark only**, not the wordmark lockup. |
| `footer-logos-strip.png`, `footer-partners-strip.png` | ~46 KB | The old flat 4320×210 strip, with the arrows *painted into the image*. Superseded by the carousel; kept as the slice source for `partners/{devmountain,sophia}.png`. |
| `footer-arrow-{next,prev}.svg` | ~0 KB | Empty files. |
| `cta-1/2/3.png` | ~2.2 MB | Legacy, see above. |
| `carousel-portrait-alumni.webp` | ~40 KB | Dr. Compton Moore — the alumni slide became the WNBA partnership card (§6). |

All of these are now safe to delete — including `hero.png`, which only existed to
regenerate a hero the page no longer has. That's ~21 MB of the repo's ~40 MB of
assets. Left in place because a few are plausible future art rather than clearly
dead, and because the Phase 3 hero is still under review.

Already deleted, so don't go looking for them: the three gold-backdrop clips
(`videos/{leftLady,middleMan,rightLady}_loop.{webm,mp4}`) and their
reduced-motion stills (`cta-people.png`, `cta-mobile.jpg`), ~6.7 MB, removed
when the closing CTA went back to the single TV-spot clip. Recover from git
history if that treatment ever returns.

---

## 4. Responsive breakpoints

Mobile-first base styles, with these override breakpoints (see `styles.css`):

| Breakpoint | Purpose |
| --- | --- |
| `max-width: 768px` | Mobile layout: stacked nav + mobile header, sticky utility bar, mobile type sizes, mobile carousel coordinates, **program-finder top stacks (title above chips)** |
| `max-width: 640px` | Phone: program-finder chips become a **2×2 grid** (`minmax(0,1fr) minmax(0,1fr)` — plain `1fr` won't shrink below the chips' content width and overflows; reduced chip `padding-inline` so labels fit), stats grid single-column |
| `max-width: 1023px` | **Phone/tablet carousel layout** (fixed `294 × 583` aspect card, absolutely-positioned elements scaled via container query) |
| `max-width: 1024px` | Tablet: hamburger nav, **program-finder top is the row layout** (title beside 2×2 chips) |
| `min-width: 768px and max-width: 1023px` | **Tablet hero** — its own composition, not a squeezed desktop: two full-bleed columns, photo left with the headline over it, solid dark form panel right. Step 2's five inputs go **2-up with email spanning**, and the panel's whole vertical rhythm tightens to fit 788px — see §5e |
| `min-width: 1024px` | **The hero's desktop breakpoint** — switches to the overlay composition (full-bleed landscape photo, transparent form left-aligned in the page measure, row-layout stepper, 3- and 5-across field rows). Lines up with the `(max-width: 1023px)` query on the `<picture>` portrait source. Also the **wide carousel layout** (`1440 × 600` card) |
| `min-width: 1024px and max-width: 1199px` | Step 2 stays **five across** (the reference does at 1024) but the error messages drop to 10px to fit — see §5b. Plus the small-desktop hero cap, `--hero-height-tablet-max` (760px) |
| `min-width: 1200px` | Desktop refinements: 4-across program-finder chips, content-band bg crop, etc. (the hero's sizing is owned by its own 1024px+ block, not this one) |
| `max-width: 1280px` / `min-width: 1920px` | `--page-gutter` adjustments only (in `tokens.css`) |

⚠️ **The 1023 / 1024 boundary is load-bearing twice over.** The phone and wide
carousel layouts are mutually exclusive and split exactly here (they use
different positioning systems — see §6), and so are the hero's tablet and
desktop compositions. If you shift it, audit both.

⚠️ **The hero is the one section that switches at 768, not 769.** Everything
else on the page treats 768 as mobile (`max-width: 768px`), but the reference
for the two-column hero is a 768-wide frame, so its block is
`min-width: 768px` and wins there by being later in the file. The practical
consequence: at exactly 768 the hero is in its tablet layout while the nav and
the program finder are still in their mobile ones. That is intentional; don't
"fix" it by moving one of them without the other.

---

## 5. Sticky header — edge cases

Two stacked sticky elements, on **both** mobile and desktop:

- `.utility-bar` → `position: sticky; top: 0; z-index: 101;` (height **40px**)
- `.main-nav` → `position: sticky; top: 40px; z-index: 100;`

The `top: 40px` on the nav is intentional — it pins the nav directly **below**
the 40px utility bar so both stay visible while scrolling. If you change the
utility bar height, **update the nav's `top` to match** (base rule + the
`≤768px` override both set this).

**Nav height = `.main-nav__bar` `min-height` (no vertical padding).** Per Figma
the global nav is **90px** (desktop, content 88), **72px** tablet, **67px**
mobile. The bar carries **no top/bottom padding** — content is centered by the
`min-height`, which alone sets the height (`88 / 72 / 67`). Don't re-add
`padding-block` to `.main-nav` or `.main-nav__bar`: it stacks on top of the
`min-height` (and the 44px hamburger) and inflated the header to 168px. So the
header total is **~128px** (40 + 88), not 168 — note the hero
`--hero-fold-reserve` values (§5 hero) were tightened ~40px to match the shorter
nav (desktop 360→320, tablet 460→420), so the hero now reaches its 755px cap on
standard desktops (≥~1075px tall) while the program finder stays above the fold.

`initNavScroll()` toggles `.main-nav--scrolled` (shrinks the nav) using
**hysteresis — on above 40px, off below 16px**, not one 24px threshold. A single
threshold made the nav flicker whenever the scroll position hovered on it
(trackpad momentum, rubber-banding), and the height change is transitioned, so
each flip was visible. Keep the two thresholds apart. `z-index: 100/101` on the
header sits above the parallax band (`z-index: 1`) and carousel content — keep
new stacking contexts below 100.

### Nav interaction states

Every interactive element in the header has hover / press / keyboard-focus
feedback, built from tokens in `tokens.css` (`--nav-pill-*`, `--nav-focus-ring`)
so they stay consistent — change the token, not the individual rules.

Specced in the **UI Elements** Figma
([utility bar `2001:2`](https://www.figma.com/design/mqSJTp9qWvsAU8n08FFlk9/UI-Elements-for-Homepage-Proto--Copy-?node-id=2001-2),
[global nav `2001:78`](https://www.figma.com/design/mqSJTp9qWvsAU8n08FFlk9/UI-Elements-for-Homepage-Proto--Copy-?node-id=2001-78)).
⚠️ **The two bars behave differently — don't unify them:**

| Element | Rest | Hover |
| --- | --- | --- |
| Utility links (phone, Log in) | plain | **underline** — *not* a pill |
| Request information | red fill, white text | **inverts**: white fill, `#c10016` text |
| Main nav links | plain | **rounded pill**, 48px tall, white @ 10% |
| Main nav links — **activated** | — | solid `--nav-pill-current` `#5e6361` pill, via `aria-current="page"` |
| Apply now | white fill, dark text | **inverts**: transparent + 2px white ring, white text |

- The main-nav pill is `48px` tall (Figma `gl-size-4xl`) — that's `12px` of
  block padding on a 24px line, not the padding you'd guess from the text.
- Its fill is white at **10%**, sampled from the Figma (the pill renders
  `#373b39` over the `#212322` bar).
- **Apply now's ring is an inset `box-shadow`, not a `border`** — a real border
  would change the button's size on hover and shift the whole bar. Its rule
  also resets `.btn:hover`'s `opacity`, which would otherwise just dim the
  outline once the fill is gone.
- Press adds a stronger fill plus a slight scale-down; `:focus-visible` is a
  white ring everywhere. The logo and hamburger have their own equivalents.

### Buttons and chips invert on hover

The same "invert" language runs through the rest of the UI — a **filled** rest
state becomes an **outlined** hover state, not a darker fill. Two places
implement it:

- **`.btn--white`** (both *Apply now* buttons): solid white → transparent with a
  2px white ring and white text
  ([hero `2001:456`](https://www.figma.com/design/mqSJTp9qWvsAU8n08FFlk9/UI-Elements-for-Homepage-Proto--Copy-?node-id=2001-448)).
  The hero's *Get started* button used to be the third usage — the Phase 3 hero
  replaced it with the RFI form, whose own pills are `.btn--primary` (red) and
  `.btn--outline` (the step-2 *Back*).
  The rule lives on the variant so all three behave identically. ⚠️ It assumes a
  **dark or photo backdrop** — true of all three current usages. A white button
  on a light background would need its own hover.
- **`.btn--secondary`** (the two carousel card buttons): the same move
  dark-on-light — solid black → transparent with a 2px black ring and black
  text, since these sit on the card's light grey panel.
- **`.btn--dark`** (action-CTA *Get started*): dark pill → transparent with a
  2px white ring.
- **`.btn--outline`** (*See all accreditations*): the reverse — the outline
  **fills white** and the text flips dark. Its 2px border exists at rest, so
  nothing resizes.
- **`.btn--primary`** (both red buttons — stats *See all Capella programs* and
  the program finder's *Explore my program*): red fill → **white fill with red
  text**, matching the utility bar's *Request information*. This is on the
  variant, so both red buttons behave the same; it replaced an earlier
  stats-only rule that inverted to a transparent white ring.
- **`.chip`** (program finder,
  [`2001:335`](https://www.figma.com/design/mqSJTp9qWvsAU8n08FFlk9/UI-Elements-for-Homepage-Proto--Copy-?node-id=2001-335)):

  | State | Fill | Border |
  | --- | --- | --- |
  | rest | `--color-chip-rest` `#4f4f4f` | none (transparent) |
  | hover | transparent | 2px `--color-chip-hover-border` `#8e8e8e` |
  | selected (`.chip--active`) | transparent | 2px `--color-stat-blue` `#94b7bb` |

  This is the **inverse** of the original implementation (which was outlined at
  rest and filled on hover) — don't "fix" it back.

### Stats section hover states

Specced in
[UI Elements `2001:548`](https://www.figma.com/design/mqSJTp9qWvsAU8n08FFlk9/UI-Elements-for-Homepage-Proto--Copy-?node-id=2001-548):

| Element | Rest | Hover |
| --- | --- | --- |
| `.stats-section__program` | dark glass card | **solid white fill**, eyebrow `#767676`, name `#505050`, arrow `--color-uni-red` |
| `.stats-section__cta` ("See all Capella programs") | red fill, white text | **white fill, red text** — now on `.btn--primary`, see above |
| `.stats-section__source a` (fact sheet) | underlined | **bold**, still underlined |

- The card rule is `.stats-section__program.glass-card:hover` — **two classes on
  purpose**, so it outranks `.glass-card:hover`, which would otherwise keep its
  translucent white wash and defeat the solid fill.
- The arrow is `stroke="currentColor"`, so setting `color` recolours it.
- The CTA's hover was later moved **onto `.btn--primary`** by request, so it and
  the program finder's red button match. There is no stats-specific rule for it
  any more.
- `.glass-card`'s diagonal shine sweep was **removed** (it was invisible against
  the new white fill). `.glass-card` is used only by these four cards, so the
  `::before` rules were deleted outright rather than scoped.

⚠️ **Rings are inset `box-shadow`s, and the chip's rest border is a transparent
2px, both for the same reason:** the element must not change size between
states. A real 0→2px border makes buttons resize and the whole chip row jiggle
on hover.
- ⚠️ **The pill's padding replaces the list gap — don't "restore" the gap.**
  `.main-nav__links` went from `gap: 30px` to `gap: 2px` when the links took on
  their own inline padding, which keeps text-to-text spacing at the same 30px
  *without widening the bar*. Adding the gap back overflows the bar at ~1025px.
  The same trick is in the utility bar: `.utility-bar__inner`'s `padding-left`
  is `5px` (not the Figma's 15px) because the links now carry 10px of their own,
  so the **text** still starts at 15px.
- **Inline pill padding is fluid** (`clamp(12px, 1.2vw, 24px)`) on purpose. The
  design's roomy ~26px only fits on a 1440-wide nav; this page's nav is
  narrower (the page gutter caps the container at 1080 on a 1440 viewport), and
  a fixed value overflows at ~1025px — the tightest width where the links are
  still shown rather than the hamburger.
- `.main-nav__links a` is `inline-flex` on purpose: `transform` is ignored on
  inline non-replaced boxes, so the press state would silently do nothing.
- **The activated state is keyed to `aria-current="page"`**, not a presentational
  class, so assistive tech gets the same "you are here" signal the fill gives
  sighted users. **No item carries it in `index.html`** — this is the homepage,
  and none of the four nav destinations is the current page; marking one would
  announce the wrong page to a screen reader. Add the attribute to a link (also
  works on `.main-nav__mobile-links`) when the nav is reused on a real section
  page. Its rule sits *after* `:hover`/`:active` at equal specificity so the
  current item keeps the stronger fill instead of appearing to downgrade to the
  hover wash when pointed at.

### Megamenus (`initMegaMenu`)

All four nav items open a dropdown. The information architecture — every label
and grouping — was lifted from the live **capella.edu** nav so the prototype
matches production; the `href`s are all `#` because this is a single page.

- **Degrees & Programs** is the two-column one (`.megamenu--split`): a dark rail
  of degree levels on the left driving a light panel of areas of study on the
  right, plus the red *Find your program* CTA. Left rail is `role="tablist"`,
  each level a `role="tab"` owning a `role="tabpanel"`.
- **Capella Experience / Financing / Admissions** are the narrow single-column
  ones (`.megamenu--list`): a 300px stack of grouped link lists, centred under
  their trigger.
- Only the **areas** level is reproduced under each degree level, not the
  individual programs (capella.edu reveals those at a third level). That
  matches the reference screenshot and keeps `index.html` reasonable — the full
  program lists would be 60+ more links.

Gotchas:
- ⚠️ **`.main-nav__item` is `position: static` on purpose.** The panel is
  absolutely positioned against `.main-nav` (sticky, so it's the containing
  block) to span the full header width. Give the `li` `position: relative` and
  the panel collapses into that one nav item's box.
- ⚠️ **`.megamenu[hidden] { display: none }` is required.** `.megamenu` itself
  sets `display: flex`, which otherwise beats the `hidden` attribute and the
  panels never close.
- Triggers stay `<a aria-haspopup>` rather than `<button>` — this is what
  capella.edu does, and it keeps all the existing `.main-nav__links a` styling
  (pill, hover, focus) applying unchanged.
- An open trigger holds the `--nav-pill-current` fill via
  `[aria-expanded="true"]`, so you can see which menu you're in while the
  pointer is down inside the panel.
- The degree rail responds to **`mouseenter` as well as click**, matching the
  real site. It is deliberately *not* wired to `focus`, or keyboard-arrowing
  through the rail would fight the roving selection.
- Dismissal: click outside, or `Escape` (which returns focus to the trigger).
- Hidden below the 1024px hamburger breakpoint — the mobile panel is the
  navigation there, and it is untouched by this.
- The mobile hamburger is a bare icon at rest but keeps `border-radius: 50%`,
  so its hover / press / open fills render as a circle rather than a square.
- Press feedback (the only motion) is disabled under `prefers-reduced-motion`;
  hover and focus colours still apply so nothing loses its affordance.

### Hero height: three layouts, three different principles

The CSS here is **mobile-first** (the reverse of the rest of the file) because
mobile is the case with structure:

- **`<768px`** — no `min-height` and no `overflow: hidden` at all. The photo is
  a fixed `--hero-photo-mobile` (568px) band pinned to the top of the hero, and
  the RFI panel sits below it in normal flow with its own solid
  `--color-uni-black` fill covering whatever of the band it overlaps. So the hero
  is exactly as tall as its content, and it grows when step 2 (which is taller)
  is showing. Nothing to tune.
- **`768–1023px`** — `min-height: var(--hero-height-tablet)` (788px), a **floor,
  not a cap**, and deliberately not tied to `svh`: the photo band is a fixed
  height, so a viewport-dependent hero would re-crop the photo on every resize.
  The form panel fills that height with its content centred, rather than the
  hero shrinking to the form. See §5e.
- **`1024px+`** — `min-height: min(var(--hero-height), calc(100svh - var(--hero-fold-reserve)))`,
  i.e. capped at the Figma height (912px) and shrinking on shorter viewports.
  `--hero-fold-reserve` is just the header (128px = 40 utility + 88 nav); the
  1024–1199 block swaps the cap for `--hero-height-tablet-max` (760px). Uses
  `svh` so mobile browser chrome doesn't break it.

⚠️ **The reserve no longer accounts for the program finder.** It used to: the
hero was sized to keep the finder above the fold. The RFI form is now *in* the
hero, so the form is the thing that has to be reachable, and capping the hero at
the design height is what achieves that. Don't re-add finder-sized reserve
values — that shrinks the hero and squeezes the form.

On desktop the hero content is **top-anchored and left-aligned** in the same
`min(--max-content, 100% - 2*--page-gutter)` measure `.page-container` gives the
rest of the page, which is what puts it at Figma's `x=240` on a 1920 frame. The
tablet layout drops that measure entirely — both columns are full-bleed.

The photo takes a **different `object-position` per breakpoint** (`center top`
below 1024, `right top` above) because the subject is off-centre in both
crops. This is load-bearing, not tuning — centred, her face leaves the frame
entirely at some common widths. See §3a for the measurements before changing it.

### 5a. Conditional questions

Three question groups are **hidden at rest** and revealed by `initHeroRfi()`.
The first two carry annotations in Figma (`345:27092`) that are the spec:

| Step | Block | Shows when | Source |
| --- | --- | --- | --- |
| 1 | RN licence (Yes/No) | area of study is **nursing** | Figma: "Only shows if nursing is selected" |
| 1 | Learning format (GuidedPath / FlexPath / both) | a **specialization** is picked and its area is in `FORMAT_AREAS` | Figma: "GuidedPath/FlexPath programs only" |
| 2 | Military education benefits (Yes/No) | the military question is answered **Yes** | requested directly, matching the paired-question layout |

Each one **clears its own answer when it closes**, so a stale "Yes"/"No" can't
be submitted for someone the question no longer applies to. The military pair
uses the same row container and 371/770 measure as step 1's pair
(`.rfi__questions` / `.rfi__followups`), stacking below 1200px.

⚠️ **The learning-format options are CONTENT-width, and the two wide questions
are uncapped.** Both were previously constrained and both were wrong:

- The options were `flex: 1 0 0` Fill columns, which made each one ~215–339px of
  mostly empty space and wrapped the labels mid-phrase ("GuidedPath:" /
  "Follow our schedule" on two lines). They are `flex: 0 0 auto` now and the
  three sit in one tidy row, labels on single lines. The `<br>` in the markup is
  gone too — it was forcing the wrap regardless of width.
- `--formats` and `--benefits` carried a `max-width: min(770px, 65.8%)`, added to
  stop the Fill columns spreading. With content-width options that cap is
  unnecessary, and on `--benefits` it was actively harmful: it wrapped "If you
  enroll at Capella University, do you intend to use education benefits earned
  through military service?" onto two lines, which the design never does.

The benefits legend needs **806px** on one line and the military legend 313px,
so the pair needs ~1149px of form. Step 2's first question is therefore
`flex: 0 1 auto` (content width) rather than the 371px basis step 1's pair uses
— at the fixed basis it was 30px short and wrapped. Step 1's RN question keeps
371px; its legend is two lines by design.

⚠️ **`FORMAT_AREAS` in `main.js` is a stand-in, not real data.** The true gate is
per-**program**, not per-area, and belongs on the program record. It is written
as an explicit `Set` so the assumption is visible in the diff rather than buried
in a boolean — replace it, don't extend it.

**Nursing + "No" is a dead end, not a validation error.** It reveals
`.rfi__disqualifier` ("…require a current, unrestricted RN license. Please select
a different area of study to continue.") **and** blocks step 2 — the copy only
makes sense if the form actually refuses to advance. Leaving nursing clears the
RN answer, so a stale "No" can't keep that block alive for another area.

These are the blocks the **hero mockup had switched off** (`Frame 9`, hidden), so
an implementation built from the hero node alone silently omits them. The
canonical source is the Form component, `variation=PMLP 2 step` (`345:30649`).

### 5b. Field states are a component contract, not decoration

`.rfi-field` implements the EC **Dropdown** (`329:14428`) and **Input field**
(`264:4641`) components at `size=med`. Nested parts, because the error message
attaches to the input's bottom edge as one continuous object rather than sitting
loose beneath it:

```
.rfi-field            the column
  .rfi-field__shell   the bordered object
    .rfi-field__box   the 48px input row — the border lives HERE
      .rfi-field__hint    12px label (selects only)
      .rfi-field__control the select / input
      .rfi-field__caret   the chevron (selects), absolute, centred in the BOX
      .rfi-field__check   the success tick (inputs), absolute
    .rfi-field__error the attached message bar
```

⚠️ **The caret must be its own element, not a `background-image` on the
`<select>`.** Figma (`329:15167`) makes the chevron a flex *sibling* of the text
column with `items-center` on the 48px row, so it is centred in the **box**. As
a background on the select it centres on the select's own line box — and since
the select is only the *value* line (the hint is a separate label above it), that
put the caret **8.2px below the box's centre**: visibly low, and it tracked the
text rather than the field. `.rfi-field__box--select` reserves the 39px Figma
leaves for it (16px padding + 15px icon + 8px gap) on the BOX, so the hint is
bounded too — on a narrow column it would otherwise run underneath. Measured
after the fix: 0.00px offset from centre at 375, 1200 and 1440.

(The `.program-finder__select` below the hero still uses a background-image
chevron. That is correct *there* — it is a single-line select with no hint, so
its line box and its field box are the same thing.)

The component also defines a 12px **helper-text** line below the shell. It is
deliberately **not** implemented: each dropdown already carries its own 12px
hint *inside* the box, and a second grey line under each one was just noise. To
bring it back, add `.rfi-field__help` inside `.rfi-field` after the shell (it
was removed in this form, not in the design).

| State | Dropdown | Input field |
| --- | --- | --- |
| inactive | white fill, 1px `#adadad`, 2px radius; 12px `#696969` hint over a 16px **bold** `#212322` value; red caret | same, minus the hint |
| focused | 2px **grey `#696969`** ring (outline, not a border swap — nothing reflows) | same |
| success | — (no such state) | `check` icon `#0a822f`, neutral border kept |
| error | border → `#d91111`, bottom corners squared, `#fff5f5` bar below with **12px bold** `#9c0c0c` | same |
| disabled | the **whole box** at 0.5 opacity — reads as a translucent panel with the photo through it | — (inputs are never gated) |
| autofilled | — (not an autofill target) | UA background clipped away, value forced back to `--field-text` |

⚠️ **The errored box keeps its bottom border.** The Figma error variant drops it,
so the box and its message read as one continuous object — which works when both
are dark. With the box filled **white** and the bar filled `#fff5f5`, dropping it
leaves the two pale surfaces bleeding into each other with no line at all. So
`.rfi-field--error .rfi-field__box` squares its bottom corners but keeps all four
border sides, and `.rfi-field__error` sets `border-top: 0` — that way there is
exactly **one** 1px `#d91111` line between them, not a doubled 2px one.

⚠️ **Autofill needs handling in both CSS and JS, and neither is obvious.**

*Visually:* Chrome and Safari paint an autofilled field with a pale UA
background and near-black text using a rule authors cannot override —
`background-color` and `color` are both ignored. On the red hero that turned the
name fields into pale blue boxes. `-webkit-background-clip: text` clips the UA
background to the glyphs — where `-webkit-text-fill-color: var(--field-text)`
then covers it — so the field keeps its own fill.

⚠️ `-webkit-text-fill-color` here **must** be `var(--field-text)`, not a literal
white. It was white while the fields were transparent over the photo; once they
were filled white, that painted the autofilled values white-on-white and made
them invisible. The `:autofill` and `:-webkit-autofill` rules are kept **separate**
because a browser that doesn't recognise one selector discards the entire rule
it appears in.

*Behaviourally:* **autofill never fires `blur`** — the browser fills the field
without it ever being focused — so the blur-only validation left autofilled
fields with no state at all: no tick on a valid value, no error on a bad one,
until the user happened to click in and out. `initHeroRfi()` listens for
`change` as well, which is what Chrome/Safari do fire on autofill.

Only the text inputs are covered. The three selects have no `autocomplete`
attribute and aren't plausible autofill targets; the rule is deliberately not
extended to them, because `-webkit-text-fill-color` on a `<select>` can leak
into its `option` list in some engines and those are styled dark-on-white.

Figma has **no disabled variant** for either component, so `.rfi-field--disabled`
follows what already shipped for the specialisation dropdown: dim the content and
leave the border at full strength, so the field still reads as a field. The class
is set by `syncChain()` alongside the `disabled` attribute — see §5d.

⚠️ **The two components disagree in Figma on the dark error colours.** The
Dropdown's dark variant points at `color/on/status/error/…-dark` (line `#ff3b3b`,
text `#ffa8a8`); the Input field's points at `gl-color/gl-on/status/error/…`
(line `#ffa8a8`, text `#ffffff`). The tokens in `tokens.css` use the second set
for both, so the two look like one system inside a single form. **This is a
design decision that hasn't been made yet** — worth resolving in the library
before it spreads.

⚠️ **The hero mockup sets the dropdown hint to 8px; the component says 12px.**
12px is used here (it's the library value, and it fits the 48px box once the
line-height is tightened to 1.2). Likewise the mockup's "Request program
information" is 28px against the component's 32px — the hero's 28px is kept,
since the hero is what's being built.

⚠️ **The fields are a LIGHT surface on a dark photo.** They went from
transparent → uni-black 20% → **white**, and every field token now resolves to
its light-theme value (tokens.css keeps the dark value in a comment beside
each). Anything added to a field has to be legible on white, not on the photo —
that is how the autofill rule nearly shipped painting values white-on-white.

Filling them white is also what finally closed most of the contrast failures,
because the backdrop stopped being a photograph and became a known colour:

| | transparent | 20% black | **white** | needs |
| --- | --- | --- | --- | --- |
| hint 12px | 2.80 | 3.57 | **5.49** | 4.5 ✅ |
| value 16px | 5.65 | 7.30 | **15.81** | 4.5 ✅ |
| focus ring (now grey `#696969`) | 1.04–1.32 | 1.38–1.71 | **5.49** | 3 ✅ |
| error border | — | — | **5.20** | 3 ✅ |
| error text on `#fff5f5` | — | — | **7.94** | 4.5 ✅ |
| red caret | — | — | **6.40** | 3 ✅ |
| border `#adadad` | 2.50–2.01 | 3.22–2.66 | **2.24** | 3 ❌ |

That includes the focus ring, which was the accessibility pass's one *critical*
finding — the DS blue on red was near-isoluminant at 1.0–1.7:1. The design uses
a **grey** ring on the white fill, which measures 5.49:1, so the fix and the
design agree. `--field-focus-ring` is grey for that reason; it is not the DS
`gl-color-outline-focused` any more.

⚠️ **The `#adadad` border is the one value that got worse** (2.24:1 on white).
It is no longer what identifies the field, though — a white box on a red
photograph is unmistakable, and 1.4.11 asks for 3:1 on the visual information
*required to identify* the component. Treat the border as decorative now. If you
want it to pass on its own, darken it toward `#767676`.

⚠️ **Error messages have to stay on ONE line in the five-across step-2 row, and
that row is what drives three breakpoints.** It is the tightest place in the
form. The longest message ("Please enter your first name") needs 165px of text
width at 12px, and the field only offers `width − 22px` of it once the bar's
padding, icon and borders are taken out.

| viewport | step-2 row | field | error size | bar padding / gap | slack |
| --- | --- | --- | --- | --- | --- |
| ≥1200 | 5 across, `column-gap: 12px` | 202–272px | 12px | 8 / 4 | huge |
| 1024–1199 | 5 across, same gap | 171–201px | 12px | 8 / 4 | **0.3px** at 1024 |
| 768–1023 | **2-up**, email spanning | 170–249px | 12px | **6 / 3** | **15px** at 768 |
| ≤767 | 1-up (mobile) | full width | 12px | 8 / 4 | huge |

⚠️ **There is no 10px tier any more, and the copy is now the constraint.** Two
bands used to drop the messages to 10px to fit. The strings were shortened
instead — measured at 12px: "First name is required" 126.7px, "Last name is
required" 125.3px, "Phone is required" 102.2px, "Zip code required" 102.9px,
"Email is required" 96.7px. The binding case is **1024, where a 171px field
leaves 138px and the longest message is 137.7px** — 0.3px of slack. **Re-measure
before rewording any message.** The wording this replaced ("Please enter your
first name", 165px at 12px) could not fit at 12px at any breakpoint below 1200.

- **Below 1024 no font size rescues five across** (it leaves ~148px fields), so
  that range goes 2-up instead. 1024 is the five-across floor.
- The 6px side padding and 3px icon gap in the 768–1023 band stay: they are worth
  5px, which is what keeps the 12px messages on one line at 170px fields.
- ⚠️ **A wrapped message bar is not just ugly** — it adds 15px to the panel, which
  at 768 pushes the hero past its photo band (§5e). That is why the fit is
  measured rather than eyeballed.
- The 8px original from the dark Figma variant is what all of this replaced — it
  fit easily and was unreadable.

**If a message gets longer, re-measure at 1024**, not at 1440. An alternative to
the 10px band, if it ever reads too small: shorten the two long messages to the
voice the others already use ("First name is required"), which fits at 12px
everywhere — but that changes copy taken from the reference, so it is a content
decision, not a CSS one.

⚠️ **The whole dropdown box must be the click target — check it if you touch
the layout.** `.rfi-field__box--select` is a single-cell **grid** with the hint
and the select stacked in it (`grid-area: 1 / 1`), the select stretched to fill
the cell and carrying the padding itself, and the hint set to
`pointer-events: none` so clicks fall through to it. Laid out the obvious way —
flex rows, hint above value — only **35%** of a 355×48 field opened the
dropdown: the top and bottom padding were dead, the left 16px and right 39px
were dead at every height, and the hint line hit the `<label>`, which focuses a
select but does *not* open it. It is **97%** now, the remainder being the 1px
border ring itself.

Two traps in that rule:
- The track must be `minmax(0, 1fr)`, not the implicit `auto`. An auto track
  sizes from the items' intrinsic contribution and, because both carry
  `width: 100%` (a cyclic percentage), it collapsed to the select's own content
  width — a 252px track in a 355px box, leaving 100px dead at the edges. The `0`
  minimum also stops a long option ("Doctor of Nursing Practice") widening it.
- The paddings are measured from the box's **content** edge, 1px inside the
  border, so they are each 1px less than the offsets they reproduce and sum to
  the 46px content height (21 + 20 + 5), not 48. Getting that wrong shifts both
  text bands down a pixel and makes the select overflow the box by 2px.

⚠️ **The radio's correct dot colour depends on whether the mark is filled —
it has been both.** `.rfi-radio__mark` is now filled **white** with a 2px grey
ring, going red (ring and dot) when selected, which is what the design shows and
measures 6.4:1 on white.
While the mark was *transparent* over the red photo, red-on-red measured
**1.42:1** behind the RN answer and **2.55:1** behind the learning-format row —
against the 3:1 a graphical state indicator needs — and the dot had to be white
(9.09:1) instead. So neither colour is right in the abstract: if the mark ever
goes back to transparent, the dot has to go back to white with it.

⚠️ **The Font Awesome kit script replaces every `<i>` with an `<svg>`.** It
carries custom classes across but obviously not the tag, so icon rules must key
off a class (`.rfi-field__error-icon`, `.rfi-field__check`) and never
`.rfi-field__error i` — an element selector silently stops matching the moment
the kit loads.

---

### 5c. The hero's height is reserved so the photo can't move

`reservePanelHeight()` in `initHeroRfi()` measures **both panels with every
conditional revealed**, takes the taller, and pins both to it with `min-height`.

This is not cosmetic. The photo is `object-fit: cover`, so its crop is a function
of the **container's aspect** — and the hero grows past its height cap whenever
the form needs more room. That made every step change and every conditional
reveal re-crop the photo: at 1280×800 the hero swung **672 → 783px** and the
visible slice of the image slid **321 source px sideways** as the form was filled
in. The subject visibly jumped. With the reservation the hero's height depends
only on the viewport, so the crop is byte-identical in all six states (step 1
bare / with follow-ups / with the disqualifier, step 2, step 2 + benefits, and
back again — verified). It also stops the buttons moving under the cursor on a
step change, and it is closer to the design, whose hero is a fixed height.

⚠️ **Desktop only** (`min-width: 1024px`) — the one range where the form is
transparent over a full-bleed photo, so the reserved height is invisible and the
only thing it can move is the crop.

Below 1024 the form sits in an **opaque** panel and the reserved height would
show. At 768–1023 reserving left ~300px of empty dark panel under step 1's
single button (the tallest state is step 1 with every follow-up revealed, in a
384px-wide column); on mobile the tallest state is ~911px. Both of those layouts
keep the photo still the other way instead — the photo is a **fixed-height
band**, so its crop doesn't depend on the form's height and there is nothing to
reserve against. Verified: zero drift across every state at 375 and 768.

⚠️ So the invariant "the photo never moves" is held by two different mechanisms
depending on the breakpoint. If you change the tablet or mobile photo band to
stretch with the hero, you have to turn reservation back on for that range —
and then deal with the empty panel.

⚠️ It runs on **init and resize only**, never from a `ResizeObserver` — it
mutates the very heights such an observer would be watching.

### 5d. Stepper tabs, and the three step-1 dropdowns as a gated chain

**The STEP 1 / STEP 2 labels are buttons, not a read-out.** Each `<li>` keeps the
progress rule and the `aria-current`; the label inside it is a real `<button>`
stretched to the full half, so the whole tab is a 44px hit area and it is
reachable by keyboard.

- **Back to step 1 is always allowed.** Forward to step 2 runs
  `step1Complete()` — the *same* gate as "Learn program details", via the shared
  `focusFirstProblem()` — so the two routes into step 2 cannot disagree about
  whether step 1 is done. Clicking the current tab is a no-op.


`syncChain()` in `initHeroRfi()`. Each link is disabled until the one before it
is answered, so the row can only be worked left to right:

```
degree  ──enables──▶  area of study  ──enables──▶  specialisation
```

- **Clearing a link tears down everything downstream.** Setting degree back to
  its placeholder empties *and* re-disables both area and specialisation;
  changing the area rebuilds the specialisation options from `SPECIALIZATIONS`.
  Without that, a stale specialisation could be submitted for an area that no
  longer applies.
- **The option list is rebuilt only when it actually changes**, keyed on
  `specSelect.dataset.forArea`. `syncChain()` runs on *every* select change, so
  an unconditional rebuild would wipe a valid specialisation the moment the user
  touched anything else.
- **The step-1 gate only flags fields that are enabled.** A gated link is empty
  *because* its predecessor is — painting it red would point at a control the
  user can't use and bury the field that actually needs them. The gate now walks
  them forward one field at a time (verified: degree → area → specialisation,
  focus following each).

### 5e. The tablet hero (768–1023) is a third composition

Two full-bleed equal columns: the **photo on the left** with the headline and
subtitle over its middle, the **solid dark form panel on the right**, starting
88px down from the top of the hero so the wall reads across the full width above
it. No page gutter on either column. Source: 768-wide reference frames — note
there is **no 768 frame in the Figma section** (it holds only 1920 and 375), so
those images are the spec and the numbers below were measured off them.

⚠️ `--page-gutter` at 768 is still the mobile value, because every other section
switches at 769. The hero doesn't use it here (both columns are full-bleed), so
it doesn't show — but don't reach for the gutter token in this block expecting
the tablet value.

Everything inside the panel is **inherited from the mobile rules** — full-width
blocks, stacked questions, full-width buttons with the submit above Back. That
is why the desktop block starts at **1024** and not 769: the tablet layout is far
closer to mobile's than to desktop's, so it overrides the handful of things that
differ instead of unwinding 13 desktop rules.

What differs:

| | value | why |
| --- | --- | --- |
| step-2 fields | 2-up, **email spans the row** | 170px each; five across would be 148px. Email is the odd one of five and the longest value |
| headline | `clamp(2.5rem, 5vw, 3.5rem)` | `.display-xl`'s `10vw` is sized to the **viewport**; in a half-width column that is 77px and stacks it three lines deep. `5vw` is the same 10vw *of the column*, and lands on the clamp floor (40px, one line, 320 of 354px) at 768 |
| error copy | 10px, bar padding 6 / gap 3 | §5b — 4.6px of slack, the tightest thing in the form |
| panel rhythm | tightened throughout | see the budget below |

⚠️ **The hero HUGS the form, and the change is animated.** The panel floats in
the middle of the right column (`align-items: center`, content-height) and the
hero is sized to it:

```
hero = clamp(--hero-tablet-min, panel + 2 * --hero-tablet-inset, --hero-tablet-max)
     = clamp(700px,            panel + 128px,                    952px)
```

`fitTabletHero()` in `initHeroRfi()` measures the panel and writes that as
`--hero-fit`; the tablet block has `transition: min-height` on `.hero`, and the
photo band hugs the hero, so the photo **scales with the form** rather than
snapping. Measured 700 → 782 → 812 → 822 → 827 → 828 across one step change.

| state | panel | hero | wall above / below |
| --- | --- | --- | --- |
| step 1 | 451 | 700 (min) | 124 / 124 |
| step 1 + formats | 635 | 763 | 64 / 64 |
| step 2 | 700 | 828 | 64 / 64 |
| step 2 + benefits | 828 | 952 (max) | 62 / 62 |
| step 2 + benefits + 5 errors | 903 | 952 (max) | 24 / 24 |

⚠️ **It has to come from JS, and that is not laziness.** The hero's height here
is content-driven, and a content-driven height change is not a *style* change —
so there is nothing for CSS to transition. `interpolate-size: allow-keywords`
does not help; it only covers animating to or from the `auto` keyword. Writing an
explicit px value on each change is what gives `transition: min-height` something
to interpolate.

⚠️ **The measurement is driven by a `ResizeObserver` on the panel.** That catches
every reason its height changes — step swap, a revealed follow-up, an error
message appearing — in one place. Safe here, unlike for `reservePanelHeight()`
(§5c): this writes the *hero's* height, and the panel's height doesn't depend on
it, so there is no loop.

⚠️ **Three earlier shapes were tried and are all worse:**

1. **Hero content-height, panel content-height** — a 539px hero on step 1, which
   jumped 250px on a step change and dragged the whole page up with it.
2. **Hero fixed at 788, panel stretched to fill it** — a black column down the
   whole right side.
3. **Hero fixed at the tallest state (952), panel floating** — consistent, but it
   holds ~500px of empty wall on step 1 to pay for step 2.

The clamp is what keeps the hug from being either of those: the min stops step 1
squeezing the photo into a letterbox, the max stops the worst state stretching
it, and both are in `tokens.css` with the measured panel heights beside them.

**`.hero__content` is taken out of the grid** (`position: absolute; inset: 0 50%
0 0`) and centred in the hero, so the copy's own length can never feed into the
hero's height and therefore the photo. Centring rather than anchoring is what
both reference frames show.

⚠️ **The photo does not rescale as the hero hugs — the hero reveals more of
it.** The band is a FIXED `height: max(--hero-tablet-max, 100%)` (952px),
top-anchored, with `overflow: hidden` on the hero. So `cover` runs once, at one
scale, and the hero shows the top 700–952px of it. Verified: the render is
660×1000 at a scale of 0.8799 in **every** state — step 1, step 1 + formats,
step 2 + benefits, and the error state.

Letting the band hug the hero instead (`inset: 0`) re-ran `cover` at every
height and visibly zoomed the subject on a step change — which defeats the point
of animating the hug. The `max(…, 100%)` is a safety net for the JS-off case,
where the hero can exceed its clamp; it keeps the band covering the hero rather
than leaving a gap.

⚠️ **ONE image, filling the whole hero, MIRRORED** (`transform: scaleX(-1)
scale(1.05)` on `.hero__bg-photo`). There is no second layer and no fill behind
the panel — the photo *is* the background, and the panel floats on it.

**The mirror is load-bearing geometry, not a style choice.** Measured in the
portrait crop: her **face** is at x 396–594 (53–79%) and the wall is on her
**left** (0–34%); to her right there is almost nothing (96%+). Full-bleed at 768
the crop scales 1.024×, so:

| | her face on screen | vs. the panel at 384+ |
| --- | --- | --- |
| unmirrored | 406–608 | entirely behind it — only her left shoulder shows |
| mirrored | 149–361 | **clear**, and the wall she stood left of now fills 507–768 |

She misses clearing the panel unmirrored by **21px of source**, and no
`object-position` or zoom closes it: full-bleed is width-bound, so there is no
horizontal slack to shift, and scaling up to create slack pushes her back right.
The trade is that the photograph is reversed — her hair parts the other way and
the sweater's colour-block runs the other way. No text or logo is on her, so
nothing reads backwards.

⚠️ **`scaleX(-1)` belongs in `transform`.** `initHeroParallax()` drives the
`translate` property precisely so the two compose instead of clobbering each
other (§7).

⚠️ **Four fake backdrops were tried behind the panel before this, and the reason
they were needed at all is that the photo runs out of wall to her right.** The
portrait crop ends ~100px past her shoulder, and in the original 4096×2303 source
(`hero-base.png`) the scene turns **beige (`#e3c08a`) from x≈2450** — it is a
wider set, not more red wall. Each workaround had its own tell:

1. **A flat colour** (`#e62f3d`, sampled at the seam) — read as **a red bar**
   once the panel floated and ~250px of flat field sat next to a lit wall.
2. **A vertical gradient** — the wall's seam column is not a gradient; sampled
   down source x 590–620 it runs `#e52d3f → #8e1d22 → #200906 → #520c05 →
   #46100c → #1a0f17 → #2e1e23 → #5b3f3c` because her hair and sleeve cross it,
   so any fitted ramp reproduces her body as **banding**.
3. **The desktop crop's left third** — also pure wall, but a different PART of
   it; the lighting falls off toward the left of the frame, so it rendered a
   **darker red with a hard vertical step** at the seam.
4. **The portrait crop's rightmost ~100px stretched 7.68×** — tonally perfect,
   but the bottom of that strip is her skirt and the rug, so it **blurred**.

Mirroring removes the need for any of them.

⚠️ **The framing tightens across the band and that is inherent.** The crop is
0.66 aspect against a box that goes from 0.81 (768×952) to 1.07 (1023×952), and
`cover` is width-bound, so at 768 it shows the full crop width and 82% of its
height, and by 1023 only 61% of the height — a close-up, her head near the top
edge. Her face stays clear of the panel throughout (149–361 at 768, 213–483 at
1023 against a panel at 511). Evening that out needs **a dedicated wide crop of
her with wall on her right**, which the source cannot provide.

`.hero__gradient` is pulled back to `right: 50%` in the same block — it exists to
seat the copy on the photo, and across the full band it would centre a
quarter-width to the right and put its darkest part under the panel.

The 788px is also why the panel's rhythm is tightened: step 2 lands at exactly
700px, so `88 + 700 = 788` and the band covers the whole column in the ordinary
states. On the mobile rhythm step 2 came to **852px**. Getting it to 700 took:
head gap 32→16, step-tab height 44→32 with 16→4px of padding under it, panel gap
24→14, prompt gap 12→8, field gap 16→12, legend margin 16→8, consent 13px/1.5 →
12px/1.4, panel bottom padding 40→24. The **total** is what matters — if a block
is added to step 2 here, re-measure rather than just adding a gap.

In the states where the form runs past 788 — a failed submit adds 25px per
errored row (863px with all five), the military follow-up adds a block — the left
column shows the hero's own background below the photo. That is why `.hero`
carries `background: var(--color-uni-black)` in this block: in the panel's own
colour the strip reads as the panel carrying on under the photo (mobile's
composition). In anything else it reads as a hole.

### 5f. The form's buttons reuse the stats CTA's recipe

The red CTAs — "Learn program details" and "I accept, get program details" — are
marked up **`btn btn--primary btn--lg`**, the same recipe as the stats section's
"See all Capella programs", **minus its arrow SVG**. So they inherit the page's
32px pill, 60px height, 20px label and its whole state set: hover/focus invert to
a white fill with red text, active goes `--color-boulder-50`, focus-visible adds
the white ring. There is no `.hero__rfi .btn` override any more — that is the
point. Verified identical to the reference at 768: radius 32px, 62px tall, 20px,
`16px 28px`, `rgb(193,0,22)` on white, and the same two matching `:hover` rules.

**Back** is `btn btn--outline btn--lg rfi__back` — the accreditation section's
"See all accreditations" recipe, and equally unoverridden: transparent with a 2px
white ring and white text, inverting to a white fill with `--color-uni-black`
text on hover/focus, `--color-boulder-50` on press, and the white ring on
focus-visible. Verified against that reference at 768: same fill, ring, radius
and the same five matching state rules. It is `--lg` where the reference is
`--md` (62px vs 52px, 20px vs 16px) only so it stands level with the red pill
beside it, and `.rfi__back`'s `padding-block: 14px` is what pulls it from 64px
(lg's 16px padding + a 30px line + the 2px ring) back to 60px.

⚠️ **Both of these reversed earlier passes.** The first made the pair square with
a shadow-only hover and colours locked in every state, on the grounds that
inverting a form's submit reads as a state change rather than a hover; the second
locked Back to a solid white fill with red text. Nothing in `.hero__rfi`
overrides either button now — that is what lets the shared `.btn--primary` and
`.btn--outline` states through. If either treatment is wanted back, scope it
under `.hero__rfi` again rather than editing those two classes, which six other
buttons on the page share. Note that a locked fill also hides the inversion that
signals focus, so it has to come back with its own `:focus-visible` ring.

⚠️ **A consequence of both buttons matching their references: on hover the red
CTA becomes white with dark-red text and Back becomes white with near-black
text** — two white buttons side by side, distinguished only by label colour.

⚠️ The labels fit on one line everywhere: the longest is 278px at 20px bold, plus
56px of padding = 334px, against 352px of panel at 768 and 345px at 375.

### 5g. On desktop the copy is capped so it clears her face

`--hero-copy-max` on `.hero` in the `1024px+` block, applied to
`.hero__content`, with `.hero__title` sized to match and
`.hero__subtitle { max-width: none }` so the container owns the measure.

The overlay lays the copy over the photo, and below ~1750 the two collide. The
cap is the geometry itself, not a curve fitted to sample widths:

```
--hero-title-size: min(72px, 42px + 0.0721 · (100vw − 1024px))
--hero-copy-max:   min(839px, 0.7726 · 100vw − --page-gutter − 321.5px)
```

**The headline is ONE LINE at every desktop width, 42px at 1024 ramping to 72px
at 1440.** Both ends are by request. One line is the binding constraint: the
whole string is **8.276× the font size** wide (measured, tracking included —
595.9px at 72px), so the cap must always be at least `8.276 × title-size`.

| viewport | cap | headline | lines | subtitle | vs. her hair | clears her face |
| --- | --- | --- | --- | --- | --- | --- |
| 1024 | 408 | **42px** | 1 | 3 lines | **2px in** | 107px |
| 1200 | 534 | 55px | 1 | — | 8px clear | 124px |
| 1440 × 1919 | 611 | **72px** | 1 | 2 lines | 40px clear | 161px |
| 1920 | 839 | **72px** | 1 | 2 lines | 244px clear | — |

⚠️ **The copy deliberately reaches into her hair at the narrow end.** At 1024 it
used to stop ~90px short of her (cap 303), which also left no room for a 42px
line. The cap is now fitted through two points — 408px at 1024, and 611px at
1440 where it is unchanged — which simplifies to the expression above. The old
`0.707 · --hero-height` term is gone, but the `--page-gutter` term stays, so the
cap still absorbs the gutter's step at 1281.

⚠️ **Her hair is not her face.** The bug this section exists for was the copy
crossing her FACE (−438px at 1024). Her face starts a further ~110–160px right
and is clear at every width in the table. Grazing her hair below ~1280 is the
intended treatment.

⚠️ **If either value changes, re-check `cap ≥ 8.276 × title-size`.** At 1024 that
is 408 ≥ 348; at 1440, 611 ≥ 596 — the tight pair. Fail it and the headline
wraps.

| term | what it is |
| --- | --- |
| `1.025·100vw` | where the render's right edge sits (the 1.05 parallax overshoot) |
| `− --page-gutter` | where the copy starts |
| `− 0.707·912px` | how far left `cover` pushes her at the design hero height |
| `− 64px` | clearance |
| `min(839px, …)` | Figma's text frame, which the ramp reaches at ~1750 |

Before the cap, the copy crossed her at every desktop width — measured
headline-right vs. her hair: −438px at 1024, −56px at 1281, −124px at 1440×1919.
The table above is where it lands now.

**It is the crop geometry, not the copy.** The hero is much squarer than the
crop's 2.104 aspect, so `cover` goes hard height-bound and only part of the
render's width is visible. `object-position: right` then holds her head a fixed
distance from the **right** edge while the headline's width is fixed from the
**left** gutter — so the narrower the viewport, the further the headline reaches
into her.

Four things this went through, all worth not repeating:

- ⚠️ **The viewport's HEIGHT is part of it**, which is why a width-only ramp is
  wrong. A taller window → a taller hero → `cover` scales the render wider → she
  moves further left. At 1440 the copy cleared her at 900px tall and ran across
  her at 1919px tall: same width, 90px more hero, her hair 64px further left.
- ⚠️ **But the height term must be the constant 912, not `min(912, 100svh −
  reserve)`.** The hero's height is the *larger* of that and its content, and the
  content wins on a short window: the svh form gave 772 at 1440×900 where the
  hero was really 918, and the copy landed 39px inside her hair. With the
  headline pinned to two lines the content tops out at ~920, so 912 plus the
  64px margin covers it.
- ⚠️ **`--page-gutter` STEPS at 1281** (6vw capped at 80 → 12.5vw, i.e. 77px →
  160px), moving the copy's left edge 83px right. Reading the token rather than
  hard-coding a slope is what absorbs that; an earlier hand-fitted ramp cleared
  her at 1280 and missed by 56px at 1281.
- ⚠️ **The cap and her position are coupled**, so a value cannot just be picked:
  capping the copy makes it taller → taller hero → she moves left, which is what
  the cap was chasing. A first pass at `37vw` with a 72px headline still
  overlapped by 38px. For the same reason, wrapping the headline cannot be used
  to buy room — wrapping adds ~90px of hero height and moves her ~64px left.

**The headline's size is derived from the cap** (`/6`) so it always wraps to
exactly two lines inside it: "CATCH WHAT YOU'RE" is 5.7× the font size, and at
`/5.7` it sat exactly on the boundary where font metrics could tip it to three.

⚠️ **This costs the Figma measure below ~1750.** At 1440 the copy is capped to
611px against Figma's 839px frame. Figma also specifies a 100px headline; that is
now **72px by request**, which is what makes a single line possible at all — at
100px the line is 837px and her hair starts at x≈831, so it could never clear
her. At **1920 the cap resolves to the full 839px** with 244px of clearance.

### 5h. The scrim: 20% black behind the copy, 0 on her face

`.hero__gradient` inside `.hero__background`. Figma draws it as a Linear fill at
20% layer opacity over the image (black 100% → black 0%).

⚠️ **Its DIRECTION changes per composition**, because "behind the copy" and "on
her face" are in different places in each one. All the stops below are measured,
not eyeballed — from a skin scan of the crop (her face and neck run **3.5%–35%**
of it, and there is **no skin at all below 37%**) and from where the copy
actually lands.

| | direction | stops | result |
| --- | --- | --- | --- |
| ≤767 | vertical, band-relative | `0 at 37% → 0.2 at 42%` | her skin ends at 206px (36%), the copy starts at 240px (42%) — **0 on her face, full 0.2 behind both headline and subtitle** |
| 768–1023 | vertical, band-relative | `0 at 34% → 0.2 at 37%` | 0 over her face (chin at 320px), 0.2 below it |
| ≥1024 | horizontal, length stops | **`0.32`** to `gutter + copy-max` → `0` at `--hero-face-left` | 0.32 flat behind the whole copy, 0 from her face on — at 1440 that is out to 791px, clear by 952px, face measured at 1007px |

⚠️ **Desktop is 32%, not 20%, and that is deliberate.** It is the one composition
where the copy sits over the LIT wall rather than over her. Measured at 1024, the
photo behind the copy is `rgb(199,13,37)` and white on it is **5.99:1 bare,
8.28:1 at 20%, 10.07:1 at 32%**. The copy was reported as hard to read at 8.28 —
a good luminance ratio is not the whole story on a saturated red, where white
vibrates — so this one is darkened past the Figma value. The other two stay at
20% because the same copy lands on her dark sweater there and already measures
**10.5:1 (headline) and 15.7:1 (subtitle)**.

**Why the desktop one holds before it ramps.** A plain two-stop 0.2 → 0 across
the frame is what Figma draws, but it decays as it goes — behind the middle of
the headline it measured only 0.10. Holding 0.2 to the copy's right edge (the
same `gutter + --hero-copy-max` the text is bounded by) and ramping from there is
what "20% behind the text, 0 on her face" actually asks for.

⚠️ **The 768 ramp is only 3 percentage points (28px) and that is forced.** At
that breakpoint the centred copy starts 8px below her chin, so "0 on her face"
and "0.2 behind the headline" are 8px apart. A gentler 34→42% ramp left the
headline on 0.06. It still reads as a soft edge rather than a band because the
entire delta is 0.2 of black.

⚠️ **At 768 the copy moves and the scrim does not.** Her face is fixed (the band
is a fixed 952px, top-anchored) but the copy is centred in a hero that ranges
700–952px, so it travels ~250px against a fixed ramp. In the shortest state
(step 1, hero 700) the headline rides up to y≈276 — above her chin at 320 — and
so sits above the scrim entirely; the subtitle below it still gets the full 0.2.
Darkening the headline there would mean darkening her jaw. **A hero-relative ramp
does not fix this** — it was tried first and drifts the other way, ending up
between her chin and the headline. What would fix it is anchoring the copy below
her chin instead of centring it, which changes the composition, so it has not
been done.

⚠️ **This replaced a radial black at 0.7 centred bottom** (the earlier Figma
spec). Much lighter, and safe: white on the wall red measures ~8:1 before any
scrim at all.

---

## 6. Carousel — the trickiest component

`initCarousel()` in `main.js`. Pointer-based drag/swipe with snap.

- **Drag vs. scroll intent:** the first few px of a pointer move decide whether
  the gesture is horizontal (carousel drag) or vertical (let the page scroll).
  Don't remove the `Math.abs(dx) > Math.abs(dy)` check or vertical scrolling
  breaks on touch.
- **Click suppression:** a real drag sets `moved`, and a capture-phase `click`
  handler cancels the click so links/buttons inside a slide don't fire after a
  swipe. Keep this if you add interactive elements to slides.
- **Two layout systems, by breakpoint:**
  - **≤1023px:** card is a fixed `294 × 583` aspect box. Children are
    absolutely positioned using a container-query unit:
    `--px: calc(100cqi / 294)`, e.g. `top: calc(284 * var(--px))`. This keeps the
    card from ballooning in height on narrow screens. Coordinates map directly
    to Figma pixel values.
    - **Don't use fluid font scaling for the card body text here** — it was
      capped to a fixed size because large fluid text overflowed the fixed-height
      card.
  - **≥1024px:** card is `1440 × 600` — the card **is** the panel (the old
    `1440 × 642` box with a 42px top inset is gone). `overflow: visible`, so the
    faculty portrait and the student card's phone intentionally **extend above**
    the card top.
    - **Content is anchored to exact Figma coordinates**, not flex gaps. Each
      slide's content (`--student` / `--partner` / `--faculty` modifier on
      `.carousel__content`) absolutely positions its title / body / attribution /
      button at the Figma `y` via the `--px` unit, which avoids vertical drift
      from accumulated line-height. Measured off the Card Update render:

      | | title | body / attribution | button |
      |---|---|---|---|
      | student | 181 | 328 | 416 |
      | partner | — | lockup 77 (992 wide) | 478 |
      | faculty | 158 | name 343 · role 387 | 461 |

      The faculty quote runs to four lines and the student headline to two, so
      their titles start at different Y — the design lands them on a shared ink
      bottom rather than a shared top. The role sits 44 below the name: the 36px
      name line box (40 × 0.9) plus the design's 8px auto-layout gap, written
      into the Y because the offsets are absolute.

      The student and faculty columns start at `x 758` and are `600` wide (the
      student body alone is `557`, which is what makes it wrap where the design
      does). The **partner card is the exception**: it has no text column, so
      `--partner` spans the full 1440 and centres both children. Its lockup sits
      ~21px right of centre in the Figma frame while the mobile card centres the
      same art exactly (`28 + 238/2 = 147`, half of 294) — the build centres it
      at both sizes rather than reproducing that offset.
    - **Type scale comes from the Figma file's own variables** — quote `20`
      Inter *italic* / 1.5, body `16`, name `40` Acumin Extra Condensed
      Semibold / 0.9 uppercase, role `16`, button `20`
      bold. The quote is **not** the big condensed display face; if it starts
      rendering as large uppercase display type, a `.carousel__title` override
      has crept back in.
    - **Buttons** ("Is FlexPath right for you?", "Join the dream team",
      "Full bio") are scaled to the Figma `60px` pill (`padding 16/28`,
      `font 20`, `radius 32`) via `--px` — do **not** let them fall back to the
      unscaled `.btn--lg`, which stretches full-width. The partner pill carries
      a `min-width` rather than a fixed width at both breakpoints: the design's
      194 (mobile) / 211 (desktop) were measured against a shorter label, and a
      fixed box wraps a longer one onto a second line.
    - **Portrait sizing.** The assets are pre-cut at exact card scale, so each
      `.carousel__portrait` is simply `left: 0` at its asset's own dimensions
      with `object-fit: fill` and a **negative `top`** for the overhang (faculty
      `-28`). There are no crop slots, scales or
      `object-position` tricks any more — if you find yourself adding one, the
      asset is probably the wrong size. See §3 for the asset contract.
    - ⚠️ **The phone mockup is the exception — do NOT re-derive its `left` and
      `width` from the Figma render.** `.carousel__image--mockup` inside it is
      positioned in *percentages of the mockup box*, so resizing that box
      rescales the phone within its crop and drags the phone's visible top edge
      down, silently killing the overhang. Only `top` and the bottom
      `clip-path` should change when the card's height changes. Also note
      `top` positions the **box**, whose top sits ~78px above the phone's
      visible top: `-119` is what puts the phone 41px above the card.
    - **The faculty attribution needs an explicit `width`.** Its children are
      absolutely positioned, so without it they inherit the wrapper's
      shrink-to-fit width (the name) and the two-line role wraps into a narrow
      column.
- **People are bottom-anchored at every width.** Portrait containers pin to the
  card's bottom edge. Below 1024 the images use `object-fit: cover` with
  `object-position: center bottom`; at ≥1024 the pre-cut assets sit 1:1 with
  `object-fit: fill`. If figures float off the bottom after an edit, check these
  two properties.
- `goTo(activeIndex, false)` re-runs on `resize` to recompute the step width.

---

## 7. Animations & microinteractions

All live in `main.js`, initialized on `DOMContentLoaded`. Every one is
**gated on `prefers-reduced-motion`** (see §8).

| Function | What it does | Trigger |
| --- | --- | --- |
| `initTextReveal()` | Splits target headings into per-word spans (`.word` mask + `.word__inner`) that rise up from behind a clip mask, staggered via `--word-index`. **`TEXT_REVEAL_SELECTORS` is deliberately down to two headings** — hero + stats. The tiles, accreditation, action-CTA and program-finder ("Catch what you're chasing") headings were all removed by request; don't add them back unless asked. | IntersectionObserver (per heading) |
| `initRevealAnimations()` | Fade-up for elements with `.reveal`. Optional stagger via `data-reveal-delay="N"` (× 80ms). | IntersectionObserver |
| Carousel card reveal (in `initCarousel()`) | Marks the card `.is-visible`; the movement itself is `initCardScroll()` below. The card's **inner text does not animate** — it rides in with the card. (An earlier version staggered title → body → button; removed by request.) | IntersectionObserver (first view) + `goTo()` + safety timeout |
| `initCardScroll()` | **Scroll-driven** (scrubbed, not timed) slide-in for the carousel cards: their `translate` tracks the carousel's position in the viewport, spread over ~90% of a viewport height so it's slow. **Ratcheted** — it only ever moves toward settled, so scrolling back up never pushes the cards out again. | `scroll`/`resize`, throttled with `requestAnimationFrame` |
| `initCountUp()` | Animates the stats numbers (40 / 80 / 1,530+ / 63%) counting up with a custom cubic-bezier ease. Preserves prefixes/suffixes/grouping. | IntersectionObserver (threshold 0.4) |
| `initParallax()` | Translates the content-band background image on scroll for depth. | `scroll`/`resize`, throttled with `requestAnimationFrame` |
| `initHeroParallax()` | The ONLY motion on the hero photo (a 13s ambient Ken Burns zoom/pan was removed — it read as the wall moving on its own). The hero is one frame now, so the **whole photo** (`.hero__bg-photo`) drifts together. Driven off `window.scrollY` so it responds from the first scroll pixel; drifts it down up to 12px. Overshoot comes only from the CSS `scale(1.05)` — see §3a before changing either number. | `scroll`/`resize`, throttled with `requestAnimationFrame` |
| `initContentParallax()` | Floats `.program-finder` (factor 0.2) up as you scroll, **capped so it can never cover the hero form** (§7a). **Driven off `window.scrollY`, NOT off each element's `getBoundingClientRect().top`** — a viewport-relative formula is already non-zero for anything on screen at load, which shoved the hero headline ~100px above its laid-out position. Must read 0 at `scrollY === 0`. ⚠️ **It deliberately skips `.hero__content` whenever `.hero__rfi` is present** (i.e. always, now): the copy would slide up away from a form that stays put, opening a growing gap between a heading and the fields it introduces. The form itself is excluded from every parallax on purpose — drifting selects and inputs are miserable to use. The old `heroContent` branch and its `offsetTop - 16` clip cap are still in the function for whenever the hero goes back to being purely decorative. | `scroll`/`resize`, throttled with `requestAnimationFrame`; plus a `ResizeObserver` for the cap |
| `initHeroRfi()` | The hero RFI's step 1 ⇄ step 2 swap (toggles `hidden` on `[data-rfi-panel]`, syncs `.rfi__step--current` + `aria-current` on the stepper); the **gated degree → area → specialisation chain** (§5d), whose options read the **same `SPECIALIZATIONS` map the program finder uses** so the two can't drift apart; the **conditional step-1 follow-ups** and the military-benefits follow-up (§5a); **per-field error / success / disabled states** (§5b); and the **height reservation** that keeps the photo still (§5c). Step 1 gates itself rather than calling `reportValidity()`, because the form is `novalidate` (so "Learn program details" — a next, not a submit — doesn't fire browser bubbles); the gate covers the three selects, any *visible* radio group, and the nursing dead end. Every step change focuses with `{ preventScroll: true }`: the panels are different heights, so a plain `focus()` scrolls the page to chase the field and drags the headline off screen. | direct listeners on the next/back/submit buttons, plus `change` on the selects/radios and `blur`/`input` on the text fields |
| Card hover scale | CSS-only `transform: scale(1.02)` on `.stats-section__program:hover` (replaced the removed VanillaTilt 3D tilt — it caused a "jiggle"). Kept deliberately, on top of the card's white hover fill. Disabled under reduced-motion. | hover |

> **Removed:** `initTilt()` / VanillaTilt. The cursor-following 3D tilt on the
> popular-program cards read as a jiggle and was replaced by the CSS hover scale
> above. The dependency is still in `package.json` but unused (§1).

### Text-reveal details / gotchas
- Headings that get the effect are listed in `TEXT_REVEAL_SELECTORS`. To add
  one, append its selector — `splitWords()` preserves `<br>` line breaks and
  inter-word spacing automatically.
- **Carousel titles are intentionally excluded from `splitWords()`** — they
  contain a decorative quote-mark span and live in an absolutely-positioned
  layout, so word-splitting would break them. They still move — they ride along
  with the whole-card **carousel card reveal** below (no per-word splitting).
- `.word` uses `overflow: hidden` with `padding-bottom: 0.12em` +
  `margin-bottom: -0.12em` so the clip mask has room for descenders without
  shifting layout. Keep this if you change heading line-heights.

### Carousel card motion details / gotchas
- **All of the card's motion comes from `initCardScroll()`**, which writes an
  inline `translate` every scroll frame. `.carousel-reveal` itself is only
  `will-change: translate` — there is **no** opacity fade, no CSS transition and
  no reduced-motion block on it, because there is nothing timed to disable.
- **It animates the `translate` property, not `transform`.** Deliberate: the
  carousel **track** uses `transform: translateX()` for navigation, so a
  `transform`-based card animation would clobber it. `translate` is a separate
  property, so the card's offset composes with the track's transform instead of
  fighting it. The card starts at `translate: 45% 0` (`START_OFFSET`) and scrubs
  to `0`.
- **The slide-in is ratcheted.** `initCardScroll` keeps a `revealed` value that
  only ever moves toward 0, so scrolling back up never pushes the cards out
  again.
- Reduced motion: `initCardScroll()` returns early, so no inline `translate` is
  ever written and the cards simply sit where they're laid out.
- ⚠️ **The `is-visible` reveal path is now vestigial.** `revealSlide()`, the
  one-shot IntersectionObserver on `.carousel__viewport` (`carouselSeen`), the
  2.5s safety `setTimeout`, and `goTo()`'s call into `revealSlide()` all still
  run, but **no CSS reads `.carousel__card.is-visible` any more** — the only
  `.is-visible` rules left are `.reveal.is-visible` and
  `.reveal-text.is-visible .word__inner`, neither of which matches a card. That
  machinery existed to drive the per-element text stagger, which was removed. It
  is harmless but dead: either wire new hover/reveal CSS to it or delete it —
  don't assume it is doing something.

### Parallax details / gotchas
- The bg image has built-in **vertical overshoot** (`height: 116%; top: -8%`),
  giving the transform room to move without exposing a band edge.
- JS amplitude (`rect.height * 0.06`) is deliberately **less than** the 8% CSS
  overshoot. If you increase the amplitude, increase the overshoot too or the
  band edge will show.
- **The desk image starts partway down the band, not at the top.** Per Figma the
  "Content Section Background Image" begins ~lower-third of the carousel, so
  `.content-band__bg` is offset (`top: var(--content-bg-top, 26%)`) with a top
  mask fade — the area above stays page-black. Adjust `--content-bg-top` to move
  the desk's start up/down.

### 7a. The program finder's drift is capped on the hero form's clearance

`.program-finder` drifts **up**, so it rides over the hero's **bottom** edge.
That edge used to be spare photo. It now holds the RFI form's action buttons, and
the full 120px travel covered them at every breakpoint — by 26px on desktop
step 1, 50px on step 2, and 72px on mobile, where the only slack is the panel's
48px bottom padding.

`measureFinderRoom()` now caps the travel at the empty space actually below the
form: `hero.offsetHeight − actionsBottom − 16`. It measures from
`.rfi__panel:not([hidden]) .rfi__actions` — the buttons are the real constraint,
and measuring the panel's own box instead would throw away the mobile panel's
bottom padding, which is legitimately coverable. `offsetTop`/`offsetHeight` are
layout values, so they ignore the `translate` this function applies and the read
can't feed back on itself.

Result: a guaranteed **16px minimum clearance** everywhere, with the parallax
still running at full strength where there's room (step 1 with no follow-ups
showing reaches the whole 120px; step 2 with the benefits question caps at 48).

⚠️ **The clearance is measured every frame inside `update()`, never cached.**
Two attempts at caching it both shipped stale values:

1. A step-change hook fired *before* the conditional reveals had reflowed — 6px
   stale, eating a third of the gutter.
2. A `ResizeObserver` on the hero and form fixed that, but then
   `reservePanelHeight()` (§5c) pinned the hero to a constant height — so
   stepping 1 → 2 moved the buttons down inside it while **nothing changed
   size**. The observer is blind to that, and the finder covered the step-2
   buttons by 56px. Revealing the benefits question did the same thing inside a
   pinned panel, for another 8px.

Enumerating the triggers is a losing game, so the value is simply recomputed
each frame. The reads are `offsetTop`/`offsetHeight` on three elements and they
all happen before the function's only write, so there is no read-write thrash.
The `ResizeObserver`, the `rfi:stepchange` event and a delegated `change`
listener remain, but only to **re-run** `update()` when the layout shifts while
the page isn't scrolling — otherwise a shrinking clearance wouldn't apply until
the next scroll.

---

## 8. Accessibility notes

- **Reduced motion:** `prefers-reduced-motion: reduce` is honored everywhere.
  - JS: each `init*` animation early-returns or jumps to the final state. Text
    reveals render fully visible; counters skip to final values; parallax/tilt
    are disabled.
  - CSS: a `@media (prefers-reduced-motion: reduce)` block neutralizes `.reveal`,
    `.reveal-text .word__inner`, and the glass-card sheen.
  - **When adding any new animation, add both the JS guard and (if CSS-driven) a
    reduced-motion override.** This is a hard requirement for this project.
- **Screen readers & split text:** `splitWords()` keeps real space text nodes
  between words, so headings still read as normal sentences. Don't strip the
  whitespace nodes.
- **Semantics already in place:**
  - Carousel dots are `role="tab"` with `aria-selected`; the viewport is
    keyboard-focusable (`tabindex=0`) with ←/→ arrow support.
  - Program-finder chips are `role="tab"` controlling a `role="tabpanel"` that is
    `hidden` until expanded.
  - Mobile menu button uses `aria-expanded` / `aria-controls`; the panel toggles
    the `hidden` attribute.
  - Decorative images use `alt=""`; meaningful images have descriptive `alt`.
    Decorative background containers use `aria-hidden="true"`.
- **Things to watch / improve:**
  - Focus styles: confirm visible focus rings on all interactive elements
    (links, chips, dots, buttons) before launch — verify against brand styling.
  - Baked-in copy: the WNBA partnership lockups carry their "official higher
    learning partner" line as pixels, so it can't be resized, translated or read
    by a screen reader — the `alt` text is the only accessible copy of it.
  - The carousel auto-snaps on drag but has **no autoplay** (good for a11y —
    don't add autoplay without a pause control + reduced-motion handling).
  - Headings: keep a single `<h1>` (hero) and logical `<h2>`/`<h3>` order if you
    add sections.

---

### Fixes from the 2026-09-22 WCAG 2.1 AA pass

1. **`.rfi-field__box` is `min-height: 48px`, never `height`** (1.4.4 Resize
   text). At a fixed height the box could not grow with its own text: under
   text-only zoom the three dropdowns' stacked hint + value measured ~100px
   against 46px of content box. Verified after: at 2× the box grows 48 → 89px,
   nothing clipped, and it is still exactly 48px at 100%.
   The select's `padding-top` went to **1.3125em** in the same fix — the two
   bands are grid-stacked in one cell, so that padding is the only thing holding
   the value clear of the hint, and in px it stayed 21 while the hint's line
   grew (they overlapped at 2×). In em it scales: 8px of clearance at 2×.
2. **The gated selects are explained non-visually** (3.3.2). Area of study and
   specialization are `disabled` until the field before them is answered, which
   removes them from the tab order — step 1 was announced as a one-field form.
   A `.visually-hidden` paragraph in `.rfi__prompt` now states the order. It
   cannot live on the selects: a `disabled` control's `aria-describedby` is never
   read out.
3. **Error messages shortened so they fit at 12px** — the 10px tier is gone. See
   §5b for the measurements; the binding case is 1024 with 0.3px of slack.

Still open, by choice:

- The `#adadad` field border is 2.24:1 against its own white fill. Not a 1.4.11
  failure — what identifies the field is the white fill on the dark panel
  (15.81:1) — so the border is decorative. Darken toward `#767676` if it should
  pass on its own.
- 2.5.5 (44×44 targets) is **AAA**, not AA. The 32px stepper tabs and 24px radio
  labels clear the AA criterion (2.5.8, 24px, WCAG 2.2); the inline privacy link
  is exempt under that criterion's inline exception.
- One VoiceOver pass on the step change is still worth doing by hand.
  `role="alert"` on a bar going from `hidden` to visible works in current
  browsers, but a persistent live region is the more robust pattern.

## 9. Image / performance optimizations already applied

- **Hero (LCP):** `<link rel="preload" as="image" fetchpriority="high">` in
  `<head>` + `fetchpriority="high"` on the `<img>`.
- **Below-the-fold images:** `loading="lazy"` + `decoding="async"`.
- **Above-the-fold / prominent images** (nav logos, content-band bg): eager but
  `decoding="async"` (the parallax band is kept eager on purpose to avoid
  pop-in during scroll).
- Images with intrinsic `width`/`height` keep them to avoid layout shift (CLS);
  the rest are CSS-sized via `object-fit`.

- **Tile images were downsized.** `tile-finish.png` (was 4096×4096 / 28 MB) and
  `tile-apply.png` (was 3000×2112 / 8.6 MB) rendered in ~380px boxes and loaded
  far slower than the others; they're now ~1000–1200px / ~1.6–1.8 MB, in line
  with the rest. If you re-export these, keep them ≲1200px on the long edge.

### Suggested next steps (not yet done)
- Convert the remaining large PNGs (`content-band-desk.png`,
  `carousel-phone-mockup.png`) to **WebP/AVIF** with a PNG fallback via
  `<picture>`. These are the biggest image payloads left; the carousel portrait
  and the hero layers are already WebP. (The tiles are now reasonable — see
  above.)
- Add `srcset`/`sizes` for the hero and CTA art to serve smaller files to phones.
- Self-host fonts (or add `&display=swap` is already set for Inter) and consider
  preloading the primary display font to reduce FOUT on the hero headline.

---

## 10. Browser support & assumptions

Relies on reasonably modern browser features — verify if you must support older
browsers:

- **CSS container queries** (`container-type`, `cqi` unit) — core to the ≤1023px
  carousel. No fallback is provided.
- **CSS `@import`** of `tokens.css`, custom properties, `clamp()`,
  `aspect-ratio`, `object-fit`/`object-position`, `backdrop-filter` (glass UI;
  has `-webkit-` prefix), `inset`.
- **JS:** ES modules, `IntersectionObserver`, Pointer Events, `matchMedia`.
- `backdrop-filter` is the one most likely to degrade — on unsupported browsers
  the glass panels fall back to their semi-transparent background (acceptable).

---

## 11. Quick "where do I change…?" index

| I want to change… | Go to |
| --- | --- |
| Colors, type scale, spacing, easings | `css/tokens.css` |
| A breakpoint's layout | the matching `@media` block in `css/styles.css` (§4) |
| Which headings animate in | `TEXT_REVEAL_SELECTORS` in `js/main.js` |
| Carousel behavior / drag | `initCarousel()` in `js/main.js` |
| Carousel card slide-in (direction / distance / trigger) | `.carousel-reveal` on `.carousel__card` in `index.html`; `.carousel-reveal` rule in `css/styles.css` (`translate: 18% 0`); `revealSlide()` + safety timeout in `initCarousel()` (§7) |
| Stat numbers or count-up speed | the markup values + `data-count-duration` attr (`js/main.js`) |
| Stat number size / overlap | `.stats-section__value` font is `min(clamp(…12.8vw…), 44cqi)`; each `.stats-section__stat` is a container so the value scales to its cell and can't overflow into the next stat |
| Hero height | desktop: `min-height: min(var(--hero-height), calc(100svh - var(--hero-fold-reserve)))` in the `1024px+` block; tablet: `--hero-height-tablet` (788px) as a floor, in the `768–1023` block (§5e); mobile: **no** `min-height` at all, the hero is content-tall (§5). The reserve is just the header now — it no longer reserves room for the program finder |
| The hero's two-column tablet layout | the `@media (min-width: 768px) and (max-width: 1023px)` block in `css/styles.css` (§5e). The hero hugs the panel via `fitTabletHero()`; the bounds are `--hero-tablet-min/max/inset` in `tokens.css`. Step 2 fits a 700px budget — re-measure the whole stack, don't just add a gap |
| How much wall shows above/below the floating panel at 768 | `--hero-tablet-inset` (64px) in `tokens.css`, applied by `fitTabletHero()` (§5e). The min/max clamp is what it degrades to at the ends of the range |
| The photo behind the panel at 768 | the photo is full-bleed and **mirrored** (`scaleX(-1)` on `.hero__bg-photo`) — §5e. The mirror is what puts her face clear of the panel and the wall behind it; unmirrored her face is at 406-608 against a panel at 384. ⚠️ Don't un-mirror it without re-reading §5e, and don't add a backdrop layer — four were tried |
| Whether the photo rescales as the hero grows at 768 | the band's fixed `height: max(--hero-tablet-max, 100%)` (§5e). Hugging the hero with `inset: 0` re-runs `cover` and zooms the subject on every step change |
| The hero's growth animation at 768 | `transition: min-height` on `.hero` in that block + `fitTabletHero()` in `js/main.js` (§5e). The px value has to come from JS or there is nothing to transition |
| Desktop headline size, or the copy's width over the photo | `--hero-title-size` and `--hero-copy-max`, both on `.hero` in the `1024px+` block (§5g). They are a PAIR: the headline is one line, so `cap >= 8.276 x title-size` or it wraps. 42px at 1024 -> 72px at 1440, and the copy reaches into her hair at the narrow end on purpose |
| The hero form's button shape / hover | nothing overrides them any more — they are `btn btn--primary btn--lg`, the stats CTA's recipe minus its arrow (§5f). Back is `btn--outline btn--lg`, the accreditations-button recipe, also unoverridden |
| Where the desk background starts | `--content-bg-top` on `.content-band__bg` (§7) |
| Parallax strength | amplitude factor in `initParallax()` + CSS overshoot (§7) |
| Hero photo parallax (amount / cap) | `initHeroParallax()` in `js/main.js` (factor `0.08` + **8px** cap, driven off `window.scrollY`); overshoot = `scale(1.05)` on `.hero__bg-photo`, and because `object-position` pins the top edge that overshoot is the entire budget — sized for the **shortest** container it runs on (mobile's 568px photo band, not desktop — see §3a) |
| Which part of the photo stays in frame | `object-position` on `.hero__bg-image` (`center top`) and its `1024px+` override (`right top`) — §3a. Centring it crops her face off at several common widths |
| Hero RFI copy, fields, or step behaviour | `index.html` `.hero__rfi` (markup), `.rfi*` / `.rfi-field*` / `.rfi-radio*` blocks in `css/styles.css`, `initHeroRfi()` in `js/main.js`. The specialization options come from `SPECIALIZATIONS` at the top of `main.js` — shared with the program finder |
| Re-export the hero photo | crop the two boxes in §3a out of the source and save as WebP; the layer classes `.hero__bg-red` / `.hero__bg-people` are **gone** — there is one `.hero__bg-photo` now |
| Hero form height reservation (empty space under step 1) | `reservePanelHeight()` in `initHeroRfi()` — it pins both panels to the tallest state so the photo's crop can't shift between steps (§5c) |
| Program finder riding over the hero buttons | `measureFinderRoom()` in `initContentParallax()` — the drift is capped on the form's clearance, recomputed every frame (§7a) |
| The degree / area / specialisation gating | `syncChain()` in `initHeroRfi()` (§5d); the disabled look is `.rfi-field--disabled` in `css/styles.css` |
| A dropdown's caret position | `.rfi-field__caret` + `.rfi-field__box--select` in `css/styles.css` — it is an element centred in the box, **not** a background image on the select (§5b) |
| How wide the three learning-format options sit | `flex` on `.rfi__question--formats` in the `1024px+` block (§5a) — change that, not the 32px gap. It is deliberately **uncapped**; a `max-width` used to live here and wrapped the benefits legend |
| The dark scrim over the hero photo | `.hero__gradient` — base rule plus an override in each hero block (§5h). Vertical below 1024, horizontal above; 20% except desktop's 32%. Every stop is measured off a skin scan, so re-measure rather than nudging |
| The form panel's corner radius | `border-radius: 8px` on `.hero__rfi` (§5e). Shows on the floating tablet panel; inert at 1024+, where the form has no fill |
| Sticky header offsets | `.utility-bar` / `.main-nav` `top`/`z-index` (§5) |
| Program-finder dropdown options | `SPECIALIZATIONS` map in `js/main.js` |
| The program finder's headline size | `--text-program-finder` in `tokens.css` (42px, by request, at 768 and up). It is the only consumer of that token; the `≤768` rule clamps against it, so 768 resolves to 42 either way and scales down to 2rem below |
| "See all Capella programs" button alignment | `.stats-section__cta { align-self }` (right-aligned/flush with cards on desktop) |
| CTA background video (clip, encodes, tiers) | `.action-cta__video` markup in `index.html` + `initCtaVideos()` in `js/main.js` (§12) |
| A carousel slide's content or layout | the `<article data-slide="N">` in `index.html` + its `.carousel__content--{student,partner,faculty}` rules in both carousel `@media` blocks (§6) |

---

## 12. CTA background videos

The closing "what are you waiting for?" section plays a **single full-bleed
TV-spot clip**. Three looping clips on a gold backdrop
(`{leftLady,middleMan,rightLady}_loop.{webm,mp4}`) plus `cta-people.png` /
`cta-mobile.jpg` were the treatment for a while; those eight files have been
deleted and live only in git history now.

- **Files:** three encodes of the same spot, each as WebM + MP4 —
  `cta-tvspot.{webm,mp4}` (1440 master, ~5.3 MB), `cta-tvspot-sm.{webm,mp4}`
  (960-wide, ~2.2 MB) and `cta-tvspot-portrait.{webm,mp4}` (374 × 686, ~1.5 MB).
  The portrait pair is a **purpose-shot crop**, not the landscape master
  squeezed by `cover`. Each `<video>` lists **WebM first, MP4 second** — the
  browser picks WebM where supported and falls back to MP4 (older Safari).
- **Codec strings are exact**, read out of each file's `avcC` box
  (`avc1.640028` for the master, `avc1.64001F` for the smaller two). That's what
  lets a browser with no VP9 — Safari < 14.1, iOS < 17.4 — skip straight to the
  MP4 without spending a request on the WebM. Each encode is a different H.264
  level, so the tier swap rewrites `type` as well as `src`.
- **Autoplay-as-background:** `muted` + `playsinline` + `loop` (required for
  autoplay, incl. iOS). There is **no `autoplay` attribute** — see lazy-load.
- **Tier selection lives in `initCtaVideos()`, not `media` on `<source>`** —
  not every browser honours that attribute, and getting it wrong would serve the
  smallest file to desktops. Rewriting `src` in JS is safe because
  `preload="none"` means nothing has been requested yet. `≤768` → portrait
  (which also swaps in `cta-tvspot-portrait-poster.webp`; the landscape poster
  would letterbox), `≤1024` → `-sm`, otherwise the master.
- **Lazy-load (`initCtaVideos()`):** `preload="none"` plus an
  IntersectionObserver that calls `play()` only when the section is within
  ~200px of the viewport, and `pause()`s when it leaves.
- **Layout:** one full-bleed video; on phones (`≤768px`) the section takes the
  design's 375 × 687 ratio and the portrait encode fills it.
- **Width:** the section is capped at `--max-content` (1440) rather than
  full-bleed, so it centres on wider displays.
- **Placeholder:** `.action-cta__video { background: #6f7472 }` avoids a black
  flash before the poster paints.
- **Reduced motion:** `initCtaVideos()` bails before calling `play()`. The
  poster frame **is** the fallback — the video element still lays out and paints
  it — so nothing extra downloads.
- **If you swap the clip:** re-export all three tiers as WebM + MP4, re-read the
  `avcC` codec strings rather than copying the old ones, and `+faststart` the
  MP4s. There is no ffmpeg on this machine by default — see DEBUGGING.md.

## 13. Footer partner carousel

The ten Strategic Education brand logos sit in a real carousel, mirroring the
behaviour on capella.edu: **manual arrows only, no autoplay**, paging by a whole
view.

- **Slides per view** is driven entirely by `--per-view` on `.footer__partners`
  (6 desktop / 3 ≤1280 / 1 ≤768, matching the live site). `initFooterPartners()`
  reads that value back out of the computed style, so adding a breakpoint means
  touching CSS only.
- **Arrows disable rather than hide** at each end, so the viewport width never
  changes and the logos don't shift. With 10 logos and 6 desktop slots there are
  two pages, so "next" is live on load and "prev" only enables once you page.
  If the brand count ever drops to `--per-view` or below, both arrows sit
  disabled — that's correct, not a broken carousel.
- `aria-hidden` and `tabindex` track which slides are in view, so off-screen
  logos aren't announced or tab-focusable.
- **Logo provenance is not what the filenames suggest** — see §3d. Devmountain
  and Sophia are PNGs sliced from the old strip because no correct SVG exists
  for either. Each was checked visually before being wired up.
- **All 10 live brands are present**, in the same order as capella.edu, using the
  official exports pulled from `capella.edu/content/dam/...` into
  `public/assets/partners/`. The three oversized PNGs (Sophia, JWMI,
  Degrees@Work — up to 7185px wide) were trimmed, resized to 176px tall and
  converted to greyscale+alpha; they are pure white artwork, so dropping colour
  is lossless. Adding an 11th brand is one `<li>` — no JS or CSS change.
- The older `footer-partner-*.svg` files are now fully superseded and unused.
