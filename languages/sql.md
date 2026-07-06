# SQL / PostgreSQL (`.sql`) — Coding Standards & State-of-the-Art Practices

> **Scope:** PostgreSQL 13+ (verified against PostgreSQL 16) for application
> schemas, queries, and migrations, plus PostGIS 3.x for geospatial data. The
> guidance targets OLTP-style application databases; it applies to raw SQL,
> query-builder output, and SQL embedded in application code alike.
> **Primary sources:** PostgreSQL documentation, PostGIS documentation, the
> PostgreSQL wiki ("Don't Do This").
> **Relationship to general docs:** extends [general docs](../00-index.md). Where a
> general rule and a SQL rule conflict, the SQL rule wins for SQL. Injection
> defense builds on `GEN-SEC-03`; transaction discipline builds on `GEN-ERR-27`
> and `GEN-ERR-29`.

**Rule ID prefix:** `SQL`

---

## Table of contents

1. [Style & formatting](#1-style--formatting)
2. [Data types](#2-data-types)
3. [Keys, constraints & schema design](#3-keys-constraints--schema-design)
4. [Indexing](#4-indexing)
5. [Query patterns](#5-query-patterns)
6. [Transactions](#6-transactions)
7. [Operations: vacuum, statistics & pooling](#7-operations-vacuum-statistics--pooling)
8. [PostGIS](#8-postgis)
9. [Migrations](#9-migrations)
10. [Anti-patterns](#10-anti-patterns)
11. [Quick checklist](#quick-checklist)
12. [References](#references)

---

## 1. Style & formatting

**`SQL-STY-01` (MUST)** Use lowercase `snake_case` for all identifiers (tables,
columns, indexes, constraints, functions). PostgreSQL folds unquoted identifiers
to lowercase, so a mixed-case name (`"OrderItems"`) must be double-quoted in
every statement forever after — one forgotten quote produces a
"relation does not exist" error.

**`SQL-STY-02` (MUST)** List columns explicitly in every `INSERT`. Positional
inserts (`INSERT INTO orders VALUES (...)`) break silently — or worse, put
values in the wrong columns — when columns are added or reordered.

```sql
-- Bad — depends on current column order
INSERT INTO orders VALUES ('7d9...', 42, now());

-- Good — survives schema evolution
INSERT INTO orders (id, item_count, created_at)
VALUES ('7d9...', 42, now());
```

**`SQL-STY-03` (MUST)** Use explicit `JOIN ... ON ...`; never implicit comma joins
(`FROM a, b WHERE a.id = b.a_id`). Comma joins hide the join condition among the
filters, and a forgotten condition silently becomes a cross join instead of a
syntax error.

---

## 2. Data types

**`SQL-TYPE-01` (MUST)** Use `timestamptz` for all timestamp columns, never
`timestamp` (without time zone). `timestamp` stores a "wall clock picture", not a
point in time — arithmetic, comparison, and display are wrong the moment two
sessions or servers disagree on time zone. `timestamptz` stores an unambiguous
instant (UTC internally) and converts on display.

**`SQL-TYPE-02` (MUST)** Use `numeric(precision, scale)` for monetary values.
`float` / `double precision` are binary floating point and cannot represent 0.1
exactly — rounding errors accumulate in sums. The built-in `money` type is also
prohibited: it is fixed-point with locale-dependent formatting, and a locale
change corrupts the interpretation of stored values. Store the currency in a
separate column.

**`SQL-TYPE-03` (SHOULD)** Prefer `text` over `char(n)` and over arbitrary
`varchar(n)` limits. `char(n)` silently space-pads and has confusing equality
semantics. `varchar(n)` has the same storage cost as `text`; an arbitrary limit
(picked "to be safe") becomes a production failure the day real data exceeds it.
Add a length `CHECK` only when it enforces a real business or interop rule.

**When a limit is right:** genuinely fixed-size or bounded external formats
(ISO country codes, IBAN, upstream API contracts) — then use
`text CHECK (length(x) = n)` or a documented `varchar(n)`, so the rule is
explicit rather than incidental.

**`SQL-TYPE-04` (SHOULD)** Use `jsonb` for semi-structured data, not `json`.
`jsonb` is stored decomposed, supports indexing (GIN) and efficient
containment/path operators; `json` is a plain text blob that is reparsed on every
access. Use `json` only if you must preserve exact key order and duplicate keys.

---

## 3. Keys, constraints & schema design

**`SQL-DDL-01` (MUST)** For UUID primary keys, use the built-in
`gen_random_uuid()` (PostgreSQL 13+); `uuid_generate_v4()` requires the
`uuid-ossp` extension and is legacy — equivalent output, unnecessary dependency.
Do not mix UUID generators within one table. For integer keys, use
`GENERATED ALWAYS AS IDENTITY`; **never `SERIAL`** — `SERIAL` is non-standard
syntax sugar with awkward sequence-ownership coupling and permission handling,
and the SQL-standard identity column replaces it.

```sql
-- Good
CREATE TABLE orders (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid()
);
-- or
CREATE TABLE order_events (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);
```

**`SQL-DDL-02` (MUST)** Name constraints explicitly:

```sql
CONSTRAINT fk_orders_user FOREIGN KEY (user_id)
  REFERENCES users (id) ON DELETE CASCADE,
CONSTRAINT chk_orders_total_positive CHECK (total_amount > 0)
```

**Why:** PostgreSQL auto-generates deterministic names (`orders_total_check`),
but they can collide (gaining numeric suffixes), diverge when DDL history differs
between environments, and make a later `DROP CONSTRAINT` a fragile guessing game.
An explicit name is stable across environments and self-documenting.

**`SQL-DDL-03` (MUST)** Declare `ON DELETE` behaviour explicitly on every foreign
key (`CASCADE`, `RESTRICT`, or `SET NULL`). The default (`NO ACTION`) may be the
right choice, but writing the clause forces the decision to be made and read
rather than inherited by accident.

**`SQL-DDL-04` (MUST)** A `CHECK` constraint on a nullable column does **not**
reject `NULL` — per SQL semantics, a check expression that evaluates to NULL
passes. When a default plus a `CHECK` is meant to define a closed set of values,
add `NOT NULL`, or the constraint has a silent bypass.

```sql
-- Bad — UPDATE orders SET status = NULL succeeds
status text DEFAULT 'pending' CHECK (status IN ('pending', 'shipped')),

-- Good — the set is actually closed
status text NOT NULL DEFAULT 'pending'
  CHECK (status IN ('pending', 'shipped'))
```

**`SQL-DDL-05` (SHOULD)** Enforce invariants in the schema, not only in
application code. The database outlives any single code path (admin consoles,
bulk imports, future services all write to it). Tools:

- **Partial unique indexes** for "at most one row in state X" invariants:

  ```sql
  -- exactly one current version per document lineage
  CREATE UNIQUE INDEX uq_document_versions_current
    ON document_versions (document_id) WHERE is_current = true;
  ```

- **`CHECK` constraints** for value ranges and cross-column rules.
- **`jsonb_typeof()` checks** to pin the top-level shape of JSONB columns:

  ```sql
  CONSTRAINT chk_attributes_is_object
    CHECK (jsonb_typeof(attributes) = 'object')
  ```

**`SQL-DDL-06` (MUST)** Treat database constraints as the authority for
uniqueness and invariants; a check-then-insert sequence is a TOCTOU race — two
concurrent requests both pass the `SELECT`, then both `INSERT`. Attempt the write
and handle the conflict (`ON CONFLICT` or catching the unique-violation error).
Pre-checks are acceptable only as advisory UX (friendlier error messages), never
as the enforcement mechanism. (General rule: `GEN-ERR-29`.)

```sql
-- Bad — race window between check and insert
-- app: SELECT 1 FROM users WHERE email = $1; if absent: INSERT ...

-- Good — the unique constraint decides
INSERT INTO users (email, name) VALUES ($1, $2)
ON CONFLICT (email) DO NOTHING;
```

**`SQL-DDL-07` (SHOULD)** An `updated_at` column without a `BEFORE UPDATE`
trigger silently goes stale: every writer that forgets to set it produces rows
whose timestamp lies. Either attach a shared touch trigger, or document exactly
which single code path owns the column and sets it.

```sql
CREATE OR REPLACE FUNCTION touch_updated_at() RETURNS trigger AS $$
BEGIN
  NEW.updated_at := now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_orders_touch
  BEFORE UPDATE ON orders
  FOR EACH ROW EXECUTE FUNCTION touch_updated_at();
```

**Trade-off:** a trigger guarantees correctness for all writers but hides a write
in the schema (surprising in debugging, one extra function call per update);
application-managed timestamps keep logic visible but rely on discipline across
every writer. Pick one per project and apply it uniformly — the failure mode is
having neither.

---

## 4. Indexing

**`SQL-IDX-01` (MUST)** Index every foreign-key column explicitly. PostgreSQL
indexes the *referenced* side (via the PK/unique constraint) but does **not**
auto-create an index on the *referencing* side. Without one, every `DELETE` or
key `UPDATE` on the parent table sequentially scans the child table to check the
constraint, and FK joins lose their index path.

```sql
CREATE TABLE order_items (
  order_id uuid NOT NULL,
  CONSTRAINT fk_order_items_order FOREIGN KEY (order_id)
    REFERENCES orders (id) ON DELETE CASCADE
);
CREATE INDEX idx_order_items_order ON order_items (order_id);
```

**`SQL-IDX-02` (SHOULD)** Use partial indexes when queries consistently filter on
a high-selectivity condition:

```sql
CREATE INDEX idx_orders_open ON orders (user_id) WHERE status = 'open';
```

**Pros:** far smaller and faster than a full index when the filtered-out rows
dominate; cheaper to maintain on writes to non-matching rows.
**Cons:** only usable by queries whose predicate provably implies the index
predicate; one more thing to keep in sync when the state model changes.
Use when the hot query always carries the filter; avoid when the same column is
queried with varied predicates.

**`SQL-IDX-03` (SHOULD)** Keep predicates **sargable**: wrapping an indexed
column in a function inside `WHERE` disables index use unless a matching
expression index exists. Either rewrite the predicate to leave the column bare,
or create an expression index:

```sql
-- Non-sargable — index on (email) is not used
SELECT id FROM users WHERE lower(email) = lower($1);

-- Expression index makes it sargable (and enforces case-insensitive uniqueness)
CREATE UNIQUE INDEX uq_users_email_lower ON users (lower(email));
```

The same applies to casts and date-part extraction: prefer
`created_at >= $1 AND created_at < $2` over `date(created_at) = $1`.

---

## 5. Query patterns

**`SQL-QRY-01` (SHOULD NOT)** Avoid `SELECT *` in application code. It fetches
unused data, prevents Index Only Scans, silently changes the result shape when
columns are added, and breaks positional consumers. List the columns the code
actually uses. (`SELECT *` is fine in ad-hoc psql exploration.)

**`SQL-QRY-02` (SHOULD)** Use CTEs (`WITH ... AS (...)`) to structure multi-step
queries — they document intent. Since PostgreSQL 12 the planner **inlines** a CTE
by rule when it is non-recursive, side-effect-free, and referenced exactly once
(it materializes otherwise), so CTEs are no longer an automatic optimization
fence. Override explicitly when needed: `MATERIALIZED` to force a
computed-once snapshot (e.g. an expensive subquery referenced in a way the
planner would inline badly), `NOT MATERIALIZED` to force inlining.

**`SQL-QRY-03` (SHOULD)** Use keyset (seek) pagination instead of `OFFSET` for
large or deep result sets. `OFFSET n` scans and discards all `n` preceding rows,
so cost grows linearly with page depth; keyset pagination costs the same for
every page.

```sql
-- Bad — page 1000 scans and throws away 50,000 rows
SELECT id, name FROM users ORDER BY id LIMIT 50 OFFSET 50000;

-- Good — constant cost at any depth
SELECT id, name FROM users WHERE id > $last_seen_id ORDER BY id LIMIT 50;
```

**Trade-off:** keyset pagination cannot jump to an arbitrary page number and
needs a stable, unique sort key (append a tiebreaker column when sorting on
non-unique columns). For small, shallow result sets `OFFSET` is fine.

**`SQL-QRY-04` (SHOULD)** Use `NOT EXISTS (SELECT 1 FROM ...)` instead of
`NOT IN (subquery)` when the subquery can return NULLs. `col NOT IN (NULL, 1, 2)`
can never evaluate to `TRUE` — non-matching rows yield `NULL` (unknown), which
`WHERE` treats as false — so a single NULL in the subquery silently returns zero
rows.

```sql
-- Bad — returns no rows if any orders.user_id is NULL
SELECT * FROM users WHERE id NOT IN (SELECT user_id FROM orders);

-- Good — NULL-safe, and typically plans better (anti-join)
SELECT * FROM users u
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

**`SQL-QRY-05` (SHOULD)** Avoid `count(*)` on large tables in hot paths. Under
MVCC an exact count must visit every visible row (full table or index scan). For
approximate counts read `pg_class.reltuples` (maintained by
`VACUUM`/`ANALYZE`); for pagination UIs, keyset pagination plus a "has next"
probe (fetch `LIMIT n + 1`) is usually sufficient.

**`SQL-QRY-06` (SHOULD)** Verify plans with `EXPLAIN (ANALYZE, BUFFERS)` during
development. `Seq Scan` on a large table, `Filter` where an `Index Cond` was
expected, and temp spills (`temp written` in the plan) are optimization targets.
Enable `pg_stat_statements` in production for slow-query triage; its
`temp_blks_written` column identifies queries spilling to disk (often a
`work_mem` or missing-index problem).

**`SQL-QRY-07` (MUST NOT)** Never build SQL by string concatenation or
interpolation of external input — that is SQL injection. Use parameterized
queries / prepared statements everywhere; identifiers that must be dynamic go
through a strict allow-list or the driver's identifier-quoting API
(`GEN-SEC-03`).

**`SQL-QRY-08` (SHOULD)** Use half-open ranges (`>= AND <`) instead of `BETWEEN`
on timestamps. `BETWEEN` includes both endpoints, so adjacent windows
double-count the boundary instant; and "end of day" literals like `23:59:59`
silently drop the final second's sub-second timestamps.

```sql
-- Bad — midnight belongs to both days
WHERE created_at BETWEEN '2026-01-01' AND '2026-01-02'

-- Good — half-open interval, no overlap, no gap
WHERE created_at >= '2026-01-01' AND created_at < '2026-01-02'
```

---

## 6. Transactions

**`SQL-TXN-01` (MUST)** Wrap multi-step write operations in a transaction.
Related `INSERT`/`UPDATE`/`DELETE` statements must succeed or fail as a unit;
without a transaction, a mid-sequence failure leaves the database in a state no
code path was designed to handle (failure atomicity: `GEN-ERR-26`).

**`SQL-TXN-02` (SHOULD)** Keep transactions short. A long-running transaction
pins the xmin horizon, which blocks `VACUUM` from reclaiming dead tuples across
the whole cluster — causing table bloat — and contributes to transaction-ID
wraparound pressure; it also holds row and relation locks that stall other
sessions. Do preparation (parsing, validation, data assembly) before `BEGIN`.

**`SQL-TXN-03` (MUST NOT)** No network I/O inside an open transaction
(`GEN-ERR-27`). Download, parse, and validate before opening the transaction. A
remote call mid-transaction holds locks and a pooled connection for the full
network latency (or timeout) and couples the transaction's fate to an external
system.

**`SQL-TXN-04` (SHOULD)** Fire side effects (notifications, emails, cache
invalidation, external API calls) only **after** commit. A side effect emitted
mid-transaction becomes a lie if the transaction rolls back, and re-fires on
every retry. If the side effect must be atomic with the write, record it in an
outbox table within the same transaction and dispatch it afterwards.

---

## 7. Operations: vacuum, statistics & pooling

**`SQL-OPS-01` (SHOULD)** Tune autovacuum on high-write tables rather than
accepting the global defaults. Lowering per-table
`autovacuum_vacuum_scale_factor` (e.g. from the default `0.2` to `0.05`) makes
vacuum run before bloat accumulates:

```sql
ALTER TABLE order_events
  SET (autovacuum_vacuum_scale_factor = 0.05);
```

Run `ANALYZE table_name` explicitly after large bulk loads — the planner's row
statistics are otherwise stale until autovacuum catches up, and stale statistics
produce bad plans (e.g. sequential scans where an index was appropriate).

**`SQL-OPS-02` (SHOULD)** Use a server-side connection pooler (PgBouncer in
`transaction` mode) instead of raising `max_connections`. Each PostgreSQL
backend is a process with significant memory overhead; a pooler multiplexes many
application connections over few backends.

**Caveat:** transaction-mode pooling breaks session-scoped state — `SET`
(session-level settings), session advisory locks, `LISTEN`/`NOTIFY`, and named
prepared statements prepared at session scope. Route workloads that need session
state around the pooler (direct connection or `session` mode), and use
`SET LOCAL` for per-transaction settings.

---

## 8. PostGIS

**`SQL-GIS-01` (MUST)** Choose `geometry` vs `geography` deliberately, per
column, and do not mix them within a query. `geography` uses geodetic arithmetic
(results in metres, accurate globally, fewer functions available); `geometry`
uses planar arithmetic (full function set, results in the SRID's units). Mixing
the two in one predicate forces casts that defeat index usage. A `::geography`
cast *inside a measurement expression* is fine (see `SQL-GIS-07`) — the rule is
about column types and index-serving predicates.

**`SQL-GIS-02` (MUST)** Type geometry columns with an explicit typmod —
`geometry(Point, 4326)`, not bare `geometry`. The typmod makes the database
enforce geometry type and SRID at write time and documents both in the column
definition; a bare `geometry` column happily accepts a mix of SRIDs, which
corrupts every subsequent distance computation.

**`SQL-GIS-03` (MUST)** Create a GiST index on every spatially-queried column,
and always write `USING GIST` — the default index type is B-tree, which cannot
serve spatial operators:

```sql
CREATE INDEX idx_locations_geom ON locations USING GIST (geom);
```

Spatial operators use the bounding-box index for O(log n) candidate filtering;
without it every spatial predicate is a full-table exact-geometry test.

**`SQL-GIS-04` (MUST)** Use `ST_DWithin(a, b, dist)` for proximity filters, not
`ST_Distance(a, b) < dist`. `ST_DWithin` is index-aware (it performs an indexed
bounding-box pre-filter, then exact tests on candidates); `ST_Distance` in a
`WHERE` clause is a plain function call on every row — non-sargable, full scan.

**`SQL-GIS-05` (SHOULD)** Rely on the built-in index awareness of the standard
relationship predicates — `ST_Intersects`, `ST_Contains`, `ST_Within`,
`ST_DWithin`, etc. already include an implicit `&&` bounding-box test that uses
the GiST index. Do not add a redundant explicit `&&` alongside them. Add an
explicit `&&` pre-filter only for predicates that lack the implicit test
(e.g. `ST_Relate`) or for custom distance/expression predicates:

```sql
WHERE geom && ST_MakeEnvelope($xmin, $ymin, $xmax, $ymax, 4326)
  AND ST_Relate(geom, $other, 'T*T***T**')
```

**`SQL-GIS-06` (SHOULD)** For nearest-neighbour queries, use the KNN operator
`<->` in `ORDER BY` with a `LIMIT` — it walks the GiST index in distance order.
`ORDER BY ST_Distance(...)` without a `LIMIT` computes the exact distance for
every row and sorts.

```sql
SELECT id, name
FROM locations
ORDER BY geom <-> ST_SetSRID(ST_MakePoint($lon, $lat), 4326)
LIMIT 10;
```

**`SQL-GIS-07` (MUST)** SRID 4326 units are **degrees**, not metres — never
compare `ST_Distance` on raw 4326 geometry against a metre value. For metric
results, cast to geography in the expression
(`ST_Distance(a::geography, b::geography)`) or use `ST_DistanceSphere`; for
heavy analytical workloads, transform to a suitable national CRS or UTM zone.
**Never use EPSG:3857 (Web Mercator) for measurement** — its scale error grows
as 1/cos(latitude) (roughly 2× at 60°N); it is a display projection only.

**`SQL-GIS-08` (CONSIDER)** To combine a spatial column with a scalar column in
one compound GiST index, the `btree_gist` extension is required to supply GiST
operator classes for the scalar type:

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;
CREATE INDEX idx_events_geom_date ON events USING GIST (geom, event_date);
```

**Trade-off:** a separate B-tree index on the scalar column is usually the
better choice — the planner can combine both indexes, and each stays smaller and
more generally useful. Reach for `btree_gist` only when the combined
spatial+scalar predicate is the dominant query pattern and profiling shows the
two-index plan is insufficient.

**`SQL-GIS-09` (SHOULD)** Run `VACUUM ANALYZE` after bulk spatial loads. The
planner decides whether to use the spatial index based on row statistics; stale
statistics after a large load cause sequential scans over freshly indexed data.

---

## 9. Migrations

**`SQL-MIG-01` (MUST)** One logical DDL change per migration file, numbered
sequentially (`042_add_order_discounts.sql`). Never bundle unrelated changes —
independent changes must be independently revertable, reviewable, and bisectable.

**`SQL-MIG-02` (MUST)** Migrations are **append-only**: once merged, a migration
is never edited. Environments that already ran the old text will silently
diverge from environments that run the edited text. If a migration was wrong,
ship a new corrective migration. **CONSIDER** enforcing this in CI with a
checksum lock (most migration frameworks record per-file hashes and fail on
mismatch).

**`SQL-MIG-03` (MUST)** Start every migration with a "why" comment: the business
reason, any caveats (e.g. an intentionally unenforced FK), and the expected state
after it runs. DDL says *what* changed; only the comment can say *why*, and the
migration file is the one place future readers will look.

**`SQL-MIG-04` (MUST)** Keep idempotency uniform within a file. Mixing guarded
statements (`CREATE TABLE IF NOT EXISTS`) with unguarded ones (bare
`CREATE INDEX`) produces a file that half-applies on replay: the guarded
statements pass, the unguarded ones abort, and the schema lands in a state
neither "applied" nor "not applied". Either the migration runner guarantees
exactly-once execution (then write plain DDL), or every statement is guarded.

**`SQL-MIG-05` (SHOULD)** Use `CREATE INDEX CONCURRENTLY` when adding indexes to
live production tables — a plain `CREATE INDEX` blocks writes to the table for
the duration of the build. Two constraints follow:

- `CONCURRENTLY` **cannot run inside a transaction block**, so it must be
  excluded from the migration runner's usual wrapping transaction.
- A failed concurrent build leaves an **`INVALID`** index behind that must be
  dropped and rebuilt manually.

**When plain `CREATE INDEX` is fine:** empty or small tables, bootstrap/dev
migrations, maintenance windows. Document which mode a migration assumes.

**`SQL-MIG-06` (SHOULD)** Treat `ALTER TYPE ... ADD VALUE` (enum extension) as
transaction-restricted, like `CONCURRENTLY`: a value added inside a transaction
block cannot be *used* until that transaction commits (and before PostgreSQL 12
the statement could not run inside a transaction block at all). A migration that
adds an enum value and immediately inserts rows using it will fail under a
transactional runner — split it into two migrations, or prefer a `text` column
with a `CHECK` constraint, which has none of these restrictions.

**`SQL-MIG-07` (SHOULD)** Never run `ALTER COLUMN ... TYPE` on a large table in a
single migration without a plan: most type changes rewrite the entire table under
an `ACCESS EXCLUSIVE` lock (all reads and writes blocked for the duration).
Binary-coercible changes — `varchar(n)` → `text`, widening a `varchar` limit —
are metadata-only and instant; verify before assuming. For everything else on
big tables, use expand/backfill/contract across multiple migrations: add the new
column → backfill in batches → switch constraints and code → drop the old
column.

---

## 10. Anti-patterns

| Avoid | Why | Instead |
|---|---|---|
| `SELECT *` in application code | Prevents Index Only Scans; fetches unused data; shape changes silently | List specific columns (`SQL-QRY-01`) |
| `NOT IN (subquery)` on nullable column | One NULL makes the predicate never TRUE — zero rows | `NOT EXISTS` (`SQL-QRY-04`) |
| `char(n)` | Space-pads silently; confusing equality | `text`, with `CHECK (length(x) = n)` if truly fixed-length (`SQL-TYPE-03`) |
| Arbitrary `varchar(n)` limits | Same storage as `text`; arbitrary limits become prod failures | `text`; constrain only for real business rules (`SQL-TYPE-03`) |
| `SERIAL` | Non-standard; sequence-ownership coupling | `GENERATED ALWAYS AS IDENTITY` or UUID (`SQL-DDL-01`) |
| `timestamp` without time zone | Stores a clock picture, not a point in time | `timestamptz` (`SQL-TYPE-01`) |
| `money` type | Locale-dependent; corruptible by locale change | `numeric` + currency column (`SQL-TYPE-02`) |
| `BETWEEN` on timestamps | Includes both endpoints — double-counts the boundary | `>= AND <` (`SQL-QRY-08`) |
| Unnamed constraints | Auto-names collide and diverge across environments | Explicit `CONSTRAINT` names (`SQL-DDL-02`) |
| Unindexed FK columns | Sequential scan on child for every parent delete/update | Index every FK column (`SQL-IDX-01`) |
| Functions on indexed columns in `WHERE` | Non-sargable; disables the index | Expression index or rewrite the predicate (`SQL-IDX-03`) |
| `OFFSET` for deep pagination | Scans and discards all preceding rows | Keyset pagination (`SQL-QRY-03`) |
| `count(*)` on large tables in hot paths | MVCC exact count scans all visible rows | `reltuples` estimate; keyset + has-next (`SQL-QRY-05`) |
| Check-then-insert uniqueness | TOCTOU race between concurrent writers | Constraint + `ON CONFLICT` (`SQL-DDL-06`) |
| Long-running transactions | Blocks VACUUM; bloat; XID-wraparound pressure; held locks | Short transactions; no network I/O inside (`SQL-TXN-02/03`) |
| String-concatenated SQL | SQL injection | Parameterized queries (`SQL-QRY-07`) |
| Rules (`CREATE RULE`) | Query rewriting with non-intuitive semantics | Triggers for logic; views for read-time transforms |
| Table inheritance | Poor real-world results (constraints, indexes don't inherit cleanly) | Foreign keys; declarative partitioning for scale |
| Bare `geometry` columns | Accepts mixed SRIDs/types; corrupts measurements | `geometry(Point, 4326)` typmod (`SQL-GIS-02`) |
| `ST_Distance(...) < x` in `WHERE` | Non-sargable; full scan | `ST_DWithin` (`SQL-GIS-04`) |
| Measuring in EPSG:3857 | Scale error grows as 1/cos(lat) | Geography cast, `ST_DistanceSphere`, or a suitable projected CRS (`SQL-GIS-07`) |
| Editing merged migrations | Environments silently diverge | Append a corrective migration (`SQL-MIG-02`) |

---

## Quick checklist

- [ ] Identifiers lowercase snake_case; explicit column lists in `INSERT`;
      explicit `JOIN ... ON` (`SQL-STY-*`).
- [ ] `timestamptz` for all timestamps; `numeric` for money (no float, no
      `money` type); `text` over `char(n)`/arbitrary `varchar(n)`; `jsonb` over
      `json` (`SQL-TYPE-*`).
- [ ] PKs via `gen_random_uuid()` or `GENERATED ALWAYS AS IDENTITY`; never
      `SERIAL`; no mixed UUID generators per table (`SQL-DDL-01`).
- [ ] Constraints named explicitly; `ON DELETE` explicit on every FK
      (`SQL-DDL-02/03`).
- [ ] `NOT NULL` wherever a default + `CHECK` implies a closed set — NULL
      bypasses `CHECK` (`SQL-DDL-04`).
- [ ] Invariants in the schema: partial unique indexes, `CHECK`,
      `jsonb_typeof()`; conflicts handled via constraints, not check-then-insert
      (`SQL-DDL-05/06`).
- [ ] `updated_at` maintained by a touch trigger or a documented owning writer
      (`SQL-DDL-07`).
- [ ] Every FK column indexed; partial/expression indexes where filters are
      fixed; predicates sargable (`SQL-IDX-*`).
- [ ] No `SELECT *` in app code; keyset over `OFFSET`; `NOT EXISTS` over
      `NOT IN`; no hot-path `count(*)`; plans verified with
      `EXPLAIN (ANALYZE, BUFFERS)`; `pg_stat_statements` enabled (`SQL-QRY-*`).
- [ ] Parameterized queries only — no string-built SQL (`SQL-QRY-07`).
- [ ] `>= AND <` instead of `BETWEEN` on timestamps (`SQL-QRY-08`).
- [ ] Multi-step writes in short transactions; no network I/O inside; side
      effects after commit (`SQL-TXN-*`).
- [ ] Autovacuum tuned on high-write tables; `ANALYZE` after bulk loads;
      pooler in transaction mode with session-state caveats routed around
      (`SQL-OPS-*`).
- [ ] PostGIS: typmod'd columns, GiST indexes (`USING GIST`), `ST_DWithin`, KNN
      `<->` + `LIMIT`, metric measurements via geography/`ST_DistanceSphere`,
      never EPSG:3857 for measurement, `VACUUM ANALYZE` after bulk loads
      (`SQL-GIS-*`).
- [ ] Migrations: one change per file, append-only with correctives, "why"
      header, uniform idempotency, `CONCURRENTLY` for live tables (outside
      transactions; watch for `INVALID`), enum `ADD VALUE` transaction
      restriction, expand/backfill/contract for type changes on big tables
      (`SQL-MIG-*`).

---

## References

- PostgreSQL documentation — https://www.postgresql.org/docs/current/
- PostgreSQL wiki: "Don't Do This" — https://wiki.postgresql.org/wiki/Don%27t_Do_This
- PostGIS documentation — https://postgis.net/documentation/
- PostGIS workshop: spatial indexing — https://postgis.net/workshops/postgis-intro/indexing.html
- `CREATE INDEX` (incl. `CONCURRENTLY` semantics) — https://www.postgresql.org/docs/current/sql-createindex.html
- `ALTER TYPE` (enum `ADD VALUE` restrictions) — https://www.postgresql.org/docs/current/sql-altertype.html
- `pg_stat_statements` — https://www.postgresql.org/docs/current/pgstatstatements.html
- PgBouncer features (pooling-mode caveats) — https://www.pgbouncer.org/features.html
