# Security & Privacy

The app ships only Supabase's **publishable key**, which is designed to be public. All protection comes from the database itself.

## Principles
1. **Never trust the client.** Permissions, limits and counters are enforced in Postgres.
2. **Least privilege.** Each role gets only the tables, columns and functions it needs.
3. **Private by default where it matters:** raw votes, blocks, DMs, notifications and saved posts are readable only by their owner.

## Row Level Security
- RLS is on for every table.
- Typical pattern: anyone can read public content; you can only insert, update or delete rows where `user_id` / `author_id = auth.uid()`.
- Visibility-aware reads: collections and lists use `can_view_audience(owner, visibility)` inside the policy, so followers-only and private rows never leave the database.
- Guests (`anon` role) get read-only access to public content and nothing else.

## Column-level grants
Clients can update only specific columns, for example on `profiles` only `display_name`, `bio`, `avatar_path`, `avatar_tone` and `collection_visibility`. That makes it impossible to:
- verify yourself (`is_verified`) or make yourself admin (`is_admin`)
- change your username without the rate limit (`username`)
- change follower or like counts

## `SECURITY DEFINER` hygiene
Functions that need elevated access follow a strict pattern:
- `set search_path = ''` and fully qualified names, which blocks search-path hijacking
- `revoke all … from public, anon` followed by explicit grants
- internal helpers (`publish_submission`, `_dm_key`, trigger functions) are revoked from every client role
- the caller is always taken from `auth.uid()`, never from a parameter

## Abuse limits
| Action | Limit |
|---|---|
| Username change | 2 per rolling 14 days |
| Fragrance submissions | 3 a day (unverified) |
| Note votes | 15 per person per fragrance |
| Reports | 20 a day; one open report per item |
| DMs | 30 a minute; 5 before a request is accepted |
| Submission approval | 3 trusted votes; trust needs an account 14+ days old with 5+ reviews |

## Direct-message privacy
- Encrypted at rest with `pgp_sym_encrypt`, using a random 256-bit key created in **Supabase Vault**.
- Only conversation members can read, through RLS and member-checked functions.
- Realtime events carry only ciphertext.
- TLS in transit. Deliberately not end-to-end, so chats sync across devices and reported chats can be reviewed.

## Storage
- Buckets limit file size and MIME type.
- Users can only write, replace, delete or list files under their own `<uid>/` folder.

## Push
- The Edge Function uses the service role on the server only; the Firebase service account is kept as a Supabase secret.
- Rows are claimed once and only while fresh, so replayed or forged calls can't trigger extra pushes.
- Dead device tokens are deleted automatically.

## Account safety
- Banning sets `banned_until = infinity` and deletes refresh tokens, so the account is signed out on every device.
- Blocking works both ways and is enforced in policies, not just hidden in the UI.

## Licensing and content
- Catalog data is used under CC BY 4.0 with attribution in the app.
- Seed and demo data uses a fictional brand.
