# GlucoseSync — Vibe Coder Agent Context

## Who you are building for

The user is a Type 2 diabetic managing their condition with medication and a **FreeStyle Libre 2 CGM sensor** from Abbott. They use the official **LibreLink iOS app** to read their sensor, but it has no Apple Watch app — so checking glucose requires pulling out a phone every time. This project eliminates that friction.

The user is a **vibe coder**: they have product instincts and can describe what they want clearly, but they rely on you to make technical decisions, write production-quality code, and guide them through setup steps. Be opinionated. Make choices. Don't ask unnecessary clarifying questions — build the right thing and explain your decisions briefly.

---

## What you are building

**GlucoseSync** — a personal glucose monitoring system with:

1. **A Supabase backend** — Postgres database + scheduled Edge Function that polls the Abbott LibreLinkUp API every 5 minutes and stores glucose readings.
2. **An iOS app (SwiftUI)** — shows current glucose + trend arrow, and a time-range chart (1h / 8h / 24h / 3d / 7d / 30d / 90d). Subscribes to Supabase Realtime for live updates.
3. **An Apple Watch app (watchOS, SwiftUI)** — shows the current glucose value and trend on a glance screen. The Digital Crown scrolls through time ranges. Has complications for the watch face.

The system is designed to be **sensor-agnostic**: FreeStyle Libre 2 is v1, but the data model and sync layer must be clean enough to add Dexcom G7 or other CGMs later.

---

## Tech Stack — stick to these

| Concern | Choice |
|---|---|
| iOS / watchOS | Swift 5.9+, SwiftUI, Swift Charts (native), WatchKit + SwiftUI |
| Database | Supabase (Postgres) |
| Auth | Supabase Auth with Apple Sign In |
| Sync / backend logic | Supabase Edge Functions (Deno/TypeScript), cron schedule |
| CGM data source (v1) | LibreLinkUp informal API (see API section below) |
| Realtime | Supabase Realtime (Postgres Changes) |
| Secrets management | Supabase Vault or environment variables in Edge Functions |

Do **not** introduce extra frameworks, cloud functions on other platforms, or third-party charting libraries. Swift Charts is sufficient.

---

## Database Schema

```sql
-- One table, keep it simple
create table glucose_readings (
  id           uuid primary key default gen_random_uuid(),
  user_id      uuid references auth.users not null,
  recorded_at  timestamptz not null,
  value_mgdl   numeric(6,2) not null,
  trend        text check (trend in (
                 'rising_rapidly', 'rising', 'stable',
                 'falling', 'falling_rapidly', 'unknown'
               )),
  source       text not null default 'libre2',
  raw          jsonb,
  created_at   timestamptz default now(),
  unique (user_id, recorded_at, source)
);

create index on glucose_readings (user_id, recorded_at desc);

-- RLS: users only see their own data
alter table glucose_readings enable row level security;
create policy "own data only" on glucose_readings
  using (auth.uid() = user_id);
```

Store everything in **mg/dL** internally. Convert to mmol/L on display based on user preference (1 mmol/L = 18.0182 mg/dL).

---

## LibreLinkUp API (CGM data source)

Abbott does not publish an official API, but the LibreLinkUp mobile app communicates with a REST API that has been reverse-engineered by the open-source community. Key points:

- Base URL varies by region. Common ones: `api-eu.libreview.io` (EU), `api.libreview.io` (US), `api-ap.libreview.io` (Asia-Pacific).
- Authentication: `POST /llu/auth/login` with `{ email, password }` → returns a bearer token.
- Connections: `GET /llu/connections` → returns linked patients/sensors.
- Current glucose: `GET /llu/connections/{patientId}/graph` → returns current glucose, trend, and historical readings.
- The token expires; implement refresh logic.
- Store the LibreLinkUp credentials securely in Supabase Vault (never in client code).

Reference open-source implementations for the exact request headers required (the API checks for specific `product` and `version` headers):
- https://github.com/timoschlueter/nightscout-librelink-up
- https://github.com/DiaKEM/libre-link-up-api-client

The Edge Function should:
1. Run on a cron every 5 minutes (`schedule: "*/5 * * * *"`)
2. Fetch credentials from Supabase Vault
3. Call the LibreLinkUp API
4. Upsert new readings into `glucose_readings` (use the `unique` constraint to avoid duplicates)
5. Map Abbott trend codes to the `trend` enum above

---

## iOS App — Key Screens & Behaviors

### 1. Auth Screen
- Apple Sign In button
- On first login, prompt for LibreLinkUp email + password (stored via Supabase — not in Keychain on device, so the Edge Function can use them server-side)

