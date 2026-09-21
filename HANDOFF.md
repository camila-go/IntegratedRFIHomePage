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

- **`hero-rfi-desktop.webp`** (2560×1217, 153 KB) — used at 769px+.
- **`hero-rfi-mobile.webp`** (750×1136, 78 KB) — used at ≤768px, via a
  `<picture>` source. `fetchpriority="high"` on both (the hero is the LCP
  element), each with its own `media`-gated `<link rel=preload>` so a phone
  never pulls the desktop frame.
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
| mobile (≤768) | `center top` | the crop goes width-bound as the width grows past 375 against the fixed 568px band, eating the top — where her head is |
| desktop (≥769) | `right top` | the crop goes height-bound below ~1270px and on short windows, eating the sides — where her head is; `top` also covers the vertical crop on ultrawide |

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
| `min-width: 769px` | **The hero's own breakpoint** — deliberately 769, not 768, so it lines up exactly with the `(max-width: 768px)` query on the `<picture>` mobile source. Switches the hero from the mobile stack (photo band + solid dark form panel) to the desktop overlay (full-bleed photo, transparent form, row-layout stepper and fields) |
| `min-width: 769px and max-width: 1199px` | **Tablet hero**: capped at `--hero-height-tablet-max` (760px), and step 2's five text inputs wrap to a 2-up grid — in one row they'd be ~150px each |
| `min-width: 1024px` | **Wide carousel layout** (`1440 × 600` card; the faculty portrait and the phone overflow above the card top) |
| `min-width: 1200px` | Desktop refinements: 4-across program-finder chips, content-band bg crop, etc. (the hero's sizing is owned by its own 769px+ block, not this one) |
| `max-width: 1280px` / `min-width: 1920px` | `--page-gutter` adjustments only (in `tokens.css`) |

⚠️ **The 1023 / 1024 boundary is load-bearing for the carousel.** The phone and
wide carousel layouts are mutually exclusive and split exactly here. If you
shift this boundary, audit both carousel layouts — they use different
positioning systems (see §6).

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

### Hero height: capped on desktop, content-driven on mobile

The two breakpoints size the hero on completely different principles, and the
CSS here is **mobile-first** (the reverse of the rest of the file) because mobile
is the case with structure:

- **`<769px`** — no `min-height` and no `overflow: hidden` at all. The photo is
  a fixed `--hero-photo-mobile` (568px) band pinned to the top of the hero, and
  the RFI panel sits below it in normal flow with its own solid
  `--color-uni-black` fill covering whatever of the band it overlaps. So the hero
  is exactly as tall as its content, and it grows when step 2 (which is taller)
  is showing. Nothing to tune.
- **`769px+`** — `min-height: min(var(--hero-height), calc(100svh - var(--hero-fold-reserve)))`,
  i.e. capped at the Figma height (912px) and shrinking on shorter viewports.
  `--hero-fold-reserve` is just the header (128px = 40 utility + 88 nav); the
  tablet block swaps the cap for `--hero-height-tablet-max` (760px). Uses `svh`
  so mobile browser chrome doesn't break it.

⚠️ **The reserve no longer accounts for the program finder.** It used to: the
hero was sized to keep the finder above the fold. The RFI form is now *in* the
hero, so the form is the thing that has to be reachable, and capping the hero at
the design height is what achieves that. Don't re-add finder-sized reserve
values — that shrinks the hero and squeezes the form.

The hero content is **top-anchored and left-aligned** in the same
`min(--max-content, 100% - 2*--page-gutter)` measure `.page-container` gives the
rest of the page, which is what puts it at Figma's `x=240` on a 1920 frame.

The photo takes a **different `object-position` per breakpoint** (`center top`
on mobile, `right top` on desktop) because the subject is off-centre in both
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

⚠️ **`.rfi__question--formats` needs its `max-width: min(770px, 65.8%)`.** The
32px gap between the three learning-format radios is exactly per Figma — but the
radios are Fill columns, so how loose the row *reads* is set by the column width,
not the gap. Two things were inflating it: the group stretched across the whole
row whenever the RN question was hidden (the common case — 339px columns, so the
options read as ~190px apart), and the hero's form is 1440 wide against the
component's 1170, which gave 295px columns at 1920 instead of 235. The cap is the
component's absolute 770px with its 770/1170 share as the fallback below that, so
the row now measures **770 / 235 / 32 at 1920 — identical to Figma** — in both
states. If the row ever looks too airy again, measure the column width before
touching the gap.

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
| inactive | 1px `#adadad`, 2px radius, `--field-fill` (uni-black 20%) | same |
| focused | 2px `#0f7bd9` **ring** (outline, not a border swap — nothing reflows) | same |
| success | — (no such state) | `check` icon `#b0e8c1`, neutral border kept |
| error | border → `#ffa8a8`, bottom corners squared, message bar below | same |
| disabled | content (hint + value + caret) at 0.55, **border kept** | — (inputs are never gated) |
| autofilled | — (not an autofill target) | UA background clipped away, value forced back to white |

⚠️ **Autofill needs handling in both CSS and JS, and neither is obvious.**

*Visually:* Chrome and Safari paint an autofilled field with a pale UA
background and near-black text using a rule authors cannot override —
`background-color` and `color` are both ignored. On the red hero that turned the
name fields into pale blue boxes. The usual workaround, a 1000px opaque inset
`box-shadow`, is **wrong here**: these fields are deliberately transparent so the
photo shows through, and an opaque shadow paints a solid block over it. Instead
`-webkit-background-clip: text` clips the UA background to the glyphs — where
`-webkit-text-fill-color: white` then covers it — so the field stays
transparent. The `:autofill` and `:-webkit-autofill` rules are kept **separate**
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

⚠️ **`--field-fill` (uni-black 20%) is load-bearing, not decoration.** It is the
Fill Figma puts on the field, and it is what lifts the hairline border and the
12px hint off the brighter parts of the hero photo. Measured worst-case, before
→ after adding it:

| | before | after | needs |
| --- | --- | --- | --- |
| border, degree / area | 2.50 / 2.55 | **3.22 / 3.30** | 3 ✅ |
| border, specialisation | 2.01 | 2.66 | 3 ❌ |
| hint 12px `#adadad` | 2.80 | 3.57 | 4.5 ❌ |
| value band (white) | 5.65 | 7.30 | 4.5 ✅ |
| focus ring `#0f7bd9` | 1.04–1.32 | 1.38–1.71 | 3 ❌ |

So it fixes two of the three borders and helps everything, but **does not close
the accessibility gaps on its own**. The specialisation field sits over her
bright sweater and still falls short; the hint needs a lighter colour than
`#adadad`, not a darker backdrop; and the focus ring barely moves because blue
on red is near-isoluminant — darkening the backdrop lowers both sides together.
Don't treat this fill as having resolved those.

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

⚠️ **The selected radio dot is WHITE here, not the component's uni-red.**
`.rfi-radio__mark` is transparent, so the dot sits straight on the hero photo —
and the dot is the only thing that distinguishes selected from unselected. Red
on red measured **1.42:1** behind the RN answer and **2.55:1** behind the
learning-format row, against the 3:1 that a graphical state indicator needs
(1.4.11); it read as barely filled. White measures **9.09:1** and **16.33:1**
and matches the ring around it. Don't "correct" it back to red without also
giving the mark an opaque fill.

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

⚠️ **Desktop only** (`min-width: 769px`). Below that the photo is a fixed 568px
band whose crop already doesn't depend on the form at all, so reserving buys
nothing there and costs a lot — the tallest state is ~911px, which would leave
step 1 with hundreds of px of empty dark panel.

⚠️ It runs on **init and resize only**, never from a `ResizeObserver` — it
mutates the very heights such an observer would be watching.

### 5d. The three step-1 dropdowns are a gated chain

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
| Hero height | desktop: `min-height: min(var(--hero-height), calc(100svh - var(--hero-fold-reserve)))` in the `769px+` block; mobile: **no** `min-height` at all, the hero is content-tall (§5). The reserve is just the header now — it no longer reserves room for the program finder |
| Where the desk background starts | `--content-bg-top` on `.content-band__bg` (§7) |
| Parallax strength | amplitude factor in `initParallax()` + CSS overshoot (§7) |
| Hero photo parallax (amount / cap) | `initHeroParallax()` in `js/main.js` (factor `0.08` + **8px** cap, driven off `window.scrollY`); overshoot = `scale(1.05)` on `.hero__bg-photo`, and because `object-position` pins the top edge that overshoot is the entire budget — sized for the **shortest** container it runs on (mobile's 568px photo band, not desktop — see §3a) |
| Which part of the photo stays in frame | `object-position` on `.hero__bg-image` (`center top`) and its `769px+` override (`right top`) — §3a. Centring it crops her face off at several common widths |
| Hero RFI copy, fields, or step behaviour | `index.html` `.hero__rfi` (markup), `.rfi*` / `.rfi-field*` / `.rfi-radio*` blocks in `css/styles.css`, `initHeroRfi()` in `js/main.js`. The specialization options come from `SPECIALIZATIONS` at the top of `main.js` — shared with the program finder |
| Re-export the hero photo | crop the two boxes in §3a out of the source and save as WebP; the layer classes `.hero__bg-red` / `.hero__bg-people` are **gone** — there is one `.hero__bg-photo` now |
| Hero form height reservation (empty space under step 1) | `reservePanelHeight()` in `initHeroRfi()` — it pins both panels to the tallest state so the photo's crop can't shift between steps (§5c) |
| Program finder riding over the hero buttons | `measureFinderRoom()` in `initContentParallax()` — the drift is capped on the form's clearance, recomputed every frame (§7a) |
| The degree / area / specialisation gating | `syncChain()` in `initHeroRfi()` (§5d); the disabled look is `.rfi-field--disabled` in `css/styles.css` |
| A dropdown's caret position | `.rfi-field__caret` + `.rfi-field__box--select` in `css/styles.css` — it is an element centred in the box, **not** a background image on the select (§5b) |
| How wide the three learning-format options sit | `max-width` on `.rfi__question--formats` in the `769px+` block (§5a) — change that, not the 32px gap |
| Sticky header offsets | `.utility-bar` / `.main-nav` `top`/`z-index` (§5) |
| Program-finder dropdown options | `SPECIALIZATIONS` map in `js/main.js` |
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
