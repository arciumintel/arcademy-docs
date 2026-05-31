# Staff Publish Lifecycle (ARC-26 + ARC-27) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Staff can run curriculum through draft/review/approve, publish immutable snapshots, rollback active version, and continue editing from a cloned workspace — all from program Overview.

**Architecture:** Pure FSM in `lib/curriculum/lifecycle.ts`; deep-copy in `lib/curriculum/clone-version.ts`; validation in `lib/curriculum/publish-validation.ts`. Repository wrappers in `staff-programs.ts` run inside `withTenantTransaction`. Server actions + Overview UI follow existing Staff Studio patterns.

**Tech Stack:** Next.js 16 App Router, React 19, Zod 4, `pg` + RLS immutability triggers, Vitest integration tests.

**Spec:** [2026-05-31-staff-publish-lifecycle-design.md](../specs/2026-05-31-staff-publish-lifecycle-design.md)

**Skills to invoke during implementation:**

| Phase | Skill | When |
| --- | --- | --- |
| Backend transactions | `neon-postgres` | Publish/rollback single-transaction design |
| UI | `.agents/skills/impeccable` | Lifecycle panel + version table polish |
| Tests | Existing Vitest patterns in `tests/integration/staff-*.test.ts` | No new skill install |
| Next.js actions | Cursor Vercel `@nextjs` | Server action redirects + revalidation |

---

## File map

| File | Action |
| --- | --- |
| `lib/curriculum/lifecycle.ts` | **Create** — FSM types, `assertDraftTransition`, `canPublish` |
| `lib/curriculum/clone-version.ts` | **Create** — `cloneCurriculumVersion(client, sourceCvId, targetVersionNumber, opts)` |
| `lib/curriculum/publish-validation.ts` | **Create** — `validateWorkspaceForPublish(lessons)` → errors + missingQuizSlugs |
| `lib/errors.ts` | **Modify** — add `ValidationError` with optional `details` for lesson errors / quiz warnings |
| `lib/validation/staff-program.ts` | **Modify** — lifecycle Zod schemas |
| `lib/tenant/repositories/staff-programs.ts` | **Modify** — export lifecycle repo functions; export `findWorkspaceVersionId` helpers for tests if needed |
| `lib/staff/actions.ts` | **Modify** — lifecycle server actions |
| `components/staff/PublishingLifecyclePanel.tsx` | **Create** |
| `components/staff/PublishConfirmDialog.tsx` | **Create** — client component |
| `components/staff/VersionHistoryTable.tsx` | **Create** |
| `app/staff/.../programs/[programSlug]/page.tsx` | **Modify** — wire panel + history |
| `tests/unit/curriculum-lifecycle-fsm.test.ts` | **Create** |
| `tests/integration/staff-publish-lifecycle.test.ts` | **Create** |
| `docs/AGENT-PLATFORM.md` | **Modify** — Staff Studio row mentions publish/rollback |
| `docs/superpowers/specs/2026-05-31-staff-publish-lifecycle-design.md` | **Modify** — set Status: Implemented when done |

---

## Task 1: FSM module + unit tests

**Files:**
- Create: `lib/curriculum/lifecycle.ts`
- Create: `tests/unit/curriculum-lifecycle-fsm.test.ts`

- [ ] **Step 1: Write failing FSM tests**

```typescript
// tests/unit/curriculum-lifecycle-fsm.test.ts
import { describe, expect, it } from "vitest";
import {
  assertDraftTransition,
  canPublish,
  type DraftStatus,
} from "@/lib/curriculum/lifecycle";

describe("curriculum draft FSM", () => {
  it("allows draft → in_review", () => {
    expect(() => assertDraftTransition("draft", "in_review")).not.toThrow();
  });

  it("rejects in_review → draft without resume path", () => {
    expect(() => assertDraftTransition("in_review", "draft")).toThrow();
  });

  it("allows rejected → draft", () => {
    expect(() => assertDraftTransition("rejected", "draft")).not.toThrow();
  });

  it("canPublish only when approved", () => {
    expect(canPublish("approved")).toBe(true);
    expect(canPublish("draft")).toBe(false);
  });
});
```

