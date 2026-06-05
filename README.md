# PestTrack — Field Inspection PWA

## File overview

| File | Purpose |
|---|---|
| `index.html` | **Field app** — technicians use this on Android devices |
| `admin.html` | **Admin dashboard** — supervisor portal to manage everything |
| `manifest.json` | PWA install manifest |
| `service-worker.js` | Offline caching & background sync |
| `icons/` | App icons (192px and 512px) |

---

## How it works

- **Sites, locations, units and technicians** all live in Supabase.
- The **admin dashboard** (`admin.html`) is the single place to manage all of them.
- **Field devices** fetch the latest config from Supabase on load and cache it for offline use. There is no admin screen on the field app.
- Inspection records save to localStorage immediately (offline-first), then sync to Supabase automatically when online.

---

## Supabase setup (run once)

In your Supabase SQL editor, run this full schema:

```sql
-- Lookup tables
create table pt_sites (
  id         uuid primary key default gen_random_uuid(),
  name       text not null,
  address    text,
  created_at timestamptz default now()
);

create table pt_locations (
  id         uuid primary key default gen_random_uuid(),
  site_id    uuid references pt_sites(id) on delete cascade,
  name       text not null,
  icon       text default 'ti-map-pin',
  created_at timestamptz default now()
);

create table pt_units (
  id          uuid primary key default gen_random_uuid(),
  location_id uuid references pt_locations(id) on delete cascade,
  label       text not null,
  barcode     text not null,
  unit_type   text not null check (unit_type in ('bait','fly')),
  created_at  timestamptz default now()
);

create table pt_technicians (
  id         uuid primary key default gen_random_uuid(),
  name       text not null,
  initials   text not null,
  created_at timestamptz default now()
);

-- Inspection records (written by field app)
create table pest_inspections (
  id                text primary key,
  unit_id           uuid,
  unit_label        text,
  unit_type         text,
  expected_barcode  text,
  scanned_barcode   text,
  barcode_match     boolean,
  site_id           uuid,
  site_name         text,
  location_id       uuid,
  location_name     text,
  tech              text,
  tech_date         date,
  timestamp         timestamptz,
  inspected         text,
  condition         text,
  bait_applied      text,
  batch_number      text,
  catch_count       integer,
  notes             text,
  synced            boolean default false,
  created_at        timestamptz default now()
);

-- RLS: anon key can read lookup tables and insert inspections
alter table pt_sites          enable row level security;
alter table pt_locations       enable row level security;
alter table pt_units           enable row level security;
alter table pt_technicians     enable row level security;
alter table pest_inspections   enable row level security;

create policy "anon read sites"       on pt_sites        for select to anon using (true);
create policy "anon read locations"   on pt_locations    for select to anon using (true);
create policy "anon read units"       on pt_units        for select to anon using (true);
create policy "anon read techs"       on pt_technicians  for select to anon using (true);
create policy "anon insert inspections" on pest_inspections for insert to anon with check (true);
create policy "anon read inspections" on pest_inspections for select to anon using (true);
-- service role key bypasses RLS automatically (used by admin dashboard)
```

---

## Admin dashboard setup

1. Open `admin.html` in a browser (can be local — no server needed).
2. Enter your **Supabase project URL** + **service role key** (Project Settings → API).
3. On first login, set an admin password — it's stored as a SHA-256 hash in your browser.
4. Use the **Sites & Units** tab to build your site hierarchy.
5. Use the **Technicians** tab to add your team.

> **Important:** The service role key has full database access. Never put it in `index.html` or on any field device. Use it only in the admin dashboard, which runs in the supervisor's browser.

---

## Field app setup

### Option A — Pre-bake credentials (recommended)
Edit the top of `index.html` before distributing:
```js
const PREBAKED_URL = 'https://xxxx.supabase.co';
const PREBAKED_KEY = 'eyJhbGci…'; // anon key only — NOT service role
```
The Settings tab in the admin dashboard generates this snippet for you to copy.

Then host `index.html`, `manifest.json`, `service-worker.js` and `icons/` on any HTTPS host (Netlify, GitHub Pages, etc.) and send the URL to technicians.

### Option B — Distribute the URL
Leave `PREBAKED_URL` and `PREBAKED_KEY` blank. On first launch, the app shows a "contact your supervisor" message. Supervisor opens the app's browser, enters the URL in the address bar.

---

## Deploying

Drag the `pesttrack/` folder to **https://app.netlify.com/drop** — done in ~30 seconds. Field devices visit the HTTPS URL and tap "Add to home screen" in Chrome to install as a PWA.

---

## Installing on Android

1. Open the hosted URL in Chrome.
2. Tap the menu (⋮) → "Add to Home screen".
3. App installs as a standalone app — no Play Store needed.
4. Works fully offline after first load.
