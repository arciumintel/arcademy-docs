# Frontend guidelines (partner agents)

Instructions for agents building the **public learner experience** in this repo: hub, program pages, lesson player, and account UI. Backend owns data, auth, APIs, and schemas; you own presentation and interaction within the constraints below.

**Read first:** [PLATFORM-OVERVIEW.md](PLATFORM-OVERVIEW.md) (product), [REPO-BOUNDARIES.md](REPO-BOUNDARIES.md) (what you may touch), [AGENT-PLATFORM.md](AGENT-PLATFORM.md) (engineering rules). Do not contradict the [design spec](superpowers/specs/2026-05-20-ecosystem-platform-design.md) unless the user explicitly changes direction.

---

## What you are building

Arcidex is **ecosystem onboarding infrastructure**—guided programs with comprehension checks and measurable progress for the Arcium ecosystem. It is **not** an education platform, generic LMS, or partner marketing site builder.

Your UI should feel like a **focused technical onboarding reader**: calm, credible, reading-first. Users are developers and ecosystem newcomers preparing to use or integrate a product—not students chasing streaks or badges.

| Build | Do not build (v1) |
| --- | --- |
| Hub catalog and program discovery | Partner-controlled page layouts or arbitrary HTML lessons |
| Lesson reader from structured blocks | Custom quiz question types or answer-key exposure |
| Quiz UI using platform question types | Leaderboards, XP, streaks, cosmetic badge systems |
| Account progress summary | Token rewards, gamified celebration overlays |
| Guest-first lesson 1 + sign-up merge | Full white-label sites per partner |

**Scope test:** Does this help someone onboard and verify they understood the product? If not, defer.

---

## Repository scope

| You own | You do not own (coordinate with backend) |
| --- | --- |
| `app/(hub)/` | `db/`, `scripts/`, migrations |
| `components/` | `lib/tenant/*`, `lib/db.ts` |
| Styling config (`tailwind.config.ts`, global CSS) when added | `lib/content-blocks/schema.ts`, `lib/quiz/schema.ts` |
| Client components that call server actions or optional `app/api/v1/*` | Raw SQL in pages or route handlers |

**PR rule:** Touch `app/(hub)/` and `components/` only unless explicitly coordinated. Import types from `@/lib/contracts/v1/*`, `@/lib/content-blocks/schema`, and `@/lib/quiz/public`—do not fork schemas.

---

## Stack and conventions

| Area | Choice |
| --- | --- |
| Framework | Next.js **16** App Router, **React 19** |
| Language | TypeScript **strict** |
| Styling | **Tailwind CSS 3**, `darkMode: "class"`, shared tokens in `theme.extend` |
| Data in pages | **Server Components** → `resolveTenantContext()` → `lib/tenant/repositories/*` |
| Imports | `@/*` → repo root |

Prefer Server Components for catalog and lesson pages. Use client components only for interactivity (quiz submit, guest localStorage, theme toggle, continue CTAs that need client state).

**Never** call `query()` directly from `app/(hub)/` or components used only there.

---

## Routes (Phase 1+)

Implement under `app/(hub)/`:

| Route | Purpose |
| --- | --- |
| `/` | Hub home: up to **6** featured programs (staff rank), “continue where you left off”, link to catalog |
| `/programs` | Full catalog (`hub_status` listed/featured only; sort: featured rank → title) |
| `/programs/[programSlug]` | Program overview: path summary, enroll / continue CTA |
| `/programs/[programSlug]/lessons/[lessonSlug]` | Lesson player + optional quiz |
| `/account` | Signed-in learner: enrollments and per-program progress |

Staff (`/staff/*`) and partner authoring (`/partner/*`) are **out of scope** for partner frontend agents in early phases.

There is no `/modules` route. Do not add parallel URL schemes.

---

## Data access pattern

```tsx
import { resolveTenantContext } from "@/lib/tenant/context";
import { listListedPrograms } from "@/lib/tenant/repositories/programs";

export default async function ProgramsPage() {
  const ctx = await resolveTenantContext();
  const programs = await listListedPrograms(ctx);
  // render
}
```

