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

Follows the visitor's OS/browser preference automatically
(`prefers-color-scheme`) — no manual toggle. Every color in the
stylesheet is a CSS custom property (`--bg`, `--text`, `--text-dim`,
`--text-muted`, `--surface-*`, `--border-*`, `--track-*`, `--gold`,
`--gold-light`, `--overlay-1/2`, …) defined once in `:root` for dark
(the default) and re-defined under `@media (prefers-color-scheme:
light)` — nothing else in the file references a literal color, so the
whole site reskins from those two blocks alone.

A few things are deliberately **not** theme-swapped:
- `--ink` (near-black text sitting on a gold-filled element — buttons,
  selected chips) stays fixed, since gold's brightness barely changes
  between themes and dark text keeps working on it either way.
- `--gold`/`--gold-light` *do* change (darkened for light mode — the
  pale dark-mode gold reads at very low contrast against a light
  background), but `--gold-dim`/`--gold-wash-*` (translucent gold used
  for hover/selected washes) don't bother, since a translucent gold
  tint reads fine as an accent on both a near-black and a near-white
  surface.
- `--overlay-1`/`--overlay-2` (the scrim over every photo background)
  flip from a dark wash to a light one in light mode, paired with dark
  `--text`, rather than just getting a little less dark — otherwise
  light-mode text would still need to be pale to read on a dark scrim,
  defeating the point.

Verified via Chrome DevTools Protocol's `Emulation.setEmulatedMedia`
(forces `prefers-color-scheme` without needing an actual OS-level
toggle) across the full quiz in both themes — landing, a choice
screen, the tag pickers, results, and signup all render with correct
contrast and no leftover hardcoded colors.

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
