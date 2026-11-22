# Architecture

## Overview
```
┌──────────────────────── Flutter app (iOS · Android · Web) ────────────────────────┐
│  features/   one folder per area: home, discover, fragrance, collection, profile, │
│              settings, messages, notifications, moderation, submissions, …        │
│  shared/     design-system widgets                                                 │
│  state/      AppState, DmController, ThemeController (ChangeNotifier)              │
│              AppScope (InheritedNotifier) exposes them to the widget tree          │
│  domain/     FeedRanker, UserStats: pure Dart, no Flutter imports                  │
│  data/       ScentedRepository (interface)                                         │
│                ├─ InMemoryScentedRepository: mock data for prototyping and tests   │
│                └─ SupabaseScentedRepository: production                            │
│              AuthRepository, DmRepository, CatalogMapper                           │
│  core/       config, responsive layout, navigation, push, theme, utils             │
└───────────────────────────────────────┬───────────────────────────────────────────┘
                                        │ HTTPS: PostgREST · Auth · Storage · RPC · Realtime
┌───────────────────────────────────────▼───────────────────────────────────────────┐
│ Supabase                                                                           │
│  PostgreSQL ─ tables, RLS policies, triggers, SECURITY DEFINER functions           │
│  Auth ─ email + password, custom SMTP (Brevo)                                      │
│  Storage ─ avatars/, post-media/ (per-user folders)                                │
│  Vault ─ DM encryption key                                                         │
│  Realtime ─ new messages, membership changes                                       │
│  Edge Function "push" ──► Firebase Cloud Messaging ──► iOS / Android               │
└────────────────────────────────────────────────────────────────────────────────────┘
  Web build → Cloudflare Pages (scentedapp.app) · DNS on Cloudflare · domain from Porkbun
```

## Layers

### Presentation (`features/`, `shared/`)
Each feature folder holds its screens, bottom sheets and widgets. Shared widgets make up a small design system, so every screen uses the same posters, ratings, meters and sheets.

### State (`state/`)
`ChangeNotifier` classes exposed through an `InheritedNotifier` (`AppScope`). I chose this over Riverpod or Bloc to avoid dependencies while the app was small and to keep data flow easy to follow. Riverpod is planned once paginated, cached server data makes it worth adding.

### Domain (`domain/`)
Pure Dart logic with no Flutter or Supabase imports, so it can be unit-tested on its own:
- `FeedRanker`: For You and Trending scoring plus author diversification ([FEED_ALGORITHM.md](FEED_ALGORITHM.md))
- `UserStats`: profile statistics computed from the shelf and reviews

### Data (`data/`)
The UI only depends on the `ScentedRepository` interface. That let me:
1. build and polish the whole UI on mock data first,
2. switch to Supabase by changing one line in `main.dart`,
3. keep an in-memory implementation for tests.

`CatalogMapper` turns catalog rows into app models (accords from character tags, poster colours from the scent family).

## Backend design principle: rules live in the database
Anything that matters for security or fairness is enforced by Postgres, never trusted to the client:
- **Who can read or write** → Row Level Security policies and column-level grants
- **Counters** (followers, likes, comments) → triggers
- **Limits** (username changes, submissions a day, note votes, DM rate) → checks inside `SECURITY DEFINER` functions
- **Multi-step actions** (submit, review, publish a fragrance, send a DM) → one RPC call each, so they happen all at once or not at all

More detail: [DATABASE.md](DATABASE.md) and [SECURITY.md](SECURITY.md).

## Push notifications
```
notifications / messages row inserted
        │  trigger (pg_net.http_post)
        ▼
Edge Function "push" (TypeScript, Deno)
  1. claims the row: sets pushed_at only if it's < 2 min old and not yet pushed
  2. builds the title and body (sender, snippet, request vs chat)
  3. signs an RS256 JWT with the Firebase service account (Web Crypto API)
  4. exchanges it for a Google OAuth access token (cached ~1 h)
  5. sends to every device token through the FCM HTTP v1 API, grouping a chat's
     messages into one thread (Android tag / iOS thread-id)
  6. deletes tokens that FCM reports as unregistered
        ▼
Firebase Cloud Messaging → phone
```
Because a row can only be claimed once, and only while fresh, the function needs no caller secret, never double-sends, and ignores replayed requests. Users' per-type settings, "pause all" and muted chats are checked before anything is sent.

## Responsive layout
`core/layout/responsive.dart`:
- `Breakpoints.rail = 840` → bottom bar becomes a side navigation rail
- `Breakpoints.wideRail = 1200` → rail shows labels
- `ContentWidth`: `auth 440`, `reading 640`, `detail 820`, `browse 1120`
- `CenteredContent` centers only the content column and narrows `MediaQuery.size`, so widgets that size themselves from the screen width lay out for the column instead of the whole window.

## Media
- Photos are picked with the system picker (no Android permission prompt), downscaled at pick time (max 2048 px, JPEG quality 85) and decoded at display size with `cacheWidth`.
- Uploaded to `post-media/<uid>/<postId>/<n>.jpg` before the post row is inserted. Storage policies only allow writes inside the user's own `<uid>/` folder.

## Navigation
Navigator 1.0 behind an `AppNavigator` helper. Moving to `go_router` is planned for deep links to posts, reviews and profiles.

## Dependencies (runtime)
| Package | Why |
|---|---|
| `supabase_flutter` | Auth, database, storage, realtime |
| `image_picker` | System photo picker |
| `firebase_core`, `firebase_messaging` | Phone push |
| `material_ui`, `cupertino_icons` | Flutter UI library and icons |

Everything else (state management, design system, ranking, layout) is written by hand.
