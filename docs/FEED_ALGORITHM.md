# Feed Ranking Algorithm

Source: `lib/domain/feed_ranker.dart`. Pure Dart, no Flutter imports, unit-testable.

The Home tab has three feeds:

| Feed | Order |
|---|---|
| **For You** | Ranked by the score below |
| **Following** | Newest first, only people you follow |
| **Trending** | Fastest-growing posts in the last 72 hours |

## For You score
```
score = (1 + ln(1 + engagement))
      × (1 + 1.5 · authority)
      × (1 + relevance)
      / (ageHours + 2) ^ 1.15
```

| Term | Formula | Purpose |
|---|---|---|
| **engagement** | `likes + 3·comments + 1.5·votes` | Comments count most because they show real discussion; poll votes are in between. Log-scaled so a viral post doesn't drown out everything else. |
| **authority** | `ln(1 + followers) / ln(100 001)` → 0 to 1 | Popular creators get a boost, but it's log-scaled and capped. Accounts with ≥ 10k followers count as creators (≈ 0.8–0.95). |
| **relevance** | `0.6` if you follow the author<br>`+ 0.3` per tagged fragrance you own or want<br>`+ 0.4 ×` overlap between the post's accords and your taste profile<br>capped at `2.0` | Personalisation from what you own, want and like the smell of. |
| **age decay** | `(ageHours + 2) ^ 1.15` | Keeps the feed fresh. The `+2` stops brand-new posts from getting near-infinite scores. |

After scoring, a **diversification pass** reorders posts so one author can't take several slots in a row.

## Trending score
```
trending = engagement / (ageHours + 2) ^ 1.6     (only posts from the last 72 hours)
```
A steeper decay than For You, so Trending reflects *right now*.

## Why this design
Product goal: the Home feed should favour popular creators, like Instagram, because fragrance communities grow around reviewers, without becoming a popularity contest.

| Option | Pros | Cons | Decision |
|---|---|---|---|
| Pure engagement with time decay (Reddit/Hacker News) | Simple, proven | New users get buried; it's a popularity contest | ✗ |
| Chronological only | Fair and predictable | Bad discovery; nothing to do until you follow people | Kept as the **Following** tab |
| Learned ranker (two-tower or gradient-boosted model on dwell time and likes) | Best possible relevance | Needs lots of interaction data that a new app doesn't have | Planned |
| **Hand-tuned heuristic with a log-capped authority boost** | Works from day one; easy to explain and tune | Weights are hand-picked | ✓ Chosen |

**Why a log-scaled multiplier for authority?** A 50k-follower account gets roughly 2.4× the boost of a brand-new one, not 50,000×. A strong post from a 300-follower friend can still beat a weak post from a big creator.

## Scaling plan
- Move scoring server-side into a Postgres function `feed_for_you(viewer, cursor)` that scores posts from the last 7 days.
- Feed it from a materialized `post_engagement` table and a `user_taste` accord vector, refreshed by cron.
- Keyset pagination on `(score, id)`.
- Trending as a materialized view refreshed every few minutes.
- Log impressions and dwell time from launch (`feed_impressions`) to build training data for a learned ranker.
