# Pickleball Open Play Queuing App: Project Plan

## 1. Overview

A web app (PWA) that manages open play and tournaments for **one venue with 3 courts**, supporting up to about **50 players** per session. All play is **doubles (4 players per match)**. The app is **free** to use.

**Core idea:** Players scan a QR code to join the open play queue. The system groups them by skill tier, assigns them to courts, runs timed matches, and reshuffles everyone after each match (except pre-formed teams).

### Key decisions already made

| Decision | Choice |
|---|---|
| Venues | 1 |
| Courts | 3 |
| Max players | ~50 |
| Match format | Doubles only (no singles) |
| After a match (open play) | Everyone reshuffled, except formed teams |
| Admin setup | **One shared Admin account, logged in only on one master tablet, used together by 2+ staff** |
| Players without phones | Admin creates guest profiles on the tablet |
| Cost to players | Free |
| Backend | Supabase (no self-managed server) |

---

## 2. Tech Stack

| Layer | Choice | Notes |
|---|---|---|
| Frontend framework | **React + Vite** (or Next.js) | Vite is simpler since the app is a single-page app |
| Language | **TypeScript** | Fewer bugs in matchmaking and state logic |
| Styling | **Tailwind CSS** | Fast to build tablet-first and mobile-first layouts |
| App type | **PWA** | Installable on phones, supports push notifications |
| State/data fetching | **TanStack Query** + Supabase client | Caching and clean realtime updates |
| Backend platform | **Supabase** | Postgres, Auth, Realtime, Storage, Edge Functions |
| Database | **Supabase Postgres** | With Row Level Security (RLS) |
| Auth | **Supabase Auth** | Google sign-in and email/password for players; one manually created Admin account |
| Realtime | **Supabase Realtime** | Live queue, court board, timers, TV display |
| File storage | **Supabase Storage** | Profile pictures |
| Server logic | **Supabase Edge Functions** + Postgres functions | Matchmaking, reshuffle, match end, tournament scheduling |
| Scheduled jobs | **pg_cron** | Backup auto-end of matches, on-leave timeouts |
| Push notifications | **Web Push** (via service worker) | "You're up next", "Go to Court 2" |
| QR code | `qrcode` library, **signed rotating token** | Generated on the admin tablet |
| Hosting (frontend) | **Cloudflare Pages / Netlify / Vercel** | Free static hosting |
| Version control | **GitHub** | Plus CI deploy on push |
| Testing | **Vitest** for matchmaking logic | Unit-test tier rules and reshuffle |

**Why this stack:** no server to maintain, free tiers cover 50 players, and realtime plus auth come built in.

**Free tier check:** Supabase free projects may pause after inactivity and have size, realtime, and backup limits. Confirm current limits on their pricing page, and consider a paid plan or scheduled ping if the venue is idle for long periods.

---

## 3. System Architecture

```
 Player phones ─┐
 Admin tablet  ─┼──►  Static host (PWA)  ──►  Supabase
 Courtside TV  ─┘                              ├─ Auth (Google / email / Admin)
                                               ├─ Postgres + RLS
                                               ├─ Realtime (live updates)
                                               ├─ Storage (profile photos)
                                               ├─ Edge Functions (matchmaking)
                                               └─ pg_cron (scheduled jobs)
```

### Design rules

1. **Server-authoritative logic.** Matchmaking, reshuffle, and match end run in Supabase, never on player phones.
2. **Timestamp-based timers.** Each match stores `ends_at` (and `remaining_seconds` when paused). Every device computes the countdown locally from that value, so all screens stay in sync without a ticking server.
3. **Backup auto-end.** The admin tablet triggers match end, and a pg_cron job catches any expired matches if the tablet disconnects.
4. **Single control point.** All court, timer, and queue controls come from the one Admin account on the one tablet, so there are no concurrent-control conflicts to reconcile.
5. **RLS everywhere.** Players can edit only their own profile; only the Admin account can change courts, timers, and queues.

---

## 4. User Roles

| Role | Who | Access |
|---|---|---|
| **Player** | Registered users with the app | Own profile, join queue, view courts |
| **Guest** | Players without phones | Created and controlled by the Admin only |
| **Admin** | One shared account, used by 2+ staff together on the master tablet | Full control of sessions, courts, matches, guests, tournaments, and settings |

There is no Owner/Staff split, no staff accounts, and no PINs for individual staff.

---

## 5. Player (User) Functionality

