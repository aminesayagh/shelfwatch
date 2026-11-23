# Use cases to confirm

Each entry describes a situation, what shelfwatch should do according to the
[architecture proposal](architecture.md), and the decision still open, if any. Mark each one
**confirmed**, **changed** or **out of scope** before implementation starts.

Status values: `proposed` (not yet discussed), `confirmed`, `changed`, `out of scope`.

---

## A. Runs and history

### A1. First run on a new URL · `proposed`
- **Situation:** the URL has never been watched.
- **Behaviour:** load the page 3 times, build the noise mask, save the first snapshot as the baseline,
  report "baseline created" and no changes.
- **Decide:** how far apart the 3 loads are (back to back, or a few seconds or minutes apart), and
  whether calibration can be rerun on demand (`shelfwatch calibrate URL`).

### A2. Normal run, nothing changed · `proposed`
- **Behaviour:** save the snapshot, report nothing, exit code `0`.
- **Decide:** whether to store an identical snapshot again, or record only "unchanged" in `runs.jsonl`
  to save space.

### A3. Normal run, something changed · `proposed`
- **Behaviour:** report each change as an event, exit code `1`.

### A4. The page fails (404, 500, timeout, DNS error) · `proposed`
- **Behaviour:** record a failed run, report no field changes, exit code `2`. The failed visit never
  becomes the reference for the next comparison.
- **Decide:** whether a page that stays 404 for N runs should become a `product_gone` event.

### A5. The product URL redirects (to a category, the home page or another product) · `proposed`
- **Behaviour:** treat it like a failure plus a `redirected` event with the new URL. Do not compare the
  category page with the product page.
- **Decide:** whether a redirect to a new product URL (a renamed slug) should move the target to the
  new URL automatically.

### A6. Long gap between runs · `proposed`
- **Behaviour:** same as a normal run. Many changes at once are expected.

### A7. Two runs of the same target at the same time · `proposed`
- **Behaviour:** the second run waits or refuses (a lock file in the target folder).

### A8. Many URLs in one command · `proposed`
- **Behaviour:** `shelfwatch run urls.txt` runs each target in turn, with per-host rate limits.
- **Decide:** whether v0.1 needs this or only a single URL.

---

## B. Noise: content that changes without meaning anything

### B1. Rotating banners, random recommendations, "people also bought" · `proposed`
- **Behaviour:** caught by calibration if they change within 3 loads (`text`, `presence` or `order` mask).

### B2. Timestamps, countdowns, "viewed 14 times today" · `proposed`
- **Behaviour:** caught by calibration when the number moves between loads.
- **Decide:** pattern masks (mask only the digits) in v0.1, or whole-node masks only.

### B3. Security tokens, nonces, session ids in attributes · `proposed`
- **Behaviour:** attributes are not stored at all, except `src`, `href`, `alt`, `content` and `value`
  on known elements. Tokens left in those are caught by calibration.

### B4. Cache-busting image URLs (`photo.jpg?v=1732`) · `proposed`
- **Behaviour:** caught by calibration when the parameter changes per load.
- **Decide:** whether to strip query strings from image URLs by default, which would also hide a real
  image swap done only through the query string.

### B5. Noise slower than calibration (daily deal, today's date, weekly banner) · `proposed`
- **Situation:** 3 loads a few seconds apart all look the same, so the mask misses it.
- **Behaviour:** reported as an `unlabeled` change on the next run.
- **Decide:** learning noise from history (a key that changed on every one of the last N runs and has
  no label is proposed as noise), and whether the user confirms that or it happens automatically.

### B6. A/B tests and personalisation · `proposed`
- **Situation:** the site serves variant A or B per session, or changes content by country, currency
  or cookies.
- **Behaviour:** calibration catches it only if the 3 loads land on different variants. Each load starts
  a fresh session, so it has a chance.
- **Decide:** whether to fix country, language and currency per target (headers, cookies) so runs are
  comparable.

