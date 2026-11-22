# Scented

**A social platform for fragrance lovers — "Letterboxd for perfume."**
Log the bottles you own, review and rate fragrances, vote on how they really smell and wear, build lists, and talk fragrance with other people in an Instagram-style feed.

Runs on **iOS, Android and the web** from one Flutter codebase, backed by Supabase (Postgres, Auth, Storage, Edge Functions).
Live site: **https://scentedapp.app**

Designed and built by **Nam Dinh**.

---

## Documentation
| Doc | What's in it |
|---|---|
| [Development log](docs/DEVELOPMENT_LOG.md) | Every phase of the build, start to finish |
| [Architecture](docs/ARCHITECTURE.md) | App layers, backend, push pipeline, responsive layout |
| [Feed algorithm](docs/FEED_ALGORITHM.md) | For You and Trending ranking and the reasoning behind them |
| [Community systems](docs/COMMUNITY_SYSTEMS.md) | Voting, submissions, verification, moderation, DMs |
| [Database](docs/DATABASE.md) | Tables, functions, migration history, catalog import |
| [Security & privacy](docs/SECURITY.md) | RLS, grants, abuse limits, encryption |
| [Infrastructure](docs/INFRASTRUCTURE.md) | Domain, hosting, email, push setup |
| [Decision log](docs/DECISIONS.md) | 24 decisions with the alternatives considered |
| [Credits](CREDITS.md) | Data licensing and attribution |

---