- [ ] **Step 2: Run tests — expect FAIL**

Run: `npm test -- tests/unit/curriculum-lifecycle-fsm.test.ts`
Expected: module not found

- [ ] **Step 3: Implement FSM**

```typescript
// lib/curriculum/lifecycle.ts
import { AppError } from "@/lib/errors";

export type DraftStatus =
  | "draft"
  | "in_review"
  | "approved"
  | "rejected"
  | "changes_requested";

const ALLOWED: Record<DraftStatus, readonly DraftStatus[]> = {
  draft: ["in_review"],
  in_review: ["approved", "rejected", "changes_requested"],
  approved: [], // publish is separate action
  rejected: ["draft"],
  changes_requested: ["draft"],
};

export function assertDraftTransition(from: DraftStatus, to: DraftStatus): void {
  if (!ALLOWED[from].includes(to)) {
    throw new AppError(`Cannot transition from ${from} to ${to}`, 400);
  }
}

export function canPublish(status: DraftStatus): boolean {
  return status === "approved";
}
```

- [ ] **Step 4: Run tests — expect PASS**

Run: `npm test -- tests/unit/curriculum-lifecycle-fsm.test.ts`

- [ ] **Step 5: Commit**

```bash
git add lib/curriculum/lifecycle.ts tests/unit/curriculum-lifecycle-fsm.test.ts
git commit -m "feat(curriculum): add draft status FSM for staff publish lifecycle"
```

---

## Task 2: Clone + publish validation modules

**Files:**
- Create: `lib/curriculum/clone-version.ts`
- Create: `lib/curriculum/publish-validation.ts`
- Modify: `lib/errors.ts`

- [ ] **Step 1: Add ValidationError for structured publish failures**

```typescript
// lib/errors.ts — append
export class ValidationError extends AppError {
  constructor(
    message: string,
    readonly details: Record<string, unknown>,
  ) {
    super(message, 400);
    this.name = "ValidationError";
  }
}
```

- [ ] **Step 2: Implement publish-validation**

```typescript
// lib/curriculum/publish-validation.ts
import { parseLessonBlocks } from "@/lib/content-blocks/schema";
import { isAllowedCloudinaryUrl } from "@/lib/media/cloudinary-url";
import { parseQuizQuestions, parseScoringConfig } from "@/lib/quiz/schema";
import type { ContentBlock } from "@/lib/content-blocks/schema";

export type WorkspaceLessonForPublish = {
  slug: string;
  blocks: unknown;
  quiz: { questions: unknown; scoringConfig: unknown } | null;
};

export type PublishValidationResult = {
  errors: Record<string, string[]>;
  missingQuizSlugs: string[];
};

function collectImageErrors(blocks: ContentBlock[]): string[] {
  const errors: string[] = [];
  for (const block of blocks) {
    if (block.type === "image" && !isAllowedCloudinaryUrl(block.cloudinaryUrl)) {
      errors.push("Image URL is not allowed.");
    }
  }
  return errors;
}

export function validateWorkspaceForPublish(
  lessons: WorkspaceLessonForPublish[],
): PublishValidationResult {
  const errors: Record<string, string[]> = {};
  const missingQuizSlugs: string[] = [];

  for (const lesson of lessons) {
    const lessonErrors: string[] = [];
    const blocksResult = parseLessonBlocks(lesson.blocks);
    if (!blocksResult.success) {
      lessonErrors.push(...blocksResult.error.issues.map((i) => i.message));
    } else {
      lessonErrors.push(...collectImageErrors(blocksResult.data));
    }

    if (lesson.quiz) {
      const q = parseQuizQuestions(lesson.quiz.questions);
      const s = parseScoringConfig(lesson.quiz.scoringConfig);
      if (!q.success) lessonErrors.push("Invalid quiz questions.");
      if (!s.success) lessonErrors.push("Invalid scoring config.");
    } else {
      missingQuizSlugs.push(lesson.slug);
    }

    if (lessonErrors.length > 0) errors[lesson.slug] = lessonErrors;
  }

  return { errors, missingQuizSlugs };
}
```

