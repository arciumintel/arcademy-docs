# Arcademy Ecosystem Onboarding Infrastructure — Design Spec (Revised)

**Status:** Draft for review  
**Date:** 2026-05-20  
**Supersedes:** Brainstorm synthesis from 2026-05-20 (pre-revision)  
**Audience:** Product, engineering, partner stakeholders

---

## 1. Executive summary

Arcademy is **ecosystem onboarding infrastructure** for apps in the Arcium ecosystem—not an education platform or generic LMS. Each ecosystem partner runs a **Program** (structured lessons + comprehension quizzes) for its users under one **Arcademy hub**, with **Arcium as Program #1**.

This revision strengthens the original plan where it was vague: **tenancy is first-class**, **published content is versioned and immutable**, **tenant isolation is enforced in the database and application layer**, **lesson content is block-structured**, **onboarding integrity over gamification for v1**, and **events + analytics are designed in from the start** rather than bolted on later.

**v1 goal (honest scope):** A user can discover curated programs on the hub, enroll, complete a linear onboarding path with comprehension checks, and retain progress across sessions—including guest-first entry on lesson 1. Staff can onboard partners, build their first program, and publish curated hub listings. One trusted pilot partner can self-serve draft content under staff review.

**Explicitly not v1:** Education-platform positioning; generic LMS features (course marketplaces, SCORM, arbitrary page builders, cross-program leaderboards, push notifications, seasonal resets, partner billing, embed SDK).

**Estimated calendar time to v1:** 12–16 weeks with 1–2 engineers (not 6–8 weeks). Phases below reflect dependencies, not parallel wishful thinking.

---

## 2. Revised strategic thesis

### What Arcademy is

Arcademy is **ecosystem onboarding infrastructure** for the Arcium ecosystem:

- Partners need new users to **activate** and developers to **integrate**—not just click through docs—and to do so with less friction, **consistent onboarding**, **visible drop-off**, and **fewer unprepared support requests**.
- Users get **progress continuity**, **verified completion**, and a path toward **ecosystem readiness**—not streaks or cosmetic rewards.
- Arcademy owns **structure, integrity, and trust**—not partner product logic.

**Partner outcomes:**

| Outcome | Mechanism |
| --- | --- |
| Reduce onboarding friction | Guided programs replace doc sprawl; one hub link per product |
| Improve activation | Comprehension checks gate “ready to build” confidence |
| Standardize onboarding | Block schema + platform quiz types + staff/trust governance |
| Identify drop-off points | Lesson-version progress + funnel analytics (Phase 2+) |
| Prepare developers before support | Progress + quiz pass signal before deep integration / support |

**One-line thesis:** *Arcademy is ecosystem onboarding infrastructure—guided programs, comprehension checks, and measurable progress for new user onboarding and developer adoption across the Arcium ecosystem.*

### What Arcademy is not

| Not this | Why |
|----------|-----|
| Education platform / ed-tech LMS | We onboard users and developers into protocols and apps—not classrooms, curricula, or instructor-led courses. |
| Generic LMS (Canvas, Moodle) | We don't do arbitrary courses, grading rubrics, instructor classrooms, or SCORM. |
| Partner CMS / marketing site builder | Lesson blocks are structured and constrained; partners don't own layout/CSS. |
| Full gamification platform | v1 ships comprehension + progress integrity; streaks/badges/leaderboards/tokens are out of scope |
| Analytics warehouse | v1 ships operational metrics and exports; not a BI product. |
| White-label auth provider | Global Arcademy account; partner SSO is post-v1. |

### Preserved from original plan (still correct)

- Single **hub** discovery model (`/` → `/programs/[slug]`).
- **Hybrid authoring:** staff-built default; self-serve only after manual trust gate.
- **Arcium = Program #1** (migrate in place, redirect legacy URLs).
- **Global learner account** with per-program progress.
- **Reading-first** lesson UX and editorial visual language.
- **Configurable guardrails** for partners (thresholds, enabled question types)—not custom question schemas.

