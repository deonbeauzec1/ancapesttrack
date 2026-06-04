# PestTrack — Field Inspection PWA

A mobile-first Progressive Web App for pest control operators to log bait box and fly unit inspections in the field, with offline support and Supabase cloud sync.

---

## Files

```
pesttrack/
├── index.html          ← Full app (single file)
├── manifest.json       ← PWA install manifest
├── service-worker.js   ← Offline caching & background sync
├── icons/
│   ├── icon-192.png    ← App icon (create these — see below)
│   └── icon-512.png
└── README.md
```

---

## Deployment

### Option A — Netlify (recommended, free)
1. Drag the entire `pesttrack/` folder onto https://app.netlify.com/drop
2. Done — you get an HTTPS URL, which PWA requires.

### Option B — GitHub Pages
1. Push to a GitHub repo
2. Go to Settings → Pages → Deploy from branch (main / root)
3. Access at `https://yourusername.github.io/pesttrack/`

### Option C — Any static host
Upload all files to any HTTPS host. The app is 100% static — no server-side code required.

---

## App icons

Create two PNG icons and place them in the `icons/` folder:
- `icon-192.png` — 192×192px
- `icon-512.png` — 512×512px

You can use any icon editor or https://favicon.io to generate them. The app still works without them — they're only needed for the home screen install prompt.

---

## Supabase setup

### 1. Create a project
Sign up at https://supabase.com, create a new project.

### 2. Create the inspections table

In the Supabase SQL editor, run:

```sql
create table pest_inspections (
  id                text primary key,
  unit_id           text,
  unit_label        text,
  unit_type         text,
  expected_barcode  text,
  scanned_barcode   text,
  barcode_match     boolean,
  site_id           text,
  site_name         text,
  location_id       text,
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

-- Allow anonymous inserts (for the field app)
alter table pest_inspections enable row level security;

create policy "Allow anon inserts"
  on pest_inspections for insert
  to anon
  with check (true);

create policy "Allow anon reads"
  on pest_inspections for select
  to anon
  using (true);
```

### 3. Get your credentials
- Go to Project Settings → API
- Copy **Project URL** (e.g. `https://xxxx.supabase.co`)
- Copy **anon public** key

### 4. Configure the app
Open the app → tap **Admin / setup** → **Config** tab → enter URL and key → **Save configuration** → **Test connection**.

---

## How the sync works

1. Every inspection is saved immediately to `localStorage` (works fully offline).
2. If the device is online when saving, the record is automatically posted to Supabase.
3. If offline, the record joins the **pending queue**.
4. When the device comes back online, a sync banner appears — tap **Sync** to upload all pending records.
5. The service worker also handles background sync events for automatic re-try.

---

## Features

- **Sign-in**: Select or type technician name at shift start
- **Site → Location → Unit** hierarchy
- **Per unit**: barcode scan (camera), inspected, condition, bait applied + batch number (bait boxes) or catch count (fly units), notes
- **Barcode scanning**: Uses native `BarcodeDetector` API on supported Android/Chrome, falls back to ZXing camera library
- **Offline-first**: All data in localStorage, sync queue, service worker cache
- **Admin panel**: Add/remove sites, locations, units, technicians; Supabase config; CSV/JSON export
- **Install as app**: PWA install prompt on Android Chrome — works like a native app

---

## Admin default credentials

There's no password on the admin screen by default (it's accessed from the login screen). For production, consider adding a PIN check before the admin screen.
