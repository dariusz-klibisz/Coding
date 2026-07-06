# SQL / PostgreSQL Review Checklist

> Use with [`../languages/sql.md`](../languages/sql.md) and
> [`general.md`](general.md).

- [ ] Lowercase snake_case identifiers; explicit column lists in `INSERT`;
      explicit `JOIN ... ON`, no comma joins (`SQL-STY-*`).
- [ ] `timestamptz` everywhere; `numeric` for money (no float/`money`); `text`
      over `char(n)`/arbitrary `varchar(n)`; `jsonb` over `json` (`SQL-TYPE-*`).
- [ ] PKs via `gen_random_uuid()` or `GENERATED ALWAYS AS IDENTITY` — never
      `SERIAL`; no mixed UUID generators in one table (`SQL-DDL-01`).
- [ ] Constraints explicitly named; `ON DELETE` declared on every FK; `NOT NULL`
      where a default + `CHECK` implies a closed set (NULL bypasses `CHECK`)
      (`SQL-DDL-02/03/04`).
- [ ] Invariants enforced in schema (partial unique indexes, `CHECK`,
      `jsonb_typeof()`); writes handle conflicts via constraints, not
      check-then-insert (`SQL-DDL-05/06`).
- [ ] `updated_at` has a touch trigger or a documented owning writer
      (`SQL-DDL-07`).
- [ ] Every FK column indexed; predicates sargable (expression indexes where
      needed); partial indexes for fixed filters (`SQL-IDX-*`).
- [ ] No `SELECT *` in app code; keyset over `OFFSET`; `NOT EXISTS` over
      `NOT IN` on nullables; no hot-path `count(*)`; `>= AND <` not `BETWEEN`
      on timestamps; plans checked with `EXPLAIN (ANALYZE, BUFFERS)`
      (`SQL-QRY-*`).
- [ ] Parameterized queries only — no string-concatenated SQL (`SQL-QRY-07`,
      `GEN-SEC-03`).
- [ ] Multi-step writes in short transactions; no network I/O inside an open
      transaction; side effects after commit (`SQL-TXN-*`, `GEN-ERR-27`).
- [ ] Autovacuum tuned on high-write tables; `ANALYZE` after bulk loads;
      transaction-mode pooler caveats (SET / advisory locks / LISTEN-NOTIFY)
      respected (`SQL-OPS-*`).
- [ ] PostGIS: `geometry(Type, SRID)` typmods; `USING GIST` on spatial columns;
      `ST_DWithin` not `ST_Distance < x`; KNN `<->` with `LIMIT`; metric
      measurements via geography cast/`ST_DistanceSphere` — never EPSG:3857;
      `VACUUM ANALYZE` after bulk spatial loads (`SQL-GIS-*`).
- [ ] Migrations: one logical change per file; append-only (correctives, never
      edits); "why" header comment; uniform idempotency within a file;
      `CONCURRENTLY` for live tables (no transaction; failed builds leave
      `INVALID` indexes); enum `ADD VALUE` split from first use;
      expand/backfill/contract for type changes on large tables (`SQL-MIG-*`).
