# Infrastructure, Domain & Email

Everything runs on free tiers today, picked so that none of them limits growth later.

| Piece | Service | Notes |
|---|---|---|
| Database, Auth, Storage, Edge Functions | **Supabase** | Managed Postgres with RLS |
| Phone push | **Firebase Cloud Messaging** | HTTP v1 API from an Edge Function |
| Web hosting | **Cloudflare Pages** | Static Flutter web build |
| DNS | **Cloudflare** | |
| Domain | **Porkbun** | `scentedapp.app` |
| Transactional email | **Brevo** (SMTP) | Sender `invite@scentedapp.app` |

## Domain and DNS
1. Bought `scentedapp.app` on Porkbun.
2. Pointed the nameservers to Cloudflare.
3. Connected the domain to the Cloudflare Pages project.
4. Added DNS records for Brevo (sender verification) so auth emails from the domain are trusted.

## Web deployment
- `flutter build web` produces `build/web`, which is deployed to Cloudflare Pages.
- `web/_headers` sets `Cache-Control: no-cache` on `/`, `index.html`, `flutter_bootstrap.js`, `main.dart.js`, `flutter_service_worker.js` and `version.json`. Flutter doesn't fingerprint these files, so without this, visitors could keep an old version until a hard refresh.
- PWA `manifest.json` with regular and maskable icons (192 and 512 px).
- `web/confirmed/index.html`: a static "you're all set" page where email confirmation links land (it's in Supabase's allowed redirect URLs).
- `web/email/logo.png` is hosted on the site for email templates.

## Auth email
- Supabase Auth uses **custom SMTP through Brevo**, so mail comes from `invite@scentedapp.app` instead of Supabase's default sender. I compared Brevo with Resend and kept Brevo.
- Five branded templates in `supabase/email-templates/`:

| Template | Subject |
|---|---|
| Confirm signup | Confirm your Scented account |
| Change email address | Confirm your new Scented email |
| Reset password | Reset your Scented password |
| Magic link | Your Scented sign-in link |
| Invite user | You're invited to Scented |

- Design choices: a light layout, because many mail apps force-invert dark emails and ruin the colours, and minimal images, so nothing breaks when a mail client blocks remote images.

## Push setup
1. Created a Firebase project and added the iOS and Android config files to the app.
2. Stored the service-account JSON as the Supabase secret `FIREBASE_SERVICE_ACCOUNT`.
3. Deployed the `push` Edge Function.
4. Database triggers on `notifications` and `messages` call the function with `pg_net`.

## Environments
- One Supabase project for now; schema changes always go through migrations so the database can be rebuilt from the repo.
- The catalog import is pinned to a dataset commit, so rebuilding gives the same data.