- Use repository return types (`ProgramCatalogRow`, `PublishedLessonRow`, `lib/contracts/v1/*`) for props—do not reshape DB rows ad hoc in every page.
- **`app/api/v1/*`** is optional (client tools, smoke tests)—not the primary UI path.
- Quiz payloads must use **`toPublicQuiz` / `PublicQuiz`**—never import full `QuizQuestion` with `correctAnswer` on the client.

---

## Visual design language

Figma wireframes may arrive later; until then, follow these principles so all pages feel like one product.

### Tone

- **Professional and restrained**—infrastructure, not consumer ed-tech.
- **Reading-first**—generous line length (~65–75ch for lesson body), comfortable line-height, clear heading hierarchy.
- **Trust over delight**—progress and quiz results are factual (“Passed · 80%”), not celebratory animations or mascot copy.

### Layout

- **Hub / catalog:** Card or list rows with program title, tagline, optional logo; featured section clearly labeled; empty states explain “no programs listed yet” without blaming the user.
- **Lesson player:** Single main column for content; quiz below the lesson or in a clearly separated section; persistent program context (breadcrumb: Program → Lesson).
- **Account:** Table or list of enrollments with program name, version-pinned progress, completion state—no cross-program leaderboard.

### Typography and color

- Establish **design tokens** in `tailwind.config.ts` (`theme.extend`): neutrals for surfaces, one accent for links/primary CTAs, semantic colors for callouts (`note`, `warning`).
- Support **light and dark** via `class` on `<html>`; avoid hard-coded colors outside tokens.
- Headings in lessons come from **blocks** (`heading` level 2–3)—page chrome uses separate, smaller nav typography so lesson content owns the hierarchy.

### Partner branding

- Program **title, tagline, and logo** come from published program metadata—display them on program home and lesson chrome.
- Partners do **not** control global hub chrome, fonts, or arbitrary CSS. The hub stays one Arcidex product; programs are differentiated within that frame.

### Components to centralize

| Component | Responsibility |
| --- | --- |
| `LessonBlockRenderer` | Single map from `ContentBlock` → React; sanitized output only |
| `QuizPlayer` (name flexible) | Renders `PublicQuiz`; submits to server action/API; never shows correct answers before scoring |
| `ProgramCard` | Hub/catalog tile |
| `ProgressMap` | Visual lesson path; uses enrollment progress data—no fake percentages |

Do not scatter block rendering logic across pages.

---

## Lesson content (blocks)

Authors (staff/partners) write **only** validated blocks—no raw HTML. Render through `LessonBlockRenderer` only.

| Block | Render guidance |
| --- | --- |
| `heading` | Semantic `<h2>` / `<h3>`; pick `level` from block |
| `paragraph` | Restricted markdown → **sanitized** HTML (use one shared sanitizer pipeline) |
| `callout` | `note`: subtle info panel; `warning`: stronger border/background—do not use for body text |
| `code` | `<pre><code>` with syntax highlighting; horizontal scroll on small screens; show `language` label when helpful |
| `image` | `<figure>`; require `alt` from locale map; optional `caption`; only allow Cloudinary URLs already validated server-side |
| `divider` | `<hr>` with low visual weight |

**Locale:** Text fields are `{ "en": "..." }`. v1 UI is **English-only**—read `en` for display; do not build a locale switcher until product says otherwise.

**Limits:** 1–200 blocks per lesson (schema). If content is huge, paginate or collapse in UI only with product approval—default is one scrollable lesson.

Reference schema: `lib/content-blocks/schema.ts`.

---

## Quizzes (UI + copy)

- Questions use platform types only: `short_text`, `multiple_choice`, `true_false` (`lib/quiz/schema.ts`).
- Show **pass threshold** and **attempts remaining** from `scoringConfig` when relevant; respect **cooldown** after fail (disable submit + show time remaining).
- After submit, show outcome from server (pass / fail / mastery)—**not** client-side grading.
- Never display `correctAnswer` or store answer keys in localStorage. Guest storage: `arcidex.guest.v2:{programId}:{lessonVersionId}` with `{ quizAttempt, readAt, schemaVersion }` only.

**Copy style for quizzes:** Direct, one idea per question; test comprehension of the lesson, not trivia. Prefer “What does X enable?” over trick wording.

