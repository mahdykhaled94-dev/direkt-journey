# Direkt Inventory — Data Completeness Tracking + Export/Reporting

Two new features for the inventory dashboard. No new tables needed — these are views, functions, and dashboard pages built on top of existing data.

---

## Feature 1: Data Completeness Tracking

### The problem

SLA tracks *how often* a developer uploads, but not *how complete* their data is. Example: Palm Hills has 2,688 units loaded from 18 projects — but the catalog says they have 61 projects. That's 30% project coverage. Their units have 0% bedrooms filled, 0% floor filled. We have no way to see this today.

### What to build

#### A. Database function: `inv_developer_completeness(p_developer_id uuid DEFAULT NULL)`

Returns one row per developer (or one developer if ID is passed). No new table — this is a live calculation.

**Columns to return:**

```
developer_id          uuid
developer_name        text
tier                  smallint

-- Upload freshness
last_upload_at        timestamptz
days_since_upload     int          -- NULL if never uploaded
data_age_status       text         -- 'fresh' (<= sla_days), 'stale' (> sla_days), 'overdue' (> overdue_after_days), 'never'

-- Project coverage
catalog_projects      int          -- from inv_project_catalog
loaded_projects       int          -- distinct project_name in listed units
project_coverage_pct  int          -- loaded/catalog * 100 (NULL if no catalog)

-- Unit counts
listed_units          int
total_units           int          -- including unlisted

-- Field completeness (% of listed units that have a non-null value)
pct_price             int          -- price_total OR price_base
pct_area              int          -- area_built
pct_type              int          -- unit_type
pct_bedrooms          int          -- bedrooms
pct_floor             int          -- floor
pct_status            int          -- unit_status
pct_delivery          int          -- delivery_planned
pct_finishing         int          -- finishing_type

-- Overall data score (weighted average of key fields)
-- Price 25%, Area 20%, Type 15%, Bedrooms 15%, Status 10%, Delivery 10%, Finishing 5%
data_score            int          -- 0-100

-- Operational completeness
has_logo              boolean
has_brief             boolean
has_contacts          boolean      -- contact_email OR contact_phone is not null (after contacts migration)
has_payment_plans     boolean      -- at least 1 payment plan
has_photos            boolean      -- at least 1 project photo
has_agent             boolean      -- current agent assignment exists
```

**Implementation notes:**
- Use LEFT JOINs from `inv_developers` to compute everything in one query
- For `catalog_projects`: join `inv_project_catalog` where `developer_id` matches
- For field percentages: `count(field) * 100 / NULLIF(count(*), 0)` on listed units grouped by developer
- For `has_contacts`: check the new contact columns (from the contacts migration being built now)
- For `has_payment_plans`: check `inv_payment_plans` exists for developer
- For `has_photos`: check `inv_project_photos` exists for developer's projects
- For `has_agent`: check `inv_developer_agents` view (current assignments)
- `SECURITY INVOKER` — uses the caller's role (RLS applies). Read-only.

#### B. Dashboard page: `/completeness`

Add a new page linked from the header navigation (next to SLA, Upload, etc.).

**Layout:**

**Top summary bar:**
- Total developers: 98
- With data: X (X%)
- Average data score: X/100
- Missing contacts: X | Missing payment plans: X | Missing photos: X

**Main table — one row per developer:**

| Developer | Tier | Data Score | Upload Age | Projects | Units | Price | Area | Type | Beds | Status | Logo | Contacts | Plans | Photos | Agent |
|-----------|------|-----------|------------|----------|-------|-------|------|------|------|--------|------|----------|-------|--------|-------|
| Palm Hills | 1 | 72/100 | 3 days | 18/61 (30%) | 2,688 | 100% | 100% | 100% | 0% | 100% | No | No | No | No | No |
| SODIC | 1 | 68/100 | 5 days | 1/10 (10%) | 114 | 100% | 100% | 100% | 100% | 100% | No | No | No | No | No |
| Horizon | 1 | 0/100 | Never | 0/12 | 0 | — | — | — | — | — | No | No | No | No | No |

**Visual treatment:**
- Data score: color-coded bar (green ≥70, yellow ≥40, red <40)
- Field percentages: same color coding
- Projects: "18/61" with a small progress bar
- Boolean columns (Logo, Contacts, etc.): green checkmark or red X
- Upload age: "3 days" green, "Never" red, "12 days" yellow/red based on SLA

**Sorting:** default by data score ascending (worst first — so you see what needs work). All columns sortable (use existing `useUrlSort` pattern).

**Filters:**
- Tier (1/2/3)
- Data score range (e.g., 0-40, 40-70, 70-100)
- "Missing" quick filters: No data, No contacts, No payment plans, No photos, No agent