- [ ] **Step 3: Implement clone-version (PoolClient helper)**

Deep-copy one `curriculum_version` row + its `main` track + all `lesson_version` + new `quiz_version` per lesson quiz.

```typescript
// lib/curriculum/clone-version.ts
import type { PoolClient } from "pg";

type CloneOptions = {
  versionNumber: number;
  status: "published";
  publishedAt?: Date | null;
  publishedBy?: string | null;
};

export async function cloneCurriculumVersion(
  client: PoolClient,
  sourceCurriculumVersionId: string,
  curriculumId: string,
  options: CloneOptions,
): Promise<string> {
  const inserted = await client.query<{ id: string }>(
    `insert into curriculum_version (
       curriculum_id, version_number, status, published_at, published_by
     ) values ($1, $2, $3, $4, $5)
     returning id`,
    [
      curriculumId,
      options.versionNumber,
      options.status,
      options.publishedAt ?? null,
      options.publishedBy ?? null,
    ],
  );
  const newCvId = inserted.rows[0].id;

  const { rows: tracks } = await client.query<{
    position: number;
    slug: string;
    title: unknown;
  }>(
    `select position, slug, title from track
     where curriculum_version_id = $1 order by position`,
    [sourceCurriculumVersionId],
  );

  for (const track of tracks) {
    const trackInsert = await client.query<{ id: string }>(
      `insert into track (curriculum_version_id, position, slug, title)
       values ($1, $2, $3, $4) returning id`,
      [newCvId, track.position, track.slug, track.title],
    );
    const newTrackId = trackInsert.rows[0].id;

    const { rows: lessons } = await client.query<{
      position: number;
      slug: string;
      title: unknown;
      blocks: unknown;
      quiz_version_id: string | null;
      questions: unknown | null;
      scoring_config: unknown | null;
    }>(
      `select lv.position, lv.slug, lv.title, lv.blocks, lv.quiz_version_id,
              qv.questions, qv.scoring_config
       from lesson_version lv
       left join quiz_version qv on qv.id = lv.quiz_version_id
       join track t on t.id = lv.track_id
       where t.id = (
         select id from track
         where curriculum_version_id = $1 and slug = $2
       )
       order by lv.position`,
      [sourceCurriculumVersionId, track.slug],
    );

    for (const lesson of lessons) {
      let quizVersionId: string | null = null;
      if (lesson.quiz_version_id && lesson.questions && lesson.scoring_config) {
        const qv = await client.query<{ id: string }>(
          `insert into quiz_version (questions, scoring_config)
           values ($1, $2) returning id`,
          [lesson.questions, lesson.scoring_config],
        );
        quizVersionId = qv.rows[0].id;
      }

      await client.query(
        `insert into lesson_version (track_id, position, slug, title, blocks, quiz_version_id)
         values ($1, $2, $3, $4, $5, $6)`,
        [newTrackId, lesson.position, lesson.slug, lesson.title, lesson.blocks, quizVersionId],
      );
    }
  }

  return newCvId;
}
```

- [ ] **Step 4: Commit**

```bash
git add lib/curriculum/clone-version.ts lib/curriculum/publish-validation.ts lib/errors.ts
git commit -m "feat(curriculum): add clone and publish validation helpers"
```

---

## Task 3: Repository lifecycle functions

**Files:**
- Modify: `lib/tenant/repositories/staff-programs.ts`

Add internal helper `loadWorkspaceLessonsForPublish(client, workspaceVersionId)` returning `WorkspaceLessonForPublish[]`.

Add `insertPlatformEvent(client, { eventType, ctx, programId, orgId, curriculumVersionId, payload })`.

- [ ] **Step 1: Write failing integration test skeleton**

Create `tests/integration/staff-publish-lifecycle.test.ts` with `beforeAll` using `ensureStaffTestOrg` from `./helpers/staff-test-org`, seed program + lesson with blocks (copy from `staff-lesson-editor.test.ts`).

First test: `submit for review requires at least one lesson`.

- [ ] **Step 2: Implement `transitionDraftStatus`**

