# Architecture (proposal)

Status: draft for discussion, 2026-11-23. Nothing here is built yet. Decisions marked **Open** are listed
again, with options, in [`use-cases.md`](use-cases.md).

## Vocabulary

| Term | Meaning |
|---|---|
| **Target** | One URL being watched, with its settings, labels and history. |
| **Snapshot** | The structured artifact produced by one visit: the page as a JSON tree. |
| **Node** | One entry of the snapshot: a piece of content (text, image, attribute) at a position in the DOM. |
| **Key** | A node's address, stable across visits when the page's structure does not change: e.g. `#product>div[1]>span`. |
| **Noise mask** | The list of keys (or parts of values) known to change on every load, ignored when comparing. |
| **Field** | A label the user gives to one node: `price`, `stock`, `main_image`. It has a type and a locator. |
| **Event** | One reported change: a field's value changed, a field drifted, or unlabelled content changed. |
| **Drift** | A field's locator no longer finds its node with confidence: the structure changed, and the label needs redefining. |

## One run

```
            ┌──────────────┐
  URL ────► │    fetch     │  HTTP, or a real browser for JavaScript-rendered pages
            └──────┬───────┘
                   ▼
            ┌──────────────┐
            │ health check │  status code, challenge page, error page → failed run, no comparison
            └──────┬───────┘
                   ▼
            ┌──────────────┐
            │   snapshot   │  DOM → node tree + structured data (JSON-LD, meta, microdata)
            └──────┬───────┘
                   ▼
   first run? ── yes ──► calibrate: 2 more snapshots → noise mask → save baseline → stop
                   │ no
                   ▼
            ┌──────────────┐
            │resolve fields│  each label → its node now → typed value, or missing, or drift
            └──────┬───────┘
                   ▼
            ┌──────────────┐
            │     diff     │  vs. the previous good snapshot, noise mask applied
            └──────┬───────┘
                   ▼
            ┌──────────────┐
            │    report    │  events appended to events.jsonl, summary printed, exit code
            └──────────────┘
```

## The snapshot

A snapshot keeps only what a visitor or a shop system would care about, and drops everything else
(scripts, styles, class names, layout boxes).

```json
{
  "schema": 1,
  "url": "https://shop.example/p/blue-shirt",
  "final_url": "https://shop.example/p/blue-shirt",
  "fetched_at": "2026-11-23T10:00:00Z",
  "mode": "http",
  "status": 200,
  "structured": {
    "jsonld": [{ "@type": "Product", "name": "Blue shirt", "offers": { "price": "19.99", "priceCurrency": "EUR", "availability": "InStock" } }],
    "meta": { "og:title": "Blue shirt", "product:price:amount": "19.99" }
  },
  "nodes": [
    { "key": "#product>h1", "tag": "h1", "text": "Blue shirt", "section": "#product" },
    { "key": "#product>div[1]>span", "tag": "span", "text": "€19,99", "section": "#product" },
    { "key": "#gallery>img[0]", "tag": "img", "src": "https://cdn.example/blue-front.jpg", "alt": "Front", "section": "#gallery" }
  ]
}
```

- **Keys** follow webprint's DOM path rule: the path from the nearest ancestor with an `id`, with
  siblings counted per tag. Class names are left out, because many sites generate them on every build
  (`css-1x2y3z`). **Open:** whether some stable classes should be allowed in keys.
- **`section`** is the nearest landmark or `id` above the node. It answers "where did the change happen"
  without running webprint's full section detection.
- **Structured data** is stored separately because it is usually the most reliable source for price and
  stock: shops publish it for search engines, and it changes much less often than the HTML around it.

## Calibration and the noise mask

On the first run, the page is snapshotted three times. Every key whose value differs between those
loads goes into `noise.json`, together with the kind of difference:

| Kind | Example |
|---|---|
| `text` | "Viewed by 14 people today", a countdown, a date |
| `attr` | an `src` carrying a cache-busting `?v=…`, a CSRF token in a `value` |
| `presence` | a node present in one load and absent in another (a random promo) |
| `order` | a list whose items come in a different order (shuffled recommendations) |

Rules:

- **A labelled field is never masked.** If the user says "this is the stock", it is compared even if it
  changed during calibration ("Only 3 left" → "Only 2 left" is real information).
- **Masks are by key, and optionally by pattern.** For partly volatile text ("Updated 3 minutes ago"),
  a pattern mask replaces only the varying part (digits) instead of hiding the whole node.
  **Open:** whether patterns are in v0.1.
