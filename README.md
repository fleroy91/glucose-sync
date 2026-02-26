# GlucoseSync 🩸⌚

> A personal glucose monitoring companion — built for FreeStyle Libre 2 (and beyond), with an Apple Watch app, beautiful charts, and full data ownership via Supabase.

---

## Context

Managing Type 2 diabetes with a FreeStyle Libre 2 sensor is effective — but the official LibreLink iOS app has a significant gap: **there is no Apple Watch app**. Checking glucose means pulling out your phone, unlocking it, opening the app. That's friction nobody needs.

**GlucoseSync** solves this by:

- Pulling glucose readings from the LibreLink ecosystem (via the LibreLinkUp / Libre API)
- Storing them in a self-owned Supabase database
- Displaying them in a native iOS app with rich time-range charts
- Surfacing the current glucose reading + trend arrow on Apple Watch, with Digital Crown navigation through time ranges

The architecture is designed to be **sensor-agnostic** from day one — FreeStyle Libre 2 is the first supported source, but Dexcom, Medtronic, and other CGM integrations can be added incrementally.

---

## Features

### iOS App

- Real-time glucose reading with trend arrow (rising, stable, falling, rapid)
- Configurable time-range charts: 1h · 8h · 24h · 3d · 7d · 30d · 90d
- Color-coded zones: hypoglycemia, target range, hyperglycemia
- Background sync — readings arrive without opening the app
- HealthKit write support (optional)

### Apple Watch App

- Current glucose value + trend on the watch face
- Complication support (corner, modular, circular)
- Digital Crown scrolls through time ranges in the main view
- Haptic alerts for out-of-range readings

### Backend (Supabase)

- Postgres table for glucose readings (timestamp, value mmol/L or mg/dL, sensor source, trend)
- Row-level security — your data, only yours
- Edge Function (or scheduled job) to pull readings from the Libre API at regular intervals
- REST + Realtime subscriptions for the iOS/watchOS clients

---

## Architecture Overview

```
┌─────────────────────────┐
│   FreeStyle Libre 2      │  (Bluetooth / NFC)
│   Sensor (Abbott)        │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│   LibreLink iOS App      │  (official Abbott app)
│   + LibreLinkUp / API   │
└────────────┬────────────┘
             │  HTTP polling (Libre API)
             ▼
┌─────────────────────────┐
│   Supabase               │
│   - glucose_readings     │
│   - Edge Function /      │
│     scheduled sync job   │
└────────────┬────────────┘
             │  REST / Realtime
             ▼
┌──────────────────────────────────────┐
│   GlucoseSync iOS App (SwiftUI)      │
│   + WatchKit Extension (watchOS)     │
└──────────────────────────────────────┘
```

---

## Tech Stack

| Layer           | Technology                                                           |
| --------------- | -------------------------------------------------------------------- |
| iOS App         | Swift 5.9, SwiftUI, Swift Charts                                     |
| Watch App       | watchOS 10, SwiftUI, Digital Crown via `focusable` + `onCrownRotate` |
| Backend         | Supabase (Postgres + Edge Functions + Realtime)                      |
| Auth            | Supabase Auth (email or Apple Sign In)                               |
| Sync            | Supabase Edge Function calling Libre API on a cron schedule          |
| CGM Source (v1) | FreeStyle Libre 2 via informal LibreLinkUp API                       |

---

## Roadmap

- [x] Define architecture & data model
- [ ] Supabase schema + RLS policies
- [ ] Edge Function: Libre API poller (cron every 5 min)
- [ ] iOS app: authentication + glucose chart view
- [ ] iOS app: time-range selector (1h → 90d)
- [ ] watchOS app: current reading + Digital Crown time range
- [ ] watchOS complications
- [ ] HealthKit integration
- [ ] Support additional CGM sources (Dexcom G7, etc.)
- [ ] Alerting (local notifications for hypo/hyper events)
- [ ] Widget (iOS Lock Screen, Home Screen)

---

## Data Model

```sql
-- glucose_readings
create table glucose_readings (
  id           uuid primary key default gen_random_uuid(),
  user_id      uuid references auth.users not null,
  recorded_at  timestamptz not null,
  value_mgdl   numeric(6,2) not null,       -- stored in mg/dL, convert on display
  trend        text,                          -- 'rising_rapidly' | 'rising' | 'stable' | 'falling' | 'falling_rapidly'
  source       text default 'libre2',        -- future-proofing for multi-sensor
  raw          jsonb,                         -- original API payload
  created_at   timestamptz default now()
);

create index on glucose_readings (user_id, recorded_at desc);
```

---

## Getting Started

### Prerequisites

- Xcode 15+
- A Supabase project (free tier is fine)
- A LibreLinkUp account linked to your LibreLink app
- An iPhone (iOS 16+) paired with an Apple Watch (watchOS 10+)

### Setup

1. **Clone the repo**

   ```bash
   git clone https://github.com/YOUR_USERNAME/glucosesync.git
   cd glucosesync
   ```

2. **Configure Supabase**
   - Create a new Supabase project
   - Run the SQL in `supabase/migrations/` to create tables and policies
   - Copy `.env.example` to `.env` and fill in your Supabase URL + anon key
   - Deploy the Edge Function: `supabase functions deploy glucose-sync`

3. **Configure the iOS project**
   - Open `GlucoseSync.xcodeproj` in Xcode
   - Fill in your Supabase credentials in `Config.swift`
   - Set your team and bundle ID
   - Run on device (Watch simulator works for UI, real device needed for sensor data)

4. **Link LibreLinkUp**
   - In the LibreLink app, enable LibreLinkUp sharing
   - Enter your LibreLinkUp credentials in the GlucoseSync app settings
   - The Edge Function will start polling on the next cron tick (every 5 min)

---

## Privacy & Disclaimer

- All glucose data is stored in **your own Supabase project**. No data is sent to any third-party service other than Supabase and the official Abbott LibreLinkUp API.
- This app is a **personal tool**, not a medical device. It is not FDA-cleared or CE-marked. Do not use it to make clinical decisions. Always verify readings with your official device and consult your healthcare provider.
- Use of the LibreLinkUp API is subject to Abbott's terms of service.

---

## Contributing

This is a personal health project. PRs are welcome — especially for:

- Additional CGM sensor integrations
- Improved chart interactions
- Accessibility improvements
- Localization (mmol/L ↔ mg/dL)

---

## License

MIT — see [LICENSE](LICENSE)
