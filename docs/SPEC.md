# Splitbook — Product Spec (v1)

*Status: draft, 29 Sep 2026*
*Location: `docs/SPEC.md` in the `SplitBook-App/app` repo*

## 1. Goal

Splitbook is a personal app that collects every completed workout from Google Health and shows all-time bests per workout type. It is a free alternative to paid PR tracking, built so a record never implies that numbers from different workouts came from one workout.

Splitbook targets regular, everyday athletes. Ultra-distance and advanced training features are out of scope; those users are served by paid apps.

## 2. Audience and access

- v1 users are the owner plus friends and family. There is no Play Store listing.
- Code lives in the `SplitBook-App` GitHub organization, split across two repositories:

| Repo | Visibility | Contents |
|---|---|---|
| `SplitBook-App/app` | Public | React Native (Expo) app, docs (including this spec), metric formulas |
| `SplitBook-App/backend` | Public | Webhook receiver, Google Health API client, record calculation, push sending, database access |

- Both repos are public. Security comes from keeping secrets out of git, not from hiding code:
  - Secrets live in a git-ignored `.env` locally, and in Google Secret Manager or Cloud Run settings once deployed. They are never committed to either repo.
  - The Google OAuth client secret, user tokens and all health data are handled only by the backend and its hosting, never by the app.
  - No real health data is committed, including test fixtures.
  - A committed secret is treated as leaked and rotated immediately.
  - Every webhook notification is verified as coming from Google, since the endpoint's existence is public.
- Both repos use the same `main-protection` ruleset: no direct pushes to `main`, PRs need 1 approval from someone other than the last pusher, stale approvals are dismissed, conversations must be resolved, squash-merge only, and no force-pushes or deletion. Repo admins can bypass only by merging a PR.
- Both repos have secret scanning with push protection and Dependabot alerts enabled.
- Contributors get Write access to both repos and work on branches.
- Google Cloud project caveats (see §17 for verification):
  - Refresh tokens expire after about 7 days while the app is in "Testing" mode.
  - Unverified apps with sensitive scopes are capped at about 100 users and show a warning screen.
- A public launch would require Google app verification. That is out of scope for v1.

## 3. Platform and architecture

| Layer | Choice | Repo |
|---|---|---|
| App | React Native with Expo (Android first; iOS and web later from the same code) | `SplitBook-App/app` |
| Data source | Google Health API (Pixel Watch and all Fitbit models), via Google OAuth 2.0 | `SplitBook-App/backend` |
| Backend | Python + FastAPI, packaged as a Docker container | `SplitBook-App/backend` |
| Backend hosting | Google Cloud Run (free tier, request-based billing, scales to zero) | `SplitBook-App/backend` |
| Background work | Google Cloud Tasks (queues webhook processing) | `SplitBook-App/backend` |
| Scheduled jobs | Google Cloud Scheduler (token refresh and other periodic jobs) | `SplitBook-App/backend` |
| Database | Neon Postgres (free plan, scales to zero) via SQLAlchemy + Alembic | `SplitBook-App/backend` |
| Raw stream storage | Google Cloud Storage (one compressed file per workout) | `SplitBook-App/backend` |
| Push | Expo push notifications (sent by backend, received by app) | both |

All Google services (OAuth, Cloud Run, Cloud Tasks, Cloud Scheduler, Cloud Storage) live in one Google Cloud project.

### Why FastAPI

- It generates an OpenAPI schema automatically, which `openapi-typescript` turns into TypeScript types for the app. Backend field changes then surface as type errors in the app.
- Pydantic validates webhook payloads and Google Health responses before they reach record calculations.
- Async I/O lets the several Google Health calls per workout run in parallel.

### Backend modules

| Module | Responsibility |
|---|---|
| `webhook` | Receives Google Health notifications, replies immediately, enqueues a Cloud Task |
| `google_client` | OAuth token storage and refresh, Google Health API calls |
| `ingestion` | Fetches a workout (summary, laps, HR, distance, GPS), stores raw data, mirrors edits and deletes |
| `records` | Pure Python, no web or database code: splits, records, efficiency scores, GPS glitch flags. Most heavily unit-tested |
| `notifications` | Sends Expo push messages |
| `api` | Endpoints the app reads from |

### Data storage

Splitbook's data is split into two kinds, stored in two places.

**Raw streams → Google Cloud Storage**
- Per-second heart rate, distance and GPS for each workout, stored as one compressed file per workout.
- These are large (thousands of points per hour of exercise) and must be kept, because adding a split distance, changing HR zones or improving a formula requires recalculating from raw data.
- Keeping them out of Postgres keeps the database well inside its free storage limit.

