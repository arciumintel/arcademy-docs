# Staff Curriculum Outline (Slice B) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Staff can manage a full draft lesson path — add, reorder, edit metadata, delete, and open a lesson workspace stub — for any org/program.

**Architecture:** Extend [`lib/tenant/repositories/staff-programs.ts`](../../lib/tenant/repositories/staff-programs.ts) with workspace lesson CRUD + reorder (transaction-safe position swap). Server actions in [`lib/staff/actions.ts`](../../lib/staff/actions.ts). UI: generalized draft lesson form, curriculum lesson list, lesson stub route under existing `StaffProgramShell`.

**Tech Stack:** Next.js 16 App Router, React 19, Zod 4, `pg` + RLS, Vitest integration tests.

**Spec:** [2026-05-29-staff-curriculum-outline-design.md](../specs/2026-05-29-staff-curriculum-outline-design.md)

---

## File map

| File | Action |
| --- | --- |
| `lib/validation/staff-program.ts` | Add `draftLessonSchema`; alias `createFirstLessonSchema` |
| `lib/tenant/repositories/staff-programs.ts` | Add position to list; `createDraftLesson`, `getWorkspaceLesson`, `updateDraftLessonMetadata`, `reorderDraftLesson`, `deleteDraftLesson` |
| `lib/staff/actions.ts` | Rename/create draft lesson actions; add update/reorder/delete |
| `components/staff/CreateDraftLessonForm.tsx` | Generalize first-lesson form (`variant`) |
| `components/staff/CurriculumLessonList.tsx` | List + up/down/edit/delete |
| `components/staff/EditDraftLessonPanel.tsx` | Inline edit form |
| `app/staff/.../curriculum/page.tsx` | Wire list + add panel |
| `app/staff/.../curriculum/lessons/[lessonSlug]/page.tsx` | Lesson workspace stub |
| `app/staff/.../programs/[programSlug]/page.tsx` | Overview links to first lesson |
| `tests/integration/staff-curriculum-outline.test.ts` | Slice B repository tests |

---

## Tasks

### Task 1: Validation + repository extensions
- Add `draftLessonSchema`, extend `StaffLessonRow` with `position`
- Implement repository functions with staff gate + workspace-only writes
- Keep `createFirstDraftLesson` as deprecated alias → `createDraftLesson`

### Task 2: Server actions
- `createDraftLessonAction`, `updateDraftLessonAction`, `reorderDraftLessonAction`, `deleteDraftLessonAction`
- Revalidate curriculum + overview paths

### Task 3: UI components + pages
- `CreateDraftLessonForm`, `CurriculumLessonList`, lesson stub route
- Update overview + curriculum pages

### Task 4: Integration tests
- Second lesson, reorder, update, delete guard, get workspace lesson
