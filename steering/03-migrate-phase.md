# Migrate Phase — PostgreSQL Migration

## Overview

The Migrate phase executes the data migration from the source PostgreSQL database to the target (Aurora or RDS PostgreSQL) using the approach selected during Assess and validated during Mobilize. This phase covers three migration paths: offline (pg_dump/pg_restore), online native logical replication, and online DMS with CDC. It also covers hybrid approaches, cutover procedures, and rollback planning.

## Migration Context

- Source: PostgreSQL databases hosted on 3rd-party platforms (Supabase, Heroku, Neon, GCP Cloud SQL, Azure Database, on-premises, etc.)
- Target: AWS RDS PostgreSQL or Amazon Aurora PostgreSQL
- Approach: Database-focused migration with application connection update or refactoring as needed
- Focus: Execute the migration, validate data integrity, cut over the application, and handle rollback if needed

## Outline

1. [Migrate Decision Tree](#migrate-decision-tree)
2. [Migration Wave Planning](#migration-wave-planning)
3. [Pre-Migration Checklist](#pre-migration-checklist)
4. [Path 1: Offline Migration (pg_dump/pg_restore)](#path-1-offline-migration-pg_dumppg_restore)
5. [Path 2: Online Migration — Native Logical Replication](#path-2-online-migration--native-logical-replication)
6. [Path 3: Online Migration — AWS DMS with CDC](#path-3-online-migration--aws-dms-with-cdc)
7. [Path 4: Hybrid Migration](#path-4-hybrid-migration)
8. [Application Code Refactoring & Testing](#application-code-refactoring--testing)
9. [Cutover Procedure](#cutover-procedure)
10. [Rollback Procedure](#rollback-procedure)
11. [Offline Validation (Non-DMS Paths)](#offline-validation-non-dms-paths)
12. [Post-Migration Activities](#post-migration-activities)
13. [Common Challenges & Solutions](#common-challenges--solutions)

## Migrate Decision Tree

Use this decision tree to confirm and refine the decisions made during Assess and Mobilize. Each decision maps to a specific section in this phase — work through them in order before starting execution.

### Decision 1: Wave Planning Needed?

```
Are you migrating multiple databases?
├── YES → Organize into waves (see Migration Wave Planning)
│   ├── Wave 0: Proof of concept (1 non-prod database)
│   ├── Wave 1: Non-production environments
│   ├── Wave 2: Non-critical production
│   ├── Wave 3: Business-critical production
│   └── Wave 4: Complex/large databases
└── NO (single database) → Skip wave planning, proceed to Pre-Migration Checklist
```

### Decision 2: Migration Execution Path (Confirm Assess Decision)

```
Which migration approach was selected during Assess?
├── Offline (pg_dump/pg_restore) → Path 1
│   ├── Database < 100 GB? → Custom format, single-threaded or low parallelism (-j 4)
│   └── Database ≥ 100 GB? → Directory format with parallel jobs (-j 8+)
├── Native logical replication → Path 2
│   └── Was replication set up during Mobilize?
│       ├── YES → Verify it's running, proceed to monitor catch-up
│       └── NO → Set up now (follow Mobilize replication setup steps)
├── DMS with CDC → Path 3
│   └── Were DMS resources provisioned during Mobilize?
│       ├── YES → Create and start the replication task
│       └── NO → Provision DMS resources first (replication instance, endpoints)
└── Hybrid → Path 4
    └── Assign each table category to its migration method (see Path 4 strategy table)
```

### Decision 3: pg_dump Format (Offline Path Only)

```
Using offline migration (Path 1)?
├── NO → Skip this decision
└── YES
    ├── Database < 10 GB → Custom format (-F c), single restore job
    ├── Database 10–100 GB → Custom format (-F c), parallel restore (-j 4)
    └── Database > 100 GB → Directory format (-F d), parallel dump and restore (-j 8+)
```

### Decision 4: Application Refactoring Timing

```
Does the application require code refactoring? (from Assess Decision 2)
├── NO (connection string update only) → Skip to Cutover Procedure
└── YES (SDK refactoring required)
    ├── Was the migration branch created during Mobilize?
    │   ├── YES → Finalize refactoring, deploy to preview/staging, test against Aurora
    │   └── NO → Create migration branch now, then refactor
    └── When to merge the migration branch?
        ├── BEFORE cutover → Only if using merge-and-deploy cutover (Option B)
        └── AFTER cutover → If using rolling deployment cutover (Option A, recommended)
```

### Decision 5: Cutover Strategy

```
Which cutover approach fits your deployment platform?
├── Platform supports preview/branch deployments (Vercel, Netlify, Amplify)?
│   └── YES → Option A: Rolling deployment (recommended)
│       └── Deploy migration branch to preview → verify → promote to production → merge to main
├── Standard CI/CD pipeline (merge triggers deploy)?
│   └── YES → Option B: Merge and deploy
│       └── Merge migration branch to main → deploy → verify
└── No application code changes needed (connection string only)?
    └── YES → Option C: Connection string update
        └── Update DATABASE_URL / env vars / Secrets Manager → restart app → verify
```

### Decision 6: Maintenance Window Approach

```
Is the migration online (CDC running)?
├── YES → Minimal maintenance window needed (just for final catch-up + switch)
│   ├── Can you use a feature flag to stop writes? → Feature flag (fastest, no user-visible downtime)
│   ├── Can you split traffic? → Traffic splitting (gradual cutover)
│   └── Neither? → Full maintenance mode during final catch-up + switch
└── NO (offline migration)
    └── Full maintenance mode required for entire dump/restore cycle
        └── Estimate window from Mobilize pilot timing (pg_dump + transfer + pg_restore + validation)
```

### Decision 7: Rollback Strategy (Confirm Assess Decision)

```
Which rollback strategy was selected during Assess?
├── Strategy 1: Basic Fallback → Revert connection to source (post-cutover writes lost)
│   └── Appropriate if: non-critical database, cutover window is short, few writes expected
├── Strategy 2: Fall Forward (recommended for production) → Roll forward to replica A'
│   └── Is the B → A' replication task set up and tested?
│       ├── YES → Ready for cutover
│       └── NO → Set up B → A' DMS task before proceeding to cutover
├── Strategy 3: Dual Write → App writes to both source and Aurora
│   └── Is dual-write code implemented and tested?
│       ├── YES → Ready for cutover
│       └── NO → Implement dual-write logic before cutover
└── Strategy 4: DMS Bidirectional Replication → B ↔ A sync after cutover
    └── Is the reverse replication (B → A) task configured?
        ├── YES → Ready for cutover
        └── NO → Configure and test B → test-A replication before cutover
```

### Decision 8: Validation Approach

```
Which validation method applies?
├── Using DMS for migration (Path 3)?
│   └── YES → DMS in-task validation (EnableValidation: true)
│       └── Monitor the awsdms_validation_failures_v2 (or _v1) table on target (see Path 3 Step 4)
├── Using native replication or pg_dump (Path 1 or 2)?
│   └── Database size?
│       ├── < 10 GB → Exact row counts + md5 checksum per table
│       ├── 10–100 GB → Exact row counts + sampled checksums on large tables
│       └── > 100 GB → DMS validation-only task (TargetTablePrepMode: DO_NOTHING)
└── Hybrid (Path 4)?
    └── Use DMS validation for DMS-migrated tables + exact row counts and checksums for others
```

### Decision 9: Replication Cleanup Timing

```
Is CDC replication running (native or DMS)?
├── NO (offline migration) → No cleanup needed, skip this decision
└── YES
    ├── When to stop replication?
    │   └── ONLY after: lag = 0, maintenance mode active, final validation passed
    ├── Native replication cleanup order:
    │   1. Disable subscription on target
    │   2. Drop subscription on target
    │   3. Drop publication on source
    │   4. Drop replication slot on source (if not auto-dropped)
    └── DMS cleanup order:
        1. Stop DMS replication task
        2. Delete DMS task (after stabilization period)
        3. Delete DMS endpoints and replication instance (after all tasks deleted)
```

### Decision 10: Post-Cutover Monitoring Duration

```
How critical is the database?
├── Non-production → 24 hours of active monitoring
├── Production (non-critical) → 48 hours of active monitoring
├── Production (business-critical) → 72 hours of active monitoring + 2 weeks of elevated alerting
└── All environments:
    ├── Keep source database accessible for rollback during monitoring period
    ├── Do NOT decommission source until stabilization period complete (2–4 weeks)
    └── Monitor: error rates, latency (P50/P95/P99), CPU, memory, connections, I/O
```

### Migrate Decision Summary Template

After working through the decisions above, document the results:

| Decision | Choice | Notes |
| -------- | ------ | ----- |
| Wave planning | Single database / Wave N of M | |
| Execution path | Path 1 (offline) / Path 2 (native replication) / Path 3 (DMS CDC) / Path 4 (hybrid) | |
| pg_dump format | Custom (-F c) / Directory (-F d) / N/A | Parallel jobs: |
| App refactoring timing | Before cutover / After cutover / N/A (connection string only) | Branch name: |
| Cutover strategy | Option A (rolling) / Option B (merge-and-deploy) / Option C (connection string) | |
| Maintenance window | Feature flag / Traffic splitting / Full maintenance mode | Estimated duration: |
| Rollback strategy | Strategy 1 (basic) / Strategy 2 (fall forward) / Strategy 3 (dual write) / Strategy 4 (bidirectional) | |
| Validation approach | DMS in-task / Exact row counts + checksums / DMS validation-only task | |
| Replication cleanup | After cutover verification / After stabilization period | |
| Monitoring duration | 24h / 48h / 72h + 2 weeks elevated | Source decommission date: |

## Migration Wave Planning

For multi-database migrations, organize databases into waves to manage risk and team capacity.

### Wave Grouping Strategy

- Group databases by application criticality and interdependencies
- Consider database size and complexity (small/simple databases first)
- Account for application dependencies — databases that serve the same application should migrate together
- Schedule based on business impact windows (avoid peak traffic periods)
- Typical wave size: 5–15 databases

### Wave Prioritization

1. **Wave 0 — Proof of concept**: One non-production database to validate the migration path end-to-end
2. **Wave 1 — Non-production environments**: Dev, test, and staging databases
3. **Wave 2 — Non-critical production**: Internal tools, low-traffic applications
4. **Wave 3 — Business-critical production**: Customer-facing applications, high-traffic databases
5. **Wave 4 — Complex/large databases**: Databases requiring extended migration windows or hybrid approaches

### Wave Execution Cadence

- Run waves sequentially — complete one wave (including stabilization) before starting the next
- Allow 1–2 weeks between waves for lessons learned and process refinement
- Each wave follows the full migration lifecycle: pre-migration → execution → validation → cutover → stabilization
- Update the migration runbook after each wave with actual timings, issues, and resolutions

### Single-Database Migrations

For single-database migrations, wave planning is not needed — proceed directly to the Pre-Migration Checklist.

## Pre-Migration Checklist

**Prerequisites from Mobilize phase** (must be complete before starting — see `02-mobilize-phase.md` checklist):

- [ ] Aurora cluster provisioned and healthy (with correct parameter group)
- [ ] Schema migrated to target and verified (object counts match)
- [ ] ENUM types created on target (if applicable)
- [ ] DMS infrastructure ready: replication instance + endpoints with tested connections (if using DMS)
- [ ] Native replication configured and active (if using native path)
- [ ] Source replication parameters configured (max_replication_slots, max_wal_senders, etc.)
- [ ] Security groups allow connectivity between source, DMS, and target
- [ ] Migration branch created (`migration/aurora-refactor`)

**Migrate phase readiness:**

- [ ] Application maintenance window scheduled (if offline migration)
- [ ] Rollback plan documented and reviewed
- [ ] Monitoring dashboards configured (CloudWatch, pg_stat_replication)
- [ ] Read `supabase-migration-feature-mapping.md` → "Application Code Refactoring" and "Vercel + Aurora Connectivity" (if refactoring needed)

## Path 1: Offline Migration (pg_dump/pg_restore)

Best for databases < 100 GB or when downtime is acceptable.

### Execution Method Decision

Before starting, determine which execution method is available:

- **EC2 bastion or CloudShell available** → use `pg_dump`/`pg_restore` (Steps 1–5 below). If no bastion exists yet, provision one per `supabase-prerequisites-and-connectivity.md` → "EC2 Bastion Setup".
- **No shell access** (agent running from local Kiro, no bastion provisioned, database < 1 GB) → use **MCP Data Copy** as an alternative:
  1. Read all table data from Supabase via Supabase MCP (`execute_sql`: `SELECT * FROM <table>`)
  2. Write to Aurora via Aurora MCP or Data API (`INSERT INTO <table> VALUES ...`)
  3. Process tables in dependency order (parents before children) to respect foreign keys
  4. After all tables copied, run `ANALYZE VERBOSE;` on Aurora
  5. Validate exact row counts (`count(*)`) and checksums, source vs target ([Offline Validation](#offline-validation-non-dms-paths))

  This approach is equivalent in result but limited to small databases (< 1 GB / < 10K rows per table) due to MCP response size limits and Data API's 45-second timeout. For anything larger, provision a bastion.

### Step 1: Put Application in Maintenance Mode

Stop writes to the source database:

- Enable application maintenance mode
- Or redirect traffic to a static page
- Verify no active connections are writing:

```sql
-- On source: check active connections
SELECT pid, usename, application_name, state, query_start, query
FROM pg_stat_activity
WHERE datname = current_database() AND state != 'idle'
ORDER BY query_start;
```

### Step 2: Export from Source

```bash
# Full dump in custom format (supports parallel restore)
pg_dump -h <source-host> -U <source-user> -d <source-database> \
  --no-owner --no-acl \
  -F c -f source_dump.backup

# Record the dump size and time for the runbook
ls -lh source_dump.backup
```

Exclude platform-specific schemas as identified in the Mobilize phase (add `-N <schema>` flags).

For very large databases, use directory format with parallel jobs:

```bash
pg_dump -h <source-host> -U <source-user> -d <source-database> \
  --no-owner --no-acl \
  -F d -j 4 -f source_dump_dir/
```

### Step 3: Transfer Dump to Target-Accessible Host

If restoring from an EC2 bastion:

```bash
# Transfer via SSM (if dump was created locally)
aws ssm start-session --target <ec2-instance-id>
# Then use scp, S3, or EFS to move the dump file
```

### Step 4: Restore to Target

```bash
# Drop and recreate public schema (clean slate)
psql -h <target-endpoint> -U postgres -d <database-name> \
  -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"

# Restore with parallel jobs
pg_restore --no-owner --no-acl \
  -h <target-endpoint> -U postgres -d <database-name> \
  -j 4 source_dump.backup
```

For directory format:

```bash
pg_restore --no-owner --no-acl \
  -h <target-endpoint> -U postgres -d <database-name> \
  -j 4 source_dump_dir/
```

### Step 5: Post-Restore Validation

```sql
-- Gather fresh statistics
ANALYZE VERBOSE;

-- Exact row counts (read-only, one count(*) per table). Run on both source and target, then compare per table.
SELECT table_schema, table_name,
  (xpath('/row/c/text()',
         query_to_xml(format('select count(*) as c from %I.%I', table_schema, table_name), false, true, '')))[1]::text::bigint AS exact_rows
FROM information_schema.tables
WHERE table_type = 'BASE TABLE' AND table_schema = 'public'
  AND table_name NOT LIKE 'awsdms\_%'
ORDER BY table_schema, table_name;

-- Object count comparison
SELECT 'tables' AS type, COUNT(*) FROM information_schema.tables WHERE table_schema = 'public'
UNION ALL
SELECT 'indexes', COUNT(*) FROM pg_indexes WHERE schemaname = 'public'
UNION ALL
SELECT 'sequences', COUNT(*) FROM information_schema.sequences WHERE sequence_schema = 'public';
```

Row-count estimates from the statistics views are not valid for validation. Compare exact counts, then compare a checksum per table as described in [Offline Validation](#offline-validation-non-dms-paths). If `query_to_xml` is not allowed, run `SELECT count(*) FROM public.<table>` once for each table on both sides.

### Step 6: Proceed to Cutover

**Before proceeding**: confirm application refactoring is complete and tested against Aurora on a preview/staging deployment. See the [Cutover Procedure](#cutover-procedure) section below.

## Path 2: Online Migration — Native Logical Replication

Best for zero-downtime migration when all tables have primary keys.

### Step 1: Verify Replication is Active

If replication was set up during Mobilize, verify it's running:

```sql
-- On source: check replication status
SELECT pid, usename, application_name, client_addr, state,
  sent_lsn, write_lsn, flush_lsn, replay_lsn,
  pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replication_lag_bytes
FROM pg_stat_replication;

-- On target: check subscription status
SELECT subname, subenabled, subslotname
FROM pg_stat_subscription;
```

If replication was not yet started, follow the setup steps from the Mobilize phase.

### Step 2: Add Tables to Publication

If not all tables were added during Mobilize:

```sql
-- On source: add remaining tables
ALTER PUBLICATION source_to_target ADD TABLE public.new_table1, public.new_table2;
```

On the target, refresh the subscription to pick up new tables:

```sql
ALTER SUBSCRIPTION migration_sub REFRESH PUBLICATION;
```

### Step 3: Monitor Initial Data Copy

When `copy_data = true`, the subscription first copies all existing data before streaming changes. Monitor progress:

```sql
-- On target: check copy state per table (i = initialize, d = copying, f = copy finished, s = synchronized, r = ready)
SELECT srrelid::regclass AS table_name, srsubstate
FROM pg_subscription_rel
ORDER BY 1;

-- On source: check replication slot lag
SELECT slot_name, active,
  pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn) AS lag_bytes,
  pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag_pretty
FROM pg_replication_slots
WHERE slot_name = 'migration_slot';
```

When every table shows `r`, the initial copy is done. Compare exact row counts and checksums at cutover ([Offline Validation](#offline-validation-non-dms-paths)); the row estimates in `pg_stat_user_tables` do not show copy progress reliably.

### Step 4: Wait for Replication to Catch Up

Before cutover, replication lag must be near zero:

```sql
-- On source: check lag is minimal
SELECT pid, application_name,
  pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes,
  pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS lag_pretty
FROM pg_stat_replication;
```

Target: lag < 1 MB before proceeding to cutover.

### Step 5: Proceed to Cutover

**Before proceeding**: confirm application refactoring is complete and tested against Aurora on a preview/staging deployment. See the [Cutover Procedure](#cutover-procedure) section below.

## Path 3: Online Migration — AWS DMS with CDC

Best for an IPv6-only source, when built-in validation is needed, or as a fallback for native replication failures such as an unsupported data type. A table without a primary key is not a reason to choose DMS: see [Tables Without Primary Keys](#tables-without-primary-keys).

> **Before you run any command**
>
> Record these four Mobilize decisions in the migration notes (see the Mobilize decision summary table in `02-mobilize-phase.md`):
>
> - Decision 1, schema method (Method A or B; Method C is the offline pg_dump route and does not apply to DMS)
> - Decision 4, plugin (`test_decoding` or `pglogical`)
> - Decision 5, target table prep mode
> - Decision 6, foreign key bypass
>
> If any of them is not recorded, stop and apply the Mobilize decisions first (Decisions 1, 4, 5 and 6 in `02-mobilize-phase.md`). Do not create the replication task until all four are recorded.
>
> The four rules: all four values are recorded with a reason for each; a preloaded schema (Method B) needs `DO_NOTHING`; a source with foreign keys needs a bypass; Method A cannot create foreign keys and needs `DROP_AND_CREATE`.

### Step 1: Create DMS Replication Task

Fill `<target-prep-mode>` from the value recorded for [Decision 5](02-mobilize-phase.md#decision-5-dms-target-table-preparation-mode). Preloaded schema (Method B): `DO_NOTHING`, and the target tables must be empty. Method A: `DROP_AND_CREATE`. Never write `TRUNCATE_BEFORE_LOAD` here for a preloaded schema: it fails on tables that foreign keys reference.

```bash
aws dms create-replication-task \
  --replication-task-identifier dms-task-<project-name> \
  --source-endpoint-arn <source-endpoint-arn> \
  --target-endpoint-arn <target-endpoint-arn> \
  --replication-instance-arn <replication-instance-arn> \
  --migration-type full-load-and-cdc \
  --table-mappings '{
    "rules": [{
      "rule-type": "selection",
      "rule-id": "1",
      "rule-name": "select-public-tables",
      "object-locator": {
        "schema-name": "public",
        "table-name": "%"
      },
      "rule-action": "include"
    }, {
      "rule-type": "selection",
      "rule-id": "2",
      "rule-name": "exclude-dms-helper-tables",
      "object-locator": {
        "schema-name": "public",
        "table-name": "awsdms_%"
      },
      "rule-action": "exclude"
    }]
  }' \
  --replication-task-settings '{
    "TargetMetadata": {
      "TargetSchema": "public",
      "SupportLobs": true,
      "LobChunkSize": 64,
      "LimitedSizeLobMode": true,
      "LobMaxSize": 32768
    },
    "FullLoadSettings": {
      "TargetTablePrepMode": "<target-prep-mode>",
      "CreatePkAfterFullLoad": false
    },
    "ValidationSettings": {
      "EnableValidation": true,
      "ThreadCount": 5,
      "PartitionSize": 10000,
      "FailureMaxCount": 10000,
      "TableFailureMaxCount": 1000,
      "ValidationQueryCdcDelaySeconds": 0
    },
    "Logging": {
      "EnableLogging": true,
      "LogComponents": [{
        "Id": "TRANSFORMATION",
        "Severity": "LOGGER_SEVERITY_DEFAULT"
      }, {
        "Id": "SOURCE_UNLOAD",
        "Severity": "LOGGER_SEVERITY_DEFAULT"
      }, {
        "Id": "TARGET_LOAD",
        "Severity": "LOGGER_SEVERITY_DEFAULT"
      }]
    }
  }' \
  --region <aws-region>
```

`ValidationQueryCdcDelaySeconds` is 0 here, the same value as in "DMS Validation Setup" in `02-mobilize-phase.md` (which explains when to raise it for a busy source).

The second selection rule keeps DMS helper tables left in the source (`awsdms_heartbeat`, `awsdms_ddl_audit`) out of the task, because they come from DMS and not from the application. In a selection rule `%` matches zero or more characters and `_` matches a single character ([wildcards](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Tasks.CustomizingTasks.TableMapping.SelectionTransformation.Wildcards.html)), so `awsdms_%` also matches a name such as `awsdmsX1`. This exclude rule was not exercised on a live task.

### Step 2: Start the DMS Task

```bash
aws dms start-replication-task \
  --replication-task-arn <task-arn> \
  --start-replication-task-type start-replication \
  --region <aws-region>
```

`start-replication` is the type for the first run. After a failed or partial run, do not start the task again as is. Follow [Reload after a failed or partial run](troubleshooting.md#reload-after-a-failed-or-partial-run) and use `reload-target` at the end of those steps. If you changed an endpoint, run `test-connection` again first (see [Mobilize Step 6](02-mobilize-phase.md#step-6-test-endpoint-connections)).

### Step 3: Monitor DMS Task Progress

```bash
# Check task status
aws dms describe-replication-tasks \
  --filters Name=replication-task-arn,Values=<task-arn> \
  --query 'ReplicationTasks[0].{Status:Status,Progress:ReplicationTaskStats}' \
  --region <aws-region>

# Check table statistics
aws dms describe-table-statistics \
  --replication-task-arn <task-arn> \
  --region <aws-region>
```

CloudWatch metrics to monitor:

- `CDCLatencySource` — lag at the source
- `CDCLatencyTarget` — lag at the target
- `CDCIncomingChanges` — pending changes
- `FullLoadThroughputRowsSource` — rows/second during full load

While the task runs, the replication slot and the `awsdms_` helper tables exist on the source. DMS creates them, not the agent. See [Mobilize Step 4b](02-mobilize-phase.md#step-4b-what-dms-writes-to-the-source) for what they are and [DMS Cleanup on the Source](#dms-cleanup-on-the-source) for removing them.

When a table shows `Table error`, follow [DMS table error: exitwhy 1 / check target database logs](troubleshooting.md#dms-table-error-exitwhy-1--check-target-database-logs) and read the target database log yourself.

### Step 4: Monitor DMS Validation

```bash
# Check validation status per table
aws dms describe-table-statistics \
  --replication-task-arn <task-arn> \
  --query 'TableStatistics[*].{Table:TableName,State:ValidationState,Pending:ValidationPendingRecords,Failed:ValidationFailedRecords,Suspended:ValidationSuspendedRecords}' \
  --region <aws-region>
```

When a table shows `Mismatched records` (or `Suspended records`), read the validation failures table on the target yourself as the first step, before you offer the user any explanation. The procedure below is the single place for the queries. The decoding steps are in [DMS validation: Mismatched records](troubleshooting.md#dms-validation-mismatched-records).

1. Find the schema of the control tables. It is a task setting (`ControlTablesSettings.ControlSchema`). When it is empty, DMS uses the engine default, which is `public` for a PostgreSQL target ([Control table task settings](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Tasks.CustomizingTasks.TaskSettings.ControlTable.html)). Do not hard-code it:

   ```bash
   aws dms describe-replication-tasks --filters Name=replication-task-arn,Values=<task-arn> --query "ReplicationTasks[0].ReplicationTaskSettings" --output text --region <aws-region>
   ```

   The output is a JSON string. Read `ControlTablesSettings.ControlSchema` from it. If you cannot tell, look the tables up by name:

   ```sql
   SELECT table_schema, table_name FROM information_schema.tables WHERE table_name LIKE 'awsdms%' ORDER BY 1, 2;
   ```

2. Pick the failures table by engine version. From replication engine 3.6.1 DMS creates `awsdms_validation_failures_v2` on a PostgreSQL target, shared by all tasks that have validation on; before that it creates `awsdms_validation_failures_v1`. Look for `_v2` first, then `_v1`, in the lookup above. Use the name you find in place of `<failures-table>` in the queries below. Never drop or truncate a `_v2` table ([DMS data validation](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Validating.html)).

3. Read the rows. The column names are uppercase and must be quoted. `<control-schema>` is `public` unless the task sets another one, and `<failures-table>` is `awsdms_validation_failures_v2` or `awsdms_validation_failures_v1`, whichever step 2 found:

   ```sql
   SELECT "TASK_NAME", "TABLE_OWNER", "TABLE_NAME", "FAILURE_TIME", "KEY_TYPE", "KEY", "FAILURE_TYPE", "DETAILS"
   FROM <control-schema>.<failures-table> ORDER BY "FAILURE_TIME" DESC LIMIT 50;
   ```

4. Rows from an earlier run stay in this table: `reload-target` and truncating the data tables do not clear it. Tell the loads apart with a grouped summary, then compare the minutes with the start time of the current load:

   ```sql
   SELECT "TABLE_NAME", "FAILURE_TYPE", date_trunc('minute', "FAILURE_TIME") AS minute, count(*)
   FROM <control-schema>.<failures-table>
   GROUP BY "TABLE_NAME", "FAILURE_TYPE", date_trunc('minute', "FAILURE_TIME") ORDER BY minute DESC;
   ```

   Before a reload, filter by `"FAILURE_TIME"` when you read it, or delete only this task's rows with `DELETE FROM <control-schema>.<failures-table> WHERE "TASK_NAME" = '<task-external-id>'` (truncate a `_v1` table only when no other task validates against the same target; never a `_v2` table; see [Reload after a failed or partial run](troubleshooting.md#reload-after-a-failed-or-partial-run)).

5. For errors while DMS applies CDC changes, read the second control table, which has `"TASK_NAME"`, `"TABLE_OWNER"`, `"TABLE_NAME"`, `"ERROR_TIME"`, `"STATEMENT"` and `"ERROR"`:

   ```sql
   SELECT "TASK_NAME", "TABLE_OWNER", "TABLE_NAME", "ERROR_TIME", "STATEMENT", "ERROR"
   FROM <control-schema>.awsdms_apply_exceptions ORDER BY "ERROR_TIME" DESC LIMIT 50;
   ```

`ControlSchema` is optional. Set it in the task settings (`"ControlTablesSettings": {"ControlSchema": "<control-schema>"}`) only if the user wants the control tables kept out of the application schema. The example task leaves it unset, so the default applies.

DMS suspends validation after 10,000 failures. Resolve failures and restart validation if needed.

If a table shows `Mismatched records`, do not act on the mismatch alone. Decode it with the key lookup and cross-check with independent counts and checksums: [DMS validation: Mismatched records](troubleshooting.md#dms-validation-mismatched-records). For a boolean column see also [DMS validation mismatch on boolean columns](troubleshooting.md#dms-validation-mismatch-on-boolean-columns) and [DMS Validation Mismatch on Boolean Columns](#dms-validation-mismatch-on-boolean-columns).

### Step 5: Wait for CDC to Catch Up

Before cutover, ensure CDC lag is minimal:

- `CDCLatencyTarget` < 5 seconds
- `CDCIncomingChanges` = 0 or near 0
- All tables show `ValidationState = Validated`, or a mismatch is explained by equal independent checksums ([boolean columns](#dms-validation-mismatch-on-boolean-columns))

### Step 6: Proceed to Cutover

**Before proceeding**: confirm application refactoring is complete and the refactored app has been tested against Aurora on a preview/staging deployment. CDC lag being near-zero is necessary but not sufficient — do NOT announce cutover readiness until the app is also ready.

See the [Cutover Procedure](#cutover-procedure) section below.

## Path 4: Hybrid Migration

Best for mixed schemas where some tables suit native replication and others need DMS or offline migration.

### Strategy by Table Category

| Category | Migration Method | Reason |
| -------- | --------------- | ------ |
| Tables with PKs, standard types | Native logical replication | Preferred, lowest latency |
| Tables with PKs, needing validation | DMS with CDC + validation | Built-in row-by-row validation |
| Tables without PKs | Offline (pg_dump/pg_restore per table) | CDC requires PKs for UPDATE/DELETE |
| Large reference/lookup tables | Offline (pg_dump/pg_restore per table) | Static data, no CDC needed |
| Tables with unsupported types for DMS | Native logical replication or offline | DMS doesn't handle INET, TSVECTOR, etc. |

### Execution Order

1. **Migrate static/reference tables offline first** — these rarely change and can be loaded before cutover
2. **Start native logical replication** for the primary table set (tables with PKs, standard types)
3. **Start DMS tasks** for tables needing validation or where native replication failed
4. **Wait for all CDC streams to catch up** — replication lag near zero on all paths
5. **Migrate PK-less tables offline** during the maintenance window (just before cutover)
6. **Proceed to cutover** once all paths are synchronized

### Monitoring Multiple Paths

Monitor all active replication paths simultaneously:

```sql
-- Native replication lag (on source)
SELECT pid, application_name,
  pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes,
  pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS lag_pretty
FROM pg_stat_replication;
```

```bash
# DMS CDC lag (via CLI)
aws dms describe-replication-tasks \
  --filters Name=replication-task-arn,Values=<task-arn> \
  --query 'ReplicationTasks[0].ReplicationTaskStats.{CDCLatency:CDCLatencyTarget,Incoming:CDCIncomingChanges}' \
  --region <aws-region>
```

All paths must show near-zero lag before proceeding to cutover.

## Application Code Refactoring & Testing

**MANDATORY CHECKPOINT — Before modifying any application code:**
1. Create a migration branch: `git checkout -b migration/aurora-refactor`
2. Read `supabase-migration-feature-mapping.md` → "Application Code Refactoring" and "Vercel + Aurora Connectivity" sections for the correct driver choice and patterns
3. Only then begin modifying files on the migration branch

This section applies if the Assess phase identified that the application requires code refactoring (platform SDK → standard PostgreSQL driver). If the app only needs a connection string update, skip to the [Cutover Procedure](#cutover-procedure).

Read `supabase-migration-feature-mapping.md` → "Application Code Refactoring — Supabase SDK to Direct PostgreSQL Connection" (driver selection, pattern mapping table, server actions refactoring pattern) and → "Vercel + Aurora Connectivity" (OIDC setup, Data API client implementation).

### Step 1: Finalize Refactoring on Migration Branch

1. Ensure the migration branch (e.g., `aurora-migration`) is up to date with main
2. The agent reads the existing application code to understand the data layer
3. The agent generates refactored code replacing platform SDK calls with standard PostgreSQL queries:
   - Replace SDK client initialization with a PostgreSQL connection pool
   - Convert all read queries (selects, filters, joins) to SQL or ORM equivalents
   - Convert all write operations (inserts, updates, deletes) to SQL or ORM equivalents
   - Update error handling for PostgreSQL-specific error codes
   - Remove platform SDK dependencies from package.json / requirements.txt
4. The user reviews the generated code before accepting

For detailed refactoring guidance, patterns, and examples, refer to the platform-specific steering file (e.g., `supabase-migration-feature-mapping.md`).

### Step 2: Deploy Migration Branch to Preview/Staging

Deploy the refactored branch to a non-production environment pointing at Aurora:

- **Vercel/Netlify**: Push the branch — a preview deployment is created automatically. Set Aurora env vars (e.g., `DATABASE_URL`) on the preview deployment only.
- **EC2/ECS**: Deploy to a staging environment with Aurora connection string in env vars.
- **Lambda**: Deploy a separate stack with Aurora connection string.

Production continues running against the old database on the main branch — no impact.

### Step 3: Test the Refactored Application

- Verify all CRUD operations work against Aurora
- Test edge cases: empty results, large datasets, concurrent writes
- Verify error handling (connection failures, constraint violations)
- Run any existing automated tests against the preview deployment
- Perform manual smoke testing of critical user flows
- Compare behavior with the production app to confirm parity
- Check application logs for unexpected errors or warnings

**Do NOT merge to main until data migration and validation are complete and cutover is ready.**

## Cutover Procedure

**MANDATORY GATE — Do NOT proceed to cutover unless ALL of the following are true:**

If the application requires code refactoring (Supabase SDK → direct PostgreSQL):
- [ ] Migration branch (`migration/aurora-refactor`) exists with refactored code
- [ ] Refactored app deployed to preview/staging environment pointing at Aurora
- [ ] All CRUD operations verified working against Aurora on preview deployment
- [ ] Application logs show no errors on preview deployment

If the application only needs a connection string update:
- [ ] New connection string tested locally or on staging against Aurora

**Do NOT announce "ready for cutover" until the app refactoring and preview testing above are complete.** Data replication being ready is necessary but not sufficient.

### Pre-Cutover Validation

Before starting the cutover window:

- [ ] All replication paths show lag < 1 MB (native) or < 5 seconds (DMS)
- [ ] DMS validation shows all tables as `Validated` (if using DMS)
- [ ] Exact row counts (`count(*)`) and table checksums (full or sampled, as chosen in Offline Validation) match between source and target (see [Offline Validation](#offline-validation-non-dms-paths); statistics-view estimates are not valid for this check)
- [ ] Rollback strategy selected and infrastructure provisioned (e.g., fall forward replica A' if using Strategy 2)
- [ ] Application maintenance page or feature flag is ready
- [ ] Rollback plan is documented and reviewed
- [ ] Team is available for the cutover window

### Reset Sequences on Target

Native logical replication does not replicate sequence values. Generate reset commands before cutover (run these on the target, see `database-operations-reference.md` → "Reset Sequences After Migration" for the full query):

### Cutover Steps

Execute these steps in order during the maintenance window:

**Step 1: Enable application maintenance mode**

Stop all writes to the source database. Redirect users to a maintenance page or enable a feature flag.

**Step 2: Wait for final replication catch-up**

```sql
-- Native replication: confirm lag is zero
SELECT pid, application_name,
  pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;
```

For DMS, confirm `CDCLatencyTarget` = 0 and `CDCIncomingChanges` = 0.

**Step 3: Stop replication**

For native logical replication:
```sql
-- On target: disable subscription
ALTER SUBSCRIPTION migration_sub DISABLE;

-- On target: drop subscription
DROP SUBSCRIPTION migration_sub;

-- On source: drop publication
DROP PUBLICATION source_to_target;

-- On source: drop replication slot (if not auto-dropped)
SELECT pg_drop_replication_slot('migration_slot');
```

For DMS:
```bash
aws dms stop-replication-task \
  --replication-task-arn <task-arn> \
  --region <aws-region>
```

The DMS logical replication slot and the `awsdms_` helper objects stay on the source after the task stops. A stopped task's slot keeps holding write-ahead log on the source, so if the source still takes writes, ask the user to drop the slot as soon as the task is stopped and the cutover is accepted. Remove the helper objects after the stabilization period. Both follow [DMS Cleanup on the Source](#dms-cleanup-on-the-source).

**Step 4: Reset sequences on target**

Run the sequence reset commands generated above.

**Step 5: Run final validation**

```sql
-- Exact row counts (read-only, one count(*) per table). Run on both source and target, then compare per table.
SELECT table_schema, table_name,
  (xpath('/row/c/text()',
         query_to_xml(format('select count(*) as c from %I.%I', table_schema, table_name), false, true, '')))[1]::text::bigint AS exact_rows
FROM information_schema.tables
WHERE table_type = 'BASE TABLE' AND table_schema = 'public'
  AND table_name NOT LIKE 'awsdms\_%'
ORDER BY table_schema, table_name;
```

Then compare a checksum per table on both sides (see [Offline Validation](#offline-validation-non-dms-paths)). Differences should be zero or explainable (e.g., platform-specific tables excluded from migration). Row-count estimates from the statistics views (`n_live_tup`) are not valid for this check: compare exact counts only.

**Step 6: Gather fresh statistics on target**

```sql
ANALYZE VERBOSE;
```

**Step 7: Switch application to Aurora**

Execute the cutover strategy chosen in Decision 5:

**Option A: Rolling deployment** — promote the migration branch preview deployment to production. Assign production domain to the preview URL or use traffic splitting. Once healthy, merge to main:

```bash
git checkout main
git merge aurora-migration
```

**Option B: Merge and deploy** — merge migration branch to main, triggering a production deploy:

```bash
git checkout main
git merge aurora-migration
```

**Option C: Connection string only** — update DATABASE_URL / Secrets Manager entry to point at Aurora writer endpoint. Restart app.

**Step 8: Verify application is working**

- Test critical application flows (login, read, write, search)
- Monitor error rates and latency
- Check database connections are established
- Verify no errors in application logs

### Post-Cutover Monitoring

For post-cutover monitoring metrics, see `supabase-migration-strategy-reference.md` → "Post-Cutover" checklist.

## Rollback Procedure

Choose a rollback strategy based on your migration complexity and risk tolerance. These strategies are based on [AWS guidance for rolling back from a migration with DMS](https://aws.amazon.com/blogs/database/rolling-back-from-a-migration-with-aws-dms/).

### Strategy 1: Basic Fallback (Simplest)

Point the application back to the original source database. Any transactions written to Aurora after cutover are lost.

Appropriate when:
- The application was on Aurora for a very short time (minutes)
- Few or no write transactions occurred on Aurora post-cutover
- You can tolerate losing transactions written to Aurora after cutover
- The source database is still intact and accessible

Steps:
1. Revert the application to the source database:
   - **If rolling deployment (Option A)**: Re-promote the original production deployment, or revert the domain assignment
   - **If merge and deploy (Option B)**: Revert the merge commit on main (`git revert -m 1 <merge-commit>`) and redeploy
   - **If connection string only (Option C)**: Revert the connection string to the source endpoint
2. Disable maintenance mode
3. Verify the application is working against the source
4. Investigate the issue before attempting cutover again

### Strategy 2: Fall Forward (Recommended for Production)

Instead of rolling back to the original source (A), roll forward to a replica of the source (A') that has been kept in sync with Aurora (B) via a reverse replication stream: A → B → A'.

This is the preferred strategy because A' contains all transactions written to Aurora after cutover — no data loss.

Setup (before cutover):
1. Create a replica of the source database (A') on RDS/Aurora or EC2
2. Set up a DMS replication task from Aurora (B) to A': `B → A'`
3. Test the B → A' replication stream thoroughly before cutover
4. At cutover, stop the A → B replication but keep B → A' running

```
Before cutover:  Source (A) --DMS--> Aurora (B) --DMS--> Replica (A')
After cutover:   App --> Aurora (B) --DMS--> Replica (A')
Rollback:        App --> Replica (A')  [all DMS tasks stopped]
```

Rollback steps:
1. Stop all DMS replication tasks
2. Point the application to A' (which has all data including post-cutover transactions)
3. Verify the application is working against A'
4. A' becomes the new source of truth

This approach requires provisioning A' and the B → A' replication task before cutover, but provides the safest rollback path.

### Strategy 3: Dual Write (For Phased or Partitioned Migrations)

Modify the application to write to both the source and Aurora simultaneously. Rollback is straightforward — stop writing to Aurora.

Appropriate when:
- Migrating a multi-tenant system one tenant at a time
- Data is partitioned and you want to migrate incrementally
- Other strategies don't fit your architecture

Trade-offs:
- Requires application code changes to support dual writes
- More complex to implement and test
- Adds latency to write operations
- Must handle write failures to one target gracefully (e.g., async writes to the secondary)

```
During migration:  App --writes--> Source (A) + Aurora (B)
                   DMS keeps B in sync until dual write is enabled
Rollback:          App stops writing to B, continues with A only
```

### Strategy 4: DMS Bidirectional Replication (When Fall Forward Is Not Possible)

Use DMS bidirectional replication to keep the source and Aurora in sync in both directions after cutover. Rollback means pointing back to the source, which has been kept up to date.

Appropriate when:
- Creating a fall forward replica (A') is not practical
- The source database is part of a complex multi-master or interdependent system

Caution:
- The reverse replication stream (B → A) cannot be fully tested until after cutover
- To mitigate this risk, create a test copy of the source and validate the B → test-A replication task before cutover
- Bidirectional replication requires loop-back prevention (DMS handles this)

```
After cutover:  App --> Aurora (B) --DMS bidirectional--> Source (A)
Rollback:       App --> Source (A)  [DMS tasks stopped, B deprecated]
```

### Choosing a Rollback Strategy

| Strategy | Data Loss Risk | Complexity | Best For |
| -------- | ------------- | ---------- | -------- |
| Basic fallback | High (post-cutover writes lost) | Low | Quick rollback within minutes, few writes |
| Fall forward | None | Medium | Production migrations (recommended) |
| Dual write | None | High | Phased/partitioned migrations |
| Bidirectional replication | None (if working) | High | Complex interdependent systems |

For most single-database migrations, start with basic fallback as the minimum plan and implement fall forward if the database is production-critical.

## Offline Validation (Non-DMS Paths)

For migrations using pg_dump/pg_restore or native logical replication (which don't have built-in validation), use these validation approaches.

### Validation Strategy by Database Size

| Database Size | Validation Approach |
| ------------- | ------------------- |
| < 10 GB | Exact row counts + md5 checksum per table, plus spot-check sample rows |
| 10–100 GB | Exact row counts + sampled checksums on large tables (full checksums on the small ones) |
| > 100 GB | DMS validation-only task (continuous, row-by-row), plus exact row counts |

`n_live_tup` and the other statistics-view counts are estimates kept by autovacuum and ANALYZE. They are not exact counts: a table that was never analyzed can report 0 even when it is full. Use them only as a planning estimate (sizing, ordering of tables). They are not used to decide anything.

### Row Count Validation

Run on both source and target, then compare per table:

```sql
-- Exact row counts (read-only, one count(*) per table). Run on both source and target, then compare per table.
SELECT table_schema, table_name,
  (xpath('/row/c/text()',
         query_to_xml(format('select count(*) as c from %I.%I', table_schema, table_name), false, true, '')))[1]::text::bigint AS exact_rows
FROM information_schema.tables
WHERE table_type = 'BASE TABLE' AND table_schema = 'public'
  AND table_name NOT LIKE 'awsdms\_%'
ORDER BY table_schema, table_name;
```

Row-count estimates from the statistics views are not valid for validation or cutover readiness. Compare exact counts only. If `query_to_xml` is not allowed, run `SELECT count(*) FROM public.<table>` once for each table on both sides. A difference stops the run: do not continue to cutover on a mismatch.

### Checksum Validation

Run `SHOW timezone` on both sides first. The two values must match (Aurora and Supabase default to UTC), because the checksum uses each row's text form. For small databases, compare a full-table checksum for every table:

```sql
-- Full checksum per table (run on both source and target)
SELECT md5(COALESCE(string_agg(t::text, ',' ORDER BY t.<primary_key_column>), ''))
FROM public.<table_name> t;
```

A table without a primary key has no stable order. Use the order-independent form `md5(COALESCE(string_agg(md5(t::text), '' ORDER BY md5(t::text)), ''))` instead.

### Sampled Checksum Validation

For large tables, compare checksums on samples of rows. Take several key ranges, not only the first rows:

```sql
-- Checksum a sample of rows from a table (run on both source and target)
SELECT md5(string_agg(t::text, ',' ORDER BY t.<primary_key_column>))
FROM (
  SELECT * FROM public.<table_name>
  WHERE <primary_key_column> >= <range_start>
  ORDER BY <primary_key_column>
  LIMIT 1000
) t;
```

Replace `<primary_key_column>`, `<table_name>` and `<range_start>` for each table and range being validated. Equal sampled checksums support the exact counts but do not prove every row is equal.

### DMS Validation-Only Task

For large databases (> 100 GB), create a DMS task that only validates — no data migration:

```bash
aws dms create-replication-task \
  --replication-task-identifier dms-validate-only-<project-name> \
  --source-endpoint-arn <source-endpoint-arn> \
  --target-endpoint-arn <target-endpoint-arn> \
  --replication-instance-arn <replication-instance-arn> \
  --migration-type full-load \
  --table-mappings '{
    "rules": [{
      "rule-type": "selection",
      "rule-id": "1",
      "rule-name": "select-public-tables",
      "object-locator": { "schema-name": "public", "table-name": "%" },
      "rule-action": "include"
    }]
  }' \
  --replication-task-settings '{
    "FullLoadSettings": {
      "TargetTablePrepMode": "DO_NOTHING"
    },
    "ValidationSettings": {
      "EnableValidation": true,
      "ThreadCount": 5,
      "PartitionSize": 10000,
      "FailureMaxCount": 10000
    }
  }' \
  --region <aws-region>
```

Key settings:
- `TargetTablePrepMode: DO_NOTHING` — don't modify target data, just validate
- `EnableValidation: true` — run row-by-row comparison
- Check results in the `awsdms_validation_failures_v2` (or `_v1`) table on target (queries: Path 3 Step 4)

## Deliverables

- Data migration completed via chosen path (offline, native replication, DMS, or hybrid)
- Data validation completed (exact row counts and checksums, or DMS validation together with exact row counts)
- Refactored application code tested against Aurora in preview/staging (if platform SDK refactoring was needed)
- Application cutover completed — pointing to target database
- Refactored application code merged to main (if platform SDK refactoring was needed)
- Post-cutover monitoring active for 24–48 hours
- Rollback procedure documented and tested
- Migration runbook updated with actual timings and any issues encountered
- Source database decommissioning plan (after stabilization period)

## Post-Migration Activities

### Immediate (0–7 Days)

- Hypercare support: dedicated team monitoring for issues
- Monitor error rates, latency, CPU, memory, connections, I/O
- Performance tuning: identify and fix slow queries on the new target
- Resolve any data discrepancies found during monitoring
- Validate backup and restore procedures on Aurora
- Test failover scenarios (manual failover to read replica and back)
- Update operational documentation with Aurora-specific procedures

### Short-Term (1–4 Weeks)

- Right-size Aurora instance based on actual production metrics
- Optimize parameter groups (shared_buffers, work_mem, effective_cache_size)
- Implement read replicas if read-heavy workload warrants it
- Configure RDS Proxy if connection pooling is needed
- Fine-tune monitoring thresholds and alerting rules
- Conduct lessons-learned session with the migration team
- Archive migration artifacts (runbooks, validation reports, DMS logs)

### Source Decommissioning

After the stabilization period (typically 2–4 weeks of stable operation on Aurora):

- Verify no applications are still connecting to the source database
- Confirm no orphaned replication slots on the source (`SELECT * FROM pg_replication_slots;`)
- Stop and delete DMS replication tasks and resources, then remove the slot and the `awsdms_` helper objects from the source ([DMS Cleanup on the Source](#dms-cleanup-on-the-source))
- Take a final backup/snapshot of the source database for archival
- Decommission the source database instance
- Update architecture diagrams and CMDB records

### DMS Cleanup on the Source

DMS writes helper objects to the source and holds a replication slot while the task exists ([Mobilize Step 4b](02-mobilize-phase.md#step-4b-what-dms-writes-to-the-source)). If the source still takes writes, drop the slot as soon as the task is stopped and the cutover is accepted, because a stopped task's slot keeps holding write-ahead log on the source. Remove the helper objects after the stabilization period. Order:

1. Stop the task and wait for `stopped`, then delete it (`aws dms delete-replication-task --replication-task-arn <task-arn>`). Ask first.
2. Find the slot with a read-only query. `active` is false once the task is stopped:

   ```sql
   select slot_name, active from pg_replication_slots;
   ```

3. The user runs the source-side commands (the agent does not write to the source). Use the slot name from step 2 and the schema the helper objects live in. `<schema>` is the value of `HeartbeatSchema` and `DdlArtifactsSchema` on the source endpoint, and `public` when they are not set. Drop the event trigger before the audit table it writes to, in the order the [DMS documentation for removing its artifacts from a PostgreSQL source](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Source.PostgreSQL.html) gives:

   ```sql
   select pg_drop_replication_slot('<slot>');
   drop event trigger if exists awsdms_intercept_ddl;
   drop function if exists <schema>.awsdms_intercept_ddl();
   drop table if exists <schema>.awsdms_heartbeat, <schema>.awsdms_ddl_audit;
   ```

   If the audit table is dropped while the event trigger still exists, DDL on the source fails (on a Supabase project that includes the platform's own migrations) until the trigger is dropped. Do not drop the schema itself: it can hold application tables. Check what is left with `select evtname from pg_event_trigger where evtname like 'awsdms\_%';` and `select n.nspname, p.proname from pg_proc p join pg_namespace n on n.oid = p.pronamespace where p.proname like 'awsdms\_%';`, and drop anything they return in the same order (trigger, function, table).
4. Verify with read-only queries: `pg_replication_slots` no longer lists the task's slot, the event trigger is gone, and no `awsdms_` tables remain:

   ```sql
   select to_regclass('<schema>.awsdms_heartbeat') as heartbeat, to_regclass('<schema>.awsdms_ddl_audit') as ddl_audit;
   select table_schema, table_name from information_schema.tables where table_name like 'awsdms\_%';
   select evtname from pg_event_trigger where evtname like 'awsdms\_%';
   ```

   Both `to_regclass` values are null and the second and third queries return no rows.

Do this before you decommission the source. An unused slot keeps WAL on the source.

## Common Challenges & Solutions

### High Replication Lag

- Increase DMS replication instance size (or use DMS Serverless with higher DCU)
- Optimize long-running transactions on the source (large transactions generate more WAL)
- Batch large data changes into smaller transactions
- For native replication: check `max_wal_senders` and network throughput

### Tables Without Primary Keys

This is the one rule for tables without a primary key, used in every file of this guide. Per the AWS documentation, a captured table must have a primary key: for a table without one, AWS DMS ignores DELETE and UPDATE operations during CDC, and AWS does not recommend migrating such a table with DMS because further limitations apply, among them no data validation ([PostgreSQL as a DMS source: limitations](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Source.PostgreSQL.html)). DMS data validation needs a primary key or unique index, and a table without one shows the validation state `No primary key` ([DMS data validation](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Validating.html)). So DMS does not fix a table without a primary key. Choose one of these for each such table:

1. Add a primary key before the migration. The user runs the DDL on the source. Preferred.
2. Migrate the table offline during the cutover window (pg_dump/pg_restore per table), and verify it with exact `count(*)` and checksums.
3. Set `REPLICA IDENTITY FULL` on the source table so the write-ahead log carries the old row. The documentation lists this as a workaround and advises it only after careful testing, because it generates extra log volume. It is supported with the `test_decoding` plugin and not with `pglogical`. The user runs the `ALTER TABLE` on the source. DMS validation still does not cover the table, so verify it with exact counts and checksums.

Add primary keys or unique constraints to these tables after the migration as a long-term fix.

### ENUM Types Not Migrating (DMS)

- DMS does not migrate PostgreSQL ENUM types
- Solution: create ENUM types manually on the target before starting the DMS task
- Extract ENUMs from source: `SELECT typname, enumlabel FROM pg_enum JOIN pg_type ON pg_enum.enumtypid = pg_type.oid ORDER BY typname, enumsortorder;`

### Performance Differences After Migration

- Run `ANALYZE VERBOSE;` to ensure the query planner has fresh statistics
- Compare execution plans (`EXPLAIN ANALYZE`) for key queries between source and target
- Adjust Aurora parameter group settings (work_mem, effective_cache_size, random_page_cost)
- Check for missing indexes (queries that were fast on source but slow on target)

### SSL/TLS Certificate Issues

- Aurora requires SSL by default — download the RDS CA certificate bundle
- Update application connection strings to include `sslmode=require` (or `verify-full` for strict validation)
- For `verify-full`: set `sslrootcert` to the downloaded RDS CA bundle path

### Extension Compatibility

- Verify all source extensions are available on Aurora/RDS before migration
- Use AWS Knowledge MCP `search_documentation` to check extension support
- If an extension is not available, find an alternative or implement the functionality in the application layer

### Foreign Key Handling for DMS Full Load

DMS loads tables in parallel during full load (`MaxFullLoadSubTasks` sets how many at once; default 8, see [AWS DMS full-load task settings](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Tasks.CustomizingTasks.TaskSettings.FullLoad.html)), so table order is not guaranteed. Foreign key constraints can cause failures if child rows arrive before parent rows. For a schema that was preloaded on the target, two settings apply together ([Decision 5](02-mobilize-phase.md#decision-5-dms-target-table-preparation-mode) for the prep mode, [Decision 6](02-mobilize-phase.md#decision-6-foreign-key-handling-for-dms-full-load-reference) for the bypass):

1. **Set `session_replication_role = 'replica'`** on the target (through `AfterConnectScript` in the DMS target endpoint settings, or directly on Aurora) to deactivate FK checks and user triggers during the DMS session. When set via `AfterConnectScript`, this applies to both full load and CDC phases within the DMS session. Remove the `AfterConnectScript` only after migration cutover is complete and the application is writing directly to Aurora. See `02-mobilize-phase.md` → Decision 8 for trigger timing details.
2. **Set `TargetTablePrepMode: DO_NOTHING`** in the DMS task, on an empty target, to avoid DROP/TRUNCATE failures caused by PostgreSQL's FK failsafe mechanism.

For reload scenarios where `TRUNCATE` or `DROP` fails due to FK dependencies, use `TRUNCATE ... CASCADE` — but you MUST reload ALL cascaded tables, not just the one you truncated.

See `troubleshooting.md` → "DMS Full Load + CDC Foreign Key Handling" and "DMS Truncate/Drop Cascade for Reload Scenarios" for detailed resolution steps, verification queries, and the full procedure.

### DMS Log Analysis

DMS task logs are in CloudWatch under the log group `dms-tasks-<replication-instance-id>` (stream `dms-task-<task-external-id>`). Key log signatures:

- `]E:` — Error messages (critical, usually cause task failure)
- `]W:` — Warning messages (data issues, task continues)
- `]I:` — Informational (normal operation)

Quick error search:

```bash
aws logs filter-log-events \
  --log-group-name dms-tasks-<replication-instance-id> \
  --filter-pattern "]E:" \
  --region <region> \
  --query "events[*].message" --output text
```

When the message says `Please check target database logs for more information`, read the Aurora PostgreSQL error log yourself: see `troubleshooting.md` → "DMS table error: exitwhy 1 / check target database logs". Aurora logs appear in CloudWatch Logs under `/aws/rds/cluster/<cluster-id>/postgresql` only when the `postgresql` log export is on.

See `troubleshooting.md` → "Reading DMS Task Logs" for the full troubleshooting workflow, common error patterns, and resolution steps.

### DMS Validation Mismatch on Boolean Columns

Observed on one run: with validation on, a table with a boolean column ended as `Mismatched records` while the data was identical. The failure rows were `RECORD_DIFF`. Decoded through the key, `"DETAILS"` held the same value in two representations: `'\u0001'` against `'1'` where the target value was true, and `'<null>'` against `'0'` where it was false (a NUL byte prints as `<null>`). The `md5(t::text)` of every failed key was identical on both sides, and exact row counts and `md5(string_agg(x::text, ',' ORDER BY pk))` checksums were equal on every table, with the session time zone set to UTC on both sides. A task built by hand earlier showed the same.

Rules:

1. Never act on a validation mismatch alone. Read the failures table first and decode it with the key lookup: [DMS validation: Mismatched records](troubleshooting.md#dms-validation-mismatched-records). The queries are in Path 3 Step 4 above (`#step-4-monitor-dms-validation`).
2. Fetch the failed rows by `"KEY"` on both sides with an independent query (`md5(t::text)` per row), and run a counts-and-checksums comparison: exact row counts and `md5(string_agg(x::text, ',' ORDER BY pk))` per table, with the session time zone set to UTC on both sides. Equal checksums are the deciding evidence. Do not rely on the order of the two entries in `"DETAILS"`: the AWS page does not say which entry is the source.
3. If only the representation differs and the checksums match, record it as a validation representation difference and continue. If values differ, treat it as data loss and stop.
4. Keep validation on. Do not suppress it.

Not claimed: the cause of the representation difference inside DMS was not determined, moving the settings between the extra connection attribute string and `PostgreSQLSettings` did not change the outcome, and this is not claimed to be a DMS defect. The same entry for troubleshooting: `troubleshooting.md` → "DMS validation mismatch on boolean columns".

## Success Criteria

- All tables migrated with zero data loss
- Exact row counts (`count(*)`) and object counts match between source and target
- Application functionality validated (including refactored code if applicable)
- Performance meets or exceeds source baseline (P50, P95, P99 latency)
- Downtime within the approved maintenance window
- No critical errors in application logs for 24–48 hours post-cutover
- Monitoring and alerting operational on the target
- Rollback plan tested (even if not executed)

## Timeline

- Single database (< 100 GB): 1–2 weeks (including validation and stabilization)
- Single database (100 GB–1 TB): 2–4 weeks
- Multi-database portfolio: 8–24 weeks (using wave-based approach)

## Best Practices

- Always use private connectivity — no public database endpoints
- Test the full migration path in a non-production environment first
- Validate data integrity at every step (after full load, during CDC, after cutover)
- Keep replication lag under 5 seconds (DMS) or 1 MB (native) before cutover
- Use AWS Secrets Manager for all database credentials
- Enable SSL/TLS for all connections
- Monitor replication continuously — don't start cutover without confirming lag is near zero
- Have the rollback plan ready and rehearsed before cutover
- Document everything: timings, issues, resolutions, deviations from the runbook
- Communicate proactively with stakeholders before, during, and after cutover
