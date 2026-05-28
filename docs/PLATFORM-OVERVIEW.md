# What is Arcademy?

Arcademy is **ecosystem onboarding infrastructure** for the Arcium ecosystem—not an education platform or generic LMS. It helps ecosystem teams turn complex protocol documentation into **guided onboarding programs**—with comprehension checks and measurable progress—so they can see whether users actually understood a product before they go deeper or integrate, not just clicked through a doc.

New users and developers get **one account** and a **curated catalog** of programs. Partners get governed onboarding infrastructure—without building their own onboarding stack from scratch—and clear outcomes from structured programs and progress data (see [Why partners use Arcademy](#why-partners-use-arcademy)).

This document explains the product in plain language. For engineering detail, see the [design spec](superpowers/specs/2026-05-20-ecosystem-platform-design.md). For day-to-day agent rules, see [AGENT-PLATFORM.md](AGENT-PLATFORM.md).

---

## The idea in one sentence

Arcademy is ecosystem onboarding infrastructure: guided programs, comprehension checks, and measurable progress for new user onboarding and developer adoption across the Arcium ecosystem.

---

## Why partners use Arcademy

Partners are ecosystem teams (protocols, apps, tooling) who need new users to **activate** and developers to **integrate**—not just read docs. Arcademy gives them shared onboarding infrastructure and a governed program model so they can:

| Outcome | What Arcademy enables |
| --- | --- |
| **Reduce onboarding friction** | Turn scattered protocol docs into a linear, guided path—one link, one program, clear next steps. |
| **Improve activation** | Comprehension checks confirm users understood key concepts before they go deeper or integrate. |
| **Standardize onboarding** | Structured blocks and platform-defined quizzes keep quality consistent across programs and partner teams. |
| **Identify drop-off points** | Lesson- and version-scoped progress shows where users stall—before it shows up only in support tickets. |
| **Prepare developers before support requests** | Quizzes and progress signal who completed fundamentals; partners spend support time on real blockers, not repeat basics. |

Partners do **not** build bespoke onboarding from scratch or run a separate LMS. Staff help new partners launch; trusted partners edit drafts under review (v1). Analytics and exports roll out in Phase 2+; the data model is designed for these outcomes from day one.

---

## Key terms

| Term | What it means |
| --- | --- |
| **Hub** | The public home page and catalog where users discover programs (`/` and `/programs`). |
| **Program** | A complete onboarding path for one product or protocol (e.g. “Arcium Fundamentals”). This is what users enroll in. |
| **Organization** | The partner company that owns one or more programs. Used for permissions and analytics boundaries. |
| **Lesson** | A reading-first page in a program. May include a short quiz at the end. |
| **Track** | A group of lessons within a program (like a chapter or module). |
| **Quiz** | A comprehension check tied to a lesson. Uses fixed question types defined by the platform—not a free-form quiz builder. |
| **Draft** | Work-in-progress content authors can still edit. |
| **Publish** | Staff action that freezes a snapshot of the draft and makes it the live version users see. Published content is not edited in place. |
| **Enrollment** | A user’s membership in a specific program. Progress is tracked per program. |
| **Staff** | Arcademy operators who curate the hub, onboard partners, review content, and publish. |
| **Trusted partner** | A partner who passed a manual quality gate and may edit **drafts** in Partner Studio. They still cannot go live without staff approval in v1. |

---

## Who uses Arcademy?

### Users & developers

**What they get:** Progress that persists, completion that means something, and a path toward ecosystem readiness—not engagement gimmicks.

| Focus | What Arcademy enables |
| --- | --- |
| **Progress continuity** | One account, enrollments pinned to curriculum versions, guest-to-account merge—pick up where you left off across sessions and programs. |
| **Verified completion** | Quizzes and mastery thresholds confirm comprehension; progress ties to immutable `lesson_version` / `quiz_version` rows, not mutable docs. |
| **Credentialing (v1 foundation)** | Program completion and quiz pass states are recorded server-side; v1 surfaces them in `/account` and partner analytics—not yet portable credentials. |
| **Ecosystem readiness** | Finish a program knowing you met the bar to use the product, integrate, build, or ask smarter support questions. |

**Day-to-day:**

- Browse the hub and open programs.
- Read lessons and take quizzes.
- Keep progress with one account across programs.
- Try the first lesson as a guest, then sign up to save progress (planned for v1).

### Partners (ecosystem apps)

**What they get:** See [Why partners use Arcademy](#why-partners-use-arcademy)—lower friction onboarding, better activation, standardized onboarding paths, drop-off visibility, and users prepared before deeper integration or support.

**How they work with Arcademy:**

- **New partners:** Submit an intake brief (goals, outline, assets). Staff builds the first version.
- **Trusted partners (later):** Edit drafts in Partner Studio and submit for staff review.
- Configure quizzes within platform limits (pass score, retries, etc.)—not custom question formats.

### Staff

- Create organizations and programs.
- Author and publish content (especially for new partners).
- Decide what appears on the hub and what gets featured.
- Approve partner submissions before they go live.
- Grant “trusted” status after a program ships successfully.

---

## How content gets created and published

1. **Draft** — Staff (or a trusted partner later) writes lessons using structured blocks: headings, paragraphs, callouts, code, images—not arbitrary web pages.
2. **Review** — For partner content, staff checks quality (quizzes make sense, images work, accessibility basics).
3. **Publish** — Staff publishes a frozen snapshot. Users see this version.
4. **List on hub** — Publishing alone is not enough. Staff also marks a program as **listed** or **featured** so it appears in the public catalog.

If staff publish an updated version, **existing enrollments stay on the version they started** unless staff manually migrate them (v1). That protects progress integrity when content changes.

---

## What users see (routes)

| Page | Purpose |
| --- | --- |
| `/` | Hub home — featured programs, continue where you left off |
| `/programs` | Full catalog |
| `/programs/[name]` | Program overview and start/continue button |
| `/programs/[name]/lessons/[lesson]` | Lesson reader + quiz |
| `/account` | Your enrollments and progress across programs |

Staff and partner authoring tools live under `/staff` and `/partner` (rolled out in later build phases).

---

## What’s in the first version (v1)

**Included:**

- Curated hub and program catalog
- Reading-first lessons with platform-defined quizzes
- One global Arcademy account
- Staff-built programs for new partners
- Arcium as the first program at launch
- **Progress continuity** across sessions (global account, version-pinned enrollments, guest merge)
- **Verified completion** via comprehension checks and immutable progress records
- Staff analytics on program performance and drop-off

**Not in v1 (on purpose):**

- Positioning or features of a generic **education platform** (classrooms, SCORM, course marketplaces)
- Full LMS features (instructor grading, arbitrary courses)
- Partner-built page design / marketing site builder
- Public leaderboards, streaks, XP, or cosmetic badge systems
- On-chain tokens or token-gated rewards (not our model)
- “Sign in with partner app” (partner SSO)
- Embeddable widget for partner apps
- Webhooks and billing

**Scope check we use:** Does this help an ecosystem team onboard new users—including developers—and verify they are ready to use or integrate the product—or credibly record that they are? If not, it waits.

---

## Proof-of-learning (future direction)

Arcademy aligns with **infrastructure identity**, not consumer gamification or token incentives. After v1, the platform is designed to evolve toward **proof-of-learning**:

| Direction | Purpose |
| --- | --- |
| **Signed completion attestations** | Cryptographically signed records that a learner completed a specific program version—verifiable by partners and tooling. |
| **Ecosystem credentials** | Named credentials (e.g. “Arcium Fundamentals complete”) partners and the hub can trust for gating docs, support tiers, or beta access. |
| **Portable progress records** | Learner-owned export of enrollment and completion history—useful across Arcademy programs without re-proving basics. |

This is **not** a token rewards program. No points-for-completion, no NFT badges as the primary model. Events and version FKs in Phase 0–1 lay groundwork; attestations and credentials ship when legal and product gates clear (Phase 4+).

---

## Partner journey (simplified)

```
New partner → intake form → staff builds program → staff publishes → hub listing
                                      ↓
                            (after quality gate)
                                      ↓
                     trusted partner edits drafts → staff review → staff publishes
```

Partners do **not** go live on their own in v1. Trust is earned manually after staff has shipped and reviewed at least one program with them.

---

## Build phases (plain language)

| Phase | What gets built | What “done” looks like |
| --- | --- | --- |
| **0 — Foundation** | Database structure, security isolation, first Arcium content seeded | Data model works; partners’ data can’t leak to each other |
| **1 — Hub & onboarding UX** | Hub, program pages, lesson player, accounts, guest try-then-sign-up | A new visitor can try Arcium lesson 1 and save progress by signing up |
| **2 — Partner onboarding** | Intake flow, staff authoring UI, publish/rollback tools, basic analytics | A second program built entirely by staff goes live on the hub |
| **3 — Trusted partners** | Partner draft editor, review queue, trust settings | A trusted partner submits a draft; staff publishes without rewriting from scratch |
| **4 — Later** | Proof-of-learning, webhooks, SSO, embeds | Signed attestations, ecosystem credentials, portable records; partner integrations |

Estimated calendar time to v1: **about 12–16 weeks** with 1–2 engineers.

---

## Decisions already made

These are settled unless leadership explicitly reopens them:

- Product category: **ecosystem onboarding infrastructure** (not education, not a generic LMS)
- Single public hub (not separate sites per partner)
- Arcium is program #1 at launch
- Staff must approve all publishes in v1
- Published snapshots are immutable
- Onboarding integrity over streaks/badges; proof-of-learning (attestations, credentials) is the post-v1 direction—not tokens
- English-only UI for v1 (content structure supports more languages later)
- Fewer than ~10 partner programs expected in year one

---

## Open questions

| Question | When we need an answer |
| --- | --- |
| Which partner app is the first non-Arcium pilot? | Before Phase 2 |
| Does enrollment happen on “Enroll” click or first lesson activity? | Before Phase 1 |
| How long should partner preview links last? | Before Phase 2 |
| Legal terms for sharing learner progress with partners | Before Phase 2 |

---

## Related documents

| Document | Audience |
| --- | --- |
| [Design spec](superpowers/specs/2026-05-20-ecosystem-platform-design.md) | Engineering, product, partners (detailed) |
| [AGENT-PLATFORM.md](AGENT-PLATFORM.md) | AI coding agents (operational rules) |

---

## Branding

- **Product name:** Arcademy
- **First program:** Arcium Fundamentals (`arcium`)
