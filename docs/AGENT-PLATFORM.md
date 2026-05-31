# Agent context: Arcademy platform

**Read this before scaffolding, migrations, routes, or any feature that touches programs, lessons, partners, or the hub.**

Arcademy is **greenfield** **ecosystem onboarding infrastructure** for the Arcium ecosystem—not an education platform or generic LMS. Each partner runs a **Program** (structured lessons + comprehension quizzes) under one curated hub. **Arcium Fundamentals** ships as **Program #1** at launch — seeded like any other program, not as a special code path.

When this file and the full design spec disagree, **follow the spec** unless the user explicitly changes direction.

---

## One-line thesis

Arcademy is ecosystem onboarding infrastructure: guided programs, comprehension checks, and measurable progress for new user onboarding and developer adoption across the Arcium ecosystem. Internally, the platform owns structure, versioning, tenant isolation, and progress integrity; users get one account and a curated hub.

**Partner outcomes (product north star):** reduce onboarding friction, improve activation, standardize onboarding, identify drop-off points, prepare developers before support requests.

---

## What Arcademy is / is not

**Is:** Ecosystem onboarding infrastructure — hub discovery, program-scoped curricula, block-structured lessons, platform-defined quizzes with partner guardrails, staff-governed publishing, event-backed progress, reading-first lesson UX.

**Is not (v1):** Education platform or generic LMS (SCORM, classrooms, marketplaces), partner CMS/page builder, custom question types, cross-program leaderboards, partner SSO, embed SDK, webhooks, billing, push notifications, analytics warehouse.

Scope test: *Does this help an ecosystem team onboard new users—including developers—and verify they are ready to use or integrate the product?* If no → defer.

---

## Locked product decisions

| Topic | Decision |
| --- | --- |
| **Hub** | Single catalog: `/` → `/programs` → `/programs/[slug]` → lessons |
| **Launch content** | Arcium = first program (`program.slug = arcium`); seed via migrations/scripts |
| **Learner accounts** | Global Arcademy account; per-program enrollment + progress |
| **Authoring default** | Staff-built; new partners start **untrusted** (intake only) |
| **Partner self-serve** | Manual trust gate → draft-only Partner Studio (Phase 3); **staff publish only in v1** |
| **Partner quizzes** | Trusted partners author quizzes in draft using **platform question types** and configurable guardrails — not custom schemas |
| **Published content** | Immutable `curriculum_version` / `lesson_version` / `quiz_version`; never UPDATE published rows |
| **Draft edits** | Always against the program's **working draft curriculum**; publish creates new immutable snapshots |
| **Enrollment pinning** | `program_enrollment.curriculum_version_id` pinned at enroll/first activity; **unchanged** when staff publishes a newer version (v1 migration is manual staff action) |
| **Onboarding integrity v1** | Progress continuity, verified completion, comprehension checks; **no** streaks/badges/leaderboards as product focus |
| **Lesson body** | Structured blocks (Zod-validated); no author-supplied raw HTML |
| **Tenant isolation** | Tenant context + scoped repositories + Postgres RLS — **all three, from day one** |
| **Content storage** | Postgres only at runtime; no JSON seed files or Keystatic as lesson source of truth |
| **Infra assumption** | Next.js on Vercel, Postgres on Neon, **shared DB across envs until explicitly changed** — every migration is production-impactful |

---

## Expected stack

| Area | Choice |
| --- | --- |
| Framework | Next.js **16** (App Router), **React 19** |
| Language | TypeScript **strict** |
| Styling | Tailwind CSS **3**, `darkMode: "class"`, design tokens in `theme.extend` |
| Database | PostgreSQL via **`pg`** pooled client — `lib/db.ts` |
| Auth | **better-auth** — `lib/auth.ts`, `/api/auth/[...all]` |
| Validation | **Zod 4** — shared schemas in `lib/validation.ts` patterns |
| Media | Cloudinary HTTPS URLs with allowlist |
| Imports | Path alias **`@/*`** → repo root |
| Hosting | Vercel (app) + Neon (Postgres) |

**Env:** `DATABASE_URL` (or project-standard name documented in `AGENTS.md`). Never commit secrets.

**NPM scripts (establish early):** `dev`, `build`, `db:migrate` (idempotent SQL in `db/migrations/`).

---

## Domain model

```
Organization (partner tenant)
  └── Program (hub listing, learner-facing; hub_status, active_published_version_id)
        └── Curriculum (one working draft container per program)
              └── CurriculumVersion (immutable once published)
                    └── Track (ordered module group)
                          └── LessonVersion (immutable block tree)
                                └── QuizVersion (optional, immutable)
```

**Enrollment:** `user_id + program_id + curriculum_version_id`.

**Progress:** `user_id + lesson_version_id` — always version FKs.

**Quiz attempts:** `user_id + quiz_version_id` — answer keys never sent to the client.

**Slugs:** unique per `(program_id, slug)`, not globally.

### Core tables (Phase 0)

