# Decision Log

Major product and technical decisions, the alternatives I considered, and why I chose what I did.

| # | Decision | Alternatives considered | Why |
|---|---|---|---|
| 1 | **Flutter** for the client | Native Swift + Kotlin; React Native | One codebase for iOS, Android and web; consistent custom design; good performance |
| 2 | **Supabase** backend | Firebase (Firestore); a custom Node server | Relational data (follows, lists, votes) fits SQL; RLS gives security in the database; no servers to run |
| 3 | **Repository interface + mock implementation first** | Building against the backend from day one | Let me design and test the UI fast, then swap in Supabase without touching screens |
| 4 | **ChangeNotifier + InheritedNotifier** for state | Riverpod; Bloc; Provider | Zero dependencies and easy to follow at this size; Riverpod planned for caching and pagination |
| 5 | **Removed the diary** feature | Keep it as a secondary tab | Daily logging felt like a chore; discussion was what people wanted |
| 6 | Tabs: **Home · Discover · Collection · Profile · Settings** | Keep Watchlist as a tab | Wants fit inside Collection; Settings needs to be reachable from anywhere |
| 7 | **Algorithmic Home feed** (For You / Following / Trending) for general fragrance talk, not just reviews | Review-only feed; chronological only | More reasons to open the app; Following stays as the chronological option |
| 8 | **Log-capped creator boost** in ranking | Pure engagement; a learned model | Favours creators without burying small accounts; works with no training data |
| 9 | **Free, openly licensed catalog** (Scento, CC BY 4.0) with in-app credit | A paid or licensed database (my first choice); scraping | Legal to use and free; scraping isn't acceptable |
| 10 | **Import inside Postgres** with `pg_net`, pinned to a commit | A Python or Node script | Re-runnable from the SQL editor, reproducible, no extra tooling |
| 11 | **Wiki-by-vote** notes, similar fragrances, performance and seasons | Show only official data; editable wiki text | What people smell often differs from official notes; voting resists vandalism better than free-form editing |
| 12 | **Private raw votes, public totals** | Public individual votes | Stops people from targeting or brigading individual voters |
| 13 | **Pending queue + duplicate check + trusted members** for new fragrances | Open adding; admin-only adding | Keeps the catalog clean while letting the community grow it |
| 14 | **Manual verified badge** (only I grant it) | Automatic by follower count | Avoids gaming; verified accounts can be trusted to add fragrances directly |
| 15 | **Instagram-style rate-limited usernames** (2 per 14 days) | Unlimited; never changeable | Prevents impersonation churn but still allows mistakes to be fixed |
| 16 | **Public / followers / private** for the collection and each list | All public | People can be private about what they own and spend |
| 17 | **Website that matches the app**, optimised for laptops | A phone layout on the web; a separate web app | Same code, but a navigation rail and content widths that suit large screens |
| 18 | **Guest browsing** on the web | Sign-in wall | Shared links work for everyone; an account is still needed to take part |
| 19 | **Light mode** in addition to the dark theme | Dark only | Accessibility and user preference |
| 20 | **DM encryption at rest** (Vault key) | End-to-end encryption; plaintext | Multi-device sync and moderation of reported chats, while backups only hold ciphertext |
| 21 | **Push through an Edge Function signing its own JWT** | Firebase Admin SDK on a server | No server to run; no heavy SDK; full control of retries and token cleanup |
| 22 | **Cloudflare Pages + Cloudflare DNS** | Firebase Hosting; Netlify; Vercel | Free, fast CDN, and DNS and hosting in one place |
| 23 | **Brevo** for SMTP, sender `invite@` | Resend; Supabase's default sender | Generous free tier; branded sender; already set up and working |
| 24 | Bottle images | Licensed photos; user uploads; generated posters | Still open. For now, posters are generated from the scent family's colours |
