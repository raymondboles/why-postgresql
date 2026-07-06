# PostgreSQL

## Getting Started

Runnable SQL examples accompany the concepts below. Each file maps to a section in this README.

### Folder structure

```text
.
├── README.md
├── docs/             # In-depth concept guides linked from the tables below
├── scripts/
│   ├── setup.sql     # Creates the `main` schema + shared tables, seeds demo data
│   └── teardown.sql  # Drops the `main` schema (tear down everything)
├── beginner/         # Runnable examples for the Beginner concepts below
├── intermediate/     # Runnable examples for the Intermediate concepts below
└── advanced/         # Runnable examples for the Advanced concepts below
```

### Running the examples

Scripts use a dedicated `main` schema so they never touch your existing objects.

```bash
createdb sandbox                       # create the database (first time only)

psql sandbox -f scripts/setup.sql      # create schema + shared tables + seed data

psql sandbox -f <level>/<example>.sql  # run a beginner example

psql sandbox -f scripts/teardown.sql   # tear down when you're done
```

`setup.sql` is re-runnable - it truncates and reseeds, so you can reset to a known state at any time.

## Context

Before choosing PostgreSQL, you need to understand where it sits in the database landscape. These sections establish the
fundamentals: what kind of workloads exist, how data platforms differ, and why relational databases (specifically
PostgreSQL) remain the default starting point for most applications.

### OLTP vs OLAP

|          | OLTP                                              | OLAP                                              |
| -------- | ------------------------------------------------- | ------------------------------------------------- |
| Purpose  | Process transactions                              | Analyze data                                      |
| Queries  | Short, frequent, row-level (INSERT/UPDATE/DELETE) | Long, complex, column-level (aggregations, scans) |
| Schema   | Normalized (3NF)                                  | Denormalized (star/snowflake)                     |
| Latency  | Milliseconds                                      | Seconds to minutes                                |
| Examples | PostgreSQL, MySQL, SQL Server                     | BigQuery, Snowflake, ClickHouse                    |

PostgreSQL is OLTP-first, but handles moderate OLAP with window functions, Common Tables Expressions (CTEs), parallel
queries, and materialized views. For heavy analytics, offload to a dedicated OLAP system.

### Modern Data Platform

- Data Lake - Raw, unstructured storage (S3, HDFS). Cheap, flexible, schema-on-read. Poor query performance without
  processing.
- Data Warehouse - Structured, optimized for analytics. Schema-on-write. Fast queries, expensive ingestion (BigQuery,
  Redshift, Snowflake).
- Lakehouse - Combines both: cheap lake storage with warehouse query performance and schema enforcement (Delta Lake,
  Apache Iceberg).

Where PostgreSQL fits: It's your OLTP source of truth. Data flows from PostgreSQL into lakes/warehouses via CDC (logical
replication, Debezium) for analytics.

### NoSQL vs SQL

|               | SQL (Relational)                         | NoSQL                                                            |
| ------------- | ---------------------------------------- | ---------------------------------------------------------------- |
| Schema        | Fixed, enforced                          | Flexible, dynamic                                                |
| Consistency   | Strong (ACID)                            | Often eventual (BASE)                                            |
| Scaling       | Vertical primarily                       | Horizontal primarily                                             |
| Relationships | Native (JOINs, FKs)                      | Application-managed                                              |
| Best for      | Transactions, integrity, complex queries | High write throughput, unstructured data, simple access patterns |

PostgreSQL blurs the line: JSONB gives you document-store flexibility, arrays give you list storage, hstore gives you
key-value - all with ACID guarantees. You often don't need a separate NoSQL database.

When you still need NoSQL over JSONB:

- Millions of writes/second - JSONB is still row-based with MVCC overhead; a key-value store or wide-column DB handles
  pure write throughput better
- Sub-millisecond reads at massive scale - in-memory stores win when you need single-digit ms latency across millions of
  concurrent reads
