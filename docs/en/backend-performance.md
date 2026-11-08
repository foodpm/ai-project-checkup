# AI Backend Performance Baseline (read before you build)

> Check IDs: BP-01 to BP-13
>
> This is a stack-agnostic performance baseline. Examples use a SQLite/D1-style database and a serverless runtime; the rules transfer to PostgreSQL, MySQL, and similar stacks. No project names, domains, or real business data appear here.

---

## 1. Think before you write SQL (mandatory)

1. **Estimate the scan first.** How many rows will this query scan? Which path does it run on (user hot path / high-frequency cron / low-frequency admin)?
2. **Optimize in three tiers:** indexed point lookup > indexed range scan > full-table aggregation.
3. **The path sets the bar:**
   - User hot path (runs on every request): indexed point lookups only; rows scanned should be single digits.
   - High-frequency scheduled job (every minute / every 5 minutes): must use indexed range scans.
   - Low-frequency admin: range scans are tolerable, but **never** run unbounded `group by` or full-table aggregation on large tables (use snapshots, see section 3). ⚠️ Admin is not a free pass: while debugging, admins issue hot-path-level load, so admin queries are held to hot-path standards. Where this conflicts, section 6 wins.
   - Any path: no unpaginated large result sets; no deep pagination with a large `OFFSET` (O(offset) scan). Use keyset pagination: `where id > last_id_from_previous_page order by id limit ?`.

---

## 2. When an index is required (decision list)

Build an index — and verify at launch that it actually exists — if any of these apply:

| Case | Requirement |
|------|-------------|
| Queries that run on every request (session lookup by auth token, config reads) | Index unconditionally, even if the table is small |
| WHERE equality/range columns | Single-column or composite index |
| JOIN / subquery join columns | Index on both sides |
| ORDER BY + LIMIT sort columns | Index the sort column to avoid full sorts |
| High-frequency cron filters | Composite index covering the combination, e.g. `(status, created_at)` |
| Columns in `count(distinct ...)` / `group by` | Index only if the query is high-frequency; otherwise use a snapshot |

**Threshold rule:** `query frequency × rows scanned per run` reaching an order of "a million rows per day" demands remediation.

**Composite index column order:** equality columns first, range columns second (e.g. `(status, created_at)`), and respect the leftmost-prefix rule — queries must match from the leftmost column, or the composite index does nothing.

### Three hard rules for index DDL (break one and the index does not exist)

1. **DDL location:** The initialization chain (e.g. `applySchema`) is short-circuited on existing databases by a version guard or a ready flag. **New DDL written there never executes.** Put it in an always-run block that executes on every cold start.
2. **Version bump:** If the project has a schema-version guard, new or changed DDL must bump the version (e.g. `SCHEMA_VERSION`), or the always-run block is short-circuited too. ⚠️ Bump from the current value in source — do not trust your memory of an old value.
3. **Ordering:** Indexes must come after the corresponding `CREATE TABLE`. On a fresh empty database, an index before the table fails with `no such table`.

**Verify after deploy** (deploy ≠ effective):

```bash
# Confirm the table/index actually exists
your-db-cli execute <db> --remote --json --command \
  "select count(*) from sqlite_master where name='<object-name>'"
# Verify after the first request creates the schema; cross-session reads have second-level lag,
# so trigger another request before concluding it is missing
```

### Patterns that defeat indexes (forbidden)

| Forbidden pattern | Consequence | Correct rewrite |
|----------|------|----------|
| `where a.col = 'const' \|\| b.col` (expression-linked join) | The optimizer cannot reverse it; table b degrades to a nested full scan | Keep the constant prefix for `like` and reverse with `substr()`: `a.col like 'const%' and substr(a.col, N) = b.col` |
| `join b on coalesce(b.x, b.id) = p.key` (function-wrapped join key) | Index unusable; every row of p triggers a full scan of b | Rewrite as an indexable OR: `b.id = p.key or b.x = p.key`; the plan should show a two-index lookup |
| Functions on columns: `where date(col) >= x`, `strftime(...)` wrapping a column, `where coalesce(x, y) = const` | A function/expression around a column defeats the index | Put functions on the constant side: `col >= strftime('%Y-%m-%dT%H:%M:%fZ','now','-30 minutes')`; expand coalesce to `x = c or (x is null and y = c)` |
| `like '%xxx'` with a leading wildcard | Full table scan | Prefix matching `like 'xxx%'` can use an index (escape `%`/`_` in user input and declare `escape '\'`); infix search needs a purpose-built index or lower frequency |
| `// comment` inside a SQL template string | SQLite only recognizes `--`; `//` is parsed as division → syntax error → 500 | Use only `--` inside SQL templates, or omit comments and explain in code comments |

