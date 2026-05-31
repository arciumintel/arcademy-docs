# Staff Curriculum Outline (Slice B) — Design Spec

**Status:** Draft (brainstorm locked)  
**Date:** 2026-05-29  
**Parent spec:** [2026-05-28-staff-program-shell-design.md](./2026-05-28-staff-program-shell-design.md)  
**Platform context:** [AGENT-PLATFORM.md](../../AGENT-PLATFORM.md) · Phase 2 Staff Studio

---

## 1. Summary

Slice B completes the **curriculum outline** for Staff Studio: staff can manage the full draft lesson path for any program (generic, not Arcium-specific). This slice adds lessons beyond the first, reorders them, edits lesson metadata, and opens a per-lesson workspace route. It does **not** include block editing, quizzes, publish, or multi-track UI.

**4-week context (locked decisions):**

| Decision | Choice |
| --- | --- |
| Build order | Strict waterfall — Slice B before editor/publish |
| Primary users (4 weeks) | Staff only |
| Program scope | Generic (any org/program) |
| Tracks | Single implicit `main` track only |
| Reorder UX | Up / Down buttons (no drag-and-drop) |
| Quiz before publish | Warn only in Slice C+ — not a Slice B concern |

**Outcome:** Staff can go from empty program → ordered draft lesson list → lesson detail stub, ready for Slice C block editor.

---

## 2. Scope

### In scope

| Area | Deliverable |
| --- | --- |
| Lesson list | Ordered list on curriculum page with position, title, slug |
| Add lesson | Create lesson 2+ (generalize first-lesson flow) |
| Reorder | Move lesson up/down within `main` track |
| Edit metadata | Update draft lesson title + slug |
| Delete lesson | Remove draft lesson when program has ≥2 lessons (optional guard: block delete if only lesson) |
| Lesson workspace route | Per-lesson page with metadata + “Editor coming in next slice” stub |
| Overview (≥1 lesson) | Lesson count, link to curriculum, secondary link to first lesson |
| Repository layer | Extend `staff-programs.ts` — no new draft storage model |
| Tests | Integration tests for add, reorder, edit, delete, staff gate |

### Out of scope (Slice C / later)

- Block editor, image upload UI, quiz panel
- Publish, version history, rollback
- Multi-track UI (tracks remain hidden; one `main` track auto-created)
- Partner intake, preview tokens
- Quiz-required publish gate (warn-only when publish ships in week 4)

---

## 3. Routes

```
/staff/organizations/[orgSlug]/programs/[programSlug]/curriculum
  page.tsx                          — lesson list + add-lesson panel

/staff/organizations/[orgSlug]/programs/[programSlug]/curriculum/lessons/[lessonSlug]
  page.tsx                          — lesson workspace stub (Slice C entry point)
```

**Sidebar:** Existing `StaffProgramShell` — **Curriculum** stays active on both routes.

**Breadcrumb:** `{Org} / Programs / {Program} / Curriculum` (lesson route adds lesson title).

---

## 4. Draft storage (reuse Slice A model)

No schema migration required.

| Concept | Implementation |
| --- | --- |
| Working draft | Latest `curriculum_version` distinct from `program.active_published_version_id`, or sole version when never published |
| Track | Single `main` track on workspace version (existing `createFirstDraftLesson` behavior) |
| Lesson rows | `lesson_version` with `position`, `slug`, `title` JSONB, `blocks` placeholder |
| Mutability | RLS + triggers allow UPDATE/DELETE only on workspace version rows |

**Repository rename / generalize:**

- `createFirstDraftLesson` → `createDraftLesson` (same logic; callable when list is empty or non-empty)
- `listWorkspaceLessons` → return `{ slug, title, position }` ordered by `position asc`
- New: `getWorkspaceLesson`, `updateDraftLessonMetadata`, `reorderDraftLesson`, `deleteDraftLesson`

All writes via `withTenantTransaction` + `requireStaff(ctx)`.

---

## 5. Screen behavior

### 5.1 Curriculum page — empty (0 lessons)

Unchanged: centered **Create first lesson** form (`CreateFirstLessonForm`).

### 5.2 Curriculum page — with lessons

**Header**

- Kicker: `Curriculum · Draft · {n} lessons`
- H1: program title (or “Curriculum” if redundant — prefer program title for consistency with Slice A)

**Lesson list** (vertical, editorial — not a data table)