**Click a developer:** goes to their profile page (where contacts, logo, brief can be edited).

#### C. SLA board enhancement

On the existing SLA board, add a small "Data score" column showing the 0-100 score with color coding. This gives a quick glance without leaving the SLA page.

---

## Feature 2: Export / Reporting

### The problem

The team and developers need to pull filtered inventory data as Excel files. Use cases:
- "All available 3BR apartments in New Cairo under 5M" → Excel for a buyer inquiry
- "All Palm Hills units" → Excel for an internal review
- "All Tier 1 developers' inventory" → Excel for a management report
- "Units that changed in the last 7 days" → Excel for a change log

### What to build

#### A. Database function: `inv_search_units(...)`

A flexible search/filter function that returns units with developer name and location area joined in.

**Parameters (all optional):**

```sql
CREATE OR REPLACE FUNCTION inv_search_units(
  p_developer_id    uuid     DEFAULT NULL,
  p_developer_tier  smallint DEFAULT NULL,
  p_project_name    text     DEFAULT NULL,     -- exact or ILIKE with %
  p_area            text     DEFAULT NULL,     -- market area from inv_project_locations
  p_unit_type       text     DEFAULT NULL,     -- Apartment, Villa, Town House, etc.
  p_bedrooms_min    smallint DEFAULT NULL,
  p_bedrooms_max    smallint DEFAULT NULL,
  p_price_min       numeric  DEFAULT NULL,     -- on price_total, fallback price_base
  p_price_max       numeric  DEFAULT NULL,
  p_area_built_min  numeric  DEFAULT NULL,
  p_area_built_max  numeric  DEFAULT NULL,
  p_unit_status     text     DEFAULT NULL,     -- Available, Sold, Reserved, etc.
  p_finishing_type  text     DEFAULT NULL,
  p_delivery_from   date     DEFAULT NULL,
  p_delivery_to     date     DEFAULT NULL,
  p_listed_only     boolean  DEFAULT true,     -- only is_listed = true by default
  p_changed_since   timestamptz DEFAULT NULL,  -- units whose last_seen_at is after this
  p_limit           int      DEFAULT 5000,
  p_offset          int      DEFAULT 0
)
RETURNS TABLE (
  unit_id           uuid,
  developer_name    text,
  developer_tier    smallint,
  project_name      text,
  area              text,        -- from inv_project_locations
  unit_code         text,
  stage             text,
  unit_status       text,
  category          text,
  unit_type         text,
  bedrooms          smallint,
  floor             text,
  price_base        numeric,
  price_total       numeric,
  area_built        numeric,
  area_land         numeric,
  area_garden       numeric,
  area_roof         numeric,
  area_terrace      numeric,
  delivery_planned  date,
  finishing_type    text,
  completion_pct    numeric,
  first_seen_at     timestamptz,
  last_seen_at      timestamptz,
  is_listed         boolean
)
```

**Implementation notes:**
- JOIN `inv_developers` for name/tier
- LEFT JOIN `inv_project_locations` for area (match on developer_id + project_name)
- WHERE clauses only apply when the parameter is NOT NULL
- Price filter: use `COALESCE(price_total, price_base)`
- `SECURITY INVOKER`, read-only
- Pin `search_path = public, pg_temp`

#### B. Dashboard page: `/export`

A new page linked from the header navigation.

**Layout — filter panel on the left/top, results + export on the right/bottom:**

**Filters (all optional, combine with AND):**

| Filter | Input Type | Options |
|--------|-----------|---------|
| Developer | searchable dropdown | All 98 developers |
| Tier | multi-select chips | 1, 2, 3 |
| Area | searchable dropdown | From inv_project_locations distinct areas (25 areas) |
| Project | searchable text | Type-ahead from project names |
| Unit Type | multi-select chips | Apartment, Villa, Town House, Twin House, Chalet, Offices, Cabins |
| Bedrooms | min-max number inputs | |
| Price (EGP) | min-max number inputs | With preset buttons: Under 3M, 3-5M, 5-10M, 10M+ |
| Built Area (m²) | min-max number inputs | |
| Status | dropdown | Available, Sold, Reserved, etc. |
| Finishing | dropdown | Distinct values |
| Delivery | date range picker | From — To |
| Changed since | date picker | "Show only units updated after this date" |
| Include unlisted | toggle (default off) | |

**Results section:**

- **Count header:** "Found 1,247 units across 5 developers, 23 projects"
- **Preview table:** first 50 rows in a scrollable table (Developer, Project, Unit Code, Type, Bedrooms, Area m², Price, Status, Delivery)
- **Export button:** "Download Excel" — generates and downloads an .xlsx file

**Excel export format:**

