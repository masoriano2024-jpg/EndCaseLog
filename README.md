# EndCaseLog# DCM Tracker — Project Handoff Notes

## What this is
A custom Disaster Case Management (DCM) tool built for Endeavors, used for DR-4879 Texas Hill Country flood case management. Built and maintained by Mikey (case manager, Kerrville office). Originally a single-file localStorage app, now a multi-user cloud-backed tool on Supabase.

## Current file
`case-tracker-16.html` — single HTML file, plain JS/CSS, no build tools, no frameworks.

## Hosting / Deploy
- Hosted on **Cloudflare Pages**, auto-deploys from a **GitHub** repo on push.
- To deploy: commit the updated HTML file to the repo and push. No build step.

## Backend — Supabase
- Project URL: `https://gsqstjkyuyztvglutuvk.supabase.co`
- Anon public key is hardcoded in the HTML `<script>` block (search `SUPABASE_KEY`)
- **IMPORTANT:** Must load Supabase JS via the UMD build, NOT the default ESM CDN path, or `supabase.createClient` won't exist as a global:
  ```html
  <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.min.js"></script>
  ```

### Tables
| Table | Purpose |
|---|---|
| `cases` | Core case records — id (survivor #), status, step, dates, user_id, office |
| `contact_logs` | Every logged contact attempt per case |
| `roster` | Optional name-to-ID mapping (kept separate for privacy) |
| `audit_entries` | Saved self-audit checklist snapshots per case |
| `step_templates` | Editable Path A (Outreach) / Path B (Intake) step definitions |
| `weekly_report` | Weekly report rows grouped by Wednesday week-ending date |
| `case_notes` | Running timestamped notes log per case (internal notes, not contact logs) |
| `todos` | Personal to-do list per user (has `user_id`) |
| `client_todos` | To-do tasks tied to a specific case, with customizable `due_date` |
| `profiles` | Extends `auth.users` — role + office assignment per user |

### RLS Status
- RLS is **enabled** on most tables.
- `profiles`: read-all, self-update only.
- `cases`, `contact_logs`, `roster`, `audit_entries`, `case_notes`: role-based policies — admin/program_manager see all, supervisor sees their office only, case_manager sees only their own (`user_id` match).
- `todos`: `auth.uid() = user_id` only.
- `client_todos`: currently has a permissive `auth.role() = 'authenticated'` policy (any logged-in user can read/write any client task) — **not yet office/role-scoped**, just a stopgap fix when tasks weren't loading.
- `step_templates`: anyone can read, only `admin` role can write.
- `weekly_report`: admin/PM/supervisor full access, case_manager read-only.

### Known schema quirks
- `step_templates` had a recurring duplicate-row bug. Fixed by always doing `delete().gte('position',0)` before reinserting the full set — never use `.neq('id', some-fake-uuid)`, it doesn't reliably match all rows.
- `cases.user_id` and `cases.office` were added later via `alter table` — added on top of the original schema, not part of initial design.
- A Postgres trigger (`handle_new_user`) auto-creates a `profiles` row whenever a new `auth.users` row is created, pulling `full_name`, `role`, `office` out of signup metadata.

## Auth & Roles
- Full Supabase Auth email/password login. Login screen is built into the HTML (`#loginScreen` div), shown/hidden via JS based on session state.
- Four roles: `admin`, `program_manager`, `supervisor`, `case_manager`.
- Mikey's user: UUID `a8e29e6a-8ccd-4e2f-b11e-a27afbf019d9`, role `admin`, office `Kerrville`.
- Admin panel exists inside the app (Admin tab, only visible to `admin` role) — lets Mikey create new user accounts with name/email/password/role/office. Uses `sb.auth.signUp()` client-side (no service-role key in the browser, so this is the workaround instead of a true admin API call).
- Three offices: **Kerrville**, **San Angelo**, **Austin**.

## App structure / Tabs
- **Dashboard** — Daily Briefing (stat row, urgency alerts, client task due/overdue alerts, outreach step breakdown, intake progress bars) + active case cards.
- **Roster** — searchable client list, click to open a profile panel with case info, contact history, audit history, case notes, and open client tasks.
- **Weekly Report** — contact logs grouped by Wednesday week-ending date, CSV export.
- **The Steps** — admin-only, lets you edit the Path A (15-step Outreach) and Path B (7-step Intake) templates. Hidden from non-admin roles.
- **Self-Audit** — mirrors the official Endeavors/TCR compliance checklist (9 sections), saves snapshots, loads history from Supabase with localStorage fallback.
- **Notes** — personal to-do list, case notes (running log per case), and client to-do list (tasks tied to a case with due dates) all live here.
- **Admin** — admin-only. Create users, view/deactivate users.

## Two survivor workflow paths
- **Path A — Outreach**: 15 steps, no-contact cases.
- **Path B — Intake**: 7 steps, clients who accept services.
- Both funnel into **Open Case → Closed**.
- Case can also close as **Declined** (TCR-specific status) or no-contact closure.

## Design system
- Brand colors: deep navy (`#1B2A4A`), Endeavors blue (`#0077c8`/`#2563EB` depending on context).
- Fonts: Montserrat 800 for headers/labels, Open Sans for body text.
- Case cards use a left-border + gradient wash colored by status: red=overdue, amber=outreach/due-soon, blue=intake, green=open.
- Background is a darkened steel-blue gradient (not pure white/pale blue) so white cards stand out.
- Nav tabs are evenly spaced (`flex:1` each) across the full width of the navy header bar.

## What's NOT built yet (next steps)
1. **client_todos RLS** needs real office/role scoping — right now it's wide open to any authenticated user.
2. **Supervisor/Program Manager dashboards** — no dedicated cross-case view built yet beyond the existing role-based data filtering on load.
3. **PWA setup** — discussed but not implemented. Would let iPhone/Android users add it to their home screen as an app-like experience. Needs a manifest.json + service worker added to the HTML.
4. **True native iOS app** — discussed as the "hard path," not started. Supabase has a Swift SDK if this ever gets prioritized.
5. **Audit history** — fixed to read from Supabase with localStorage fallback, but double check this still works correctly after all the auth changes (it was built before RLS was turned on).
6. **client_todos** has no office assignment column — if/when this expands to other offices, will need an `office` column + RLS policy like `cases` has, not just `user_id`.

## Business context
- This is being proposed as an enterprise tool for all 3 Endeavors offices (~30 staff total: 25 case managers, 3 supervisors, 1 program manager, Mikey as admin).
- A formal cost/compensation proposal document was drafted (not in this repo, lives in Mikey's docs) — Supabase Pro tier ($25/mo) recommended once this goes to multiple offices for daily backups.
- Endeavors leadership has expressed interest in expanding this tool org-wide.

## Mikey's working style (for whoever picks this up)
- Builds iteratively, confirms each feature works before adding the next.
- Wants to preserve flexibility (e.g. "make sure we don't lose the ability to add steps" when fixing the step_templates bug).
- Prefers being told directly when something's a bad idea rather than being told what he wants to hear.
- Client privacy matters — cases are ID-only, names live in a separate `roster` table by design.