Each row:

- Position number (mono, muted): `01`, `02`, …
- Title (link → lesson workspace route)
- Slug (mono, muted)
- Actions (text buttons or icon row):
  - **Up** — disabled on first row
  - **Down** — disabled on last row
  - **Edit** — inline expand or small modal for title + slug (server action)
  - **Delete** — confirm dialog; disabled when only one lesson remains

**Add lesson panel** (below list or collapsible)

- Reuse `CreateFirstLessonForm` fields; copy changes:
  - Kicker: `Add lesson`
  - H2: **Add another lesson**
  - Submit: **Add lesson**
- On success: revalidate curriculum page; optional flash `?lessonCreated=1`

### 5.3 Lesson workspace stub

Route: `.../curriculum/lessons/[lessonSlug]`

**Content**

- Kicker: `Lesson · Draft`
- H1: lesson title
- Slug + position (read-only display)
- Body card: “Content editor ships next. Blocks and quizzes will be editable here.”
- Primary: disabled **Edit content** (or hidden until Slice C)
- Secondary: **Back to curriculum**

**404:** Unknown slug or lesson on published-only version → `notFound()`.

### 5.4 Overview page (≥1 lesson)

Replace generic summary card with:

- Kicker: `Curriculum · {n} lessons`
- Body: one line on draft status
- Primary CTA: **View curriculum**
- Secondary: **Open first lesson** → first lesson workspace route (by position)

Keep empty state unchanged for `lessonCount === 0`.

---

## 6. Server actions

Extend `lib/staff/actions.ts`:

| Action | Input | Result |
| --- | --- | --- |
| `createDraftLessonAction` | orgSlug, programSlug, title, slug | redirect or field errors (rename from `createFirstLessonAction`) |
| `updateDraftLessonAction` | orgSlug, programSlug, lessonSlug, title, slug | field errors or success + revalidate |
| `reorderDraftLessonAction` | orgSlug, programSlug, lessonSlug, direction: `up` \| `down` | revalidate |
| `deleteDraftLessonAction` | orgSlug, programSlug, lessonSlug | revalidate or error if last lesson |

**Validation (Zod):** Reuse `createFirstLessonSchema` as `draftLessonSchema` for create + update.

**Reorder algorithm:** Swap `position` with adjacent lesson in same track inside one transaction. Handle unique `(track_id, position)` via brief three-step swap if needed.

---

## 7. Components (suggested)

| Component | Responsibility |
| --- | --- |
| `CreateDraftLessonForm` | Rename/generalize `CreateFirstLessonForm`; `variant: "first" \| "add"` for copy |
| `CurriculumLessonList` | Ordered list + reorder/edit/delete controls |
| `EditDraftLessonForm` | Inline or panel for title/slug patch |
| `LessonWorkspaceStub` | Slice B placeholder content on lesson route |

---

## 8. Authorization & errors

- All routes: staff layout gate (existing)
- All mutations: `requireStaff(ctx)` in repository
- Slug collision: `ConflictError` → field error on slug
- Delete last lesson: `AppError` 400 — “Programs must keep at least one draft lesson.”
- Cross-org: existing `NotFoundError` on bad org/program slug

---

## 9. Testing

| Test | Expectation |
| --- | --- |
| Staff adds second lesson | Two rows ordered by position |
| Reorder down | Positions swap; list order updates |
| Update title/slug | Persists on workspace `lesson_version` only |
| Delete with 2+ lessons | Row removed; positions compact optional (gaps ok if positions stay dense via reorder) |
| Delete last lesson | 400 / action error |
| Non-staff POST | 403 |
| Lesson workspace GET | 200 for staff + valid slug; 404 for unknown |
| Published version rows | Reorder/update/delete do not affect `active_published_version_id` lessons |

---

## 10. Slice C handoff

When Slice B ships, lesson workspace route becomes the mount point for:

- Block editor (read/write `lesson_version.blocks`)
- Quiz panel (optional — warn if missing at publish time)
- Image upload via `POST /api/staff/media/upload`
- Learner preview link

No URL changes required — only replace stub body with editor.

---

## 11. Approval log

| Section | Status | Date |
| --- | --- | --- |
| Locked product decisions | Approved (brainstorm) | 2026-05-29 |
| Routes & behavior | Draft | 2026-05-29 |
| Backend | Draft | 2026-05-29 |