- Horizontal sharding by default - NoSQL distributes data across nodes natively; PostgreSQL requires extensions or
  application-level sharding
- Graph traversals - deeply nested relationship queries (6+ hops) perform poorly with JOINs; graph databases are
  purpose-built for this
- Time-series at extreme ingestion - billions of data points/day with auto-downsampling, retention policies, and
  compression. TimescaleDB helps but purpose-built time-series databases handle this natively at higher scale
- Unstructured search with relevance scoring - facets, fuzzy matching, typo tolerance, synonyms, weighted multi-field
  ranking across millions of docs. PostgreSQL's tsvector handles basic search but breaks down for search-product UX
  (autocomplete, "did you mean", complex relevance tuning)

### ACID vs BASE

ACID (SQL databases) - Atomicity (all or nothing), Consistency (valid state to valid state), Isolation (concurrent
transactions don't interfere), Durability (committed = permanent). Every transaction either fully completes or fully
rolls back.

BASE (NoSQL databases) - Basically Available, Soft state, Eventually consistent. The system always accepts writes but
reads may return stale data until replicas converge. Trades consistency for availability and partition tolerance.

PostgreSQL is ACID. It will reject a write rather than risk inconsistency. When a transaction commits, the data is
durable and immediately visible to all readers.

### CAP

CAP theorem - a distributed system can only guarantee two of three: Consistency (every read sees the latest write),
Availability (every request gets a response), Partition tolerance (system works despite network splits). Since network
partitions are unavoidable, the real choice is CP (consistent but may reject requests) vs AP (available but may return
stale data). PostgreSQL is CP. Most NoSQL databases choose AP.

Making PostgreSQL more available while staying CP: multiple synchronous replicas with `ANY N (...)` so the primary
doesn't hang if one standby dies. Streaming replicas for read availability - reads survive primary failure via failover
(Patroni, `pg_auto_failover`). Relaxing `synchronous_commit` trades durability of the last few transactions for write
availability under pressure. Connection pooling (PgBouncer) keeps the system responsive under connection storms. These
don't make PostgreSQL AP - it still refuses inconsistent writes - but they push availability as high as possible within
CP constraints.

## What's PostgreSQL?

An open-source object-relational database. Process-per-connection architecture, MVCC concurrency, WAL-based durability.
Runs on every major OS. Closest to the SQL standard of any database. Extensible by design - custom types, operators,
index methods, and procedural languages.

## Why PostgreSQL?

- One database, many workloads - OLTP, document store (JSONB), full-text search, geospatial (PostGIS), vector search
  (pgvector), time-series (TimescaleDB), queue (SKIP LOCKED), key-value (hstore)
- Correctness by default - ACID, transactional DDL, strong typing, enforced constraints
- Extensible - Extensions, custom types, FDWs, PL/pgSQL - adapt without switching databases
- No vendor lock-in - Open-source, PostgreSQL License, runs anywhere (self-hosted, RDS, Cloud SQL, AlloyDB, Neon,
  Supabase)
- SQL standard - Skills transfer, less proprietary syntax to unlearn
- Battle-tested at scale - Instagram, Spotify, Discord, Shopify

### Trade-offs

#### vs MySQL

##### MySQL pros

- Simpler to run out-of-the-box - less operational overhead
- Lighter connection model - threads vs processes at very high connection counts
- Simpler replication setup - basic primary-replica with minimal configuration
- Auto-indexes on foreign keys - one less thing to forget

##### MySQL cons

- Limited SQL features - no CTEs (until 8.0), no LATERAL, no DISTINCT ON, weaker window functions
- Permissive defaults - silent data truncation, implicit type coercion, zero-dates
- No transactional DDL - can't rollback ALTER TABLE
- Weaker JSONB - JSON support exists but lacks GIN indexing on arbitrary paths, no @> containment operator, fewer
  operators. Gap narrowed in 8.0.17+ (multi-valued indexes, JSON_TABLE) but PostgreSQL still leads
- No partial indexes, no expression indexes, no EXCLUDE constraints

#### vs SQL Server

##### SQL Server pros

- Built-in GUI tooling (SSMS) - query editor, execution plans, visual debugging
- Native job scheduler (SQL Agent) - no external cron/scheduler needed
- Integrated BI stack (SSRS/SSIS) - reporting and ETL out of the box
- Better enterprise support contracts - Microsoft-backed SLAs
- Tight Active Directory/.NET/Azure integration

##### SQL Server cons

- Licensing cost - per-core pricing ($$$$), PostgreSQL is free
- Vendor lock-in - tied to Microsoft ecosystem for full benefit
- Weaker extensibility - no equivalent to PostgreSQL's extension system
- Less SQL standard compliance - T-SQL diverges from standard SQL
- Limited open-source tooling and community extensions

## Levels

Core concepts used throughout this doc:

| Term                 | What it is                                | What it does                                                                                   |
| -------------------- | ----------------------------------------- | ---------------------------------------------------------------------------------------------- |
| MVCC                 | Multi-Version Concurrency Control         | Keeps multiple row versions so readers and writers never block each other                      |
| WAL                  | Write-Ahead Log                           | Records every change to a durable log before the data files, enabling recovery and replication |
| VACUUM               | Storage maintenance command               | Reclaims dead tuples left by MVCC and prevents transaction-ID wraparound                       |
| DDL                  | Data Definition Language                  | CREATE, ALTER, DROP - changes schema; transactional so it can roll back                        |
| DML                  | Data Manipulation Language                | INSERT, UPDATE, DELETE - adds, changes, and removes rows in your tables                        |
| PITR                 | Point-In-Time Recovery                    | Replays WAL to restore the database to any past second                                         |
| GIN                  | Generalized Inverted Index                | Indexes the elements inside a value for fast containment lookups (arrays, JSONB, full-text)    |
| RLS                  | Row-Level Security                        | Filters which rows each user can read or modify via per-row policies                           |
| CDC                  | Change Data Capture                       | Streams row-level changes out to other systems                                                 |
| TOAST                | The Oversized-Attribute Storage Technique | Compresses and stores large values (>2KB) off-row                                              |
| FDW                  | Foreign Data Wrapper                      | Exposes external data sources as queryable tables                                              |
| BRIN                 | Block Range Index                         | Stores per-block min/max for cheap scans of ordered data                                       |
| GiST                 | Generalized Search Tree                   | Indexes geometry, ranges, and nearest-neighbor queries                                         |
| GUCs                 | Grand Unified Configuration               | Holds the parameters that tune server behavior                                                 |
| JIT                  | Just-In-Time compilation                  | Compiles hot query expressions to machine code at runtime                                      |
| RAG                  | Retrieval-Augmented Generation            | Uses vector search to feed context to an LLM                                                   |

### Beginner

Database Fundamentals:

- Transactions - atomic operations, commit/rollback
- Constraints - enforcing data integrity (NOT NULL, UNIQUE, CHECK, foreign keys)
- Indexes - how databases find data fast
- JOINs - combining related tables
- Aggregates and GROUP BY - summarizing data

Keys:

- Primary key - uniquely identifies each row (SERIAL, BIGSERIAL, UUID)
- Foreign key - references primary key in another table, enforces referential integrity
- Unique key - ensures no duplicate values in a column (like primary key but allows NULL, multiple per table)
- Composite key - primary or unique key spanning multiple columns
- Natural key - real-world identifier as primary key (email, SSN) - fragile if business rules change
- Surrogate key - system-generated identifier with no business meaning (BIGSERIAL, UUID) - preferred default

Connecting and Tooling:

- pg_hba.conf - connection authentication file (no equivalent in MySQL/SQL Server, you will hit this first)
- Connection cost - 10MB+ per connection vs lightweight in MySQL/SQL Server (why pooling is mandatory)
- psql vs mysql/sqlcmd - command-line tools and meta-commands
- psql meta-commands - \d, \dt, \l, \c, \x, \timing
- `pg_dump` and `pg_restore` - vs mysqldump vs bak files

Types and Schema:

- Primary key choice - SERIAL (int4, max 2.1B) vs BIGSERIAL (int8) vs UUID (`gen_random_uuid()`). Use BIGSERIAL by default,
  UUID when you need distributed ID generation
- SERIAL vs AUTO_INCREMENT vs IDENTITY - auto-increment differences
- TEXT vs VARCHAR vs NVARCHAR - text storage differences
- TIMESTAMPTZ vs DATETIME vs DATETIME2 - timezone handling
- Boolean type - true/false vs 0/1 vs bit
- JSONB - binary, indexable JSON storage
- Array types - native array support
- ENUM types - custom enumerated types
- Sequences - explicit sequence objects, shareable
- Schema organization - database -> schema -> table

SQL Differences:

- RETURNING clause - get values from INSERT/UPDATE/DELETE
- INSERT ON CONFLICT - upsert vs INSERT IGNORE vs MERGE
- LATERAL joins - vs CROSS APPLY or not available
- DISTINCT ON - PostgreSQL unique feature
- LIMIT/OFFSET vs TOP/OFFSET-FETCH
- Case sensitivity - lowercase default, quotes preserve case

Behavior:

- Transactional DDL - can rollback ALTER TABLE
- MVCC - no read locks, different locking model

Patterns:

- PostgreSQL as a queue - SKIP LOCKED pattern (vs RabbitMQ/SQS)
- PostgreSQL as a key-value store - hstore and JSONB (vs Redis/DynamoDB)

### Intermediate

PostgreSQL Transaction Features:

- Transactional DDL - ALTER TABLE in transactions (SQL Server yes, MySQL no)
- Isolation levels - PostgreSQL defaults vs SQL Server snapshot isolation
- MVCC internals - readers don't block writers vs SQL Server row versioning
- Deadlocks - PostgreSQL-specific patterns vs SQL Server
- Advisory locks - application-level locking unique to PostgreSQL (vs using Redis for same purpose)
- `lock_timeout` and `statement_timeout` - prevent indefinite waits in production (default is 0 = infinite, causes connection
  pool exhaustion)
- Serialization failures require app retry - SERIALIZABLE/REPEATABLE READ throw errors (40001) apps must handle, no
  auto-retry

PostgreSQL-Specific Constraints:

- Range types - tstzrange, int4range, numrange (needed for EXCLUDE)
- EXCLUDE constraints - prevent overlapping ranges (PostgreSQL unique)
- CHECK constraints - fully functional (vs MySQL limited, SQL Server full)

Primary Key Performance:

- Random UUID write amplification - UUIDv4 causes B-tree page splits and index bloat on heap tables. UUIDv7
  (time-ordered, PostgreSQL 17+) fixes this.
- Logical replication requires primary key - tables without PK can't replicate UPDATE/DELETE, fails silently or with
  cryptic errors
- Partitioning forces key design - partition key must be part of any UNIQUE constraint, can't enforce UNIQUE(email)
  alone if partitioned by tenant_id
- Integer PKs and index size - int4 PK = 4 bytes/row in every index vs UUID = 16 bytes/row. At billions of rows, this
  multiplies across all indexes

PostgreSQL Indexing:

- B-tree - default index type
- GIN indexes - for JSONB, arrays, full-text search
- GiST indexes - for geospatial, range types
- BRIN indexes - for time-series data
- Partial indexes - PostgreSQL unique feature
- Expression indexes - index computed values
- Multi-column index order - leftmost columns must match query filters

PostgreSQL Views:

- Views vs Materialized Views - precomputed query results
- REFRESH MATERIALIZED VIEW CONCURRENTLY - no lock refresh
- When to use materialized views vs indexes

PostgreSQL Full-Text Search:

- tsvector and tsquery types
- `to_tsvector` and `to_tsquery` functions
- GIN indexes for text search

PostgreSQL-Specific Operations:

- COPY - bulk data loading (vs SQL Server BULK INSERT)
- VACUUM vs VACUUM FULL - FULL takes exclusive lock and rewrites table, dangerous in production
- ANALYZE - updating statistics for query planner
- Foreign key no-auto-index - PostgreSQL does NOT auto-create indexes on FK columns (MySQL does, silent performance
  killer on parent DELETE/UPDATE)
- `pg_dump` vs `pg_dumpall`
- Connection pooling with PgBouncer - why PostgreSQL needs it more
- Roles and grants - different from MySQL users
- ALTER DEFAULT PRIVILEGES - new tables don't inherit grants, must set defaults per schema (MySQL GRANT covers future
  tables)
