# Adversarial Review (v2, source-verified) — "Choosing a Database in 2026" Comparison Guide

- **Reviewer:** Adversarial red-team review
- **Version:** 2.0 (revised after primary-source verification; v1 was knowledge-only)
- **Date of review:** 2026-09-17
- **Subject:** The comparison article covering PostgreSQL, SQLite, DuckDB, MongoDB, DoltgreSQL, and "other notable options"
- **Method:** Claim-by-claim challenge. Every disputed claim was checked against primary sources (vendor documentation, release notes, official blogs, GitHub, license pages). Findings are classified **Correct ✅**, **Qualified/Needs caveat ⚠️**, **Incorrect ❌**, or **Unverifiable/Unsourced ❓**, each with a citation.

---

## 0. Revision Note — What Changed After Checking Sources

v1 of this review was written from static knowledge only. Primary-source verification **confirmed some findings and overturned others**. This is worth stating plainly, because it demonstrates exactly why the underlying article needed a source check:

| Claim | v1 verdict | v2 verdict (verified) |
| :--- | :--- | :--- |
| "MariaDB … lacks `LATERAL`" | ❌ Incorrect | ❌ **Confirmed incorrect** (MariaDB has LATERAL since 10.2) |
| "SQL/PGQ in PostgreSQL core" | ❌ Incorrect | ⚠️ **Overturned** — SQL/PGQ is committed to PostgreSQL 19 (beta as of Aug 2026) |
| "DoltgreSQL hit 1.0 in August 2026" | ❓ Unverifiable | ✅ **Confirmed** (v1.0.0 released 2026-08-06) |
| "Doltgres 99% Postgres-compatible" | ❓ Marketing | ⚠️ **Refined** — it *is* the vendor's published number (99.3% on sqllogictest), but uncited and not feature-completeness |
| "Doltgres Postgres-level OLTP" | ⚠️ Overstated | ⚠️ **Confirmed overstated** — vendor's own benchmark: 2.7× slower overall |
| SQLite "10M users / 10x faster / 10k writes" | ❓ Unsourced | ❓ **Confirmed unsourced/misleading** (SQLite's real figures differ) |
| TiDB "strong consistency" | ⚠️ Overstated | ⚠️ **Confirmed** — TiDB default is Snapshot Isolation |
| Licensing omission | ⚠️ Major gap | ⚠️ **Confirmed** (SSPL/RSAL specifics pinned down) |

---

## 1. Verdict (Executive Summary)

The article is **directionally useful as an orientation**, and — after verification — **more factually accurate than a first pass suggested**. But it still fails as reference-grade guidance for three reasons:

1. **One hard factual error** (MariaDB "lacks `LATERAL`").
2. **One premature/uncaveated claim** (SQL/PGQ "in core" — true of PostgreSQL 19 *beta*, but not yet in any GA release).
3. **Multiple uncited quantitative claims**, including at least one that contradicts the vendor's own published data ("Postgres-level OLTP" for Doltgres, which is 2.7× slower per the vendor).

It also **relays vendor marketing numbers ("99% compatible") without attribution or caveats** and **omits licensing entirely** — the single biggest real-world decision factor for MongoDB, Redis, MariaDB, and CockroachDB.

**Bottom line:** Usable as a first orientation after correcting the items in §8. Do not use for build decisions in its current form.

---

## 2. Confirmed Factual Error ❌

### 2.1 "MariaDB … lacks … `LATERAL`" — **Incorrect** ❌

> Original: *"It lacks some Postgres features like `FULL OUTER JOIN` and `LATERAL`."*

- **MariaDB has supported `LATERAL` derived tables since MariaDB 10.2** (GA May 2017). MariaDB's own documentation describes "Lateral Derived" as the form "the SQL Standard refers to as *lateral*."
  - Source: <https://mariadb.com/kb/en/lateral-derived-optimization/>
  - Source: MariaDB 10.2 release (first stable May 2017) — <https://mariadb.com/docs/release-notes/community-server/old-releases/10.2/what-is-mariadb-102/>
- The **`FULL OUTER JOIN` half of the sentence is correct**: MariaDB (like MySQL) does not support `FULL OUTER JOIN` and requires a `LEFT JOIN … UNION … RIGHT JOIN` workaround.
  - Source: MariaDB join-syntax documentation; corroborated by Stack Overflow #4796872 ("How can I do a FULL OUTER JOIN in MySQL?").
- **Confidence:** High. **Fix:** keep "no `FULL OUTER JOIN`," delete "`LATERAL`," and substitute a genuine gap (e.g., no `DISTINCT ON`, no GIN/GiST-equivalent index types of the same power, weaker JSONB/function surface).

---

## 3. Premature / Unattributed Claims

### 3.1 "SQL/PGQ syntax in core" (PostgreSQL) — **Premature, but essentially correct in direction** ⚠️

> Original: *"It also has the most advanced graph query support with SQL/PGQ syntax in core."* (and matrix row "Graph Queries: ✅ SQL/PGQ in core")

- **SQL/PGQ has been committed to PostgreSQL 19** ("Add support for SQL Property Graph Queries (SQL/PGQ)" — Peter Eisentraut, Ashutosh Bapat).
  - Source: PostgreSQL 19 release notes — <https://www.postgresql.org/docs/current/release-19.html>
- **However, PostgreSQL 19 is not yet GA.** As of the latest news at review time (2026-08-13), PostgreSQL 19 was in **Beta 3**; the current released line is PostgreSQL 18.6.
  - Source: <https://www.postgresql.org/about/news/> ("August 13, 2026: PostgreSQL 18.6 … and 19 Beta 3 Released!")
- **Assessment:** The article's claim is **correct about the codebase but premature for production**. "In core" is accurate for the dev branch; it is not accurate for any *released* version as of this review. A careful phrasing is: *"SQL/PGQ property-graph queries land in PostgreSQL 19 (in beta as of Aug 2026); through PostgreSQL 18, graph support is via recursive CTEs or the Apache AGE extension."*
- **Confidence:** High.

### 3.2 "99% Postgres-compatible" (Doltgres) — **Vendor's own number, relayed without attribution** ⚠️

> Original: *"DoltgreSQL … is 99% Postgres-compatible."*

- The "99%" figure **is** DoltHub's published claim: "Doltgres 1.0 is 99% Postgres compatible," based on a 5.6-million-query sqllogictest suite scoring **99.317%**.
  - Source: DoltHub blog, "Doltgres 1.0" (2026-08-06) — <https://www.dolthub.com/blog/2026-08-06-doltgres-1-0/>
  - Source: Doltgres README correctness table — <https://github.com/dolthub/doltgresql>
- **The problem is not the number's existence; it's what it measures.** A sqllogictest *correctness* percentage ≠ "99% of Postgres features work." The same vendor sources list material gaps: **PostGIS, vector indexes, row-level security, collation support, "better DDL support," and "limited extension support."** Also note the test breakdown itself: of ~5.67M queries, 13,197 were "not ok" and 25,552 "did not run."
- **Assessment:** The article states a vendor marketing figure as an independent fact, without attribution or the feature-gap caveats. That is the adversarial failure — not that the number is fabricated.
- **Confidence:** High. **Fix:** attribute to the vendor and add "per DoltHub's sqllogictest suite; major extensions (PostGIS, vector indexes, RLS) are still on the roadmap."

---

## 4. Confirmed Overstatement ⚠️

### 4.1 Doltgres "Concurrency: Postgres-level (OLTP)" — **Contradicted by the vendor's own benchmark** ⚠️

- DoltHub's published Sysbench comparison (Doltgres 1.0.0 vs PostgreSQL) shows Doltgres **~2.3× slower on reads (mean), ~3.4× slower on writes (mean), ~2.7× slower overall** (median latency).
  - Source: Doltgres README, "Performance" section — <https://github.com/dolthub/doltgresql>
- The 1.0 blog states it plainly: *"Doltgres is slower than Postgres but still more than fast enough…"* — i.e., the vendor does **not** claim parity.
  - Source: <https://www.dolthub.com/blog/2026-08-06-doltgres-1-0/>
- **Assessment:** "Postgres-level (OLTP)" is inaccurate. It *is* correct that Doltgres speaks the Postgres wire protocol and accepts concurrent OLTP connections. **Fix:** "Wire-compatible with Postgres clients; ~2.7× slower than Postgres on vendor Sysbench (1.0.0)."

### 4.2 TiDB "strong consistency" — **Overstated** ⚠️

- TiDB's documented default is **Snapshot Isolation (SI)**, "advertised as REPEATABLE-READ for compatibility with MySQL" — explicitly *not* the serializable-by-default guarantee CockroachDB provides.
  - Source: <https://docs.pingcap.com/tidb/stable/transaction-isolation-levels>
- **Confidence:** High. **Fix:** say "horizontal scaling with MySQL compatibility" and either drop "strong consistency" or qualify it (SI, not serializable-by-default).

---

## 5. Unsourced / Misleading Quantitative Claims ❓

### 5.1 SQLite "up to 10 million users for read-heavy applications" — **Conflates "users" with "hits/day"** ❓

- SQLite's official guidance is about **traffic, not users**: "any site that gets fewer than 100K hits/day should work fine with SQLite" (conservative), "demonstrated to work with 10 times that amount."
  - Source: <https://www.sqlite.org/whentouse.html>
- "10 million **users**" is an unsupported extrapolation, with no stated schema, query mix, or caching. Combined with SQLite's **single-writer** model, the unconditional phrasing is misleading.
- **Fix:** quote SQLite's own 100K-hits/day (conservative) / ~1M-hits/day (demonstrated) guidance and state the single-writer constraint.

### 5.2 SQLite "10x faster than networked Postgres for local data reads" — **No source; scenario-dependent** ❓

- No "10×" figure appears in SQLite's official materials. The real, documented advantage is architectural (no client/server round-trip), not a universal 10× multiplier.
- **Fix:** replace with a qualified statement ("eliminates client/server IPC overhead; can be faster for single-process, local, read-mostly workloads").

### 5.3 SQLite "handles 10,000 writes/sec in WAL mode" — **True only with batching or `synchronous=OFF`** ⚠️

- SQLite's own FAQ: "SQLite will easily do 50,000 or more INSERT statements per second … **But it will only do a few dozen transactions per second**" (durable commits, default `synchronous=FULL`, fsync-bound). The fast path requires wrapping inserts in one transaction or `PRAGMA synchronous=OFF`, which "might go corrupt" on power loss.
  - Source: <https://www.sqlite.org/faq.html> (FAQ #19, "INSERT is really slow…")
- **Assessment:** "10,000 writes/sec" is defensible *only* with batching or relaxed durability. As an unqualified figure it misrepresents default behavior.
- **Fix:** specify preconditions or drop the number.

---

## 6. Confirmed-Correct Claims (for balance) ✅

| Claim in article | Source | Verdict |
| :--- | :--- | :--- |
| DoltgreSQL hit 1.0 in August 2026 | GitHub releases: v1.0.0 = 2026-08-06 | ✅ Correct (now at v1.3.3 as of 2026-09-15) |
| Doltgres uses content-addressed tree storage | Dolt docs: Prolly Tree = "content-addressed B-tree" | ✅ Correct |
| SQLite foreign keys "off by default" | sqlite.org pragma doc: "default … is OFF" | ✅ Correct |
| MongoDB has no FK enforcement | MongoDB docs (well-established) | ✅ Correct |
| DuckDB enforces foreign keys | duckdb.org constraints page | ✅ Correct (creates ART index; has an "eager evaluation" caveat) |
| DuckDB not for high-frequency OLTP | duckdb.org: "bulk-optimized MVCC" | ✅ Correct |
| ClickHouse JSON = typed subcolumns | clickhouse.com JSON type page ("sub-columns", "typed paths") | ✅ Correct |
| DuckDB supports Iceberg and Delta Lake | `duckdb-iceberg` and `duckdb-delta` extensions | ✅ Correct — **but via extensions, both vendor-marked experimental** |
| Doltgres "reads can be slower due to tree traversal" | vendor benchmark: reads 2.3× slower than Postgres | ✅ Directionally correct |
| MariaDB lacks `FULL OUTER JOIN` | MariaDB join docs / Stack Overflow | ✅ Correct |

---

## 7. Critical Omissions (unchanged, now sourced)

1. **Licensing — the biggest gap.** The article never mentions it, and it is frequently *the* deciding factor:
   - **MongoDB** → SSPLv1 (source-available, **not** OSI-approved). Source: <https://www.mongodb.com/legal/licensing/server-side-public-license>
   - **Redis** → BSD-3 for ≤7.2; RSALv2/SSPLv1 for 7.4–7.8; RSALv2/SSPLv1/**AGPLv3** (AGPLv3 is OSI-approved) for 8+. Source: <https://redis.io/legal/licenses/>
   - **MariaDB** → GPLv2 (commercial dual-license for enterprise features).
   - **CockroachDB** → source-available (BSL) with a free tier limit.
   - (Note: my v1 said Redis "relicensed portions … to RSAL/SSPL"; the current, accurate picture is the three-license regime above.)
2. **Operational reality** — HA/failover, backups/PITR, managed-service cost, on-call burden. Compared engines, not *running* them.
3. **Consistency & isolation models** — no mention of isolation levels, write skew, linearizability, or durability (`fsync`) semantics, despite making concurrency claims (and getting TiDB's wrong).
4. **Sources** — zero citations anywhere; this review found that absence actively concealed two errors.
5. **Version pinning** — no claim is tied to a version, which is why the SQL/PGQ claim drifted.
6. **"Polyglot vs. one DB"** — real systems run PostgreSQL *and* Redis *and* DuckDB together; the "pick one" framing undersells this.

---

## 8. Recommended Corrections (concrete, ordered by severity)

| # | Location | Change |
| :--- | :--- | :--- |
| 1 | MariaDB section | Delete "and `LATERAL`". Keep "no `FULL OUTER JOIN`". |
| 2 | PostgreSQL + matrix | Re-say SQL/PGQ as "ships in PostgreSQL 19 (beta as of Aug 2026); not yet GA." |
| 3 | DoltgreSQL section | Attribute "99% compatible" to DoltHub's sqllogictest score and list the missing features (PostGIS, vector indexes, RLS, collations, DDL). Replace "Postgres-level OLTP" with "~2.7× slower than Postgres (vendor Sysbench, 1.0.0)." |
| 4 | SQLite section | Replace "10M users / 10× faster / 10k writes/sec" with SQLite's own traffic guidance and the transaction/`synchronous` caveats. |
| 5 | TiDB line | Drop or qualify "strong consistency" (default is Snapshot Isolation). |
| 6 | DuckDB line | Note Iceberg/Delta are via extensions, vendor-marked experimental. |
| 7 | New section | Add "Licensing & operational cost." |
| 8 | Global | Cite sources; pin every claim to a version/date. |

---

## 9. Confidence Summary (post-verification)

| Finding | Verdict | Confidence |
| :--- | :--- | :--- |
| MariaDB "lacks LATERAL" is wrong | ❌ | **High** (primary docs) |
| SQL/PGQ in PostgreSQL core (PG 19 beta, not GA) | ⚠️ premature | **High** (release notes + news) |
| Doltgres 1.0 in Aug 2026 | ✅ correct | **High** (GitHub) |
| "99% compatible" = vendor sqllogictest score, uncited | ⚠️ unattributed | **High** (blog + README) |
| Doltgres "Postgres-level OLTP" overstated (2.7× slower) | ⚠️ | **High** (vendor benchmark) |
| SQLite "10M/10×/10k" unsourced & misleading | ❓ | **High** (sqlite.org FAQ/whentouse) |
| TiDB "strong consistency" overstated (SI default) | ⚠️ | **High** (TiDB docs) |
| DuckDB FK enforced; ClickHouse JSON subcolumns; Dolt Prolly tree; SQLite FK off-by-default | ✅ | **High** |
| Iceberg/Delta via experimental extensions | ⚠️ nuance | **High** (extension repos) |
| Licensing omission is decision-relevant | ⚠️ | **High** (license pages) |

---

## 10. Final Recommendation

Keep the orientation framing. Before publishing or relying on it:

1. Fix the one hard error (MariaDB `LATERAL`).
2. Re-time the SQL/PGQ claim (PG 19 beta, not GA) and attribute the Doltgres "99%".
3. Replace the Doltgres "Postgres-level OLTP" with the vendor's actual 2.7× figure.
4. Source or delete every number.
5. Add licensing/operational-cost — it is the largest real-world differentiator the piece currently omits.

**Net of this review:** the article is a **reasonable orientation** with **one factual error, two overstatements, and several uncited numbers**, not a fabrication. It should be treated as a conversation starter until the §8 corrections are applied.
