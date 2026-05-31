# Staff Program Shell (Slice A) — Design Spec

**Status:** Approved  
**Date:** 2026-05-28  
**Parent spec:** [2026-05-20-ecosystem-platform-design.md](./2026-05-20-ecosystem-platform-design.md)  
**Linear:** ARC-25 (Staff Studio org/program scoping), partial ARC-24 hook for intake  
**Implementation plan:** [2026-05-28-staff-program-shell.md](../plans/2026-05-28-staff-program-shell.md)

---

## 1. Summary

Slice A delivers the **staff program shell**: authenticated staff can create a program under an organization, land on a program workspace home, edit basic program metadata, and reach a minimal curriculum stub to name the first lesson. It does **not** include block editing, publish, hub listing, or partner intake queue UI.

**Outcome:** Staff can go from “new partner program” to a named internal program with an empty curriculum path, using the same editorial visual language as the public hub.

---

## 2. Scope

### In scope

| Area | Deliverable |
| --- | --- |
| Staff auth gate | All `/staff/*` requires `resolveTenantContext()` → `kind === "staff"` |
| Org-scoped navigation | List orgs; org overview with program list |
| Create program | Form: title, slug, tagline; creates `program` + `curriculum` draft row |
| Program workspace home | Status chips, empty state, primary/secondary actions |
| Program settings | Edit title + tagline; read-only slug, org, hub status |
| Curriculum stub | Empty state + first-lesson title/slug form (no block editor) |
| Intake hook | Same create form with prefilled/locked fields when `?intakeId=` is present (API/table may follow in ARC-24) |

### Out of scope (later slices)

- Lesson block editor and quiz authoring (slice C)
- Full curriculum outline (tracks, reorder, lesson list) beyond first-lesson stub (slice B)
- Publish, version list, rollback (ARC-26 / ARC-27)
- Hub listing, featured rank, `program_hub_settings` writes
- Partner intake queue list UI (ARC-24) — only form variant hook in A
- Preview tokens for unpublished drafts (ARC-28)

---

## 3. Locked product decisions

| Topic | Decision |
| --- | --- |
| URL pattern | **Org-scoped canonical:** `/staff/organizations/[orgSlug]/programs/[programSlug]/…` |
| Create entry (normal) | Context-first: `/staff/organizations/[orgSlug]/programs/new` — org not selectable on form |
| Create entry (intake) | Same route + query; org/program fields prefilled and locked |
| Post-create redirect | Program workspace **Overview** |
| Create fields | Title, slug, tagline |
| Default hub status | `internal` |
| On create | Insert `program` + `curriculum` (`draft_status = draft`); no tracks/lessons until curriculum stub submit |
| Preview public page | Links to `/programs/[programSlug]`; disabled until published + listable |

---

## 4. Routes and information architecture

```
/staff
  layout.tsx                    — staff shell (top nav + optional program sidebar)

/staff/organizations
  page.tsx                      — org list

/staff/organizations/[orgSlug]
  page.tsx                      — org overview + program list + “New program”

/staff/organizations/[orgSlug]/programs/new
  page.tsx                      — create program form

/staff/organizations/[orgSlug]/programs/[programSlug]
  page.tsx                      — program workspace: Overview (default)

/staff/organizations/[orgSlug]/programs/[programSlug]/curriculum
  page.tsx                      — curriculum stub / first lesson form

/staff/organizations/[orgSlug]/programs/[programSlug]/settings
  page.tsx                      — edit program details
```

**Staff gate:** Non-staff → redirect to `/login`. Cross-org access uses staff RLS bypass; still validate org/program slugs exist.

**Breadcrumb pattern:** `{Org name} / Programs / {Program title}`

---

## 5. Visual design (editorial tokens)

Implementation follows the existing Arcademy design system (`tailwind.config.ts`, hub pages). No external mockup asset is required. Layout and hierarchy below are binding.

### 5.1 Global staff chrome