### Scope drift guardrails

Any feature request is evaluated against: *Does this help an ecosystem team onboard new users—including developers—and verify they are ready to use or integrate the product?* If no → defer or reject.

---

## 3. Revised architecture and data model

### 3.1 Tenancy hierarchy (not namespaces)

Tenancy is a **chain of ownership and publish boundaries**, not a slug prefix.

```
Organization (legal/partner entity)
  └── Program (user-facing onboarding curriculum)
        └── Curriculum (ordered collection of tracks + publish unit)
              └── CurriculumVersion (immutable snapshot once published)
                    └── Track (module group within a version)
                          └── LessonVersion (immutable lesson snapshot)
                                └── QuizVersion (immutable quiz snapshot, optional)
```

**Organization** — Partner entity. Has members, trust flags, billing placeholder (unused v1).  
**Program** — What users see in the hub ("Arcium Fundamentals", "App X Onboarding"). Belongs to exactly one org. Has hub metadata (logo, tagline, status).  
**Curriculum** — Logical container for versioning. A program has one *active published* curriculum version at a time; drafts target the *working* draft curriculum.  
**CurriculumVersion** — Immutable after publish. Contains ordered tracks and lesson version references. Progress and analytics reference **version IDs**, not mutable lesson rows.  
**Track** — Equivalent to today's `category_id` grouping within a program (e.g. "Architecture"). Scoped to a curriculum version.  
**LessonVersion** — Immutable block tree + metadata at publish time.  
**QuizVersion** — Immutable question set + scoring config snapshot; may be null if lesson has no quiz.

**Enrollment** ties `user_id + program_id + curriculum_version_id` (version pinned at enroll or first activity).  
**Progress** ties `user_id + lesson_version_id` (never a mutable lesson row).

### 3.2 Core tables (implementation-ready sketch)

New tables (names illustrative; use snake_case in migrations):

| Table | Purpose |
|-------|---------|
| `organization` | Partner tenant root |
| `organization_member` | `user_id`, `organization_id`, `role` |
| `program` | Hub listing, `organization_id`, status, featured_rank |
| `curriculum` | Working draft container per program |
| `curriculum_version` | Published snapshot; `status`, `published_at`, `published_by` |
| `track` | `curriculum_version_id`, `position`, `slug`, `title` |
| `lesson_version` | Block JSON, quiz_version_id FK, slug, track_id, position |
| `quiz_version` | Questions JSON, scoring config JSON |
| `program_enrollment` | user + program + pinned curriculum_version_id |
| `lesson_progress` | user + lesson_version_id + completion state |
| `quiz_attempt` | user + quiz_version_id + answers + score (replaces slug-based submissions) |
| `platform_event` | Append-only event log (see §8) |
| `program_hub_settings` | Curation: featured, tags, sunset date |

**Migration from today:** Create org + program `arcium`; wrap existing `module_lesson` rows into `lesson_version` v1 under `curriculum_version` v1; backfill progress/submissions with version FKs; keep `(program_id, lesson_slug)` redirect map for URLs.

### 3.3 Content versioning strategy

**States:**

| Entity | States |
|--------|--------|
| Curriculum (draft) | `draft` → `in_review` → `published` (creates new curriculum_version) |
| CurriculumVersion | `published` (immutable), `superseded`, `archived` |
| Lesson edits | Always edit **draft** curriculum; never UPDATE published lesson_version rows |

**Publish flow:**

1. Author edits draft curriculum (lessons, tracks, quiz config).
2. Submit for review (if partner untrusted or policy requires).
3. Staff approves → **publish** creates new `curriculum_version` with copied immutable lesson/quiz versions.
4. `program.active_published_version_id` updates.
5. Existing enrollments **stay pinned** to their enrollment version unless staff runs **migration policy** (below).

**Progress integrity when content changes:**