`organization`, `organization_member`, `program`, `curriculum`, `curriculum_version`, `track`, `lesson_version`, `quiz_version`, `program_enrollment`, `lesson_progress`, `quiz_attempt`, `platform_event`, `program_hub_settings`.

Key columns agents must not forget:

- `program.active_published_version_id` — what learners see
- `program.hub_status` — `internal` \| `listed` \| `featured` \| `sunset` \| `archived`
- `program.featured_rank` — staff-ordered 1–6 for hub home (max **6** featured programs)
- `organization.trust_level` — gates Partner Studio access

---

## Content and publish lifecycle

### Draft curriculum (mutable)

States: `draft` → `in_review` → `approved` → **publish** (creates new `curriculum_version`).

Authors edit the working draft only. Partner authors (when trusted) and staff use the same editor surfaces.

### Publish (immutable snapshot)

1. Validate blocks + quiz JSON (Zod).
2. Copy draft into new `curriculum_version`, `lesson_version`, and `quiz_version` rows.
3. Set `program.active_published_version_id`.
4. Emit audit/platform events in the **same transaction** as state writes (Phase 1+).

**Never** UPDATE published version JSON in place.

### Hub visibility (separate from publish)

A program appears on the public hub only when **all** are true: `hub_status` is `listed` or `featured`, `active_published_version_id` is set, ≥1 published lesson exists, staff checklist complete, org not suspended.

---

## URL map

| Route | Phase | Role |
| --- | --- | --- |
| `/` | 1 | Hub home (featured programs, continue where you left off) |
| `/programs` | 1 | Catalog (listed/featured only) |
| `/programs/[programSlug]` | 1 | Program home: path, enroll/continue CTA |
| `/programs/[programSlug]/lessons/[lessonSlug]` | 1 | Lesson player |
| `/account` | 1 | Global learner: enrollments, per-program progress |
| `/staff/*` | 2 | Staff Studio (org/program scoped authoring + publish). **Slices A–C shipped:** program shell, curriculum outline (add/reorder/edit/delete lessons), lesson block editor + quiz + preview — see `docs/superpowers/specs/2026-05-28-staff-program-shell-design.md`, `2026-05-29-staff-curriculum-outline-design.md`, `2026-05-29-staff-lesson-editor-design.md`. |
| `/partner/*` | 3 | Partner Studio (trusted orgs, draft + submit for review) |

There is no `/modules` route. Optional redirects from a prior product belong in edge config or `middleware.ts` — not in core routing design.

---

## Tenant context and RLS

### Context kinds (`lib/tenant/context.ts`)

| Kind | Use |
| --- | --- |
| `anonymous` | Public published content; guest-first lesson 1 (Phase 1) |
| `learner` | Authenticated learner; own progress only |
| `partner` | Org member; draft curriculum for own org when trusted |
| `staff` | Cross-org read/write per role; hub curation and publish |
| `system` | Cron/rollup jobs (Phase 2+); scoped job identity |

All server routes/actions: `resolveTenantContext()` → `requireProgramAccess` / `requireOrganizationAccess` → repository. **No raw `query()` in route handlers.**

### RLS rules (summary)

- **Published** program content: readable without org membership (including anonymous where guest-first applies).
- **Draft** curriculum: org members (trusted) or staff only.
- **Progress / attempts:** owner learner, or staff; never cross-program.
- **Staff bypass:** `app.is_staff = true` policy branch.
- Set session vars at transaction start via `withTenantTransaction(ctx, fn)`:

```sql
SET app.current_user_id = '...';
SET app.current_org_ids = '{...}';
SET app.is_staff = 'true' | 'false';
```

Cross-tenant tests must expect **403 or 404** (never leak via 200 + empty).

---

## Module layout (establish early)

| Path | Responsibility |
| --- | --- |
| `lib/tenant/context.ts` | `resolveTenantContext()` — integrates with better-auth session |
| `lib/tenant/scope.ts` | `requireProgramAccess`, `requireOrganizationAccess` |
| `lib/tenant/repositories/*` | All tenant-scoped reads/writes |
| `lib/db.ts` | Pooled client; `withTenantTransaction` |
| `lib/content-blocks/schema.ts` | Zod block schema v1 |
| `components/lessons/LessonBlockRenderer.tsx` | Single render map; sanitized output |
| `lib/quiz/schema.ts` | Platform question types + scoring config bounds |
| `lib/quiz/public.ts` | Client-safe payloads — **no answer keys** |
| `lib/quiz/score.ts` | Server-side scoring only |
| `lib/events/platform-event.ts` | Append-only writes; fail TX on emit failure |
| `lib/guest/merge-progress.ts` | Guest merge server action (Phase 1) |
| `db/migrations/*.sql` | Idempotent schema |

---

## v1 block types

| Type | Notes |
| --- | --- |
| `heading` | level 2–3, locale text map |
| `paragraph` | restricted markdown → sanitized HTML |
| `callout` | variant note/warning |
| `code` | language + snippet |
| `image` | Cloudinary URL, alt, caption |
| `divider` | horizontal rule |

