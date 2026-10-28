# Pickleball Queuing App: Build Phases (Start from Scratch)

**Stack:** React + Vite + TypeScript + Tailwind (PWA) on static hosting, with **Supabase** for Postgres, Auth, Realtime, Storage, Edge Functions, and pg_cron.

**Scope:** 1 venue, 3 courts, ~50 players, doubles only, free. One shared Admin account on one master tablet. Guests are created by the Admin.

**How to use this file:** work top to bottom. Each phase has tasks (check them off), a deliverable, and a "done when" test. Do not start a phase until the previous one passes its test.

---

## Phase overview

| Phase | Goal | Result |
|---|---|---|
| 0 | Setup | Empty app deployed, connected to Supabase |
| 1 | Auth and profiles | Players can sign up, onboard, edit profile |
| 2 | Admin console and courts | Admin logs in on tablet, controls court status |
| 3 | Sessions, QR, queue, guests | Players and guests are in a live queue |
| 4 | Matchmaking and match timers | Courts fill automatically and run timed matches |
| 5 | TV display, polish, pilot (MVP launch) | Real session run at the venue |
| 6 | Teams and notifications | Pairs stay together, push alerts work |
| 7 | Reservations, skill approvals, reports | Operations tools |
| 8 | Tournaments | Four tournament modes |
| 9 | Stats and analytics | History and insights |

**MVP = Phases 0 to 5.** Phases 6 to 9 come after the pilot.

---

## Phase 0: Setup (about 1 to 2 days)

### Tasks
- [ ] Create a GitHub repo (`pickleball-queue`)
- [ ] Create a Supabase project. Pick the region closest to the venue
- [ ] Scaffold the app: `npm create vite@latest` (React + TypeScript)
- [ ] Install: Tailwind CSS, `@supabase/supabase-js`, TanStack Query, React Router, `vite-plugin-pwa`, Vitest
- [ ] Install the Supabase CLI and run `supabase init` so schema changes live in migration files in the repo
- [ ] Create `.env` with the Supabase URL and anon key (never commit the service role key)
- [ ] Set up three route areas: `/` (player), `/admin` (tablet), `/tv` (display)
- [ ] Deploy to Cloudflare Pages, Netlify, or Vercel with auto-deploy from GitHub
- [ ] Add the PWA manifest and icons so the app is installable

### Suggested folder layout
```
src/
  pages/        player, admin, tv
  components/
  lib/          supabase client, helpers
  features/     auth, profile, queue, courts, matches
supabase/
  migrations/
  functions/
```

### Done when
The empty app loads on your phone from the live URL and can read a test row from Supabase.

---

## Phase 1: Auth and player profiles (about 1 week)

### Database
- [ ] Table `profiles` (id references `auth.users`, first_name, last_name, birthdate, photo_url, skill_level, dupr_band, role, waiver_accepted_at)
- [ ] Define skill level and DUPR band as enums, with the tier mapping in one place (a SQL function and a matching TypeScript helper)
- [ ] Trigger to create a profile row when a user signs up
- [ ] RLS: players read and update only their own row; birthdate not readable by other players

### Auth
- [ ] Enable email/password and Google sign-in in Supabase Auth
- [ ] Set up the Google OAuth client and redirect URLs
- [ ] Build signup and login screens
- [ ] Route guard: logged-out users go to login; incomplete profiles go to onboarding

### Onboarding (required before joining)
- [ ] Form: first name, last name, birthdate, skill level, optional DUPR band
- [ ] Waiver checkbox with timestamp
- [ ] Validate everything; decide the minimum age and how minors are handled

### Profile page
- [ ] Edit name and birthdate
- [ ] Profile photo upload to Supabase Storage (resize on the client, private bucket rules, one folder per user)
- [ ] Skill level change: for now, save a **request** (`skill_change_requests` table) for Admin approval

### Done when
A new player can sign up with Google or email, complete onboarding, edit their profile and photo, and cannot read another player's birthdate.

---

## Phase 2: Admin console and courts (about 1 week)