### 2. Dashboard (main screen)
- Large current glucose value (e.g., **108 mg/dL** or **6.0 mmol/L**)
- Trend arrow: ↑↑ ↑ → ↓ ↓↓
- Color coding:
  - Red: < 70 mg/dL (hypo)
  - Green: 70–180 mg/dL (target)
  - Orange/Yellow: > 180 mg/dL (hyper)
- Time-range picker: segmented control or horizontal scroll: `1h | 8h | 24h | 3d | 7d | 30d | 90d`
- Swift Charts line chart below, filling the selected range
- Chart has a horizontal band shading the target range (70–180)
- Supabase Realtime subscription: chart updates live when a new reading arrives

### 3. Settings
- mmol/L vs mg/dL toggle
- Target range customization (default 70–180 mg/dL)
- LibreLinkUp credentials management
- (Future) Add sensor source

---

## Apple Watch App — Key Screens & Behaviors

### Main View
- Full-screen: current glucose value + trend arrow
- Subtitle: time since last reading (e.g., "2 min ago")
- Color background matches hypo/target/hyper zone
- **Digital Crown**: rotating changes the displayed time range (cycles through 1h → 8h → 24h → 3d → 7d)
- Mini chart appears below when a range > 1h is selected

### Complications
- Implement at minimum: Graphic Corner and Circular complications
- Show current value + trend arrow
- Refresh via `WKExtension.requestedUpdateBudget` / background refresh

### Data sync
- The Watch app fetches from Supabase directly (same REST API as iOS)
- Use `WKURLSession` background transfers for battery efficiency
- Do not rely on WatchConnectivity as the primary data path (phone may not be reachable)

---

## Project Structure

```
GlucoseSync/
├── supabase/
│   ├── migrations/
│   │   └── 001_glucose_readings.sql
│   └── functions/
│       └── glucose-sync/
│           └── index.ts          # Edge Function: LibreLinkUp poller
│
├── GlucoseSync.xcodeproj
│
├── GlucoseSync/                  # iOS target
│   ├── App/
│   │   └── GlucoseSyncApp.swift
│   ├── Features/
│   │   ├── Auth/
│   │   ├── Dashboard/
│   │   └── Settings/
│   ├── Models/
│   │   └── GlucoseReading.swift
│   ├── Services/
│   │   ├── SupabaseService.swift
│   │   └── RealtimeService.swift
│   └── Config.swift              # Supabase URL + anon key
│
├── GlucoseSyncWatch/             # watchOS target
│   ├── GlucoseSyncWatchApp.swift
│   ├── Views/
│   │   ├── GlucoseMainView.swift
│   │   └── GlucoseMiniChart.swift
│   ├── Complications/
│   └── Services/
│       └── WatchSupabaseService.swift
│
└── README.md
```

---

## Conventions & Code Style

- **Swift**: use `async/await` everywhere, no Combine unless forced. Use `@Observable` (Swift 5.9 macro) for view models, not `ObservableObject`.
- **Error handling**: never silently swallow errors. Surface them as user-visible alerts in the UI.
- **Units**: internal storage always mg/dL. A `GlucoseReading` model has a computed `valueMmol` property.
- **Supabase Swift SDK**: use the official `supabase-swift` package (`github.com/supabase/supabase-swift`).
- **No third-party charting**: Swift Charts only.
- **Edge Function**: TypeScript, Deno. Keep it under 150 lines. One function, one job.
- **Comments**: write comments explaining *why*, not *what*.

---

## What "done" looks like for v1

1. `supabase/migrations/001_glucose_readings.sql` runs cleanly on a fresh Supabase project.
2. The Edge Function deploys, runs on cron, and populates the table with real readings from LibreLinkUp.
3. The iOS app authenticates, shows the current glucose + trend, and the chart works for all 7 time ranges.
4. The Watch app shows the current reading and the Digital Crown changes the time range.
5. A new reading arriving via Realtime updates the iOS chart without a manual refresh.

---

## Known Constraints / Gotchas

- The LibreLinkUp API is unofficial. If Abbott changes it, the Edge Function needs updating. Design the sync layer as a thin adapter so swapping sources is easy.
- FreeStyle Libre 2 readings are every **1 minute** when scanned, but the LibreLinkUp API typically returns a reading every **5 minutes**. Don't over-poll.
- Apple Watch background refresh budget is limited. Prioritize the complication refresh over full app refresh.
- The Supabase anon key is safe to embed in the iOS app (it's public by design). RLS policies enforce data isolation.
- Region matters for the LibreLinkUp API base URL. Make it a configurable environment variable in the Edge Function.