---

## Guest-first flow (lesson 1)

When `guest_first_lesson_enabled` (default on; Arcium uses it):

1. Anonymous user can **read lesson 1** and **submit quiz** for that lesson only.
2. Other lessons show a **sign-in gate** (clear CTA, not a dead end).
3. On sign-up/sign-in, call server **merge** action; UI should explain that progress will be saved to their Arcidex account.

Do not implement guest progress across multiple programs without an account (not v1).

---

## Progress and enrollments (UI)

- Progress is tied to **`lesson_version_id`** under a pinned **`curriculum_version_id`**—show “in progress / completed / locked” per lesson, not vague percentages unless derived from real data.
- **Linear unlock** may be on (partner config)—honor locked state from server, do not infer unlock client-only.
- **Program completion** is a recorded state—surface it plainly on `/account` (“Completed · Arcium Fundamentals”).
- Do **not** add streak counters, XP bars, level names, or public rankings.

---

## Voice and content (microcopy)

Write for **developers and technical users** onboarding to an ecosystem product.

| Do | Avoid |
| --- | --- |
| “Continue lesson”, “Enroll”, “Sign in to save progress” | “Level up”, “Earn XP”, “You’re on fire!” |
| “Passed comprehension check” | “Awesome job!!!” |
| “Arcidex account” (one account, all programs) | “Classroom”, “course”, “instructor” |
| Name the **program** and **lesson** clearly | Generic “Module 3” without program context |

**Product name:** **Arcidex** (hub/platform). **First program:** **Arcium Fundamentals** (`program.slug = arcium`). Use partner program titles from data, not invented names.

**Hub listing copy** comes from `title` and `tagline` in the database—your job is to display them well, not rewrite them in code.

Example lesson tone (from seed content): clear, factual, one concept per paragraph; callouts for important constraints, not marketing fluff.

---

## Accessibility (v1 minimum)

- Logical heading order (lesson blocks + page `<h1>` once per view).
- Images: always render `alt` from block data; flag missing alt in dev-only warnings if needed.
- Interactive controls: visible focus, keyboard-operable quiz and nav.
- Color: callouts and pass/fail states must not rely on color alone (icon or text label).
- Respect `prefers-reduced-motion` for any transitions you add.

Staff review includes a basic accessibility spot-check—build so that checklist is easy to pass.

---

## Security and privacy (UI)

- Do not log quiz answers or PII to the browser console in production paths.
- Do not embed third-party trackers without explicit product approval.
- Cloudinary images: use URLs from API only; no arbitrary `img src` from user input in components.

---

## Phase gating

| Phase | Frontend work |
| --- | --- |
| **0** | No public hub UI; backend only |
| **1** | Hub, program home, lesson player, account, guest merge UX |
| **2+** | Staff/partner studios are separate surfaces—do not block Phase 1 on them |

Do not ship learner routes that bypass repositories or RLS-backed reads.

---

## Checklist before opening a PR

- [ ] Changes limited to `app/(hub)/` and `components/` (unless coordinated)
- [ ] Pages use `resolveTenantContext()` + repositories
- [ ] Lessons rendered only via `LessonBlockRenderer` with sanitization
- [ ] Quiz uses `PublicQuiz` only; grading is server-side
- [ ] No gamification, leaderboards, or custom question types
- [ ] English UI; block text read from `en`
- [ ] Light/dark tokens used consistently
- [ ] Guest lesson 1 and sign-in gate behavior match spec if touching those flows

---

## Related documents

| Document | Use |
| --- | --- |
| [PLATFORM-OVERVIEW.md](PLATFORM-OVERVIEW.md) | Product terms and partner journey |
| [AGENT-PLATFORM.md](AGENT-PLATFORM.md) | Locked decisions, phases, block/quiz tables |
| [REPO-BOUNDARIES.md](REPO-BOUNDARIES.md) | Path ownership |
| [app/(hub)/README.md](../app/(hub)/README.md) | Route list for this app group |
| [Design spec](superpowers/specs/2026-05-20-ecosystem-platform-design.md) | Full UX, events, hub curation rules |

When Figma or brand tokens land, add a short **Design assets** section here with links—do not duplicate wireframes in prose.
