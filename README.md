# Fratelli Digital Sommelier — Web (HTML/CSS/JS)

A single self-contained page (`index.html`) — no build step, no server,
no dependencies. Open the file directly in a browser, or serve the
`site/` folder with any static host.

## What it does

Landing page → 7-question quiz (wine type → occasion → food pairing →
aroma → flavour → style → budget) → 3 ranked recommendations with a
live match-score breakdown → optional email signup → thank-you screen.
Built for a client demo: dark wine/gold brand styling (matching
`resources/Dig som/index.html`), animated screen transitions, Ken Burns
backgrounds, typewriter copy, animated count-up match percentages and
score bars.

Occasion, food, aroma and style are all **skippable and unscored** —
they shape the results-screen copy ("Curated for your Hosting evening,
paired with Italian cuisine, with a nose of Vanilla & Cherry…") and the
captured lead, but never affect ranking.

## Where things live

- **Wine catalogue**: inlined as the `WINES` array in `index.html`,
  extracted from `resources/Wine_logic.xlsx` (see
  `digital_sommelier/data.py` in the repo root for the extraction/
  cleanup logic — the same source both the Python CLI and this site
  use).
- **Background photos**: `img/*.jpg`, re-compressed from
  `resources/Dig som/img/` (the originals total ~10.7MB; these are
  ~860KB combined) so the page stays light and fast to load.

## Recommendation scoring (as specified by the client)

Every wine is scored 0–100% as a weighted blend:

| Weight | Signal | Excel column(s) | How it's matched |
|---|---|---|---|
| **50%** | Category | Wine type | Exact type match = full credit; "Surprise Me" = full credit for every type |
| **25%** | Price | Price Range | Exact band = full credit; an adjacent band = partial credit; "doesn't matter" = full credit |
| **25%** | Flavour | **Flavour1 / Flavour2** (the actual tasting-note text columns, e.g. "Apple", "Baked Spices" — *not* the Body/Fruit/Oak/Sweetness L/M/H scale) | Guest multi-selects flavour notes on the Flavour screen; score = the fraction of that wine's own 1–2 flavour notes the guest picked (no picks = neutral 50%, never a penalty) |

The Flavour (and Aroma) picker only shows notes that actually occur on
wines of the type the guest already chose in step 1 — e.g. picking
"Red" restricts the list to the ~14 notes that appear on the 9 red
wines, not all ~29 across the whole catalogue — so a guest is never
offered a note that can't possibly show up in their results.
Choosing "Surprise Me" shows the full catalogue-wide list.

**Everything else is unscored, personalization-only**, all following
the same "asked, shown in results copy, captured on the lead, zero
ranking weight" pattern:
- **Occasion** and **Food** (single-select, each with an explicit Skip)
- **Aroma** (Aroma1/Aroma2 columns — same restricted multi-select
  picker as Flavour, just never fed into `scoreWine`)
- **Style** — the spreadsheet's Body/Fruit/Oak/Sweetness L/M/H scale.
  This *used* to be the scored "flavour" signal in an earlier version
  of this build; it's kept as a style question guests can still
  answer, but the Flavour1/2 tag match above is what actually earns
  the 25%.

The exact formulas (`categoryMatch`, `priceMatch`, `flavourTagMatch`,
`getAvailableTags`, `scoreWine`) are in the `<script>` block in
`index.html`, each with a short comment — search for "SCORING".

## Before going live

1. **Meta Pixel** — in `<head>`, replace `YOUR_PIXEL_ID` (two
   occurrences: the `fbq('init', …)` call and the `<noscript>` fallback
   `<img>` src) with the real Pixel ID from Meta Events Manager. Until
   then you'll see a harmless `[Meta Pixel] - Invalid PixelID` console
   warning — the rest of the funnel is unaffected.
2. **Visitor location** — the landing page silently calls
   [ipapi.co](https://ipapi.co)'s free JSON endpoint to greet visitors
   by city and to tag captured leads with approximate location. The
   free tier is rate-limited and fine for a demo/kiosk; for production
   traffic, swap `detectVisitorLocation()` for a server-side lookup or
   a paid geolocation service.
3. **Lead capture** — there's no backend. Submitted signups are logged
   to the browser console and saved to `localStorage['fratelli_leads']`
   so you can see the exact payload a real integration would receive
   (name, email, mobile, quiz answers, recommendations, visitor
   location, referrer/UTM, timestamp). Wire `saveLead()` in
   `index.html` to your actual CRM/email service before launch.

## Signup fields

Per spec: **email is required**, **mobile is optional** (name is
optional too, to keep friction low). Client-side validation only —
add server-side validation once this is wired to a real endpoint.
