# Direkt Developer App — Phase 1 Spec

## What is this

The Developer App is a B2B portal where developer salesmen log in to see and manage buyer activity on their units. It's one of Direkt's 3 platforms — alongside the Buyer App (anonymous, consumer-facing) and the Direkt Inventory Platform (central data hub/admin).

**Phase 1 scope:** Authentication, dashboard, client activity feed, lead management, read-only unit portfolio, buyer communication. **No inventory uploading** (that's Phase 2).

---

## Existing infrastructure (already built)

The `direkt-production` Supabase project (`beizrkeqlpvzfxgoejku`) already has working tables from an earlier prototype:

### Tables that exist:
- **`developers`** — developer companies (`id`, `name`, `created_at`). 2 rows: Mountain View, Sodic.
- **`developer_users`** — individual salesman accounts (`id`, `developer_id`, `email`, `password_hash`, `name`, `role`). 4 test users.
- **`developer_sessions`** — auth sessions (`token_hash`, `user_id`, `created_at`, `expires_at`). 22 sessions.
- **`partner_login_attempts`** — rate limiting (`key`, `failed_count`, `locked_until`).
- **`partner_notifications`** — notifications to developers (`user_id`, `kind`, `title`, `body`, `meeting_id`, `is_read`).
- **`meetings`** — buyer-developer meetings (`buyer_handle`, `unit_id`, `developer`, `meeting_type`, `status`, `ai_brief`, `dev_outcome`, `dev_notes`, `assigned_to`). 12 meetings with AI briefs.
- **`deals`** — completed deals (`buyer_handle`, `unit_id`, `developer`, `deal_value`, `commission_due`, `reward_amount`, `status`).
- **`buyers`** — anonymous buyers (`handle`, encrypted PII, `identity_revealed`).
- **`analytics_events`** — 759 buyer activity events.

### What this tells us:
The core data model for the Developer App already exists. Meetings have AI-generated briefs, deals track commissions, and there's a working auth system with sessions and rate limiting. **We should build on this, not start from scratch.**

### Repos:
- `direkt-client-journey` (public, last push Aug 10) — likely the buyer app prototype
- `direkt-inventory` (private) — the inventory platform (active)
- `direkt-journey` (public) — project management / specs

### Security issue:
**15 tables in `direkt-production` have RLS disabled**, including `buyers`, `meetings`, `deals`, `developer_users`. This needs fixing before any production use. All `inv_` tables (the frozen copy from the inventory migration) have RLS enabled.

---

## Architecture decisions

### 1. Where does it live?

**New repo: `direkt-developer-app`** — separate from inventory and buyer app. It's a different product with different users and different deployment.

### 2. Tech stack

Match the inventory dashboard for consistency (Mahdy's team already knows this stack):
- **Next.js** (App Router) — the UI framework
- **Supabase** (`direkt-production` project) — already has the developer tables
- **Tailwind CSS** — styling
- **Deploy on Railway** — same as inventory dashboard

### 3. Database: `direkt-production` or `direkt-inventory`?

**Use `direkt-production`** (`beizrkeqlpvzfxgoejku`). Reasons:
- Developer auth tables already live there (`developer_users`, `developer_sessions`)
- Buyer activity tables already live there (`meetings`, `deals`, `analytics_events`, `buyers`)
- The Developer App needs to JOIN buyer activity with developer data — same database is simpler
- The inventory data the Developer App needs (units, projects, locations) can be read from `direkt-inventory` via Supabase's cross-project connection or synced

### 4. How does the Developer App read inventory data?

The Developer App needs read-only access to units, projects, locations, and photos from the Inventory Platform. Options:

**Option A — Direct cross-DB read (recommended for Phase 1):**
Create a Postgres foreign data wrapper (FDW) or use Supabase's DB Webhooks to sync key tables. Simplest: create read-only **views** in `direkt-production` that query `direkt-inventory` via `postgres_fdw`.

**Option B — API calls:**
The Developer App backend calls the Inventory Platform's Supabase API. More network hops, but clean separation.

**Option C — Data sync:**
A scheduled job copies inventory snapshots from `direkt-inventory` to `direkt-production`. Slight lag but the Developer App queries are all local.

**Recommendation:** Start with **Option B** (API calls) for Phase 1 — it's the cleanest, no cross-DB complexity, and read-only inventory access doesn't need real-time. The Developer App server fetches from the Inventory Platform's Supabase API using a read-only service key.

---

## Phase 1 Features

### 1. Authentication

**Use existing tables:** `developer_users`, `developer_sessions`, `partner_login_attempts`.

**Screens:**
- **Login page** — Email + password. Rate-limited (existing `partner_login_attempts` table).
- **Session management** — Token-based sessions (existing `developer_sessions` pattern). Auto-expire, refresh on activity.

**Roles:**
- `admin` — developer's account admin (can manage their salesmen, see all activity)
- `member` — individual salesman (sees only their assigned leads)

**What's needed:**
- Fix the auth flow to use proper password hashing (bcrypt)
- Add "forgot password" flow (or admin-only password reset for Phase 1)
- Enable RLS on all developer tables with proper policies
- Link `developer_users.developer_id` to `inv_developers.id` in the inventory DB (by developer name match or a new linking column)

### 2. Dashboard

The landing page after login. Shows the salesman's activity overview.

**Content:**
- **Stats bar:** Active leads (count), Meetings this week, Pending follow-ups, Deals closed (this month)
- **Recent activity feed** — last 10 events (new inquiry, meeting requested, meeting completed, deal submitted)
- **Upcoming meetings** — next 5 meetings with AI briefs
- **Follow-up alerts** — meetings past their `follow_up_due_at` that haven't been actioned

**Data source:** `meetings`, `deals` tables filtered by `developer` or `assigned_to` matching the logged-in user.

### 3. Client Activity Feed

Real-time view of all buyer interactions with this developer's units.

**Event types (from `analytics_events` and `meetings`):**
- Unit viewed by a buyer
- Unit shortlisted / saved
- Inquiry submitted (callback request)
- Meeting requested (video/call)
- Meeting completed
- Deal submitted

**Display:** Timeline feed, newest first. Each event shows:
- Anonymous buyer handle (e.g., "EG-GZ8U")
- Event type + timestamp
- Unit info (project, type, price) — fetched from inventory
- Action button (e.g., "View meeting details", "Update outcome")

**Filters:** By project, by event type, date range.

### 4. Lead Management

Track and manage buyer leads through their lifecycle.

**Lead statuses:** New → Contacted → Meeting Scheduled → Meeting Done → Negotiating → Deal Closed / Lost

**Table view — one row per lead (buyer-developer pair):**
| Buyer Handle | Unit/Project | Status | Last Activity | Assigned To | Follow-up Due | Actions |
|---|---|---|---|---|---|---|

**Detail view — click a lead:**
- Full timeline of this buyer's interactions
- Meeting history with AI briefs
- Developer notes (editable)
- Outcome tracking
- Follow-up scheduling

**Assignment:** Admins can assign/reassign leads to team members. Members see only their assigned leads.

### 5. Unit Portfolio (Read-Only)

Browse the developer's inventory — pulled from the Inventory Platform.

**Table view:**
| Project | Unit Code | Type | Bedrooms | Area (m²) | Price (EGP) | Status | Delivery |

**Filters:** Project, unit type, bedrooms, price range, status.

**No editing** — this is read-only in Phase 1. Shows a note: "To update inventory, contact the Direkt team."

**Data source:** `inv_units` from `direkt-inventory` where `developer_id` matches, via API call.

### 6. Communication (Buyer Interaction)

When a buyer requests a meeting or inquiry, the developer salesman needs to respond — without seeing the buyer's real identity.

**Meeting management:**
- View meeting request + AI brief
- Accept / propose new time
- Record outcome after meeting (`dev_outcome`, `dev_notes`)
- Schedule follow-up
- Recommend alternative units (`recommended_unit_id`)

**Buyer rating:** After a meeting, the developer can rate buyer seriousness (1-5) to help Direkt prioritize.

**Anonymity preserved:** The salesman sees only the buyer's handle (e.g., "EG-GZ8U"), never their real name, phone, or email. Direkt mediates all communication.

---

## Database Changes Needed

### In `direkt-production`:

#### Enable RLS on all 15 unprotected tables
This is critical. Every table needs RLS + proper policies before the app goes live.

#### Update `developer_users` table:
```sql
-- Add missing fields
ALTER TABLE developer_users
  ADD COLUMN phone text,
  ADD COLUMN is_active boolean DEFAULT true,
  ADD COLUMN last_login_at timestamptz,
  ADD COLUMN updated_at timestamptz DEFAULT now();
```

#### Update `developers` table:
```sql
-- Link to inventory platform and add contact fields
ALTER TABLE developers
  ADD COLUMN inv_developer_id uuid,        -- links to inv_developers.id in direkt-inventory
  ADD COLUMN logo_url text,
  ADD COLUMN tier smallint;
```

#### Add `lead_status` tracking:
```sql
CREATE TABLE lead_tracking (
  id text PRIMARY KEY DEFAULT 'lt_' || substr(md5(random()::text), 1, 12),
  buyer_handle text NOT NULL,
  developer_id text NOT NULL REFERENCES developers(id),
  assigned_to text REFERENCES developer_users(id),
  status text NOT NULL DEFAULT 'new',  -- new, contacted, meeting_scheduled, meeting_done, negotiating, closed_won, closed_lost
  notes text,
  follow_up_at timestamptz,
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now(),
  UNIQUE(buyer_handle, developer_id)
);
```

#### RLS policies pattern:
```sql
-- Developer users can only see their own developer's data
CREATE POLICY "users_see_own_developer" ON meetings
  FOR SELECT USING (developer = (
    SELECT d.name FROM developers d
    JOIN developer_users du ON du.developer_id = d.id
    WHERE du.id = current_setting('app.user_id', true)
  ));

-- Members see only assigned leads; admins see all for their developer
CREATE POLICY "lead_access" ON lead_tracking
  FOR SELECT USING (
    developer_id = (SELECT developer_id FROM developer_users WHERE id = current_setting('app.user_id', true))
    AND (
      (SELECT role FROM developer_users WHERE id = current_setting('app.user_id', true)) = 'admin'
      OR assigned_to = current_setting('app.user_id', true)
    )
  );
```

### Linking inventory data:

The `developers` table in `direkt-production` needs to be linked to `inv_developers` in `direkt-inventory`. This can be done by:
1. Adding `inv_developer_id` column (UUID) to `developers`
2. Manually mapping the 98 developers by name match
3. The Developer App backend uses this ID to fetch inventory data from the inventory API

---

## Pages / Routes

```
/login                    — Email + password login
/                         — Dashboard (redirect to /dashboard)
/dashboard                — Stats, recent activity, upcoming meetings, alerts
/activity                 — Full client activity feed
/leads                    — Lead management table
/leads/[buyer_handle]     — Lead detail + timeline
/meetings                 — All meetings
/meetings/[id]            — Meeting detail + AI brief + outcome form
/portfolio                — Unit portfolio (read-only from inventory)
/portfolio/[project]      — Project detail with unit list
/team                     — Team management (admin only)
/settings                 — Account settings, password change
```

---

## Files Structure (new repo)

```
direkt-developer-app/
├── app/
│   ├── layout.tsx
│   ├── login/page.tsx
│   ├── dashboard/page.tsx
│   ├── activity/page.tsx
│   ├── leads/page.tsx
│   ├── leads/[handle]/page.tsx
│   ├── meetings/page.tsx
│   ├── meetings/[id]/page.tsx
│   ├── portfolio/page.tsx
│   ├── portfolio/[project]/page.tsx
│   ├── team/page.tsx
│   └── settings/page.tsx
├── components/
│   ├── auth/
│   ├── dashboard/
│   ├── activity/
│   ├── leads/
│   ├── meetings/
│   ├── portfolio/
│   └── shared/
├── lib/
│   ├── supabase.ts           — Supabase client (direkt-production)
│   ├── inventory-api.ts      — Fetch from inventory platform API
│   ├── auth.ts               — Session management
│   └── types.ts
├── CLAUDE.md
├── Dockerfile
├── next.config.ts
├── package.json
└── tailwind.config.ts
```

---

## Environment Variables

```
NEXT_PUBLIC_SUPABASE_URL=https://beizrkeqlpvzfxgoejku.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=...
SUPABASE_SERVICE_ROLE_KEY=...        # for admin operations only

# Inventory Platform read access
INVENTORY_SUPABASE_URL=https://gwjbqoiltuhwxayxsqgc.supabase.co
INVENTORY_SUPABASE_ANON_KEY=...      # read-only, RLS-scoped
```

---

## Acceptance Criteria

- [ ] Developer salesman can log in with email + password
- [ ] Dashboard shows stats, recent activity, upcoming meetings
- [ ] Activity feed shows buyer events for this developer
- [ ] Leads table shows all buyer leads with status tracking
- [ ] Lead detail shows full timeline + notes + follow-up
- [ ] Meetings show AI brief, allow outcome recording
- [ ] Portfolio shows read-only units from inventory platform
- [ ] Admin can manage team members
- [ ] RLS enforced on all tables — no data leakage between developers
- [ ] Anonymity preserved — buyer identity never exposed to developers
- [ ] Mobile-responsive (salesmen use phones)
- [ ] Deployed on Railway with custom domain (e.g., partners.go-direkt.com)

---

## Security Requirements

1. **Enable RLS on all 15 unprotected tables** before launch
2. **Buyer anonymity is sacred** — never expose `full_name_encrypted`, `mobile_encrypted`, `email_encrypted` to the Developer App
3. **Developer isolation** — a developer can NEVER see another developer's data (leads, meetings, units)
4. **Password hashing** — bcrypt, never plain text
5. **Session tokens** — hashed in DB, expire after inactivity
6. **Rate limiting** — on login (existing), on API calls
7. **No service_role key in client code** — server-side only

---

## What Phase 2 adds (later, not now)

- Inventory upload from the Developer App (with AI column mapping)
- Push notifications (Firebase/APNs) for new leads
- In-app messaging between buyer and developer (mediated by Direkt)
- Analytics dashboard (conversion rates, response times)
- Developer billing / commission tracking
