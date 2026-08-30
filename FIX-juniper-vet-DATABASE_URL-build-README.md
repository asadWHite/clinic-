# Fix: juniper-vet — `DATABASE_URL` breaks `next build`

## Problem

`src/db/index.ts` threw `Error: DATABASE_URL is required` at **module import time** when
`DATABASE_URL` was not set. Server components import `@/db` statically (e.g.
`src/components/home/Ending.tsx`, `src/components/home/Middle.tsx`), so Next.js evaluates
the module during build and `next build` fails while collecting page data:

```
Error: Failed to collect configuration for /_not-found
  [cause]: Error: DATABASE_URL is required
```

Reproduced without `DATABASE_URL` in the environment (as in CI).

## Fix

Make the pool construction lazy. `pg.Pool` does not open a connection and does not throw
until a query actually runs, so the pool is created without a connection string when the
env var is absent. Builds succeed everywhere; runtime behavior is unchanged — the
connection string is resolved when the first query runs (and PostgreSQL then reports an
auth/connection error as usual if `DATABASE_URL` is truly missing).

```ts
const databaseUrl = process.env.DATABASE_URL;

export const pool =
  globalForDb.__arenaNextJsPostgresqlPool ??
  new Pool(databaseUrl ? { connectionString: databaseUrl } : {});
```

## Verification (on `asadWHite/juniper-vet` @ `3309c0c`)

- `env -u DATABASE_URL npm run build` → ✅ build succeeds (was failing before)
- `DATABASE_URL=... npm run build` → ✅ build succeeds
- `npm run typecheck` → ✅ passes
- `npm run lint` → 0 errors (42 pre-existing unrelated `@next/next/no-img-element` warnings)

## Apply to your repo

```bash
git -C /path/to/juniper-vet apply juniper-vet-fix-0001-Fix-DATABASE_URL-build-crash.patch
```

The patch was validated with `git apply --check` against a pristine checkout of
`juniper-vet@3309c0c`.
