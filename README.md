# PostgreSQL

## Context

_Coming soon..._

### OLTP vs OLAP

_Coming soon..._

### NoSQL vs SQL

_Coming soon..._

## What's PostgreSQL?

_Coming soon..._

## Why PostgreSQL?

### Beginner

Assumes: Basic SQL knowledge from MySQL, SQL Server, or SQLite

If you've never used a database before:

- Transactions - atomic operations, commit/rollback
- Constraints - enforcing data integrity (NOT NULL, UNIQUE, CHECK, foreign keys)
- Indexes - how databases find data fast
- JOINs - combining related tables
- Aggregates and GROUP BY - summarizing data

PostgreSQL vs Other Databases:

- pg_hba.conf - connection authentication file (no equivalent in MySQL/SQL Server, you will hit this first)
- Connection cost - 10MB+ per connection vs lightweight in MySQL/SQL Server (why pooling is mandatory)
- psql vs mysql/sqlcmd - command-line tools and meta-commands
- SERIAL vs AUTO_INCREMENT vs IDENTITY - auto-increment differences
- TEXT vs VARCHAR vs NVARCHAR - text storage differences
- TIMESTAMPTZ vs DATETIME vs DATETIME2 - timezone handling
- Schema organization - database → schema → table
- Boolean type - true/false vs 0/1 vs bit
- Case sensitivity - lowercase default, quotes preserve case
- RETURNING clause - get values from INSERT/UPDATE/DELETE
- INSERT ON CONFLICT - upsert vs INSERT IGNORE vs MERGE
- LATERAL joins - vs CROSS APPLY or not available
- DISTINCT ON - PostgreSQL unique feature
- Dollar-quoted strings - $$string$$ vs N'string'
- LIMIT/OFFSET vs TOP/OFFSET-FETCH
- Transactional DDL - can rollback ALTER TABLE
- MVCC - no read locks, different locking model
- Sequences - explicit sequence objects, shareable
- psql meta-commands - \d, \dt, \l, \c, \x, \timing
- pg_dump and pg_restore - vs mysqldump vs bak files
- JSONB - binary, indexable JSON storage
- Array types - native array support
- ENUM types - custom enumerated types
- PostgreSQL as a queue - SKIP LOCKED pattern (vs RabbitMQ/SQS)
- PostgreSQL as a key-value store - hstore and JSONB (vs Redis/DynamoDB)

### Intermediate

PostgreSQL Transaction Features:

- Transactional DDL - ALTER TABLE in transactions (SQL Server yes, MySQL no)
- Isolation levels - PostgreSQL defaults vs SQL Server snapshot isolation
- MVCC internals - readers don't block writers vs SQL Server row versioning
- Deadlocks - PostgreSQL-specific patterns vs SQL Server
- Savepoints and subtransactions
- Advisory locks - application-level locking unique to PostgreSQL (engineers use Redis without knowing this exists)
- lock_timeout and statement_timeout - prevent indefinite waits in production (default is 0 = infinite, causes connection pool exhaustion)
- Serialization failures require app retry - SERIALIZABLE/REPEATABLE READ throw errors (40001) apps must handle, no auto-retry

PostgreSQL-Specific Constraints:

- Range types - tstzrange, int4range, numrange (needed for EXCLUDE)
- EXCLUDE constraints - prevent overlapping ranges (PostgreSQL unique)
- CHECK constraints - fully functional (vs MySQL limited, SQL Server full)
- Deferrable constraints - check at transaction end (not in MySQL/SQL Server)

Advanced PostgreSQL SQL:

- Window functions - PostgreSQL pioneered these
- Recursive CTEs - more powerful than MySQL
- LATERAL joins - correlated subqueries as joins
- INSERT ON CONFLICT - complex upsert targets
- FILTER clause - for aggregates
- GROUPING SETS, CUBE, ROLLUP
- Array operations - array_agg and array functions
- JSONB operators - ->>, ->, @>, ?