### Admin account
- [ ] Create the single Admin user manually in the Supabase dashboard
- [ ] Mark it `role = 'admin'` and write an `is_admin()` SQL helper used by all admin RLS policies
- [ ] Disable any way to self-assign the admin role
- [ ] Build the admin login screen at `/admin`
- [ ] Add the re-lock PIN: hashed PIN in `app_settings`, lock screen after 3 to 5 minutes of inactivity

### Courts
- [ ] Table `courts` with 3 seeded rows
- [ ] Status enum: `reserved`, `open_play`, `not_available`
- [ ] Admin console: large buttons to set each court's status
- [ ] Realtime subscription so changes show instantly on every device

### Player court board (read-only for now)
- [ ] Main page shows the 3 courts and their status labels

### Audit log
- [ ] Table `audit_log` (action, details, created_at). Log every admin action as "Admin"

### Tablet layout
- [ ] Design the console for a landscape tablet: courts on one side, queue on the other, large touch targets, no deep menus. Two staff must be able to use it together

### Done when
Changing a court status on the tablet updates a player's phone within about a second, and only the Admin account can change it.

---

## Phase 3: Sessions, QR join, queue, and guests (about 1.5 weeks)

### Sessions
- [ ] Table `sessions` (status, started_at, ended_at, match_minutes, max_players, qr_secret)
- [ ] Admin: start and close an open play session; set match length (12, 15, or 20 min) and max players
- [ ] Only one active session at a time

### QR join
- [ ] Edge Function `rotate_qr_token`: generates a signed token that expires (for example, every 60 seconds)
- [ ] Admin tablet displays the QR and refreshes it automatically
- [ ] Edge Function `join_session`: validates the token and session status, checks capacity, adds the player to the queue
- [ ] Player scans the QR, which opens the app, logs in if needed, and joins
- [ ] "Session full" message when at capacity
- [ ] Optional: geofence or on-site check (can wait until after the pilot)

### Queue
- [ ] Table `queue_entries` (session_id, player or guest reference, status, joined_at, games_played, on_leave_at)
- [ ] Status enum: `in_queue`, `called`, `playing`, `on_leave`, `left`
- [ ] Player view: queue position and estimated wait time
- [ ] **On leave** button, **I'm back** button
- [ ] pg_cron job: remove players who stay on leave longer than the limit (default 15 minutes)
- [ ] Player can leave the queue

### Guests
- [ ] Table `guests` (name, skill_level, dupr_band, waiver_acknowledged, last_seen)
- [ ] Admin: create guest in about 10 seconds (name, skill, optional DUPR, waiver checkbox)
- [ ] Admin: search existing guests by name before creating a new one
- [ ] Admin: add a guest to the queue, set them on leave, mark them back, remove them
- [ ] Guests are clearly marked in the queue

### Admin queue view
- [ ] Live list of the queue with tier, wait time, games played, and status
- [ ] Remove or move a player manually

### Done when
At least 10 test players (phones plus guests) can join through the QR, appear in the admin queue, toggle on leave, and see their position update live.

---

## Phase 4: Matchmaking and match timers (about 2 weeks)

This is the heart of the app. Build and test the logic **before** wiring it to the UI.

### Matchmaking logic (pure TypeScript or SQL, unit-tested)
- [ ] Tier function: DUPR band wins; otherwise the skill level
- [ ] Candidate ordering: longest wait first, then fewest games played
- [ ] Group rule: 4 players, tier gap of 1 or less
- [ ] Team split inside a group: highest with lowest tier on each side
- [ ] Skip anyone on leave
- [ ] Optional tier widening after a wait threshold (Admin toggle)
- [ ] Fair rotation when the queue size isn't divisible by 4
- [ ] Design the matcher around "units" (a solo player or a pair) from day one, so teams in Phase 6 fit in without a rewrite
- [ ] Write Vitest tests for: normal case, tier gap violations, too few players, all on leave, large queue of 50

