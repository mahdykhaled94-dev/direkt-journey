# Direkt Inventory — Add Developer Contacts

## Context

The `inv_developers` table currently stores: `name`, `tier`, `sla_days`, `overdue_after_days`, `assigned_team_email`, `column_mapping`, `last_upload_at`, `logo_url`, `brief`, `profile_updated_at`.

There are **no contact fields** — no phone, no email, no address, no website, no contact person name. We need these because:

1. **Developer App (Phase 1):** developers will log in using their contact info (email/phone) stored here
2. **Buyer App:** developer contact info will be shown so buyers can reach them
3. **Direkt team operations:** the team needs to call/email developers about inventory updates, SLA follow-ups, etc.

---

## 1. Database Changes

### New columns on `inv_developers`

Add these columns to `inv_developers` via a Supabase migration:

```
contact_person       text      -- Primary contact person's full name (e.g. "Ahmed Mostafa")
contact_title        text      -- Their job title (e.g. "Sales Director", "Head of Sales")
contact_email        text      -- Primary email (e.g. "ahmed@palmhills.com")
contact_phone        text      -- Primary phone with country code (e.g. "+201012345678")
contact_phone_2      text      -- Secondary phone (optional)
company_email        text      -- General company email (e.g. "sales@palmhills.com")
company_phone        text      -- General company phone
company_website      text      -- Developer website (e.g. "https://www.palmhills.com")
company_address      text      -- HQ address (free text)
```

All nullable. No defaults needed.

### RLS / Column Privileges

Currently, `authenticated` can UPDATE only: `logo_url`, `brief`, `profile_updated_at`.

**Add UPDATE privilege** on the new contact columns for `authenticated` role — same as the existing profile columns, so the dashboard team can fill them in:

```sql
GRANT UPDATE (contact_person, contact_title, contact_email, contact_phone, contact_phone_2,
              company_email, company_phone, company_website, company_address)
ON inv_developers TO authenticated;
```

The existing `team_edit_profile` UPDATE policy already allows authenticated users to update (using = true, with_check = true), so no new policy is needed — just the column grants.

### Update `inv_developers_status` view

The view currently adds `days_since_upload`, `next_due_at`, `is_overdue`, `sla_state`, `listed_units`, `listed_projects`. It does NOT need changing — the new columns will come through on `SELECT *` from the base table. But **verify** this after the migration.

---

## 2. Dashboard Changes

### A. Developer Profile Page — Add "Contacts" Section

The developer profile is accessible from the SLA board (clicking a developer). Currently it shows: logo upload, brief/description text area, and profile_updated_at.

**Add a "Contacts" section** below or next to the existing profile fields. It should have:

| Field | Input Type | Placeholder |
|-------|-----------|-------------|
| Contact Person | text input | "e.g. Ahmed Mostafa" |
| Title | text input | "e.g. Sales Director" |
| Email | email input | "e.g. ahmed@palmhills.com" |
| Phone | tel input | "e.g. +201012345678" |
| Phone 2 | tel input | "e.g. +201112345678" |
| Company Email | email input | "e.g. sales@palmhills.com" |
| Company Phone | tel input | "e.g. +20223456789" |
| Website | url input | "e.g. https://www.palmhills.com" |
| Address | textarea | "HQ address" |

**Behavior:**
- Load current values from `inv_developers` on page load
- Save on blur or with a "Save" button (same pattern as existing profile fields)
- Update `profile_updated_at` on every save
- Show success/error toast on save
- Basic validation: email format, phone starts with "+" or is numeric, website starts with "http"

### B. SLA Board — Add Contact Indicators

On the main SLA board table, add small visual indicators:
- A column or icon showing whether a developer has contacts filled in (a checkmark or "No contacts" warning)
- This helps the team see which developers still need contact info entered

### C. Developer Detail / Hover — Show Contact Quick-View

When clicking or hovering on a developer name anywhere in the dashboard, show a quick summary:
- Contact person + title
- Phone (clickable `tel:` link)
- Email (clickable `mailto:` link)

---

## 3. API (if needed)

The dashboard already talks directly to Supabase via the client SDK (RLS enforced). The new fields are just columns on `inv_developers`, so:

- **Read:** existing `supabase.from('inv_developers').select('*')` will include the new columns automatically
- **Write:** `supabase.from('inv_developers').update({ contact_person: '...', ... }).eq('id', devId)` — same pattern as logo_url/brief

No new API routes needed.

---

## 4. Migration SQL

```sql
-- Migration: Add developer contact fields
ALTER TABLE inv_developers
  ADD COLUMN contact_person   text,
  ADD COLUMN contact_title    text,
  ADD COLUMN contact_email    text,
  ADD COLUMN contact_phone    text,
  ADD COLUMN contact_phone_2  text,
  ADD COLUMN company_email    text,
  ADD COLUMN company_phone    text,
  ADD COLUMN company_website  text,
  ADD COLUMN company_address  text;

-- Grant UPDATE on contact columns to authenticated users
GRANT UPDATE (contact_person, contact_title, contact_email, contact_phone, contact_phone_2,
              company_email, company_phone, company_website, company_address)
ON inv_developers TO authenticated;

-- Comment the columns
COMMENT ON COLUMN inv_developers.contact_person IS 'Primary contact person full name';
COMMENT ON COLUMN inv_developers.contact_title IS 'Contact person job title';
COMMENT ON COLUMN inv_developers.contact_email IS 'Primary contact email';
COMMENT ON COLUMN inv_developers.contact_phone IS 'Primary phone with country code';
COMMENT ON COLUMN inv_developers.contact_phone_2 IS 'Secondary phone';
COMMENT ON COLUMN inv_developers.company_email IS 'General company email';
COMMENT ON COLUMN inv_developers.company_phone IS 'General company phone';
COMMENT ON COLUMN inv_developers.company_website IS 'Developer website URL';
COMMENT ON COLUMN inv_developers.company_address IS 'HQ address';
```

---

## 5. What NOT to Change

- **Parser:** no changes — the parser doesn't touch contact fields
- **RLS policies:** no new policies needed — existing `team_edit_profile` covers UPDATE, `team_read` covers SELECT
- **Views:** `inv_developers_status` doesn't need updating — it already selects from `inv_developers` and the new columns will be available
- **Column mapping / upload flow:** completely unrelated, don't touch

---

## 6. Files to Edit (in the direkt-inventory repo)

1. **New migration file** — `migrations/0XX_add_developer_contacts.sql` (the SQL above)
2. **Dashboard developer profile component** — add the contacts form section (likely in `dashboard/components/` or `dashboard/app/` wherever the developer profile page lives)
3. **Dashboard SLA board** — add contact status indicator column
4. **Dashboard shared component** — developer contact quick-view (reusable)
5. **Dashboard types** — update TypeScript types if there's a Developer interface/type definition
6. **CLAUDE.md** — update the `inv_developers` column list to include the new contact fields

---

## 7. Acceptance Criteria

- [ ] Migration runs cleanly on the live `direkt-inventory` Supabase project
- [ ] All 98 developers show the new empty contact fields in the dashboard
- [ ] Team members can fill in and save contact info for any developer
- [ ] `profile_updated_at` updates when contacts are saved
- [ ] SLA board shows which developers have / don't have contacts
- [ ] Contact quick-view works from developer name click
- [ ] No changes to parser, upload flow, or existing column mapping behavior
- [ ] CI passes (type-check, lint, build)
