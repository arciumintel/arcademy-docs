# Staff Program Shell (Slice A) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship staff-only `/staff/*` routes so staff can create an internal program under an org, land on a program workspace overview, edit title/tagline, and name a first draft lesson—without block editor or publish.

**Architecture:** Org-scoped App Router pages under `app/staff/`, shared staff chrome in `components/staff/`, all writes through new `lib/tenant/repositories/staff-*.ts` modules using `resolveTenantContext()` + `withTenantTransaction`. Draft lesson content uses an unpublished `curriculum_version` workspace (not `program.active_published_version_id`) plus default track + `lesson_version` with a placeholder paragraph block.

**Tech Stack:** Next.js 16 App Router, React 19, TypeScript strict, Tailwind 3, Zod 4, `pg` + RLS, better-auth staff gate via `STAFF_USER_IDS`.

**Spec:** [2026-05-28-staff-program-shell-design.md](../specs/2026-05-28-staff-program-shell-design.md)

---

## File map

| File | Responsibility |
| --- | --- |
| `lib/tenant/require-staff.ts` | `requireStaff(ctx)` → redirect/throw |
| `lib/validation/staff-program.ts` | Zod schemas for create/update/first lesson |
| `lib/tenant/repositories/staff-organizations.ts` | List orgs, get org by slug |
| `lib/tenant/repositories/staff-programs.ts` | Create program, overview, update, draft lesson |
| `lib/staff/actions.ts` | Server actions wrapping repositories |
| `app/staff/layout.tsx` | Staff gate + top nav |
| `app/staff/organizations/page.tsx` | Org list |
| `app/staff/organizations/[orgSlug]/page.tsx` | Org programs list |
| `app/staff/organizations/[orgSlug]/programs/new/page.tsx` | Create form |
| `app/staff/organizations/[orgSlug]/programs/[programSlug]/page.tsx` | Overview |
| `app/staff/organizations/[orgSlug]/programs/[programSlug]/curriculum/page.tsx` | First lesson stub |
| `app/staff/organizations/[orgSlug]/programs/[programSlug]/settings/page.tsx` | Settings |
| `components/staff/*` | Shell, chips, empty state, forms, toast |
| `tests/integration/staff-program-shell.test.ts` | Repository + isolation tests |

---

## Draft workspace convention (slice A)

Until ARC-26 formalizes draft tables:

1. Each `curriculum` has at most one **workspace** `curriculum_version` row used while `program.active_published_version_id` IS NULL or differs from that row’s id.
2. On first lesson create: insert `curriculum_version` (`version_number = 1`, `status = 'published'`, `published_at = null`), default `track` (`slug = 'main'`), `lesson_version` with placeholder blocks.
3. **Do not** set `program.active_published_version_id` until real publish (ARC-27).
4. Lesson count on overview: count `lesson_version` rows under the workspace version for that program’s curriculum.

---

### Task 1: Staff guard and validation schemas

**Files:**
- Create: `lib/tenant/require-staff.ts`
- Create: `lib/validation/staff-program.ts`
- Modify: `lib/validation.ts` (optional export re-export only if useful)

- [ ] **Step 1: Add `requireStaff`**

```typescript
// lib/tenant/require-staff.ts
import { redirect } from "next/navigation";
import type { TenantContext } from "@/lib/tenant/context";
import { ForbiddenError } from "@/lib/errors";

export function requireStaff(ctx: TenantContext): asserts ctx is Extract<TenantContext, { kind: "staff" }> {
  if (ctx.kind !== "staff") {
    throw new ForbiddenError();
  }
}

export function requireStaffOrRedirect(ctx: TenantContext, loginNext: string) {
  if (ctx.kind !== "staff") {
    redirect(`/login?next=${encodeURIComponent(loginNext)}`);
  }
}
```

- [ ] **Step 2: Add Zod schemas**

```typescript
// lib/validation/staff-program.ts
import { z } from "zod";
import { slugSchema } from "@/lib/validation";

export const createProgramSchema = z.object({
  title: z.string().trim().min(1).max(200),
  slug: slugSchema,
  tagline: z.string().trim().max(500).optional().or(z.literal("")),
});

export const updateProgramDetailsSchema = z.object({
  title: z.string().trim().min(1).max(200),
  tagline: z.string().trim().max(500).optional().or(z.literal("")),
});

export const createFirstLessonSchema = z.object({
  title: z.string().trim().min(1).max(200),
  slug: slugSchema,
});

export type CreateProgramInput = z.infer<typeof createProgramSchema>;
export type UpdateProgramDetailsInput = z.infer<typeof updateProgramDetailsSchema>;
export type CreateFirstLessonInput = z.infer<typeof createFirstLessonSchema>;
```