PostgreSQL Indexing:

- B-tree - default index type
- GIN indexes - for JSONB, arrays, full-text search
- GiST indexes - for geospatial, range types
- BRIN indexes - for time-series data
- Hash indexes - equality only
- Partial indexes - PostgreSQL unique feature
- Expression indexes - index computed values
- Multi-column index order matters

PostgreSQL Views:

- Views vs Materialized Views - precomputed query results
- REFRESH MATERIALIZED VIEW CONCURRENTLY - no lock refresh
- When to use materialized views vs indexes

PostgreSQL Full-Text Search:

- tsvector and tsquery types
- to_tsvector and to_tsquery functions
- GIN indexes for text search
- ts_rank for ranking
- Text search configuration for languages

PostgreSQL-Specific Operations:

- COPY - bulk data loading (vs SQL Server BULK INSERT)
- VACUUM vs VACUUM FULL - FULL takes exclusive lock and rewrites table, dangerous in production
- ANALYZE - updating statistics for query planner
- Foreign key no-auto-index - PostgreSQL does NOT auto-create indexes on FK columns (MySQL does, silent performance killer on parent DELETE/UPDATE)
- pg_dump vs pg_dumpall
- Connection pooling with PgBouncer - why PostgreSQL needs it more
- Roles and grants - different from MySQL users
- ALTER DEFAULT PRIVILEGES - new tables don't inherit grants, must set defaults per schema (MySQL GRANT covers future tables, engineers always get burned)
- Concurrent index creation - CREATE INDEX CONCURRENTLY
- Schema migrations - adding columns, NOT NULL transitions
- ALTER TABLE performance characteristics
- When to use raw SQL over ORM - ORMs can't express PostgreSQL-specific features (JSONB ops, CTEs, RETURNING, ON CONFLICT, LATERAL, array ops, window functions)

PostgreSQL as a Platform:

- As a data mart - materialized views + GROUPING SETS for analytics workloads

### Advanced

PostgreSQL WAL and Replication:

- WAL architecture - Write-Ahead Log fundamentals (unique implementation)
- Streaming replication - vs MySQL binlog, SQL Server Always On
- Logical replication - selective, version-independent vs SQL Server transactional
- Replication slots and WAL retention
- Patroni and pg_auto_failover - vs SQL Server availability groups
- Replication lag monitoring - pg_stat_replication, pg_wal_lsn_diff (no dashboard like SQL Server Always On)
- Conflict resolution in logical replication
- Logical replication does not replicate DDL - schema changes must be applied manually on subscribers (MySQL binlog and SQL Server transactional replication include DDL)
- Logical replication requires replica identity - tables without primary key can't replicate UPDATE/DELETE, fails silently or with cryptic errors

PostgreSQL Performance Internals:

- Query planner - differs from MySQL/SQL Server optimizers
- EXPLAIN ANALYZE - buffers, timing vs SQL Server execution plans
- Statistics system - pg_statistic, ANALYZE vs SQL Server statistics
- Vacuum internals - why PostgreSQL needs it, others don't
- Autovacuum tuning - no equivalent in MySQL/SQL Server
- Transaction ID wraparound - PostgreSQL refuses ALL writes if not vacuumed, only database with this catastrophic failure mode
- TOAST - The Oversized-Attribute Storage Technique (unique)
- Table and index bloat - PostgreSQL-specific problem
- pg_stat_statements - vs SQL Server Query Store
- JIT compilation - PostgreSQL 11+ unique feature
- Checkpoint tuning - checkpoint_timeout, max_wal_size (defaults cause I/O spikes on write-heavy workloads)
- Prepared statement plan caching - PREPARE/DEALLOCATE, wrong plans with skewed data distributions
- Lock modes (8 types) - what DDL blocks what DML in production (AccessExclusiveLock vs ShareLock vs RowExclusiveLock)
- pg_stat_activity - find running queries, blocking chains, idle connections (different from pg_stat_statements which is historical)
- synchronous_commit per-transaction - trade durability for speed per transaction (unique per-transaction control, not in MySQL/SQL Server)