| Scenario | Behavior |
|----------|----------|
| Typo fix, no quiz change | New version optional; if published, old enrollments unchanged. |
| Quiz scoring/threshold change | New curriculum_version; old attempts remain tied to old quiz_version. |
| Lesson removed | Old progress row preserved; path recalc only for new enrollments. |
| Lesson added mid-program | New enrollments see it; existing enrollments get "optional new content" banner (v1: no forced retroactive unlock). |
| Force upgrade | Staff action: `migrate_enrollment_to_version` with explicit audit event—v1 manual only. |

**Rule:** No in-place mutation of published JSON. Ever.

### 3.4 Structured lesson blocks

Replace free-form `body_sections` JSON over time with a **versioned block schema** (Zod-validated at author time and publish time).

**v1 block types:**

| Block type | Fields | Render |
|------------|--------|--------|
| `heading` | level (2–3), text | Semantic `<h2>`/`<h3>` |
| `paragraph` | markdown/plain text (restricted) | Sanitized MD → HTML |
| `callout` | variant (note/warning), text | Styled aside |
| `code` | language, snippet | Syntax-highlighted `<pre>` |
| `image` | cloudinary_url, alt, caption | `<figure>`; URL allowlist |
| `divider` | — | `<hr>` |

**Authoring:** Block list editor in Staff Studio (existing form evolves); partners use same editor when trusted.  
**Validation:** `lib/content-blocks/schema.ts` — single schema for staff + partner.  
**Rendering:** `LessonBlockRenderer` — one component map; no raw HTML from authors.  
**i18n-ready:** Block text stored as `{ "en": "..." }` objects in v1 with only `en` required—schema accepts locale map from day one.  
**Extension:** New block types require platform release (not partner plugins in v1).

Quiz questions remain a **separate structured schema** (existing types), snapshotted into `quiz_version`.

### 3.5 Tenant isolation and security enforcement

**Defense in depth—three layers, all required:**

#### Layer 1: Request tenant context (application)

```typescript
// lib/tenant/context.ts — mandatory entry for all server reads/writes
type TenantContext =
  | { kind: "anonymous"; programId?: string }
  | { kind: "learner"; userId: string }
  | { kind: "partner"; userId: string; organizationId: string; membershipRole }
  | { kind: "staff"; userId: string }
  | { kind: "system"; jobId: string };

// lib/tenant/scope.ts
requireProgramAccess(ctx, programId, "read" | "write" | "publish");
requireOrganizationAccess(ctx, orgId, "read" | "write");
```

All API routes and server actions call `resolveTenantContext()` then `require*`. **No direct `query()` from route handlers.**

#### Layer 2: Scoped repositories

```typescript
// lib/tenant/repositories/program-lessons.ts
getLessonVersion(ctx, { programId, lessonSlug, versionId? })
// Internally: JOIN program → curriculum_version → lesson_version
// Throws ForbiddenError if ctx lacks access
```

Repositories **always** filter by org/program derived from ctx—not from client-supplied IDs alone. Client IDs are validated against ctx scope.

#### Layer 3: Postgres Row Level Security (RLS)

Enable RLS on tenant tables (`program`, `lesson_version`, `quiz_attempt`, etc.). Session variable set per request:

```sql
SET app.current_user_id = '...';
SET app.current_org_ids = '{uuid1,uuid2}';  -- from membership
SET app.is_staff = 'true' | 'false';
```

Policies:

- **Learners:** read published program content; read/write own progress rows for enrolled programs.
- **Partner members:** read/write draft curriculum for their org only.
- **Staff:** bypass via `app.is_staff = true` policy branch.

**App uses pooled Neon connection:** set session vars at start of each transaction/request handler (`withTenantTransaction(ctx, fn)`).

**Leakage prevention checklist:**

- Integration tests: user A cannot read program B progress (expect 404/403).
- Slugs unique per `(program_id, slug)` not globally.
- Audit log on staff cross-org access.
- Published content readable without org membership; draft requires org/staff.