```typescript
export async function transitionDraftStatus(
  ctx: TenantContext,
  input: {
    orgSlug: string;
    programSlug: string;
    toStatus: DraftStatus;
    note?: string;
  },
): Promise<void> {
  requireStaff(ctx);
  return withTenantTransaction(ctx, async (client) => {
    const row = await getProgramRow(client, input.orgSlug, input.programSlug);
    const from = row.draft_status as DraftStatus;
    assertDraftTransition(from, input.toStatus);

    if (from === "draft" && input.toStatus === "in_review") {
      const count = await countWorkspaceLessons(
        client,
        row.curriculum_id,
        row.active_published_version_id,
      );
      if (count < 1) {
        throw new AppError("Add at least one lesson before submitting for review.", 400);
      }
    }

    await client.query(
      `update curriculum set draft_status = $2::curriculum_draft_status, updated_at = now()
       where id = $1`,
      [row.curriculum_id, input.toStatus],
    );

    if (input.toStatus === "in_review") {
      await insertPlatformEvent(client, {
        eventType: "content.curriculum_submitted_for_review",
        actorUserId: ctx.userId,
        organizationId: /* from row */,
        programId: row.program_id,
        curriculumVersionId: null,
        payload: { note: input.note ?? null },
      });
    }
    // reject/changes_requested: same event type with payload.outcome
  });
}
```

- [ ] **Step 3: Implement `publishCurriculum`**

Flow inside one transaction:
1. Load row; `canPublish(draft_status)` or 400
2. Resolve workspace version id
3. Load lessons; `validateWorkspaceForPublish`
4. If errors → `ValidationError`
5. If `missingQuizSlugs.length && !confirmMissingQuiz` → `ValidationError` with `details.warnings`
6. `nextVersion = max(version_number)+1`
7. `publishedCvId = cloneCurriculumVersion(client, workspaceId, curriculumId, { versionNumber: nextVersion, status: 'published', publishedAt: now, publishedBy: ctx.userId })`
8. Supersede old active if set
9. `update program set active_published_version_id = publishedCvId`
10. `workspaceCloneId = cloneCurriculumVersion(client, publishedCvId, curriculumId, { versionNumber: nextVersion+1, status: 'published' })`
11. Supersede pre-publish workspace cv if not active
12. `update curriculum set draft_status = 'draft'`
13. Insert `content.curriculum_published` event

- [ ] **Step 4: Implement `rollbackCurriculum` + `listCurriculumVersions`**

Rollback: verify target cv belongs to curriculum; ConflictError if already active; repoint active; supersede previous active; event `content.curriculum_rolled_back`.

List: join curriculum_version ordered desc with lesson counts and `isActive` flag.

- [ ] **Step 5: Run integration tests as you add each behavior**

Run: `npm test -- tests/integration/staff-publish-lifecycle.test.ts`

- [ ] **Step 6: Commit**

```bash
git add lib/tenant/repositories/staff-programs.ts tests/integration/staff-publish-lifecycle.test.ts
git commit -m "feat(staff): curriculum lifecycle publish, rollback, and FSM transitions"
```

---

## Task 4: Complete integration test coverage

**Files:**
- Modify: `tests/integration/staff-publish-lifecycle.test.ts`

- [ ] **Step 1: Add tests per spec §10**

| Test | Assert |
| --- | --- |
| Full happy path | draft → in_review → approved → publish; active set |
| Publish immutability | Direct SQL UPDATE on published lesson raises |
| Workspace isolation | `updateDraftLessonBlocks` after publish doesn't change published blocks |
| Post-publish workspace | `findWorkspaceVersionId` returns clone; slug count matches |
| Rollback | active moves; enrollment unchanged if seeded |
| Events | rows in `platform_event` for publish + rollback |
| Illegal transition | `transitionDraftStatus` in_review→draft throws |

- [ ] **Step 2: Run full suite**

Run: `npm test`
Expected: all tests pass

- [ ] **Step 3: Commit**

```bash
git add tests/integration/staff-publish-lifecycle.test.ts
git commit -m "test(staff): cover publish lifecycle integration scenarios"
```

---

## Task 5: Validation schemas + server actions