PostgreSQL-Specific Architecture:

- Declarative partitioning - PostgreSQL 10+ vs MySQL partitioning
- Table inheritance - older partitioning method
- Row-level security (RLS) - PostgreSQL unique feature
- Multi-tenant patterns - RLS and schemas
- Foreign Data Wrappers - query other databases
- Parallel query execution

PostgreSQL Memory:

- shared_buffers - vs MySQL buffer pool, SQL Server buffer pool
- work_mem - vs SQL Server memory grants
- maintenance_work_mem - for VACUUM and CREATE INDEX
- effective_cache_size - hint parameter
- Connection memory overhead - why pooling matters more

PostgreSQL Extensibility:

- Extension system - vs MySQL plugins, SQL Server assemblies
- PL/pgSQL - like Oracle PL/SQL and SQL Server T-SQL
- PL/Python, PL/Perl, PL/V8 - vs SQL Server CLR
- Writing extensions in C - vs SQL Server extended stored procedures
- Background workers - unique to PostgreSQL
- Event triggers - DDL triggers
- LISTEN/NOTIFY - pub-sub patterns (unique)
- Custom operators and types

PostgreSQL Enterprise:

- Point-in-time recovery with WAL archiving
- pg_basebackup - physical backup strategy
- Upgrade strategies - pg_upgrade, logical replication
- Disaster recovery testing
- Zero-downtime schema changes
- Connection pooling - pgBouncer transaction vs session mode
- Security hardening - pg_hba.conf, SSL, SCRAM-SHA-256
- pg_audit extension - PostgreSQL has no built-in audit trail (SQL Server has native Audit, MySQL has audit plugin, compliance requires this)
- PostGIS - geospatial industry standard
- TimescaleDB - time-series on PostgreSQL
- Citus - distributed PostgreSQL
- pgvector - vector similarity search

PostgreSQL Architecture Patterns:

- Separation of storage and compute - Neon, AlloyDB, Aurora (vs traditional PostgreSQL)
- Sharding - Citus, application-level, FDW-based (trade-offs and when)
- Multi-region - logical replication across regions, conflict resolution, latency
- CDC - logical replication as change data capture (vs Debezium, SQL Server CDC)
- As a data lake - foreign data wrappers to Parquet/S3, columnar extensions (pg_analytics, Hydra)
- As a vector store - pgvector extension, HNSW/IVFFlat indexes for RAG
- Backup replication - streaming replica as hot standby for reads and backup

### Constraints

