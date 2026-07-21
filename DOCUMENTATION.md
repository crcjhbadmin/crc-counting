# CRC JHB Steward — Documentation

Last updated: 21 July 2026

## What this application is

Steward is CRC Johannesburg's internal operations app. It is a single static page (`index.html`) that talks directly to a Supabase project (database + auth + file storage) — there is no separate backend server. It is deployed on Vercel and hosted at:

https://crc-counting-crc-jhb.vercel.app/

It currently has three modules: **Counters** (Sunday attendance counting), **Venue Bookings**, and **Events** (vendor management for Dreamweek and future CRC events). All three share one login system and one database.

---

## Architecture, for developers

- **Frontend:** one HTML file, vanilla JavaScript (no framework, no build step). Routing is done by hand with `location.hash` (`#/count`, `#/venues`, `#/events`, `#/dashboard`, `#/totals`, `#/signin`) and a single `route()` dispatcher that re-renders `#app` on every hash change.
- **Backend:** Supabase (Postgres + Auth + Storage). The browser talks to Supabase directly using the publishable ("anon") key embedded in the page — this is safe only because every table has Row Level Security (RLS) enabled, so the database itself enforces who can read or write what. There is no separate API layer to bypass.
- **Auth:** Supabase Auth, but disguised as a simple "login name + 6-digit PIN" for staff. Internally, the login name is turned into a fake email (`name@steward.crcjhb.app`) and the PIN is the password. New staff accounts are created on the backend (ask the admin) — there is no self-service staff signup.
- **Roles:** every signed-in staff member has a row in `profiles` with a `role` of `counter`, `coordinator`, or `admin`. A shared helper function in the database, `is_staff()`, treats `coordinator` and `admin` as "staff" for permission checks; plain `counter` accounts can only do counting.
- **Vendor applicants** (Events module) do **not** get staff accounts. Applications are submitted without login, the same way public venue bookings already work in this app — this deliberately avoids creating a login for every vendor, which would otherwise pollute the same `profiles` table used for Sunday Counting's counter lists.
- **File storage:** one Supabase Storage bucket, `photos`, used for counting-tally photos and (new) vendor event venue-layout images, under different path prefixes.
- **Deployment:** the repo (`crcjhbadmin/crc-counting` on GitHub) is connected to Vercel. Any commit to `main` triggers an automatic redeploy — there is no separate staging environment.
- **Global navigation:** a small fixed element in the bottom-left corner (`Dashboard` / `Log Out`) is present on every screen. "Dashboard" always returns to the module-switcher home screen (`#/`). "Log Out" only appears when signed in.

### Known technical debt / limitations for developers
- Single 90KB+ HTML file — there is no module bundler, so every change is a manual find-and-edit. This keeps the app dependency-free and fast, but means large features (like this Events module) add real bulk to one file.
- No automated test suite. Verification is manual (syntax check + click-through) after each change.
- No staging environment — commits to `main` go straight to production.
- The `photos` storage bucket currently has an open read/write policy (any request, even unauthenticated, can read or write objects in it) predating this work. Low risk today since paths are unguessable, but worth tightening if the app starts storing anything sensitive.

---

## Module 1: Counters (Sunday attendance counting)

**Purpose:** record attendance for each Sunday service, block by block, with a two-counter cross-check.

**Roles:**
- `counter` — can count only the blocks/rooms they're assigned to for the current open service.
- `coordinator` / `admin` ("staff") — everything a counter can do, plus: open/pause/close services, assign counters to blocks, reconcile mismatches, view live totals, run reports, manage the team list.

**Features:**
- **Count entry** (`#/count`): a counter sees only their assigned blocks for the currently open service, enters one number per row (how many people are present), and can attach a photo of their paper tally. A small schematic map shows every auditorium section and highlights the one(s) assigned to that counter, so it's obvious where to count regardless of how the campus names its sections (Left/Centre/Right, Front/Back, or anything else — it's driven entirely by whatever section names are configured, nothing is hardcoded).
- **Two-counter reconciliation:** each block is normally assigned two counters. If their totals differ by more than 5, the coordinator sees a row-level comparison and must ask for a recheck; small differences (≤5) can be accepted as an average.
- **Service management** (staff): open a service for a date/period, pause/resume/close it, edit its date or label, reopen a closed one to correct it.
- **Assignments** (staff): assign up to two counters per block per service, or set default assignments that auto-apply to newly opened services.
- **Reports:** per-service report with capacity/occupancy, CSV export, print-to-PDF, and week-over-week / year-over-year comparisons.
- **Public totals view** (`#/totals`, and the "Pastor view" link on the home screen): live attendance and trend charts, no login required.

**What changed (this update):**
- "Empty seats" tracking was removed from the counting workflow. Counters now enter one number — people present — rather than seated + empty. The capacity cross-check ("seated X, empty Y, total Z vs capacity") that depended on that second number is gone from the entry screen. Historical empty-seat data already in the database was left untouched; reports still show a capacity-derived estimate (capacity minus seated) where useful.
- The two-counter reconciliation check is unaffected — it compares seated counts between counters, not empty seats, so it still works exactly as before.
- All references to the Venue Bookings module were removed from inside Counters (the "Venues" tab that used to sit inside the counting dashboard, and quick-links to venue bookings from the counting screen). Venue Bookings is now only reachable from the main dashboard.

**Current limitations:**
- The section map is schematic (a labelled grid, not a photographed floor plan). A real floor-plan image with clickable hotspots would need new admin tooling to upload and position it — noted as a future enhancement.
- No offline mode — counters need connectivity to submit.
- No undo on a submitted count beyond re-submitting a new attempt.

---

## Module 2: Venue Bookings

**Purpose:** book church venues/rooms for events, run a setup checklist, and track physical key sign-outs.

