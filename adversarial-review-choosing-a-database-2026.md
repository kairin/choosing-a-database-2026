# Adversarial Review — "Choosing a Database in 2026"

**Date of review:** 2026-09-17
**Reviewer:** Hermes Agent (adversarial fact-check pass)
**Method:** Every checkable claim in the source document was verified against primary sources (vendor documentation, release notes, project READMEs, Jira trackers) retrieved during this review. Verdicts: 🔴 **False / Dangerous**, 🟠 **Misleading / Ungrounded**, 🟡 **Imprecise / Stale-risk**, ✅ **Verified accurate**, ⚪ **Reviewer knowledge, not re-verified**.

**Overall verdict:** The document is well-structured and its *directional* guidance (Postgres as default, DuckDB for local OLAP, SQLite for embedded, MongoDB only when the document model truly fits) is sound. However, it contains **one operationally dangerous recommendation**, **one fundamental mischaracterization of DoltgreSQL's architecture**, **one prematurely-claimed PostgreSQL feature**, and **several invented quantitative figures**. It also omits licensing and operations — two dimensions its own introduction promises to cover. Publishable after the fixes in §6.

---

## 1. Findings Scoreboard

| # | Claim (abridged) | Severity | Verdict |
|---|---|---|---|
| F1 | "backups trivial (`cp db.sqlite`)" | 🔴 High | Dangerous advice for a live DB |
| F2 | DoltgreSQL is "PostgreSQL with Git"; matrix rows "Postgres-level" (concurrency, partitioning, joins, graph) | 🔴 High | False/misleading — it is a Postgres-*compatible* reimplementation on Dolt's engine |
| F3 | "99% Postgres-compatible" (Doltgres) | 🟠 High | True only as one specific test-suite metric; cherry-picked |
| F4 | "PostgreSQL has SQL/PGQ in core" | 🟠 High | Premature — committed for PG19, still beta; no stable release has it |
| F5 | SQLite: "millions of users" / "10 million users" | 🟠 High | Ungrounded; contradicts sqlite.org's own guidance |
| F6 | SQLite: "10x faster than networked Postgres" | 🟠 High | No source; not a claim sqlite.org makes |
| F7 | SQLite: "10,000 writes/sec in WAL mode" | 🟡 Medium | Ungrounded; hardware/durability-dependent |
| F8 | MongoDB "best when … you need to store large binary objects" | 🟠 Medium | Misleading — 16 MiB BSON limit; GridFS required beyond it |
| F9 | Doltgres: "your existing Postgres tools and libraries work" | 🟠 Medium | Overbroad — no extension support (no PostGIS/pgvector); feature gaps |
| F10 | Doltgres "Concurrency: Postgres-level (OLTP)" | 🟠 Medium | Ungrounded; vendor data shows a large high-concurrency gap |
| F11 | MariaDB "lacks `FULL OUTER JOIN` and `LATERAL`" | 🟡 Low | Accurate today; `FULL JOIN` fix committed for v13.2 — going stale |
| F12 | DuckDB "Foreign Keys ✅ Enforced" | 🟡 Low | Accurate with material caveats (auto ART index, load/update slowdown) |
| F13 | "steeper tuning curve than MySQL, fewer tutorials" (Postgres) | 🟡 Low | Contestable opinion stated as fact, unsourced |
| F14 | "Redis: not a primary database" | 🟡 Low | Defensible heuristic stated as absolute; Redis 8 broadened scope |
| F15 | "Snowflake/Databricks … batch analytics" | 🟡 Low | Simplification; both have moved beyond batch-only positioning ⚪ |
| F16 | Litestream/rqlite/LiteFS as SQLite distribution help | ✅ — | Verified; Litestream actively developed |
| F17 | Doltgres 1.0 "August 2026" | ✅ — | Verified — released 2026-08-06 |
| F18 | ClickHouse "JSON type storing each path as a typed subcolumn" | ✅ — | Verified against ClickHouse docs |
| F19 | Matrix labels (DuckDB partitioning "❌ (Hive Parquet)" etc.) | 🟡 Low | Emoji cells compress away material nuance |
| F20 | No versions cited anywhere in a "2026" guide | 🟡 Low | Editorial gap; several claims are version-sensitive |

---

