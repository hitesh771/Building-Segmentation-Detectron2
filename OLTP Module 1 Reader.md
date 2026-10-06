# OLTP Module 1 Reader: Foundations of Transactional Systems

Each topic follows one pattern: **What it is → Why it's needed → What if it's missing → Industry methods → Tools.**

---

## 1. Introduction to OLTP

### 1.1 Core definition

OLTP is a workload class: a very large number of **short, concurrent, small read/write transactions** (each touching a handful of rows by key) that must be **correct, immediate, and durable**. Typical metrics: transactions per second (TPS), p95/p99 latency (milliseconds), availability (99.95%+), and zero lost commits.

Workload signature:

| Trait | OLTP |
| --- | --- |
| Operation | INSERT / UPDATE / DELETE / point SELECT |
| Rows per txn | 1 to \~100 |
| Latency target | 1–50 ms |
| Concurrency | Thousands of simultaneous sessions |
| Data freshness | Current state, always |
| Users | End customers, apps, APIs (not analysts) |
| Size | GB to low TB (hot set fits mostly in RAM) |

**Why needed:** Every business action that changes state (pay, book, ship, withdraw) must be recorded exactly once, immediately, and be visible to the next user. **If absent:** You'd have spreadsheets or batch files: double-booked seats, oversold stock, lost payments, no audit trail.

### 1.2 Business importance

- **System of record:** the single source of truth for money, inventory, identity, orders.
- **Revenue-path:** checkout down = revenue down by the minute.
- **Legal/compliance:** SOX, PCI-DSS, GDPR, RBI/SEBI audits need exact, traceable records.
- **Feeds everything downstream:** analytics, ML, search, and caches are all derived from OLTP data.

### 1.3 Real-world use cases (with the invariant each must protect)

| Domain | Typical transaction | Invariant that must never break |
| --- | --- | --- |
| Banking / UPI / payments | Debit A, credit B | Total money conserved; no double-spend |
| E-commerce | Place order, reserve stock, charge | Stock never negative; one charge per order |
| Inventory / WMS | Pick, pack, transfer | On-hand = sum of movements |
| Ticketing / airlines / hotels | Book seat/room | One seat, one holder |
| Telecom billing | Usage record, balance deduct | No lost or duplicate CDRs |
| Healthcare (EHR) | Admit, prescribe | Correct patient, auditable history |
| SaaS / ride-hailing / food delivery | Create trip/order, status updates | State machine order respected |

**Industry methods:** capacity planning by peak TPS × safety factor 3–5; SLOs (latency/availability) with error budgets; idempotency keys on every write API so retries don't duplicate (Stripe-style); the **transactional outbox** pattern to publish events reliably with the DB write. **Tools:** PostgreSQL, MySQL/MariaDB (InnoDB), Oracle Database, Microsoft SQL Server, IBM Db2; cloud: Amazon Aurora/RDS, Google Cloud SQL/AlloyDB/Spanner, Azure SQL; NewSQL: CockroachDB, YugabyteDB, TiDB; NoSQL OLTP: DynamoDB, MongoDB, Cassandra (when relational guarantees are relaxed deliberately). Load/benchmark: **pgbench, sysbench, HammerDB (TPC-C), Percona/TPC benchmarks, k6, JMeter**. Observability: Prometheus + Grafana, Datadog, pg_stat_statements, Percona PMM.

---

## 2. OLTP vs. OLAP

### 2.1 Architectural and workload differences

| Dimension | OLTP | OLAP |
| --- | --- | --- |
| Purpose | Run the business | Analyze the business |
| Query shape | `WHERE id = ?` point lookups, tiny writes | Scan millions/billions of rows, GROUP BY, joins, aggregates |
| Columns used | Most columns of a few rows | Few columns of many rows |
| Schema | Normalized (3NF) | Star/snowflake, wide denormalized tables |
| Storage layout | **Row-oriented** | **Column-oriented** |
| Writes | Constant, small, transactional | Bulk batch / streaming loads, rarely updated |
| Consistency | Strict ACID | Often eventual / snapshot at load time |
| History | Current state (plus audit) | Full history, time-variant |
| Optimizes | Latency, concurrency, correctness | Throughput, scan speed, compression |
| Scaling | Vertical first, then replicas/sharding | MPP: scale-out compute, separate storage |