**Top bar**

- Left: **Arcademy** wordmark + **Staff Studio** label
- Center: text links **Organizations**, **Programs** — active route uses accent underline (`border-b` / `text-accent`)
- Right: user avatar or initials (account menu out of scope for A; placeholder ok)

**Program workspace sidebar** (shown on program routes only)

- Section label (mono caps): **Program workspace**
- Sub-label (muted): production context line optional (e.g. org name or “Production desk”)
- Nav items:
  - **Overview** — active state: `sage-soft` background + `sage` left border (4px)
  - **Curriculum**
- Footer: **Settings** with gear icon

Use existing tokens from `tailwind.config.ts`: `background` / `paper`, `ink` / `ink-muted`, `accent` (vermillion), `sage` / `sage-soft`, `rule` hairlines.

### 5.2 Program workspace — Overview (empty state)

**Header block**

- Breadcrumb (ink-soft, small)
- H1: program title
- Subtitle: `{Organization name} · Internal program` (or dynamic hub status label when not internal)
- **Status chips** (horizontal):
  - **Internal** — hairline pill + small accent dot
  - **Draft** — hairline pill + small sage dot
  - **Not published** — hairline pill, no dot (muted)

**Success toast** (after create redirect; dismissible)

- Position: top-right of content area
- Icon: sage check in circle
- Title: **Program created**
- Body: Add your first lesson to start building the curriculum.
- Close control

**Empty state card** (centered, `paper-deep` / white elevated surface, hairline border)

- Kicker (mono caps): `Curriculum · 0 lessons`
- Headline: **No lessons yet**
- Body: Programs are built from structured lessons and comprehension checks. Start with a single lesson — you can reorder and expand the path later.
- **Primary CTA:** filled `accent` button — **Add first lesson** → curriculum route

**Secondary actions** (below card, centered)

- **Edit program details** — text link, accent color → settings route
- **Preview public page** — disabled (`ink-faint`), helper: *Available after publish*

### 5.3 Create program form

- Kicker: `New program · {Org name}`
- H1: **Create program**
- Helper: Programs start internal. Hub listing and publish come later.
- Fields: title, slug (with `/programs/{slug}` hint, mono slug segment), tagline (textarea)
- Actions: **Create program** (primary), **Cancel** (ghost → org program list)

**Intake variant:** Banner with ochre left rule — created from partner intake + request id; org locked; title/slug prefilled.

### 5.4 Curriculum stub

- Same shell + sidebar (**Curriculum** active)
- Empty panel: **Name your first lesson**
- Fields: lesson title, lesson slug
- **Create lesson** (primary), **Back to overview** (text)
- Does not show block editor or track list in slice A

### 5.5 Settings

- Same shell + sidebar (**Settings** active)
- Editable: title, tagline
- Read-only: organization, slug, hub status
- **Save changes** → success toast

### 5.6 Anti-patterns

- No LMS dashboard cards, gamification, charts, or dense data tables in slice A
- No second saturated accent competing with vermillion on the same view
- No global `/staff/programs/[slug]` without org segment (slug is unique per org only)

---

## 6. Screen behavior

### 6.1 Create program

**Server action** (single transaction):

1. Validate staff context + org slug
2. Validate slug unique per `(organization_id, slug)`
3. Insert `program` (`hub_status = 'internal'`, title, tagline)
4. Insert `curriculum` (`draft_status = 'draft'`)
5. Redirect to Overview with flash flag for success toast

### 6.2 Overview

| State | UI |
| --- | --- |
| 0 lessons | Empty state card (§5.2) |
| ≥1 lesson (stub) | Replace empty card with summary: lesson count + link to curriculum |

**Chip logic**

| Chip | Source |
| --- | --- |
| Internal / Listed / … | `program.hub_status` (display friendly label) |
| Draft / In review / … | `curriculum.draft_status` |
| Not published / Published | `active_published_version_id` null or set |