## 2. High-Severity Findings

### F1 — 🔴 "`cp db.sqlite` makes backups trivial" is dangerous advice for a live database

**Claim:** "The single-file nature makes backups trivial (`cp db.sqlite`)."

**Evidence:** SQLite's own documentation describes the historical copy-the-file approach and its failure modes: writers must wait on a shared lock, and critically, *"If a power failure or operating system failure occurs while copying the database file the backup database may be corrupted following system recovery."* The Online Backup API exists specifically to address these concerns, producing a consistent snapshot of a running database.[1]

**Why it matters:** A reader following this advice on a live, actively-written database (especially in WAL mode, which the document itself recommends) can silently capture a torn or inconsistent backup. This is the only recommendation in the document that can cause actual data loss.

**Fix:** Replace with: use the Online Backup API, `VACUUM INTO`, or the CLI `.backup` command for live databases; plain `cp` is only safe when the database is closed or properly locked. Mention Litestream for continuous backup-to-object-storage (actively maintained, revamped in 2025).[1][29]

---

### F2 — 🔴 DoltgreSQL is not "PostgreSQL with Git"; the matrix's "Postgres-level" rows are false

**Claims:** "DoltgreSQL (Doltgres): PostgreSQL with Git" … matrix rows: Concurrency "Postgres-level (OLTP)", Partitioning "Postgres-level", Analytical Joins "Postgres-level", Graph Queries "Postgres-level", Foreign Keys "✅ Enforced".

**Evidence:**

- DoltgreSQL contains **no PostgreSQL code**. DoltHub's own engineering blog: *"DoltgreSQL is built on top of Dolt, which uses go-mysql-server as its query engine. Effectively, we are emulating Postgres on top of a custom engine we built for MySQL."*[5] The README confirms: *"Doltgres emulates a Postgres server … Doltgres uses the same SQL engine and storage format as Dolt."*[4]
- The README's stated limitations: *"No extension support yet. Backup and replication are a work in progress. Some Postgres syntax, types, functions, and features are not yet implemented."*[4] **No extension support means PostGIS, pgvector, and TimescaleDB — the very ecosystem the document touts as Postgres's strength — cannot run on Doltgres.**
- **Partitioning is not supported at all.** In DoltHub's Postgres-regression-test run, the single largest failure category (885 tests, ~2.1% of the suite) was `PARTITION OF is not yet supported`, and DoltHub explicitly states *"we don't currently support table partitioning."*[6] The matrix's "Partitioning: Postgres-level" is simply false.
- "Postgres-level (OLTP)" concurrency is ungrounded and contradicted by vendor benchmarks for the underlying engine: on high-concurrency TPC-C, *"MySQL does about 2.5X more transactions per second than Dolt,"* Dolt lacks `SELECT FOR UPDATE`, and its concurrent-write conflict semantics deliberately differ from both MySQL and Postgres.[9] The much-publicized "as fast as MySQL" result covers **single-threaded sysbench** of simple queries (read/write multiplier 1.07).[10]
- "Graph Queries: Postgres-level" inherits F4's problem — and there is no indication Doltgres supports PG19's beta SQL/PGQ at all.

**Fix:** Recharacterize as "a PostgreSQL-*compatible* version-controlled database built on Dolt's engine (not a Postgres fork)." Replace all four "Postgres-level" matrix cells with verified values: Partitioning ❌; Extensions ❌; Concurrency "OLTP-capable with different conflict semantics; ~2.5× TPS gap vs. conventional engines under high concurrency (vendor TPC-C, Jan 2025)"[9]; Graph "recursive CTEs" ⚪.

---

### F3 — 🟠 "99% Postgres-compatible" is a cherry-picked metric presented without context

**Claim:** "DoltgreSQL (Doltgres) hit 1.0 in August 2026 and is 99% Postgres-compatible."

**Evidence:** The 1.0 release (2026-08-06) does market "99% Postgres compatible," but defines it narrowly: a sqllogictest-derived suite of 5.6M queries — which *"specifically stress tests the expression support"* — plus ~20 client-library integration tests.[7][8] Against PostgreSQL's **own regression suite** (the tests that exercise Postgres-specific features), Doltgres passed **39.38%** as of July 2025 — DoltHub itself calls this *"the long tail of compatibility."*[6] The document repeats the marketing number without the metric, and places it directly next to the false "PostgreSQL with Git" framing, compounding F2.

