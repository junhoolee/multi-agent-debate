# Decision: Should we use SQLite or PostgreSQL for a new SaaS product expecting fewer than 100 concurrent users?

## Recommendation

Start with PostgreSQL if your day-1 deployment involves multiple application instances, background workers on separate machines, or a managed platform that may relocate your app across hosts. Start with SQLite (WAL mode) if your day-1 deployment is a single persistent node with no network-separated database. Do not adopt "distributed SQLite" (Turso, Litestream replication) for v1 — it adds more operational surface area than either plain SQLite or managed PostgreSQL without proven need.

## Why

- ✅ SQLite's single-writer WAL constraint is a hard architectural limit, not a theoretical concern — verified via sqlite.org/wal.html. Any deployment topology with multiple writers makes SQLite untenable without third-party replication layers.
- ✅ PostgreSQL's MVCC provides "reading never blocks writing and writing never blocks reading" — verified via postgresql.org docs. This is materially superior for mixed SaaS workloads (auth, admin, background jobs, tenant activity).
- ✅ For single-node deployments, SQLite handles <100K hits/day conservatively — verified via sqlite.org/whentouse.html. The sqlite.org site itself runs on SQLite at 400-500K requests/day.
- ✅ Turso embedded replicas still route writes to a remote primary, with caveats around syncing and serverless environments — verified via docs.turso.tech. This negates the "just a file" simplicity argument for distributed SQLite.
- 🔍 The "distributed SQLite" pattern (Fly.io, Litestream, Turso) is production-viable and enables interesting product patterns (per-tenant isolation, local-first), but specific case studies (buckets.io) could not be fully verified. These are future options, not day-1 requirements.
- The strongest counter-argument (Gemini's) — that distributed SQLite enables product-differentiating features like data sovereignty and offline-first — is valid as a long-term consideration but premature for a team trying to ship a conventional SaaS to <100 users.

## Residual Risks

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| SQLite-first team outgrows single-node and faces painful migration | Medium | Use an ORM (SQLAlchemy, Prisma) from day 1 to abstract DB access. Keep migration path documented. |
| PostgreSQL-first team over-engineers for <100 users | Low | Use a managed service (Neon, Supabase, Railway) to eliminate ops overhead. Free tiers exist. |
| Choosing "distributed SQLite" too early adds unnecessary complexity | Medium | Defer Litestream/Turso until product-market fit is validated and specific requirements (multi-region, offline) emerge. |

## Next Actions

1. **Decide deployment topology first** — will the app run on a single node (VPS, single container) or multiple instances (Kubernetes, serverless, multi-region)? This is the actual decision fork.
2. **If single-node**: Use SQLite with WAL mode + Litestream for backups (not replication). Deploy and iterate.
3. **If multi-instance**: Use managed PostgreSQL (Neon free tier or Supabase). No operational overhead.
4. **Regardless of choice**: Use an ORM abstraction layer to keep the migration path open.

## Debate Summary

| Agent | Rounds | Core Position | Fact-Check Score |
|-------|--------|---------------|------------------|
| Codex (Pragmatist) | 2 | Choose based on deployment topology, not user count. Single-node → SQLite. Distributed → PostgreSQL. Distributed SQLite adds complexity without simplicity gains. | 5✅ 0❌ 0🔍 0⚠️ 0⛔ |
| Gemini (Explorer) | 2 | Distributed SQLite (Litestream, Turso) enables product-differentiating features like per-tenant isolation, data sovereignty, and local-first. Don't dismiss SQLite as only for single-node. | 3✅ 0❌ 2🔍 0⚠️ 0⛔ |

## Economics

- **Mode**: full
- **Rounds**: 2
- **Estimated tokens**: ~45,000 (across all agents + fact-checks)
- **Estimated latency**: ~8 minutes
- **WebSearches performed**: 10
- **Revision Loop invocations**: 0
