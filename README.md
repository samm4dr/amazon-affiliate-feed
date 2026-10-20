# Build an Amazon Affiliate Feed with Live Prices in 2026

Static prices kill affiliate conversions. A "best standing desks" table that says $299 while Amazon shows $349? Your reader bounces. Live data converts — here's how to build a self-updating product feed with Apify actors.

## The stack

- **[Apify](https://console.apify.com/sign-up?fpr=dx06p)** — `apify/amazon-crawler` handles Amazon's anti-bot wall. Free credits to start.
- **Nightly schedule** — fresh prices every morning, zero maintenance.

## Step 1 — Token

[Create an Apify account](https://console.apify.com/sign-up?fpr=dx06p) → Settings → Integrations → API token → `export APIFY_TOKEN=...`

## Step 2 — Scrape ASINs in batch

```python
from apify_client import ApifyClient

client = ApifyClient("APIFY_TOKEN")
asins = ["B0ABC12345", "B0DEF67890", "B0GHI11223"]

run = client.actor("apify/amazon-crawler").call(run_input={
    "productUrls": [{"url": f"https://www.amazon.com/dp/{a}"} for a in asins],
    "proxyCountry": "AUTO_SELECT_PROXY_COUNTRY",
})

for item in client.dataset(run["defaultDatasetId"]).iterate_items():
    print(item["asin"], item["title"], item.get("price"), item.get("inStock"))
```

Fields: `asin`, `title`, `price`, `listPrice`, `stars`, `reviewsCount`, `inStock`, `thumbnailImage`, `url`.

## Step 3 — Scheduled snapshots → live table

```python
import json, os

DB = "prices.json"
current = {}
for item in client.dataset(run["defaultDatasetId"]).iterate_items():
    current[item["asin"]] = {
        "price": item.get("price"),
        "stock": item.get("inStock"),
        "stars": item.get("stars"),
    }

old = json.load(open(DB)) if os.path.exists(DB) else {}
for asin, p in current.items():
    if asin in old and p["price"] and old[asin]["price"]:
        drop = (old[asin]["price"] - p["price"]) / old[asin]["price"]
        if drop > 0.10:
            print(f"PRICE DROP {asin}: {old[asin]['price']} -> {p['price']}")

json.dump(current, open(DB, "w"))
```

Put the actor on a daily schedule in Apify Console → this script runs via cron/Actions → your comparison pages always show today's price.

## Step 4 — Reviews for content angles

`junglee/amazon-reviews-scraper` with `filterByRating: "critical"` pulls the worst reviews first — exactly what you need for "pros & cons" sections that readers trust:

```bash
curl -X POST "https://api.apify.com/v2/acts/junglee~amazon-reviews-scraper/run-sync-get-dataset-items?token=$APIFY_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"productUrls": [{"url": "https://www.amazon.com/dp/B0EXAMPLE"}], "maxReviews": 100, "filterByRating": "critical"}'
```

## Pro tips

- `proxyCountry` must match the marketplace (US → .com, DE → .de) or data comes back wrong/blocked.
- Batch ASINs in one run; re-run only failures.
- Pair prices with your Associates tag in the outbound URL — `?tag=yourtag-20`.

More tools on [awesome-web-scraping](https://github.com/g4dr/awesome-web-scraping). Agent users: `npx skills add g4dr/apify-skills` — the `apify-amazon-products` skill does this end-to-end ([repo](https://github.com/g4dr/apify-skills)).

*What niche are you building a feed for?*

---

## Resources

- [Apify](https://console.apify.com/sign-up?fpr=dx06p) — the scraping platform used in this tutorial (free credits on signup)
- [g4dr/apify-skills](https://github.com/g4dr/apify-skills) — Agent Skills that run these pipelines from a prompt
- [g4dr/awesome-web-scraping](https://github.com/g4dr/awesome-web-scraping) — curated list of scraping tools, proxies and actors