- Table bloat - dead tuples accumulate, table grows 2-10x actual data. Workaround: autovacuum tuning, pg_repack
- Transaction ID wraparound - 2 billion TX limit, database refuses ALL writes. Workaround: aggressive vacuum, monitor xid age
- Single-node write throughput - one primary, WAL fsync ceiling. Workaround: batching, synchronous_commit off, Citus sharding
- Table size without partitioning - billions of rows = slow scans, VACUUM takes hours. Workaround: declarative partitioning
- Index size exceeds RAM - random I/O spikes when indexes don't fit shared_buffers. Workaround: partial indexes, BRIN, partitioning
- Lock contention - hot rows, DDL blocks all DML (AccessExclusiveLock). Workaround: lock_timeout, retry logic, CREATE INDEX CONCURRENTLY
- Replication lag - replicas fall behind under heavy writes, stale reads. Workaround: synchronous_commit = remote_apply for critical reads, multiple replicas
- Vacuum can't keep up - autovacuum workers saturated on write-heavy tables. Workaround: per-table autovacuum settings, more workers, partitioning
- Checkpoint I/O spikes - periodic WAL flush causes latency spikes. Workaround: checkpoint_completion_target, increase max_wal_size
- work_mem explosion - work_mem x active queries x sorts = OOM. Workaround: lower work_mem, fewer connections, optimize queries
- Long-running transactions - hold back VACUUM, cause bloat cascade. Workaround: statement_timeout, idle_in_transaction_session_timeout
- Heap table no clustered index - range scans hit random pages unlike SQL Server/InnoDB. Workaround: covering indexes, CLUSTER (takes lock), pg_repack
- No native query hints - planner picks bad plans, can't override. Workaround: pg_hint_plan extension, optimizer GUCs
- Sequence overflow - SERIAL = int4 max 2.1 billion, silently halts inserts. Workaround: use BIGSERIAL from day one
- Subtransaction overflow - more than 64 savepoints per TX degrades ALL connections (SubXID cache). Workaround: avoid deep subtransactions, restructure batch logic
- Dead connection detection - TCP keepalive default too high, dead clients hold slots for hours. Workaround: tcp_keepalives_idle, idle_in_transaction_session_timeout
- Schema-per-tenant catalog bloat - pg_class grows massive at 10,000+ tenants, query planning slows, pg_dump takes hours. Workaround: shared-table with RLS instead
- No resource quotas per role - one tenant consumes all CPU/IO/connections, no SQL Server Resource Governor equivalent. Workaround: application-level throttling, separate pools per tier
- Connection pooling conflicts with schema-per-tenant - SET search_path requires session-mode PgBouncer, can't reuse connections. Workaround: shared-table + RLS avoids this
- Schema migration coordination - no atomic cross-schema DDL, partial failures leave tenants on inconsistent schema versions. Workaround: idempotent migrations, tenant-aware version tracking
- Partition-per-tenant planner degradation - thousands of partitions = slow planning, session memory grows unbounded per tenant accessed. Workaround: shared-table or hash partitioning with fewer partitions
- Partitioning breaks global unique constraints - can't enforce UNIQUE(email) if partitioned by tenant_id, must include partition key. Workaround: separate global lookup table or app-layer enforcement
- RLS silent filtering breaks backups - backup as restricted user silently omits rows with no error. Workaround: BYPASSRLS role or row_security = off for admin connections
- Replication slot unbounded WAL retention - disconnected slot prevents ALL WAL cleanup until disk fills, crashing cluster. Workaround: max_slot_wal_keep_size, monitor slot lag
- Synchronous replication commit hang - all sync standbys die, primary hangs ALL writes forever with no timeout (MySQL semi-sync falls back to async). Workaround: ANY N (...) with multiple standbys

### Migration

#### From MySQL

- || is concatenation not OR - WHERE active || pending silently produces string concatenation instead of boolean logic
- No UNSIGNED integers - MySQL INT UNSIGNED (0 to 4.2B) overflows PostgreSQL integer (max 2.1B), need bigint
- Backslash not an escape by default - '\n' is literal two characters, need E'\n' or standard_conforming_strings = off
- No # comments - only -- and /**/ work, imported scripts break at runtime
- Backticks don't work - MySQL `table` syntax errors, use "table" for reserved words
- Integer division returns integer - MySQL SELECT 5/2 returns 2.5, PostgreSQL returns 2, financial calculations silently wrong
- Division by zero is a hard error - MySQL returns NULL, PostgreSQL throws error, need NULLIF(denominator, 0) guard

#### From Oracle

- Empty string is NOT NULL - Oracle treats '' as NULL, PostgreSQL treats '' as empty string, IS NULL checks silently miss data
- DATE loses time component - Oracle DATE includes hours/minutes/seconds, PostgreSQL DATE truncates time with no warning
- Transaction abort blocks all commands - after one error, every subsequent command fails until ROLLBACK (Oracle continues)
- Auto-commit by default - Oracle wraps in implicit transaction until COMMIT, PostgreSQL commits each statement immediately (partial writes persist if no explicit BEGIN)

#### From SQL Server

- No NEWID() - use gen_random_uuid() for UUID generation
- No ISNULL() - use COALESCE() (which is standard SQL)

## How PostgreSQL Thrives in the Agentic Era?

_Coming soon..._
