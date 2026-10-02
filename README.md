# Peakopoly

Mobile-first HTML/CSS/JavaScript PWA for the Training Team. The dedicated Supabase backend is already deployed in London at project `vcyfkdelmgowmhfffjgq`. The game is paused with four dummy players. Family HQ is unchanged.

## Live repository layout

The deployed PWA files are at repository root for GitHub Pages. `Peakopoly_Source.zip` contains the full source project with backend, schema, tests and optional Actions workflow. The separate Admin Access file is private and is never uploaded.

## Published app

Open https://brdlnx-ld.github.io/peakopoly/ . GitHub Pages is enabled from `main` / root. Upload updated frontend files to the same root to publish changes. Backend updates must be deployed separately to the Supabase Edge Function.

Install on Android using the browser's Install app / Add to Home screen menu; on iPhone use Safari → Share → Add to Home Screen. The app uses HTTPS.

## Sign-up requests

New players can open the app and select **Request to join**, or use https://brdlnx-ld.github.io/peakopoly/#signup . They choose a display name, token and four-digit PIN. PINs are bcrypt hashed immediately; Admin never sees them. Pending players cannot sign in.

Admin opens **More → Admin desk → Join requests**, then approves or rejects each request. Approval atomically creates a player with £1,200 and one roll using their chosen PIN. Requests are rate limited and duplicate approvals are blocked. Account creation through approval cannot be undone; deactivate a player if needed.

## Accounts and launch

- Admin login credentials are in the **separate private Admin Access file**.
- Dummy logins: Demo Amy / 1111; Demo Ben / 2222; Demo Ryan / 3333; Demo Sam / 4444.
- Start/resume the game in More → Admin desk to play with dummy accounts.
- Before real use, deactivate dummy players, add real accounts and choose a launch time. For a completely clean live game, create a new initial state and replace the dummy-state row before starting. Do not repurpose tested portfolios as real player accounts.
- Admin is an observer account with a strong password. Players use four-digit PINs. Never publish account hashes or credentials.

## Included

40-space board, all purchase/rent tables and role licences, server-generated dice, pending buy/pass decisions, cash-limited payments, doubles bonuses, START SHIFT, shuffled card decks, Break exits, fixed Peak Bonus, evenly developed upgrades, asynchronous mixed-asset trades with counter/reject/cancel/48-hour expiry, open-to-offers flags, net-worth leaderboard and alternate rankings, profiles/statistics, activity, in-app notifications, weekly results, administrative corrections and restricted Undo, session login/logout and installable/offline-cached shell.

## Rules decisions