- Concurrent index creation - CREATE INDEX CONCURRENTLY
- Schema migrations - adding columns, NOT NULL transitions
- ALTER TABLE performance characteristics
- When to use raw SQL over ORM - ORMs can't express JSONB ops, CTEs, RETURNING, ON CONFLICT, LATERAL

### Advanced

Advanced PostgreSQL SQL:

- Window functions - PostgreSQL pioneered these
- Recursive CTEs - more powerful than MySQL
- LATERAL joins - correlated subqueries as joins
- INSERT ON CONFLICT - complex upsert targets
- GROUPING SETS, CUBE, ROLLUP - multi-dimensional aggregation in one query
- Array operations - array_agg and array functions
- JSONB operators - ->>, ->, @>, ?

PostgreSQL WAL and Replication:

- WAL architecture - Write-Ahead Log fundamentals (unique implementation)
- Streaming replication - vs MySQL binlog, SQL Server Always On
- Logical replication - selective, version-independent vs SQL Server transactional
- Replication slots and WAL retention
- Patroni and `pg_auto_failover` - vs SQL Server availability groups
- Replication lag monitoring - `pg_stat_replication`, `pg_wal_lsn_diff` (no dashboard like SQL Server Always On)
- Conflict resolution in logical replication - no built-in resolution, must handle manually
- Logical replication does not replicate DDL - must apply schema changes manually (MySQL/SQL Server include DDL)
- Logical replication requires replica identity - tables without primary key can't replicate UPDATE/DELETE, fails
  silently or with cryptic errors

