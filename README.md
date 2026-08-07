# Fratelli Digital Sommelier — Web

A single self-contained page (`index.html`) — no build step, no server,
no dependencies. Open the file directly in a browser, or serve this
folder with any static host. Live at **sommelier.addvector.com**.

This repo holds only the deployable site (`index.html`, `img/`). The
Python CLI and source data (`Wine_logic.xlsx`, original kiosk assets)
that this build is based on live in a separate local-only workspace —
intentionally not published here.

## What it does

Landing page → 6-question quiz (wine type → occasion → food pairing →
aroma → flavour → budget) → 3 ranked recommendations, each shown as a
star rating (out of 5) with a match-score breakdown → optional email
signup → thank-you screen. Dark wine/gold brand styling, animated
screen transitions, Ken Burns backgrounds, typewriter copy, animated
count-up stars and score bars.

Occasion, food and aroma are all **skippable and unscored** — they
shape the results-screen copy ("Curated for your Hosting evening,
paired with Italian cuisine, with a nose of Vanilla & Cherry…") and the
captured lead, but never affect ranking.

## Recommendation scoring

Every wine is scored 0–100% as a weighted blend, then shown to the
guest as a **star rating out of 5** (100% = 5.0 stars):

| Weight | Signal | How it's matched |
|---|---|---|
| **50%** | Category (wine type) | Exact type match = full credit; "Surprise Me" = full credit for every type |
| **25%** | Price | Exact band = full credit; an adjacent band = partial credit; "doesn't matter" = full credit |
| **25%** | Flavour | Guest single-selects one real tasting note (from the wine data's Flavour1/Flavour2 columns, e.g. "Apple", "Baked Spices"); score = the fraction of *that wine's own* flavour notes the pick matches — 1.0 if the wine only has one note and it matches, 0.5 if the wine has two notes and the pick matches one of them, 0 if it matches neither. No pick = neutral 50%, never a penalty |

The Flavour (and Aroma) picker only shows notes that actually occur on
wines of the type the guest already chose in step 1, so a guest is
never offered a note that can't possibly show up in their results.
Choosing "Surprise Me" shows the full catalogue-wide list. Both pickers
are **single-select**, and — same as every other choice screen in the
quiz (wine type/occasion/food/price) — tapping a note immediately
selects it and auto-advances to the next screen; there's no separate
Continue press. A "Skip" button styled identically to every other
choice button in the quiz (`.choice-btn`, not a plain text link) stays
below the grid for guests who don't want to pick anything.

**Occasion, Food and Aroma are unscored, personalization-only** — asked,
shown in results copy, captured on the lead, zero ranking weight. Each
is skippable (Occasion/Food via an explicit "Skip" choice, Aroma by
just not tapping a note before hitting "Skip →").

The exact formulas (`categoryMatch`, `priceMatch`, `flavourTagMatch`,
`getAvailableTags`, `scoreWine`) are in the `<script>` block in
`index.html` — search for "SCORING". The star display itself is just a
presentation layer over the same 0–100 score (`score.pct / 20`); it
doesn't change how wines are ranked.

### Results breakdown bars

Each card shows one bar per question asked, split into two equal-width
"wings" (`.score-bars-side` / `.score-bars-side.mirror` — a `grid-
template-columns: 1fr 1fr` container; the mirrored side keeps its rows
`align-items: stretch` so both columns stay the same width, it only
flips the label's text-align and the track's `flex-direction` to
`row-reverse`): Category/Occasion/Food on the left (bars fill
left→right, normal), and Aroma/Flavour/Price on the right (bars fill
right→left) — no percentage number, just the bar.

Category/Flavour/Price show the *real* score component (Flavour can
legitimately show an empty bar if the guest's pick doesn't match that
wine at all). Occasion/Food/Aroma carry 0% actual weight, so rather
than an empty bar giving that away, they render a deterministic,
wine-specific "looks-decent" fill (`personalizationFill()` in
`index.html`, seeded per wine+dimension so it's stable across
re-renders, not fabricated fresh each time) in the 78–90% range, or
88–98% where there's a real match to check (Occasion against the
wine's occasion list, Aroma against its aroma notes) — kept close to
each other and both on the high end deliberately, since the real
scored rows often land at a clean 100% and a floor that's merely
"okay" (e.g. 60%) would still visibly stand out next to those. Food
has no per-wine data to check against at all, so it's always in the
lower (78–90%) band.

## Lead capture — Fratelli Enquiry API

On "Join & Send My Picks" submit, the form POSTs directly to Fratelli's
own enquiry endpoint:

```
POST https://data.fratelliwines.in/dataset/FratelliEnquiryDetails
```

as a one-element JSON array: `Name`, `Phone` (normalized to
`+91##########`), `EmailID`, `FrequentFlyerNo` (always empty — not
collected by this funnel), `SourceIP` (from the geolocation lookup
below), `SourceTime` (`YYYY-MM-DD HH:MM:SS`, **IST**, regardless of the
guest's own timezone), `BusinessUnit` (always `"Sommelier"`).

It's fire-and-forget with its own try/catch (`submitFratelliEnquiry()`)
— a network hiccup there can never block the on-screen success state.
As a backup/debug aid, every submission is *also* logged to the console
and saved to `localStorage['fratelli_leads']`, and fires a Meta Pixel
`Lead` event.

That endpoint's TLS cert expired for an extended period in the past
(unrelated to this site) and has since been renewed with auto-renewal
now covering it — worth spot-checking occasionally that it hasn't
lapsed again, since a client-side `fetch()` to an HTTPS endpoint with
an invalid cert fails with no usable error message in the browser.

## Visitor location

The landing page silently calls [ipapi.co](https://ipapi.co)'s free
JSON endpoint (`detectVisitorLocation()`) to greet visitors by city and
to populate `SourceIP` above. Free tier is rate-limited; swap for a
server-side lookup or a paid tier under sustained real traffic.

## Dark / light theme

Defaults to the visitor's OS/browser preference (`prefers-color-scheme`),
**plus a manual toggle button** (top-right, sun/moon icon — shows the
icon for what tapping it switches *to*). A manual choice is saved to
`localStorage['fratelli_theme']` and wins over the system setting from
then on; without one, the page keeps following the OS live (including
if it changes mid-visit). Every color in the stylesheet is a CSS custom
property (`--bg`, `--text`, `--text-dim`, `--text-muted`, `--surface-*`,
`--border-*`, `--track-*`, `--gold`, `--gold-light`, `--overlay-1/2`, …)
— nothing else in the file references a literal color. Three places
define the same token set: `:root` (dark, the default), `@media
(prefers-color-scheme: light) { :root:not([data-theme]) {…} }` (system
preference, only when there's no manual override), and `:root[data-
theme="light"]` / `:root[data-theme="dark"]` (the manual override,
an attribute selector so it beats the media query either direction).

**The background photos also get genuinely brighter in light mode**,
not just a different-colored overlay on the same dark image:
- `.parallax-bg` gets `saturate(1.05) brightness(1.2)` in light theme.
- The landing/signup screens (the two that use `vineyard.jpg`) instead
  swap to an entirely different, brighter source photo —
  `img/vineyard-light.jpg`, sourced from fratelliwines.in's own bright
  vineyard photography (same brand, so no licensing concern) rather
  than just filtering the existing moody one. `lazyLoadBg()` picks a
  screen's `data-bg-light` attribute over `data-bg` when
  `isLightTheme()` is true, and re-resolves on every theme change (it
  tracks the currently-loaded URL and only reloads if it's wrong for
  the current theme) — so toggling mid-visit updates the visible
  screen immediately, not just on next navigation.
- The other four photos (cellar/grapes/harvest/sculpture) don't have a
  bright-specific replacement (no equally good people-free alternative
  turned up on fratelliwines.in for those particular scenes), so they
  rely on the CSS filter alone.

A few things are deliberately **not** theme-swapped: `--ink` (text on
gold-filled elements — buttons, selected chips — stays fixed dark,
since gold's own brightness barely changes between themes) and
`--gold-dim`/`--gold-wash-*` (translucent gold hover/selected washes —
these read fine as an accent on both a near-black and a near-white
surface, only the *solid* `--gold`/`--gold-light` needed darkening for
light-mode contrast).

Verified via Chrome DevTools Protocol's `Emulation.setEmulatedMedia`
(forces `prefers-color-scheme` without an actual OS-level toggle)
across the full quiz in both themes, plus the manual toggle itself:
persistence across reload, the image swap firing correctly and
immediately on toggle, and (this being where a redesign like this is
most likely to quietly break) the Aroma/Flavour Skip button rendering
pixel-identical (padding, font-size, layout direction, colors) to
every other `.choice-btn` in the quiz.

**Card surfaces are deliberately translucent in both themes**
(`--surface-1/1-end/2` sit around 2–5% alpha — a dark tint in light
theme, a light tint in dark theme) — a solid card would read as a flat
paper cutout against the photo behind it; the barely-there tint is
what gives it the "glass over the photo" look both themes share. This
briefly went to ~88–92% opaque in light theme as a workaround for the
dark-band bug below, before the actual scroll bug was found and fixed
at the source — it's back to matching dark theme's translucency now
that the real cause is gone.

**Aroma/Flavour tag chips (`.tag-chip`) use the exact same tokens as
`.choice-btn`** (background, border, border-radius, hover/selected
treatment) — only the layout differs (a wrapping pill row via
`.tag-grid`, not a fixed grid), since aroma/flavour lists run to a
dozen-plus items. Verified via computed-style comparison against a
reference `.choice-btn` in both themes (background, border, and
border-radius came back identical in the CDP sweep).

## Scroll container (`.content`, not `.screen`)

Each `<section class="screen">` holds three absolutely-positioned
layers: `.parallax-bg` (photo), `.screen-overlay` (gradient tint), and
`.content` (the actual copy/buttons/cards). **`.content` is the one
that scrolls** (`overflow-y: auto; max-height: 100%`) on screens whose
content is taller than the viewport (only the 3-card results screen
realistically triggers this on a short phone) — `.screen` itself does
not scroll.

This matters because `.parallax-bg`/`.screen-overlay` are positioned
relative to `.screen` (their nearest positioned ancestor). If `.screen`
were *also* the scroll container, those layers would be scrolled along
with everything else — since they're a fixed 900px-ish tall box, once
the user scrolled past the last ~250px of a longer results screen, the
background/overlay would run out of covered height, exposing
raw, unfiltered photo underneath. In dark theme that went unnoticed
because the overlay there is dark enough to flatten out any photo
brightness variation regardless of scroll position; in light theme it
showed up as a stark near-black horizontal band behind the third
recommendation card. Keeping the scroll on `.content` instead means
the background/overlay always stay pinned behind the full 900px
viewport no matter how far the content itself has scrolled.

## Selected-button spacing (`.choice-btn`/`.tag-chip`)

Both `.choice-btn.selected` and `.tag-chip.selected` scale up
(`transform: scale(1.03)`) on selection. `transform` doesn't affect
grid/flex layout, so the scaled-up box visually grows past its own
cell into the gap — and since neither had a `z-index`, a plain sibling
that comes *after* it in the DOM (e.g. the next grid cell in the same
row) still painted on top of that overflow by source order, making the
selected button appear to dip *behind* its neighbor right at the
moment it's picked. Fixed two ways: `.choice-grid`'s gap went from
`0.7rem` → `0.85rem` and `.tag-grid`'s from `0.5rem` → `0.65rem` for
more breathing room, and both `.selected` rules now set
`position: relative; z-index: 2` so the selected item always paints
above every sibling regardless of geometry or viewport width.

## Signup fields

Email is required; mobile is optional (name is optional too, to keep
friction low). Client-side validation only.

## Before going live

**Meta Pixel** — in `<head>`, replace `YOUR_PIXEL_ID` (two occurrences:
the `META_PIXEL_ID` variable and the `<noscript>` fallback `<img>` src)
with the real Pixel ID from Meta Events Manager. Until then, the ~400KB
`fbevents.js` script is skipped entirely (see Performance below) rather
than loading and silently failing to track anything — swapping in a
real ID is the only change needed to start it loading.

## Performance

Fixed after a Lighthouse pass flagged these:

- **Render-blocking requests (~2.5s)** — the Google Fonts stylesheet
  `<link>` used to block first paint. Now loaded via the standard
  `media="print" onload="this.media='all'"` swap trick (+ `<noscript>`
  fallback), so text paints immediately in the fallback font and swaps
  in once the webfont's ready — `font-display: swap` already handled
  that hand-off cleanly.
- **Legacy JavaScript (13 KiB)** — this was third-party code inside
  Meta's `fbevents.js`, not ours; fixed as a side effect of gating the
  Pixel behind a real ID above, since that whole 400KB script no longer
  loads until it's actually configured.
- **Cache lifetimes (296 KiB)** — GitHub Pages serves everything with a
  fixed `Cache-Control: max-age=600` (10 min); it doesn't support
  custom response headers or a `_headers`-style config, so this isn't
  fixable while hosted here. Not urgent for a low-traffic marketing
  site, but if it matters later: front the domain with Cloudflare (own
  cache rules) or serve `img/` through a CDN that fronts this repo
  (e.g. jsDelivr) instead of directly from Pages.
