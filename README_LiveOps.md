# INK BALL - LiveOps

Server-driven content for INK BALL: change tuning, schedule events and run experiments **after release, without shipping a new build**. No third-party SDK or account is needed. The game reads a JSON file over HTTPS and sends analytics to an HTTP endpoint, and the whole backend is one Python file.

## What it does

| Feature | What the player sees | What it demonstrates |
|---|---|---|
| **Remote config** | Time limit, XP and ink change after the next refresh | Server-driven tuning, offline cache, safe defaults |
| **Scheduled live events** | A banner on the title screen with a countdown ("Double XP Weekend, ends in 1d 4h"). Modifiers apply automatically | Time-windowed content, event calendar, upcoming-event teasers |
| **Daily challenge** | One generated level per day, identical for every player. First clear pays a bonus plus a streak bonus (up to 7 days) | Date-seeded procedural content, retention loop, streaks |
| **A/B experiments** | Players are split into variants (for example 60s vs 75s timer) | Sticky weighted assignment, variant tagged on every analytics event |
| **Server clock** | Event windows and the daily reset can't be skipped by changing the phone's clock (after one sync) | Anti-cheat basics using the HTTP `Date` header |
| **Analytics** | Nothing visible. Events are queued, saved to a file and POSTed in batches | Event pipeline, batching, retry, offline queue |
| **Live Ops screen** | Daily status, live and upcoming events, config version, last sync, A/B variant, analytics counters | Debug and observability surface |

## Architecture

```
liveops.json (static file or any web server)         POST /events (batches of JSON lines)
        |  GET, refreshed every 10 min + on app focus          ^
        v                                                      |
 LiveOpsService  --Changed-->  LevelManager (partial)      Telemetry
  - config + PlayerPrefs cache    - time limit, ink, XP        - in-memory queue (max 500)
  - server clock offset           - daily challenge flow       - telemetry.jsonl on device
  - events / multipliers          - main menu banner + button  - batched POST, retry on failure
  - A/B assignment                - Live Ops screen
  - daily seed + streak
```

Precedence for tuning values: **game defaults < remote tuning < A/B variant < live events** (time bonuses add, multipliers multiply).

Failure handling: no network or a bad config keeps the last good cached config, then falls back to built-in defaults. Malformed events (bad dates, end before start) are skipped, not fatal. Analytics never throws.

## Files

```
Scripts/
  LiveOps/LiveOpsData.cs        config model (JsonUtility-friendly; 0 means "not set")
  LiveOps/LiveOpsService.cs     fetch, cache, clock, events, experiments, daily + streak
  LiveOps/Telemetry.cs          analytics queue, file log, batched POST
  LevelManager.LiveOps.cs       UI + daily flow (partial class)
  LevelManager.cs, LevelManager.Help.cs   small hooks (replace your copies)
LiveOpsServer/
  liveops.json                  sample config (edit the dates)
  liveops_server.py             local backend: serves the config, receives analytics
```

## Setup

1. Copy the `Scripts` folder contents into your project's scripts folder, replacing `LevelManager.cs` and `LevelManager.Help.cs`.
2. Select the object with `LevelManager`. A new **LiveOps** section is in the Inspector:
   - **Enable Live Ops**: on.
   - **Live Ops Config Url**: leave empty for **demo mode** (sample events generated relative to now, ideal for screenshots), or set the address of your `liveops.json`.
   - **Telemetry Url**: optional endpoint for analytics batches.
3. Android: Player Settings > Other Settings > **Internet Access: Require**.

### Hosting the config for free (HTTPS)
Put `liveops.json` in a GitHub repository with GitHub Pages enabled and use `https://<user>.github.io/<repo>/liveops.json`. Edit the file on GitHub and every running copy of the game picks it up within 10 minutes (or on the next app focus). Pages sends a `Date` header, so the server clock sync works.

### Running the local backend (config + analytics)
```
cd LiveOpsServer
python liveops_server.py --rolling        # http://localhost:8080, events always "live" for demos
```
- Config URL: `http://localhost:8080/liveops.json`, Telemetry URL: `http://localhost:8080/events`
- `GET /stats` shows event counts per type. Received events are in `telemetry_received.jsonl`.
- Unity blocks plain HTTP by default: Player Settings > Other Settings > **Allow downloads over HTTP: Always allowed** (local testing only; use HTTPS in production). To test from a phone, use your computer's LAN address instead of `localhost`.

## Config reference (`liveops.json`)

| Field | Meaning |
|---|---|
| `version` | Shown on the Live Ops screen; bump it when you change the file |
| `motd` | Message on the title banner when no event is live |
| `dailyDisabled`, `eventsDisabled` | Kill switches |
| `telemetryUrl` | Analytics endpoint (the Inspector value wins if both are set) |
| `tuning.timeLimit / xpMultiplier / inkMultiplier` | Global tuning. **0 = keep the game's value** |
| `tuning.dailyBonusXp / streakBonusXp` | Daily rewards (0 = 100 / 25) |
| `events[]` | `id, title, description, startUtc, endUtc` (ISO 8601, UTC) plus `xpMultiplier, inkMultiplier, timeBonus` (0 = no effect) |
| `experiments[]` | `id` and `variants[]` (`name, weight`, overrides `timeLimit, xpMultiplier, inkMultiplier`) |

## Analytics events

Every event carries `ts, install, session, platform, version` and the A/B tag `ab` (for example `timer_test=B`).

| Event | Extra fields |
|---|---|
| `session_start` | `touch, player_level` |
| `level_start` | `level` |
| `level_retry` | `level, mode` |
| `level_complete` | `level, mode, stars, lives_left, time_left, ink_left_pct, xp, ball` |
| `level_fail` | `level, mode, ball` |
| `ball_swap` | `level, ball` |
| `daily_start` | `day, streak, done_today` |

`mode` is `normal` or `daily`; `level` is 0 for the daily challenge. Example questions these answer: completion rate by level, drop-off level, which ball players bring, whether the 75s timer (variant B) changes stars or retries.

## Design decisions

- **Static JSON instead of an SDK:** zero setup, free hosting, easy to inspect. The service is the only place that talks to the backend, so it can be swapped for Unity Gaming Services Remote Config or Firebase later.
- **0 means "not set":** `JsonUtility` can't tell a missing number from 0, so a 0 in tuning, events and variants always means "use the default". Kill switches are inverted (`dailyDisabled`) so a missing field leaves a feature on.
- **Daily level from a date seed:** reuses the existing level generator, so no extra content is stored. The seed is the server's UTC date, so everyone gets the same level.
- **XP can't be farmed:** best score is stored from base XP, and events multiply the *improvement* only. The daily bonus is paid once per day.
- **Sticky experiments:** the variant is stored on first assignment, so a player's experience doesn't flip when weights change.

## Known limitations

- Rewards live in `PlayerPrefs`, so a determined player can edit them. Server-side validation would need a real backend (for example Cloud Save or Firebase).
- The clock offset is measured at sync time. If the phone's clock is moved while offline, it is corrected on the next successful sync.
- Config refresh is polling (10 min). Push updates would need Firebase Cloud Messaging or similar.