### 3.6 Analytics boundaries

| Data | Scope | Versioning |
|------|-------|------------|
| Learner progress | user + lesson_version_id | Tied to quiz_version for attempts |
| Program aggregates | program_id + curriculum_version_id | Rollups keyed by version |
| Org dashboard | organization_id | Cross-program within org only |
| Platform dashboard | staff only | All orgs |

Cross-program learner analytics on hub: **aggregate counts only** (programs started)—not shared leaderboards in v1.

---

## 4. Revised roles, permissions, and trust model

### 4.1 Role matrix

| Capability | Learner | Partner author | Partner admin | Staff | Platform admin |
|------------|---------|----------------|---------------|-------|------------------|
| View published hub programs | ✓ | ✓ | ✓ | ✓ | ✓ |
| Enroll / progress | ✓ | ✓ | ✓ | ✓ | ✓ |
| Edit draft curriculum (own org) | — | if trusted | ✓ | ✓ | ✓ |
| Submit for review | — | if trusted | ✓ | ✓ | ✓ |
| Publish to production | — | — | — | ✓ | ✓ |
| Publish to hub (curated) | — | — | — | ✓ | ✓ |
| Set org trust flags | — | — | — | — | ✓ |
| Rollback active version | — | — | — | ✓ | ✓ |
| View org analytics | — | read | ✓ | ✓ | ✓ |

**Untrusted partner (default):** No Studio access. Delivers content via intake template; staff authors in Staff Studio scoped to org.

**Trusted partner (`organization.trust_level = 'self_serve_draft'`):** Partner authors edit drafts, submit for review. **Publish remains staff-only in v1.**

**v1.1 (defer):** `self_serve_publish` flag for named partners after 2 successful staff-reviewed publishes.

### 4.2 Trust gate (manual, as decided)

Staff sets `organization.trust_level` after first staff-co-built program ships and passes content QA checklist:

- [ ] All quiz questions validated  
- [ ] Accessibility spot-check (headings, alt text)  
- [ ] No broken images / links  
- [ ] Quiz scoring thresholds documented and appropriate  
- [ ] Partner member accounts provisioned  

### 4.3 Content governance state machine

**Draft curriculum lifecycle:**

```
draft → in_review → approved → published (new curriculum_version)
         ↓              ↓
      changes_requested  rejected → draft
```

**Program hub lifecycle:**

```
internal (hidden) → listed (hub visible) → featured → sunset → archived
```

- **listed:** Appears in `/programs`; partner link works.  
- **featured:** Ranked on hub home; max **6** featured slots (operational rule).  
- **sunset:** No new enrollments; existing learners finish.  
- **archived:** 404 for new visitors; progress export only.

**Rollback:** Staff sets `active_published_version_id` to prior version (event logged). Does not delete newer version rows.

**Moderation (v1 minimal):** Staff can unpublish program to `internal` if content violates standards; `moderation_note` stored on org.

---

## 5. Revised authoring and governance model

### 5.1 Authoring surfaces

| Surface | User | Scope |
|---------|------|-------|
| **Staff Studio** | Staff | Any org/program; full publish |
| **Partner Studio** | Trusted partner authors | Own org drafts only; submit for review |
| **Intake form** | Untrusted partner | Brief, outline, assets—not live editor |

Staff Studio is the **evolution of `/staff/modules`**, not a parallel system.

### 5.2 Partner intake (untrusted)

1. Partner submits: product summary, audience, learning goals, outline (5–15 lessons), asset links.  
2. Staff creates org + program (`internal`).  
3. Staff builds draft curriculum in Staff Studio.  
4. Partner reviews staging preview URL.  
5. Staff publishes curriculum_version + sets hub to `listed`.  
6. Staff evaluates trust checklist → may set `self_serve_draft`.

### 5.3 Validation pipeline (all authors)

