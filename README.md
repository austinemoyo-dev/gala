# Gala Night Gate Pass

A lightweight, offline-capable PWA for checking students in at the
Vision University Student Week Gala Night. Gate staff search by name
or matric number, mark students as entered, and everything syncs live
across devices via Supabase.

## 1. Create the Supabase table

In your Supabase project, open the **SQL editor** and run:

```sql
create table students (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  matric text,
  status text not null default 'paid',
  entered_at timestamptz,
  created_at timestamptz not null default now()
);

-- status is either 'paid' or 'entered'
```

This is a low-stakes event app, so Row Level Security is left
**disabled** and the app uses the anon key directly. Do not store
sensitive data in this table.

## 2. Enable Realtime

1. In the Supabase dashboard, go to **Database → Replication**
   (or **Table Editor → students → ... → Realtime**).
2. Enable Realtime for the `students` table (toggle on **Insert**,
   **Update**, and **Delete** events).

This lets a check-in on one device show up live on every other
device at the gate.

## 3. Fill in the CONFIG block

Open `index.html` and edit the `CONFIG` object near the top of the
`<script>` section:

```js
const CONFIG = {
  SUPABASE_URL: 'https://YOUR_PROJECT.supabase.co',
  SUPABASE_ANON_KEY: 'YOUR_ANON_KEY',
  ADMIN_PIN: '4321',
  EVENT_NAME: 'Gala Night 2026',
  TOTAL_LABEL: 'Students'
};
```

- `SUPABASE_URL` / `SUPABASE_ANON_KEY` — from your Supabase project's
  **Settings → API** page.
- `ADMIN_PIN` — 4-digit PIN required to access the Upload screen.
- `EVENT_NAME` — shown in the header (e.g. "Gala Night 2026").
- `TOTAL_LABEL` — label used for the total count.

## 4. Deploy to GitHub Pages

1. Push this folder (`index.html`, `manifest.json`, `sw.js`,
   `icons/`) to a GitHub repository.
2. In the repo, go to **Settings → Pages**.
3. Under **Source**, choose the branch (e.g. `main`) and root folder
   (`/`).
4. Save — GitHub will publish the app at
   `https://<your-username>.github.io/<repo-name>/`.
5. Open that URL on gate staff phones and use **Add to Home Screen**
   to install it as a PWA (works offline after first load).

> Note: GitHub Pages serves over HTTPS, which is required for the
> service worker and Supabase Realtime (WebSocket) to work correctly.

## 5. CSV format for the upload

The Upload screen (PIN-protected) accepts a `.csv`, `.xlsx`, or `.xls`
file. Column headers are matched case-insensitively, so any of these
work: `Name` / `name`, `Matric Number` / `Matric No` / `matric`.

Example CSV to hand to the accounting department for the initial
list:

```csv
Name,Matric Number
Adeola Johnson,VU/2023/00123
Bisi Adeyemi,VU/2022/00456
Chinedu Okafor,VU/2024/00789
```

- Uploading a new file **clears the existing student list** and
  replaces it with the new one — only do this once before the event,
  or if you need to start over.
- The "Not Yet Entered" export on the Summary screen produces a CSV
  in the same `Name,Matric Number` format, useful for handing back to
  accounting after the event.

## How it works (quick overview)

- **Gate screen** (default): live search, mark/unmark entry, online
  status badge, pending-sync badge.
- **Summary screen**: entered vs. not-entered counts and lists, with
  CSV export of the not-entered list.
- **Upload screen** (PIN-protected): drag-and-drop or pick a CSV/Excel
  file, preview the first 10 rows, then bulk-upload to Supabase.
- **Offline mode**: the student list is cached in `localStorage`
  (`gala_students`). Mark/unmark actions update the cache instantly
  and queue (`gala_sync_queue`) any Supabase update that fails,
  retrying every 30 seconds and whenever the device comes back online.
- **Realtime**: Supabase Realtime pushes updates to every connected
  device so the entered/total counter and student statuses stay in
  sync live.