### 5.1 Account and onboarding
- Sign up or log in with **Google** or **email and password**.
- First-time setup is required before joining any queue:
  - First name and last name
  - Birthdate (stored privately, only age is shown)
  - Skill level: Beginner, Beginner ProMax, Novice, Low Intermediate, High Intermediate, Advanced, Expert
  - DUPR rating (optional): N/A, or a rating band (see tier table in section 8)
  - Waiver acceptance (checkbox, timestamp saved)
- Minors: collect parental consent or block signup under a set age (decide before launch).

### 5.2 Profile
- Change profile picture, name, and age (birthdate).
- Change skill level, subject to rules:
  - Skill changes need **Admin approval** or a **cooldown** (e.g., once every 30 days) to prevent gaming the system.
- View games played this session and all-time.
- View match history (Phase 3).

### 5.3 Joining open play
- Scan the venue **QR code** to automatically enter the active open play session.
- The QR works only while a session is open, and uses a signed token that rotates so it can't be reused remotely.
- Optional geofence or on-site check to confirm the player is at the venue.
- If the session is full, show a "Session full" message with a waitlist option.

### 5.4 Main page (what players see)
- **Court board:** all 3 courts with their status and the 4 players on each.
- **Match timer:** time left on each court's current match, at the top.
- **Court status labels:** Reserved, Open Play, Not Available.
- **My status card:** current status, queue position, and estimated wait time.
- **Next-up list:** who is queued next.

### 5.5 Player status

| Status | Meaning |
|---|---|
| **In queue** | Present and waiting |
| **Called** | Assigned to a court, about 2 minutes to check in |
| **Playing** | On court |
| **On leave** | Queued but away; skipped in matchmaking |
| **Left** | Session ended or removed |

- **On leave button:** the player taps it whenever stepping away.
- **I'm back button:** one tap to return to the queue.
- **On-leave timeout:** auto-removed after about 15 minutes (Admin adjustable).
- **Check-in:** if a called player doesn't check in within the window, they are skipped and the next player is pulled in.

### 5.6 Teams
- A player can invite one other player to form a **pair**.
- Pairs are **not split or shuffled** after games; they stay together.
- A team goes to the back of the line after each match like everyone else.
- Team tier for matching is the **higher** of the two members' tiers.
- If one partner goes on leave, the team spot is held for about 5 minutes. After that, the present partner returns to the solo queue.
- Either partner can disband the team.

### 5.7 Notifications
- "You're up next"
- "Go to Court 2"
- "Match ending in 2 minutes"
- "Your team spot is being released"
- Web push plus in-app alert and optional sound.

### 5.8 Other player features
- Report a misrated player (goes to the Admin for review).
- Leave the queue and rejoin (position rules defined by Admin).
- Light and dark mode.

---

## 6. Admin Functionality

The Admin Console is a **tablet-first interface** with large touch targets, designed so **two staff members can use it together at the same time** from one screen. The queue, courts, and timers stay visible without deep menus.

### 6.1 Admin access and security
- **One Admin account**, created manually in Supabase. There is no public admin signup.
- Logged in **only on the master tablet**, with a long-lived session on that device.
- **Re-lock PIN:** one app-level PIN that locks the console after a few minutes of inactivity, so an unattended tablet can't be used by players. This is a single shared PIN, not a per-staff system.
- The **audit log** records actions as "Admin" with a timestamp. Individual staff are not identified.
- **Fallback:** store the Admin credentials somewhere safe so a spare device can log in if the tablet fails. Server-side timers keep matches running during a tablet outage.
- Keep the tablet on a charger during sessions.

### 6.2 Session management
- Start and close an **Open Play session**.
- Generate and display the **rotating QR code**.
- Set the **max queue size** (up to about 50-60) and lock the session when full.
- Set the **match duration**: 20, 15, or 12 minutes (or custom).
- Set tier-widening rules (see section 8).

### 6.3 Court management
- Set each court's status:
  1. **Reserved:** only for ongoing major events
  2. **Open Play**
  3. **Not Available:** used for something other than pickleball
- Status changes show on all player devices immediately.
- Schedule reservations by date and time (Phase 2).

### 6.4 Match controls
- Start a match on a court.
- **Pause**, **resume**, **force stop**, and **add minutes** to an ongoing match.
- **Manual override:** drag and drop players between courts and the queue.
- Approve or reject a suggested group before it starts (optional setting).

### 6.5 Guest management
- Create a **guest profile** in about 10 seconds: name, skill level, optional DUPR.
- Search for returning guests by name so regulars aren't recreated.
- Waiver acknowledged checkbox.
- Control guest on-leave, I'm back, and removal from the tablet.
- **Call player:** shows the guest's name on the TV and plays an alert.
- Convert a guest to a full account (Phase 3).