PostgreSQL Performance Internals:

- Query planner - differs from MySQL/SQL Server optimizers
- EXPLAIN ANALYZE - buffers, timing vs SQL Server execution plans
- Statistics system - pg_statistic, ANALYZE vs SQL Server statistics
- Vacuum internals - why PostgreSQL needs it, others don't
- Autovacuum tuning - no equivalent in MySQL/SQL Server
- Transaction ID wraparound - refuses ALL writes if not vacuumed (unique catastrophic failure mode)
- TOAST - The Oversized-Attribute Storage Technique (unique)
- Table and index bloat - PostgreSQL-specific problem
- `pg_stat_statements` - vs SQL Server Query Store
- JIT compilation - PostgreSQL 11+ unique feature
- Checkpoint tuning - `checkpoint_timeout`, `max_wal_size` (defaults cause I/O spikes on write-heavy workloads)
- Prepared statement plan caching - PREPARE/DEALLOCATE, wrong plans with skewed data distributions
- Lock modes (8 types) - what DDL blocks what DML in production (AccessExclusiveLock vs ShareLock vs RowExclusiveLock)
- `pg_stat_activity` - find running queries, blocking chains, idle connections (different from `pg_stat_statements`
  which is historical)

PostgreSQL-Specific Architecture:

- Declarative partitioning - PostgreSQL 10+ vs MySQL partitioning
- Row-level security (RLS) - PostgreSQL unique feature
- Multi-tenant patterns - RLS and schemas
- Foreign Data Wrappers - query other databases
- Parallel query execution - automatic for large scans, tunable via `max_parallel_workers`