**Structured data → Neon Postgres**
- Users, OAuth token references, workouts (summaries), splits, records, settings, hidden types, exclusions and tombstones.
- Small and relational. Queries like "fastest 10K ride this year" or "best efforts in the last 90 days" are straightforward in SQL.
- Unique keys on source workout IDs make webhook ingestion idempotent.
- Models via SQLAlchemy; schema migrations via Alembic.
- Neon computes scale to zero after 5 minutes idle, matching Cloud Run. Hitting a free limit suspends compute until the next month instead of billing, and does not delete data.

**Database options considered and rejected**
- **Supabase (free):** Postgres too, but free projects pause after inactivity, and its bundled auth and storage aren't needed alongside Google OAuth.
- **Firestore:** same Google project, but cross-window record queries are clunky and the 1 MiB document limit rules it out for raw streams.
- **Cloud SQL:** no free tier.

### Hosting options considered and rejected

- **Render (free):** spins down after 15 minutes idle and takes about a minute to wake, which risks webhook timeouts and retries. Free Postgres expires after 30 days.
- **GitHub Pages:** static files only; cannot run a backend. Still the place for any v1 web page or docs.
- **Fly.io / Railway:** believed to have no ongoing free tier, only trials or credits (see §17).

### Ingestion flow

1. A user finishes a workout and it syncs to Google Health.
2. The webhook notifies the backend, which replies immediately and enqueues a Cloud Task.
3. The task fetches the exercise session: summary, laps, heart rate, distance and GPS.
4. Raw streams are written to Cloud Storage and the workout summary and splits to Postgres; then records for that workout type are recalculated.
5. If a new record is set, a push notification is sent.

Rules:
- Edits and deletes made in Google Health are mirrored into the app.
- Ingestion is idempotent. Receiving the same event twice changes nothing.
- There is no backfill. Only workouts completed after the user connects are imported.

## 4. Workout types and tabs

- Every workout type the API reports gets its own tab: running, cycling, pickleball, swimming, strength and so on.
- Tabs appear only for types the user has actually recorded. There are no empty tabs.
- **Hide tab:** a hidden type keeps recording, but its workouts go into a **Hidden** folder.
- Hidden types can be unhidden at any time, with all their history intact.

## 5. Records

### 5.1 Whole-workout records

Each record is shown as its own card with the value, the date, and a link to the source workout.

- **Distance-based types:** longest distance, longest duration, fastest average pace or speed.
- **Non-distance types:** longest duration, most calories, and the highest max HR recorded. Max HR is shown as a reference value, not framed as an achievement.
- **Qualifying minimums:** a workout must reach its type's minimum distance to count toward records. Minimums apply in both unit modes.

| Workout type | Minimum distance |
|---|---|
| Running | 500 m |
| Cycling | 1 km |
| Walking | 500 m |
| Hiking | 1 km |
| Swimming | 100 m |
| Any other distance type | 500 m |

### 5.2 Best efforts (clean splits)

- Splits always start at 0: the 5K split is 0–5 km and the 10K split is 0–10 km.
- A workout qualifies for a split only if it covered at least that distance.
- Split distances are set per workout type by the user, with editable defaults. Defaults cover regular distances; Splitbook targets everyday athletes, not ultra-distance users:

| Workout type | Metric default splits | Imperial default splits |
|---|---|---|
| Running | 1K, 5K, 10K, half marathon (21.0975 km), marathon (42.195 km) | 1, 3, 5, 10 mi, half marathon (13.1 mi), marathon (26.2 mi) |
| Cycling | 5K, 10K, 20K, 40K, 100K | 1, 3, 5, 10, 25, 50 mi |
| Walking | 1K, 5K, 10K, 20K | 1, 3, 5, 10 mi |
| Hiking | 5K, 10K, 20K | 3, 5, 10 mi |
| Swimming | 100 m, 400 m, 1500 m | 100, 500, 1650 yd |
| Any other distance type | 1K, 5K, 10K | 1, 3, 5 mi |

- The unit setting decides which split set is shown. Half marathon and marathon are the same distances in both modes, only labeled differently.
- Records for both split sets are calculated from the stored raw streams and kept up to date, so switching units is instant and needs no re-import.

- Swimming best efforts depend on the distance data the watch records. Pool swims usually have no GPS and rely on lap counts (see §17).

- Best efforts are calculated only from workouts that have GPS or distance data.
- **Split timing is pause-adjusted:** a split's time is the time the watch was recording. Time is excluded only while the workout was paused, and auto-pause counts the same as a manual pause. Stops without a pause count toward the split. Users who want moving time pause the workout.

Each split distance shows two cards.