Text fields: `{ "en": "..." }` (only `en` required in v1). New block types ship with platform releases.

---

## Quiz and partner guardrails

- Quizzes are optional per lesson; snapshotted to `quiz_version` at publish.
- **Platform-defined question types only** — custom types deferred.
- Partners configure within bounds: pass/mastery thresholds, retry cooldown, linear unlock, enabled question-type subset.
- Config stored in `quiz_version.scoring_config` at publish time.
- Rate-limit quiz submissions per user/IP (cooldown + API limits).

---

## Engineering rules

### Do

- Complete **Phase 0** (schema, RLS, repositories, block schema, Arcium seed, isolation tests) before public hub UI.
- Validate blocks and quiz JSON at **author time** and **publish time**.
- Copy-on-publish for all versioned content.
- Emit `platform_event` learner events in the **same transaction** as progress writes (Phase 1+).
- Guest storage key (Phase 1): `arcademy.guest.v2:{programId}:{lessonVersionId}`.
- Log tenant context resolution failures; alert on RLS policy violations.
- Audit staff cross-org access.

### Do not (v1)

- Ship `/programs` learner routes without repositories + RLS.
- Build Staff Studio authoring UI in Phase 0 — use seed scripts; UI lands Phase 2.
- Let partners publish to production or the hub without staff approval.
- Mutate published version rows in place.
- Expose quiz answer keys to the client.
- Add custom question types, leaderboards, SSO, webhooks, embed SDK, or SCORM.
- Ship streaks, XP, cosmetic badges, or token-gated rewards as core product narrative.
- Introduce parallel lesson content systems (JSON in git, Keystatic, etc.).

---

## Phased roadmap

Build in order. Do not skip phases.

| Phase | Goal | Key deliverables |
| --- | --- | --- |
| **0** | Tenancy foundation | Migrations, RLS, repositories, block schema, Arcium seed, isolation tests. **No public hub UI.** |
| **1** | Hub + onboarding loop | Hub routes, lesson player, enrollments, guest merge, `platform_event`, basic staff hub curation |
| **2** | Partner onboarding | Intake, Staff Studio UI, publish/rollback, preview links, analytics rollups |
| **3** | Trusted partner drafts | Partner Studio, approval queue, trust admin, partner analytics |
| **4** | Post-v1 | Proof-of-learning (signed attestations, ecosystem credentials, portable records), webhooks, SSO, embed, block expansion |

### Phase 0 bootstrap sequence

1. Migrations: full tenancy + version tables + key indexes
2. RLS policies + `withTenantTransaction` (resolve open question #4 first if possible)
3. `lib/tenant/*` context, scope, repositories
4. Block schema v1 + `LessonBlockRenderer`
5. Seed org + program `arcium` + curriculum v1 + launch lessons/quizzes (script or migration)
6. Cross-tenant integration tests

**Phase 0 exit:** Arcium content readable through scoped repositories; isolation tests pass. Learner-facing routes come in Phase 1.

**Phase 1 exit:** Arcium on hub; guest can complete lesson 1 and merge progress on signup.

**v1 total:** ~12–16 weeks (spec §9); assumes fewer than 10 partner programs in year one and English-only UI.

---

## Roles (summary)

| Role | Publish | Edit drafts | Hub curation | Org analytics |
| --- | --- | --- | --- | --- |
| Learner | — | — | — | — |
| Untrusted partner | — | — (intake only) | — | — |
| Trusted partner author | — | own org | — | — |
| Partner admin | — | own org | — | read own org |
| Staff | ✓ | any org | ✓ | ✓ |
| Platform admin | ✓ | any org | ✓ + trust flags | ✓ |

Trust gate is **manual**: staff sets `organization.trust_level = self_serve_draft` after QA checklist (spec §4.2).

---

## Open questions (resolve before coding the area)

| # | Question | Phase |
| --- | --- | --- |
| 1 | First non-Arcium pilot partner — which app? | 2 |
| 2 | Enroll on **explicit Enroll button** on program home (not first activity). Pins `active_published_version_id` at enroll time. | 1 |
| 3 | Partner preview tokens: TTL and auth? | 2 |
| 4 | ~~Neon pooler vs unpooled for RLS session vars?~~ **Resolved:** use `DATABASE_URL_UNPOOLED` for migrations, seeds, and all `withTenantTransaction()` calls; pooled URL reserved for future read-only paths. | 0 |
| 6 | Legal: partner terms for user progress data sharing? | 2 |

Record answers here when decided.

**Phase 0 exit (2026-05-26):** Migrations, RLS, repositories, block/quiz schemas, Arcium seed, and cross-tenant integration tests are in place. No public hub UI yet (Phase 1).

---

## Branding

- Product: **Arcademy**
- First launch program slug: **`arcium`**

---

## When to update this file

- A phase exit criterion is met.
- A locked decision or open question is resolved.
- Bootstrap module paths or env conventions change.

Keep this file **short and operational**. Deep design belongs in the spec.