### 6.6 Player management
- Search, view, and edit players.
- Approve skill-level change requests.
- Review misrating reports.
- Remove or ban a player from a session.

### 6.7 Tournament mode (Phase 3)

| Mode | Rules |
|---|---|
| **DUPR mixer (12 players)** | Players shuffled each round. No one partners with or opposes the same person more than once. Max about 5 rounds. Uses all 3 courts. |
| **Round Robin** | Every pair plays every other pair. Needs fixed pairs. |
| **Single Elimination** | Bracket; losers are out. Needs fixed pairs. |
| **King of the Court** | Winning pair stays, challengers rotate in. |

- Choose pair registration or random partner assignment by tier.
- Warn the Admin that a tournament can take over all 3 courts and block open play.
- Enter scores and advance rounds.

### 6.8 Courtside TV display
- Read-only screen showing all courts, timers, current players, and the next-up queue.
- Shows called names, including guests.
- Opens from a simple URL with a display token. No login needed on the TV, and it never uses the Admin login.

### 6.9 Reports and analytics (Phase 3)
- Peak hours, average wait time, court utilization.
- Games played per player.
- Audit log viewer.

### 6.10 Admin settings
- Change the re-lock PIN.
- Edit court names.
- Change tier and matchmaking rules.
- Export data.

---

## 7. Feature Summary by Role

| Feature | Player | Guest | Admin |
|---|:-:|:-:|:-:|
| Sign up / log in | ✅ | ❌ | ✅ (one account) |
| Edit own profile | ✅ | ❌ | ❌ |
| Scan QR to join | ✅ | ❌ | ❌ |
| Join queue (on behalf) | ❌ | ✅ via Admin | ✅ |
| On leave / I'm back | ✅ | ✅ via Admin | ✅ |
| View court board | ✅ | ✅ via TV | ✅ |
| Form a team | ✅ | ❌ | ✅ |
| Receive push notifications | ✅ | ❌ | ❌ |
| Start/close session | ❌ | ❌ | ✅ |
| Set court status | ❌ | ❌ | ✅ |
| Pause / stop / add time | ❌ | ❌ | ✅ |
| Move players manually | ❌ | ❌ | ✅ |
| Create guest profiles | ❌ | ❌ | ✅ |
| Approve skill changes | ❌ | ❌ | ✅ |
| Run tournaments | ❌ | ❌ | ✅ |
| Edit settings and rules | ❌ | ❌ | ✅ |

---

## 8. Matchmaking Rules

### 8.1 Tier mapping (proposed)
Your original DUPR ranges overlapped at the edges. These clean bands give each of the 7 skill levels exactly one DUPR band.

| Tier | Skill level | DUPR band |
|:-:|---|---|
| 1 | Beginner | Below 2.00 |
| 2 | Beginner ProMax | 2.00 - 2.49 |
| 3 | Novice | 2.50 - 2.99 |
| 4 | Low Intermediate | 3.00 - 3.49 |
| 5 | High Intermediate | 3.50 - 3.99 |
| 6 | Advanced | 4.00 - 4.49 |
| 7 | Expert | 4.50 and above |

- If a player has a DUPR rating, **DUPR decides the tier**. Otherwise the self-declared skill level is used.
- Confirm this mapping and the "DUPR wins" rule before building.

### 8.2 Grouping rule
- A match is 4 players (2 vs 2).
- **Tier gap within a match must be 1 or less** (highest tier minus lowest tier).
- Confirm this interpretation of "same tier or 1 tier up."

### 8.3 Selection order
1. Longest wait time first.
2. Fewest games played this session.
3. Avoid repeat partners and opponents when alternatives exist.
4. Avoid the same 4 players meeting twice in a row.

### 8.4 Building teams inside a group of 4
- Pair the highest tier with the lowest tier on each side so the sides balance.
- Pre-formed teams stay together and count as one unit.

### 8.5 Edge cases
- **Not enough players in a tier:** after a wait threshold (e.g., 5 minutes), widen the allowed gap. Admin can toggle this.
- **Not divisible by 4:** the longest-waiting or most-played players rotate out fairly.
- **Called player no-show:** skip and pull in the next eligible player.
- **Team partner on leave:** hold the spot, then release.
- **Full reshuffle:** after each match, everyone returns to the queue and new groups are built.

---

## 9. Database Design (Supabase Postgres)