1. **Schema validation** (Zod blocks + quiz schema).  
2. **Media allowlist** (Cloudinary HTTPS).  
3. **Quiz security** (no answer keys in client payloads—existing pattern).  
4. **Review diff** (v1: staff reads changelog of lesson titles + quiz count; v1.1: automated diff summary).  
5. **Publish** snapshots immutable versions.

### 5.4 Centralized staff controls (always)

- Hub curation status and featured rank  
- Final publish authority (v1)  
- Trust level  
- Version rollback  
- Enrollment migration (manual)  
- Moderation / sunset  

---

## 6. Revised user onboarding experience and discovery model

### 6.1 Routes

| Route | Purpose |
|-------|---------|
| `/` | Hub: featured programs (max 6), continue where you left off, catalog link |
| `/programs` | Full catalog with filters (status=listed only) |
| `/programs/[programSlug]` | Program home: path, enroll CTA |
| `/programs/[programSlug]/lessons/[lessonSlug]` | Lesson player |
| `/account` | Global learner: enrollments, per-program progress |

**Legacy:** `/modules/*` → 301 to `/programs/arcium/*`.

### 6.2 Global account model

- One Arcademy user across all programs.  
- **Enrollment** created on explicit enroll or first authenticated lesson/quiz action (configurable per program; default: first action).  
- Progress rows keyed by `lesson_version_id` under enrollment's pinned curriculum version.

### 6.3 Guest-first flow (precise)

**Per-program setting:** `guest_first_lesson_enabled` (default: true for new programs; Arcium keeps current behavior).

| Step | Behavior |
|------|----------|
| Anonymous visitor opens first lesson | Full read access; quiz scored via guest API for **first lesson only** |
| Guest storage key | `arcademy.guest.v2:{programId}:{lessonVersionId}` in localStorage |
| Guest payload | `{ quizAttempt, readAt, schemaVersion }` — no answer keys stored |
| Sign up / sign in | `mergeGuestProgress(userId, guestPayload)` server action |
| Merge rules | If user has no progress → import guest attempt. If user has progress → keep better score; never downgrade mastery. Emit `guest_progress_merged` event. |
| Other lessons | Require auth; show sign-in gate (existing pattern) |

**Not v1:** Guest enrollments across multiple programs without account.

### 6.4 Hub curation rules (operational)

Programs appear on hub only when **all** are true:

- `hub_status = listed | featured`  
- `active_published_version_id` IS NOT NULL  
- ≥1 published lesson  
- Staff checklist completed (metadata, logo, summary, support contact)  
- Org not `moderated_suspended`

**Featured selection:** Staff ordered rank 1–6; algorithm does not auto-feature.  
**Discovery sort:** featured rank → recently updated → alphabetical.  
**Search (v1):** Title + summary full-text only; no user-generated tags beyond staff-assigned categories.

---

## 7. Onboarding integrity and user value (v1)

### 7.1 Design principle

Arcademy optimizes for **credible onboarding outcomes** in technical ecosystems—not engagement mechanics that feel superficial. Partners and users should trust that progress means comprehension and readiness to use or integrate, not vanity metrics.

**v1 emphasis:**

| Theme | What it means |
| --- | --- |
| **Progress continuity** | One global account; enrollments pinned to `curriculum_version`; guest merge preserves first-lesson work. |
| **Verified completion** | Quiz pass/mastery tied to immutable `lesson_version` / `quiz_version` rows. |
| **Credentialing foundation** | Server-side completion and mastery states; surfaced in `/account` and partner analytics—not yet portable credentials. |
| **Ecosystem readiness** | Finish a program knowing you met the bar to integrate or ask smarter support questions. |

Separate **core learning mechanics** (integrity contract) from **proof-of-learning** (post-v1: attestations and portable records—not tokens).

### 7.2 Core learning mechanics (v1 — always on)

Not marketed as "gamification"; these are the learning contract:

