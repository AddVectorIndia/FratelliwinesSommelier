# Fratelli Digital Sommelier — Web

A single self-contained page (`index.html`) — no build step, no server,
no dependencies. Open the file directly in a browser, or serve this
folder with any static host. Live at **sommelier.addvector.com**.

This repo holds only the deployable site (`index.html`, `img/`). The
Python CLI and source data (`Wine_logic.xlsx`, original kiosk assets)
that this build is based on live in a separate local-only workspace —
intentionally not published here.

## What it does

Landing page → 5-question quiz (wine type → occasion → food pairing →
palate sliders → budget) → 3 ranked recommendations in a single-card
carousel (bottle photo, Variety/Serve/Pair, prev/next pagination) →
optional email signup → thank-you screen. Dark wine/gold brand
styling, animated screen transitions, Ken Burns backgrounds,
typewriter copy.

Food is **skippable and unscored** — it shapes the captured lead but
never affects ranking (see Recommendation scoring below for what
*does* rank the wines).

Choice buttons are plain text, no emoji glyphs — every `.choice-btn`
used to lead with a pictograph icon (🍷, 🥳, 🍛, …); those are gone,
the buttons are centered label text only. The `✦` used on Skip / "no
particular occasion" buttons and the "doesn't matter" price option is
gone too, dropped along with the rest for a consistent, icon-free look
across every choice screen.

## Navigation

A back button (top-left, mirrors the theme toggle's circular top-right
placement) appears on every screen from Wine Type through Signup —
hidden on Landing (nothing before it) and Thanks (a terminal
confirmation screen, not part of the back chain). It's one shared
element (`#back-btn`), not one per screen: `goBack()` looks up the
current screen in `PREV_SCREEN` (a flat "what comes before me" map,
sufficient since the whole quiz is a single straight line — even the
Signup "Skip for now" link lands on the same next screen a real submit
would) and re-shows/hides the
button via `updateBackButton()`, called from `goTo()` on every
navigation. Going back preserves whatever was previously selected on
that screen (`.selected` classes aren't cleared by navigating away),
so retracing your steps shows your prior answer still highlighted.

## Recommendation scoring

Every wine is scored 0–100% as a weighted blend. That score isn't
shown to the guest directly (no star rating or match percentage on the
card) — it's used only to rank and order the three recommendations:

| Weight | Signal | How it's matched |
|---|---|---|
| **45%** | Category (wine type) | Exact type match = full credit; "Surprise Me" = full credit for every type |
| **35%** | Price | Exact band = full credit; an adjacent band = partial credit; "doesn't matter" = full credit |
| **15%** | Occasion | Exact membership in the wine's own `occasions` list = full credit, otherwise zero; "no particular occasion"/no pick = full credit for every wine |
| **5%** | Palate (Body/Fruit/Oak/Sweetness sliders) | See below |

Food is asked and captured on the lead but has no per-wine data to
check against at all, so it stays personalization-only, zero weight —
same as it's always been.

**Palate matching**: the wine data's own Body/Fruit/Oak/Sweetness
columns (Low/Medium/High, mapped to 1/2/3 — Oak's extra "None" value
folds into "Low") are compared against the guest's 4 slider picks one
dimension at a time: `1 − |userValue − wineValue| / 2`, so a 2-step
gap (e.g. picking Low against a High wine) scores 0, adjacent scores
0.5, exact scores 1.0. The 4 dimension scores are averaged into one
number for the 5% weight — the same distance-based partial credit
`priceMatch` already uses for adjacent price bands, just applied
per-dimension. No slider input at all (shouldn't happen — they default
to Medium/2 on page load) falls back to neutral 0.5.

The exact formulas (`categoryMatch`, `priceMatch`, `occasionMatch`,
`palateMatch`, `scoreWine`) are in the `<script>` block in
`index.html` — search for "SCORING". `score.pct` (0–100) only ever
drives sort order now — there's no visible rating number, star
display, or score-bar breakdown on the card at all (see Results card
below for what replaced it).

### Palate screen

Replaces what used to be two separate screens (Aroma, then Flavour —
both single-select tag pickers). One screen now, "Let's understand
your palate," with 4 native `<input type="range">` sliders (Body,
Fruit, Oak, Sweetness; min 1/max 3/step 1, defaulting to 2/Medium).
Each slider's fill and its Low/Medium/High value label repaint live on
every `input` event (`paintPalateSlider()`) — plain range inputs have
no cross-browser "filled track" styling, so the gold portion is a
manually-computed `background: linear-gradient(...)` recalculated on
every drag. No auto-advance here (unlike every single-select screen
elsewhere in the quiz) — with 4 independent sliders there's no one tap
that means "done," so guests move on via a **Next →** button, styled
and positioned exactly like the primary button on every other screen
(`.btn-primary`, centered below the sliders) rather than a bespoke
control of its own. No instructional subtext under the headline either
("Slide each to match your taste…") — the Low/Medium/High tick labels
under each slider already say that, so the line was just repeating
itself while eating vertical space the sliders can use instead.

### Results card