### Verifying equivalence and speed after a rewrite

- **Semantic equivalence:** Compare old and new result sets with a bidirectional `EXCEPT` diff; replace only when diff = 0. (Live data changes constantly, so compare mismatches rather than totals.)
- **Performance:** Time the old and new versions and record elapsed time plus rows scanned (two orders of magnitude is common).
- **Execution plan:** `explain query plan` showing `SCAN` means a full table scan. (Dialect differences can make explain itself error out; fall back to timing.) The correct shape for an OR is `MULTI-INDEX OR` with `USING INDEX` on each branch.

### Query patterns in code: no N+1

A query per loop iteration (read N+1) or an insert per iteration (write N+1) turns one request into N round trips; no amount of SQL tuning saves it:

- Reads: merge per-row queries into a batched `in (...)` query and join in memory.
- Writes: use a batch API (e.g. `db.batch()`) or a multi-row `insert into ... values (...), (...)`; never commit one row per iteration.

---

## 3. When to compute stats via scheduled snapshots (decision list)

**Take a snapshot instead of recomputing per request when all of these hold:**

1. The query is a full-table or wide-range `group by`, or a cumulative metric (historical TopN, historical daily peak, total ranking).
2. Day- or hour-level staleness is acceptable.
3. The query is read-heavy and recomputes the same thing on every request.

**Anti-pattern (a common real-world cost):** an admin leaderboard with several cumulative Top-10s plus a report with several historical peaks, each page load running a dozen full-table `group by` queries over millions of rows.

### Standard pattern: daily snapshot + incremental merge