### B7. Cookie banners, newsletter pop-ups, chat widgets · `proposed`
- **Behaviour:** usually identical every load, so not noise, and not interesting either.
- **Decide:** a built-in ignore list (common consent managers), user ignore rules per target, or both.

### B8. User marks something as noise by hand · `proposed`
- **Behaviour:** `shelfwatch ignore KEY` adds it to the mask.

### B9. Something in the noise mask turns out to matter · `proposed`
- **Behaviour:** labelling a node as a field always overrides the mask; `shelfwatch unignore KEY`
  removes a mask entry.

---

## C. Fields: the values to track

### C1. Price · `proposed`
- **Behaviour:** type `money`, normalised to an amount and a currency, so `€19,99`, `19.99 €` and
  `EUR 19.99` are equal. Prefer JSON-LD `offers.price` when the page publishes it.
- **Decide:** what happens when JSON-LD and the visible price disagree (it happens): trust one, or
  report the disagreement.

### C2. Sale price and original price · `proposed`
- **Situation:** "~~€29.99~~ €19.99". When the sale ends, the struck price disappears.
- **Behaviour:** two fields (`price`, `compare_at_price`). The end of the sale is `compare_at_price`
  becoming `missing` (a real change), not drift.

### C3. Stock as a status · `proposed`
- **Situation:** "In stock", "Out of stock", "Sold out", "Pre-order", "Back soon", a disabled
  "Add to cart" button, schema.org `availability`.
- **Behaviour:** type `availability`, normalised to `in_stock`, `out_of_stock`, `preorder`, `unknown`.
- **Decide:** whether a disabled buy button counts as a stock signal, and which languages the word lists
  cover (English and French first?).

### C4. Stock as a number ("Only 3 left") · `proposed`
- **Behaviour:** type `integer`, extracted from the text. Labelled, so never masked even if it moves.

### C5. Main image and gallery · `proposed`
- **Behaviour:** type `image` for one image; a `gallery` type for a list, reporting images added,
  removed and reordered. Images are compared by URL.
- **Decide:** whether to also download and hash images, to catch a new picture published under the same
  URL. This costs bandwidth on every run.

### C6. Product variants (size, colour) · `proposed`
- **Situation:** the price or stock differs per variant, often held in a JSON block in the page or
  loaded when a variant is clicked.
- **Behaviour:** v0.1 reads what the page shows by default, plus JSON-LD offers, which often list every
  variant.
- **Decide:** whether variant-level tracking is in scope, and how a variant is identified (SKU,
  option name).

### C7. Title, description, other text · `proposed`
- **Behaviour:** type `text`, compared after whitespace normalisation.

### C8. A value that exists only in structured data · `proposed`
- **Situation:** the SKU or GTIN is in JSON-LD but not visible on the page.
- **Behaviour:** fields can point into structured data only.

### C9. A page that lists several products (category, search, "shop the look") · `proposed`
- **Situation:** the price of product 3 out of 24 changed, but positions also shift.
- **Behaviour:** out of scope for v0.1: watch product pages.
- **Decide:** whether repeated items should later be keyed by product identity (link or SKU) instead of
  position.

### C10. Unlabelled changes · `proposed`
- **Behaviour:** anything that changed outside fields and outside the noise mask is reported as
  `unlabeled`, grouped by section, so the user can decide to label it, ignore it, or neither.
- **Decide:** whether unlabelled changes are reported by default or only with a flag (`--all`), since
  many users only care about their fields.

---

## D. Structure changes and drift

### D1. Small structure change around a field (a wrapper `div` added) · `proposed`
- **Behaviour:** the DOM key no longer matches; the fingerprint finds one clear candidate. v0.1 reports
  `field_drift` with that candidate as a hint. v0.2 proposes it for confirmation.
- **Decide:** whether a single, very confident candidate (same value, same nearby label, same section)
  may be accepted automatically.

### D2. The field's value disappears but the section is intact · `proposed`
- **Behaviour:** `field_missing`, a real change (see C2). Not drift.