- [ ] **Step 3: Commit**

```bash
git add lib/tenant/require-staff.ts lib/validation/staff-program.ts
git commit -m "feat(staff): add staff guard and program validation schemas"
```

---

### Task 2: Staff organization repository

**Files:**
- Create: `lib/tenant/repositories/staff-organizations.ts`

- [ ] **Step 1: Implement list + get**

```typescript
import { NotFoundError } from "@/lib/errors";
import { withTenantTransaction } from "@/lib/db";
import { toTenantSession, type TenantContext } from "@/lib/tenant/context";
import { requireStaff } from "@/lib/tenant/require-staff";

export type StaffOrganizationRow = {
  slug: string;
  name: string;
  trustLevel: string;
  programCount: number;
};

export async function listOrganizationsForStaff(
  ctx: TenantContext,
): Promise<StaffOrganizationRow[]> {
  requireStaff(ctx);
  return withTenantTransaction(toTenantSession(ctx), async (client) => {
    const { rows } = await client.query<{
      slug: string;
      name: string;
      trust_level: string;
      program_count: string;
    }>(
      `select o.slug, o.name, o.trust_level::text as trust_level,
              count(p.id)::text as program_count
       from organization o
       left join program p on p.organization_id = o.id
       group by o.id
       order by o.name asc`,
    );
    return rows.map((r) => ({
      slug: r.slug,
      name: r.name,
      trustLevel: r.trust_level,
      programCount: Number(r.program_count),
    }));
  });
}

export async function getOrganizationBySlugForStaff(
  ctx: TenantContext,
  orgSlug: string,
): Promise<{ id: string; slug: string; name: string; trustLevel: string }> {
  requireStaff(ctx);
  return withTenantTransaction(toTenantSession(ctx), async (client) => {
    const { rows } = await client.query<{
      id: string;
      slug: string;
      name: string;
      trust_level: string;
    }>(
      `select id, slug, name, trust_level::text as trust_level
       from organization where slug = $1`,
      [orgSlug],
    );
    const row = rows[0];
    if (!row) throw new NotFoundError();
    return {
      id: row.id,
      slug: row.slug,
      name: row.name,
      trustLevel: row.trust_level,
    };
  });
}
```

- [ ] **Step 2: Commit**

```bash
git add lib/tenant/repositories/staff-organizations.ts
git commit -m "feat(staff): add organization repository for staff studio"
```

---

### Task 3: Staff programs repository (create, overview, update)

**Files:**
- Create: `lib/tenant/repositories/staff-programs.ts`

- [ ] **Step 1: Implement `createProgram`**

Insert `program` + `curriculum` in one transaction. On unique violation for `(organization_id, slug)`, throw a typed error (add `ConflictError` in `lib/errors.ts` with status 409) or return `{ ok: false, field: "slug" }`—pick one pattern and use in action.

```typescript
// lib/errors.ts — add
export class ConflictError extends AppError {
  constructor(message = "Conflict") {
    super(message, 409);
    this.name = "ConflictError";
  }
}
```

`createProgram(ctx, { orgSlug, title, slug, tagline })` → `{ orgSlug, programSlug }`.

- [ ] **Step 2: Implement `getStaffProgramOverview`**

Return:

```typescript
export type StaffProgramOverview = {
  orgSlug: string;
  orgName: string;
  programSlug: string;
  title: string;
  tagline: string | null;
  hubStatus: string;
  draftStatus: string;
  activePublishedVersionId: string | null;
  lessonCount: number;
};
```

Lesson count query joins `curriculum` → workspace `curriculum_version` (where `cv.id IS DISTINCT FROM p.active_published_version_id` OR `p.active_published_version_id IS NULL`) → `track` → `lesson_version`.

- [ ] **Step 3: Implement `updateProgramDetails`**

Patch `title`, `tagline` where org slug + program slug match.

- [ ] **Step 4: Implement `createFirstDraftLesson`**

