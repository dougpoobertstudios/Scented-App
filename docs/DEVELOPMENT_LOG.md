# Development Log

A phase-by-phase record of how I designed and built Scented, from idea to a live app on iOS, Android and the web. Backend work in November 2026 is dated from the migration files.

---

## Phase 0 — Idea and goals
**Problem:** Fragrance fans track collections in spreadsheets, read reviews on old forums, and argue about notes in comment sections. There is no modern, social, mobile-first home for the hobby — nothing like what Letterboxd is for film.

**Goal:** A polished app that could be released commercially on the App Store and Google Play, plus a website that works the same way.

**Constraints I set for myself**
- One codebase for iOS, Android and web.
- Small, modular code with as few third-party packages as possible.
- Work in small increments, each one ending with a working app.
- Before any big architectural decision, write down the alternatives and why I picked one.
- Keep running costs at zero while the app is free, without picking services that would block growth later.

**Stack chosen:** Flutter/Dart (cross-platform UI), Supabase (managed Postgres, Auth, Storage, Edge Functions).

---

## Phase 1 — UI prototype on mock data
- Created the Flutter project and a feature-first folder structure (`core`, `data`, `domain`, `features`, `shared`, `state`).
- Designed the visual identity: dark warm-charcoal surfaces, ivory text and one amber accent, with typography and spacing scales in `core/theme`.
- Built ~16 reusable design-system widgets: fragrance posters, rated posters, star ratings, performance meters, pills, follow button, verified badge, avatars, list cards, section headers, a shared bottom-sheet scaffold and a press-feedback wrapper.
- Defined the data models: `Fragrance`, `AppUser`, `Review`, `FragranceList`, `Post` (with media, poll options and comments).
- Put every read and write behind a `ScentedRepository` interface and wrote an `InMemoryScentedRepository` with seed data, so the whole app could be clicked through before any backend existed.
- Used a fictional brand ("Maison Aurel") in seed data so no real company was shown with made-up products.

**Outcome:** a fully navigable app to test the product idea quickly.

---

## Phase 2 — Changing the product: from diary to social feed
The first version had a **diary** for logging which fragrance you wore each day. Using it showed that logging was a chore, while talking about fragrances was what people actually enjoy.

Changes:
- **Removed the diary** entirely (screens, models and stats). Stats now come from your shelf and reviews.
- Built an **Instagram-style Home feed** with three tabs: For You, Following and Trending.
- Added seven post types: discussion, question, debate (fragrance-vs-fragrance poll), recommendation, new release, collection, and review (a review becomes a post when shared).
- Added a post composer with up to 4 photos (via the system photo picker, no extra Android permissions) and fragrance tagging.
- Wrote the **For You ranking algorithm** in pure Dart (`domain/feed_ranker.dart`) — see [FEED_ALGORITHM.md](FEED_ALGORITHM.md).
- Reorganised navigation to **Home · Discover · Collection · Profile · Settings**. Settings replaced the Watchlist tab and sits on the far right; wanted bottles moved into Collection.

---

## Phase 3 — Real backend on Supabase (16 Nov 2026)
- Wrote the initial schema as a SQL migration: `profiles`, `follows`, `reviews`, `collection_items`, `lists`, `list_items`, `posts`, `post_media`, `poll_options`, `poll_votes`, `post_likes`, `comments`.
- A sign-up trigger (`handle_new_user`) creates a profile automatically, with a validated username and a random avatar colour.
- Counters (followers, following, likes, comments) are kept by **triggers**, so clients can't fake them.
- **Row Level Security** on every table, plus **column-level grants** so users can only edit safe fields.
- Storage buckets `avatars` (5 MB) and `post-media` (10 MB), where each user can only write inside their own folder.
- Built `SupabaseScentedRepository` and an auth layer (`AuthRepository`, `AuthScope`), plus welcome, sign-in and sign-up screens with live username-availability checks.
- Swapped the mock repository for the Supabase one in `main.dart`. The UI didn't change, which confirmed the repository design worked.
- Follow-up fix: added a missing foreign key between poll votes and posts.

---

## Phase 4 — A real fragrance catalog (16 Nov 2026)
- Needed a large catalog I was legally allowed to use. My first choice wasn't available, so I picked the **Scento Perfume Database**, released under CC BY 4.0.
- Created `houses`, `fragrances` and `materials` tables, readable by everyone and writable by no client.
- Wrote `supabase/scripts/import_scento_catalog.sql`, a **re-runnable import that runs entirely inside Postgres**:
  1. `pg_net` downloads the dataset's JSONL and CSV files from GitHub, pinned to one commit so imports are reproducible.
  2. The script splits and parses the files in SQL (including a regex-based CSV parser that handles quoted fields) and upserts houses and fragrances.