## Contents
1. [What the app does](#what-the-app-does)
2. [Tech stack](#tech-stack)
3. [How I built it — the process](#how-i-built-it--the-process)
4. [Architecture](#architecture)
5. [Engineering highlights](#engineering-highlights)
6. [Database & security design](#database--security-design)
7. [Product decisions and trade-offs](#product-decisions-and-trade-offs)
8. [Infrastructure, domain & email](#infrastructure-domain--email)
9. [Data licensing](#data-licensing)
10. [Project structure](#project-structure)
11. [Running it locally](#running-it-locally)
12. [What I learned](#what-i-learned)
13. [Roadmap](#roadmap)

---

## What the app does

| Tab | What's there |
|---|---|
| **Home** | A social feed with three modes: **For You** (ranked by my own algorithm), **Following** (chronological) and **Trending** (fastest-growing posts in the last 72 hours). Post types: discussion, question, debate (a fragrance-vs-fragrance poll), recommendation, new release, collection and review. Up to 4 photos and tagged fragrances per post. Likes, comments, comment likes and saved posts. |
| **Discover** | Search and browse a real catalog of fragrances, houses and notes (fuzzy, typo-tolerant search). |
| **Collection** | Your shelf of owned bottles plus your own lists. The collection and each list can be **public, followers-only or private**. |
| **Profile** | Avatar (with crop), bio, followers/following, stats derived from your shelf and reviews, your posts and lists. |
| **Settings** | Instagram-style settings: account & security, rate-limited username changes, light/dark appearance, notification preferences, blocked accounts, about/credits. |

Every **fragrance page** works like a community wiki decided by votes:
- **Notes people actually smell** (one vote per note per person)
- **"Smells like"** comparisons between fragrances (agree / disagree)
- **Performance** — longevity (1–5) and sillage (1–4)
- **When to wear** — seasons and day/night
- Your own votes are shown when you open a fragrance from your profile

Also included:
- **Direct messages** — 1:1 chats with Instagram-style message requests, encrypted at rest
- **Notifications** — in-app and phone push for follows, likes, replies and submission decisions
- **Community catalog submissions** — anyone can propose a missing fragrance; it goes through a duplicate check and a pending queue reviewed by trusted members. Verified accounts (badge granted manually) can add fragrances directly.
- **Moderation** — report posts, comments, users and conversations; block users; an admin queue to dismiss, remove content or ban
- **Guest browsing on the web** — signed-out visitors can read everything public (like logged-out X); an account is needed to post, like, follow or reply
- **Responsive web layout** — a side navigation rail and centered content columns on laptops instead of a stretched phone layout

---

## Tech stack

| Layer | Technology |
|---|---|
| App (iOS / Android / Web) | **Flutter**, **Dart** |
| Backend | **Supabase** — PostgreSQL, Auth, Storage, Row Level Security, Vault |
| Server logic | **PL/pgSQL** functions & triggers, **Supabase Edge Function** (TypeScript / Deno) |
| Push notifications | **Firebase Cloud Messaging** (HTTP v1 API) |
| Web hosting & DNS | **Cloudflare Pages** + Cloudflare DNS, domain from **Porkbun** |
| Transactional email | **Brevo** SMTP, custom HTML templates, sender `invite@scentedapp.app` |
| Catalog data | **Scento Perfume Database** (CC BY 4.0) |

Dependencies were kept deliberately small: `supabase_flutter`, `image_picker`, `firebase_core` / `firebase_messaging`, and Flutter's own UI library. State management, navigation and the design system are hand-written.

---

## How I built it — the process

I built Scented incrementally: each step produced a working app before the next one started, and I wrote down the major architectural decisions with the alternatives I considered (see [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)). A fuller write-up of each phase is in [`docs/DEVELOPMENT_LOG.md`](docs/DEVELOPMENT_LOG.md).

### 1. Concept and product definition
- Started from the idea of "Letterboxd, but for fragrances" and set the goal of a polished, commercially viable iOS/Android app.
- Defined the core loop: **own → wear → review → discuss**.
- Chose Flutter (one codebase for iOS, Android and web) and Supabase (Postgres with real relational modelling, auth and storage, without running my own servers).

### 2. Increment 1 — UI prototype on mock data
- Built the app shell, design system (warm charcoal/ivory palette with a single amber accent, typography and spacing scales) and reusable widgets: fragrance posters, rating stars, meters, pills, avatars, list cards, bottom sheets.
- Put all data access behind a `ScentedRepository` interface with an **in-memory implementation** and seed data, so the whole UI could be designed and tested before a backend existed.
- Seed data uses a fictional house ("Maison Aurel") so no real brand is credited with an invented product.

### 3. Increment 2 — Rethinking the product: from diary to social feed
- The first version had a wear-diary tab. After using it, I **removed the diary** and replaced it with an Instagram-style **social Home feed**, because discussion was more engaging than logging.
- Reworked navigation to **Home · Discover · Collection · Profile · Settings** (Settings replaced the Watchlist tab and moved to the far right).
- Designed and implemented the **For You ranking algorithm** (details below) as pure, unit-testable Dart.

### 4. Increment 3 — Real backend (Supabase)
- Wrote the full Postgres schema as versioned SQL migrations: profiles, follows, reviews, collections, lists, posts, media, polls, likes, comments.
- Locked every table down with **Row Level Security** and column-level grants.
- Used **database triggers** for counters (followers, likes, comments) so clients can't fake them.
- Set up Storage buckets for avatars and post photos with per-user folder policies.
- Implemented `SupabaseScentedRepository` and real sign-up / sign-in, swapping it in for the mock repository without touching the UI.

### 5. Increment 4 — A real fragrance catalog
- Looked for a fragrance data source I was allowed to use. My first choice wasn't viable, so I switched to the **free, openly licensed Scento Perfume Database**.
- Wrote a **re-runnable SQL import script** that fetches the dataset straight from GitHub (pinned to one commit, for reproducibility) using `pg_net`, parses JSONL and CSV inside Postgres, and upserts houses and fragrances.
- Filled in missing release years, mapped "character" tags to accords, and added **trigram indexes** for fast fuzzy search.
- Added visible in-app credit on every fragrance page and Discover, plus [`CREDITS.md`](CREDITS.md).

### 6. Increment 5 — Community features
- **Consensus voting** for notes, "smells like" pairs and wear performance, with private raw votes and public aggregates.
- **Fragrance submissions** with a pending queue, duplicate detection, trusted-reviewer voting and a manually granted **verified badge**.
- **Rate-limited username changes** (Instagram-style).
- **Audience controls** (public / followers / private) for collections and lists, enforced in the database.

### 7. Increment 6 — Website version
- Made the web build feel like a real website on a laptop: breakpoints, a navigation rail at ≥ 840 px (with labels at ≥ 1200 px), and max content widths per page type (reading, detail, browsing grids).
- Opened read access to signed-out guests while keeping every write authenticated-only.
- Deployed to **Cloudflare Pages** with cache headers that make new releases show up without a hard refresh.

### 8. Increment 7 — Safety, notifications and messaging
- **Moderation**: reports, blocks, admin actions (dismiss / remove / ban), report-a-conversation. This is also required for App Store approval of apps with user-generated content.
- **Notifications**: in-app inbox plus phone push through a Supabase Edge Function that talks to Firebase Cloud Messaging.
- **Direct messages** with message requests and encryption at rest.
- Smaller polish: comment likes, saved posts, letting post authors delete comments on their own posts.

### 9. Launch infrastructure
- Bought the domain **scentedapp.app**, moved DNS to Cloudflare, and pointed it at Cloudflare Pages.
- Set up custom SMTP through **Brevo** so auth emails come from `invite@scentedapp.app`, and designed branded HTML email templates (confirm sign-up, reset password, change email, magic link, invite) plus a "you're confirmed" landing page.

---

## Architecture

```
┌──────────────────────── Flutter app (iOS · Android · Web) ────────────────────────┐
│  features/   screens per area (home, discover, fragrance, collection, profile…)   │
│  shared/     design-system widgets                                                 │
│  state/      AppState (ChangeNotifier) + AppScope (InheritedNotifier)              │
│  domain/     FeedRanker, UserStats — pure Dart, no Flutter imports                 │
│  data/       ScentedRepository interface                                           │
│                ├─ InMemoryScentedRepository (mock / tests)                         │
│                └─ SupabaseScentedRepository (production)                           │
└───────────────────────────────────────┬───────────────────────────────────────────┘
                                        │ HTTPS (PostgREST, Auth, Storage, RPC)
┌───────────────────────────────────────▼───────────────────────────────────────────┐
│ Supabase                                                                           │
│  Postgres: tables + RLS + triggers + SECURITY DEFINER RPCs (16 migrations)         │
│  Auth (email) · Storage (avatars, post-media) · Vault (DM key)                     │
│  Edge Function "push" ──► Firebase Cloud Messaging ──► phones                      │
└────────────────────────────────────────────────────────────────────────────────────┘
       Web build hosted on Cloudflare Pages · Auth email via Brevo SMTP
```

**Key choices**
- **Repository pattern** — the UI only knows the `ScentedRepository` interface, which let me build the whole UI on mock data first and swap in Supabase later.
- **Dependency-free state** — `ChangeNotifier` + `InheritedNotifier` instead of a state-management package, to keep the app small and easy to follow (Riverpod is a planned upgrade for caching and pagination).
- **Business rules live in the database** — limits, permissions and counters are enforced by Postgres, not trusted to the client.

---

## Engineering highlights

### Feed ranking algorithm (`lib/domain/feed_ranker.dart`)
```
ForYou   = (1 + ln(1 + engagement)) × (1 + 1.5·authority) × (1 + relevance) / (ageHours + 2)^1.15
  engagement = likes + 3·comments + 1.5·votes
  authority  = ln(1 + followers) / ln(100 001)          → 0–1
  relevance  = 0.6 if you follow the author
             + 0.3 per tagged fragrance you own or want
             + 0.4 × accord overlap with your taste profile   (capped at 2.0)
Trending = engagement / (ageHours + 2)^1.6, last 72 hours only
```
- A **log-scaled, capped authority boost** favours popular creators without letting a 50k-follower account bury a strong post from a small account.
- A **diversification pass** stops one author from filling consecutive slots.
- I compared this against pure engagement ranking (Reddit/HN style, which buries new users) and a learned ranker (the right end state, but it needs interaction data first). The plan is to log impressions and dwell time from launch and train a model later.

### Consensus voting without brigading
- Primary keys enforce **one vote per person per item**.
- Raw votes are **private** (RLS: you can only read your own); everyone reads **aggregates** through one `fragrance_consensus()` function that returns JSON, so individual voters can't be singled out or targeted.
- "A smells like B" is stored **once, in a canonical order** (byte-order collation so the Dart client and Postgres agree), so the same vote appears on both fragrance pages.
- A trigger caps note votes at 15 per person per fragrance to stop spam.

### Direct messages encrypted at rest
- Message bodies are encrypted with a key stored in **Supabase Vault**, so table browsing, dumps and backups only show ciphertext.
- Only the two members can read a conversation (RLS plus member-checked functions).
- I chose this over end-to-end encryption on purpose: it lets chats sync across phone and web and lets a reported conversation be reviewed by moderators — a trade-off I documented in the migration.

### Push notifications from scratch
- A database trigger calls a **Supabase Edge Function** (TypeScript/Deno) for each new notification.
- The function **signs its own Google OAuth JWT (RS256)** with the Web Crypto API from a service-account key, caches the access token, and calls the **FCM HTTP v1 API** — no extra SDK.
- It only pushes rows that are fresh and not yet pushed, so it's safe to call without a shared secret and never double-sends.

### Rate-limited username changes
- The `username` column is **not updatable by clients**; the only way to change it is a `change_username` function that allows **2 changes per rolling 14 days** and logs history, so the limit can't be bypassed.

### Responsive web from the same codebase
- `Breakpoints` and `ContentWidth` constants plus a `CenteredContent` widget that constrains only the content column (and narrows `MediaQuery` so child layouts size correctly), keeping app bars and transitions full-width.

### Performance details
- Images are downscaled when picked (max 2048 px, quality 85) and decoded at display size.
- GIN indexes on tagged fragrances and trigram indexes on names for search.

---

## Database & security design

16 versioned migrations in `supabase/migrations`:

| Migration | What it adds |
|---|---|
| `initial_schema` | profiles + sign-up trigger, follows, reviews, collection, lists, posts, media, polls, likes, comments, storage buckets |
| `poll_votes_post_fk` | integrity fix for poll votes |
| `fragrance_catalog` | houses, fragrances, materials; trigram search; foreign keys from user data to the catalog |
| `community_consensus` | note / similar / wear votes and the aggregate function |
| `verified_submissions_user_votes` | verified flag, trusted reviewers, fragrance submission queue |
| `username_changes` | rate-limited username changes with history |
| `collection_list_visibility` | public / followers / private audiences |
| `public_read_for_guests` | read-only access for signed-out web visitors |
| `moderation` | admins, blocks, reports, admin actions |
| `notifications` | notification inbox, push trigger |
| `comment_likes`, `saved_posts` | engagement features |
| `decide_submission` | verified members and admins approve/edit submissions directly |
| `direct_messages` | conversations, requests, Vault-encrypted messages |
| `report_conversations`, `post_authors_delete_comments` | moderation polish |

**Security principles applied throughout**
- RLS on every table; reads and writes restricted to `auth.uid()` where appropriate.
- **Column-level grants** so clients can update only safe columns (e.g. not `username`, `is_verified`, `is_admin` or counters).
- `SECURITY DEFINER` functions with `search_path = ''` and explicit `revoke … from public` to avoid privilege-escalation pitfalls.
- **Trusted reviewer** status is earned (account 14+ days old with 5+ reviews) or granted (verified); admin and verified flags can only be set by hand.
- The app ships only the publishable key — RLS is what protects the data.

---

## Product decisions and trade-offs

| Decision | Why |
|---|---|
| Removed the wear diary | The social feed was more engaging; stats now come from the shelf and reviews. |
| Settings tab instead of Watchlist | Settings needed to be reachable everywhere; wants are part of Collection. |
| Algorithmic home feed favouring creators | Fragrance communities grow around reviewers; the boost is capped to stay fair to small accounts. |
| Wiki-by-vote fragrance pages | Official note pyramids often differ from what people smell; the crowd decides. |
| Pending queue + verified badge for new fragrances | Keeps the catalog clean without blocking trusted contributors. |
| Per-collection and per-list privacy | People are sensitive about how much they've spent on bottles. |
| Guest browsing on the web | Lets shared links work for anyone, while still requiring an account to participate. |
| Encryption at rest instead of E2E for DMs | Multi-device sync and moderation of reported chats. |
| Free-tier services first | The app is free; hosting and email were chosen to cost nothing at the start without limiting growth. |

---

## Infrastructure, domain & email
- **Domain:** `scentedapp.app` (Porkbun), DNS on Cloudflare.
- **Web hosting:** Cloudflare Pages; `web/_headers` sets `no-cache` on Flutter's un-fingerprinted entry files so users get new releases immediately.
- **Auth email:** Supabase Auth with custom SMTP through Brevo, sender `invite@scentedapp.app`. Templates are in `supabase/email-templates` — intentionally light-themed and image-light because many mail apps force-invert dark emails.
- **Email confirmation landing page:** `web/confirmed/index.html`.

## Data licensing
Fragrance catalog data comes from the **Scento Perfume Database** (CC BY 4.0), pinned to a specific commit, with attribution shown in the app. Full details and a list of changes made are in [`CREDITS.md`](CREDITS.md).

---

## Project structure
```
lib/
  core/        config, responsive layout, navigation, push, theme, utils
  data/        auth, catalog mapping, DMs, models, repositories (in-memory + Supabase), mock seed
  domain/      feed ranking, user stats (pure Dart)
  features/    activity, auth, collection, discover, fragrance (+ community voting), home,
               lists, messages, moderation, notifications, people, profile, settings,
               shell, submissions
  shared/      design-system widgets
  state/       app state, DM controller, theme controller
supabase/
  migrations/      16 SQL migrations
  functions/push/  Edge Function for FCM push
  scripts/         re-runnable Scento catalog import
  email-templates/ branded auth emails
web/           web entry, PWA manifest/icons, Cloudflare headers, confirmation page
docs/          detailed documentation (see the table above)
```

## Running it locally
```bash
flutter pub get
flutter run              # phone / simulator
flutter run -d chrome    # web
flutter build web        # output in build/web, deployed to Cloudflare Pages
```
Backend: apply `supabase/migrations` in order to a Supabase project, run `supabase/scripts/import_scento_catalog.sql` (two steps, see the file), deploy `supabase/functions/push` with the `FIREBASE_SERVICE_ACCOUNT` secret, and add the Firebase config files for push on iOS/Android.

---

## What I learned
- **Designing a relational schema for a social product** — follows, feeds, polls, votes, privacy audiences and moderation, and how they interact.
- **Security as a database concern** — RLS, column grants, `SECURITY DEFINER` hygiene, and moving every rule that matters off the client.
- **Ranking systems** — balancing engagement, recency, authority and personal relevance, and knowing when a heuristic beats a model.
- **Cross-platform UI** — one Flutter codebase that feels native on phones and like a real website on laptops.
- **Shipping real infrastructure** — domains, DNS, static hosting and caching, SMTP and email deliverability, push notifications, and secrets.
- **Product iteration** — cutting a feature I'd already built (the diary) when the product was better without it.
- **Open data licensing** — finding a dataset I'm allowed to use and crediting it correctly.

## Roadmap
- Riverpod for async caching and paginated feeds; server-side `feed_for_you` ranking in Postgres
- go_router deep links and share URLs for posts and reviews
- Impression and dwell-time logging to train a learned ranker
- Licensed bottle images
- App Store and Google Play release

---
© Nam Dinh. Fragrance data © Scento, CC BY 4.0.
