# Claude's Viral Growth Machine
## Setup

```bash
pip install aiohttp pandas requests

# Run the scraper
python scrapers/reddit_scraper.py          # outputs data/reddit_data.csv

# Run enrichment (comments + author history for top posts)
python scrapers/reddit_enrichment.py       # outputs reddit_comments.csv, reddit_authors.csv
```

No API keys required. Both scripts use Reddit's public JSON endpoints with a standard User-Agent header.

**This repository demonstrates local implementation of parser solution.**

**Link for the repository that shows implementation to use in automated machine:**
https://github.com/osp-d/test-hackathon-pipeline

For the automated pipeline, set two environment variables in GitHub Actions secrets:

```
TELEGRAM_BOT_TOKEN=...
TELEGRAM_CHAT_ID=...
```

---

## Repository Structure

```
├── scrapers/
├── └─reddit_scraper.py        # Async scraper — Tier 1 & 2 subreddits
├── └─reddit_enrichment.py     # Comment + author enrichment for top 200 posts
├── data/
│   ├──reddit_data.csv         # Primary dataset (scraper output)
├── ├──reddit_comments.csv     # Enrichment output — top-post comments
├── └──reddit_authors.csv      # Enrichment output — author behavior signals
├── docs/
│   ├── Playbook_Part_2.pdf         # Key findings + charts
│   ├── Building_Machine_Part_3.pdf # Architecture + cost analysis
│   └── Counter_Playbook_Part_4.pdf # Distribution strategy
└── README.md
```

---

## Assumptions

**Platform scope.** Only Reddit was scraped. YouTube, X, LinkedIn, and TikTok were evaluated and deprioritized: YouTube requires either the Data API (quota-constrained) or browser automation, X's free API tier is too restrictive for meaningful volume, and LinkedIn/TikTok lack usable public data endpoints. Reddit concentrates the highest-signal comparison and switching discussions and is the platform where Claude's growth is most legible from public data.

**Relevance filter.** A post is included only if its title or body contains `claude` or `anthropic`. Secondary AI terms (`llm`, `gpt`, `gemini`) were considered but dropped — they generate too many false positives (Claude Shannon references, generic LLM threads with no Claude mention). The stricter filter reduces volume but improves signal quality.

**Date floor.** Posts before March 1, 2023 (Claude's launch date) are dropped at parse time. Any earlier match is a false positive.

**Score floor.** Posts with score ≤ 0 are excluded. Zero-score posts are almost universally removed, deleted, or spam on Reddit and add no analytical value.

**Deduplication order.** The scraper deduplicates first by URL (catches exact reposts), then by title keeping the highest-score copy (catches the same story crossposted to multiple subreddits under the same headline). Title-dedup was added after noticing that identical posts were inflating subreddit-level counts.

**Enrichment scope.** Comments and author history are fetched only for the top 200 and top 100 posts by score, respectively. Enriching the full dataset would take several hours and hit rate limits. The top-post subset captures the posts that actually drove reach and is sufficient for the patterns being analyzed.

**Viral threshold.** "Viral" is defined as score > 5,000. "High" is score > 1,000. These thresholds were set by inspecting the score distribution of the collected dataset, not taken from external benchmarks.

---

## Tradeoffs

**Async scraping vs. rate limits.** The scraper runs at concurrency 5, which is near Reddit's informal cap for unauthenticated requests. This gives acceptable speed on a single daily run but will produce repeated 429s at 8+ runs per day (the 10x scenario). The fix is registering a free OAuth app, which raises the guaranteed limit to 100 QPM. This was not implemented here because it requires account credentials, but it is the first change needed before scaling.

**Cleaning inside the scraper vs. a separate step.** Score filtering, date filtering, and relevance filtering are all done inside `parse_post()` rather than in a post-processing script. This reduces dataset size early and avoids writing junk rows to disk, but it means the raw data is never saved. If the filter logic turns out to be wrong, you have to re-scrape. At the current volume this is acceptable; at 10x it would be worth separating the steps.

**Synchronous enrichment.** `reddit_enrichment.py` is fully synchronous. At current volume (~200 posts, ~100 authors) it runs in roughly 7 minutes. At 10x that becomes ~71 minutes, which breaks the daily-run assumption. Rewriting with `aiohttp` at concurrency 5 would drop it back to ~8 minutes. This is the highest-priority technical debt in the pipeline.

**GitHub Actions vs. a VPS.** GitHub Actions is free at current volume (under 1,200 minutes/month). At 10x (8 runs/day across more subreddits) the estimate is ~5,500 minutes/month, which exceeds the free tier and costs roughly $13/month in overage. A Hetzner CX11 VPS at €4/month is cheaper and removes the per-minute constraint entirely. The switch point is around month 3–4 of a scaled operation.

**Analysis engine.** All trend detection and alerting uses rule-based thresholds (pandas) rather than a statistical or ML model. This is a deliberate choice at the current scale: rules are interpretable, easy to tune, and don't require training data. The tradeoff is brittleness — a threshold that worked last month may miss a new type of spike. At 10x volume, a simple z-score anomaly detector would be worth adding alongside the existing rules.

**What the data cannot tell you.** The scraper captures public posts and their metadata. It cannot observe deleted posts before they are removed, shadow-banned accounts, coordinated upvote activity, or anything behind a login wall. The engineered vs. organic question (addressed in Part 2) is answered probabilistically from the shape of the data, not from direct observation of amplification mechanisms.

---

## What Broke and How It Was Handled

- **Pagination cutoff.** Reddit's public JSON endpoint returns a maximum of 1,000 results per search query regardless of pagination. High-volume queries hit this ceiling. Mitigation: multiple overlapping queries per topic (e.g., separate queries for "Claude vs ChatGPT", "switched to Claude", "Claude Sonnet") rather than one broad query, which surfaces different slices of the result set.

- **Rate limiting (429s).** Handled with a flat 30-second sleep on 429 responses. This is functional but brittle under sustained load. The correct fix is exponential backoff with jitter, noted as a future improvement.

- **Deleted/removed posts in enrichment.** Comment bodies containing `[deleted]` or `[removed]` are flagged with `is_deleted: True` and stored with an empty body rather than excluded. This preserves the comment's structural metadata (position, score, timing) which still carries signal even without text.