### D3. The field's key now points to something of the wrong type · `proposed`
- **Situation:** the node that held "€19.99" now holds "Sold out".
- **Behaviour:** the value cannot be parsed as money, so it is `field_drift`, not a price of zero.
- **Decide:** whether "Sold out" in the price slot should also feed the `stock` field.

### D4. The key matches several nodes · `proposed`
- **Behaviour:** `field_drift`, never a guess.

### D5. Full redesign · `proposed`
- **Behaviour:** most keys changed; one `structure_changed` event, each field resolved or marked drift,
  and the noise mask flagged as stale.
- **Decide:** the threshold (share of keys changed), and whether a redesign triggers a new calibration
  automatically.

### D6. Relabelling after drift (v0.2) · `proposed`
- **Behaviour:** `shelfwatch relabel URL` shows each drifted field, its old value and fingerprint, and the
  best candidates in the new snapshot with the reason for each. The user picks one.

---

## E. Fetching and access

### E1. Content rendered by JavaScript · `proposed`
- **Behaviour:** HTTP mode finds no product content; the run reports this and suggests browser mode.
- **Decide:** whether to switch to browser mode automatically.

### E2. Bot protection, captchas, challenge pages · `proposed`
- **Behaviour:** detect them, fail the run with a clear reason, never try to get around them.

### E3. Prices that need a login or a postcode · `proposed`
- **Behaviour:** out of scope for v0.1.
- **Decide:** whether to support cookies supplied by the user for their own accounts (for example a
  B2B supplier portal the user has access to).

### E4. robots.txt disallows the page · `proposed`
- **Behaviour:** refuse to fetch and say so.
- **Decide:** whether the owner of a site may override this for their own site, with an explicit flag.

### E5. Rate limiting · `proposed`
- **Behaviour:** one request at a time per host, with a pause between requests. Calibration's 3 loads
  are spaced too.

### E6. Watching the user's own shop · `proposed`
- **Situation:** checking that the user's own storefront shows what the back office says.
- **Behaviour:** same tool; this is the simplest legal case.

---

## F. Output and integration

### F1. Sync into another system · `proposed`
- **Behaviour:** consumers read `events.jsonl` or `--json` output. shelfwatch does not write into other
  systems itself.

### F2. Human summary · `proposed`
- **Behaviour:** a short table in the terminal: field, before, after.

### F3. Exit codes for scripts and CI · `proposed`
- **Behaviour:** `0` no change, `1` changes, `2` failed run, `3` drift needing attention.

### F4. Webhooks or notifications · `proposed`
- **Behaviour:** later, not v0.1.

### F5. History of a field · `proposed`
- **Behaviour:** `shelfwatch history URL price` lists every value with its date, read from the events.

---

## G. Labelling

### G1. Labelling a field the first time · `proposed`
- **Behaviour:** `shelfwatch label URL price --value "19,99"` searches the latest snapshot for that value,
  lists the matching nodes (key, section, nearby text), and the user picks one. Structured data matches
  are shown first.
- **Decide:** whether to also accept a CSS selector or a DOM key directly, for users who know the page.

### G2. Labels shared by many pages on one site · `proposed`
- **Situation:** 500 product pages on one shop use the same template.
- **Behaviour:** later. Labels are per target in v0.1.
- **Decide:** whether site-level label templates are worth designing now, since that is how the tool
  would scale.

### G3. Changing or removing a label · `proposed`
- **Behaviour:** edit `fields.toml` or use `shelfwatch unlabel`. The field's history in `events.jsonl`
  stays.

---

## Decisions summary

The open decisions above, in suggested order of importance:

1. `missing` versus `drift`: the rule in the architecture document (D2, C2).
2. Noise slower than calibration: learning from history (B5).
3. Scope of v0.1: single URL or a list (A8), unlabelled changes on or off (C10), variants (C6).
4. Images: compare by URL only, or also by content hash (C5, B4).
5. JSON-LD versus visible value when they disagree (C1).
6. How a user picks a node to label (G1).
7. Automatic acceptance of a confident relocation (D1).
8. Locale, currency and session settings per target (B6).
