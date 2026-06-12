# Gala Night Gate Pass — Full Spec

## Purpose
A lightweight web app for Vision University Student Week Gala Night.
Gate staff check in paid students by searching their name/matric,
marking them as entered. Works across multiple devices via Supabase.
Falls back to offline mode with sync queue on network failure.

## Supabase Schema

### Table: students
| Column      | Type        | Default                    |
|-------------|-------------|----------------------------|
| id          | uuid        | gen_random_uuid()          |
| name        | text        | NOT NULL                   |
| matric      | text        | nullable                   |
| status      | text        | 'paid'                     |
| entered_at  | timestamptz | null                       |
| created_at  | timestamptz | now()                      |

status values: 'paid' | 'entered'

### Supabase Config
- Enable Realtime on students table
- RLS: disabled (anon key access, low-stakes event app)

## App Screens

### 1. Upload Screen (PIN protected)
- PIN entry gate (4-digit, configurable in config block)
- Drag & drop OR file picker for CSV or Excel (.xlsx/.xls)
- PapaParse for CSV, SheetJS for Excel
- Expected columns: Name, Matric Number (case-insensitive header match)
- Preview table: first 10 rows + total count
- "Confirm & Upload" button → bulk insert to Supabase
- Clears existing students table before insert (fresh upload)
- Shows progress bar during upload

### 2. Gate Screen (default/home screen)
- Header: "Gala Night 2025 🎉" + live counter "87 / 142 Entered"
- Online/Offline status badge (top right)
- Search bar: real-time filter, searches name + matric, 
  case-insensitive, trims whitespace
- Results list: shows matched students
  - Each card: Name (bold), Matric, Status badge
  - If status = 'paid': green "Mark as Entered" button
  - If status = 'entered': grey "✅ Entered" + timestamp
  - Unmark button (small, requires confirm) for mistakes
- Empty state: "No student found" with name shown

### 3. Summary Screen
- Tab or button to switch to summary
- Two sections: Entered (green) | Not Yet Entered (amber)
- Export "Not Entered" list as CSV button
- Total stats: entered count, remaining count

## Offline Strategy
- On app load: fetch full student list → cache in localStorage key 'gala_students'
- On mark action:
  1. Update localStorage immediately (instant UI feedback)
  2. Attempt Supabase update
  3. If fails: push to localStorage queue 'gala_sync_queue'
     { id, status, entered_at, action: 'mark' | 'unmark' }
- Sync queue:
  - Retry every 30 seconds
  - Retry on window online event
  - Clear queue items on successful sync
- Offline badge shown when navigator.onLine = false
- Realtime subscription: auto-reconnect on network restore

## Realtime Sync
- Subscribe to Supabase Realtime on students table
- On UPDATE event: patch local cache + update UI
- This means Device A marking student reflects on Device B live

## PWA
- manifest.json: name, short_name, icons, display: standalone
- sw.js: cache-first for app shell (index.html, manifest)
- Install prompt handled gracefully

## Config Block (top of index.html)
const CONFIG = {
  SUPABASE_URL: 'YOUR_URL',
  SUPABASE_ANON_KEY: 'YOUR_KEY',
  ADMIN_PIN: '4321',
  EVENT_NAME: 'Gala Night 2026',
  TOTAL_LABEL: 'Students'
}

## Design
- Dark theme: #0a0a0a background
- Gold accent: #c9a84c (matching VUSRC awards aesthetic)
- Clean, high contrast — usable in dim event lighting
- Large touch targets (min 48px) — gate staff using phones
- Search bar always visible, auto-focused on gate screen