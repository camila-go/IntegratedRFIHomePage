# Accessibility audit — hero + integrated RFI form

**Standard:** WCAG 2.1 Level AA
**Audited:** 2026-09-22
**Scope:** the hero section and the two-step Request Program Information form
(`.hero`, `.hero__rfi`), plus the program-finder headline directly below it
**Widths tested:** 320 · 375 · 768 · 1000 · 1024 · 1200 · 1281 · 1440 · 1920
**Code audited:** commit `e5d0128`; fixes landed in `e5d0128` and `f79a28d`

---

## Summary

| | Count |
| --- | --- |
| 🔴 Critical | 0 |
| 🟡 Major | 1 — **fixed** |
| 🟢 Minor | 2 — **fixed** |
| Open by choice | 3 (documented below, none a conformance failure) |

All three findings were fixed in the same pass. The audit also caught one
contrast failure that a conventional check would have missed — see
[Method](#method-and-how-to-reproduce), which is the part of this document most
worth reading if you are re-testing later.

---

## Findings

### 🟡 1 — Fixed field height broke at 200% text zoom

**Criterion:** 1.4.4 Resize text · **Severity:** Major · **Status:** ✅ fixed

`.rfi-field__box` was `height: 48px` — a fixed height around text. Under
text-only zoom the three step-1 dropdowns stack a 12px hint over a 16px value in
one grid cell; at 2× that content measured **~100px against 46px of content
box** and overflowed it.

A second, subtler half of the same bug: the value's `padding-top` was `21px`,
and that padding is the only thing holding it clear of the hint above it. In px
it stayed 21 while the hint's line grew, so at 2× **the two text bands
overlapped**.

**Fix**
- `min-height: 48px` instead of `height` — renders identically at 100% (the
  content is 44px, so the min-height is what sets the 48).
- The select's `padding-top` is now `1.3125em` — the same 21px at 100%, and it
  scales with the text.

**Verified after:** at 2× the box grows 48 → 89px, `scrollHeight === clientHeight`
(nothing clipped), and the hint/value have 8px of clearance instead of
overlapping. Still exactly 48px at 100%.

> Worth knowing: Chrome only offers full-page zoom, which scales the box too and
> hides this entirely. It surfaces with Firefox's text-only zoom or a minimum
> font-size preference.

---

### 🟢 2 — Gated selects were unexplained non-visually

**Criterion:** 3.3.2 Labels or instructions · **Severity:** Minor · **Status:** ✅ fixed

Step 1 is a gated chain: *Area of study* is `disabled` until a degree is chosen,
and *Program/specialization* until an area is. `disabled` removes a control from
the accessibility tree and the tab order, so the measured tab order for step 1
was:

```
STEP 1 → STEP 2 → Degree → Learn program details
```

A screen-reader user perceived step 1 as a **one-field form**, with no
indication that two more questions follow. Sighted users get that information
from the dimmed fields.

**Fix** — a `.visually-hidden` paragraph in `.rfi__prompt` stating the order:
“Three questions, answered in order: area of study becomes available once you
choose a degree, and program or specialization once you choose an area of
study.”

It cannot live on the selects themselves: a `disabled` control's
`aria-describedby` is never read out.

---

### 🟢 3 — Error messages were 10px at two breakpoints

**Criterion:** readability (no AA minimum font size exists) · **Severity:** Minor · **Status:** ✅ fixed

The step-2 field row is the tightest place in the form. At 768–1199 the messages
had been dropped to **10px** so they would fit on one line, because the original
wording needed 165px of text width against ~138–142px available.

**Fix** — the copy was shortened instead, and the 10px tier removed entirely:

| before | after | width @12px |
| --- | --- | --- |
| Please enter your first name | **First name is required** | 126.7px |
| Please enter your last name | **Last name is required** | 125.3px |
| Phone number is required | **Phone is required** | 102.2px |
| Email is required | *unchanged* | 96.7px |
| Zip code required | **Zip/Postal is required** | 124.9px |

All five now render at **12px on one line** at every width. Note that "Phone
number is required" also had to be shortened — at 150.6px it became the longest
string once the name fields were fixed, and would have kept the 10px tier alive
on its own.

> ⚠️ **The copy is now the constraint, not the font size.** The binding case is
> 1024, where a 171px field leaves 138px and the longest message is 137.7px —
> **0.3px of slack**. Re-measure before rewording any message.

*Updated 2026-09-22: the ZIP field became “Zip/Postal”, so its message is now
“Zip/Postal is required” at 124.9px — still inside 138px.*

---

## Colour contrast

### Text (1.4.3)

Measured against the first opaque ancestor background.

| Element | Size / weight | Ratio | Required | Pass |
| --- | --- | --- | --- | --- |
| Panel title “Request program information” | 22px 700 | 15.81 | 3 | ✅ |
| Stepper, current step | 16px 700 | 15.81 | 4.5 | ✅ |
| Stepper, other step | 16px 400 | 15.81 | 4.5 | ✅ |
| Prompt “What are you interested in?” | 16px 700 | 15.81 | 4.5 | ✅ |
| “*All fields are required” | 12px | 15.81 | 4.5 | ✅ |
| Field label inside the box | 12px, `#696969` on white | 5.49 | 4.5 | ✅ |
| Field value | 16px 700, `#212322` on white | 15.81 | 4.5 | ✅ |
| Question legend | 16px | 15.81 | 4.5 | ✅ |
| Radio label | 16px | 15.81 | 4.5 | ✅ |
| Consent paragraph and its link | 12px | 15.81 | 4.5 | ✅ |
| Error message | 12px, `#9c0c0c` on `#fff5f5` | 7.94 | 4.5 | ✅ |
| Submit button label | 20px 700, white on `#c10016` | 6.40 | 3 | ✅ |
| Back button label | 20px 700 | 15.81 | 3 | ✅ |
| Program-finder headline | 42px | 21.00 | 3 | ✅ |

### Hero copy over the photograph

The copy sits on a photograph, so this was measured by **sampling the decoded
image** behind each text box and compositing the scrim over it — and by the
**worst 8×12px patch**, not the average. See [Method](#method-and-how-to-reproduce).

| Width | Element | Worst patch | Required | Pass |
| --- | --- | --- | --- | --- |
| 375 | headline | 8.25 | 3 | ✅ |
| 375 | subtitle | 5.26 | 4.5 | ✅ |
| 768 | headline | 7.74 | 3 | ✅ |
| 768 | subtitle | 8.32 | 4.5 | ✅ |
| 1024 | copy (average) | 10.07 | 4.5 | ✅ |
| 1440 | headline | 9.42 | 3 | ✅ |
| 1440 | subtitle | 9.22 | 4.5 | ✅ |

### Non-text (1.4.11)

| Element | Ratio | Required | Pass |
| --- | --- | --- | --- |
| Field fill (white) vs panel | 15.81 | 3 | ✅ |
| Error border vs panel | 3.04 | 3 | ✅ |
| Error message bar vs panel | 14.78 | 3 | ✅ |
| Focus ring `#696969` on the white field | 5.49 | 3 | ✅ |
| Stepper rule, current step | 15.81 | 3 | ✅ |
| Stepper rule, other step (40% white) | 3.72 | 3 | ✅ |
| Radio mark fill vs panel | 15.81 | 3 | ✅ |
| Field border `#adadad` vs its own white fill | 2.24 | 3 | ⚠️ see below |

---

## Keyboard navigation (2.1.1, 2.4.3, 2.4.7)

| | Result |
| --- | --- |
| Tab order, step 1 | `STEP 1 → STEP 2 → Degree → Learn program details` ✅ logical |
| Tab order, step 2 | `STEP 1 → STEP 2 → first → last → email → phone → zip → military radios → privacy link → Back → submit` ✅ |
| Hidden panel | `hidden` keeps step 2's controls out of the tab order until reached ✅ |
| Stepper tabs | real `<button>`s — Enter/Space native; forward runs the same validation gate as the CTA ✅ |
| Focus visible — fields | `.rfi-field__box:focus-within` → 2px ring, 1px offset ✅ |
| Focus visible — radios | `.rfi-radio__input:focus-visible + .rfi-radio__mark` → white ring ✅ correct pattern for a visually-hidden input |
| Focus visible — tabs, buttons | white ring at 2px offset ✅ |
| Predictable on focus (3.2.1) | the gated chain fires on `change`, never on focus ✅ |

## Screen reader semantics (1.1.1, 1.3.1, 3.3.1, 4.1.2, 4.1.3)

| Element | State | Result |
| --- | --- | --- |
| All 8 fields | `label[for]`, `required`, `aria-describedby` → its error bar (every target id exists) | ✅ |
| Text inputs | `autocomplete`: given-name, family-name, email, tel, postal-code | ✅ also satisfies 1.3.5 Identify Input Purpose |
| Radio groups | `<fieldset>` + `<legend>` | ✅ |
| Error bars | `role="alert"`; `aria-invalid="true"` set on the field when invalid | ✅ |
| Stepper | `<ol>` with `aria-current="step"` on the current item | ✅ |
| Hero photograph | `alt=""` plus `aria-hidden="true"` on its container | ✅ decorative |
| Headings | one `<h1>`, then `h2 → h3` | ✅ |

## Zoom and reflow

| | Result |
| --- | --- |
| 1.4.4 Resize text — 200% | ✅ after finding 1. Select box grows 48 → 89px, nothing clipped, 8px clearance between hint and value |
| 1.4.10 Reflow — 320px | ✅ `scrollWidth` is exactly 320, no horizontal scroll, no button wrapping. The only element wider than the viewport is the background photo's 1.05 parallax overshoot, which is clipped by `overflow: hidden` |

---

## Open by choice

None of these is a conformance failure; each is a judgement call worth a second
opinion.

1. **The `#adadad` field border is 2.24:1 against its own white fill.** Not a
   1.4.11 failure: the criterion asks for 3:1 on the visual information
   *required to identify* the component, and what identifies these fields is the
   white fill against the dark panel at 15.81:1. Treat the border as decorative.
   Darken toward `#767676` if it should pass standalone.
2. **Touch targets are AA-sized, not AAA.** 2.5.5 (44×44) is a **AAA**
   criterion. The 32px stepper tabs and 24px radio labels clear the AA
   criterion — 2.5.8, 24×24, from WCAG 2.2 — and the inline privacy link is
   exempt under that criterion's inline exception. The stepper tabs are 32px
   rather than 44 because the panel's vertical budget at 768 is fully spent; see
   HANDOFF §5e.
3. **One message per field, whatever the failure.** Each field has a single
   static string, so a malformed value reports the same text as an empty one —
   `type="email"` with “not-an-email” still says “Email is required”, which does
   not describe what went wrong (3.3.1). The ZIP field was taken out of this
   category on 2026-09-22 by removing its `[0-9]{5}` pattern: with `required` as
   its only rule, “Zip/Postal is required” is always accurate. Email and phone
   still have the mismatch. Fixing them properly means a dynamic message — one
   string for empty, another for malformed.
4. **Error announcement should get one manual pass.** `role="alert"` on a bar
   that goes from `hidden` to visible is announced by current browsers, but a
   persistent live region is the more robust pattern. This is the one item below
   that a human should verify with real AT.

---

## Method, and how to reproduce

Everything above was measured in the browser against the running build rather
than read off the design, because two of the findings are invisible to a static
review.

- **Contrast on solid backgrounds** — computed from `getComputedStyle`, walking
  up for the first opaque ancestor background, using the WCAG relative-luminance
  and contrast-ratio formulas.
- **Contrast over the photograph** — the hero image is decoded into a `<canvas>`,
  the text box is mapped back to source pixels through the `object-fit: cover`
  transform (including the mirror at 768–1023 and the 1.05 parallax scale), and
  the scrim's alpha at that position is composited over the sampled colour.
- **Worst-patch sampling, not averages.** ⚠️ **This is the important one.** At
  768 the headline's line *averaged* 10.45:1 and looked fine — but a patch scan
  found `rgb(227,208,199)` at **2.11:1** where it crossed her white collar,
  while the darkest patch in the same line read 20.66:1. Her dark hair either
  side of the collar pulls the mean up and hides the failure. A row-by-row scan
  of the whole copy column then identified the safe band:

  | rows | worst patch | content |
  | --- | --- | --- |
  | 200–392 | 2.0–4.8 ❌ | face, white collar, lit shoulder |
  | 416–632 | 7.3–12.5 ✅ | dark sweater |
  | 656–800 | 6.9–7.5 ✅ | sweater into skirt |

  The copy is now anchored at the top of that band. **Any future check of text
  over this photograph must sample patches, not averages.**
- **Text extent** — `Range.getClientRects()` for glyph-accurate line widths and
  line counts, rather than element boxes (a box can be much wider than its ink).
- **Tab order** — collected in DOM order from the focusable set, filtered by
  `disabled` and `offsetParent`.
- **Focus indicators** — verified by reading the matched CSS rules, since
  programmatic `focus()` does not reliably match `:focus-visible`.

### Limits of this audit

- **No real assistive technology was used.** Semantics were verified
  structurally (roles, names, relationships, tab order), which is not the same
  as hearing them. One VoiceOver or NVDA pass is still recommended, especially
  for the step change and error announcement.
- Automated and programmatic checking catches a minority of real barriers.
  Nothing here covers cognitive load, the clarity of the error wording, or
  whether the two-step flow is understandable under stress.
- Contrast over a photograph is sampled, not exhaustive. The patch grid is
  8×12px at 10–14px intervals; a small bright detail between samples could still
  be missed.

### Retest checklist

Re-run this audit if any of the following change, because each one has a
measured margin that is thinner than it looks:

- [ ] Any error message wording — 0.3px of slack at 1024 (finding 3).
- [ ] Any field's validation rules — if a format check is added back, its
      message has to distinguish empty from malformed (see open item 3).
- [ ] The hero photograph or either crop — every contrast figure over the photo,
      and the safe-band rows, are specific to this image.
- [ ] The hero copy's position or size at any breakpoint — it was moved off a
      2.11:1 patch to get here.
- [ ] The scrim's alpha or stops — `.hero__gradient`, HANDOFF §5h.
- [ ] Field height, padding, or the stacked hint/value geometry — finding 1.