### 2.2 Row vs. column storage (why it matters)

- **Row store** keeps all fields of one record together on a page. Fetching or updating one order = one page read/write. Perfect for OLTP.
- **Column store** keeps each column contiguous and compressed (dictionary, run-length, delta encoding), uses vectorized execution. Summing one column over 1B rows reads only that column.
- **If you run analytics on the row store:** a `SUM(amount)` over 500M rows reads every column of every row, thrashes the buffer pool, evicts hot OLTP pages, takes locks/snapshots long enough to bloat MVCC, and slows checkout for real customers.
- **If you run OLTP on a column store:** each single-row insert/update touches many column segments; high write amplification, weak row-level locking, poor point-lookup latency.

### 2.3 Why keep them separate at all

Different access patterns need opposite physical designs. One engine can't be perfect at both, and analytics must never share resources with revenue-critical traffic. Separation gives **isolation of performance, independent scaling, and different retention/security policies.**

### 2.4 ETL / ELT pipeline

- **ETL:** Extract → Transform (in a separate engine) → Load. Used when the target is weak or data must be cleansed/masked first.
- **ELT:** Extract → Load raw → Transform inside the warehouse (cheap elastic compute). Modern default.
- **Extraction methods (increasing freshness):** full dump → timestamp/incremental query → **log-based CDC** (reads the WAL/binlog; no load on tables, captures deletes) → streaming.
- **Layers:** raw/bronze → cleaned/silver → modeled/gold (medallion), or staging → core → marts.
- **Practices:** idempotent loads, schema-evolution handling, data contracts, data-quality tests, lineage, backfills, late-arriving data handling, slowly changing dimensions (SCD Type 2).

**If the pipeline is missing:** analysts query production directly (outage risk), reports disagree, no history is kept (OLTP overwrites), ML teams scrape APIs, and compliance can't prove what data looked like on a given date.

**Tools**

| Role | Tools |
| --- | --- |
| Analytic stores (OLAP) | Snowflake, Google BigQuery, Amazon Redshift, Databricks (Delta Lake), ClickHouse, Apache Druid, Apache Pinot, Azure Synapse |
| CDC | **Debezium**, AWS DMS, Oracle GoldenGate, Striim, Qlik Replicate, Maxwell |
| Streaming backbone | Apache Kafka, Redpanda, Kinesis, Pub/Sub, Flink |
| EL connectors | Fivetran, Airbyte, Stitch, Meltano, Hevo |
| Transform | **dbt**, Spark, SQLMesh, Dataform |
| Orchestration | Apache Airflow, Dagster, Prefect, Azure Data Factory, AWS Glue |
| Legacy ETL | Informatica, Talend, SSIS, DataStage |
| Quality/lineage | Great Expectations, Soda, dbt tests, OpenLineage, DataHub, Monte Carlo |
| BI | Tableau, Power BI, Looker, Metabase, Superset |

### 2.5 The blurring line: HTAP

Hybrid systems serve both from one platform: TiDB (TiKV+TiFlash), SingleStore, Oracle Database In-Memory, SQL Server columnstore indexes, Aurora zero-ETL to Redshift, AlloyDB columnar engine, Postgres + pg_duckdb. Useful for light real-time analytics; still, heavy enterprise analytics usually stays separate.

---

## 3. Data Modeling for OLTP

### 3.1 Modeling workflow (industry standard)

1. **Requirements & access patterns:** list entities, the top 10 transactions, volumes, and invariants.
2. **Conceptual model:** entities and relationships (business language).
3. **Logical model:** attributes, keys, normalized to 3NF/BCNF.
4. **Physical model:** types, constraints, indexes, partitioning, engine-specific choices.
5. **Review, migrate, evolve:** versioned migrations, never hand edits in production.