**Fix:** "Doltgres reports 99% correctness on a sqllogictest-derived expression suite, but passed ~39% of PostgreSQL's own regression tests as of mid-2025; feature-level gaps (partitioning, extensions, some types/functions) remain."[6][7]

---

### F4 — 🟠 "PostgreSQL has SQL/PGQ in core" is premature for a production guide

**Claim:** "Graph Queries: ✅ SQL/PGQ in core" … "It also has the most advanced graph query support with SQL/PGQ syntax in core."

**Evidence:** SQL/PGQ was committed to PostgreSQL master on **2026-03-16** and ships with **PostgreSQL 19**[15][16] — which as of this review is still in **beta (19 Beta 3, 2026-08-13); the latest stable line is 18.x, whose feature list contains no SQL/PGQ.[14][17]** Additionally, the implementation is property graphs as **read-only views over relational tables**, not native graph storage.[18] "Most advanced graph query support" is an unsourced superlative — dedicated graph databases (and even the PGQ ecosystem) would contest it, and the document names no graph-native option for comparison.

**Fix:** "SQL/PGQ (GRAPH_TABLE, CREATE PROPERTY GRAPH) is committed for PostgreSQL 19, in beta as of Sept 2026 and not yet in any stable release; graphs are read-only views over tables. Until PG19 GA, recursive CTEs remain the production option."[14][15][18]

---

### F5/F6/F7 — 🟠 The SQLite performance figures are ungrounded and contradict sqlite.org

**Claims:** "production apps up to millions of users" (overview); "production workloads up to 10 million users"; "10x faster than networked Postgres for local data reads"; "handles 10,000 writes/sec in WAL mode."

**Evidence:**