- **Three loads close together miss slow noise:** daily deals, a "today" date, A/B tests that switch per
  session. **Open:** spacing the loads apart, and learning noise from history later (a key that changed in
  every one of the last N runs, with no label, is proposed as noise).

## Fields (labels)

A field is defined in the target's `fields.toml`:

```toml
[price]
type = "money"                         # money | integer | availability | image | url | text
locate.jsonld = "Product.offers.price" # preferred when the page publishes structured data

[stock]
type = "availability"                  # normalised to in_stock | out_of_stock | preorder | unknown
locate.path = "#product>div[2]>p"
locate.near_text = "Availability"      # a label printed next to the value: helps detect drift

[main_image]
type = "image"
locate.path = "#gallery>img[0]"
```

- **Locators**, tried in order: structured data path, then DOM key. Each field also stores a
  **fingerprint** taken when it was labelled: its tag, its section, the text printed next to it, and its
  value at that time.
- **Types normalise values** so formatting is not reported as change: `€19,99`, `19.99 €` and
  `EUR 19.99` are the same money amount.
- **How the user picks a node:** **Open.** The simplest option is a command that searches the latest
  snapshot for a value the user can see on the page (`shelfwatch label price --value "19,99"`) and
  shows the matching keys.

### Field states after each run

| State | Meaning | Event |
|---|---|---|
| `unchanged` | Found, same normalised value | none |
| `changed` | Found, different value | `field_changed` with before and after |
| `missing` | The structure around it is intact, but the node is gone (a sale price that ended) | `field_missing` |
| `drift` | The locator fails or matches a node whose fingerprint no longer fits | `field_drift`: relabel needed, no value reported |

The difference between `missing` and `drift` is the hardest decision in the design. The proposal: if
the field's section and its neighbours are still there, the value is **missing** (a real change);
if they are gone or moved, it is **drift** (the page changed, not the product).

## The diff

- Nodes are matched by key, then by tag and text (content that moved), as webprint does across
  breakpoints. Unmatched nodes are added or removed.
- Masked keys are skipped. Labelled fields are reported through their fields, not again as nodes.
- Changes outside fields are reported as `unlabeled` events, grouped by section, so a new block of
  content shows up once, not once per line.
- A large share of changed keys (**Open:** threshold) means the page was redesigned. One `structure_changed`
  event is reported, plus each field's state, instead of hundreds of node events.
- The comparison is always against the **previous good snapshot**: a failed run (404, timeout, captcha)
  never becomes the reference.

## Storage

Plain files, one folder per target, readable and easy to put under version control:

```
watch/
  shop.example__p__blue-shirt/
    target.toml          URL, fetch mode, calibration settings
    fields.toml          labels
    noise.json           noise mask
    snapshots/
      2026-11-23T10-00-00Z.json
      2026-11-24T10-00-00Z.json
    events.jsonl         one JSON line per event, appended
    runs.jsonl           one line per run: time, status, counts
```

**Open:** how many snapshots to keep. SQLite is the upgrade path when there are thousands of targets.

## Events (the sync interface)

Other systems read `events.jsonl` (or the command's output with `--json`):

```json
{"run": "2026-11-24T10-00-00Z", "url": "https://shop.example/p/blue-shirt", "type": "field_changed",
 "field": "price", "before": {"amount": 19.99, "currency": "EUR"}, "after": {"amount": 24.99, "currency": "EUR"}}
```

Exit codes make scripting easy: `0` nothing changed, `1` changes found, `2` the run failed, `3` a field
drifted.

## Fetching

- **HTTP mode** (default): fast and cheap, enough when the page or its JSON-LD carries the values.
- **Browser mode** (Playwright): for pages that render their content with JavaScript. Slower, and
  closer to what webprint does.
- **Always:** obey `robots.txt`, one request at a time per host by default, a fixed user agent, and stop
  with a clear failure on challenge pages, logins and captchas.

## Technology (proposal)

Python 3.12 with uv, like webprint, so code and conventions can be shared. HTTP with `httpx`, HTML with
`lxml`, an optional browser with Playwright, models with Pydantic, CLI with Typer. Copy what is needed
from webprint rather than depending on it, since webprint has no stable API yet.

## Version 2: assisted relabelling

When a field drifts, shelfwatch searches the new snapshot for its fingerprint: the same value (if it did
not change), the same nearby label text ("Price"), the same section, the same tag. It proposes the best
candidates with a reason for each, and the user confirms one. Nothing is relabelled silently.