- **Snapshot table design:** composite primary key `(tenant, stat_date)` (one row per tenant/product per day; for a single tenant this can degrade to `stat_date` alone) + a metrics JSON column + an **`as_of` timestamp** (the snapshot's data cutoff and the anchor for incremental merging). Generate with `upsert on conflict do update`.
- **Generation:** A daily low-peak cron computes "yesterday". Backfill history on first launch, **at most N days per run** (batch it to avoid execution timeouts).
- **Reading (incremental merge):** Read the latest snapshot → aggregate only the delta where `time_column >= as_of` → merge and re-sort by key. The snapshot is always slightly stale but the page is real-time.
- **Progressive rollout (important):** The read path must check whether the snapshot covers the full history (`min(stat_date) <= data_start && max(stat_date) >= yesterday`). **If not covered, fall back to the old real-time query automatically** — launch-day behavior stays identical, and the switch happens once backfill completes, with no manual intervention. The fallback path must log or emit a counter; otherwise a failed backfill becomes an unnoticed "permanent silent degradation".
- **Freeze the metric definition:** Decide at snapshot-generation time which tables and which statuses count as "paid". Multi-tenant/multi-product definitions may differ; hard-code each one rather than inferring at runtime.
- **Time zone and day boundary:** Which time zone `stat_date` uses (e.g. an Asia/Shanghai calendar day) and what hour the daily cron runs must be hard-coded, never left to the runtime's default time zone.
- **Robustness:** The snapshot reader must handle "table missing", "snapshot row empty", and "snapshot JSON column fails to parse" — otherwise one `select` takes the whole endpoint down with a 500.

### When not to snapshot

Today's real-time data, per-user detail, payment status, and other strongly real-time data should use indexed range scans (`created_at >= today`) and naturally touch few rows. No snapshot needed.

---

## 4. When to cache (decision list)

**Cache only when all three hold: same-parameter requests repeat heavily + computation is expensive + minute-level staleness is acceptable.**

| Suitable | Not suitable |
|------|--------|
| Admin stats pages (admins repeatedly refresh the same date range) | Per-user data (caching mixes users unless partitioned by user) |
| Derived queries over external APIs (forwarding is costly) | Strongly consistent data such as amounts, balances, payment status |
| Near-static global config (up to 10 minutes) | Read-after-write scenarios |

**Implementation notes:**

- Map cache: key = fully serialized query parameters; TTL per tolerance (5 minutes is plenty for a stats page).
- In multi-tenant/multi-product systems the key must include tenant/product dimensions, or data crosses tenants.
- **Enforce a capacity cap** (e.g. clear everything past 50 entries); otherwise memory grows unbounded.
- On serverless cold instances, cache lifetime = instance lifetime. Hits are a bonus; misses are normal. Never rely on the cache for correctness.
- Cache **read results**, not write intent. No write path may pass through the cache layer.

---

## 5. Launch verification and monitoring backstops

1. **Deploy flow:** Build the front end before deploying (deploy does not build automatically). After deployment, confirm a key string or version from the built artifact is actually live.
2. **DDL triple check:** Version bumped → object exists in `sqlite_master` → actual queries use the index.
3. **Usage alerts:** Performance problems often burn money silently for days. Configure hourly alerts for database rows read and rows written (unbounded insert tables burn write rows, see 6.4), plus hourly/per-database usage reports. A spike in the curve points directly to "when it started and which query introduced it".
4. **Govern the alerts themselves:** Deduplicate alerts from the same source (when multiple tenants share an account-level aggregate, only one should report) and use human units (millions/billions, not raw digits).
5. **Lessons first:** Turn every incident's fix into a written rule. When a similar need appears, apply the rule before writing code.

---

## 6. Admin, upsert, and data governance

> The five patterns below come from real usage-alert incident reviews; business data has been abstracted to generic magnitudes.

### 1. Admin is not a "low-frequency free pass"

**Typical incident:** One admin list endpoint = a count plus a list query on each page load, and each one unconditionally runs a window aggregation over a million-row table (nearly 10 million rows per load, multi-second P50). An admin debugging for half an hour, searching/paging/filtering dozens of times, reads hundreds of millions of rows and trips an hourly usage alert. All business traffic combined is a fraction of the alert baseline — **the culprit is the admin panel, not users**.

**Rules:**

- Audit admin SQL by "total rows scanned per page load": `one page load = every endpoint × every query in it`. More than 100,000 rows per load requires remediation.
- **Never materialize display-only data in the main query.** Fields like "fingerprint" that exist only for display must not be fetched by joining a full-table CTE. Instead: fetch the current page only, then supplement by primary key (`user_id in (...current page...)`) — the deferred-join pattern. With pageSize ≤ 100 that is a few hundred rows per load.
- When count and list are separate queries, count may use only necessary filters; display joins never enter count.
- Down-rate count: cache the same-condition count for 1–5 minutes, or compute it on first load and reuse it across pages. Never recompute count on every page turn.
- An admin debugging at dozens of queries per minute is equivalent to a hot path. Admin query standards follow hot-path standards.

### 2. The constraint behind `on conflict` must actually exist

**Typical incident:** Code says `insert ... on conflict (a, b) do nothing`, but the production table lacks that unique constraint (the `primary key` in the DDL never took effect because the table already existed and `create table if not exists` skipped it) → upsert semantics silently fail, one row per request, growing to millions over a few months.

**Rules:**

- Before writing `on conflict`, check `sqlite_master` to confirm the conflict target's unique constraint/index really exists. Make this a required code-review item.
- `create table if not exists` **adds nothing** to an existing table. To add a constraint to an old database, use an explicit `create unique index if not exists` (in the always-run block, with a version bump).
- Deduplicate before creating a unique index, or the DDL fails. Standard dedupe: `delete from t where rowid not in (select max(rowid) from t group by unique_key_columns)`.

### 3. Write timestamps explicitly in INSERT; do not rely on a column default

**Typical incident:** The DDL declares `created_at TEXT default (datetime('now'))`, but the old table has no such default. INSERT does not pass `created_at` → new rows have NULL in that column. Combined with mixed time formats in one column (ISO and `Date.toString()`), `max(created_at)` compares strings and returns garbage ("W" > "2"), nearly leading to a false diagnosis of "data stalled for months" — the feature was working fine; only the time dimension was lost.

**Rules:**

- Every INSERT writes the time column explicitly (e.g. `new Date().toISOString()`). **Never rely on a DDL default** — old databases never receive a new default.
- Write time columns in ISO format consistently; forbid `Date.toString()`. Before doing min/max on a time column, verify the format is uniform (`group by substr(created_at,1,2)`).
- When diagnosing "stalled/pipeline broken" problems, suspect the integrity of the time column first (NULLs, mixed formats, string ordering), then the business logic.

### 4. Every-request insert tables must be capped

**Test:** If a table receives an INSERT on every request/login path and has no unique constraint, TTL, or cleanup, rows = requests × time. Unbounded by definition.

**Rules:** At launch, such tables need either (a) a unique-key constraint (idempotent dedupe) or (b) a daily snapshot cron that also cleans up (keep only the latest N rows / 30 days). At least one, as a launch checklist item. The cleanup query itself must use an index (index the WHERE columns) and run in batches (`delete ... where ... limit N` in a loop), or the cleanup becomes its own full-table scan.

### 5. Alert dedupe keys use the measured bucket, not wall-clock time

**Typical incident:** The usage alert's dedupe key is the current hour at send time, while the measurement window is "the previous full hour". After the top of the hour, the 07:00 send still carries 06:00 bucket data, so the same anomaly is announced again as new.

**Rules:** Dedupe key = the time bucket actually measured in the alert body (e.g. `alert_hour`), not the current hour derived from `Date.now()`. Every alert that reads a delayed window must align these two concepts.

### 6. Shortest path to a usage anomaly: query-level stats, not guesses

The shortest path from "alert received" to "culprit SQL" is two steps: database console → Metrics → the **per-query** view (sorted by runtime). The top entries are the culprits (calls, rows read, P50, time share are all visible), then `EXPLAIN QUERY PLAN` plus an `EXCEPT` diff verifies the fix. For a "rows read spiked" alert, step one is always the per-query view — do not start by assuming business traffic.

---

## 7. Pre-launch self-check (tick each before committing)

| # | Check | Section |
|---|--------|------|
| BP-01 | Scan volume estimated; the execution path matches its standard (hot path = indexed point lookup, single-digit rows) | 1 |
| BP-02 | Required indexes exist: in the always-run block, version bumped, placed after `CREATE TABLE` | 2 |
| BP-03 | No function-wrapped columns / expression joins / leading-wildcard `like` / `//` comments in SQL templates | 2 |
| BP-04 | No per-row queries or inserts in loops (N+1 merged into batches) | 2 |
| BP-05 | Cumulative stats use snapshots (tenant dimension + `as_of`); automatic fallback when not covered, and the fallback is observable | 3 |
| BP-06 | Time zone and day boundary are hard-coded | 3 |
| BP-07 | Cache has TTL + capacity cap; key includes user/tenant dimension; write paths bypass the cache | 4 |
| BP-08 | Front end is built before deploy; live artifact confirmed | 5 |
| BP-09 | Hourly alerts for rows read and rows written are configured; dedupe key = measured bucket | 5 / 6 |
| BP-10 | The unique constraint behind `on conflict` is confirmed in `sqlite_master`; dedupe ran before creating the unique index | 6 |
| BP-11 | INSERT writes timestamps explicitly; time columns use ISO format consistently | 6 |
| BP-12 | Every-request insert tables have a unique constraint or cleanup cron; cleanup is indexed and batched | 6 |
| BP-13 | Admin page load scans ≤ 100,000 rows; deep pagination uses keyset; count is down-rated | 6 |