| Mechanic | Configurable by partner | Bounds |
|----------|-------------------------|--------|
| Pass threshold | yes | 60–80% (platform default 70%) |
| Mastery threshold | yes | 80–95% (default 90%) |
| Retry cooldown after fail | yes | 0–240 min (default 120) |
| Linear lesson unlock | yes | on/off (default on) |
| Best score retained | no | always on |
| Question types enabled | yes | subset of platform types |

Stored in `quiz_version.scoring_config` snapshot at publish.

### 7.3 Progress surfaces (v1)

| Surface | v1 status | Notes |
|---------|-----------|-------|
| **Enrollment + version-pinned progress** | ship | Core continuity model |
| **Quiz pass / mastery states** | ship | Verified comprehension per lesson |
| **Program completion record** | ship | `learner.program_completed` event; `/account` summary |
| **Progress map UI** | ship | Visual only; uses core progress data |

### 7.4 Explicitly out of scope (v1 and product narrative)

| Feature | Status | Why |
|---------|--------|-----|
| Daily streaks | not v1 | Superficial in technical ecosystems; timezone/ops burden |
| Completion / mastery badges (cosmetic) | not v1 | De-emphasized; completion **record** is the signal |
| Leaderboards | defer | Abuse, privacy, not our thesis |
| XP / levels / tier labels | never | Duolingo drift; conflicts with infrastructure identity |
| Token / NFT rewards | never | Not our model; see proof-of-learning instead |
| Seasonal resets | never | Conflicts with progress integrity |
| Push/email notifications | v1.2+ | Deliverability, consent, ops |
| Cross-program streaks or badges | never | Dilutes program focus |

### 7.5 Proof-of-learning (Phase 4+ direction)

Arcademy aligns with **infrastructure identity**. After v1, evolve toward verifiable outcomes—not token incentives:

| Direction | Purpose |
| --- | --- |
| **Signed completion attestations** | Cryptographically signed proof a learner completed a specific program + curriculum version—verifiable by partners and tooling. |
| **Ecosystem credentials** | Named credentials (e.g. "Arcium Fundamentals complete") for gating docs, support tiers, or beta access. |
| **Portable progress records** | Learner-owned export of enrollment/completion history across Arcademy programs. |

**Not in scope:** points-for-completion, token-gated rewards, NFT badges as primary model. Phase 0–1 `platform_event` + version FKs lay groundwork; attestations ship when legal/product gates clear.

### 7.6 Abuse handling (v1 minimal)

- Quiz submissions rate-limited per user/IP (cooldown + API rate limit).  
- No public leaderboards or streak incentives → reduced scoreboard gaming surface in v1.

---

## 7. Revised event and analytics model

*(Section renumbered in output—user asked for section 8 as events; keeping content under §8 below.)*

---

## 8. Event and analytics model

### 8.1 Internal event architecture

**Table: `platform_event`** (append-only)

| Column | Type | Notes |
|--------|------|-------|
| id | bigint | PK |
| occurred_at | timestamptz | |
| event_type | text | enum (see below) |
| actor_user_id | text | nullable for system |
| organization_id | uuid | nullable |
| program_id | uuid | nullable |
| curriculum_version_id | uuid | nullable |
| lesson_version_id | uuid | nullable |
| payload | jsonb | event-specific, no PII beyond ids |

**v1 event types:**

```
learner.enrolled
learner.lesson_started
learner.lesson_completed
learner.quiz_submitted
learner.quiz_passed
learner.quiz_mastered
learner.program_completed
learner.guest_progress_merged

content.curriculum_submitted_for_review
content.curriculum_published
content.curriculum_rolled_back

program.hub_listed
program.hub_sunset

org.trust_level_changed
```

**Emission:** Synchronous insert in same transaction as state change (progress write + event).  
**Outbox (v1.1):** `platform_event_outbox` for webhook delivery retries—defer until webhooks ship.

**Future webhooks:** Subscribe by `organization_id` + event_type; HMAC-signed POST. Not v1.