PostgreSQL Memory:

- shared_buffers - vs MySQL buffer pool, SQL Server buffer pool
- work_mem - vs SQL Server memory grants
- `maintenance_work_mem` - for VACUUM and CREATE INDEX
- `effective_cache_size` - hint parameter
- Connection memory overhead - why pooling matters more

PostgreSQL Extensibility:

- Extension system - vs MySQL plugins, SQL Server assemblies
- PL/pgSQL - like Oracle PL/SQL and SQL Server T-SQL
- LISTEN/NOTIFY - pub-sub patterns (unique)

PostgreSQL Enterprise:

- Point-in-time recovery with WAL archiving
- pg_basebackup - physical backup strategy
- Upgrade strategies - pg_upgrade, logical replication
- Disaster recovery testing - PITR restore, failover simulation, recovery time validation
- Zero-downtime schema changes
- Connection pooling - PgBouncer transaction vs session mode
- Security hardening - pg_hba.conf, SSL, SCRAM-SHA-256
- `pgaudit` extension - no built-in audit trail (SQL Server/MySQL have native audit, compliance requires this)
- PostGIS - geospatial industry standard
- Citus - distributed PostgreSQL
- pgvector - vector similarity search

PostgreSQL Architecture Patterns:

- Separation of storage and compute - Neon, AlloyDB, Aurora (vs traditional PostgreSQL)
- Sharding - Citus, application-level, FDW-based (trade-offs and when)
- Multi-region - logical replication across regions, conflict resolution, latency
- CDC - logical replication as change data capture (vs Debezium, SQL Server CDC)
- As a vector store - pgvector extension, HNSW/IVFFlat indexes for RAG
- Backup replication - streaming replica as hot standby for reads and backup