### Matches and timers
- [ ] Tables `matches` (court, session, started_at, ends_at, paused_remaining_seconds, state) and `match_players` (match, player or guest, side)
- [ ] State: `called`, `playing`, `paused`, `finished`, `stopped`
- [ ] Edge Function `build_matches`: fills each open-play court with an eligible group
- [ ] Edge Function `start_match`: sets `ends_at = now + match length`
- [ ] Admin controls: **pause**, **resume**, **force stop**, **add minutes**
- [ ] Edge Function `end_match`: puts all 4 players back in the queue (increments games played) and triggers the next `build_matches`
- [ ] Countdown on every device computed from `ends_at` (and `paused_remaining_seconds` when paused)
- [ ] pg_cron backup job (every minute) ends any expired match the tablet missed
- [ ] Check-in window for called players (about 2 minutes); skip no-shows and pull the next player
- [ ] Admin manual override: drag and drop players between courts and the queue
- [ ] Optional setting: Admin approves each suggested group before it starts

### Player main page
- [ ] Court board shows the 4 players on each court
- [ ] Timer at the top for the player's court
- [ ] Status card: In queue, Called (go to Court X), Playing, On leave

### Done when
With 14 or more test players, three courts fill automatically, matches count down in sync on all devices, pausing and adding time work, and finished matches reshuffle everyone into new valid groups.

---

## Phase 5: TV display, polish, and pilot (about 1 week) (MVP launch)

### TV display
- [ ] `/tv` page, read-only, no login. Access via a display token in the URL
- [ ] Shows courts, timers, players on each court, and the next-up queue
- [ ] "Call player" action on the tablet shows the name on the TV with an alert sound (important for guests)
- [ ] Large text, readable from across the venue, auto-reconnects

### Reliability
- [ ] Handle disconnects: auto-reconnect and a "last updated" indicator
- [ ] Re-sync state when a phone wakes from sleep
- [ ] Test the tablet going offline: timers must keep running server-side
- [ ] Keep the tablet plugged in; document the emergency Admin login on a spare device

### Compliance and safety
- [ ] Privacy policy and terms pages
- [ ] Final rules for minors
- [ ] Confirm only the Admin can reach the admin routes and data (try as a normal player)

### Pilot
- [ ] Run a real open play session with staff on the tablet
- [ ] Record issues: wait times, confusing screens, wrong groupings, connection problems
- [ ] Fix blockers and run a second session

### Done when
A full real session runs end to end without needing the developer, and staff say they can run it themselves.

---

## Phase 6: Teams and notifications (about 2 weeks)

### Teams
- [ ] Table `teams` (two members, session, status)
- [ ] Invite flow: Player A invites Player B, who accepts
- [ ] Matcher treats a team as one unit. Team tier is the higher of the two members
- [ ] Teams stay together after matches and go to the back of the queue
- [ ] If one partner goes on leave, hold the spot about 5 minutes, then release the present partner to the solo queue
- [ ] Either partner can disband
- [ ] Admin can view and break teams

### Web push notifications
- [ ] Service worker and push subscription storage (`push_subscriptions`)
- [ ] Edge Function `send_push` with VAPID keys
- [ ] Notifications: "You're up next", "Go to Court X", "Match ending in 2 minutes", "Team spot being released"
- [ ] In-app alert and optional sound as a fallback (push on iPhones needs the PWA installed to the home screen, so test this early)

### Fairness upgrades
- [ ] Avoid repeat partners and opponents when alternatives exist
- [ ] Avoid the same 4 players meeting twice in a row
- [ ] Optional max games per player per session

### Done when
Two friends can form a team, play consecutive matches together against other valid groups, and receive a push when they are called.

---

## Phase 7: Reservations, skill approvals, and reports (about 1 week)

- [ ] Table `court_schedules`; Admin schedules reservations by date and time
- [ ] pg_cron applies and releases scheduled court status automatically
- [ ] Admin screen to approve or reject skill change requests
- [ ] "Report misrated player" for players; review screen for the Admin
- [ ] Skill change cooldown (for example, once every 30 days)
- [ ] Admin player search, edit, remove, and ban from a session