**Fastest [distance]**
- Shows time and date.
- A "that day" strip shows avg, max and low HR over that split, plus the dominant HR zone.

**Most efficient [distance]**
- Uses a different color and has its own date badge.
- Shows avg, max and low HR, all from that one workout, plus the pace held.
- Scoring:
  - Efficiency = speed ÷ avg HR (meters per beat)
  - Steadiness = avg HR ÷ max HR
  - **Score = efficiency × √steadiness**
- Low HR is displayed but not scored, because on a split starting at 0 it mostly reflects the resting reading at the start.
- An **ⓘ** icon explains the formula and links to a GitHub Discussions thread in the `SplitBook-App/app` repo for suggestions.

Display rules:
- If the two cards come from different workouts, a "different workout" label sits between them.
- If both come from the same workout, they merge into one card with a "same day" badge.
- Tapping any card opens the source workout.

## 6. Heart rate zones

- **Default:** Google Health's zones.
- **Custom 5 zones:** the user enters each boundary.
- **Calculated 5 zones:** percentages of max HR, where max HR is either entered by the user or the highest ever recorded. A heart rate reserve option using resting HR from the API is also possible.
- Only raw HR is stored. Zone labels are derived at display time, so changing zones updates all history.

## 7. Units

- The default is whatever the watch provides.
- Users can switch between metric and imperial in the app.
- Values are stored in SI units and converted for display. All numbers recalculate when units change.
- Switching units also switches the split set (metric km splits vs imperial mile/yard splits; see §5.2).

## 8. Time windows

- v1: All-time, This year and Last 90 days.
- These apply to every record, best effort and total.

## 9. Progress

- Each record has a history graph showing every time it was broken.
- Each record shows a green or red % change against the previous best or the selected comparison.

## 10. Totals and achievements

- **Yearly totals:** distance run this year, distance cycled this year, and totals for other distance types. Manual entries (v2) count toward these.
- **Achievements:** a section of milestones to keep users motivated. Achievements are per user, with no leaderboards, and use only data from the API.

### 10.1 Achievement sets

- **Firsts:** first workout of each type, first 5K run, first 10K ride, first half marathon.
- **Consistency:** workouts in 4, 12 and 26 consecutive weeks.
- **Record-setting:** first PR, 10 PRs, and a PR in 3 different workout types.
- **Variety:** 3, 5 and 10 different workout types recorded.
- **Yearly totals** and **lifetime totals** per distance type (below).

### 10.2 Yearly totals

- Reset on 1 January in the user's time zone. They can be earned again each year (e.g. "100 km run in 2026").

| Type | Yearly tiers |
|---|---|
| Running | 50, 100, 250, 500, 1,000 km |
| Cycling | 100, 200, 500, 1,000, 2,500 km |
| Walking | 100, 250, 500, 1,000 km |
| Hiking | 50, 100, 250 km |
| Swimming | 10, 25, 50, 100 km |
| Other distance types | 50, 100, 250 km |

### 10.3 Lifetime totals

- Earned once and never reset.
- In v1, "lifetime" means since the user connected Splitbook, because there is no backfill.
- When backfill is added (v2), lifetime totals and lifetime achievements are recalculated to include imported past workouts.

| Type | Lifetime tiers |
|---|---|
| Running | 100, 500, 1,000, 2,500, 5,000 km |
| Cycling | 500, 1,000, 5,000, 10,000 km |
| Walking | 500, 1,000, 2,500, 5,000 km |
| Hiking | 100, 250, 500, 1,000 km |
| Swimming | 25, 50, 100, 250 km |
| Other distance types | 100, 500, 1,000 km |

### 10.4 Units for totals achievements

- In imperial mode, tiers use the same round numbers in miles (100 mi, 500 mi and so on), not converted values.

## 11. Data quality

- The backend flags suspected GPS glitches using two rules:
  - **Speed limit:** any point implies a speed above the workout type's limit.
  - **Teleport:** consecutive GPS points jump more than 200 m within 2 seconds.

| Workout type | Flag above |
|---|---|
| Running | 30 km/h |
| Walking | 15 km/h |
| Hiking | 15 km/h |
| Cycling | 100 km/h |
| Swimming | 10 km/h |
| Any other distance type | 100 km/h |

- A flagged workout still counts until the user acts on it.
- The user is shown the flagged workout and asked whether to exclude it.
- **Exclude** removes the entire workout from all records and totals.
- Excluded workouts sit in a restorable list.

## 12. Activities list

- A full list of all ingested workouts.
- Users can delete a workout. This deletes it only in Splitbook, and a tombstone stops it from being re-imported.
- Manual add comes in v2.

## 13. Notifications