1. Resolve program + curriculum id.
2. Get or create workspace `curriculum_version` (version_number = 1 if none).
3. Insert track `main` position 1 if none.
4. Insert `lesson_version` with `title: { en: title }`, `blocks: [{ type: "paragraph", text: { en: "Draft — add content in the lesson editor." } }]`.
5. Enforce unique `(track_id, slug)` for lesson.

- [ ] **Step 5: Implement `listProgramsForOrg`**

Rows: `slug`, `title`, `hubStatus`, `draftStatus`, `lessonCount`, `updatedAt`.

- [ ] **Step 6: Commit**

```bash
git add lib/errors.ts lib/tenant/repositories/staff-programs.ts
git commit -m "feat(staff): add staff program repository with draft workspace"
```

---

### Task 4: Integration tests

**Files:**
- Create: `tests/integration/staff-program-shell.test.ts`
- Modify: `tests/integration/helpers/isolation-fixtures.ts` (add `staff-user` + `STAFF_ORG_SLUG` test org, or create org inline in test)

- [ ] **Step 1: Write tests**

```typescript
// tests/integration/staff-program-shell.test.ts
import { beforeAll, describe, expect, it } from "vitest";
// ... dotenv load same as tenant-isolation.test.ts

const staffCtx = {
  kind: "staff" as const,
  userId: "staff-user",
  orgIds: [] as string[],
  isStaff: true as const,
};

describe("staff program shell", () => {
  beforeAll(() => {
    process.env.STAFF_USER_IDS = "staff-user";
    // seed staff-user row + test org via helper or inline SQL with is_staff session
  });

  it("staff creates program with curriculum draft row", async () => {
    const { createProgram, getStaffProgramOverview } = await import(
      "@/lib/tenant/repositories/staff-programs"
    );
    const created = await createProgram(staffCtx, {
      orgSlug: "staff-test-org",
      title: "Test Program",
      slug: `test-${Date.now()}`,
      tagline: "Tagline",
    });
    const overview = await getStaffProgramOverview(staffCtx, {
      orgSlug: created.orgSlug,
      programSlug: created.programSlug,
    });
    expect(overview.hubStatus).toBe("internal");
    expect(overview.draftStatus).toBe("draft");
    expect(overview.activePublishedVersionId).toBeNull();
    expect(overview.lessonCount).toBe(0);
  });

  it("duplicate slug returns conflict", async () => { /* ... */ });

  it("internal program not in listListedPrograms", async () => {
    const { listListedPrograms } = await import("@/lib/tenant/repositories/programs");
    const listed = await listListedPrograms({ kind: "anonymous", isStaff: false });
    expect(listed.some((p) => p.slug === "internal-only-slug")).toBe(false);
  });
});
```

- [ ] **Step 2: Run tests**

```bash
npm run test:integration -- tests/integration/staff-program-shell.test.ts
```

Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add tests/integration/staff-program-shell.test.ts tests/integration/helpers/isolation-fixtures.ts
git commit -m "test(staff): integration tests for program shell repository"
```

---

### Task 5: Server actions

**Files:**
- Create: `lib/staff/actions.ts`

- [ ] **Step 1: Implement actions**

```typescript
"use server";

import { redirect } from "next/navigation";
import { revalidatePath } from "next/cache";
import { resolveTenantContext } from "@/lib/tenant/context";
import { requireStaff } from "@/lib/tenant/require-staff";
import { createProgramSchema, updateProgramDetailsSchema, createFirstLessonSchema } from "@/lib/validation/staff-program";
import {
  createProgram,
  updateProgramDetails,
  createFirstDraftLesson,
} from "@/lib/tenant/repositories/staff-programs";
import { ConflictError } from "@/lib/errors";

export async function createProgramAction(orgSlug: string, formData: FormData) {
  const ctx = await resolveTenantContext();
  requireStaff(ctx);
  const parsed = createProgramSchema.safeParse({
    title: formData.get("title"),
    slug: formData.get("slug"),
    tagline: formData.get("tagline") ?? "",
  });
  if (!parsed.success) {
    return { ok: false as const, errors: parsed.error.flatten().fieldErrors };
  }
  try {
    const { programSlug } = await createProgram(ctx, {
      orgSlug,
      ...parsed.data,
      tagline: parsed.data.tagline || null,
    });
    redirect(
      `/staff/organizations/${orgSlug}/programs/${programSlug}?created=1`,
    );
  } catch (e) {
    if (e instanceof ConflictError) {
      return { ok: false as const, errors: { slug: ["Slug already in use."] } };
    }
    throw e;
  }
}

