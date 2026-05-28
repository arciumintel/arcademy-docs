# Repository boundaries

Single canonical **arcintel** repo. Backend/data layer (Phase 0) and hub UI (Phase 1+, partner PRs) live here.

## Path ownership

| Path | Owner | Notes |
| --- | --- | --- |
| `db/`, `scripts/` | Backend | Migrations are production-impactful; review required |
| `lib/db.ts`, `lib/tenant/*` | Backend | All tenant-scoped data access |
| `lib/content-blocks/`, `lib/quiz/`, `lib/contracts/` | Backend | Partner imports types/schemas only |
| `app/api/auth/`, `app/api/health/`, `app/api/v1/` | Backend | JSON API + auth handlers |
| `app/(hub)/`, `components/` | Partner (Phase 1+) | Hub, lesson player, account UI — follow [`FRONTEND-GUIDELINES.md`](FRONTEND-GUIDELINES.md) |
| `tests/integration/` | Backend | Cross-tenant isolation tests |

## Data access rules

1. **No raw `query()` in route handlers or `app/(hub)/` pages.** Use `resolveTenantContext()` → repositories.
2. **Prefer Server Components** that import `lib/tenant/repositories/*` and `lib/contracts/v1/*`.
3. **`app/api/v1/*`** is optional — for client components, smoke tests, and tooling. Not the primary UI integration path.
4. **Quiz answer keys** never leave the server (`lib/quiz/public.ts`).

## Example (Phase 1 partner page)

```tsx
import { resolveTenantContext } from "@/lib/tenant/context";
import { listListedPrograms } from "@/lib/tenant/repositories/programs";

export default async function ProgramsPage() {
  const ctx = await resolveTenantContext();
  const programs = await listListedPrograms(ctx);
  return <ul>{programs.map((p) => <li key={p.slug}>{p.title}</li>)}</ul>;
}
```

## PR expectations

- Partner PRs should touch `app/(hub)/` and `components/` only unless coordinated.
- Schema changes require backend review and idempotent migrations.
- Do not commit `.env` or secrets.