### 6.3 Curriculum stub — first lesson

On submit, create minimal draft lesson structure (implementation must align with ARC-26 draft storage; see §8). Redirect to Overview or lesson stub page with success toast.

### 6.4 Settings

Patch `program.title`, `program.tagline` only. Slug immutable after create in v1.

### 6.5 Preview public page

- `href="/programs/{programSlug}"` `target="_blank"`
- `disabled` when `hub_status === 'internal'` OR `active_published_version_id` IS NULL
- Tooltip: Available after publish

---

## 7. Backend

### 7.1 Repository functions (new or extended)

| Function | Notes |
| --- | --- |
| `listOrganizationsForStaff(ctx)` | All orgs for staff user |
| `listProgramsForOrg(ctx, orgSlug)` | Programs for org overview |
| `createProgram(ctx, input)` | Transaction: program + curriculum |
| `getStaffProgramOverview(ctx, orgSlug, programSlug)` | Title, org, chips data, lesson count |
| `updateProgramDetails(ctx, …)` | Title, tagline patch |
| `createFirstDraftLesson(ctx, …)` | Stub; see §8 |

All writes via `withTenantTransaction` + staff session. **No raw `query()` in route handlers.**

### 7.2 Validation (Zod)

- `title`: non-empty, max 200 characters
- `slug`: `^[a-z0-9]+(?:-[a-z0-9]+)*$`, max 80 characters, unique per org
- `tagline`: optional, max 500 characters

### 7.3 Authorization

- `ctx.kind === 'staff'` required
- `isStaffUser` / `STAFF_USER_IDS` per existing `lib/tenant/context.ts`

---

## 8. Draft lesson storage (dependency)

Current schema attaches `track` and `lesson_version` to `curriculum_version`, while product model edits a **working draft** on `curriculum`. Slice A’s “create first lesson” stub must not write to published version rows.

**Requirement for implementation plan:**

- Confirm draft authoring model with ARC-26 (draft tables vs draft `curriculum_version` workspace)
- Slice A may ship Overview + create program + settings **before** first-lesson persist if draft model is unresolved; curriculum route can show form UI with save disabled + staff-only message until backend lands

---

## 9. Components (suggested)

| Component | Responsibility |
| --- | --- |
| `app/staff/layout.tsx` | Top nav, staff gate |
| `components/staff/StaffProgramShell.tsx` | Sidebar + breadcrumb slot |
| `components/staff/ProgramStatusChips.tsx` | Chip row |
| `components/staff/ProgramOverviewEmptyState.tsx` | Center card + CTA |
| `components/staff/CreateProgramForm.tsx` | Normal + intake modes |
| `components/staff/StaffToast.tsx` | Success/dismiss (or use URL flash + client toast) |

Reuse learner hub typography utilities where possible; staff-specific layout lives under `app/staff/` and `components/staff/`.

---

## 10. Testing

| Test | Expectation |
| --- | --- |
| Staff creates program | `program` + `curriculum` rows; `hub_status = internal` |
| Slug collision | Field error, no partial insert |
| Non-staff GET `/staff/...` | Redirect or 403 |
| Internal program | Not in `listListedPrograms` for anonymous/learner catalog |
| Cross-tenant | Staff program in org B not readable by partner user in org A (404) |

---

## 11. Follow-on slices

| Slice | Adds |
| --- | --- |
| B | Full curriculum outline, tracks, lesson list, reorder |
| C | Block editor + quiz panel + learner preview |
| E | Intake queue UI + create-from-intake flow |
| ARC-26/27 | Publish, rollback, version history |

---

## 12. Approval log

| Section | Status | Date |
| --- | --- | --- |
| Routes & IA | Approved | 2026-05-28 |
| Screens & behavior | Approved | 2026-05-28 |
| Backend & data | Approved | 2026-05-28 |
| Visual design (tokens + §5) | Approved | 2026-05-28 |