**Files:**
- Modify: `lib/validation/staff-program.ts`
- Modify: `lib/staff/actions.ts`

- [ ] **Step 1: Add Zod schemas** (per spec §7.4)

- [ ] **Step 2: Add server actions**

Pattern mirrors `createProgramAction`:
- Parse formData
- Call repo
- On `ValidationError`, map `details` to `StaffActionResult`
- On success, `revalidateStaffProgram` + redirect `?published=1` / `?submitted=1` / etc.

Actions:
- `submitForReviewAction(orgSlug, programSlug, prev, formData)`
- `approveCurriculumAction`
- `rejectCurriculumAction` (note field)
- `requestChangesAction` (note field)
- `resumeEditingAction`
- `publishCurriculumAction` (hidden `confirmMissingQuiz` field)
- `rollbackCurriculumAction` (hidden `targetVersionId`)

- [ ] **Step 3: Commit**

```bash
git add lib/validation/staff-program.ts lib/staff/actions.ts
git commit -m "feat(staff): server actions for curriculum publish lifecycle"
```

---

## Task 6: Staff UI (invoke Impeccable before ship)

**Files:**
- Create: `components/staff/PublishingLifecyclePanel.tsx`
- Create: `components/staff/PublishConfirmDialog.tsx` (`"use client"`)
- Create: `components/staff/VersionHistoryTable.tsx`
- Modify: `app/staff/organizations/[orgSlug]/programs/[programSlug]/page.tsx`

- [ ] **Step 1: Extend overview data fetch**

Load `listCurriculumVersions` + optional `getPublishWarnings` when `draftStatus === 'approved'`.

- [ ] **Step 2: PublishingLifecyclePanel**

Server component with `<form action={...}>` buttons per status table (spec §8.3). Reject/changes show optional `<textarea name="note">`.

- [ ] **Step 3: PublishConfirmDialog**

Client wrapper: if warnings returned from server on first submit, show checkbox "Publish without quizzes on: slug1, slug2" and resubmit with `confirmMissingQuiz=1`.

- [ ] **Step 4: VersionHistoryTable**

Table: version #, date, lesson count, Active badge, Rollback form with confirm.

- [ ] **Step 5: Add StaffFlashToast entries**

`published`, `submitted`, `approved`, `rolledBack` query params.

- [ ] **Step 6: Run `/i-impeccable polish` on new components** (or manual pass against `design.md`)

- [ ] **Step 7: Commit**

```bash
git add components/staff/PublishingLifecyclePanel.tsx components/staff/PublishConfirmDialog.tsx components/staff/VersionHistoryTable.tsx app/staff/organizations/
git commit -m "feat(staff): publish lifecycle panel and version history on overview"
```

---

## Task 7: Docs + spec status

- [ ] **Update `docs/AGENT-PLATFORM.md`** Staff Studio row: publish + rollback shipped
- [ ] **Set spec Status:** Implemented; approval log dated
- [ ] **Linear:** Mark ARC-26 and ARC-27 Done
- [ ] **Commit**

```bash
git add docs/
git commit -m "docs: staff publish lifecycle implemented (ARC-26, ARC-27)"
```

---

## Self-review (plan vs spec)

| Spec requirement | Task |
| --- | --- |
| FSM transitions §4 | Task 1, 3 |
| Review events §4.2 | Task 3 `transitionDraftStatus` |
| Publish transaction §5 | Task 2, 3 |
| Post-publish clone §5.4 | Task 3 `publishCurriculum` |
| Rollback §6 | Task 3 |
| Version list §6.2 | Task 3 |
| Server actions §7.3 | Task 5 |
| UI §8 | Task 6 |
| Error handling §9 | Task 2 ValidationError, Task 3 |
| Tests §10 | Task 1, 4 |

No TBD placeholders. Type names consistent: `DraftStatus`, `cloneCurriculumVersion`, `validateWorkspaceForPublish`.

---

## Execution handoff

Plan saved. Recommended: **inline execution** in this session — backend Tasks 1–4 first, then UI Task 6, verify with `npm test` + manual staff Overview check on dev server.