- sqlite.org's own positioning: *"any site that gets fewer than 100K hits/day should work fine with SQLite … a conservative estimate, not a hard upper bound. SQLite has been demonstrated to work with 10 times that amount of traffic."*[2] That is ~1M **hits/day** demonstrated — the document's "10 million **users**" converts an already-loose traffic ceiling into a user count with no methodology, and inflates it a further order of magnitude. The same page says "many concurrent writers → choose client/server," which the document correctly notes elsewhere — making the "millions of users" framing internally incoherent (millions of users with a single-writer engine requires heavy read-replica architecture the document doesn't describe).[2]
- The "10x faster than networked Postgres" figure appears nowhere in SQLite's documentation. The adjacent, real claim is that SQLite reads/writes small blobs **~35% faster than the filesystem** (up to ~5× on Windows in one chart).[3] Comparing an embedded engine to a *networked* server conflates network latency with database speed — an apples-to-oranges benchmark presented with a fabricated precision ("10x").
- "10,000 writes/sec in WAL" has no source and is meaningless without transaction size, durability setting (`synchronous=NORMAL` vs `FULL`), and hardware. WAL mode remains **single-writer** regardless.[2]

**Fix:** Delete the absolute numbers or tie them to conditions and sources. Honest version: "sqlite.org positions SQLite for low-to-medium-traffic sites (~100K hits/day conservative, 10× demonstrated); single-writer concurrency in WAL; local reads avoid client-server round-trips."[2][3]

---

## 3. Medium-Severity Findings

### F8 — 🟠 MongoDB as the choice for "large binary objects"

**Claim:** MongoDB is best "when your data is deeply nested, schema is evolving rapidly, or you need to store large binary objects."

**Evidence:** MongoDB's hard BSON document limit is **16 mebibytes**; larger payloads require the GridFS API — a convention layered on top, not native large-object storage.[19] Recommending MongoDB *for* large binaries without stating the 16 MiB ceiling invites a design surprise. (Postgres `bytea`/TOAST and plain object storage are the common alternatives; S3-compatible stores usually win past small thumbnail sizes.) ⚪

**Fix:** "…or you need flexible documents up to 16 MiB (GridFS beyond that)."[19]

### F9 — 🟠 "Your existing Postgres tools and libraries work" (Doltgres)

**Evidence:** Wire-protocol compatibility is real for the ~20 officially tested client libraries[8], but "tools" broadly fails where it matters most: **no extension support** (PostGIS, pgvector, TimescaleDB excluded)[4], and DoltHub's regression run catalogs missing surface area — `CREATE FUNCTION` restricted to PL/pgSQL, missing psql commands, missing `jsonb_path_query` and other functions, no publications/cursors.[6] Import via `pg_dump`/`psql` works only when the dump avoids unsupported features.[4]

**Fix:** Qualify: "supported client drivers and plain-SQL workloads port well; extensions, advanced catalogs, and some functions do not."[4][6][8]

### F10 — 🟠 "Concurrency: Postgres-level (OLTP)" (Doltgres cell)

Covered in F2 — restated here because the matrix cell will be screenshotted and quoted out of context. Vendor data: 2.5× TPS deficit vs MySQL on high-concurrency TPC-C; near-parity only on single-threaded sysbench.[9][10]

### F12 — 🟡 DuckDB "Foreign Keys ✅ Enforced"

**Evidence:** True — DuckDB enforces FKs and automatically creates an ART index per FK. But the same documentation warns constraints *"have a strong impact on performance: they slow down loading and updates,"* and that index limitations can cause *"constraints being evaluated too eagerly"* (spurious constraint-violation errors).[28] For a guide whose audience will bulk-load into DuckDB, that caveat changes behavior (many pipelines drop constraints for load, re-add after). ⚪

### F13 — 🟡 "Steeper tuning curve than MySQL, fewer 'just works' tutorials" (Postgres)

Unsourced, and arguably inverted: Postgres ships saner defaults and the ecosystem tutorial volume is enormous. Presented as fact under "Watch out for" with no citation. Either source it or soften to "more knobs to learn; defaults have improved substantially in recent releases." ⚪

### F14 — 🟡 "Redis: Not a primary database"

A defensible heuristic stated as absolute. Redis 8 (May 2025) integrated Redis Stack capabilities — JSON, time series, probabilistic types, query engine — into core under AGPLv3, explicitly broadening beyond cache duty.[21] Keep the heuristic, add the nuance and the durability/persistence caveat. ⚪

### F15 — 🟡 "Snowflake / Databricks: cloud data warehouses for enterprise-scale batch analytics"

Simplification: Snowflake has pushed into hybrid transactional tables and Databricks positions as a lakehouse with OLTP ambitions; "batch analytics" undersells both. ⚪ *(Reviewer knowledge — not independently re-verified in this pass.)*

---

## 4. Internal Inconsistencies & Editorial Issues

1. **"millions of users" (overview) vs "10 million users" (deep dive)** — two different unsourced numbers for the same claim; both contradict sqlite.org guidance.[2]
2. **"eliminates joins by embedding related data" vs praising the aggregation pipeline's `$lookup`/`$graphLookup`** — if embedding eliminated joins, the join operators wouldn't be headline features. The honest statement is "reduces join frequency; joins exist but are manual and order-sensitive."
3. **Intro promises "operational reality"; the matrix contains zero operational rows** — no replication/HA, no backup, no managed-service availability, no license, no cost. The matrix measures features, not operations. This is the document's biggest structural broken promise.
4. **Emoji cells compress away nuance** — e.g. DuckDB partitioning "❌ (Hive Parquet)": DuckDB *can* write hive-partitioned Parquet datasets, so the ❌ (no native table partitioning) and the parenthetical (it does partition externally) half-contradict; a reader can't tell which. ⚪
5. **No versions cited anywhere** despite the "in 2026" framing — fatal for a time-sensitive guide: PG18 is the current stable, PG19 is beta[14], Doltgres is 1.0.x[13], Redis is 8.x under a new license[21], MariaDB's FULL JOIN fix targets 13.2.[24]
6. **MongoDB section tension** — the document says choose MongoDB "only if your data model truly benefits from flexible documents" while also claiming Postgres JSONB is "often a better choice than MongoDB for semi-structured data." Both may be true, but the document never reconciles them into a decision rule (e.g., "when does JSONB stop being enough?").

---

## 5. Material Omissions

1. **Licensing — absent entirely, and it changes real 2026 decisions.**
   - MongoDB: SSPL, *"not considered open source by the OSI"* — matters for vendors and some enterprises.[20]
   - Redis: tri-license (RSALv2 / SSPLv1 / **AGPLv3**) since Redis 8, May 2025 — a major, recent licensing event the guide misses.[21][22]
   - CockroachDB: consolidated under the CockroachDB Software License (Nov 18, 2024); free below **$10M annual revenue**, paid above — directly relevant to the startups this guide addresses.[23]
2. **MySQL proper is missing from "other notable options"** while its fork MariaDB is included — an odd inversion for a guide aimed at generalists.
3. **libsql/Turso absent** — the document names Litestream, rqlite, and LiteFS for distributed SQLite but omits the SQLite fork/edge-replication story that defined 2024–2026 SQLite discourse. (Litestream at least is verified actively maintained — v0.5 revamp 2025, releases continuing into 2026.[29][30]) ⚪
4. **Managed/serverless Postgres (Neon, Supabase, Aurora/RDS) absent** — for many 2026 teams the delivery model *is* the database decision.
5. **No graph-native database (Neo4j et al.)** despite a dedicated "Graph Queries" matrix row and the "most advanced graph support" superlative for Postgres — the comparison class is missing. ⚪
6. **No search engine (Elasticsearch/OpenSearch)** and **no wide-column store (Cassandra/ScyllaDB/DynamoDB)** — two of the most common "second database" categories. ⚪
7. **No cost/TCO discussion** — licensing thresholds, managed-vs-self-hosted, and DuckDB/SQLite's ~zero marginal cost are decision-relevant.

---

## 6. Recommended Fixes (in priority order)

1. **(Safety)** Replace the `cp db.sqlite` backup advice with Online Backup API / `VACUUM INTO` / `.backup`; mention Litestream for continuous replication.[1][29]
2. **(Accuracy)** Rewrite the Doltgres subsection and all five "Postgres-level" matrix cells: Postgres-*compatible* emulation on Dolt's engine; no extensions; no table partitioning; backup/replication immature; distinct concurrency semantics; qualify "99%" as one expression-test metric against ~39% on Postgres's own regression suite.[4][5][6][7][9]
3. **(Currency)** Fix SQL/PGQ: committed for PostgreSQL 19 (beta as of this writing), absent from stable 18.x, property graphs are read-only views.[14][15][17][18]
4. **(Honesty)** Delete or properly condition the SQLite numbers ("10 million users", "10x", "10,000 writes/sec"); align with sqlite.org's stated envelope.[2][3]
5. **(Completeness)** Add the 16 MiB BSON limit + GridFS to the MongoDB binary-object claim.[19]
6. **(Structure)** Add matrix rows: License, Replication/HA, Backup, Managed offerings, Current major version — fulfilling the intro's "operational reality" promise.
7. **(Scope)** Add a licensing paragraph (SSPL, Redis AGPLv3, CockroachDB $10M threshold)[20][21][23] and the missing categories: MySQL, libsql/Turso, serverless Postgres, graph-native, search, wide-column.
8. **(Rigor)** Attach a version + citation to every time-sensitive claim; mark opinions ("tuning curve", "fewer tutorials") as opinion or source them.
9. **(Consistency)** Pick one SQLite user-scale claim and source it; reconcile the JSONB-vs-MongoDB tension into an explicit decision rule.

---

## 7. What Checks Out (credit where due)

- **Doltgres 1.0 "August 2026"** — verified: announced for Aug 6, shipped v1.0.0 on 2026-08-06.[12][13]
- **Doltgres version-control model** (branch/diff/merge/clone; content-addressed tree storage; fast branching, some read overhead) — consistent with Dolt's architecture and published benchmarks (sysbench near-parity single-threaded; ~5% slower reads / ~10% faster writes vs MySQL in vendor latency tests).[4][10][11]
- **SQLite**: single-writer under WAL; FK enforcement off by default; massive ecosystem; Litestream/rqlite/LiteFS named as distribution aids — all accurate.[2][29]
- **MongoDB**: no FK/referential integrity (application-level); pipeline stage order matters with no cost-based join reorder; `$graphLookup` exists; WiredTiger B+trees; sharding; 16 MiB doc limit mechanics — accurate.[19]
- **DuckDB**: OLAP/columnar/vectorized positioning; direct reads of Parquet/CSV/JSON over HTTP and cloud storage; Iceberg/Delta support; ASOF JOIN; "not for high-frequency small-transaction OLTP"; FKs enforced (with the F12 caveats) — accurate.[28]
- **ClickHouse**: MergeTree family; JSON type storing paths as typed subcolumns; no FKs — verified against current docs.[27]
- **MariaDB**: pluggable engines (InnoDB/ColumnStore/MyRocks); LATERAL unsupported (MDEV-33018 open); FULL OUTER JOIN unsupported *in current releases* — but note the committed fix targeting 13.2 (F11).[24][25][26]
- **CockroachDB/TiDB characterization** (distributed SQL, Postgres/MySQL compatibility, horizontal scale) — accurate; only the licensing context is missing.[23]

---

## 8. Appendix — Claim-by-Claim Verification Log

| Claim in document | Verdict | Key source(s) |
|---|---|---|
| Doltgres 1.0 August 2026 | ✅ Verified (Aug 6, 2026) | [12][13] |
| Doltgres "99% Postgres-compatible" | 🟠 Metric-limited (sqllogictest expression suite; 39.38% on Postgres regression suite) | [6][7][8] |
| Doltgres "PostgreSQL with Git" | 🔴 No Postgres code; emulation on Dolt/go-mysql-server | [4][5] |
| Doltgres matrix "Postgres-level" partitioning | 🔴 Partitioning unsupported | [6] |
| Doltgres matrix "Postgres-level" concurrency | 🟠 2.5× TPS gap on high-concurrency TPC-C (vendor) | [9][10] |
| Doltgres "existing Postgres tools and libraries work" | 🟠 Clients yes; extensions no; feature gaps | [4][6][8] |
| Doltgres content-addressed tree, branch fast/reads slower | ✅ Directionally consistent with vendor benchmarks | [10][11] |
| Postgres "SQL/PGQ in core" | 🟠 PG19 beta only; not in stable 18.x; read-only views | [14][15][17][18] |
| Postgres "most advanced graph query support" | 🟠 Unsourced superlative; graph-native DBs unmentioned | [18] |
| SQLite "millions/10M users" | 🟠 Ungrounded; sqlite.org: ~100K hits/day, 10× demonstrated | [2] |
| SQLite "10x faster than networked Postgres" | 🟠 No source; real claim is ~35% vs filesystem blobs | [3] |
| SQLite "10,000 writes/sec WAL" | 🟡 Ungrounded, condition-dependent | [2] |
| SQLite `cp` backups trivial | 🔴 Corruption risk on live DB; use Online Backup API | [1] |
| SQLite WAL single-writer; FK off by default | ✅ Verified | [2] |
| Litestream/rqlite/LiteFS help distribute SQLite | ✅ Litestream active (v0.5, 2025–2026 releases) | [29][30] |
| DuckDB OLAP positioning, ASOF, direct file reads, Iceberg/Delta | ✅ Verified (docs) | [28] |
| DuckDB FK ✅ enforced | 🟡 True + auto ART index + load/update slowdown caveats | [28] |
| DuckDB partitioning "❌ (Hive Parquet)" | 🟡 Label muddles native vs hive-partitioned writes ⚪ | — |
| MongoDB 16 MiB limit / "large binary objects" | 🟠 GridFS required beyond 16 MiB | [19] |
| MongoDB FK app-level; pipeline order manual; $graphLookup | ✅ Verified | [19] |
| ClickHouse JSON typed subcolumns; MergeTree; no FK | ✅ Verified | [27] |
| MariaDB lacks FULL OUTER JOIN | 🟡 True today; fix committed for 13.2 (Apr 2026) | [24][25] |
| MariaDB lacks LATERAL | ✅ Still open (MDEV-33018) | [26] |
| Redis "not a primary database" | 🟡 Heuristic; Redis 8 broadened scope + AGPLv3 | [21] |
| MongoDB/Redis/Cockroach licensing (absent) | 🟠 Omission: SSPL / AGPLv3 tri-license / $10M threshold | [20][21][22][23] |
| "Snowflake/Databricks batch analytics" | 🟡 Simplification ⚪ | — |
| "Postgres steeper tuning curve than MySQL" | 🟡 Unsourced opinion ⚪ | — |

---

## Sources

1. https://www.sqlite.org/backup.html — SQLite Online Backup API (corruption risk of naive file copy)
2. https://www.sqlite.org/whentouse.html — Appropriate Uses for SQLite (~100K hits/day guidance; concurrent writers → client/server)
3. https://www.sqlite.org/fasterthanfs.html — 35% Faster Than The Filesystem
4. https://github.com/dolthub/doltgresql — DoltgreSQL README (limitations; architecture)
5. https://www.dolthub.com/blog/2024-04-01-prepared-statements-postgres/ — "emulating Postgres on top of a custom engine we built for MySQL"
6. https://dolthub.com/blog/2025-07-08-doltgres-correctness-testing/ — 39.38% Postgres regression pass; `PARTITION OF is not yet supported`
7. https://www.dolthub.com/blog/2026-07-10-doltgres-99-percent-sql-logic-tests/ — definition of the 99% metric
8. https://www.dolthub.com/blog/2026-08-06-doltgres-1-0/ — Doltgres 1.0 release ("99% Postgres compatible"; 5.6M queries; 20+ client libraries)
9. https://www.dolthub.com/blog/2025-01-02-dolt-mysql-differences — TPC-C 2.5× gap; no `SELECT FOR UPDATE`; concurrent-write semantics
10. https://www.dolthub.com/blog/2025-12-12-how-dolt-got-as-fast-as-mysql — sysbench read/write multiplier 1.07
11. https://www.dolthub.com/docs/sql-reference/benchmarks/latency — Dolt vs MySQL latency benchmarks
12. https://www.dolthub.com/blog/2026-06-26-doltgres-1-0-coming-this-fall — "Doltgres 1.0 Coming August 6th"
13. https://pkg.go.dev/github.com/dolthub/doltgresql@v1.0.0 — v1.0.0 published 2026-08-06
14. https://www.postgresql.org/ — latest releases: 19 Beta 3, 18.6 (2026-08-13)
15. https://postgresql.org/docs/19/release-19.html — PostgreSQL 19 release notes (SQL/PGQ)
16. https://www.depesz.com/2026/07/31/waiting-for-postgresql-19-sql-property-graph-queries-sql-pgq/ — SQL/PGQ committed 2026-03-16
17. https://www.postgresql.org/about/news/postgresql-18-released-3142 — PostgreSQL 18 feature list (no SQL/PGQ)
18. https://www.postgresql.org/docs/devel/ddl-property-graphs.html — property graphs as read-only views over tables
19. https://www.mongodb.com/docs/manual/reference/limits/ — BSON 16 mebibyte limit; GridFS
20. https://www.mongodb.com/legal/licensing/server-side-public-license/faq — SSPL not OSI-approved
21. https://redis.io/blog/agplv3/ — Redis 8 adds AGPLv3 option (2025-05-01)
22. https://redis.io/legal/licenses/ — Redis tri-license (RSALv2 / SSPLv1 / AGPLv3)
23. https://techcrunch.com/2024/08/15/cockroach-labs-shakes-up-its-licensing-to-force-bigger-companies-to-pay/ — CockroachDB Software License, $10M revenue threshold (2024-11-18)
24. https://jira.mariadb.org/browse/MDEV-39014 — FULL JOIN Algorithm Phase 2, fix version 13.2, in review
25. https://github.com/MariaDB/server/commit/80489d27e35557a51593492d75b4a03f7afe7381 — FULL JOIN Phase 2 commit (2026-04-13)
26. https://jira.mariadb.org/browse/MDEV-33018 — LEFT JOIN LATERAL unsupported
27. https://clickhouse.com/docs/reference/data-types/newjson — ClickHouse JSON type subcolumns
28. https://duckdb.org/docs/stable/sql/constraints.html — DuckDB FK enforcement via automatic ART index; constraint performance caveats
29. https://fly.io/blog/litestream-revamped/ — Litestream revamped (2025-05)
30. https://pkg.go.dev/github.com/benbjohnson/litestream — Litestream release history (active into 2026)

*Notes on method: all sources above were retrieved during this review session on 2026-09-17. Items marked ⚪ are reviewer knowledge not independently re-verified in this pass; treat them as lower-confidence. Verdicts on time-sensitive claims (F4, F11) should be re-checked after PostgreSQL 19 GA and MariaDB 13.2 release.*
