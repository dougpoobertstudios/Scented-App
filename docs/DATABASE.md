# Database

PostgreSQL on Supabase, defined in 16 versioned SQL migrations. Every table has Row Level Security turned on.

## Tables by area

### Accounts and social graph
| Table | Purpose | Notable rules |
|---|---|---|
| `profiles` | One row per user (linked to `auth.users`) | Username `^[a-z0-9._]{3,24}$`, unique ignoring case; display name 1–50 characters; bio ≤ 160. `follower_count` and `following_count` are kept by triggers. `is_verified`, `is_admin` and `collection_visibility` live here. |
| `follows` | Who follows whom | No self-follow; blocked pairs can't follow |
| `username_changes` | History of username changes | Drives the 2-per-14-days limit |
| `blocks` | Who blocked whom | Blocking removes follows both ways |

### Catalog (read-only to clients)
| Table | Purpose |
|---|---|
| `houses` | Perfume houses (country, year founded, type) |
| `fragrances` | Name, house, year, perfumers, concentration, families, character, seasons, time of day, top/heart/base notes, popularity, source (`scento` or `community`) |
| `materials` | Raw materials and notes with aliases and smell descriptions |

Indexes: trigram (`pg_trgm`) on fragrance and house names for typo-tolerant search, GIN on notes, b-tree on house and popularity.

### User content
| Table | Purpose | Notable rules |
|---|---|---|
| `reviews` | Rating and text per user per fragrance | Ratings 0.5–5 in half steps (`rating * 2 = floor(rating * 2)`); one review per user per fragrance; body ≤ 5000 |
| `collection_items` | Owned bottles and wants | `status in ('owned','watchlist')`; visibility follows the owner's setting |
| `lists`, `list_items` | Ordered user lists | Per-list visibility |
| `posts` | Feed posts | 7 types; body ≤ 2000; `fragrance_ids text[]` with a GIN index; like and comment counts kept by triggers |
| `post_media` | Up to 4 photos per post | Only the post's author can attach |
| `poll_options`, `poll_votes` | Fragrance-vs-fragrance debates | Up to 4 options; one vote per user (composite key) |
| `post_likes`, `comments`, `comment_likes` | Engagement | Comments 1–1000 characters |
| `saved_posts` | Bookmarks | Private to the user |

### Community consensus
| Table | Purpose |
|---|---|
| `fragrance_note_votes` | Notes each person smells (≤ 15 per person per fragrance) |
| `fragrance_similar_votes` | "A smells like B", agree or disagree, stored once per ordered pair |
| `fragrance_wear_votes` | Longevity 1–5, sillage 1–4, seasons, day/night |
| `fragrance_submissions`, `fragrance_submission_votes` | Community-added fragrances and trusted-member reviews |

### Safety, notifications, messaging
| Table | Purpose |
|---|---|
| `reports` | Reports with reason, details, content snapshot, status and admin action |
| `notifications` | Follow, like, reply, submission approved or rejected; `read_at`, `pushed_at` |
| `notification_settings` | Per-type toggles, "pause all", messages |
| `push_tokens` | Device tokens for FCM |
| `conversations`, `conversation_members`, `messages` | 1:1 DMs; `messages.body_enc` is `bytea` ciphertext |

## Storage
| Bucket | Limit | Types | Path rule |
|---|---|---|---|
| `avatars` | 5 MB | jpeg, png, webp, heic | `<uid>/…` only |
| `post-media` | 10 MB | jpeg, png, webp, heic | `<uid>/<postId>/<n>.jpg` |

## Main functions (RPC)
| Function | What it does |
|---|---|
| `handle_new_user` (trigger) | Creates a profile on sign-up |
| `username_available`, `change_username`, `username_change_status` | Usernames |
| `fragrance_consensus`, `user_fragrance_votes` | Vote totals and one person's votes |
| `submit_fragrance`, `review_submission`, `decide_submission`, `publish_submission` (internal) | Catalog submissions |
| `can_view_audience` | Visibility check used by RLS |
| `is_blocked_between`, `moderate_report`, `is_admin` | Moderation |
| `open_conversation`, `send_message`, `list_conversations`, `get_messages`, `accept_conversation`, `mark_conversation_read`, `set_conversation_muted`, `delete_conversation`, `unsend_message` | Direct messages |
| `message_push_payload` (service role only) | Claims a message for push |
| `sync_follow_counts`, `sync_post_counts`, `cap_note_votes`, `prepare_report`, `unfollow_on_block` (triggers) | Integrity |

## Migration history
| # | File | Summary |
|---|---|---|
| 1 | `20261116000000_initial_schema` | Core social schema, storage |
| 2 | `20261116010000_poll_votes_post_fk` | Integrity fix |
| 3 | `20261116020000_fragrance_catalog` | Catalog, search indexes, foreign keys |
| 4 | `20261118000000_community_consensus` | Voting |
| 5 | `20261118010000_verified_submissions_user_votes` | Verified, submissions |
| 6 | `20261118020000_username_changes` | Rate-limited renames |
| 7 | `20261118030000_collection_list_visibility` | Privacy audiences |
| 8 | `20261119000000_public_read_for_guests` | Signed-out browsing |
| 9 | `20261119010000_moderation` | Reports, blocks, admin |
| 10 | `20261119020000_notifications` | Inbox and push |
| 11 | `20261119030000_comment_likes` | Comment likes |
| 12 | `20261119040000_saved_posts` | Bookmarks |
| 13 | `20261119050000_decide_submission` | Verified and admin decisions |
| 14 | `20261119060000_direct_messages` | Encrypted DMs |
| 15 | `20261120000000_report_conversations` | Report a chat |
| 16 | `20261120010000_post_authors_delete_comments` | Authors moderate their own threads |

## Catalog import
`supabase/scripts/import_scento_catalog.sql` is a two-step, re-runnable import:
1. `net.http_get` downloads `perfumes.jsonl` and `houses.csv` from the dataset's GitHub repo, pinned to a commit.
2. A `DO` block reads the responses, splits lines, parses JSON and quoted CSV (with a regex), and upserts into `houses` and `fragrances`.