**Roles:** open to the whole team, no login required, by design (matches how bookings happen in practice — someone at the front desk or a ministry leader books a room). Staff (`coordinator`/`admin`) additionally see and can edit the "function types" and checklist templates used across all bookings.

**Features:**
- Book a venue for a date/time range, with expected attendance, sound/visual needs, and a function type (e.g. wedding, baby dedication).
- Each booking gets a checklist auto-filled from that function type's template; anyone can tick items off.
- Approve/decline a booking, see venue availability.
- Sign a physical key in/out per venue, with who took it and when it was returned.
- Manage function types and their checklist templates.

**Current limitations:**
- Booking read/update access is intentionally very open (`true` policies) so the whole team can coordinate without individual logins — this is a deliberate trade-off for usability, not an oversight, but it does mean anyone with the link can edit any booking.
- No conflict detection beyond what's visible on the availability view — double-booking is possible if two people book at the same time.

---

## Module 3: Events (Dreamweek & future CRC events — vendor management)

**Purpose:** replace the Microsoft Form vendor application process with a native, reusable vendor management workflow. Built generically so any future CRC event (Colour Conference, Easter, Christmas, Youth Conference, etc.) can reuse it — an admin just creates a new event and configures categories, no code changes needed.

**Roles:**
- Public / vendors — no login. Anyone can browse events currently open for applications and submit an application.
- Staff (`coordinator`/`admin`) — create events, configure vendor categories, review and approve/decline applications, upload a venue layout image, and assign approved vendors to a stand/location.

**Features (current phase):**
- **Event setup:** name, dates, vendor fee, status (`draft` → `accepting applications` → `closed` → `active` → `completed` → `archived`). Nothing is hardcoded to Dreamweek — any event lives in the same table.
- **Vendor categories:** configurable per event, each with a maximum number of vendors (matches the "2–3 vendors per category" rule from the Dreamweek vendor pack), shown to applicants when they apply.
- **Public application form:** company/stall name, contact person, phone, email, category, setup type, parking spaces (1 or 2), team size, product/service description, power/water/storage notes. Submits directly into the database — no Microsoft Form, no spreadsheet.
- **Review workflow:** every application has a status (submitted → under review → approved → sample testing → walkthrough → cleared to trade, or declined/withdrawn/cancelled at any point). Staff have quick Approve/Decline buttons for the common case, plus a full status dropdown for the rest of the lifecycle. Every status change is written to an audit history table.
- **Venue layout:** staff can upload a layout image for the event once vendors start being approved; it's stored privately and shown in the admin view via a short-lived signed link.
- **Stall/stand allocation:** once an application is approved (or further along), staff can record its zone, position label, and parking spaces — a simple, event-scoped way to say "you're in Row A, Bay 3."

**Backend already built, no screens yet (safe to add later without restructuring):**
- Internal notes per application (staff-only, never vendor-visible).
- Vendor communications log (for templated, auditable Outlook email — table exists, sending isn't wired up yet).
- Payment tracking (R700 fee, method, reference, status).
- Compliance document tracking (type, status, expiry).
- Sample testing reviews and final walkthrough inspections, each with pass/fail/corrective-action fields matching the vendor pack's checklist.

**Designed for future phases (per the roadmap requested):**
- *Vendor profiles / accounts:* deliberately not built yet, because giving vendors real login accounts would need a separate identity path from staff `profiles` (to avoid vendors showing up in Sunday Counting's counter lists). Each application already has a private `access_token` reserved for a future "check my application status" magic link, so this can be added without changing the applications table.
- *Vendor check-in during the event:* would read the same `vendor_applications` + `vendor_stall_allocations` tables; no schema change anticipated, just a new check-in screen and maybe a timestamp column.
- *Vendor communication:* the `vendor_communications` table already logs channel, template, subject/body, and an external message ID — an Outlook-sending edge function can write into it without any table changes.
- *Session attendance visibility for vendors:* Counters already writes accepted totals to `section_results` / `attendance_history`. A future vendor-facing "expected traffic" view would read from there — no new counting-side changes needed, just a new read-only query and screen in Events.

**Current limitations:**
- No email/WhatsApp is actually sent yet on approval/decline — the log table exists, sending doesn't.
- No payments, compliance docs, sample testing or walkthrough screens yet — data model is ready, UI isn't built.
- The venue layout is one image per event (not a zoomable/annotated map with per-vendor pins).
- Category vendor limits are shown for reference but not yet enforced automatically when approving (an admin could approve more than the stated maximum; nothing currently blocks it).

---

## Roles & permissions summary

| Role | Counters | Venue Bookings | Events (admin) | Events (apply) |
|---|---|---|---|---|
| Public (no login) | View public totals only | Full booking access | Apply, browse open events | Apply |
| `counter` | Count assigned blocks | Full booking access | — | — |
| `coordinator` | Full Counters admin | Full booking access | Full | — |
| `admin` | Full Counters admin | Full booking access | Full | — |

---

## High-level workflows

**Sunday morning (Counting):** coordinator opens the service → counters count their assigned blocks (guided by the section map) → coordinator watches the live dashboard, reconciles any mismatches over 5 → coordinator closes the service, which saves the auditorium total to attendance history.

**Booking a venue:** anyone requests a booking with a function type → staff confirms or declines → the auto-generated checklist gets ticked off as setup happens → whoever needs the key signs it out and back in.

**Vendor application (Dreamweek, or any future event):** admin creates the event and its categories, sets status to "accepting applications" → vendor applies with no login → staff review, approve or decline (with quick buttons) → once approved, staff can upload the venue layout and allocate a stand → (next phase) staff run sample testing and a final walkthrough before the vendor is cleared to trade.