One recommendation shown at a time in a paginated carousel
(`.rec-carousel`, `showRecCard()`), not a scrollable stack of three —
prev/next arrows plus a "Recommendation 0X/03" label, clamped (not
wrapping) at both ends. Each card: the wine's own bottle photo on the
left (`img/bottles/`, see below), and on the right an eyebrow ("Perfect
Match!" for rank 1, "Great Match!" for the other two), the wine name
in large italic gold script, then three labeled spec rows — Variety,
Serve, Pair — each separated by a thin divider. A consent checkbox
("I agree to receive updates…", `state.marketingConsent`) and two
side-by-side buttons, Email My Selection (→ the existing signup
screen, unchanged) and Restart (→ `restartQuiz()`, same as the Thanks
screen's "Take the Quiz Again"), sit below the carousel; a smaller "No
thanks, I'm done" link (→ `skipSignup()`) stays available underneath
for guests who want to leave without either.

### Wine data: variety, bottle photos, Serve/Pair

`variety` (grape blend, e.g. "Sangiovese & Cabernet Sauvignon") and
`bottleImg` (`img/bottles/<slug>.jpg`) are real, wine-specific data —
scraped once from each wine's own product page on fratelliwines.in
(their `Variety: …` description text and `og:image` product photo) and
saved directly into the `WINES` array and `img/bottles/`, not
generated or fetched at runtime. Three wines (Noi Sparkling, Noi
Sparkling Rosé, Master Selection Late Harvest) don't state an exact
blend on their page, so those three carry an honest generic label
("Sparkling Blend", etc.) rather than a guessed one. Bottle photos were
flattened onto white and re-compressed as JPEG (~20–35 KB each,
matching this repo's existing photo convention) — the source PNGs'
off-white backdrop is genuinely baked into every pixel (verified: 100%
opaque, not unset alpha), so it can't be dropped out losslessly;
`.rec-card-photo` frames it with a border + rounded corners instead of
fighting it. Each photo is also auto-cropped tight to just the bottle's
own bounding box (detected by color-diffing every pixel against that
image's own corner/background color, at a high enough threshold to
ignore the very faint "F" watermark baked into the backdrop and lock
onto the much higher-contrast bottle silhouette) plus a small margin,
rather than shipping the full product-shot canvas — since the card
frame renders at a fixed width (`object-fit: contain`), a tighter crop
means the bottle actually fills that frame instead of floating in a
sea of white space.

**Serve (temperature/decant) and Pair (food pairing) are *not*
per-wine scraped data** — fratelliwines.in doesn't publish either field
on individual product pages, only Variety. `SERVE_PAIR_BY_TYPE` in
`index.html` is standard sommelier serving convention keyed by wine
*type* (red/white/rosé/sparkling/dessert), not a per-SKU fact, and is
documented as such in the code rather than presented as something
specific to that exact bottle.

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
`Lead` event. The Results screen's "I agree to receive updates"
checkbox (`state.marketingConsent`) rides along in that local backup
log too, but isn't forced into the Fratelli Enquiry API call above —
that endpoint's payload shape is fixed/external, not something to
extend on our own.

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

**The Club Fratelli (signup) screen gets its own stronger overlay.**
Its heading/labels sit higher up `vineyard.jpg` than most screens'
content — right over a bright open-sky patch the shared gradient
(`--overlay-1` at the top, fading to `--overlay-2`) doesn't reliably
darken enough, most visible in dark theme where light text lost
contrast against that patch. `#s-signup .screen-overlay` overrides the
gradient to start at the darker `--overlay-2` immediately rather than
easing in from `--overlay-1`; a `--text-scrim` text-shadow (a black
halo in dark theme, white in light theme) on the eyebrow/headline/
subtext/first field label backs it up as a second layer, so legibility
doesn't depend on getting the overlay tuning exactly right for every
possible photo underneath.

**The landing page's brand mark is the real Fratelli monogram**, not a
placeholder. `img/logo.png` — the "F" + vine leaves + grape cluster
icon cropped out of fratelliwines.in's own footer logo (`Footer_logo.png`
on their Shopify CDN) and recolored solid to this site's gold accent
(`#C4973E`, dark theme's `--gold`) — replaces the old plain-text "F"
in a circle. It's used as-is across both themes (a fixed gold rather
than swapping per-theme like `--gold` does) since it always sits over
a photo + overlay, never directly on the flat `--bg` color, so the
small light/dark-theme gold difference isn't worth two separate
recolored assets.

Verified via Chrome DevTools Protocol's `Emulation.setEmulatedMedia`
(forces `prefers-color-scheme` without an actual OS-level toggle)
across the full quiz in both themes, plus the manual toggle itself:
persistence across reload and the image swap firing correctly and
immediately on toggle.

**Card surfaces are deliberately translucent in both themes**
(`--surface-1/1-end/2` sit around 1–5% alpha — a dark tint in light
theme, a light tint in dark theme) — a solid card would read as a flat
paper cutout against the photo behind it; the barely-there tint is
what gives it the "glass over the photo" look both themes share. Light
theme's tint briefly went to ~88–92% opaque as a workaround for the
dark-band bug below, before the actual scroll bug was found and fixed
at the source, and even after reverting toward the original ~3.5%
value it still read as a slightly whitish, too-solid block against the
photo — currently at ~1.8–2.2%, near-invisible on its own and only
really visible via the border + text contrast.

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

## Selected buttons don't scale (`.choice-btn`)

`.choice-btn.selected` used to also `transform: scale(1.03)` on
selection. `transform` doesn't affect grid/flex layout, so the
scaled-up box visually grew past its own cell into the gap — and since
it had no `z-index`, a plain sibling that comes *after* it in the DOM
(e.g. the next grid cell in the same row) still painted on top of that
overflow by source order, making the selected button appear to dip
*behind* its neighbor right at the moment it's picked. First fix
attempt widened the grid gaps and added `z-index`, which reduced but
didn't fully eliminate it at every viewport width. Current fix drops
the scale transform entirely — selection is border-color + background
only, the same same-footprint treatment `.price-btn.selected` already
used without ever having this problem — so there's no growth to spill
into a neighbor in the first place, regardless of gap size or
viewport. `.choice-grid`'s gap (`0.85rem`) stays at its widened value
for general breathing room, it just isn't load-bearing for this fix
anymore.

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