**If skipped:** you discover design flaws after data exists; fixing them means painful migrations on live tables, or living with them permanently.

### 3.2 Normalization: reduce redundancy, avoid anomalies

**Anomalies it prevents (the real reason it exists):**

- *Update anomaly:* the customer's address stored on 10,000 orders; change in one place, stale in others.
- *Insert anomaly:* can't record a product until someone orders it.
- *Delete anomaly:* deleting the last order erases the only copy of customer data.

| Form | Rule | Violation example | Fix |
| --- | --- | --- | --- |
| **1NF** | Atomic values, no repeating groups, each row identifiable | `phones = "98xx, 97xx"` in one cell | Separate `phone` rows / child table |
| **2NF** | 1NF + no partial dependency on part of a composite key | `OrderItem(order_id, product_id, product_name)`: name depends only on product_id | Move product_name to `Product` |
| **3NF** | 2NF + no transitive dependency (non-key → non-key) | `Employee(emp_id, dept_id, dept_name)` | Separate `Department` |
| **BCNF** | Every determinant is a candidate key | `Class(student, course, teacher)` where teacher → course | Split into `Teacher(teacher, course)` and `Enrollment(student, teacher)` |

(Beyond: 4NF multivalued dependencies, 5NF join dependencies, and 6NF temporal, rarely required but good to know.)

**Why it suits OLTP:** each fact is stored once, so a write touches one row, locks are tiny, and constraints stay simple. **If absent:** wide duplicated rows, inconsistent copies, larger storage, and wider rows = more I/O and more lock contention.

### 3.3 Deliberate denormalization (the industry exception)

Normalize first, denormalize only with evidence (a measured hot read path). Accepted techniques: cached counters (`order_count`), snapshotting price/address on the order at purchase time (this is *correct*, not redundant: history must not change), materialized views, read models via CQRS, JSONB columns for truly sparse attributes. **Risk if overdone:** drift between copies; must be protected with triggers, transactions, or reconciliation jobs.

### 3.4 ER diagramming

- **Elements:** entities, attributes, relationships, cardinality (1:1, 1:N, M:N, resolved via a junction table), optionality, weak entities, supertype/subtype (single-table, class-table, or concrete-table inheritance).
- **Notations:** Crow's Foot (industry default), Chen, IDEF1X, UML class diagrams.
- **If absent:** teams hold conflicting mental models; onboarding is slow; missing relationships become orphaned data.
- **Tools:** dbdiagram.io (DBML), draw.io, Lucidchart, Miro, ERwin Data Modeler, ER/Studio, Oracle SQL Developer Data Modeler, MySQL Workbench, pgModeler, DbSchema, DBeaver ER view, SchemaSpy (reverse-engineers docs from a live DB), Mermaid ER in docs-as-code.

### 3.5 Schema design for rapid writes (physical decisions)