- Cleaned the data: filled missing release years from each perfume's page (a few from Wikidata), turned "character" tags into accords with strengths based on their order, and derived poster colours from the scent family.
- Added **trigram (pg_trgm) indexes** on fragrance and house names for fast, typo-tolerant search, plus GIN indexes on notes.
- Linked user data (reviews, collections, list items, poll options) to the catalog with foreign keys.
- Added visible credit in the app (every fragrance page and Discover) and a `CREDITS.md`.

---

## Phase 5 — Community systems (18 Nov 2026)
Details in [COMMUNITY_SYSTEMS.md](COMMUNITY_SYSTEMS.md).
- **Wiki-style consensus voting** on every fragrance: notes people smell, "smells like" pairs, longevity, sillage, seasons and time of day. Raw votes are private; everyone sees totals.
- **Your own votes on your profile:** opening a fragrance from someone's profile shows how *they* voted.
- **Fragrance submissions:** anyone can add a missing fragrance. It is checked for duplicates, limited to 3 a day, and approved after 3 trusted votes. Verified members publish straight away.
- **Verified badge:** a flag only I can set from the database.
- **Username changes:** at most 2 per rolling 14 days, with history, like Instagram.
- **Privacy audiences:** the collection and each list can be public, followers-only or private, enforced by RLS.
- New screens: note and wear voting sheets, add-fragrance form, submissions queue and review screen, username settings, audience picker.

---

## Phase 6 — Website version (19 Nov 2026)
- Made the Flutter web build feel like a real website instead of a stretched phone:
  - Breakpoints: a **side navigation rail** from 840 px, with labels from 1200 px.
  - Max content widths per page type: 440 px (auth), 640 px (feeds and forms), 820 px (fragrance pages), 1120 px (poster grids).
  - A `CenteredContent` widget that narrows only the content column and its `MediaQuery`, so app bars and page transitions stay full width.
- **Guest browsing:** signed-out visitors can read public posts, profiles, reviews, lists and the catalog, like logged-out X. Any action that writes (post, like, follow, reply) shows a join prompt. Implemented by granting read-only access to the `anon` role on the right tables; the privacy rules still apply.
- PWA manifest, icons and favicon.
- Deployed to **Cloudflare Pages** with a `_headers` file that disables caching on Flutter's un-fingerprinted entry files, so new releases appear without a hard refresh.

---

## Phase 7 — Safety, notifications and messaging (19–20 Nov 2026)
**Moderation** (also needed for App Store approval of apps with user-generated content)
- Report posts, comments, accounts and conversations; block accounts; a blocked-accounts screen.
- Admin role (set by hand) with a moderation queue to dismiss reports, remove content or ban accounts.

**Notifications**
- An in-app notifications inbox for follows, likes, replies and submission decisions, with per-type settings and "pause all".
- **Phone push:** a database trigger calls a Supabase Edge Function that signs its own Google OAuth token and sends through Firebase Cloud Messaging. See [ARCHITECTURE.md](ARCHITECTURE.md#push-notifications).

**Direct messages**
- 1:1 chats with **message requests** (people you don't follow land in Requests), read receipts only after accepting, mute, delete-for-me and unsend.
- Message bodies **encrypted at rest** with a key in Supabase Vault; realtime delivery; rate limits (30 messages a minute, 5 before a request is accepted).
- Reporting a conversation to moderators.

**Engagement polish**
- Comment likes, saved posts, and letting post authors delete comments on their own posts.
- Activity screen, people search, follower/following lists, avatar crop, profile editing, light/dark appearance setting.

---

## Phase 8 — Domain, email and launch setup (Nov 2026)
- Bought **scentedapp.app** from Porkbun and moved DNS to Cloudflare.
- Connected the domain to Cloudflare Pages.
- Set up **custom SMTP through Brevo** so auth emails come from `invite@scentedapp.app` (I compared it with Resend and kept Brevo).
- Designed five branded HTML auth emails (confirm sign-up, change email, reset password, magic link, invite). They use a light layout because many mail apps force-invert dark emails.
- Built a "you're all set" confirmation page at `scentedapp.app/confirmed`, where email links land.
- Configured redirect URLs in Supabase Auth.
- See [INFRASTRUCTURE.md](INFRASTRUCTURE.md).

---

## By the numbers
- 3 platforms (iOS, Android, Web) from 1 codebase
- 16 versioned database migrations
- About 30 tables and dozens of `SECURITY DEFINER` RPC functions, all behind Row Level Security
- 16 feature areas and 40+ screens and sheets
- 1 Edge Function, 5 email templates, 1 re-runnable data import
- 6 runtime dependencies
