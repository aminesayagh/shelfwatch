# shelfwatch

**Watch a product page and report what changed since the last visit: price, stock, images, and anything
else you label.**

Every run of shelfwatch on a URL saves a structured snapshot of the page, compares it with the previous
snapshot, and reports the differences as events you can sync into other systems (a shop, a spreadsheet,
an ERP):

```
price      19.99 → 24.99
stock      in_stock → out_of_stock
images     gallery: 1 image added (https://…/blue-side.jpg)
unlabeled  "Free shipping over $50" changed in section #shipping
```

> **Status: design phase.** No code yet. This repository currently holds the design so the use cases can
> be discussed and confirmed before anything is built:
> - [`docs/architecture.md`](docs/architecture.md): how it is meant to work.
> - [`docs/use-cases.md`](docs/use-cases.md): every situation it should handle, each with a proposed
>   behaviour and the decisions still open.

## The idea in four steps

1. **Calibrate (first run only).** The page is loaded three times. Any part of it that differs between
   those loads (a rotating banner, a timestamp, a security token, a random recommendation) is not real
   content. It is saved in a **noise mask** and ignored in later comparisons.
2. **Snapshot (every run).** The page becomes a JSON tree: one entry per piece of visible content, with
   a key that says where it sits in the DOM, plus the page's structured data (JSON-LD, meta tags).
3. **Compare.** The new snapshot is compared with the previous good one, with the noise mask applied.
   Each difference is reported together with the place it happened (which part of the page).
4. **Label.** You tell shelfwatch which part of the tree holds a value you care about ("this node is
   `price`, a money amount"). From then on, a change there is reported as `price: 19.99 → 24.99`.
   When the page's structure changes so that the label no longer finds its value, shelfwatch reports
   **structure drift** and asks for the label to be redefined, instead of reporting a wrong value. A
   later version will help with that relabelling by finding where the old value moved.

## What it is not

- **Not a scraper of whole catalogues.** It watches pages you list. Discovering which pages exist is a
  different job; see [webprint](https://github.com/aminesayagh/webprint)'s `route-map` for that.
- **Not a way past bot protection.** It respects `robots.txt`, rate-limits itself, and stops when a page
  answers with a challenge or a captcha. It does not try to evade them.
- **Not a scheduler.** One command does one run. Use cron, a CI schedule or any task runner to repeat it.

## Related project

shelfwatch reuses ideas from [webprint](https://github.com/aminesayagh/webprint), which captures a page
in a real browser and describes its sections. In particular, it reuses the DOM-path keys and the
"same text first, then same DOM slot" matching that webprint uses to compare three loads of a page.

## Roadmap

| Version | Scope |
|---|---|
| **v0.1** | Calibration, snapshots, the diff, labelled fields, drift detection, events as JSON lines, one URL or a list of URLs |
| **v0.2** | Assisted relabelling: when a field drifts, propose where its value moved |
| **later** | Learning noise over time, webhooks, a small history viewer |