## Constraints

- Table bloat - dead tuples accumulate, table grows 2-10x actual data. Workaround: autovacuum tuning, pg_repack
- Transaction ID wraparound - 2 billion TX limit, database refuses ALL writes. Workaround: aggressive vacuum, monitor
  xid age
- Single-node write throughput - one primary, WAL fsync ceiling. Workaround: batching, synchronous_commit off, Citus
  sharding
- Table size without partitioning - billions of rows = slow scans, VACUUM takes hours. Workaround: declarative
  partitioning
- Index size exceeds RAM - random I/O spikes when indexes don't fit shared_buffers. Workaround: partial indexes, BRIN,
  partitioning
- Lock contention - hot rows, DDL blocks all DML (AccessExclusiveLock). Workaround: lock_timeout, retry logic, CREATE
  INDEX CONCURRENTLY
- Replication lag - replicas fall behind under heavy writes, stale reads. Workaround: `synchronous_commit = remote_apply`
  for critical reads, multiple replicas
- Vacuum can't keep up - autovacuum workers saturated on write-heavy tables. Workaround: per-table autovacuum settings,
  more workers, partitioning
- Checkpoint I/O spikes - periodic WAL flush causes latency spikes. Workaround: `checkpoint_completion_target`, increase
  `max_wal_size`
- `work_mem` explosion - `work_mem` x active queries x sorts = OOM. Workaround: lower `work_mem`, fewer connections, optimize
  queries
- Long-running transactions - hold back VACUUM, cause bloat cascade. Workaround: `statement_timeout`,
  `idle_in_transaction_session_timeout`
- Heap table no clustered index - range scans hit random pages unlike SQL Server/InnoDB. Workaround: covering indexes,
  CLUSTER (takes lock), pg_repack
