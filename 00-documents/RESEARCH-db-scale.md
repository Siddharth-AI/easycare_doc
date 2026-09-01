# Binary-Tree MLM Genealogy at 1–5 Lakh Members — Database Architecture Research

Scope: PostgreSQL-backed binary genealogy + volume/commission engine for 100,000–500,000 members.
Everything marked **[UNVERIFIED — engineering judgement]** is my own inference, not a cited claim.

---

## 1 Hierarchy storage

### 1.1 The four models against the three MLM access patterns

MLM genealogy has an access profile that is unusual and that changes the normal answer:

- **Placement is permanent.** A member is inserted at a leaf slot (or spillover slot) and essentially never moves. Re-parenting — the operation that destroys materialized path and nested set — barely exists.
- **Inserts are constant** (every signup) but low volume (500k inserts over the product's whole life).
- **Ancestor fetch is the hot path** — every purchase must walk up 30–40 levels.
- **Subtree/leg volume is enormous** — a root's left leg can be 250,000 nodes. It can never be computed live.

| Model | (a) Insert | (b) All ancestors of one node | (c) Subtree / leg volume | Verdict |
|---|---|---|---|---|
| Adjacency list (`parent_id`) + `WITH RECURSIVE` | O(1), 1 row | O(depth) serial index lookups (~40) | O(subtree) — walks 250k rows. Unusable live | Keep as source of truth |
| Closure table | O(depth) inserts (~41 rows) | **1 index range scan**, 41 rows | Index scan on `ancestor_id`, but still returns 250k rows | Best read model; still needs counters for (c) |
| Materialized path (`ltree` / `bigint[]`) | O(1) — path is written once and never rewritten (placement is immutable) | **0 joins** — ancestors are already in the row | GiST `<@` prefix match; still returns 250k rows | Co-winner, cheapest of all here |
| Nested set (`lft`/`rgt`) | **O(N)** — one signup rewrites lft/rgt across a large fraction of the tree | O(1)-ish | Very fast range query | **Disqualified.** Nested sets are "excellent for read-heavy, stable hierarchies; complex for updates" ([SQL Antipatterns / tree model comparison](https://www.educative.io/courses/sql-antipatterns-database-programming/comparison-of-different-tree-implementations)); an MLM tree is the opposite of stable |

Bill Karwin's *SQL Antipatterns* names naive adjacency-list-only trees an antipattern and presents the closure table as "generally the simplest for all operations", with the tree maintenance delegable to a trigger ([Karwin, Rendering Trees with Closure Tables](https://karwin.com/blog/index.php/2010/03/24/rendering-trees-with-closure-tables/)).

### 1.2 PostgreSQL `WITH RECURSIVE` performance notes (from the docs)

Per [PostgreSQL 18 docs §7.8](https://www.postgresql.org/docs/current/queries-with.html):

- "While `RECURSIVE` allows queries to be specified recursively, **internally such queries are evaluated iteratively**." Each level is a separate execution of the recursive term against a working table. There is **no pipelining and no parallelism across iterations** — depth 40 means 40 sequential round trips through the executor.
- Inlining/folding (`NOT MATERIALIZED`) is documented only for CTEs that are "non-recursive and side-effect-free". A recursive CTE is therefore effectively an **optimization fence**: it is materialized, and the planner joins the materialized result to the rest of your query with weak cardinality knowledge. **[UNVERIFIED — engineering judgement]** this is the practical reason recursive-CTE ancestor lookups produce bad plans once you join them to `orders`/`ledger`.
- The docs warn explicitly about non-terminating recursion — with MLM data corruption (an accidental cycle in `parent_id`) this is a live risk. Use `UNION` (not `UNION ALL`) or PG14+ `CYCLE` detection, plus a `CHECK (depth < 200)` guard.

**When does a closure table beat the recursive CTE?**

- **Ancestor fetch (b):** honestly, both are fine. 40 PK lookups is ~40 × 5–20 µs ≈ **0.2–1 ms** warm. The closure table turns it into one range scan, ~**50–150 µs**. A 3–10× win, on a query that runs once per purchase. Worth it, not dramatic.
- **The real win is anything set-based over many nodes at once**: the nightly close, "expand these 1,000 purchases to their ancestors", "which uplines qualify for rank X". A closure table lets you write those as a single `JOIN` that the planner can hash/merge and parallelize. A recursive CTE cannot be parallelized and must be re-run per node.
- Cybertec's write-up on speeding up recursive queries is the standard reference here, though the page returned HTTP 403 to automated fetch: <https://www.cybertec-postgresql.com/en/postgresql-speeding-up-recursive-queries-and-hierarchic-data/> **[UNVERIFIED — could not fetch]**.

### 1.3 Closure-table row-count math at 5 lakh users

Closure rows = Σ over all nodes of (that node's depth + 1) — i.e. every node stores one row per ancestor including itself.

**Case A — perfectly balanced binary tree.** N = 2¹⁹ − 1 = **524,287** nodes, max depth 18.
Σ_{d=0}^{18} 2^d·(d+1) = 18·2¹⁹ + 1 = **9,437,185 rows** (avg path length 18.0).

**Case B — realistic spillover tree, average depth 40** (the number in the brief; max depth will be 80–150 because spillover creates long thin chains).
Rows ≈ 500,000 × 41 = **20,500,000 rows**.

**Case C — max depth 40, average depth ~25.** 500,000 × 26 = **13,000,000 rows**.

**Storage for Case B (20.5 M rows)**, schema `(ancestor_id bigint, descendant_id bigint, depth smallint)`:

| Component | Per row | Total |
|---|---|---|
| Heap tuple (24 B header + 24 B aligned payload) + 4 B line pointer | ~52 B | **~1.07 GB** |
| B-tree on `(descendant_id, ancestor_id)` — the ancestor lookup | ~31 B | **~0.64 GB** |
| B-tree on `(ancestor_id, descendant_id)` — the subtree lookup | ~31 B | **~0.64 GB** |
| **Total** | | **≈ 2.3–2.5 GB** |

That fits comfortably in the buffer cache of a 32 GB machine. **A closure table at 5 lakh × depth 40 is a non-problem.** The often-quoted O(N²) worst case only bites for wide, shallow, or arbitrary DAG structures — a binary MLM tree is O(N · avg_depth).

**Insert cost per signup:** 41 heap rows + 82 index entries. Sub-millisecond. Maintain it in the same transaction as the member insert:

```sql
INSERT INTO member_closure (ancestor_id, descendant_id, depth)
SELECT c.ancestor_id, :new_id, c.depth + 1
FROM   member_closure c WHERE c.descendant_id = :parent_id
UNION ALL SELECT :new_id, :new_id, 0;
```

### 1.4 Materialized path as the cheap co-winner

Because placement is immutable, store the full upline on the member row itself: `upline_path bigint[]` (or `ltree`). Ancestors then cost **zero joins**. `ltree` limits per the [ltree docs](https://www.postgresql.org/docs/current/ltree.html): max 1000 chars per label, **65,535 labels per path** (depth 40 is nowhere near), GiST `gist_ltree_ops` supports `@>` / `<@` for ancestor/descendant. For `bigint[]`, a GIN index gives `@>` containment for "everyone under X".

---

## 2 Running totals

### 2.1 Why production systems denormalise

Recomputing a leg's volume means aggregating a subtree. At 5 lakh members the root's weak leg is on the order of 250,000 members × their monthly purchases. Even with a perfect closure-table index that is a **250k-row aggregate on every page load and every commission calculation** — 100–500 ms warm, worse cold, and it scales linearly with the business. It is not viable at any depth for the top 100 members. **[UNVERIFIED — engineering judgement]** but the arithmetic is not in dispute.

Every real system therefore keeps per-member, per-leg counters. ByDesign confirms the business reason: binary plans pay "commission from only the lower earning leg of their downline, which is called the 'pay leg'" ([ByDesign, MLM Genealogy Trees](https://bydesign.com/mlm-genealogy-trees-and-how-they-grow/)) — so you need *both* leg totals, live, for every member, on every purchase. Exigo advertises pushing "volume updates in near real-time in just 1 to 5 minutes" ([exigo.com](https://www.exigo.com/)) — note that is **minutes, not milliseconds**: even a market leader treats volume propagation as an asynchronous, batched pipeline, not a synchronous write.

### 2.2 Suggested counter table

```sql
CREATE TABLE member_leg_volume (
  member_id     bigint  NOT NULL,
  period        date    NOT NULL,        -- month or day bucket
  left_pv       bigint  NOT NULL DEFAULT 0,   -- integer PV, see §6
  right_pv      bigint  NOT NULL DEFAULT 0,
  left_carry    bigint  NOT NULL DEFAULT 0,   -- carry-forward after matching
  right_carry   bigint  NOT NULL DEFAULT 0,
  matched_pv    bigint  NOT NULL DEFAULT 0,
  updated_at    timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (member_id, period)
) WITH (fillfactor = 85);
```

Two rules that matter:

1. **`fillfactor = 85`.** Adyen measured, on a real production partition, HOT-update rate rising from **63.9% → 92.2%**, peak dead rows falling ~70M → ~40M, table size 30 GB → 24 GB, and cluster WAL 3.15 TB/day → 2.80 TB/day (**~10% WAL reduction, ~20% smaller table**) ([Adyen, Fighting PostgreSQL write amplification with HOT updates](https://www.adyen.com/knowledge-hub/postgresql-hot-updates)).
2. **Never index `left_pv` / `right_pv` / any counter column.** Adyen: the HOT optimisation "does not apply when updating a column that is covered by an index." One index on a counter column turns each counter bump from 1 page write into 2+.

### 2.3 Keeping counters correct

The counters are a **cache**, and must be provably reconstructible:

- **Source of truth is an append-only fact table**, not the counter. Every purchase writes an immutable `pv_event(order_id, member_id, pv, period, created_at)` row. The counter is a fold over those events.
- **Idempotent increment.** Every increment is tied to a unique `(source_type, source_id, ancestor_id)` key recorded in an `applied_credits` table with a `UNIQUE` constraint. The worker does `INSERT ... ON CONFLICT DO NOTHING` on that key first; only rows that actually inserted contribute to the batched `UPDATE`. A retry after a crash inserts zero rows and therefore adds zero volume.
- **Transactional increment.** The `applied_credits` insert and the counter `UPDATE` must be in the *same* transaction. Never "update counter, then mark applied" across two transactions.
- **Nightly reconciliation.** A read-replica job recomputes `SUM(pv_event)` per member per leg via the closure table and diffs it against `member_leg_volume`. Alert on any non-zero diff. This is the same discipline as Uber's LedgerStore validation jobs, whose largest offline comparison covered **760 billion records / 70 TB compressed** ([Uber, How LedgerStore Supports Trillions of Indexes](https://www.uber.com/blog/how-ledgerstore-supports-trillions-of-indexes/)).

---

## 3 Fan-out writes

### 3.1 The shape of the problem

This is exactly the newsfeed **fan-out-on-write vs fan-out-on-read** trade-off, with one brutal difference: in a social graph only celebrities have huge fan-in, but **in an MLM tree the root and the top ~50 members are on the ancestor path of literally every purchase in the company**. The "celebrity problem" is structural and permanent ([fan-out trade-off summary](https://www.systemoverflow.com/learn/design-fundamentals/back-of-envelope/fan-out-calculations-and-write-amplification-trade-offs), [GetStream, How to Fan-Out Activities to Millions of Followers](https://getstream.io/blog/fan-out-activities-followers/)).

Concretely, at 5 lakh members with 30% monthly activity:

- 150,000 purchases/month × avg depth 40 = **6.0 M ancestor credits/month**
- Averaged: 0.06 purchases/s. But MLM traffic is spiky — assume 30% of the month's volume in the last 48 hours: 45,000 purchases / 172,800 s ≈ 0.26/s average, **peak 10–30 purchases/s**.
- At 20 purchases/s synchronous: **800 row UPDATEs/s** — Postgres handles the throughput fine.
- **But the root row gets 20 serialised `UPDATE`s per second.** "A single row that becomes a contention hotspot requires all transactions updating it to need a row lock, and Postgres queues them" ([Postgres lock-contention patterns](https://oneuptime.com/blog/post/2026-02-02-postgresql-lock-contention/view)). Each purchase transaction now blocks on the root for the duration of every other purchase transaction. **This, not raw throughput, is what kills the synchronous design.**

### 3.2 Synchronous vs queued — recommendation

**Do not fan out synchronously.** Inside the purchase transaction do only:

1. `INSERT INTO orders …`
2. `INSERT INTO pv_event …` (1 row, the immutable fact)
3. `INSERT INTO outbox (aggregate_id, event_type, payload, created_at)` — 1 row

Three inserts, zero contended updates, commits in ~1 ms. Then an async worker does the fan-out.

### 3.3 The aggregation trick that removes the write amplification

The naive async worker still does 40 updates per purchase. Instead, **expand a whole batch, then group by ancestor before writing**:

```sql
WITH batch AS (
  SELECT o.id AS outbox_id, e.member_id, e.pv, e.period
  FROM   outbox o JOIN pv_event e ON e.id = (o.payload->>'pv_event_id')::bigint
  WHERE  o.published_at IS NULL
  ORDER  BY o.id
  LIMIT  1000
  FOR UPDATE SKIP LOCKED
),
expanded AS (                              -- closure table does the fan-out
  SELECT c.ancestor_id, b.period,
         SUM(b.pv) FILTER (WHERE m.leg = 'L') AS left_pv,
         SUM(b.pv) FILTER (WHERE m.leg = 'R') AS right_pv
  FROM   batch b
  JOIN   member_closure c ON c.descendant_id = b.member_id AND c.depth > 0
  JOIN   member_leg_side m ON m.ancestor_id = c.ancestor_id AND m.descendant_id = b.member_id
  GROUP  BY c.ancestor_id, b.period
)
UPDATE member_leg_volume v
SET    left_pv  = v.left_pv  + e.left_pv,
       right_pv = v.right_pv + e.right_pv
FROM   expanded e
WHERE  v.member_id = e.ancestor_id AND v.period = e.period;
```

**Why this wins:** 1,000 purchases × 40 ancestors = 40,000 (ancestor, pv) pairs, but the upper levels of the tree are shared by everybody, so `GROUP BY ancestor_id` collapses those to roughly **8,000–15,000 distinct rows [UNVERIFIED — engineering judgement, depends on tree shape]**. Write amplification drops from ~40× to ~10×, and — the important part — **the root row is updated once per batch instead of 1,000 times.** The hot-row contention disappears by construction.

Use `UPDATE … FROM unnest($1::bigint[], $2::bigint[])` when driving the batch from application code: it uses a fixed 2 parameters regardless of batch size, dodging PostgreSQL's **32,767 parameter limit**, and TigerData measured `INSERT … UNNEST` at **2.13× faster than `INSERT … VALUES` at batch size 1000**, and **5.02× faster** on a 10-column schema, with the saving coming from planning time ([TigerData, Boosting Postgres INSERT Performance by 2x With UNNEST](https://www.tigerdata.com/blog/boosting-postgres-insert-performance)).

Note the honest caveat: a job queue "does not raise the database's sustained write throughput, since the background worker still writes back to the same PostgreSQL instance" ([PostgreSQL write performance](https://dev.to/haikasatryan/postgresql-write-performance-what-the-benchmarks-wont-tell-you-mm7)). The win here is **latency isolation + batch aggregation + spike absorption**, not magic extra capacity.

### 3.4 Transactional outbox

The dual-write problem — update the DB *and* enqueue a job, atomically — is solved by the [transactional outbox pattern](https://microservices.io/patterns/data/transactional-outbox.html) ([AWS Prescriptive Guidance version](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html)). The order and the outbox row commit in one local transaction; a relay/poller publishes them.

Critical property to design around: **"The outbox guarantees at-least-once delivery, not exactly-once. If the relay publishes an event but crashes before marking it delivered, it will republish on restart. Consumers must be idempotent."** ([Transactional Outbox: trade-offs](https://www.softwarecraftsperson.com/posts/2025-10-08-transactional-outbox-pattern/))

Outbox table hygiene:
- Partial index: `CREATE INDEX ON outbox (id) WHERE published_at IS NULL;` — keeps the poller's index tiny as the table grows.
- `FOR UPDATE SKIP LOCKED` for multi-worker consumption without contention.
- Delete/archive published rows on a schedule, or partition the outbox by day and drop partitions.

### 3.5 Idempotency keys

Follow Stripe's contract ([Stripe API — Idempotent requests](https://docs.stripe.com/api/idempotent_requests)):

- Client generates the key; Stripe "suggest[s] using V4 UUIDs, or another random string with enough entropy to avoid collisions", up to 255 chars.
- Stripe "works by **saving the resulting status code and body of the first request** made for any given idempotency key, regardless of whether it succeeds or fails. Subsequent requests with the same key return the same result, including 500 errors."
- "The idempotency layer **compares incoming parameters to those of the original request and errors if they're not the same**" — catches client bugs that reuse a key for different money.
- Keys are pruned after ≥24 hours. **[UNVERIFIED — engineering judgement]** for MLM, retain commission idempotency keys for the life of the payout period (e.g. 90 days), not 24 h, because reruns of a failed close can happen days later.

For internal fan-out, the idempotency key is the natural composite: `UNIQUE (source_type, source_id, ancestor_id, period)` in `applied_credits`. Retries then cost an index probe and credit nothing.

---

## 4 Closing & ledger

### 4.1 Why balances are never mutated in place

The consensus of the fintech ledger literature ([Modern Treasury, Enforcing Immutability in your Double-Entry Ledger](https://www.moderntreasury.com/journal/enforcing-immutability-in-your-double-entry-ledger); [Part V: Immutability and Double-Entry](https://www.moderntreasury.com/journal/how-to-scale-a-ledger-part-v)):

- The ledger is "an **immutable, append-only log**" underneath any mutable surface fields. Balances are *derived* by folding entries, never overwritten.
- Overwriting "destroys the audit trail; if a balance is calculated incorrectly, there is no way to reconstruct the sequence of events that led to the error."
- **Corrections are contra entries, not edits.** Modern Treasury uses a `discarded_at` field rather than deletion, and filters historical balances with `WHERE account_id = ? AND effective_at <= ? AND (discarded_at IS NULL OR discarded_at >= ?)`.
- **Versioning beats timestamps at scale**: "Account versions that are persisted on Entries allow us to query exactly which Entries correspond to a given Account balance," because many entries share the same `effective_at`.

Double-entry itself is the invariant that makes it provable: TigerBeetle argues "debit/credit is minimal and complete: two entities (accounts, transfers) and one invariant (every debit has an equal and opposite credit) model any exchange of value, in any domain" ([TigerBeetle — Debit/Credit: The Schema for OLTP](https://docs.tigerbeetle.com/concepts/debit-credit/)). Enforce it at the DB level: every `ledger_transaction` must have ≥2 entries summing to zero **per currency** (Modern Treasury is explicit that you cannot balance across currencies via FX rates).

### 4.2 MLM ledger schema sketch

```sql
-- immutable; no UPDATE, no DELETE, revoke them from the app role
CREATE TABLE ledger_entry (
  id             bigserial,
  txn_id         uuid        NOT NULL,     -- groups the debit+credit
  account_id     bigint      NOT NULL,     -- (member_id, account_type) resolved
  direction      char(1)     NOT NULL CHECK (direction IN ('D','C')),
  amount_minor   bigint      NOT NULL CHECK (amount_minor > 0),
  currency       char(3)     NOT NULL,
  reason_code    text        NOT NULL,     -- MATCHING, MENTOR, CAPPING_WASHOUT, ...
  closing_run_id bigint,                   -- null for real-time events
  idem_key       text        NOT NULL,
  effective_at   timestamptz NOT NULL,
  created_at     timestamptz NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE UNIQUE INDEX ON ledger_entry (idem_key, created_at);   -- see §5.1 gotcha
```

Balances live in a separate `account_balance` cache row carrying `as_of_entry_id`, so any balance is verifiable as `SUM(entries WHERE id <= as_of_entry_id)`.

### 4.3 Restartable period close

The Spring Batch / chunk-and-checkpoint model is the well-trodden design ([restartable batch design](https://oneuptime.com/blog/post/2026-01-26-spring-batch-jobs/view); [chunked idempotent payouts in Oracle](https://tech-champion.com/database/oracle/solving-non-idempotent-batch-procedures-in-oracle-preventing-duplicate-payouts/)): "Jobs may restart from a failed chunk, and your processor should produce the same output for the same input"; "each committed chunk moves the 'high-water mark' of processed data forward, reducing the work for subsequent retries." The named risk is precisely ours: "When a simple `INSERT INTO SELECT` statement lacks a deduplication guard, retrying a failed job can lead to **catastrophic duplicate payouts**."

Concrete design for a daily binary close over 5 lakh members:

```
closing_run(id, period_date, status, started_at, finished_at, config_hash)
closing_run_chunk(run_id, chunk_no, member_id_lo, member_id_hi, status, entries_written)
```

- **Chunk size 5,000 members → 100 chunks.** Each chunk is one transaction: read leg volumes → compute matched PV → apply capping table → write ledger entries → write carry-forward → mark chunk `DONE`.
- **Idempotency at the chunk boundary:** `UNIQUE (closing_run_id, account_id, reason_code)` on ledger entries. A re-run of chunk 47 after a crash hits the constraint and writes nothing.
- **Parallelism:** 8 workers pulling chunks with `FOR UPDATE SKIP LOCKED`. **[UNVERIFIED — engineering judgement]** at ~2 ms of work per member, 500k members is ~17 min single-threaded, **~2–3 min at 8-way parallelism**. Exigo's public claim — "averages complex global commission runs in **under 10 minutes**" — is the right order-of-magnitude target ([exigo.com](https://www.exigo.com/)).
- **Freeze the inputs.** Record `config_hash` (the commission plan version, capping table, rates) on the run. A re-run with a different plan version must be a *new* run, not a resume.
- **Never mutate on rerun.** Reversal of a bad run = a new run of contra entries referencing the original `closing_run_id`, per §4.1.
- **Washout / carry-forward** are just more ledger entries with `reason_code = 'CAPPING_WASHOUT'` / `'CARRY_FORWARD'`, so the member's statement explains where every point went.

---

## 5 PostgreSQL scaling

### 5.1 Monthly RANGE partitioning — and the one gotcha that will bite you

From [PostgreSQL 18 docs §5.12 Table Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html):

- "The query planner is generally able to handle partition hierarchies with **up to a few thousand partitions** fairly well, provided that typical queries allow the query planner to prune all but a small number of partitions." Monthly partitions for 10 years = 120. Fine.
- "Never just assume that more partitions are better than fewer partitions, nor vice-versa." Planning time and memory grow with the number of *unpruned* partitions; OLTP workloads should prefer fewer.
- **The gotcha:** "To create a unique or primary key constraint on a partitioned table, … the constraint's columns must **include all of the partition key columns**." So your idempotency `UNIQUE (idem_key)` becomes `UNIQUE (idem_key, created_at)` — which **no longer enforces global uniqueness of the key**. Two fixes: (i) keep `idempotency_key` in a small **non-partitioned** table that the transaction inserts into first, or (ii) make the partition key derivable from the key itself. Choose (i).
- Indexes: `CREATE INDEX` on the parent auto-creates matching per-partition indexes, but **`CONCURRENTLY` is not allowed on a partitioned table** — use the documented `CREATE INDEX CONCURRENTLY` per partition + `ALTER INDEX … ATTACH PARTITION` dance.
- Dropping a month is `DETACH`/`DROP` — "Instead of deleting old rows one by one, DBAs can simply drop old partitions."
- Automate with `pg_partman` rather than hand-rolled DDL cron.

### 5.2 Index choices

| Table | Index | Why |
|---|---|---|
| `member` | `(parent_id, leg)` unique | enforces binary: at most one L and one R child |
| `member` | GIN on `upline_path bigint[]` | "all descendants of X" without a join |
| `member_closure` | PK `(descendant_id, ancestor_id) INCLUDE (depth)` | ancestor fetch, index-only scan |
| `member_closure` | `(ancestor_id, descendant_id)` | subtree/leg enumeration, batch expansion joins |
| `ledger_entry` | `(account_id, created_at DESC)` | member statement, latest-first |
| `outbox` | partial `(id) WHERE published_at IS NULL` | poller stays O(backlog), not O(history) |
| `member_leg_volume` | PK only — **no index on counter columns** | preserves HOT updates (Adyen) |

### 5.3 PgBouncer

Run **transaction pooling**: "A server connection is assigned to a client only during a transaction. When PgBouncer notices that the transaction is over, the server will be put back into the pool." Per [pgbouncer.org/features](https://www.pgbouncer.org/features.html), transaction mode **breaks**: `SET`/`RESET`, `LISTEN`, `WITH HOLD CURSOR`, `PREPARE`/`DEALLOCATE`, `PRESERVE/DELETE ROWS` temp tables, `LOAD`, and **session-level advisory locks**. The docs are blunt: "transaction pooling breaks client expectations of the server *by design*."

Two consequences for this system:
- **Do not use session-level advisory locks to serialise the closing job.** Use a row in a `job_lock` table with `SELECT … FOR UPDATE NOWAIT`, or transaction-scoped `pg_advisory_xact_lock` (which *is* safe in transaction mode).
- Prisma/node-postgres prepared-statement caching must be disabled or the driver put in a compatible mode.

Sizing: each PostgreSQL backend costs ~10 MB and a process; 500 direct connections ≈ 5 GB of RAM purely on overhead. Typical production PgBouncer config is `max_client_conn = 10000` with a small `reserve_pool_size` of 5–10, and a server-side pool sized in the low tens ([PgBouncer sizing guidance](https://oneuptime.com/blog/post/2026-01-26-pgbouncer-connection-pooling/view), [PlanetScale, Scaling Postgres connections with PgBouncer](https://planetscale.com/blog/scaling-postgres-connections-with-pgbouncer)). **[UNVERIFIED — engineering judgement]** for 5 lakh members: `default_pool_size` 40–60 for the app pool, a separate pool of 8–12 for the closing workers so a runaway close cannot starve the storefront.

### 5.4 Read replicas for reporting

Genealogy tree browsing, downline reports, and rank dashboards are exactly the "BI and analytical workloads use the read replica as the data source" case. Practical settings, per the cloud-provider guidance:

- `hot_standby = on`; alert when `pg_stat_replication.replay_lag` exceeds ~30 s for 2 minutes.
- `max_standby_streaming_delay` default 30 s; raise to 60–120 s if genealogy reports run long — otherwise the replica cancels them.
- `hot_standby_feedback = on` avoids query cancellation but **causes bloat on the primary** by deferring vacuum. Given that `member_leg_volume` is your hottest-updated table, turning this on is dangerous here. **[UNVERIFIED — engineering judgement]** prefer a higher `max_standby_streaming_delay` over `hot_standby_feedback`.
- Never read leg volumes for a *payout decision* from a replica. Money reads go to the primary.

### 5.5 Storage math

Assumptions: 500,000 members, 30% purchase per month = 150,000 purchases/month, avg upline depth 40.

| Object | Rows/month | Bytes/row (heap+idx) | Size/month | Size/year |
|---|---|---|---|---|
| `orders` | 150 k | ~400 | 60 MB | 0.7 GB |
| `pv_event` (1/purchase) | 150 k | ~150 | 23 MB | 0.3 GB |
| `applied_credits` (fan-out audit, 40×) | **6.0 M** | ~110 | **660 MB** | **7.9 GB** |
| `ledger_entry` (double-entry, ~4 entries/member/close × 30 days, plus commissions) | ~3–6 M | ~250 | 0.75–1.5 GB | 9–18 GB |
| `outbox` (deleted after publish) | 150 k transient | — | <50 MB | — |
| **Static:** `member_closure` (one-time, grows with signups) | 20.5 M total | ~115 | — | **~2.4 GB total** |

**Total ≈ 1.5–2.2 GB/month, 18–27 GB/year of hot transactional data**, plus a ~2.4 GB closure table. With monthly partitions and detaching anything older than 24 months to cold storage, the working set stays under ~50 GB — a single well-provisioned Postgres primary (16 vCPU / 64 GB) handles this with a lot of headroom. **The scale problem here is contention and batch-job design, not data volume.**

The biggest single line item is the fan-out audit table at 6 M rows/month. That is the price of provable idempotency. Partition it monthly and drop after the dispute window (e.g. 12 months).

---

## 6 Money precision

### 6.1 NUMERIC vs float vs integer

From the [PostgreSQL docs §8.1](https://www.postgresql.org/docs/current/datatype-numeric.html):

- Floats: "If you require exact storage and calculations (such as for monetary amounts), **use the `numeric` type instead**." `real`/`double precision` are "inexact, variable-precision".
- But also: "**Calculations with `numeric` values are very slow** compared to the integer types, or to the floating-point types."
- `numeric` storage: "two bytes for each group of four decimal digits, plus three to eight bytes overhead" — so a typical money value is 8–12 bytes vs `bigint`'s fixed 8.

Modern Treasury's position is the sharper one: use **64-bit integers in minor units**. $12.34 stores as `1234`. Their stated range is −92,233,720,368,547,758.08 to +92,233,720,368,547,758.07 — "over 92 quadrillion dollars… exceeds global GDP by 800×". Floats fail concretely: "A 32-bit float has to approximate $25,474,937.47 as $25,474,936.32 — **off by $1.15**", and different languages default to different rounding modes (Ruby half-away-from-zero, Python 3 half-even, JavaScript half-toward-+∞) creating cross-system disagreement ([Modern Treasury, Floats Don't Work For Storing Cents](https://www.moderntreasury.com/journal/floats-dont-work-for-storing-cents)). They pair each integer with an ISO 4217 currency code to derive the decimal places.

### 6.2 Recommendation for this system

| Quantity | Type | Rationale |
|---|---|---|
| Money (wallet, commission, payout) | `bigint` **paise** (INR minor unit, 10⁻²) | exact, 8 bytes, fast to `SUM()` over millions of ledger rows; matches Stripe/Modern Treasury practice |
| PV / BV / GV points | `bigint` scaled ×1000 (i.e. milli-PV) | points frequently carry 2–3 decimals in plan documents; scaling to integers keeps matching/carry-forward arithmetic exact |
| Percentage rates (80%, 7%) | `integer` **basis points** (8000, 700) or `numeric(9,6)` in a config table | never store as float; basis points let the whole split be integer arithmetic |
| Display / API | format at the edge from the integer + currency exponent | one formatting layer, no drift |

Use `numeric(18,4)` only where a human reads the raw column and arithmetic is rare (e.g. an FX rate table). **[UNVERIFIED — engineering judgement]** for a plan with percentage splits recomputed over 500k accounts nightly, integers are meaningfully faster and, more importantly, remove an entire class of "the sum of the splits is 1 paisa off" bugs.

### 6.3 Rounding rules for percentage splits

The failure mode: 80% + 7% + 13% of a pool, each rounded independently, does not sum to the pool. Money is created or destroyed.

**Use the largest-remainder method.** "Standard accounting practice dictates allocating the base divided amount to all parties, then distributing the remainder penny by penny until the remainder is zero. This ensures no money is ever created or destroyed by rounding." The method "gives each recipient the floor of their share in cents, then assigns one extra cent to recipients ordered by the size of their fractional remainder (largest first) until the total is fully allocated" — it is "the standard in U.S. payroll systems and stock dividend distributions" ([summary](https://cardinalby.github.io/blog/post/best-practices/storing-currency-values-data-types/), [largest remainder method](https://github.com/PHP-algo/largest-remainder-method)).

Worked example. Matched value = ₹1,234.56 = **123,456 paise**. Splits: matching 80%, mentor 7%, company 13%.

```
floor(123456 * 8000 / 10000) =  98,764   remainder .80
floor(123456 *  700 / 10000) =   8,641   remainder .92
floor(123456 * 1300 / 10000) =  16,049   remainder .28
                    subtotal = 123,454   → 2 paise unallocated
allocate to largest remainders: mentor (.92) +1 → 8,642
                                matching (.80) +1 → 98,765
                    total     = 123,456  ✔ exactly conserved
```

Hard rules to encode:
1. **Compute every split from the same integer pool**, in one function, in one transaction. Never `round(x*0.80)` in one service and `round(x*0.07)` in another.
2. **Assert `SUM(splits) == pool`** before writing the ledger; fail the transaction otherwise. This is a cheap invariant that catches plan-configuration mistakes.
3. **Fix a deterministic tie-break** (e.g. by `reason_code` alphabetically) so equal remainders allocate identically on every re-run — otherwise your "idempotent" close produces different output on retry.
4. **Capping is applied before splitting**, and the capped-off amount must be written as an explicit `CAPPING_WASHOUT` entry, not silently dropped — otherwise the ledger does not balance.

---

## 7 References

Real-world / primary sources actually used:

1. **Uber — LedgerStore.** Immutable, append-only ledger storage: "over 2 trillion unique indexes, and not a single data inconsistency has been detected"; petabyte-scale index footprint; largest offline validation job compared **760 billion records / 70 TB compressed** using Apache Spark. <https://www.uber.com/blog/how-ledgerstore-supports-trillions-of-indexes/> and <https://www.uber.com/us/en/blog/migrating-from-dynamodb-to-ledgerstore/>

2. **Adyen — PostgreSQL write amplification / HOT updates** (Feb 2022, production numbers). fillfactor 85 vs 100: HOT rate 63.9% → 92.2%, peak dead rows ~70 M → ~40 M, table 30 GB → 24 GB, cluster WAL 3.15 TB/day → 2.80 TB/day. <https://www.adyen.com/knowledge-hub/postgresql-hot-updates>

3. **Modern Treasury — "How to Scale a Ledger" series + immutability essays.** The reference articulation of append-only double-entry ledgers, entry versioning, `discarded_at` reversals, and 64-bit-integer money. Parts I–V: <https://www.moderntreasury.com/journal/how-to-scale-a-ledger-part-i>, <https://www.moderntreasury.com/journal/how-to-scale-a-ledger-part-v>, <https://www.moderntreasury.com/journal/floats-dont-work-for-storing-cents>

4. **TigerBeetle — Debit/Credit: The Schema for OLTP.** Argues double-entry as the minimal complete OLTP financial schema; two-phase (pending/post/void) transfers; notes Uber ran "a 2-year, 40-engineer effort to migrate their collection and disbursement payment platform to one based on the principles of double-entry accounting". <https://docs.tigerbeetle.com/concepts/debit-credit/>

5. **Exigo (enterprise direct-selling platform)** — vendor claims, useful as a target benchmark: volume updates pushed "in near real-time in just 1 to 5 minutes"; "averages complex global commission runs in under 10 minutes"; "$12k+ orders processed per minute for a single customer"; "millions of distributors supported globally". <https://www.exigo.com/> *(vendor marketing — treat as directional, not measured)*

6. **Epixel MLM Software** — stack disclosed as Python/Django, Golang, Node.js with an auto-scaling architecture and a dedicated commission engine. <https://www.epixelmlmsoftware.com/> *(vendor marketing)*

7. **ByDesign Technologies — MLM Genealogy Trees and How They Grow.** Business mechanics of the binary "pay leg" (lesser leg) vs "reference leg" — the reason both leg totals must be maintained. <https://bydesign.com/mlm-genealogy-trees-and-how-they-grow/>

8. **Vendor benchmark claims [UNVERIFIED — marketing, no methodology published]:** global-mlm.com claims 2M+ users on one platform with genealogy rendering, and a simulated 5-million-node dataset holding **p95 < 400 ms on full-tree traversal**, node search among 5 M records **< 50 ms**, top-5-levels subtree load of a 50,000-member downline **< 800 ms**, and binary volume recalculation on placement **< 100 ms**. <https://global-mlm.com/genealogy-tree.html>, <https://www.hybridmlm.io/blogs/genealogy-tree-in-mlm-visualizing-growth-and-downline-potential/>

Technical/primary docs: [PostgreSQL Table Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html) · [PostgreSQL WITH Queries](https://www.postgresql.org/docs/current/queries-with.html) · [PostgreSQL Numeric Types](https://www.postgresql.org/docs/current/datatype-numeric.html) · [PostgreSQL ltree](https://www.postgresql.org/docs/current/ltree.html) · [PgBouncer Features](https://www.pgbouncer.org/features.html) · [Stripe Idempotent Requests](https://docs.stripe.com/api/idempotent_requests) · [microservices.io Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html) · [AWS Transactional Outbox](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html) · [Karwin, Closure Tables](https://karwin.com/blog/index.php/2010/03/24/rendering-trees-with-closure-tables/) · [TigerData UNNEST benchmarks](https://www.tigerdata.com/blog/boosting-postgres-insert-performance)

---

## 8 Final recommendation

**Storage model — use all three, they are cheap:**
- `member(id, parent_id, leg, sponsor_id, depth, upline_path bigint[])` as the source of truth. Adjacency list for writes, `upline_path` for zero-join ancestor reads.
- `member_closure(ancestor_id, descendant_id, depth)` maintained in the signup transaction. **~20.5 M rows / ~2.4 GB at 5 lakh × depth 40** — trivially affordable, and it is what makes set-based batch jobs possible.
- **No nested sets.** O(N) rewrite per signup is disqualifying.

**Volume propagation — never synchronous:**
- Purchase transaction writes 3 rows: `order`, `pv_event`, `outbox`. Commits in ~1 ms with zero contended updates.
- An async worker pulls batches of ~1,000 outbox rows with `FOR UPDATE SKIP LOCKED`, expands them via the closure table, **aggregates by ancestor before writing**, and applies one `UPDATE … FROM unnest(...)`. This drops write amplification ~40× → ~10× and, critically, updates the root row **once per batch** instead of once per purchase — the hot-row contention disappears.
- `member_leg_volume` at `fillfactor = 85`, with **no index on any counter column**, to preserve HOT updates.
- Idempotency via `UNIQUE (source_type, source_id, ancestor_id, period)` in `applied_credits`, inserted in the *same* transaction as the counter update.

**Money movement — append-only double-entry:**
- `ledger_entry` is immutable (revoke `UPDATE`/`DELETE` from the app role), monthly `RANGE`-partitioned on `created_at`, with balances as a derived cache carrying `as_of_entry_id`.
- Corrections are contra entries referencing the original. Capping washout and carry-forward are explicit entries, never silent drops.
- Keep the global idempotency-key uniqueness in a **separate non-partitioned** table — a `UNIQUE` on a partitioned table must include the partition key and therefore cannot enforce global uniqueness.

**Closing job — chunked, parallel, restartable:**
- `closing_run` + `closing_run_chunk`, 5,000 members per chunk (100 chunks), one transaction per chunk, 8 workers via `SKIP LOCKED`, `UNIQUE (closing_run_id, account_id, reason_code)` making re-runs a no-op. Freeze the plan version as `config_hash` on the run. Target **< 10 minutes** for a full close, matching the industry benchmark.

**Money precision:**
- `bigint` paise for money, `bigint` milli-PV for points, basis points for rates. Largest-remainder allocation for every split, with a deterministic tie-break and a `SUM(splits) == pool` assertion before writing.

**Infrastructure:**
- Single Postgres primary (16 vCPU / 64 GB is ample — the whole hot working set is under ~50 GB), PgBouncer in transaction mode with a separate pool for closing workers, one read replica for genealogy browsing and reports with `max_standby_streaming_delay` raised rather than `hot_standby_feedback` enabled. Never read a payout input from the replica.

**The one-line summary:** at 1–5 lakh members the data volume is small; what will break the system is (i) synchronous 40-row fan-out creating permanent hot-row contention at the top of the tree, and (ii) a non-restartable close that double-pays on retry. Fix those two with batch-aggregated async fan-out and a chunked idempotent close, and the rest is ordinary PostgreSQL.
