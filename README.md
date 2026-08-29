# PSTCC Media Technologies — Shift Console

A single-file web app for scheduling Fall 2026 work-study students in Media
Technologies. Given each student's stated availability, a grade-weighted
proportional allocation of the department's fall hours budget, and a few
policy knobs (weekly cap, minimum shift length, max concurrent workers,
department open hours), it computes and displays a weekly schedule — with
support for students who work at a different physical location, students
with an externally fixed (non-negotiable) schedule, and per-student class /
blocked-off time tracking.

Live site: https://pstcc-shift-console.netlify.app/

## How it's built

- **`index.html`** is the entire application — markup, styles, and
  JavaScript in one file. No build step, no npm dependencies, nothing to
  install. It's deployed as a static file.
- **Netlify** hosts the static file and (once the GitHub repo is linked
  under Project configuration → Build & deploy → Continuous deployment)
  redeploys automatically whenever `index.html` changes on the `main`
  branch.
- **Supabase** (hosted Postgres + auto-generated REST API, plus its Auth
  service) is the backend. There is no separate server — the page talks to
  Supabase directly from the browser using `fetch()`.

## Data model

One table, one row:

```sql
create table if not exists app_state (
  id int primary key default 1,
  data jsonb not null,
  updated_at timestamptz not null default now()
);
```

`data` holds the entire saved application state as one JSON document:
roster (`students`, each with availability/classes/location/fixed-schedule
info), `grades`, `wages`, `knownLocations`, and the console's `settings`
(budget, usable weeks, weekly cap, min shift, max concurrent, open/close
hours). Loading the app fetches this one row; saving overwrites it.

## Access model

- **Anyone with the link can open the page and view/interact with it.**
  Dragging any slider recomputes the schedule live in their browser — none
  of that touches the database, so it's safe for colleagues (or students)
  to explore without risk of overwriting the saved schedule.
- **Only the admin account can save.** Sign-in uses Supabase Auth's email
  magic-link flow (implemented with plain `fetch()` calls against
  Supabase's `/auth/v1/otp` and `/auth/v1/token` endpoints — deliberately
  not the `@supabase/supabase-js` library, to keep this a zero-dependency
  single file). The session is kept in the browser's `localStorage`.
- **The actual enforcement is server-side**, via Postgres row-level
  security — not just a hidden button in the UI:

  ```sql
  alter table app_state enable row level security;

  create policy "anyone read" on app_state
    for select using (true);

  create policy "admin write" on app_state
    for insert
    with check (auth.jwt() ->> 'email' = 'paulwise@me.com');

  create policy "admin update" on app_state
    for update
    using (auth.jwt() ->> 'email' = 'paulwise@me.com')
    with check (auth.jwt() ->> 'email' = 'paulwise@me.com');
  ```

  If the admin email ever needs to change, it has to be updated in **two**
  places that must match exactly, or the admin gets locked out of saving:
  the SQL above (run in Supabase's SQL Editor) and the `ADMIN_EMAIL`
  constant near the top of the `<script>` block in `index.html`.

- **Grade, wage, and payroll figures are hidden from anyone not signed in
  as the admin.** This is a UI-level mask (CSS classes toggled by a
  `document.body.classList` flag, set once the signed-in session's email is
  checked against `ADMIN_EMAIL`) — the raw numbers still load into the
  browser's memory either way, because the scheduling algorithm needs real
  grade weights to compute correct hours for *anyone* watching it. That
  means this hides the numbers from ordinary viewing, but not from someone
  technical enough to open browser dev tools and inspect the page's JS
  state or network responses. If that residual risk ever matters more than
  it does today, the real fix is splitting `grades`/`wages` into a second
  table with `select` also restricted to the admin email by RLS (same
  pattern as the write policies above) — a bigger change, since it means
  non-admin viewers would need to work from a precomputed schedule snapshot
  rather than recomputing it live.

## Updating the live site

1. Get the new `index.html` (from this Claude conversation, or wherever
   it's being edited).
2. In the GitHub repo, replace `index.html` with the new version and commit
   (GitHub's web UI supports drag-and-drop uploads — no local git required).
3. Netlify picks up the push and redeploys automatically, usually within
   about 30 seconds. No manual deploy step.

## Known limitations

- **No automatic database backups.** Point-in-time recovery / scheduled
  backups on Supabase are typically a paid-tier feature. If you want your
  own copy of the data, Supabase's Table Editor can export the `app_state`
  row as JSON in a few seconds — worth doing occasionally.
- **Free-tier Supabase projects can pause themselves after a stretch of
  inactivity**, which would make the site fail to load/save until the
  project is resumed from the Supabase dashboard.
- **The grade/wage masking is UI-level, not a hard security boundary** —
  see the note above.