### 8.2 Analytics (lightweight v1)

**Goals:** Program health for staff/partners; drop-off visibility; no custom BI.

**Materialized views / nightly job (pick one for v1):**

| Metric | Grain | Use |
|--------|-------|-----|
| `program_enrollment_count` | program + day | Hub health |
| `lesson_funnel` | program + curriculum_version + lesson_version | Drop-off |
| `quiz_pass_rate` | quiz_version | Content quality |
| `median_time_to_complete` | program + version | Partner success |
| `active_learners_7d` | program | Engagement |

**Dashboards:**

- **Staff:** all programs, version-aware funnels.  
- **Partner admin:** own programs only (read via RLS).  

**Export:** CSV by program + version range (staff/partner admin).  

**Not v1:** Cohort comparisons, A/B content tests, real-time streaming analytics.

### 8.3 Observability

- Structured logs on tenant context resolution failures (potential leakage attempts).  
- Alert on RLS policy violation errors (Postgres error code).  
- Event emission failure rolls back transaction (progress + event atomic).

---

## 9. Revised phased roadmap

### Phase 0 — Tenancy foundation (3–4 weeks)

**Goal:** Data model + isolation + Arcium migration; no hub UI yet.

- [ ] Migrations: organization, program, curriculum/version, lesson_version, quiz_version  
- [ ] RLS policies + `withTenantTransaction`  
- [ ] Scoped repositories; refactor `lib/content.ts` → program-aware reads  
- [ ] Migrate Arcium content to program `arcium` v1  
- [ ] Redirect `/modules/*` → `/programs/arcium/*`  
- [ ] Progress backfill with version FKs  
- [ ] Block schema v1 (validate existing content; migrate body_sections)  

**Exit criteria:** Arcium lessons work under new model; cross-tenant integration tests pass.

**Depends on:** Nothing. **Blocks everything.**

### Phase 1 — Hub + onboarding loop (3–4 weeks)

**Goal:** Public catalog and program-scoped onboarding UX.

- [ ] Hub home + `/programs` with curation rules  
- [ ] Program home + lesson player (program-scoped)  
- [ ] Global account enrollments  
- [ ] Guest-first merge flow (program-scoped keys)  
- [ ] `platform_event` emission for core learner events  
- [ ] Basic staff program admin (status, featured rank)  

**Exit criteria:** Arcium discoverable as hub program; new learner can complete lesson 1 as guest and merge on signup.

### Phase 2 — Partner onboarding (2–3 weeks)

**Goal:** Second program built by staff; operational pipeline.

- [ ] Organization + intake workflow (form → staff queue)  
- [ ] Staff Studio scoped by org/program  
- [ ] Draft/review/publish state machine  
- [ ] Version publish + rollback UI (staff)  
- [ ] Partner preview links (signed token, read-only)  
- [ ] Nightly analytics job + staff funnel dashboard  

**Exit criteria:** One non-Arcium pilot program live on hub, staff-authored.

### Phase 3 — Trusted partner authoring (3–4 weeks)

**Goal:** Self-serve drafts for trusted orgs.

- [ ] Partner Studio (draft only)  
- [ ] Submit for review flow + staff approval queue  
- [ ] Trust level admin UI  
- [ ] Partner admin analytics read-only  

**Exit criteria:** Trusted pilot partner submits draft; staff publishes without re-authoring from scratch.

### Phase 4 — Deferred backlog (post-v1)

- **Proof-of-learning:** signed completion attestations, ecosystem credentials, portable progress records (not tokens)  
- Webhooks + outbox  
- Partner self-publish flag  
- Partner SSO  
- Embed SDK  
- Block type expansion  
- Automated enrollment migration tooling  

**Total to v1:** ~11–15 weeks engineering + 1–2 weeks buffer for migration QA and partner pilot content.

---

## 10. Keep / change / defer summary

### Keep