- Normal daily grants occur at Europe/London midnight and accumulate to 10 rolls.
- At Sunday 04:30, every unused roll (including that Sunday's midnight grant) expires; the new week starts with one roll. Monday–Saturday midnight grants then continue. This gives seven usable normal allocations per game week.
- A game that has started continues weekly calendar maintenance while paused. An ended game is frozen.
- Break exit by roll counts toward rolls used but does not roll dice or move.
- Full-set base rent is doubled on each unupgraded location; upgraded locations use their level rent.
- Upgraded groups cannot be traded. Offers do not reserve assets or cash; ownership, balances and expiry are rechecked on acceptance.
- Back To Work cards and rolls cannot be traded.
- Card moves to a specific location travel forwards; backwards cards never collect START SHIFT.
- One normal roll per day, capped at 10 with bonuses. Cap overflow is discarded.

## Server and transaction design

The browser receives public state and sends an action name, limited arguments and an idempotency UUID. It cannot directly update database tables, choose dice, set cash, invent roll balances or assign ownership. No Supabase secret/service-role key is in the browser.

`peak_accounts` contains bcrypt hashes and persistent lockout counters; `peak_sessions` stores SHA-256 hashes of random 256-bit session tokens. Sessions last 30 days. Five failed attempts lock an account for 15 minutes; IP buckets add a second login limit. Resetting a PIN revokes that player's sessions. Deactivation revokes sessions and denies authentication. Admin authorization comes from the server account record, never a browser flag.

A dedicated Edge Function authenticates every action against the session table. Important rules execute in `server/engine.mjs`, imported only on the server. A single state document is practical for this 20–30-player game. `peak_commit` takes a row lock and performs a version-checked atomic commit of player state, assets, decks, trades, account changes, idempotency result and audit history. Competing changes retry against fresh state. Multi-player trades/payments commit together, avoiding partial transfers. Cryptographically secure, unbiased randomness supplies dice and Fisher–Yates deck shuffles. A retry uses the same request UUID so a lost response cannot spend another roll.

All exposed tables have RLS enabled and grants removed from `anon`/`authenticated`. Service-role-only functions are SECURITY INVOKER with an empty search path. No direct client database access is used. Edge JWT verification is disabled intentionally because the function implements custom PIN/session authentication; all routes except directory/login still require validated session tokens, and the scheduled route validates a private Vault secret.

`peak_audit` preserves transaction events, roll outcomes, admin action arguments (without PINs), corrections and reasons. `peak_requests` retains seven days of deduplication; old sessions and login-limit buckets are cleaned daily. UI roll history retains 1,000 latest rolls, feed 500 items and notifications 100 per player; the database audit retains full action history and supports pagination. Weekly summaries retain 52 weeks.

Cron calls the server once a minute. The server calculates London local boundaries, including daylight saving, rather than assuming a fixed UTC Sunday time. Every game action also performs calendar maintenance, so a delayed cron cannot allow an expired roll to be used. Sunday summaries compare cash plus portfolio/upgrades against the prior weekly baseline. Scheduler authentication uses a secret stored only in Vault.

Undo restores the complete pre-action game snapshot for the latest action only. The Admin UI sends the reviewed revision; a later commit causes Undo to fail. Undo never removes the audit history. Account creation and PIN reset cannot be undone; the next correction must be explicit. State reads are not commits unless calendar maintenance changed state.

## Local checks

Node 20+:

```sh
npm test
npm run serve
```

No frontend build step or npm runtime dependencies are required. The live integration test is intentionally scoped to the deployed dummy instance and needs the private Admin Access file; it changes test data. `tests/ui.cjs` additionally requires Playwright and Chromium and tests a 390px viewport, all screens, overflow and offline shell.

## Recreate a backend elsewhere

1. Create a Supabase **Free** project in London. Apply `supabase/schema.sql` in SQL Editor.
2. Seed `peak_state` with a fresh state from `initial()` in `server/engine.mjs`; add player accounts with `extensions.crypt(pin,extensions.gen_salt('bf',12))`. Create a separate observer Admin account with a strong password.
3. Deploy `server/index.ts` and `server/engine.mjs` as Edge Function `peakopoly`, using the built-in service-role environment. Keep custom authentication code; disable gateway JWT verification for this function.
4. Change the URL in `supabase/scheduler.sql`, then apply the scheduler setup once. It requires pg_net, pg_cron and Vault.
5. Change `web/config.js` to that project's function endpoint. Run security advisors and live tests.

No initial credential SQL is included in this package. `initial-state.json` is a non-secret example containing dummy player state only.

## Free-tier operation

The project was created with Supabase reporting £0/month. Keep it on Free. There are no paid APIs, custom domains, external push services or frontend dependencies. Background client reads run every five minutes only while visible, with a refresh on return to the app. Thirty continuously visible clients produce about 267,840 monthly reads; the minute clock adds 44,640, leaving room within the currently documented 500,000 free Edge invocations. Normal team use should be substantially lower. Usage limits can restrict service; do not upgrade to a paid plan to resolve them. The fixed-size state/notification/feed windows and compact audit rows limit database growth, but full history will eventually need an exported archive for a very long-running game.

Official references: https://supabase.com/docs/guides/functions/pricing and https://docs.github.com/en/pages/getting-started-with-github-pages .

## Verification status

22 deterministic engine tests passed. Live deployed checks passed for authentication, authorization, private state, idempotency, atomic concurrent updates, stale Undo rejection, roll/purchase, trade acceptance, lockout, PIN reset and logout. Supabase security advisors returned no warning/error findings (only informational notices for intentionally inaccessible RLS tables with no client policies). Cron responses were verified successful.

Mobile browser visual QA is **not yet completed**: this environment has no installed browser, and the Chromium download was blocked/invalid. The runnable UI test is included so it can be completed after hosting. Do a short dummy-player phone playtest before inviting the Training Team. GitHub Pages publication completed successfully. The live HTTPS login screen, backend player directory and desktop layout were checked. A physical phone playtest is still recommended; the automated 390px browser test has not been completed.