- No native query hints - planner picks bad plans, can't override. Workaround: `pg_hint_plan` extension, optimizer GUCs
- Sequence overflow - SERIAL = int4 max 2.1 billion, silently halts inserts. Workaround: use BIGSERIAL from day one
- Subtransaction overflow - more than 64 savepoints per TX degrades ALL connections (SubXID cache). Workaround: avoid
  deep subtransactions, restructure batch logic
- Dead connection detection - TCP keepalive default too high, dead clients hold slots for hours. Workaround:
  `tcp_keepalives_idle`, `idle_in_transaction_session_timeout`
- Schema-per-tenant catalog bloat - `pg_class` grows massive at 10,000+ tenants, query planning slows, `pg_dump` takes hours.
  Workaround: shared-table with RLS instead
- Partitioning breaks global unique constraints - can't enforce UNIQUE(email) if partitioned by tenant_id, must include
  partition key. Workaround: separate global lookup table or app-layer enforcement
- Replication slot unbounded WAL retention - disconnected slot prevents ALL WAL cleanup until disk fills, crashing
  cluster. Workaround: `max_slot_wal_keep_size`, monitor slot lag
- Synchronous replication commit hang - all sync standbys die, primary hangs ALL writes forever with no timeout (MySQL
  semi-sync falls back to async). Workaround: ANY N (...) with multiple standbys

## Migration

### From MySQL

- || is concatenation not OR - WHERE active || pending silently produces string concatenation instead of boolean logic
- No UNSIGNED integers - MySQL INT UNSIGNED (0 to 4.2B) overflows PostgreSQL integer (max 2.1B), need bigint
- Backslash not an escape by default - '\n' is literal two characters, need E'\n' or `standard_conforming_strings = off`
- Backticks don't work - MySQL `table` syntax errors, use "table" for reserved words
- Integer division returns integer - MySQL SELECT 5/2 returns 2.5, PostgreSQL returns 2, financial calculations silently
  wrong
- Division by zero is a hard error - MySQL returns NULL, PostgreSQL throws error, need NULLIF(denominator, 0) guard

### From Oracle

- Empty string is NOT NULL - Oracle treats '' as NULL, PostgreSQL treats '' as empty string, IS NULL checks silently
  miss data
- DATE loses time component - Oracle DATE includes hours/minutes/seconds, PostgreSQL DATE truncates time with no warning
- Transaction abort blocks all commands - after one error, every subsequent command fails until ROLLBACK (Oracle
  continues)
- Auto-commit by default - Oracle wraps in implicit transaction until COMMIT, PostgreSQL commits each statement
  immediately (partial writes persist if no explicit BEGIN)

#### From SQL Server

- No NEWID() - use `gen_random_uuid()` for UUID generation
- No ISNULL() - use COALESCE() (which is standard SQL)

## How PostgreSQL Thrives in the Agentic Era?

- Fintech loan agent - retrieves policy docs (pgvector), reasons over applicant data (JSONB), writes decision, logs
  audit trail - one transaction. Anything fails, whole thing rolls back. No partial state. With a separate vector
  database + document store + audit service, that atomicity doesn't exist.

- Multi-tenant AI support - RLS ensures agents physically can't read other orgs' data, even with application bugs. SKIP
  LOCKED for task claiming, LISTEN/NOTIFY to wake idle agents. Vector search, queuing, sessions, isolation, audit on one
  managed PostgreSQL instance (`$$`) instead of five separate services (`$$$$$`).

- Legal contract analysis - agent finds clauses (pgvector), updates metadata, retries idempotently (ON CONFLICT).
  Concurrent modification? Transaction rolls back cleanly. Compliance asks "what did the agent see at decision time?"
  tstzrange reconstructs the exact state.

- Healthcare prompt injection incident - agents go rogue for 20 minutes. PITR restores to the minute before. WAL
  archiving was already running for HIPAA. Re-process legitimate work from the task queue. One command, not a week of
  forensics.

One database. One connection pool. One backup. One security perimeter. One query language to hire for. `$$` vs `$$$$$`
for the multi-vendor stack.