| Table | Purpose |
|---|---|
| `profiles` | Name, birthdate, photo URL, skill level, DUPR, role (Player or Admin), waiver status |
| `guests` | Admin-created players: name, skill, DUPR, waiver flag, last seen |
| `courts` | Court name, status (Reserved, Open Play, Not Available) |
| `court_schedules` | Reservations by date and time (Phase 2) |
| `sessions` | Open play sessions: start, end, match duration, max players, settings |
| `queue_entries` | Player or guest in a session, status, joined time, games played, on-leave time |
| `teams` | Two members, status, session link |
| `matches` | Court, session, `started_at`, `ends_at`, `paused_remaining`, state |
| `match_players` | Players per match with team side and optional score |
| `tournaments` | Mode, status, settings |
| `tournament_rounds` | Round number, match pairings |
| `skill_change_requests` | Pending skill edits for Admin approval |
| `misrating_reports` | Reports against players |
| `push_subscriptions` | Web push endpoints per user |
| `app_settings` | Re-lock PIN (hashed), default match duration, timeouts, tier-widening rules |
| `audit_log` | Action, timestamp, details (all attributed to "Admin") |

There is no `staff` table.

**Security:** enable Row Level Security on every table. Admin-only policies are tied to the single Admin account. Birthdates are visible only to the owner of the profile and the Admin.

---

## 10. Edge Functions and Jobs

| Function / Job | Trigger | Does |
|---|---|---|
| `join_session` | Player scans QR | Validates token, adds to queue |
| `build_matches` | Match ended or court opened | Picks next 4, applies tier and fairness rules |
| `start_match` | Admin action | Sets `ends_at`, notifies players |
| `end_match` | Timer expiry or force stop | Returns players to queue, triggers reshuffle |
| `pause` / `resume` / `add_time` | Admin action | Updates match timing fields |
| `expire_on_leave` | pg_cron every minute | Removes players past the on-leave limit |
| `expire_checkins` | pg_cron every minute | Skips no-shows |
| `auto_end_matches` | pg_cron every minute | Backup for expired matches |
| `send_push` | Status changes | Sends web push to players |
| `rotate_qr_token` | Timer | Issues a new signed QR token |

---

## 11. Build Phases

### Phase 1: MVP
- Supabase setup, schema, and RLS
- Google and email login for players
- Single manually created Admin account, tablet-only session, and re-lock PIN
- Onboarding and profile with photo upload
- Admin tablet console: sessions, courts, timers
- Session start/close and rotating QR join
- Single queue, tier matching, full reshuffle
- Guest profiles and guest search
- Court board for players
- Courtside TV display
- Pause, force stop, add minutes
- On leave and I'm back, with timeouts
- Waiver acknowledgment
- Basic audit log

### Phase 2
- Teams (consider moving into the MVP, since they affect matchmaking design)
- Web push notifications
- Anti-repeat partner and opponent logic
- Scheduled court reservations
- Skill change approvals and misrating reports
- Max games and sit-out fairness caps

### Phase 3
- Tournament modes: Round Robin, King of the Court, DUPR mixer, Single Elimination
- Score entry, match history, and stats
- Analytics dashboard
- Guest-to-account conversion
- Optional DUPR API verification
- Optional: separate staff accounts or an on-duty name tag, only if accountability becomes necessary

---

## 12. Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Players lie about skill level | Admin approval or cooldown on changes; misrating reports; DUPR verification later |
| Wi-Fi drops at the venue | Auto-reconnect, "last updated" indicator, server-side timers continue |
| Tablet fails or battery dies (single point of failure) | Keep it on a charger; store Admin credentials safely so a spare device can log in; timers keep running server-side |
| Tablet left unlocked | Re-lock PIN after inactivity plus the device's own screen lock |
| Actions not attributed to individual staff | Accepted trade-off for simplicity; add an on-duty tag or staff accounts later if needed |
| Two staff using the tablet at once | Single-screen console with large buttons and no deep menus |
| Supabase free tier pausing | Scheduled ping or paid plan |
| Long waits with 50 players | Show estimated wait; cap session size; fair rotation |
| Guests can't get notifications | TV display, call player alert, verbal calls |
| Remote QR joining | Signed rotating token plus optional geofence |
| Personal data and minors | Private birthdate, waiver, parental consent decision, privacy policy |

---

## 13. Items to Confirm Before Building

- [ ] Tier-to-DUPR mapping in section 8.1
- [ ] Meaning of "same tier or 1 tier up"
- [ ] Whether teams belong in the MVP
- [ ] On-leave timeout (suggested 15 minutes) and team spot hold (suggested 5 minutes)
- [ ] Minimum age and rules for minors
- [ ] Skill change approach: Admin approval or cooldown
- [ ] Wait threshold before widening the tier gap
- [ ] Re-lock PIN timeout (suggested 3 to 5 minutes)
- [ ] Where the Admin credentials are stored for emergency access
- [ ] Wi-Fi reliability at the venue