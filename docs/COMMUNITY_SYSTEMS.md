# Community Systems

Scented's fragrance pages and catalog are shaped by the community, with rules in the database that keep this fair and resistant to spam.

---

## 1. Wiki-by-vote fragrance pages
Official note pyramids often don't match what people actually smell, so every fragrance page shows what the community agrees on.

| What people vote on | Rule |
|---|---|
| **Notes you smell** | One vote per note per person, up to 15 notes per fragrance (enforced by a trigger). Notes are normalised (lowercase, trimmed, 2–40 characters, letters, digits and a few symbols). |
| **"Smells like"** | Agree or disagree that fragrance A smells like B. |
| **Longevity** | 1–5 |
| **Sillage** (how far it projects) | 1–4 |
| **Seasons** | spring / summer / autumn / winter |
| **Time of day** | day / night |

### Design details
- **One vote per person** is guaranteed by primary keys, not app code.
- **Raw votes are private.** RLS lets you read only your own votes. Everyone else sees totals through one function, `fragrance_consensus(fragrance_id)`, which returns a single JSON object with top notes, voter counts, similar fragrances, longevity and sillage counts, season and time counts, and *your* votes. This means individual voters can't be singled out or targeted.
- **Symmetric pairs:** "A smells like B" is the same claim as "B smells like A", so each pair is stored once with the ids in a fixed order (`fragrance_a < fragrance_b`), and it shows on both pages. The order uses byte-wise (`"C"`) collation so Postgres and Dart's `String.compareTo` always agree on which id comes first.
- **Upserts:** changing a vote is a single upsert. I learned PostgREST updates every supplied column on conflict, so the key columns also need an UPDATE grant, while the policy still pins `user_id` to the caller.
- **Profile view:** `user_fragrance_votes(user, fragrance)` returns one person's votes, so opening a fragrance from someone's profile shows their opinion.

---

## 2. Community fragrance submissions
Anyone can add a fragrance that's missing from the catalog.

```
submit_fragrance()
  ├─ already in the catalog?            → error "already in the catalog"
  ├─ same submission already pending?   → error "waiting for review"
  ├─ unverified and 3+ today?           → error "3 a day"
  ├─ verified submitter?                → publish straight away
  └─ otherwise                          → pending queue
                                              │
review_submission()  (trusted members only, not your own submission)
  ├─ 3 approvals, or 1 from a verified member → publish
  └─ 3 rejections, or 1 from a verified member → reject
```

- **Duplicate protection twice over:** a check in the function, plus a partial unique index on `(lower(house), lower(name), concentration) WHERE status = 'pending'`.
- **Trusted reviewer** = verified, **or** an account at least 14 days old with at least 5 reviews. Trust has to be earned, which makes sock-puppet approval hard.
- **Publishing** (`publish_submission`) finds or creates the house, generates a unique URL-safe id (slugify plus a counter on collisions), inserts the fragrance with `source = 'community'`, and links it back to the submission. It's internal only; clients can't call it.
- `decide_submission` lets verified members and admins fix a submission's details and decide it in one step.
- Submitters get a notification when their submission is approved or rejected.

---

## 3. Verified badge
- An `is_verified` flag that **only I can set**, by hand in the database. It isn't in the client's update grant, so nobody can verify themselves.
- Verified accounts can add fragrances without review and decide submissions alone.

---

## 4. Username changes (Instagram-style)
- Clients can't update `profiles.username` directly. The only way is `change_username()`.
- At most **2 changes per rolling 14 days**. `username_change_status()` tells the UI how many changes are left and when the next one becomes available.
- Every change is logged in `username_changes`.
- Availability is checked live during sign-up and in settings with `username_available()`, using the format `^[a-z0-9._]{3,24}$`, case-insensitive and unique.

---

## 5. Privacy audiences
- The collection and each list can be **public**, **followers-only** or **private**.
- `can_view_audience(owner, visibility)` is used inside RLS policies, so hidden rows never leave the database. Signed-out guests only ever see public items, because `auth.uid()` is null for them.

---

## 6. Moderation
- **Reports** on posts, comments, reviews, lists, accounts and conversations, with nine reasons (spam, harassment, hate, nudity, violence, self-harm, impersonation, misinformation, other).
  - A trigger records **who the report is about and a snapshot of the content** when it's filed, so evidence survives even if the author deletes it.
  - One open report per person per item, at most 20 reports a day, and no reporting yourself.
- **Blocks** work in both directions: blocking removes follows both ways, and the blocked person can't follow you, comment on your posts or message you. RLS policies and the DM functions all check `is_blocked_between()`.
- **Admins** (flag set by hand) use a moderation queue with three actions:
  - **dismiss**
  - **remove**: deletes the post, comment, review or list; for an account, clears its bio and photo
  - **ban**: sets the account's `banned_until` to infinity and revokes its refresh tokens, so it's signed out everywhere

  Acting on one report closes every open report about the same item, or about the same person on a ban. Admins can't be banned through this path.
- Post authors can delete comments on their own posts.
- Reporting and blocking are App Store requirements for apps with user-generated content, so they were built before launch.

---

## 7. Direct messages
- **Message requests:** a chat from someone you don't follow lands in Requests. Until you accept, they can send at most 5 messages and don't get read receipts.
- **Rate limit:** 30 messages a minute per sender.
- **Encryption at rest:** bodies are stored as `pgp_sym_encrypt` ciphertext with a 256-bit key created in **Supabase Vault**. The key never appears in a table or the app. Only member-checked functions decrypt.
- **Why not end-to-end?** Server-side decryption lets chats sync to every device (phone and web) and lets moderators review a *reported* conversation. I documented this trade-off in the migration.
- Realtime delivery, unread counts, mute, "delete chat" (hides history for you only) and unsend (removes the message for both people).
