# Staff Publish Lifecycle (ARC-26 + ARC-27) — Design Spec

**Status:** Implemented  
**Date:** 2026-05-31  
**Linear:** [ARC-26](https://linear.app/arcedu/issue/ARC-26) · [ARC-27](https://linear.app/arcedu/issue/ARC-27)  
**Parent specs:** [2026-05-28-staff-program-shell-design.md](./2026-05-28-staff-program-shell-design.md) · [2026-05-29-staff-curriculum-outline-design.md](./2026-05-29-staff-curriculum-outline-design.md) · [2026-05-29-staff-lesson-editor-design.md](./2026-05-29-staff-lesson-editor-design.md)  
**Platform context:** [AGENT-PLATFORM.md](../../AGENT-PLATFORM.md) · [2026-05-20-ecosystem-platform-design.md](./2026-05-20-ecosystem-platform-design.md) §3.3 · §4.3 · §8.1

---

## 1. Summary

Ship **ARC-26** and **ARC-27** together: a staff-governed curriculum lifecycle (`draft` → `in_review` → `approved` → publish) plus Staff Studio UI to **publish** immutable snapshots and **rollback** the active published version.

Staff can move a program from draft edits through review gates, publish a new `curriculum_version`, and repoint `program.active_published_version_id` to a prior version — without mutating published rows or changing existing enrollment pins.

**Locked decisions:**

| Decision | Choice |
| --- | --- |
| Scope | Bundle ARC-26 FSM + ARC-27 publish/rollback UI |
| Architecture | Monolithic `lib/curriculum/lifecycle.ts` module; repos call it inside transactions |
| Post-publish workspace | **Clone published snapshot** into a new mutable workspace `curriculum_version` |
| Lifecycle UI placement | Program **Overview** page — panel below status chips |
| Quiz missing at publish | **Warn + confirm** — not a hard block (Slice C rule) |
| Review notes | Optional text stored in `platform_event.payload` only (no new DB column) |
| v1 actors | Staff only; FSM matches platform spec for future Partner Studio |
| Implementation PRs | One spec; may split into backend PR then UI PR for review size |

**Outcome:** Staff publish from Studio without seed scripts; learners see published content via `active_published_version_id`; audit trail via `platform_event`.

---

## 2. Scope

### In scope

| Area | Deliverable |
| --- | --- |
| FSM (ARC-26) | `curriculum.draft_status` transitions with guards |
| Publish (ARC-27) | Deep-copy workspace → immutable published snapshot; set active pointer |
| Post-publish workspace | Clone active published snapshot → new mutable workspace version |
| Rollback (ARC-27) | Repoint `active_published_version_id`; audit event |
| Events | `content.curriculum_published`, `content.curriculum_rolled_back`, review transition events |
| Repository API | `transitionDraftStatus`, `publishCurriculum`, `rollbackCurriculum`, `listCurriculumVersions` |
| Server actions | Staff actions in `lib/staff/actions.ts` |
| UI | Lifecycle panel + version history on program Overview |
| Tests | Integration tests for FSM, publish immutability, rollback, events |

### Out of scope

- Partner Studio submit-for-review UI (FSM ready; staff performs all transitions in v1)
- Preview tokens for unpublished drafts (ARC-28)
- Hub listing / featured curation workflow changes
- Enrollment migration to newer versions (`migrate_enrollment_to_version` — manual v1)
- `platform_event_outbox` / webhooks
- Multi-track UI (single implicit `main` track continues)

---

## 3. Routes

No new routes. All actions on existing Overview:

```
/staff/organizations/[orgSlug]/programs/[programSlug]
  page.tsx  — add PublishingLifecyclePanel + VersionHistoryTable
```

Server actions POST via forms (same pattern as `CreateProgramForm`, lesson editor saves).

---

## 4. State machine (ARC-26)

### 4.1 Transition table

| From | To | Action label | Guard |
| --- | --- | --- | --- |
| `draft` | `in_review` | Submit for review | ≥1 workspace lesson |
| `in_review` | `approved` | Approve | — |
| `in_review` | `rejected` | Reject | Optional note |
| `in_review` | `changes_requested` | Request changes | Optional note |
| `rejected` | `draft` | Resume editing | — |
| `changes_requested` | `draft` | Resume editing | — |
| `approved` | *(publish)* | Publish curriculum | Publish validation passes; see §5 |

Illegal transitions return `400` with a clear message. No automatic transitions except publish side-effects (§5.4).

### 4.2 Review events

On each non-publish transition, insert `platform_event`:

| Transition | `event_type` |
| --- | --- |
| → `in_review` | `content.curriculum_submitted_for_review` |
| → `approved` | *(no separate type — optional: include in publish audit only)* |
| → `rejected` | `content.curriculum_submitted_for_review` with `payload.outcome: "rejected"` |
| → `changes_requested` | `content.curriculum_submitted_for_review` with `payload.outcome: "changes_requested"` |
| → `draft` (resume) | *(no event — low signal)* |

Keep v1 minimal: emit `content.curriculum_submitted_for_review` on submit; store approve/reject/changes notes in `payload.note` when provided.

### 4.3 Pure FSM module

```typescript
// lib/curriculum/lifecycle.ts (conceptual)
type DraftStatus = "draft" | "in_review" | "approved" | "rejected" | "changes_requested";

function assertTransition(from: DraftStatus, to: DraftStatus): void;
function canPublish(draftStatus: DraftStatus): boolean; // approved only
```

Unit-test the transition matrix without DB.

---

## 5. Publish transaction (ARC-27)

All steps in **one** `withTenantTransaction` block.

### 5.1 Preconditions

- `ctx.kind === "staff"` (`requireStaff`)
- `curriculum.draft_status === "approved"`
- Workspace version exists with ≥1 lesson on `main` track
- Program resolved by `orgSlug` + `programSlug` (404 cross-tenant)

### 5.2 Publish-time validation

For each workspace lesson (ordered by position):

1. `parseLessonBlocks(blocks)` — Zod
2. If quiz attached: `parseQuizQuestions`, `parseScoringConfig`, Cloudinary allowlist on image blocks
3. Collect errors keyed by lesson slug

If any hard validation errors → abort transaction, return `400` with lesson-level errors.

**Quiz optional:** If any lesson has no quiz, return a **soft warning** to the client before publish (confirm checkbox). Server accepts `confirmMissingQuiz: true` in publish action when warnings present.

### 5.3 Snapshot copy (immutable published version)

1. Allocate `version_number = max(curriculum_version.version_number) + 1`
2. `INSERT curriculum_version` — `status = 'published'`, `published_at = now()`, `published_by = ctx.userId`
3. Deep-copy `main` track → `lesson_version` rows:
   - New `quiz_version` row per lesson quiz (never reuse draft quiz IDs)
   - Copy `blocks`, `title`, `slug`, `position` verbatim (validated JSON)
4. `UPDATE program SET active_published_version_id = new_cv_id`
5. If prior active exists: `UPDATE curriculum_version SET status = 'superseded' WHERE id = old_active_id` (trigger allows this status transition per migration 007)

**Rule:** Never UPDATE existing published `lesson_version` / `quiz_version` rows.

### 5.4 Post-publish workspace clone

After setting active pointer, the prior workspace `curriculum_version` must **not** remain the editable workspace (it may be an older version number still distinct from active — see `findWorkspaceVersionId` in `staff-programs.ts`).

1. Deep-copy **newly published** snapshot into a **new** `curriculum_version` row (`version_number + 1`, status `published` — mutable until linked as active; matches existing workspace insert convention)
2. Copy tracks, lessons, quizzes into this workspace clone (new row IDs throughout)
3. Mark the **pre-publish workspace** cv `status = 'superseded'` if it is not the new active published version
4. `UPDATE curriculum SET draft_status = 'draft'`

Result: `findWorkspaceVersionId(curriculumId, activePublishedId)` returns the clone; staff continue editing from what learners just received.

### 5.5 Audit event

```sql
INSERT platform_event (
  event_type, actor_user_id, organization_id, program_id,
  curriculum_version_id, payload
) VALUES (
  'content.curriculum_published', ..., new_cv_id,
  jsonb_build_object(
    'version_number', N,
    'lesson_count', L,
    'prior_active_version_id', old_id | null
  )
);
```

---

## 6. Rollback (ARC-27)

### 6.1 Behavior

- Staff selects a **prior published** `curriculum_version` from version history
- Confirm dialog copy: *"Learners without an enrollment pin will see this version. Existing enrollments stay on their pinned version."*
- Transaction:
  1. Verify target belongs to program's curriculum and `status IN ('published', 'superseded')`
  2. Verify target ≠ current `active_published_version_id` (else `409`)
  3. `UPDATE program SET active_published_version_id = target_id`
  4. Optionally set previous active to `superseded` if not already
  5. Set target row `status = 'published'` if it was `superseded`
  6. Insert `content.curriculum_rolled_back` event with `payload.from_version_id`, `payload.to_version_id`

**Does not:** delete version rows, mutate enrollments, or change workspace draft content.

### 6.2 Version list query

`listCurriculumVersions(ctx, { orgSlug, programSlug })` returns:

| Field | Source |
| --- | --- |
| `id` | `curriculum_version.id` |
| `versionNumber` | `version_number` |
| `status` | `status` |
| `publishedAt` | `published_at` |
| `publishedBy` | `published_by` (join user display if available) |
| `isActive` | `id === program.active_published_version_id` |
| `lessonCount` | count via track → lesson_version |

Ordered by `version_number DESC`.

---

## 7. Repository and action layer

### 7.1 New module

| File | Responsibility |
| --- | --- |
| `lib/curriculum/lifecycle.ts` | FSM asserts, publish/rollback orchestration helpers |
| `lib/curriculum/clone-version.ts` | Deep-copy cv + tracks + lessons + quizzes (shared by publish + post-publish clone) |
| `lib/curriculum/publish-validation.ts` | Aggregate workspace validation + quiz warnings |

### 7.2 Extensions to `staff-programs.ts`

Export thin wrappers that call lifecycle inside `withTenantTransaction`:

- `transitionDraftStatus(ctx, { orgSlug, programSlug, toStatus, note? })`
- `publishCurriculum(ctx, { orgSlug, programSlug, confirmMissingQuiz? })`
- `rollbackCurriculum(ctx, { orgSlug, programSlug, targetVersionId })`
- `listCurriculumVersions(ctx, { orgSlug, programSlug })`
- `getPublishWarnings(ctx, { orgSlug, programSlug })` — optional read for UI confirm dialog

### 7.3 Server actions (`lib/staff/actions.ts`)

| Action | Revalidates |
| --- | --- |
| `submitForReviewAction` | program overview |
| `approveCurriculumAction` | overview |
| `rejectCurriculumAction` | overview |
| `requestChangesAction` | overview |
| `resumeEditingAction` | overview |
| `publishCurriculumAction` | overview + curriculum + lesson routes |
| `rollbackCurriculumAction` | overview |

Use existing `StaffActionResult` error shape. Success → redirect with flash query param (existing toast pattern).

### 7.4 Validation schemas (`lib/validation/staff-program.ts`)

```typescript
export const transitionDraftStatusSchema = z.object({
  toStatus: z.enum(["in_review", "approved", "rejected", "changes_requested", "draft"]),
  note: z.string().trim().max(2000).optional(),
});

export const publishCurriculumSchema = z.object({
  confirmMissingQuiz: z.coerce.boolean().optional(),
});

export const rollbackCurriculumSchema = z.object({
  targetVersionId: z.string().uuid(),
});
```

---

## 8. Staff UI

### 8.1 Components

| Component | Responsibility |
| --- | --- |
| `PublishingLifecyclePanel.tsx` | Contextual actions from `draftStatus`; note field on reject/changes |
| `PublishConfirmDialog.tsx` | Client confirm for publish; shows validation errors + quiz warnings |
| `VersionHistoryTable.tsx` | Version list + rollback button + active badge |
| `ResumeEditingButton.tsx` | `rejected` / `changes_requested` → `draft` |

Follow [`design.md`](../../../design.md) tokens and existing staff patterns (`ProgramStatusChips`, `StaffFlashToast`).

### 8.2 Overview layout

```
[Header + title]
[ProgramStatusChips]
[PublishingLifecyclePanel]     ← new
[VersionHistoryTable]          ← new (when ≥1 published version exists)
[Curriculum summary card]      ← existing
```

### 8.3 Action visibility by status

| `draft_status` | Primary | Secondary |
| --- | --- | --- |
| `draft` | Submit for review | — |
| `in_review` | Approve | Request changes · Reject |
| `approved` | Publish curriculum | — |
| `rejected` / `changes_requested` | Resume editing | — |

Publish button uses accent styling with confirmation step. Rollback uses muted destructive confirm per version row.

### 8.4 Preview link

Existing gate unchanged: preview enabled when `active_published_version_id` set and hub listable. Publish success toast mentions preview availability when `hub_status` allows.

---

## 9. Error handling

| Case | HTTP / result |
| --- | --- |
| Illegal FSM transition | 400 — `"Cannot transition from {from} to {to}"` |
| Publish when not `approved` | 400 |
| Publish with validation errors | 400 — `{ errors: { lessons: { [slug]: string[] } } }` |
| Publish with missing quiz, no confirm flag | 400 — `{ warnings: { missingQuizSlugs: string[] } }` |
| Rollback to current active | 409 |
| Rollback to version outside program | 404 |
| Non-staff | 403 |
| Cross-tenant program | 404 |

---

## 10. Testing

**File:** `tests/integration/staff-publish-lifecycle.test.ts`

| Test | Expectation |
| --- | --- |
| Submit without lessons | 400 |
| Full happy path | draft → in_review → approved → publish |
| Publish creates new cv | `active_published_version_id` updated; lesson slugs match |
| Published immutability | UPDATE on published lesson_version raises |
| Workspace isolation | Edit workspace after publish does not change published lesson blocks |
| Post-publish workspace | New workspace cv exists; clone matches published content |
| Rollback | Active pointer moves; enrollment row unchanged |
| Events | `content.curriculum_published` and `content.curriculum_rolled_back` inserted |
| Illegal transition | in_review → draft without resume action rejected |

Reuse `isolation-fixtures` staff user seed patterns from existing integration tests.

**Unit:** `tests/unit/curriculum-lifecycle-fsm.test.ts` for transition matrix.

---

## 11. Implementation notes

### 11.1 Workspace version status quirk

`getOrCreateWorkspaceVersionId` inserts workspace rows with `status = 'published'`. Mutability is determined by **not** being `active_published_version_id`, not by status alone. Do not change this convention in v1 — clone logic relies on immutability triggers.

### 11.2 Single track

Continue auto-creating / using single `main` track slug on workspace and published copies (Slice B/C behavior).

### 11.3 Arcium bootstrap

Existing Arcium v1 seed remains valid. First staff publish on Arcium creates v2+; no migration required.

### 11.4 Docs follow-up

After implementation, update `docs/AGENT-PLATFORM.md` Staff Studio row to mention publish/rollback shipped.

---

## 12. Approval log

| Section | Status | Date |
| --- | --- | --- |
| Scope + locked decisions | Approved (plan) | 2026-05-31 |
| FSM + publish + rollback | Approved (plan) | 2026-05-31 |
| UI + testing | Approved (plan) | 2026-05-31 |
| Written spec review | Approved | 2026-05-31 |

---

## 13. References

- [`lib/tenant/repositories/staff-programs.ts`](../../../lib/tenant/repositories/staff-programs.ts) — workspace model
- [`db/migrations/007_indexes_immutability.sql`](../../../db/migrations/007_indexes_immutability.sql) — immutability triggers
- [`db/migrations/004_progress_events.sql`](../../../db/migrations/004_progress_events.sql) — `platform_event` schema
- [`components/staff/ProgramStatusChips.tsx`](../../../components/staff/ProgramStatusChips.tsx) — status display
- [`app/staff/organizations/[orgSlug]/programs/[programSlug]/page.tsx`](../../../app/staff/organizations/[orgSlug]/programs/[programSlug]/page.tsx) — Overview target