### Done when
A reservation set for tomorrow switches the court's status on time, and a skill change request can be approved from the tablet.

---

## Phase 8: Tournaments (about 3 to 4 weeks)

Build one mode at a time, easiest first. Warn the Admin when a tournament will occupy all 3 courts and pause open play.

### 8a. Round Robin
- [ ] Pair registration (or random partner assignment by tier)
- [ ] Schedule generator: every pair plays every other pair, spread over 3 courts
- [ ] Score entry and standings table

### 8b. King of the Court
- [ ] Winning pair stays, challenger pair rotates in
- [ ] Track streaks and games won

### 8c. DUPR mixer (12 players)
- [ ] Constraint solver (backtracking): no repeated partner, no repeated opponent
- [ ] Up to about 5 rounds is the realistic maximum for 12 players; generate as many valid rounds as possible and fall back gracefully
- [ ] Score entry and individual standings

### 8d. Single Elimination
- [ ] Bracket generator with seeding by tier and byes for non-power-of-two counts
- [ ] Advance winners and show the bracket on the TV display

### Done when
Each mode can run a full mock tournament with 12 or more players on the tablet and shows results on the TV.

---

## Phase 9: Stats, history, and analytics (about 1 to 2 weeks)

- [ ] Match history per player
- [ ] Games played and win/loss stats (once scores are being recorded)
- [ ] Admin dashboard: peak hours, average wait time, court utilization
- [ ] Audit log viewer
- [ ] Data export
- [ ] Guest-to-account conversion (guest scans a QR to claim their profile and history)
- [ ] Optional: DUPR verification through their partner API

---

## Always-on checklist (all phases)

- [ ] Every table has RLS enabled before it is used
- [ ] All schema changes go through migration files in Git
- [ ] Matchmaking and timer logic stay on the server, never on player phones
- [ ] Test on real phones (iPhone and Android) and the real tablet, not just a desktop browser
- [ ] Back up the database regularly (export or scheduled dump); free tier backups are limited
- [ ] Check Supabase free tier limits and project pausing; consider a paid plan or a scheduled ping before the pilot
- [ ] Keep a short `README` with setup steps and where credentials are stored

---

## Decisions to lock in during Phase 0 and 1

| Decision | Suggested default |
|---|---|
| Tier mapping | Beginner <2.00, Beginner ProMax 2.00-2.49, Novice 2.50-2.99, Low Intermediate 3.00-3.49, High Intermediate 3.50-3.99, Advanced 4.00-4.49, Expert 4.50+ |
| DUPR vs self-declared | DUPR wins when provided |
| Tier gap | Max 1 within a match of 4 |
| Match length options | 12, 15, 20 minutes |
| Check-in window | 2 minutes |
| On-leave timeout | 15 minutes |
| Team spot hold | 5 minutes |
| Re-lock PIN timeout | 3 to 5 minutes |
| Minimum age / minors | Decide before Phase 1 onboarding ships |

---

## Suggested timeline

| Phase | Estimate | Running total |
|---|---|---|
| 0 | 1-2 days | ~2 days |
| 1 | 1 week | ~1.5 weeks |
| 2 | 1 week | ~2.5 weeks |
| 3 | 1.5 weeks | ~4 weeks |
| 4 | 2 weeks | ~6 weeks |
| 5 | 1 week | ~7 weeks (MVP) |
| 6 | 2 weeks | ~9 weeks |
| 7 | 1 week | ~10 weeks |
| 8 | 3-4 weeks | ~13-14 weeks |
| 9 | 1-2 weeks | ~15 weeks |

These are rough estimates for one developer working steadily and will shift with your experience and availability.

---

## First day: do this now

1. Create the GitHub repo and Supabase project.
2. Scaffold Vite + React + TypeScript + Tailwind.
3. Install the Supabase client and connect it with your `.env` values.
4. Deploy the empty app and open it on your phone.
5. Start Phase 1 with the `profiles` migration.