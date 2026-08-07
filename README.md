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
| **25%** | Flavour | Guest multi-selects real tasting notes (from the wine data's Flavour1/Flavour2 columns, e.g. "Apple", "Baked Spices"); score = the fraction of that wine's own 1–2 flavour notes the guest picked. No picks = neutral 50%, never a penalty |

The Flavour (and Aroma) picker only shows notes that actually occur on
wines of the type the guest already chose in step 1, so a guest is
never offered a note that can't possibly show up in their results.
Choosing "Surprise Me" shows the full catalogue-wide list.

**Occasion, Food and Aroma are unscored, personalization-only** — asked,
shown in results copy, captured on the lead, zero ranking weight. Each
is skippable (Occasion/Food via an explicit "Skip" choice, Aroma by
simply not tapping any tag).

The exact formulas (`categoryMatch`, `priceMatch`, `flavourTagMatch`,
`getAvailableTags`, `scoreWine`) are in the `<script>` block in
`index.html` — search for "SCORING". The star display itself is just a
presentation layer over the same 0–100 score (`score.pct / 20`); it
doesn't change how wines are ranked.

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