Sheet 1 — "Units" (the filtered results):
| Developer | Tier | Project | Area | Unit Code | Stage | Status | Type | Bedrooms | Floor | BUA (m²) | Land (m²) | Garden (m²) | Roof (m²) | Terrace (m²) | Base Price (EGP) | Total Price (EGP) | Finishing | Delivery | Last Updated |

Sheet 2 — "Summary":
| Developer | Projects | Units | Avg Price | Min Price | Max Price | Avg BUA |

Sheet 3 — "Filters Applied":
The exact filters used to generate this export (for audit trail).

**Export implementation:**
- API route: `POST /api/export` — receives the filter params, calls `inv_search_units`, generates .xlsx with openpyxl (Python), returns the file
- OR: generate client-side with a JS xlsx library (SheetJS) from the Supabase query results. Simpler, no new API route needed. **Prefer this approach** — call `inv_search_units` via Supabase RPC from the client, then build the Excel in the browser.
- Max export: 10,000 units (with a warning if the result is larger — "Too many results, narrow your filters")
- File name: `Direkt_Inventory_Export_YYYY-MM-DD.xlsx`

#### C. Quick export from other pages

Add an "Export" button to existing pages where it makes sense:
- **SLA board:** Export all developers with their SLA status as Excel
- **Projects page:** Export a developer's project list with unit counts
- **Upload detail page:** Export the units from a specific upload
- **Completeness page:** Export the completeness report as Excel

These can all use the same Excel generation pattern (client-side SheetJS).

---

## Files to Edit (in the direkt-inventory repo)

### Database (migrations):
1. `migrations/0XX_completeness_function.sql` — `inv_developer_completeness()` function
2. `migrations/0XX_search_units_function.sql` — `inv_search_units()` function

### Dashboard (new pages):
3. `dashboard/app/completeness/page.tsx` — Completeness tracking page
4. `dashboard/app/export/page.tsx` — Export/reporting page

### Dashboard (components):
5. `dashboard/components/completeness/completeness-table.tsx` — The main table
6. `dashboard/components/completeness/score-bar.tsx` — Visual data score bar
7. `dashboard/components/export/filter-panel.tsx` — Filter controls
8. `dashboard/components/export/results-preview.tsx` — Preview table
9. `dashboard/components/export/excel-export.ts` — Client-side Excel generation (SheetJS)

### Dashboard (existing page updates):
10. `dashboard/components/sla-board.tsx` (or wherever the SLA board is) — Add data score column
11. Navigation component — Add "Completeness" and "Export" links

### Dashboard (lib):
12. `dashboard/lib/completeness.ts` — Supabase RPC wrapper for completeness function
13. `dashboard/lib/export.ts` — Supabase RPC wrapper for search + Excel generation

### Docs:
14. Root `CLAUDE.md` — Document new functions and pages
15. `dashboard/CLAUDE.md` — Document new pages

### Dependencies:
16. `npm install xlsx` (SheetJS) in dashboard — for client-side Excel generation. If the parser's Python openpyxl approach is preferred (server-side), add an API route instead.

---

## Acceptance Criteria

### Completeness:
- [ ] `inv_developer_completeness()` returns accurate data for all 98 developers
- [ ] Palm Hills shows: 18/61 projects (30%), 100% price, 0% bedrooms, data score ~72
- [ ] SODIC shows: 1/10 projects (10%), 100% most fields, 0% finishing, data score ~68
- [ ] Developers with no data show 0/100 score and "Never" upload age
- [ ] `/completeness` page loads with all 98 developers
- [ ] Sorting works on all columns (default: worst data score first)
- [ ] Tier filter works
- [ ] SLA board shows the data score column
- [ ] CI passes

### Export:
- [ ] `/export` page loads with all filter controls
- [ ] Filtering by developer, tier, area, type, bedrooms, price range all work
- [ ] Filters combine correctly (AND logic)
- [ ] Count header updates as filters change
- [ ] Preview shows first 50 matching rows
- [ ] "Download Excel" generates a valid .xlsx with 3 sheets
- [ ] Excel contains all filtered data (up to 10,000 units)
- [ ] Export from SLA board works
- [ ] CI passes

---

## Important constraints

- **No new tables** — completeness is a live calculation, not stored data
- **RLS applies** — both functions use SECURITY INVOKER, so only authenticated users see data
- **Pin search_path** — `search_path = public, pg_temp` on both functions
- **Don't touch parser** — these features are dashboard-only
- **Use existing patterns** — sorting via `useUrlSort`, Supabase client via existing `lib/supabase.ts`
- **Client-side Excel preferred** — avoids adding server-side dependencies; the dashboard already runs Python for uploads but Excel export is simpler in the browser
