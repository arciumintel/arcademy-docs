# Staff Lesson Editor (Slice C) — Design Spec

**Status:** Implemented  
**Date:** 2026-05-29  
**Parent spec:** [2026-05-28-staff-program-shell-design.md](./2026-05-28-staff-program-shell-design.md)  
**Depends on:** [2026-05-29-staff-curriculum-outline-design.md](./2026-05-29-staff-curriculum-outline-design.md)

---

## Summary

Slice C replaces the lesson workspace stub with a **draft block editor**, **quiz panel**, and **inline preview** for Staff Studio. Staff edit `lesson_version.blocks` and optional `quiz_version` on the mutable workspace curriculum only.

## In scope

| Area | Deliverable |
| --- | --- |
| Read | `getWorkspaceLessonContent` — blocks + quiz for draft lesson |
| Blocks | Form-based list: add/remove, Up/Down reorder, save via server action |
| Quiz | Optional questions + scoring; upsert/clear; warn banner when missing |
| Media | Cloudinary upload field + allowlist validation on save |
| Preview | Client tab using `LessonBlockRenderer` (not public hub draft routes) |
| Tests | Integration tests for read/save blocks/quiz/clear/bad image URL |

## Out of scope

- Publish flow and preview tokens
- Markdown / raw HTML authoring
- Multi-locale UI (English `en` fields only)
- Drag-and-drop reorder

## Routes

Unchanged: `/staff/organizations/[orgSlug]/programs/[programSlug]/curriculum/lessons/[lessonSlug]`

## Data rules

- All writes via `resolveTenantContext()` → `staff-programs` repository
- `parseLessonBlocks` / quiz schemas at validation boundary
- Image URLs must pass `isAllowedCloudinaryUrl`
- Published snapshots remain immutable (DB triggers)