- Single Arcademy hub and global learner account  
- Hybrid authoring (staff default + manual trust gate)  
- Arcium as Program #1  
- Reading-first lesson UX and editorial design language  
- Existing quiz question types and scoring engine (adapt to quiz_version)  
- Quiz answer-key sanitization pattern  
- Staff publish authority in v1  
- Cloudinary image allowlist  

### Change (from original plan)

| Original | Revised |
|----------|---------|
| Programs as namespaces | Full tenancy hierarchy with versioned immutability |
| App-layer scoping only | RLS + scoped repositories + tenant context |
| Free-form body_sections | Structured block schema with render pipeline |
| light/medium/heavy gamification | Core mechanics + progress surfaces; streaks/badges/leaderboards removed from v1 |
| Leaderboards / streaks / XP | Out of scope; proof-of-learning in Phase 4+ instead |
| 6–8 week total estimate | 12–16 weeks to v1 |
| Slug-global lessons | `(program_id, slug)` + lesson_version FK on progress |
| Keystatic for lessons | Deprecate for lesson content; hub metadata in Postgres |

### Defer

- Streaks, cosmetic badges, XP, tiers, seasonal resets, public leaderboards  
- Token / NFT rewards (proof-of-learning is the future direction)  
- Push/email notifications  
- Webhooks (design events now; delivery later)  
- Partner self-publish without staff  
- White-label / embed SDK  
- SCORM, custom question types, arbitrary HTML  
- Full i18n UI (schema-ready only)  
- Automated curriculum migration for existing enrollments  

---

## 11. Open questions and risks

### Open questions

| # | Question | Owner | Needed by |
|---|----------|-------|-----------|
| 1 | First non-Arcium pilot partner—which app? | Product | Phase 2 start |
| 2 | Enroll explicitly vs on first activity? | Product | Phase 1 |
| 3 | Partner preview tokens: TTL and auth? | Eng | Phase 2 |
| 4 | Neon RLS + pooled connections: use `@neondatabase/serverless` transaction hooks or unpooled for writes? | Eng | Phase 0 |
| 5 | Existing Arcium learners: pin to curriculum v1 automatically on deploy? | Eng | Phase 0 migration |
| 6 | Legal: partner terms for user progress data sharing? | Product/Legal | Phase 2 |

### Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| RLS + pooler complexity | Medium | High | Spike in week 1; fallback to unpooled for scoped writes |
| Migration breaks progress | Medium | High | Dry-run on Neon branch; reversible migration scripts |
| Scope creep ("just one LMS/education feature") | High | High | Thesis + keep/defer list in PR template |
| Partner expects full design control | Medium | Medium | Intake + block constraints documented in partner agreement |
| Single shared Neon DB (all envs) | Existing | High | Treat migrations as prod; add staging branch later |

### Assumptions (explicit)

- v1 runs on existing stack (Next.js 16, Neon Postgres, better-auth, Vercel).  
- One Neon database remains shared across envs until infra investment.  
- Partner count in year one is **<10** programs—not thousands of tenants.  
- English-only UI for v1; block schema supports future locales.  
- Staff capacity exists to review all publishes in v1.

---

## Appendix A — Partner collaboration stack (unchanged recommendation)

For introducing this plan to your partner:

| Tool | Role |
|------|------|
| **Notion** | Partner-readable strategy, decisions, open questions |
| **GitHub** | Canonical spec (this doc) + issues + Projects board |
| **Figma** | Hub/program wireframes when ready |

Do not duplicate published lesson content in Notion.

---

## Appendix B — Suggested Notion page outline for partner

1. Executive summary (§1)  
2. What Arcademy is / is not (§2)  
3. Locked decisions table  
4. Hub + program UX (wireframes TBD)  
5. Partner journey (intake → staff build → trust → studio)  
6. Roadmap with dates (§9)  
7. Open questions (§11)  

---

*End of spec. Review comments → update this file → then proceed to implementation plan (writing-plans).*