// updateProgramDetailsAction, createFirstLessonAction — same pattern
// redirect with ?saved=1 or ?lessonCreated=1
```

- [ ] **Step 2: Commit**

```bash
git add lib/staff/actions.ts
git commit -m "feat(staff): server actions for program shell"
```

---

### Task 6: Staff layout and top navigation

**Files:**
- Create: `app/staff/layout.tsx`
- Create: `components/staff/StaffTopNav.tsx`

- [ ] **Step 1: Layout with staff gate**

```tsx
// app/staff/layout.tsx
import { resolveTenantContext } from "@/lib/tenant/context";
import { requireStaffOrRedirect } from "@/lib/tenant/require-staff";
import StaffTopNav from "@/components/staff/StaffTopNav";

export default async function StaffLayout({ children }: { children: React.ReactNode }) {
  const ctx = await resolveTenantContext();
  requireStaffOrRedirect(ctx, "/staff/organizations");
  return (
    <div className="min-h-screen bg-background text-ink">
      <StaffTopNav />
      {children}
    </div>
  );
}
```

`StaffTopNav` — client or server: links to `/staff/organizations`, highlight active using `usePathname` in a small client wrapper `StaffTopNavLinks.tsx`.

- [ ] **Step 2: Verify build**

```bash
npm run build
```

Expected: success (staff routes compile)

- [ ] **Step 3: Commit**

```bash
git add app/staff/layout.tsx components/staff/StaffTopNav.tsx components/staff/StaffTopNavLinks.tsx
git commit -m "feat(staff): staff layout with auth gate and top nav"
```

---

### Task 7: Organization list and org program list

**Files:**
- Create: `app/staff/organizations/page.tsx`
- Create: `app/staff/organizations/[orgSlug]/page.tsx`

- [ ] **Step 1: Org list page**

Server component: `listOrganizationsForStaff`, editorial list styling (hairlines, mono kickers), each row links to `/staff/organizations/[slug]`.

- [ ] **Step 2: Org detail page**

`getOrganizationBySlugForStaff` + `listProgramsForOrg`. Header with org name. **New program** link → `.../programs/new`. Program rows link to overview.

- [ ] **Step 3: Commit**

```bash
git add app/staff/organizations/page.tsx app/staff/organizations/[orgSlug]/page.tsx
git commit -m "feat(staff): organization list and org program list pages"
```

---

### Task 8: Create program form

**Files:**
- Create: `app/staff/organizations/[orgSlug]/programs/new/page.tsx`
- Create: `components/staff/CreateProgramForm.tsx`

- [ ] **Step 1: Form component**

Client component: controlled title/slug (auto-slugify title on blur), tagline textarea, submit via `createProgramAction`. Display field errors from action result. Use `inputClass` pattern from `LoginForm.tsx`.

Intake mode: when `searchParams.intakeId` present, show `IntakeBanner` and disable org fields (org not on form—already in URL).

- [ ] **Step 2: Page wires org name in header**

- [ ] **Step 3: Commit**

```bash
git add app/staff/organizations/[orgSlug]/programs/new/page.tsx components/staff/CreateProgramForm.tsx components/staff/IntakeBanner.tsx
git commit -m "feat(staff): create program form under org context"
```

---

### Task 9: Program workspace shell and overview

**Files:**
- Create: `components/staff/StaffProgramShell.tsx`
- Create: `components/staff/ProgramStatusChips.tsx`
- Create: `components/staff/ProgramOverviewEmptyState.tsx`
- Create: `components/staff/StaffFlashToast.tsx`
- Create: `app/staff/organizations/[orgSlug]/programs/[programSlug]/layout.tsx`
- Create: `app/staff/organizations/[orgSlug]/programs/[programSlug]/page.tsx`

- [ ] **Step 1: `StaffProgramShell`**

Props: `orgSlug`, `programSlug`, `active: "overview" | "curriculum" | "settings"`, `orgName`, `children`.

Sidebar per spec §5.1 (Program workspace label, Overview/Curriculum, Settings footer with `Settings` icon from `lucide-react`).

- [ ] **Step 2: Overview page**

Load `getStaffProgramOverview`. Breadcrumb, H1, subtitle `{orgName} · Internal program`, `ProgramStatusChips`, `StaffFlashToast` when `?created=1`.

If `lessonCount === 0`: `ProgramOverviewEmptyState` with link to curriculum route.

Else: summary card with lesson count + link to curriculum.

Secondary row: link to settings; preview link disabled when `!activePublishedVersionId || hubStatus === 'internal'`.

- [ ] **Step 3: Program layout wraps shell**

```tsx
// layout.tsx loads overview metadata once for shell title optional
```

- [ ] **Step 4: Commit**

```bash
git add components/staff/StaffProgramShell.tsx components/staff/ProgramStatusChips.tsx components/staff/ProgramOverviewEmptyState.tsx components/staff/StaffFlashToast.tsx app/staff/organizations/[orgSlug]/programs/[programSlug]/layout.tsx app/staff/organizations/[orgSlug]/programs/[programSlug]/page.tsx
git commit -m "feat(staff): program workspace overview with empty state"
```

---

### Task 10: Settings page

**Files:**
- Create: `app/staff/organizations/[orgSlug]/programs/[programSlug]/settings/page.tsx`
- Create: `components/staff/EditProgramDetailsForm.tsx`

- [ ] **Step 1: Settings form**

Read-only rows: organization, slug, hub status. Editable title/tagline. `updateProgramDetailsAction`. Toast on `?saved=1`.

- [ ] **Step 2: Commit**

```bash
git add app/staff/organizations/[orgSlug]/programs/[programSlug]/settings/page.tsx components/staff/EditProgramDetailsForm.tsx
git commit -m "feat(staff): program settings page"
```

---

### Task 11: Curriculum stub — first lesson

**Files:**
- Create: `app/staff/organizations/[orgSlug]/programs/[programSlug]/curriculum/page.tsx`
- Create: `components/staff/CreateFirstLessonForm.tsx`

- [ ] **Step 1: Curriculum page**

If `lessonCount > 0`: show simple list (slug + title) and link back to overview—minimal for slice A.

If `lessonCount === 0`: empty state + `CreateFirstLessonForm` calling `createFirstLessonAction` → redirect overview `?lessonCreated=1`.

- [ ] **Step 2: Commit**

```bash
git add app/staff/organizations/[orgSlug]/programs/[programSlug]/curriculum/page.tsx components/staff/CreateFirstLessonForm.tsx
git commit -m "feat(staff): curriculum stub with first lesson create"
```

---

### Task 12: Docs and manual smoke test

**Files:**
- Modify: `docs/AGENT-PLATFORM.md` (add Staff Studio slice A exit note under Phase 2)
- Modify: `AGENTS.md` (one line: staff routes exist)

- [ ] **Step 1: Document env**

Staff local dev requires `STAFF_USER_IDS=<your-user-id>` in `.env.local` matching a signed-in better-auth user.

- [ ] **Step 2: Manual smoke**

```bash
npm run dev
```

1. Sign in as staff user
2. Visit `/staff/organizations`
3. Open org → New program → create → overview toast + empty state
4. Add first lesson → overview shows count 1
5. Edit settings → save
6. Sign in as non-staff → `/staff/*` redirects to login

- [ ] **Step 3: Final commit**

```bash
git add docs/AGENT-PLATFORM.md AGENTS.md
git commit -m "docs: note staff program shell slice A"
```

---

## Spec coverage checklist

| Spec requirement | Task |
| --- | --- |
| Staff auth gate | 1, 6 |
| Org list + org programs | 2, 7 |
| Create program + curriculum row | 3, 8 |
| Overview empty state + chips + toast | 9 |
| Settings edit title/tagline | 10 |
| Curriculum first lesson stub | 11 |
| Intake hook (`?intakeId=`) | 8 (banner + locked fields; no queue) |
| Preview disabled until publish | 9 |
| Integration tests | 4 |
| No block editor / publish | out of scope |

## Deferred (explicit)

- Intake queue API/table (ARC-24)
- Block editor (slice C)
- Publish / hub listing (ARC-26/27)
- `program_hub_settings` writes
- Org create UI (seed/script only for new orgs in dev)
