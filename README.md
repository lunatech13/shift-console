# PSTCC Media Technologies — Shift Console

A single-file web app for scheduling Fall 2026 work-study students in Media
Technologies. Given each student's stated availability, a policy of how the
department's fall hours budget gets split between them, and a few policy
knobs (weekly cap, minimum shift length, max concurrent workers, department
open hours), it computes and displays a weekly schedule — with support for
students who work at a different physical location, students with an
externally fixed (non-negotiable) schedule, and per-student class /
blocked-off time tracking.

Live site: `pstcc-shift-console.netlify.app` — fill in once you know the exact
Netlify domain.

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

## Student pay types: SEP vs. FWS

Each student on the roster is flagged **SEP** (Student Employment Program)
or **FWS** (Federal Work-Study), set on the Roster tab. The two draw from
completely separate pots of money and are scheduled differently:

- **SEP students** split the shared Fall/Spring hours budget (see below),
  proportionally by grade weight, same as the app has always done.
- **FWS students** are paid from a different budget entirely and never touch
  the SEP pool. Instead, each FWS student gets their own **semester award
  ($)**, entered on the Roster tab, which — divided by their hourly wage and
  the usable-weeks count — determines how many hours a week they can work.
  Their hours don't reduce what's left for SEP students, and changing an FWS
  student's award doesn't change anyone else's schedule.

## Budget: annual award, fall/spring split

The Budget panel on the landing page tracks the department's **hours award
for the fiscal year (fall + spring combined)** — the fixed ceiling handed
down from the department — separately from how much of it is committed to
**fall**:

- **Annual award (fiscal year, h)** — the full-year ceiling (defaults to
  1750h). This number rarely changes once it's confirmed.
- **Fall hours budget** — how much of that annual award fall is using. This
  can be entered two ways, toggled with the "Budget entry" dropdown:
  - **Enter hours directly** — a plain hours number (defaults to 390h).
    Handy while the exact fall number is still being worked out.
  - **Enter $ award ÷ hourly rate** — a semester dollar award divided by an
    hourly rate, which keeps the hours figure traceable back to an actual
    award letter.
- **Spring hours remaining** — computed automatically as Annual award minus
  Fall hours budget, so it's always up to date as either number changes. If
  fall's budget is ever set higher than the annual award, a warning banner
  says so (nothing would be left for spring).

This budget (in whichever mode) only governs the shared SEP pool — FWS
students' hours come from their own semester award and aren't part of this
calculation.

## Landing page vs. Roster tab

The landing page's **Grades** area just lists each student's name and a
grade slider — that's what drives the SEP proportional split. Everything
else about a student (availability, hourly wage, physical location, fixed
schedule, class blocks, SEP/FWS flag, FWS semester award) lives on the
**Roster tab**, along with the "Add Student" intake form. Availability for
a given student can be set to **Auto** (available whenever they're not in a
listed class/internship/job block) or entered manually block by block.

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
roster (`students`, each with availability/classes/location/fixed-schedule/
SEP-or-FWS/semester-award info), `grades`, `wages`, `knownLocations`, and
the console's `settings` (annual award, fall budget mode/hours/$-award/rate,
usable weeks, weekly cap, min shift, max concurrent, open/close hours).
Loading the app fetches this one row; saving overwrites it.

## Access model

- **Anyone with the link can open the page and view/interact with it.**
  Dragging any slider recomputes the schedule live in their browser — none
  of that touches the database, so it's safe for colleagues (or students)
  to explore without risk of overwriting the saved schedule.
- **By default, only the admin account can save.** Sign-in uses Supabase
  Auth's email magic-link flow (implemented with plain `fetch()` calls
  against Supabase's `/auth/v1/otp` and `/auth/v1/token` endpoints —
  deliberately not the `@supabase/supabase-js` library, to keep this a
  zero-dependency single file). The session is kept in the browser's
  `localStorage`.
- **Temporary write access for other people, without touching the file,**
  is controlled by a second table, `app_access` (one row, `id = 1`):

  ```sql
  create table if not exists app_access (
    id int primary key default 1,
    open_write boolean not null default false,
    extra_writers text[] not null default '{}',
    public_write boolean not null default false
  );
  insert into app_access (id) values (1) on conflict (id) do nothing;

  alter table app_access enable row level security;

  create policy "anyone read access config" on app_access
    for select using (true);

  create policy "admin manage access config" on app_access
    for all
    using (auth.jwt() ->> 'email' = 'paulwise@me.com')
    with check (auth.jwt() ->> 'email' = 'paulwise@me.com');
  ```

  - `open_write` + `extra_writers`: specific pre-approved emails may sign in
    (magic link, same as the admin) and save. Flip `open_write` off to
    revert instantly without editing the list.
  - `public_write`: bypasses sign-in entirely — anyone with the link can
    save, no auth at all. Grade/wage/payroll masking also lifts while this
    is on, since an editor needs to see what they're editing. Flip off the
    same way.
  - Only the admin can change any of these three values — enforced by the
    RLS policy above, not just by hiding the controls in the UI.
  - This table fails closed: if it can't be loaded (missing, or a network
    hiccup), the app behaves as if all three are off and only the admin can
    save.

- **The actual save enforcement is server-side**, via Postgres row-level
  security on `app_state` — not just a hidden button in the UI. The admin
  is always allowed; `extra_writers` (while `open_write` is on) are allowed
  by re-checking their email the same way; `public_write` needs a matching
  permissive policy if it's ever turned on for `app_state` itself:

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
  the SQL above (run in Supabase's SQL Editor, for both `app_state` and
  `app_access`) and the `ADMIN_EMAIL` constant near the top of the
  `<script>` block in `index.html`.

- **Grade, wage, and payroll figures are hidden from anyone not signed in
  with write access.** This is a UI-level mask (CSS classes toggled by a
  flag set once the current session's access is checked) — the raw numbers
  still load into the browser's memory either way, because the scheduling
  algorithm needs real grade weights and wages to compute correct hours for
  *anyone* watching it. That means this hides the numbers from ordinary
  viewing, but not from someone technical enough to open browser dev tools
  and inspect the page's JS state or network responses. If that residual
  risk ever matters more than it does today, the real fix is splitting
  `grades`/`wages` into a second table with `select` also restricted by RLS
  (same pattern as the write policies above) — a bigger change, since it
  means non-privileged viewers would need to work from a precomputed
  schedule snapshot rather than recomputing it live.

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
- **There's no separate spring-semester budget/schedule of its own yet** —
  "Spring hours remaining" is just what's left of the annual award after
  fall, shown as a planning number. If a full second scheduling pass for
  spring is wanted later, that would be a new feature built the same way
  fall's was.