| Decision | Industry practice | If ignored |
| --- | --- | --- |
| **Primary keys** | Narrow, immutable, unique. Surrogate `BIGINT IDENTITY`, or time-ordered IDs (UUIDv7, ULID, Snowflake IDs) for distributed systems | Random UUIDv4 as clustered PK in InnoDB → page splits, fragmentation, slow inserts; natural keys that change → cascading updates |
| **Foreign keys** | Declare them; index the child column | Orphans; FK checks and parent deletes full-scan child table |
| **Constraints** | NOT NULL, CHECK, UNIQUE, FK, exclusion constraints (Postgres) at DB level | Bad data enters from any buggy app/script; "validated only in app code" always fails eventually |
| **Data types** | Smallest correct type; `NUMERIC/DECIMAL` for money (or integer minor units: paise/cents), `timestamptz` in UTC, enums/lookup tables for states | Float rounding errors in money; timezone bugs; oversized rows |
| **Indexes** | Only those the workload needs; every extra index slows every write | Over-indexing = write amplification; under-indexing = full scans and lock waits |
| **Row width / hot columns** | Keep hot, small columns in main table; push blobs/large text out (TOAST, separate table, object storage) | Fat rows reduce rows per page, cache efficiency, and increase I/O |
| **Hot rows** | Avoid single-row counters/balances updated by everyone; use sharded counters, append-only ledger, or queue | Row-lock convoy: all transactions serialize on one row |
| **Append-only/ledger design** | Immutable events, double-entry (debit/credit lines summing to 0); derive balances | Overwritten history, no audit trail, impossible reconciliation |
| **State machines** | `status` with CHECK/enum + legal transitions, `version` column for optimistic locking | Illegal states (shipped before paid), lost updates |
| **Soft delete / audit** | `deleted_at`, `created_at`, `updated_at`, history tables or temporal tables; partial unique indexes | Irrecoverable deletions, no "who changed what" |
| **Idempotency** | Unique `request_id`/idempotency key column | Retries double-charge |
| **Fill factor, partitioning** | Tune fillfactor on update-heavy tables; time-based partitions for large append tables, easy retention via DROP PARTITION | Page splits/bloat; huge-table vacuum and deletes that lock for hours |
| **Multi-tenancy** | Tenant ID in every key/index (shared schema), schema-per-tenant, or DB-per-tenant; Postgres row-level security | Cross-tenant data leaks, noisy neighbors |
| **Schema evolution** | Backward-compatible expand/contract migrations, online DDL, feature flags | Downtime on `ALTER TABLE`, broken old app versions during deploy |

**Tools for schema lifecycle:** migrations: Flyway, Liquibase, Alembic, Django Migrations, Rails Active Record, Prisma Migrate, Sqitch, Atlas. Online schema change: gh-ost, pt-online-schema-change (Percona), pg_repack, native `ALGORITHM=INPLACE` / `CREATE INDEX CONCURRENTLY`. Linting/CI: squawk, SQLFluff, Atlas lint. Drift/diff: Redgate, Liquibase diff, DBeaver. ORMs: Hibernate/JPA, SQLAlchemy, Entity Framework, Sequelize, Prisma, GORM.

---

## 4. Why only these three topics in Module 1?

1. **Dependency order.** You can't understand ACID, locks, or indexes (Modules 2–4) until you know *what* is being protected (OLTP workload), *why* it's isolated from analytics (OLTP vs OLAP), and *what shape the data has* (modeling). It is the "nouns" before the "verbs."
2. **The three questions of foundations:** *What is the workload? What is it not? How is the data structured for it?* Everything else is mechanism, and mechanism is Modules 2–5.
3. **Design choices here are the most expensive to reverse.** Schema, keys, and system boundaries are decided once and live for years; transaction and tuning knobs can be changed later.
4. **Deliberately out of scope here** (taught later): transaction semantics/WAL (M2), locking/MVCC (M3), index internals/query plans (M4), replication/sharding/2PC (M5).
5. **Adjacent gaps worth self-study:** dimensional modeling (Kimball star schemas, Data Vault) for the OLAP side; NoSQL data modeling (single-table design in DynamoDB); event sourcing/CQRS; data governance, PII/GDPR handling; and security basics (least privilege, encryption at rest/in transit).

---

## 5. Self-check questions

1. Why does a 500M-row `SUM()` on the production row store hurt checkout latency?
2. Give one update, insert, and delete anomaly from a single un-normalized `Orders` table.
3. When is it *correct* to copy a price into an order line?
4. Why is UUIDv4 as a clustered key harmful in InnoDB, and what replaces it?
5. Why is log-based CDC preferred over timestamp polling?
6. Design a ledger for a wallet that avoids a single hot balance row.

## 6. Suggested hands-on

Install PostgreSQL; model a shop (customers, products, orders, order_items, payments) in dbdiagram.io; migrate with Flyway; load with pgbench; stream changes with Debezium → Kafka → ClickHouse/BigQuery; build a dbt mart and compare a report run on both sides.