- A push notification when any record or best effort is broken, naming the record and the % improvement.

## 14. Transparency

- Every calculated metric has an ⓘ explaining how it is computed.
- The formulas are public in the `SplitBook-App/app` repo, and improvements are welcome through GitHub Discussions.

## 15. v2 backlog

- Custom date ranges: All-time plus up to 3 user-defined ranges.
- Manual workout entry, counting toward whole-workout records and totals but not best efforts.
- Logging sets, reps and weight for strength training.
- A preference for clean splits vs best-anywhere segments.
- A preference for split timing: pause-adjusted (v1 default), elapsed, or moving time.
- User profiles with cloud storage and a web version.
- Backfill of past workouts. Backfilled workouts update records, best efforts, lifetime totals and lifetime achievements.
- Optional custom domain (e.g. `splitbook.com`); v1 uses free GitHub Pages hosting if a web page is needed.

## 16. Open decisions

None. All v1 decisions are made. New questions that come up during the build are added here.

## 17. To verify

Claims made during planning that must be checked against official docs before relying on them. Tick each box and note the date and source.

### Google Cloud Run
- [ ] Free tier with request-based billing (us-central1 and other Tier 1 regions): first 180,000 vCPU-seconds, 360,000 GiB-seconds and 2 million requests per month. Source: cloud.google.com/run/pricing
- [ ] Idle time is not billed with request-based billing, and the service scales to zero.
- [ ] Typical cold start for a small FastAPI container is a few seconds.
- [ ] A billing account with a card is required even when staying within the free tier.
- [ ] Budget alert (e.g. $1) set up on the project on day one.
- [ ] Pick a Tier 1 region (e.g. us-central1); Tier 2 regions have smaller free allowances.

### Google Cloud Tasks and Cloud Scheduler
- [ ] Cloud Tasks free tier covers Splitbook's volume (believed to be about 1 million operations per month).
- [ ] Cloud Scheduler free tier (believed to be a few jobs per billing account) covers token refresh and periodic jobs.
- [ ] Cloud Tasks can call a Cloud Run endpoint with authentication (OIDC token), so the task endpoint isn't public.

### Google OAuth and Google Health API
- [ ] Refresh tokens expire after about 7 days while the OAuth app is in "Testing" mode. If true, decide: accept weekly re-sign-in, or publish the app (unverified) to get long-lived tokens.
- [ ] User cap and warning screen for unverified apps using sensitive health scopes (believed to be about 100 users).
- [ ] Google Health API webhook: required response time, retry behavior, and how the subscriber endpoint is verified.
- [ ] Which data is available per exercise session for Pixel Watch vs older Fitbit models (HR, distance, GPS, laps).
- [ ] Heart rate zones provided by the API and how many there are.
- [ ] Exercise sessions expose pause and resume events, including auto-pause, for Pixel Watch and older Fitbit models (needed for pause-adjusted split timing).
- [ ] Whether distance/HR samples are recorded during pauses, and if so, that they can be excluded using the pause intervals.
- [ ] Swimming data: whether pool swims provide lap or length data with timestamps (needed for 100 m / 400 m / 1500 m splits), and whether open-water swims include GPS.

### Neon Postgres
- [ ] Free plan limits per project: 0.5 GB storage, 100 CU-hours compute per month, 5 GB network transfer per month. Source: neon.com/docs/introduction/plans
- [ ] Scale to zero after 5 minutes idle (cannot be disabled on Free), and typical wake-up latency is acceptable for webhook tasks.
- [ ] Hitting a Free limit suspends compute until the next month and does not delete data or bill.
- [ ] No credit card required for the Free plan.
- [ ] Choose a Neon region close to the Cloud Run region (e.g. AWS us-east or GCP us-central) to limit latency.
- [ ] Connect via the pooled (PgBouncer) connection string so Cloud Run instances don't exhaust connections.

### Google Cloud Storage
- [ ] Always-free tier (believed to be about 5 GB of Standard storage per month in certain US regions, plus limited operations and egress). Source: cloud.google.com/storage/pricing
- [ ] Bucket must be in an always-free-eligible region (believed to be us-central1, us-east1 or us-west1).
- [ ] Measure the real compressed size of one workout's streams (HR + distance + GPS) to estimate how many workouts fit in the free tier.

### Supabase (only if Neon doesn't work out)
- [ ] Free plan: 500 MB per project, 2 projects, and the inactivity period after which free projects pause.

### Hosting alternatives (only if Cloud Run doesn't work out)
- [ ] Fly.io and Railway: confirm whether any ongoing free tier exists.
- [ ] Render free tier: 15-minute spin-down and ~1 minute wake-up still apply.