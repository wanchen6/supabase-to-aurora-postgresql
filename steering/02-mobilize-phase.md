# Mobilize Phase — PostgreSQL Migration

## Overview

The Mobilize phase provisions the target PostgreSQL environment (Aurora or RDS), migrates the schema, configures the source database for replication, sets up the chosen migration tooling (native logical replication and/or AWS DMS), and runs a pilot migration to validate the approach before full execution.

## Migration Context

- Source: PostgreSQL databases hosted on 3rd-party platforms (Supabase, Heroku, Neon, GCP Cloud SQL, Azure Database, on-premises, etc.)
- Target: AWS RDS PostgreSQL or Amazon Aurora PostgreSQL
- Approach: Database-focused migration with application connection update or refactoring as needed
- Focus: Provision the target, migrate the schema, configure replication, and validate with a pilot

## Outline

1. [Mobilize Decision Tree](#mobilize-decision-tree)
2. [Target Environment Setup](#target-environment-setup)
3. [Schema Migration](#schema-migration)
4. [ENUM Types (DMS Requirement)](#enum-types-dms-requirement)
5. [Source Database Configuration](#source-database-configuration)
6. [Native Logical Replication Setup](#native-logical-replication-setup)
7. [AWS DMS Setup (Fallback Path)](#aws-dms-setup-fallback-path)
8. [DMS Validation Setup](#dms-validation-setup)
9. [Pilot Migration](#pilot-migration)

## Quick Navigation

| Decision | Summary |
| -------- | ------- |
| Decision 0 | Network assessment — MANDATORY before provisioning any resource |
| Decision 1 | Schema migration method (DMS auto-create vs pre-create vs pg_dump) |
| Decision 2 | Schema export scenario (simple, ENUM-aware, partitioned, complex) |
| Decision 3 | Replication path setup (native / DMS / hybrid / offline) |
| Decision 4 | DMS logical decoding plugin (test_decoding vs pglogical) |
| Decision 5 | DMS target table preparation mode |
| Decision 6 | Foreign key handling for DMS full load |
| Decision 7 | Index and constraint creation timing |
| Decision 8 | Trigger handling during DMS migration |
| Decision 9 | Pilot migration scope |
| Decision 10 | DMS validation strategy |
| Decision 11 | Sequence handling |

## Mobilize Decision Tree

Use this decision tree to guide the key decisions during the Mobilize phase. These decisions build on the Assess phase outcomes and determine how the target environment, schema, replication, and pilot are configured.

### Decision 0: Network Assessment (MANDATORY before provisioning)

**CRITICAL**: Complete this assessment BEFORE creating Aurora, DMS, or any VPC resources. Provisioning without this assessment leads to delete/recreate cycles.

**Step 1: Detect source connectivity**

```bash
# Check if source has IPv4 (A record)
dig A db.<project-ref>.supabase.co +short
# Check if source has IPv6 (AAAA record)
dig AAAA db.<project-ref>.supabase.co +short
```

Result determines online migration path:
- **IPv4 available** → Prefer native logical replication (CREATE PUBLICATION + CREATE SUBSCRIPTION). Aurora connects outbound via NAT gateway to Supabase's IPv4 address.
- **IPv6 only** → Online: DMS (publicly accessible, dual-stack) with CDC, or native replication once the IPv4 add-on and NAT provide an IPv4 route. A direct subscription from Aurora to an IPv6-only publisher is not yet validated. Offline: pg_dump via bastion. Steps: `supabase-prerequisites-and-connectivity.md` → "Online Migration from IPv6-Only Supabase Sources".

Also check: Supabase network restrictions (API call), Supabase IPv4 add-on status.

**Step 2: Assess target VPC**

```bash
# List all subnets with CIDRs, AZs, IPv6 status
aws ec2 describe-subnets --filters "Name=vpc-id,Values=<vpc-id>" \
  --query "Subnets[*].{SubnetId:SubnetId,AZ:AvailabilityZone,Cidr:CidrBlock,Ipv6:Ipv6CidrBlockAssociationSet[0].Ipv6CidrBlock}"

# Check route tables for each subnet (identify private vs public)
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=<vpc-id>" \
  --query "RouteTables[*].{RTId:RouteTableId,Routes:Routes[?DestinationCidrBlock=='0.0.0.0/0'].GatewayId}"

# Check for NAT gateways
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=<vpc-id>" "Name=state,Values=available"

# Check existing resources in each subnet
aws ec2 describe-network-interfaces --filters "Name=subnet-id,Values=<subnet-id>" \
  --query "length(NetworkInterfaces)"
```

**Step 3: Determine Aurora subnet placement**

```
Are there existing private subnets (no IGW route) in 2+ AZs with no conflicting resources?
├── YES → Use them for Aurora DB subnet group
└── NO
    ├── Are there subnets with only migration-related resources that can be converted?
    │   └── YES → Convert to private (change route table)
    └── NO → Create new private subnets
        ├── Calculate available CIDR blocks (see Subnet CIDR Calculation below)
        ├── Need minimum 2 subnets in different AZs
        └── Present proposed CIDRs to user for approval
```

**Step 4: Determine migration connectivity path**

Present cost comparison to user:

| Approach | When Source is IPv6-only | When Source has IPv4 | Cost/week | Complexity |
|----------|------------------------|---------------------|-----------|------------|
| DMS (`dms.t3.small`, publicly-accessible, dual-stack) | ✅ Works | ✅ Works | ~$6 | Medium |
| Native replication + NAT + IPv4 add-on | Requires IPv4 add-on ($4/mo) + NAT ($32/mo) | ✅ Works (NAT only) | ~$9 | Low |
| pg_dump at cutover (bastion with IPv6) | ✅ Works | ✅ Works | ~$0.01 | Lowest (seconds downtime) |

For an IPv6-only source, the online route is DMS in dual-stack mode or native replication over the IPv4 add-on path. A direct Aurora subscription to an IPv6-only publisher is not yet validated (Verification status table in `supabase-prerequisites-and-connectivity.md`).

The DMS cost is compute only: `dms.t3.small`, Single-AZ, 168 hours at the US East (N. Virginia) on-demand rate in the [AWS DMS price list](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AWSDatabaseMigrationSvc/current/index.json) (about $6 per week; an estimate). Rates differ by Region and instance class. Check the live [AWS DMS pricing page](https://aws.amazon.com/dms/pricing/) before quoting a figure to the user. The IPv4 add-on ($4/month) and NAT gateway ($32/month) figures in the table are estimates that were not re-checked for this release; check before relying on it (current Supabase and AWS prices).

**Step 5: Present networking plan to user**

Before provisioning, present:
- Proposed subnets (existing or new, with CIDRs)
- Proposed route tables and gateways
- Migration approach with cost
- Get explicit user approval

### Subnet CIDR Calculation

Before creating subnets, ALWAYS:

```bash
# 1. List all existing subnet CIDRs
aws ec2 describe-subnets --filters "Name=vpc-id,Values=<vpc-id>" \
  --query "Subnets[*].CidrBlock" --output text

# 2. Identify VPC CIDR range
aws ec2 describe-vpcs --vpc-ids <vpc-id> --query "Vpcs[0].CidrBlock"

# 3. Calculate available ranges (avoid overlap with ANY existing subnet)
# For a /16 VPC (e.g., 172.31.0.0/16) with existing /20 subnets:
# Existing: 172.31.0.0/20, 172.31.16.0/20, 172.31.32.0/20, 172.31.48.0/20, 172.31.64.0/20, 172.31.80.0/20
# Available for /28: 172.31.96.0/28, 172.31.96.16/28, etc. (verify each doesn't conflict)

# 4. Present to user: "I'll create subnet-a (X.X.X.X/28 in AZ-a) and subnet-b (Y.Y.Y.Y/28 in AZ-b)"
# 5. Only create after user approval
```

NEVER create subnets without verifying CIDR availability first. CIDR conflicts require delete/recreate which wastes time.

### DMS Creation Checklist (Generic)

For the complete DMS creation steps with exact settings (secrets, IAM role, subnet group, security group, replication instance, endpoints), see "AWS DMS Setup (Fallback Path)" in this file. The connectivity rules are in `supabase-prerequisites-and-connectivity.md` → "DMS Connectivity Prerequisites".

### Decision 1: Schema Migration Method

First, decide HOW to get the schema onto Aurora. Then decide the scenario based on schema complexity.

```
Is DMS part of the migration? (from Assess phase)
├── YES (DMS full-load or full-load-and-cdc)
│   ├── Is the schema simple? (no FKs, no CHECK constraints, no custom ENUM/domain types,
│   │   no partitions, no triggers — ONLY columns and PKs that DMS creates)
│   │   └── YES → Method A: Let DMS create tables automatically
│   │       Use TargetTablePrepMode: DROP_AND_CREATE
│   │       DMS creates tables with columns and PKs during full load
│   │       ★ Simplest — no separate schema migration step, no additional objects to manage
│   │
│   └── NO (schema has FKs, CHECK constraints, triggers, indexes, or other objects
│       beyond what DMS creates) → Method B: Pre-create schema, DMS uses DO_NOTHING
│       Export: pg_dump --schema-only from source (exclude platform schemas)
│       Import: Use one of the execution methods below (Data API, CloudShell, or EC2 bastion)
│       Then configure DMS task with TargetTablePrepMode: DO_NOTHING
│       Handle additional objects based on database size and type:
│       │
│       ├── Secondary indexes:
│       │   ├── Small DB (< 10 GB): Keep indexes during full load (overhead negligible,
│       │   │   full load completes in seconds/minutes regardless)
│       │   └── Large DB (≥ 10 GB): Defer indexes — create at StopTaskCachedChangesNotApplied
│       │       (speeds up full load, indexes then speed up CDC apply)
│       │
│       ├── Foreign keys:
│       │   ├── Preferred: Use AfterConnectScript: "SET session_replication_role = 'replica';"
│       │   │   on the DMS target endpoint (bypasses FK checks during full load AND CDC)
│       │   │   FKs can be added at ANY time since the DMS session bypasses them —
│       │   │   they only enforce on direct app connections after cutover
│       │   └── Fallback (if AfterConnectScript is not applicable):
│       │       Add FK constraints at StopTaskCachedChangesApplied (target is consistent)
│       │       See Decision 6 reference for details
│       │
│       ├── Triggers:
│       │   ├── Do NOT create triggers before or during migration
│       │   ├── AfterConnectScript bypasses triggers during both full load AND CDC
│       │   │   (confirmed by testing — see Decision 8 reference)
│       │   └── Create triggers ONLY after migration cutover when app writes directly to Aurora
│       │
│       ├── CHECK constraints:
│       │   └── Add at StopTaskCachedChangesApplied (target is consistent, data satisfies checks)
│       │       Or add at any time if using AfterConnectScript (DMS session bypasses checks)
│       │
│       ├── Partitioned tables:
│       │   └── Create partitioned table structure on target BEFORE DMS full load
│       │       (parent table + child partitions with correct bounds)
│       │       DMS inserts into the parent table; PostgreSQL partition routing
│       │       automatically places rows in the correct partition
│       │       Reference: https://aws.amazon.com/blogs/database/migrate-to-native-partitioned-tables-in-postgresql-10-using-aws-database-migration-service/
│       │
│       ├── Default values (e.g., DEFAULT gen_random_uuid()):
│       │   └── Include in pre-created schema — DMS inserts explicit values so defaults
│       │       don't conflict, but defaults are needed post-migration for app inserts
│       │
│       └── Sequences:
│           └── Include in pre-created schema — reset values at cutover (Decision 11)
│
└── NO (offline pg_dump/pg_restore only)
    └── Method C: Schema migrated as part of pg_dump/pg_restore
        pg_dump exports schema + data together; pg_restore applies both
        No separate schema migration step needed
```

**Execution methods for Method B** (when you need to run DDL against private Aurora):

| Method | Ease | Reliability | Cost | When to Use |
| ------ | ---- | ----------- | ---- | ----------- |
| **EC2 bastion + SSM** (recommended for automation) | High (agent provisions and runs via SSM) | Highest (full psql, scriptable) | ~$0.01/hr (t4g.nano) | Default for seamless automation without user intervention |
| **CloudShell VPC environment** (recommended for manual) | Medium (user opens console) | Highest (full psql) | Free | When user prefers manual control or cost is a concern |
| **Aurora Data API** (recommended for agent) | High (agent runs directly, no VPC needed) | High (works from anywhere, private subnets included) | ~$1/million requests (32KB/request unit). Free tier: 1M requests/month for first year | DDL, verification queries, and schema operations from outside VPC. Must use transactions for multi-statement sequences. |

**Aurora Data API notes:**
- Works with private Aurora clusters — calls go through AWS control plane (HTTPS), not VPC networking
- Does NOT support multi-statement queries in a single call. Use transactions: `begin-transaction` → multiple `execute-statement` → `commit-transaction`
- 45-second execution timeout per statement
- 1 MiB response size limit
- Enable with: `aws rds enable-http-endpoint --resource-arn <cluster-arn> --region <region>` (NOT `modify-db-cluster`)
- For operations exceeding these limits, use CloudShell or EC2 bastion + SSM

**Aurora PostgreSQL MCP server** — uses Data API (`rdsapi` connection method) or pgwire. Same cost as Data API when using `rdsapi`. Subject to injection filters that block `CREATE FUNCTION` with `$$`, `DROP TABLE/DATABASE/SCHEMA`, and multi-statement queries.

**Agent decision logic:**
1. If DMS is being used AND schema has ONLY columns and PKs (no FKs, no CHECK, no triggers, no indexes): use Method A (DMS creates tables). Simplest path.
2. If DMS is being used AND schema has any additional objects: use Method B (pre-create schema via Data API or CloudShell, DMS uses DO_NOTHING). Handle additional objects per Decision 1 tree above.
3. If no DMS (offline migration): use Method C (pg_dump/pg_restore handles everything).
4. Skip tables named `awsdms_*` when exporting or pre-creating the schema: they are leftovers from earlier DMS tasks (`--exclude-table='*.awsdms_*'` for `pg_dump`).
5. For running DDL against private Aurora: prefer Data API (works from anywhere, no VPC needed) or CloudShell (free). Use EC2 bastion for complex scripts that exceed Data API's 45-second timeout.

### Decision 2: Schema Export Scenario (for Method B only)

If using Method B (pre-create schema), choose the export/import approach based on schema characteristics. This decision is about HOW to get the DDL onto the target — object timing (indexes, FKs, triggers) is handled by Decision 1.

```
Does the schema have ENUM types or custom domains?
├── NO
│   ├── Does the schema have partitioned tables?
│   │   ├── NO → Scenario A: Simple schema (single pg_dump --schema-only + apply via Data API or psql)
│   │   └── YES → Scenario C: Create partitioned structure on target first (see Scenario C details below)
└── YES → Scenario B: ENUM/domain-aware schema migration
    ├── Try Scenario A first (pg_dump usually orders types correctly)
    └── If type errors → Extract types first, apply types, then apply full schema
```

Is the schema very complex with many interdependencies?
- YES → Scenario E: TOC reordering (pg_restore --use-list) for full control over object creation order

### Decision 3: Replication Path Setup

```
What migration approach was selected in the Assess phase?
├── Offline (pg_dump/pg_restore)
│   └── No replication setup needed — skip to Pilot Migration
├── Native logical replication
│   └── Set up publication + replication slot on source, subscription on target
│       └── Does the source platform support logical replication?
│           ├── YES → Proceed with native replication setup
│           └── NO → Fall back to DMS with CDC
├── DMS with CDC
│   └── Set up DMS replication instance, endpoints, and task
│       └── Proceed to Decision 4 (plugin choice)
└── Hybrid (native + DMS)
    ├── Tables WITH PKs and standard types → Native logical replication
    ├── Tables needing validation → DMS with CDC + validation
    └── Tables WITHOUT PKs → Offline (pg_dump per table at cutover)
```

### Decision 4: DMS Logical Decoding Plugin

```
Is DMS part of the migration? (from Assess Decision 5)
├── NO → Skip (not applicable)
└── YES
    ├── Is the pglogical extension available on the source?
    │   ├── NO (Supabase, Neon, most managed platforms)
    │   │   └── Use test_decoding (built into PostgreSQL, no extension needed)
    │   └── YES (GCP Cloud SQL, Azure, RDS, Aurora, on-premises)
    │       ├── Are ALL source tables included in CDC?
    │       │   ├── YES → test_decoding is fine (no filtering advantage from pglogical)
    │       │   └── NO (only a subset of tables in CDC)
    │       │       └── Use pglogical (filters non-CDC tables at slot level, reduces WAL overhead)
    │       └── Unsure → test_decoding (safe default, always available)
    └── Set extra-connection-attributes: pluginName=<choice>;heartbeatEnable=true;heartbeatFrequency=5;mapBooleanAsBoolean=true
```

### Decision 5: DMS Target Table Preparation Mode

This is the single statement of the rule. Other files link here. Record the value in the migration notes (the Mobilize decision summary table).

```
Is DMS part of the migration?
├── NO → Skip
└── YES
    ├── Is this a reload after a failed or partial run?
    │   └── YES → Stop the task, TRUNCATE ... CASCADE every task table on the target,
    │            then DO_NOTHING (recovery steps: troubleshooting.md,
    │            "Reload after a failed or partial run")
    ├── Did the power preload the schema on the target (Method B: FKs, CHECKs, indexes, triggers)?
    │   └── YES → DO_NOTHING, and the target tables must be empty before the task starts
    │       ├── TRUNCATE_BEFORE_LOAD fails on tables that foreign keys reference, even on an
    │       │   empty target and even with session_replication_role = 'replica'
    │       │   (DMS log: "cannot truncate a table referenced in a foreign key constraint")
    │       ├── Do not use TRUNCATE_BEFORE_LOAD for a preloaded schema
    │       └── DO_NOTHING keeps whatever rows are there: a non-empty target gives duplicate key errors
    └── Does DMS create the tables (Method A)?
        └── YES → DROP_AND_CREATE
            ├── Do not use TRUNCATE_BEFORE_LOAD on tables that foreign keys reference
            └── Warning: DMS builds columns and primary keys only. Indexes, constraints and triggers must be added after the load
```

Decision 6 (foreign key bypass) is separate and applies together with `DO_NOTHING` for a preloaded schema.

### Decision 6: Foreign Key Handling for DMS Full Load (Reference)

**This section provides detailed guidance for FK handling. Decision 1 references this for the fallback case when `AfterConnectScript` is not applicable. The bypass applies together with `DO_NOTHING` for a preloaded schema (the prep mode rule is stated once, in Decision 5). Record the choice in the migration notes.**

**CRITICAL — MUST be configured BEFORE starting the DMS task. Failure to handle FKs causes `Table error` on child tables during full load because DMS loads tables in parallel and child rows may arrive before parent rows exist.**

```
Does the target schema have foreign keys?
├── NO → No special handling needed
└── YES
    ├── Option A (STRONGLY recommended): session_replication_role = 'replica' via the DMS target endpoint `AfterConnectScript`
    │   ├── Add to target endpoint PostgreSQLSettings when CREATING the endpoint:
    │   │   "AfterConnectScript": "SET session_replication_role = 'replica';"
    │   ├── Bypasses FK checks, triggers, and rules during the DMS session
    │   ├── No need to drop/recreate FKs — simplest and safest approach
    │   ├── Works during both full load AND CDC phases
    │   └── After migration cutover: modify endpoint to remove AfterConnectScript
    └── Option B (NOT recommended for CDC): Manually drop FKs before load, re-add after
        ├── Fragile — dropping FKs on a live DB with CDC is complex and error-prone
        ├── Generate drop/re-add scripts from pg_constraint
        ├── Drop FKs → run DMS full load → re-add FKs
        └── Only use if session_replication_role is not available
```

**Implementation — add AfterConnectScript to the target endpoint settings (the complete command is Step 5 of "AWS DMS Setup (Fallback Path)" below):**

```json
{
  "EndpointIdentifier": "dms-target-<project-name>",
  "EndpointType": "target",
  "EngineName": "aurora-postgresql",
  "DatabaseName": "<database>",
  "SslMode": "require",
  "PostgreSQLSettings": {
    "SecretsManagerSecretId": "<target-secret-arn>",
    "SecretsManagerAccessRoleArn": "<dms-secrets-role-arn>",
    "AfterConnectScript": "SET session_replication_role = 'replica';"
  }
}
```

Note: `ALTER SYSTEM` is not allowed on RDS/Aurora — use the parameter group for `session_replication_role` if you need it cluster-wide. The `AfterConnectScript` approach is session-scoped and preferred.

### Decision 7: Index and Constraint Creation Timing (Reference)

**This section provides detailed guidance for the two-stop approach. Decision 1 references this for large databases (≥ 10 GB) where deferring indexes improves full load performance.**

DMS `full-load-and-cdc` tasks accumulate CDC changes in a cache while the full load runs. The `StopTaskCachedChanges*` settings control what happens after full load completes:

- **`StopTaskCachedChangesNotApplied = true`** — Task stops BEFORE applying cached CDC changes. Target has full-load data only (**not transactionally consistent** with source). Use this window to add **secondary indexes** — they speed up the subsequent CDC apply phase.
- **`StopTaskCachedChangesApplied = true`** — Task stops AFTER applying all cached CDC changes. Target is **transactionally consistent** at a clean boundary. Use this window to add **foreign keys** and **CHECK constraints**.

```
Are you using DMS for full load?
├── NO (pg_dump/pg_restore handles indexes automatically) → No decision needed
└── YES
    ├── Recommended: Two-stop approach for optimal performance and consistency
    │   ├── Set StopTaskCachedChangesNotApplied = true in task settings
    │   ├── Full load completes → task stops (target NOT consistent yet)
    │   ├── ACTION: Add secondary indexes now (speeds up CDC apply)
    │   ├── Resume task → DMS applies cached changes + continues CDC
    │   ├── CDC catches up (lag → 0), ready for cutover
    │   ├── Stop task with StopTaskCachedChangesApplied or cdc-stop-position commit_time
    │   ├── Target is now CONSISTENT at a transaction boundary
    │   └── ACTION: Add foreign keys and CHECK constraints now
    ├── Alternative: Single-stop approach (simpler, slightly slower CDC apply)
    │   ├── Set StopTaskCachedChangesApplied = true only
    │   ├── Full load + cached changes applied → task stops (target consistent)
    │   ├── ACTION: Add secondary indexes, FKs, and CHECK constraints together
    │   └── Resume task → CDC continues (indexes already in place)
    └── Simplest: No stop (Don't stop)
        ├── Task runs straight through full load → cached changes → CDC
        ├── Add indexes, FKs, and constraints while CDC is running
        └── Acceptable for small databases where full load is fast
```

**Do NOT add triggers at either stop point** — triggers should only be created after migration cutover (see Decision 8).

**Reference**: [Achieve transaction consistency with DMS](https://aws.amazon.com/blogs/database/achieve-transaction-consistency-on-your-target-database-when-using-multiple-tasks-with-aws-dms-replication/) — especially important when using multiple DMS tasks; use `cdc-stop-position commit_time` to stop all tasks at the same source transaction point.

### Decision 8: Trigger Handling (Reference)

**This section provides detailed guidance for trigger behavior during DMS migration. Decision 1 references this — triggers should only be created after cutover.**

**Key distinction**: The behavior of triggers during DMS migration depends on HOW `session_replication_role = 'replica'` is set:

- **Via Aurora cluster parameter group** (global): Bypasses triggers for ALL sessions on the cluster, including CDC apply. Triggers will NOT fire during CDC. Only use this if you want triggers completely disabled cluster-wide.
- **Via DMS target endpoint `AfterConnectScript`** (session-scoped): Bypasses triggers and FKs during the DMS session only. Per AWS documentation, this "has AWS DMS bypass foreign keys and user triggers to reduce the time it takes to bulk load data." During CDC phase, DMS applies changes through the same session, so triggers are also bypassed during CDC.

**IMPORTANT**: If triggers exist on the target and you want them to fire during CDC (e.g., audit triggers that should log replicated changes), do NOT use `session_replication_role = 'replica'` — instead, manually drop FKs before full load and re-add them after. However, this is rarely desired because triggers firing on replicated changes typically duplicate effects that already happened on the source.

```
Does the target schema have triggers (from schema migration)?
├── NO → No action needed
└── YES
    ├── Are triggers for audit logging or side effects?
    │   └── Do you want them to fire on replicated CDC changes?
    │       ├── NO (typical) → session_replication_role = 'replica' handles this
    │       │   Triggers are bypassed during both full load AND CDC
    │       │   Re-enable triggers ONLY after cutover (remove AfterConnectScript or
    │       │   reset parameter group), when the app writes directly to Aurora
    │       └── YES (rare) → Do NOT use session_replication_role = 'replica'
    │           Use manual FK drop/re-add instead, and keep triggers active
    │           WARNING: This may cause duplicate effects if the trigger's action
    │           already occurred on the source
    ├── Are triggers for data integrity (e.g., computed columns)?
    │   └── Same logic as above — if the source trigger already computed the value,
    │       the replicated row already has the correct value. Firing the trigger
    │       again on the target is redundant (or harmful if it overwrites with a
    │       different computation). Keep triggers bypassed during migration.
    └── Are triggers for replication or platform-specific features?
        └── Remove them entirely (they belong to the source platform, not the target)
```

**When to re-enable triggers**: ONLY after migration cutover is complete and the application is writing directly to Aurora. At that point:
1. Remove `AfterConnectScript` from the DMS target endpoint (if DMS is still running for rollback)
2. Or reset the cluster parameter group `session_replication_role` back to default
3. Verify triggers fire correctly on new application writes

### Decision 9: Pilot Migration Scope

```
Is this a multi-database migration (10+ databases)?
├── YES → Run a full pilot migration on one non-production database
└── NO
    ├── Is the team experienced with PostgreSQL replication / DMS?
    │   ├── YES
    │   │   ├── Is the database small (< 10 GB) and simple?
    │   │   │   └── YES → Skip pilot, proceed to full migration
    │   │   └── NO → Run a table-level pilot (3–5 representative tables)
    │   └── NO → Run a table-level pilot (3–5 representative tables)
    └── Is the schema complex (partitions, many FKs, custom types)?
        └── YES → Run a table-level pilot regardless of team experience
```

### Decision 10: DMS Validation Strategy

```
Is DMS part of the migration?
├── NO → Use offline validation (row counts, checksums, sample comparisons)
│   └── See 03-migrate-phase.md "Offline Validation" section
└── YES
    ├── Is DMS performing the full load + CDC?
    │   └── YES → Enable validation in the DMS task settings
    │       └── EnableValidation: true, ThreadCount: 5, FailureMaxCount: 10000
    ├── Is DMS used only for specific tables (hybrid)?
    │   └── YES → Enable validation on the DMS task for those tables
    │       └── Use offline validation for tables migrated via native replication or pg_dump
    └── Is DMS not migrating data but you want row-by-row validation?
        └── Create a DMS validation-only task (TargetTablePrepMode: DO_NOTHING)
            └── Best for databases > 100 GB migrated via native replication
```

### Decision 11: Sequence Handling

```
Are you using DMS or native logical replication?
├── NO (pg_dump/pg_restore) → Sequences are included in the dump — no action needed
└── YES (DMS or native replication do NOT replicate sequences)
    ├── When to reset sequences?
    │   ├── Option A: After full load, before cutover (recommended)
    │   │   └── Query source for current values, setval() on target
    │   └── Option B: During cutover window (just before switching the app)
    │       └── More accurate but adds time to the cutover window
    └── How to reset?
        └── SELECT setval(pg_get_serial_sequence('<table>', '<column>'), (SELECT MAX(<column>) FROM <table>));
```

### Mobilize Decision Summary Template

| Decision | Choice | Notes |
| -------- | ------ | ----- |
| Schema migration method | Method A (DMS creates tables) / Method B (pre-create via EC2 bastion+SSM or CloudShell) / Method C (pg_dump/pg_restore) | |
| Schema migration scenario (if Method B) | A / B / C / D / E | |
| Replication path | Native / DMS / Hybrid / Offline only | |
| DMS plugin | test_decoding / pglogical / N/A | |
| Target table prep mode | DO_NOTHING (preloaded schema, empty target) / DROP_AND_CREATE (Method A) / N/A (see Decision 5) | |
| FK handling | session_replication_role / Manual drop-add / N/A | |
| Index timing | Two-stop (indexes at NotApplied, FKs at Applied) / Single-stop / No stop / N/A | |
| Trigger handling | Bypassed via session_replication_role (re-enable after cutover) / Remove / N/A | |
| Pilot scope | Full database pilot / Table-level pilot / Skip | |
| Validation strategy | DMS in-task / DMS validation-only / Offline / N/A | |
| Sequence reset timing | After full load / During cutover / N/A | |

## Target Environment Setup

### Cluster Provisioning Decisions

Before provisioning, decide:

| Decision | Options | Guidance |
| -------- | ------- | -------- |
| Engine | Aurora PostgreSQL vs RDS PostgreSQL | Aurora for HA, read replicas, cloning; RDS for simpler workloads |
| Capacity model | Provisioned vs Serverless v2 | Serverless v2 for variable workloads; provisioned for steady-state |
| Instance type | Graviton (db.r7g) vs Intel (db.r6i) | Graviton recommended (up to 20% price-performance improvement) |
| Engine version | Match or exceed source PG version | Check `SELECT version();` on source |
| Multi-AZ | Yes / No | Yes for production workloads |
| Storage encryption | Yes (default) | Always enable — cannot be changed after creation |

Use AWS Pricing MCP to compare costs across instance types and capacity models for the target region.

### Provision with the AWS CLI (no Aurora MCP needed)

This recipe creates the target with the AWS CLI alone. The Aurora PostgreSQL MCP server is optional and is not needed here. Run every command with `--profile <profile> --region <region>` (the AWS profile from Step 3 of the intake). It was run live once on a small test database; the AWS documentation for [Aurora Serverless v2 automatic pause](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2-auto-pause.html) and the [`create-db-cluster` reference](https://docs.aws.amazon.com/cli/latest/reference/rds/create-db-cluster.html) describe the flags.

**Inputs**, and where each comes from:

| Input | Source |
| ----- | ------ |
| `<region>` | the region the user chose in Step 3 |
| `<app>` (cluster id) | the cluster identifier proposed in Step 3 |
| `<database-name>` | the database the application uses on the source |
| `<subnet-group>` and `<vpc-id>` | Decision 0 Step 3, or an existing private subnet group the user confirmed |
| `<major.minor>` | the engine version rule below |
| `<project-tag>` | the project name, used as the `Project` tag on every resource |
| Capacity | the confirmed plan: ACU range and whether automatic pause is on |

**Consent before any create call.** Creating resources is a write. List every resource, the cost, the tags and the clean-up, then wait for an explicit yes:

```text
"I will create these resources in <region> on account <account-alias-or-id>:
- a security group <app>-aurora with no inbound rules
- an Aurora PostgreSQL cluster <app> (Serverless v2, encrypted, Data API on)
- a writer instance <app>-writer
- a secret for the administrator password, created and managed by RDS
Estimated monthly cost: $<monthly from the plan>, plus the Secrets Manager secret and Data API request charges.
Every resource is tagged Project=<project-tag>.
To remove them later you run: aws rds delete-db-instance, aws rds delete-db-cluster, aws ec2 delete-security-group.
Shall I go ahead?"
```

**Subnet group.** Reuse a private group and check it first (read-only):

```bash
aws rds describe-db-subnet-groups --db-subnet-group-name <subnet-group> --query 'DBSubnetGroups[0].[VpcId,Subnets[].SubnetAvailabilityZone.Name]'
```

Run `aws rds create-db-subnet-group` only after the user consented to the subnets and CIDRs in Decision 0 Step 5.

**Engine version.** Use the newest minor of the source major unless the user pinned one. This read-only query lists the minors and whether each supports a 0 ACU minimum:

```bash
aws rds describe-db-engine-versions --engine aurora-postgresql --query "DBEngineVersions[?starts_with(EngineVersion, '<major>.')].[EngineVersion,ServerlessV2FeaturesSupport.MinCapacity,ServerlessV2FeaturesSupport.MaxCapacity]" --output text
```

Run the query and read the result; do not rely on an earlier run. Automatic pause needs a minor that reports a MinCapacity of 0 (the AWS page lists the minimum engine versions).

**1. Security group with no inbound rules.** The Data API is reached over HTTPS, so the cluster needs none. Add port 5432 rules only for an application host in the VPC, native replication, or DMS.

```bash
aws ec2 create-security-group --group-name <app>-aurora --description "Aurora <app>: no inbound rules, Data API access only" --vpc-id <vpc-id> --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=<app>-aurora},{Key=Project,Value=<project-tag>}]'
```

**2. Cluster.** Serverless v2 with automatic pause, an RDS-managed administrator password, the Data API on at creation, and encryption:

```bash
aws rds create-db-cluster --db-cluster-identifier <app> --engine aurora-postgresql --engine-version <major.minor> \
  --serverless-v2-scaling-configuration MinCapacity=0,MaxCapacity=<acu-max>,SecondsUntilAutoPause=600 \
  --master-username postgres --manage-master-user-password --db-subnet-group-name <subnet-group> \
  --vpc-security-group-ids <sg-id> --storage-encrypted --backup-retention-period 7 --enable-http-endpoint \
  --tags Key=Project,Value=<project-tag>
```

**3. Writer instance.** The cluster is available within seconds. The writer took about 4 minutes on one run (creating, configuring-enhanced-monitoring, available):

```bash
aws rds create-db-instance --db-instance-identifier <app>-writer --db-cluster-identifier <app> --engine aurora-postgresql --db-instance-class db.serverless --no-publicly-accessible
aws rds wait db-instance-available --db-instance-identifier <app>-writer
```

**4. Verify.** The instance status is `available`, the Data API is on, the scaling configuration matches the plan, and the managed secret exists:

```bash
aws rds describe-db-instances --db-instance-identifier <app>-writer --query 'DBInstances[0].DBInstanceStatus'
aws rds describe-db-clusters --db-cluster-identifier <app> --query 'DBClusters[0].[Status,HttpEndpointEnabled,ServerlessV2ScalingConfiguration,MasterUserSecret.SecretArn]'
```

For a cluster that already exists, turn the Data API on with `aws rds enable-http-endpoint --resource-arn <cluster-arn>`. This is a write, so it needs consent too.

**Capacity defaults.** Use the confirmed plan. The ceiling is the plan's ACU maximum; propose 2 only when the plan has none. When the user has not chosen otherwise:

| Case | Scaling flag |
| ---- | ------------ |
| Non-production, automatic pause on | `MinCapacity=0,MaxCapacity=<acu-max>,SecondsUntilAutoPause=600` |
| Production, or automatic pause off | `MinCapacity=0.5,MaxCapacity=<acu-max>` |
| Provisioned model | No scaling flag; pass `--db-instance-class <class>` on the instance call instead of `db.serverless` |

The AWS page gives 300 seconds as the minimum and the default. The recipe uses 600 seconds so that the cluster does not pause between short test runs. With a minimum of 0, the first request after a pause gets a resume error and must retry (`troubleshooting.md` → "DatabaseResumingException / first request after idle is slow").

**Administrator password.** RDS creates and rotates the secret (`rds!cluster-...`) because of `--manage-master-user-password`. Never print or paste its value. Pass its ARN by reference, read with `aws rds describe-db-clusters --db-cluster-identifier <app> --query 'DBClusters[0].MasterUserSecret.SecretArn' --output text`. The hand-made secret with a secure password prompt stays as the alternative: see Step 1 of "AWS DMS Setup (Fallback Path)".

**Describe before retry.** A create call in a shell can return exit -1 with no output although it succeeded. After any create or enable call that gives no clear answer, run the matching `describe-*` command first, and call create again only if the resource is absent.

**Tags.** Every resource the agent creates carries `Project=<project-tag>`. RDS tags the managed secret itself, so the agent does not tag it.

**After provisioning.** Continue with "Check extensions on the cluster" below, then Post-Provisioning Configuration, then the schema steps.

### Check extensions on the cluster

Assess labels each extension from the AWS documentation. Once the cluster is available, and before any copy or schema step, ask the target which extensions it can install. Reading the catalog changes nothing. The cluster result overrides the documentation: if the two disagree, plan around the cluster.

Data API form (the rows come back in `formattedRecords`; retry a `DatabaseResumingException` while the cluster resumes):

```bash
aws rds-data execute-statement --resource-arn "$CLUSTER_ARN" --secret-arn "$SECRET_ARN" --database "$DB_NAME" --sql "SELECT name, default_version, installed_version FROM pg_available_extensions ORDER BY name" --format-records-as JSON
```

When the Data API is off, run the same query with `psql`. Hold the connection string in an environment variable and never echo it:

```bash
psql "$TARGET_DATABASE_URL" -c "SELECT name, default_version, installed_version FROM pg_available_extensions ORDER BY name;"
```

Compare the `name` column with every extension found at Assess:

| Assess label | On the cluster | What to do |
| ------------ | -------------- | ---------- |
| Compatible | Listed | Keep it |
| Compatible | Missing | Stop the copy for that extension. Relabel it unsupported, tell the user, and ask what to do: drop the usage, use a replacement, or choose another target |
| Unsupported or needs review | Listed | Report it as a finding. Ask before adding it |
| Unsupported or needs review | Missing | Confirm unsupported |

If the check cannot run (the cluster does not answer, or neither the Data API nor `psql` works), stop and ask. Do not fall back to the Assess labels.

### Post-Provisioning Configuration

After the cluster is available, configure the parameter group:

```sql
-- Recommended parameter group settings
-- rds.logical_replication = 1   (if Aurora will also act as a replication source later)
-- max_replication_slots = 10
-- max_wal_senders = 10
-- shared_preload_libraries = 'pg_stat_statements'
```

Install the extensions the schema needs, and only those that passed the cluster check (plus `pg_stat_statements` for tuning). The list below is an example, based on the Assess phase extension audit:

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
-- Add other extensions identified in the Assess phase
```

Extension availability is decided by the cluster check above (`pg_available_extensions`), not by a documentation search at this point.

### Security Group Configuration

The target security group must allow inbound TCP 5432 from:

- Application hosts (EC2, ECS, Lambda VPC)
- Source database IP range (for native logical replication)
- DMS replication instance security group (if using DMS; security-group-to-security-group rule, Step 3a of the DMS setup)

## Schema Migration

### Export Schema from Source

Export the schema from the source database, excluding platform-specific schemas:

```bash
pg_dump -h <source-host> -U <source-user> -d <source-database> \
  --schema-only \
  --no-owner --no-acl \
  -F p -f schema.sql
```

Add `-N <schema>` flags to exclude platform-specific schemas. Common exclusions by platform:

| Source Platform | Schemas to Exclude (`-N` flags) |
| --------------- | ------------------------------- |
| Supabase | `auth`, `storage`, `realtime`, `extensions`, `supabase_functions` |
| Heroku | None typically (Heroku uses `public` schema) |
| Neon | `neon` (if present) |
| GCP Cloud SQL | None typically |
| Azure Database | None typically |
| On-premises | Any custom internal/admin schemas |

Example with Supabase exclusions:

```bash
pg_dump -h <source-host> -U <source-user> -d <source-database> \
  --schema-only --no-owner --no-acl \
  -N auth -N storage -N realtime -N extensions -N supabase_functions \
  -F p -f schema.sql
```

### Review and Apply Schema

Review `schema.sql` before applying — remove any platform-specific references, functions, or triggers that won't work on the target.

**Executing DDL against a private Aurora cluster** — Aurora is in a private VPC, so you need a method to reach it:

| Method | When to Use | How |
| ------ | ----------- | --- |
| **Aurora Data API** (recommended for agent) | Default — agent runs DDL directly, no VPC needed. Works with private subnets. | `aws rds-data execute-statement` — use transactions for multi-statement. Enable with `aws rds enable-http-endpoint --resource-arn <arn>` |
| **EC2 bastion + SSM** (recommended for large schemas) | Scripts exceeding Data API's 45-second timeout or needing `psql -f` | Agent provisions t4g.nano, runs psql -f schema.sql via SSM, terminates after |
| **RDS Console integrated CloudShell** (recommended for manual) | When user prefers manual control | RDS Console → select cluster → Connect → integrated CloudShell with pre-configured connection |
| **CloudShell VPC environment** (manual alternative) | When RDS console CloudShell is unavailable | CloudShell → "+" → Create VPC environment in Aurora's VPC/subnet |

**Agent decision logic for Method B execution:**
1. For simple DDL (individual statements, < 45s each): use Data API directly (`aws rds-data execute-statement`). Use transactions for multi-statement sequences.
2. For large schema files or complex scripts: use EC2 bastion + SSM (full psql, no timeout limits).
3. If user prefers manual: direct them to RDS Console → select Aurora cluster → Connect button → integrated CloudShell.

**EC2 bastion approach (automated):**

```bash
# Agent provisions a t4g.nano in Aurora's VPC, uploads schema.sql via SSM, runs psql:
aws ssm send-command \
  --instance-ids <bastion-instance-id> \
  --document-name "AWS-RunShellScript" \
  --parameters commands='["export PGPASSWORD=$(aws secretsmanager get-secret-value --secret-id <rds-secret-id> --region <region> --query SecretString --output text | python3 -c \"import sys,json; print(json.load(sys.stdin)[\\\"password\\\"])\") && psql -h <target-endpoint> -U postgres -d <database-name> -f /tmp/schema.sql"]'
```

**CloudShell approach (manual):**

```bash
# In RDS Console integrated CloudShell or CloudShell VPC environment:
export PGPASSWORD=$(aws secretsmanager get-secret-value \
  --secret-id '<rds-secret-id>' --region <region> \
  --query SecretString --output text | python3 -c "import sys,json; print(json.load(sys.stdin)['password'])")

psql -h <target-endpoint> -U postgres -d <database-name> -f schema.sql
```


### Verify Schema Migration

```sql
-- On target: count objects and compare with source
SELECT 'tables' AS type, COUNT(*) FROM information_schema.tables WHERE table_schema = 'public'
UNION ALL
SELECT 'indexes', COUNT(*) FROM pg_indexes WHERE schemaname = 'public'
UNION ALL
SELECT 'functions', COUNT(*) FROM information_schema.routines WHERE routine_schema = 'public'
UNION ALL
SELECT 'triggers', COUNT(*) FROM information_schema.triggers WHERE trigger_schema = 'public'
UNION ALL
SELECT 'sequences', COUNT(*) FROM information_schema.sequences WHERE sequence_schema = 'public';
```

Run the same query on the source (excluding platform-specific schemas) and compare counts.

### Method A: Let DMS Create Tables (Recommended for Simple Schemas)

When using DMS with a simple schema (no CHECK constraints, no custom ENUM types, no partitions, no triggers), let DMS create the tables automatically during full load. This is the easiest and most reliable path — no separate schema migration step, no need to connect to private Aurora for DDL.

**How it works:**
1. Set `TargetTablePrepMode: DROP_AND_CREATE` in the DMS task settings
2. DMS creates tables with columns and primary keys during full load
3. No additional schema objects to manage — Method A is only for schemas with columns and PKs

**What DMS creates automatically:**
- Tables with correct column names and data types
- Primary key constraints

**What DMS does NOT create:**
- Secondary indexes, CHECK constraints, UNIQUE constraints, foreign keys, triggers, defaults, sequences
- If your schema needs ANY of these → use Method B instead (see Decision 1)

**When NOT to use Method A** (use Method B instead):
- Schema has ANY foreign keys, CHECK constraints, or UNIQUE constraints (DMS doesn't create these)
- Schema has ENUM types or custom domains (DMS maps ENUMs to VARCHAR)
- Schema has partitioned tables (must pre-create partition structure — see Scenario C)
- Schema has secondary indexes you want to preserve (DMS only creates PKs)
- You need column defaults (e.g., `DEFAULT gen_random_uuid()`) — DMS doesn't preserve these
- Schema has triggers (create only after cutover — see Decision 8 Reference)

### Schema Migration Scenarios (Method B — Pre-Create Schema)

Schema migration complexity varies. Use the appropriate approach based on your schema characteristics:

**Scenario A: Simple schema — no partitions, no custom types, no complex dependencies**

Single `pg_dump --schema-only` export and `psql -f` import. This is the default path.

```bash
pg_dump -h <source> -U postgres -d <db> --schema-only --no-owner --no-acl \
  -N auth -N storage -N realtime -N extensions -N supabase_functions \
  -F p -f schema.sql
psql -h <target> -U postgres -d <db> -f schema.sql
```

**Scenario B: Schema with ENUM types or custom domains**

ENUM types and custom domains must exist before tables that reference them. If `pg_dump` orders them correctly (it usually does), Scenario A works. If not:

```bash
# Step 1: Extract and create types first
pg_dump -h <source> -U postgres -d <db> --schema-only --no-owner --no-acl \
  -N auth -N storage -N realtime -N extensions -N supabase_functions \
  -F p -f schema.sql

# Step 2: Extract just the type definitions
grep -E "^CREATE TYPE" schema.sql > types.sql
psql -h <target> -U postgres -d <db> -f types.sql

# Step 3: Apply full schema (types already exist, will get "already exists" notices — safe to ignore)
psql -h <target> -U postgres -d <db> -f schema.sql
```

**Scenario C: Schema with partitioned tables**

**Preferred approach: Create partitioned table structure on target first, then let DMS + PostgreSQL partition routing handle data placement.**

DMS (version 2.4.3+) supports migrating to native PostgreSQL partitioned tables. When the partitioned table structure exists on the target, DMS inserts into the parent table and PostgreSQL's partition routing automatically places rows in the correct child partition. No special DMS configuration is needed.

Steps:
1. Export partition DDL from source (parent table + child partitions with bounds)
2. Create the partitioned structure on the target (parent + all partitions)
3. Create indexes on each partition (not the parent)
4. Run DMS with `TargetTablePrepMode: DO_NOTHING` — DMS inserts into the parent, PG routes to partitions

```sql
-- List partitioned tables and their partitions (run on source to discover structure)
SELECT parent.relname AS parent_table, child.relname AS partition_name
FROM pg_inherits
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON pg_inherits.inhrelid = child.oid
WHERE parent.relkind = 'p'
ORDER BY parent.relname, child.relname;

-- Example: Create partitioned table on target
CREATE TABLE partition_schema.my_table (
  id bigint NOT NULL,
  data text,
  created_at timestamptz NOT NULL
) PARTITION BY RANGE (created_at);

-- Create partitions with correct bounds
CREATE TABLE partition_schema.my_table_2024 PARTITION OF partition_schema.my_table
  FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
CREATE TABLE partition_schema.my_table_2025 PARTITION OF partition_schema.my_table
  FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

-- Create indexes on each partition
CREATE INDEX ON partition_schema.my_table_2024 (id);
CREATE INDEX ON partition_schema.my_table_2025 (id);
```

Reference: [Migrate to native partitioned tables in PostgreSQL 10 using AWS DMS](https://aws.amazon.com/blogs/database/migrate-to-native-partitioned-tables-in-postgresql-10-using-aws-database-migration-service/)

Alternative approaches (less preferred):
- **pg_dump per partitioned table**: If downtime is acceptable, use `pg_dump -t <table>` for each partitioned table instead of DMS.

**Scenario D: Schema with foreign keys and DMS full load**

**Preferred approach**: Include FKs in the pre-created schema and use `AfterConnectScript: "SET session_replication_role = 'replica';"` on the DMS target endpoint. This bypasses FK checks during both full load and CDC — no need to drop/recreate FKs. See Decision 1 (Foreign keys section) and Decision 6 (Reference) for details.

**Fallback** (only if `AfterConnectScript` is not applicable): Manually drop FKs before full load, re-add after at `StopTaskCachedChangesApplied`:

```bash
# Generate drop FK script
psql -h <target> -U postgres -d <db> -c "
  SELECT 'ALTER TABLE ' || conrelid::regclass || ' DROP CONSTRAINT ' || conname || ';'
  FROM pg_constraint WHERE contype = 'f';" -t -o drop_fks.sql

# Generate re-add FK script (save for after load)
psql -h <target> -U postgres -d <db> -c "
  SELECT 'ALTER TABLE ' || conrelid::regclass || ' ADD CONSTRAINT ' || conname || ' ' || pg_get_constraintdef(oid) || ';'
  FROM pg_constraint WHERE contype = 'f';" -t -o add_fks.sql

# Drop FKs → run DMS full load → stop at StopTaskCachedChangesApplied → re-add FKs
psql -h <target> -U postgres -d <db> -f drop_fks.sql
# ... DMS full load runs ...
psql -h <target> -U postgres -d <db> -f add_fks.sql
```

**Scenario E: Large schema with many dependencies — use pg_restore TOC reordering**

For complex schemas where object creation order matters:

```bash
# Step 1: Export in custom format
pg_dump -h <source> -U postgres -d <db> --schema-only --no-owner --no-acl \
  -N auth -N storage -N realtime -N extensions -N supabase_functions \
  -F c -f schema.backup

# Step 2: Generate table of contents (TOC)
pg_restore --list schema.backup > toc.list

# Step 3: Edit toc.list to reorder objects (move types before tables, tables before FKs, etc.)
# Lines starting with ; are comments. Reorder or remove lines as needed.

# Step 4: Restore using the reordered TOC
pg_restore --no-owner --no-acl --use-list=toc.list -h <target> -U postgres -d <db> schema.backup
```

This gives full control over object creation order and is the most flexible approach for complex schemas.

### Migration Branch for Application Refactoring

Before starting application code refactoring (e.g., replacing Supabase SDK with direct PostgreSQL), create a dedicated migration branch:

```bash
git checkout -b migration/aurora-refactor
```

This keeps migration changes isolated from ongoing feature development and allows:
- Easy rollback if the migration is delayed
- Parallel development on the main branch
- Clean PR/CR review of all migration-related code changes
- Testing the refactored app against Aurora without affecting the main branch

**The actual refactoring work is done during the Migrate phase** — see `03-migrate-phase.md` → "Application Code Refactoring & Testing" for the full workflow. During Mobilize, only create the branch and optionally plan the refactoring approach by reading `supabase-migration-feature-mapping.md` → "Application Code Refactoring".

## ENUM Types (DMS Requirement)

If using DMS, ENUM types must exist on the target before migration. DMS does not create or migrate ENUM types.

```sql
-- On source: list all ENUM types and their values
SELECT t.typname, e.enumlabel
FROM pg_type t
JOIN pg_enum e ON t.oid = e.enumtypid
ORDER BY t.typname, e.enumsortorder;
```

Recreate each ENUM on the target manually before starting DMS tasks.

## Source Database Configuration

### Enable Logical Replication on Source

The source database must have `wal_level = logical` and sufficient replication slots/senders. Configuration method varies by platform:

| Source Platform | How to Enable Logical Replication |
| --------------- | --------------------------------- |
| Supabase | Supabase CLI: `supabase --experimental --project-ref <ref> postgres-config update --config max_replication_slots=10` (see platform-specific steering for full commands) |
| Heroku | `heroku pg:psql` — check plan supports logical replication (Standard/Premium); contact Heroku support if `max_wal_senders` needs increase |
| Neon | Neon Dashboard → Project Settings → enable logical replication (Pro/Scale plans) |
| GCP Cloud SQL | Set flag `cloudsql.logical_decoding = on` via Console or `gcloud sql instances patch` |
| Azure Database | Set server parameter `wal_level = logical` via Azure Portal or CLI; requires restart |
| On-premises | Edit `postgresql.conf`: `wal_level = logical`, `max_replication_slots = 10`, `max_wal_senders = 10`; restart PostgreSQL |

### Replication Parameters

Ensure these parameters are set on the source:

```
max_replication_slots = 10    (or higher, based on number of subscriptions/DMS tasks)
max_wal_senders = 10          (or higher)
wal_sender_timeout = 0        (prevents timeout during large initial copy)
max_slot_wal_keep_size = 4GB  (prevents unbounded WAL growth)
max_wal_size = 2GB            (allows larger WAL segments for replication)
```

### Verify Source Configuration

```sql
SELECT name, setting, context, pending_restart
FROM pg_settings
WHERE name IN ('wal_level', 'max_replication_slots', 'max_wal_senders',
               'wal_sender_timeout', 'max_slot_wal_keep_size', 'max_wal_size');
```

`wal_level` must show `logical`. If `pending_restart` is `on` for any parameter, the source database needs a restart before replication will work.

## Native Logical Replication Setup

Use this path when all target tables have primary keys and native replication is the preferred approach.

### Step 1: Create Publication on Source

```sql
-- Create publication for all tables in public schema
CREATE PUBLICATION source_to_target FOR ALL TABLES IN SCHEMA public;

-- Or create for specific tables
CREATE PUBLICATION source_to_target FOR TABLE public.table1, public.table2, public.table3;
```

### Step 2: Create Replication Slot on Source

```sql
SELECT pg_create_logical_replication_slot('migration_slot', 'pgoutput');
```

### Step 3: Verify Publication and Slot

```sql
-- Verify publication
SELECT pubname, puballtables, pubinsert, pubupdate, pubdelete FROM pg_publication;

-- Verify tables in publication
SELECT schemaname, tablename FROM pg_publication_tables WHERE pubname = 'source_to_target';

-- Verify replication slot
SELECT slot_name, plugin, slot_type, active FROM pg_replication_slots;
```

### Step 4: Create Subscription on Target

```sql
CREATE SUBSCRIPTION migration_sub
CONNECTION 'host=<source-host> user=<source-user> password=<source-password> dbname=<source-database>'
PUBLICATION source_to_target
WITH (copy_data = true, create_slot = false, slot_name = 'migration_slot');
```

Key parameters:
- `copy_data = true` — copies existing data before streaming changes
- `create_slot = false` — uses the slot created in Step 2
- `slot_name` — references the slot from Step 2

### Step 5: Verify Replication is Active

```sql
-- On source: check active replication connections
SELECT pid, usename, application_name, client_addr, state,
  sent_lsn, write_lsn, flush_lsn, replay_lsn
FROM pg_stat_replication;

-- On target: check subscription status
SELECT subname, subenabled, subslotname
FROM pg_stat_subscription;
```

If replication is active, the source will show a row in `pg_stat_replication` with `state = 'streaming'` and the target will show the subscription as enabled.

### Platform-Specific Notes for Native Replication

- **Supabase**: Steps 1–2 require write access — the agent provides SQL commands for the user to run in the Supabase SQL Editor or psql. The Supabase MCP server (read-only) cannot create publications or slots. See the Supabase-specific steering file for details.
- **Heroku**: Use `heroku pg:psql` to run Steps 1–2. Verify the plan supports logical replication.
- **Managed platforms (GCP, Azure, Neon)**: Ensure logical replication is enabled at the platform level before running Steps 1–2.
- **On-premises**: Full control — run Steps 1–2 directly via psql.

## AWS DMS Setup (Fallback Path)

Use DMS for tables where native replication doesn't work (no primary key, unsupported types), when the source is IPv6-only, or when built-in validation is needed.

This section holds the only DMS create commands in the power. Other files state the rules and point here.

> **Before you run any command**
>
> Record these four Mobilize decisions in the migration notes (the Mobilize decision summary table):
>
> - Decision 1, schema method (Method A, B or C)
> - Decision 4, plugin (`test_decoding` or `pglogical`)
> - Decision 5, target table prep mode
> - Decision 6, foreign key bypass (for example `AfterConnectScript` on the target endpoint)
>
> If any of them is not recorded, stop and apply the Mobilize decisions first (Decisions 1, 4, 5 and 6 in this file).
>
> The four rules: all four values are recorded with a reason for each; a preloaded schema (Method B) needs `DO_NOTHING`; a source with foreign keys needs a bypass; Method A cannot create foreign keys and needs `DROP_AND_CREATE`.

### What you need before the first command

- The recorded Mobilize decisions: schema method (Decision 1), plugin (Decision 4), target table prep mode (Decision 5) and foreign key bypass (Decision 6). Steps 4 and 5 use the plugin and the foreign key bypass. Decision 5 sets the task template in `03-migrate-phase.md` Path 3.
- Two secrets in Secrets Manager, one for the source and one for the target (Step 1).
- An IAM role that lets DMS read those two secrets (Step 1b).
- A replication subnet group of dual-stack public subnets (Step 2).
- A security group for the replication instance, and an inbound rule on the Aurora security group (Step 3a).
- The replication instance (Step 3b), then the source and target endpoints (Steps 4 and 5), then connection tests (Step 6).

Ask for the user's consent before each create call (see `agent-behavioral-rules.md`). Print the matching delete command at the same time. In every command below, `<region>` and `<aws-region>` mean the same Region.

### Step 1: Create Secrets in AWS Secrets Manager

**IMPORTANT**: Do NOT pass passwords as plaintext in CLI commands — they will be visible in shell history. Use the secure input pattern below (works in both bash and zsh):

```bash
# Source credentials (password entered interactively, hidden from terminal)
echo -n "Enter source DB password: " && read -s SOURCE_PASS && echo && \
aws secretsmanager create-secret \
  --name dms-source-<project-name> \
  --secret-string "{\"username\":\"<source-user>\",\"password\":\"$SOURCE_PASS\",\"host\":\"<source-host>\",\"port\":5432,\"dbname\":\"<source-database>\"}" \
  --region <aws-region> && \
unset SOURCE_PASS

# Target credentials (password entered interactively, hidden from terminal)
echo -n "Enter target DB password: " && read -s TARGET_PASS && echo && \
aws secretsmanager create-secret \
  --name dms-target-<project-name> \
  --secret-string "{\"username\":\"<target-user>\",\"password\":\"$TARGET_PASS\",\"host\":\"<target-endpoint>\",\"port\":5432,\"dbname\":\"<database-name>\"}" \
  --region <aws-region> && \
unset TARGET_PASS
```

Alternative: Create secrets via the AWS Console (Secrets Manager → Store a new secret → Other type of secret) to avoid CLI entirely.

What the secret must hold. Each secret is a JSON object with `username`, `password`, `host` and `port`. The AWS documentation calls these the minimum members and says DMS ignores any other member ([Using secrets to access DMS endpoints](https://docs.aws.amazon.com/dms/latest/userguide/security_iam_secretsmanager.html)). The `dbname` member above is a note for people reading the secret. DMS does not use it. The same page says DMS does not allow these characters in endpoint passwords: `;.:+{}*&,%\`. Leave them out of the password.

What follows for the endpoint commands:

- With `SecretsManagerSecretId` and `SecretsManagerAccessRoleArn` set, DMS rejects `--server-name` and `--port`. The call fails with `InvalidParameterCombinationException`: give either username, password, server name and port, or the secret and its role. Host and port therefore live inside the secret.
- `--database-name` is still required. Without it the call fails with `InvalidParameterValueException` (the parameter must be provided and must not be blank).

**Aurora's cluster-managed secret cannot be the target secret.** The secret that Aurora manages for the cluster (`MasterUserSecret`, named `rds!cluster-...`) holds only a username and a password. The AWS documentation says it lacks the host and port that DMS needs, and recommends a new user and secret instead. It also rotates, so a DMS-format secret that copies its password stops matching after the next rotation. Pick one:

- Preferred: a dedicated migration user on the target, with its own secret. The user or the Data API runs the SQL, and the password comes from the secret, never from chat.
- Otherwise: copy the admin credentials into the DMS-format secret, tell the user that rotation will make it stale, and after any rotation update the secret (`aws secretsmanager put-secret-value`) and run Step 6 again.

```sql
CREATE ROLE <migration-user> LOGIN PASSWORD '<password-from-secret>';
GRANT USAGE, CREATE ON SCHEMA public TO <migration-user>;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO <migration-user>;
-- PostgreSQL 15 or later: lets the endpoint script set session_replication_role (Decision 6)
GRANT SET ON PARAMETER session_replication_role TO <migration-user>;
```

This grant list has not been run end to end. The tested run used the cluster admin user. After creating the role, run Step 6 and a small load before relying on it. The `GRANT SET ON PARAMETER` form comes from [How do I turn off foreign keys or triggers in RDS for PostgreSQL or Aurora PostgreSQL](https://repost.aws/knowledge-center/rds-postgresql-foreign-keys). That page says PostgreSQL 14 and earlier need `rds_superuser` instead. `CREATE` on the schema lets DMS create its control tables.

### Step 1b: Create the IAM Role That Lets DMS Read the Secrets

DMS reads the secrets through a role that you pass as `SecretsManagerAccessRoleArn`. Create it before the endpoints. The trust policy names the regional DMS service principal. The AWS documentation names only the regional principal ([Using secrets to access DMS endpoints](https://docs.aws.amazon.com/dms/latest/userguide/security_iam_secretsmanager.html)). The tested run also listed the global principal `dms.amazonaws.com`, so the recipe keeps both.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": ["dms.<region>.amazonaws.com", "dms.amazonaws.com"]},
    "Action": "sts:AssumeRole"
  }]
}
```

Save it as `dms-secrets-trust.json`. The permission policy grants read access on exactly the two secrets from Step 1 and nothing else:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["secretsmanager:GetSecretValue", "secretsmanager:DescribeSecret"],
    "Resource": ["<source-secret-arn>", "<target-secret-arn>"]
  }]
}
```

Save it as `dms-secrets-policy.json`, then:

```bash
aws iam create-role \
  --role-name dms-secrets-<project-name> \
  --assume-role-policy-document file://dms-secrets-trust.json

aws iam put-role-policy \
  --role-name dms-secrets-<project-name> \
  --policy-name read-dms-secrets \
  --policy-document file://dms-secrets-policy.json

# Delete commands (print them at the consent step)
# aws iam delete-role-policy --role-name dms-secrets-<project-name> --policy-name read-dms-secrets
# aws iam delete-role --role-name dms-secrets-<project-name>
```

Notes from the AWS documentation page above:

- `GetSecretValue` is the permission DMS needs. The page lists `DescribeSecret` for a secret in another account. The recipe grants it on the same two secrets either way.
- If a customer-managed KMS key encrypts a secret, add `kms:Decrypt` and `kms:DescribeKey` on that key. The default Secrets Manager key needs no KMS statement.
- The IAM principal that runs `create-endpoint` and `modify-endpoint` needs `iam:GetRole` and `iam:PassRole` on this role, and `secretsmanager:DescribeSecret` on the secrets. DMS checks both.
- A replication instance in a private subnet needs a Secrets Manager VPC endpoint and the extra connection attribute `secretsManagerEndpointOverride`. This recipe uses a publicly accessible instance in public subnets, which was not observed to need it.

### Step 2: Create DMS Subnet Group

```bash
aws dms create-replication-subnet-group \
  --replication-subnet-group-identifier dms-subnet-group-<project-name> \
  --replication-subnet-group-description "PostgreSQL migration" \
  --subnet-ids <public-subnet-id-1> <public-subnet-id-2> \
  --region <aws-region>
```

Use at least two subnets in two Availability Zones. For a dual-stack instance, each subnet in the group needs an IPv6 CIDR block ([IP addressing and network types](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_ReplicationInstance.IPAddressing.html)). The same page says DMS disables internet gateway access for the IPv6 endpoints of a private dual-stack instance. So this group uses public subnets with a `::/0` route to the internet gateway. That differs from the Aurora subnet group, which stays private. Check both facts first (read-only):

```bash
aws ec2 describe-subnets --subnet-ids <public-subnet-id-1> <public-subnet-id-2> \
  --query "Subnets[*].[SubnetId,AvailabilityZone,Ipv6CidrBlockAssociationSet[0].Ipv6CidrBlock]" --output text
aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=<public-subnet-id-1>" \
  --query "RouteTables[*].Routes[?DestinationIpv6CidrBlock=='::/0'].GatewayId" --output text
```

The decision tree is in "IPv6 and Dual-Stack Connectivity" in `supabase-prerequisites-and-connectivity.md`.

### Step 3a: Create the Security Group for the Replication Instance

The replication instance needs its own security group with no inbound rules. Aurora then accepts connections from that group on port 5432.

```bash
aws ec2 create-security-group \
  --group-name dms-<project-name> \
  --description "DMS replication instance" \
  --vpc-id <vpc-id> \
  --region <aws-region>

aws ec2 authorize-security-group-ingress \
  --group-id <aurora-sg-id> \
  --protocol tcp --port 5432 \
  --source-group <dms-sg-id> \
  --region <aws-region>

# Delete commands (print them at the consent step)
# aws ec2 revoke-security-group-ingress --group-id <aurora-sg-id> --protocol tcp --port 5432 --source-group <dms-sg-id>
# aws ec2 delete-security-group --group-id <dms-sg-id>
```

The AWS documentation says a new security group starts with no inbound rules and with an outbound rule that allows all outbound traffic ([Security group rules](https://docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html)). The instance needs outbound access to the source on port 5432, to Aurora on port 5432 and to Secrets Manager on port 443. Confirm the outbound rules cover the source's address family before creating the instance (read-only). An IPv6-only source needs an IPv6 rule:

```bash
aws ec2 describe-security-groups --group-ids <dms-sg-id> \
  --query "SecurityGroups[0].IpPermissionsEgress" --region <aws-region>
```

In the tested run the instance reached Aurora over the VPC in dual-stack mode, and the source over IPv6.

### Step 3b: Create DMS Replication Instance

Check which classes and engine versions the Region can order (read-only):

```bash
aws dms describe-orderable-replication-instances --region <aws-region> \
  --query "OrderableReplicationInstances[].[ReplicationInstanceClass,EngineVersion]" --output text
```

Then create the instance. This is the only replication instance command in the power:

```bash
aws dms create-replication-instance \
  --replication-instance-identifier dms-instance-<project-name> \
  --replication-instance-class <instance-class> \
  --engine-version <engine-version> \
  --allocated-storage <gb> \
  --no-multi-az \
  --publicly-accessible \
  --network-type DUAL \
  --vpc-security-group-ids <dms-sg-id> \
  --replication-subnet-group-identifier dms-subnet-group-<project-name> \
  --region <aws-region>
```

- `--publicly-accessible` gives the instance a public address so it can reach an external source. It cannot be changed after creation. Without it, delete and recreate the instance.
- `--network-type DUAL` enables IPv4 and IPv6. Write the value in uppercase. `modify-replication-instance --network-type DUAL` changes it later (see "IPv6 and Dual-Stack Connectivity" in `supabase-prerequisites-and-connectivity.md`).
- `--engine-version`: choose a version from the check above and record it in the migration notes. The API default is the latest available version, which can change between runs. The steps were exercised on engine version 3.5.4 (observed on one run), so use that when it is listed and check the current version before you rely on it. Other versions have not been exercised. The validation failures table differs by version: DMS creates `awsdms_validation_failures_v2` on a PostgreSQL target from version 3.6.1 on, and `_v1` before that ([AWS DMS data validation](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Validating.html)).
- `--allocated-storage`: 20 GB is a sample for a small database. Raise it for a heavy full load, because changes that do not fit in memory and the task logs use this disk ([Selecting the best size for a replication instance](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_BestPractices.SizingReplicationInstance.html)).

Sizing. The AWS documentation lists the class memory in [Choosing the right replication instance](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_ReplicationInstance.Types.html). Start from the smallest class that fits and watch CPU, free memory, free storage and IOPS.

| Situation | Class to start with | Why |
| --------- | ------------------- | --- |
| Small database, test run, light CDC | `dms.t3.small` (the sample, 2 vCPU, 2 GiB) | Burstable T3 class, lowest cost |
| Larger tables, several tasks in parallel | `dms.t3.medium` or `dms.t3.large` (4 or 8 GiB) | More memory for the caches. Parallel tasks and threads use CPU |
| Heavy CDC, many LOB columns, large databases | A memory-optimized R class, for example `dms.r6i.large` (16 GiB) | The sizing page says LOBs are processed in memory and a high change rate grows the caches |
| Compute-bound work | A C class, for example `dms.c6i.large` (4 GiB) | More CPU for the memory |

In one run, `CreateReplicationInstance` rejected the micro class with "Invalid ReplicationInstance class", and `dms.t3.small` was the smallest class that Region could order. The AWS class table lists a smaller class, so run the check above and do not assume. A `dms.t3.small` instance costs about $6/week (an estimate, compute only; check the live price for your Region).

Poll until the instance is ready before creating endpoints. Wait at intervals (see the table under Step 6), not in a tight loop:

```bash
aws dms describe-replication-instances \
  --filters Name=replication-instance-id,Values=dms-instance-<project-name> \
  --query "ReplicationInstances[0].ReplicationInstanceStatus" --output text --region <aws-region>
```

The status must be `available`.

### Step 4: Create Source Endpoint

```bash
aws dms create-endpoint \
  --endpoint-identifier dms-source-<project-name> \
  --endpoint-type source \
  --engine-name postgres \
  --database-name <source-database> \
  --ssl-mode require \
  --postgre-sql-settings file://source-settings.json \
  --extra-connection-attributes "pluginName=<plugin>;heartbeatEnable=true;heartbeatFrequency=5;mapBooleanAsBoolean=true" \
  --region <aws-region>
```

`<plugin>` is the value chosen in Decision 4 (`test_decoding` or `pglogical`). The command carries no `--server-name`, `--port`, `--username` or `--password`. `source-settings.json` holds only ARNs, never a password:

```json
{
  "SecretsManagerSecretId": "<source-secret-arn>",
  "SecretsManagerAccessRoleArn": "<dms-secrets-role-arn>"
}
```

After any `aws dms modify-endpoint`, run Step 6 again for that endpoint (see Step 6).

AWS DMS supports two logical decoding plugins: `test_decoding` and `pglogical`. If the `pglogical` extension is installed on the source database, DMS uses it by default. Otherwise, DMS uses `test_decoding`. You can override this behavior with the `pluginName` extra connection attribute.

### Choosing Between test_decoding and pglogical

| Plugin | Advantages | Best For |
| ------ | ---------- | -------- |
| `test_decoding` | Built into PostgreSQL, no extension install needed. Captures entire row for UPDATE/DELETE (useful for tables without PKs). | Platforms where pglogical is unavailable (Supabase, Neon). Migrations where all tables are part of CDC. Tables without primary keys. |
| `pglogical` | Filters non-CDC tables at the replication slot level — reduces WAL decoding overhead, network throughput, and DMS latency. | Migrations where only a subset of tables are part of CDC. Large databases with many non-migrated tables. Self-managed PostgreSQL, RDS PostgreSQL 9.6+, Aurora PostgreSQL (check extension availability for your version). |

Reference: [Comparison of test_decoding and pglogical plugins in Amazon Aurora PostgreSQL for data migration using AWS DMS](https://aws.amazon.com/blogs/database/comparison-of-test_decoding-and-pglogical-plugins-in-amazon-aurora-postgresql-for-data-migration-using-aws-dms/)

### Plugin Availability by Source Platform

| Source Platform | `test_decoding` | `pglogical` | Recommended Plugin |
| --------------- | --------------- | ----------- | ------------------ |
| Supabase | Yes (built-in) | No (extension not available) | `test_decoding` |
| Neon | Yes (built-in) | Check plan/version | `test_decoding` (safe default) |
| RDS PostgreSQL | Yes (built-in) | Yes (9.6+) | `pglogical` if not all tables are in CDC; `test_decoding` otherwise |
| Aurora PostgreSQL | Yes (built-in) | Yes (check extension availability for your version) | `pglogical` if not all tables are in CDC; `test_decoding` otherwise |
| On-premises | Yes (built-in) | Yes (install extension) | `pglogical` if not all tables are in CDC; `test_decoding` otherwise |

### Extra Connection Attributes

Set the `extra-connection-attributes` based on the chosen plugin (Decision 4):

```
pluginName=<plugin>;heartbeatEnable=true;heartbeatFrequency=5;mapBooleanAsBoolean=true
```

`<plugin>` is `test_decoding` or `pglogical`.

Key attributes:

- `pluginName` — `test_decoding` or `pglogical` (see tables above for platform availability and recommendation)
- `heartbeatEnable=true` — prevents replication slot WAL growth during idle periods
- `heartbeatFrequency=5` — heartbeat every 5 minutes
- `mapBooleanAsBoolean=true` — correct boolean type mapping (DMS defaults to `varchar(5)` without this). **Must be set on both source and target endpoints.**

**Plugin spelling.** The plugin can be set in two places. Use one, not both.

| Where | Spelling | Notes |
| ----- | -------- | ----- |
| Extra connection attribute string | `pluginName=test_decoding` (underscore) | Valid on an endpoint that uses a secret. The DMS task log showed the slot using `test_decoding` |
| `PluginName` member of `PostgreSQLSettings` | `test-decoding` (hyphen) or `pglogical` | The API model lists `no-preference`, `test-decoding` and `pglogical`. The user guide table of PostgreSQL source settings lists `pglogical` and `test_decoding` |

The steps in this power use the attribute string. If you move the plugin into `source-settings.json`, remove it from the attribute string and use the spelling the API accepts.

### Step 4b: What DMS Writes to the Source

The source is read-only for the agent, but the DMS task is not. When the task starts, DMS writes to the source database through the endpoint user. Tell the user at the consent step, before the task exists:

- **Heartbeat table.** `heartbeatEnable=true` makes DMS create and update a heartbeat table in the schema set by `HeartbeatSchema` (default `public`). In the tested run it was `public.awsdms_heartbeat`. Its purpose is to keep the slot's `restart_lsn` moving ([PostgreSQL source endpoint settings](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Source.PostgreSQL.html)).
- **DDL audit objects.** With `CaptureDdls` at its default of true, DMS creates DDL-capture artifacts when the task starts, in the schema set by `DdlArtifactsSchema` (default `public`). The same page names an audit table `awsdms_ddl_audit`, an event trigger `awsdms_intercept_ddl` and a function. In the tested run `public.awsdms_ddl_audit` appeared on the source.
- **Privilege.** The endpoint user must be able to create these objects in the schema, and to use logical replication. For a self-managed source AWS lists a superuser for CDC. For an AWS-managed source it lists the `rds_superuser` and `rds_replication` roles. For a Supabase source the power uses the `postgres` role (`supabase-migration-strategy-reference.md`).
- **Replication slot.** A full load plus CDC task holds a logical replication slot on the source while it exists. A slot that nothing reads keeps WAL on the source. The heartbeat runs only while a task is running, so a stopped task with its slot still in place can fill the source disk ([Why do I see high storage consumption on my source PostgreSQL database when I run an AWS DMS CDC task?](https://repost.aws/knowledge-center/dms-high-storage-consumption-postgresql)).

The agent does not create, drop or edit any of these. DMS writes them, and the user drops them after cutover. The cleanup steps are in `03-migrate-phase.md` → "DMS Cleanup on the Source".

### Step 5: Create Target Endpoint

```bash
aws dms create-endpoint \
  --endpoint-identifier dms-target-<project-name> \
  --endpoint-type target \
  --engine-name aurora-postgresql \
  --database-name <database-name> \
  --ssl-mode require \
  --postgre-sql-settings file://target-settings.json \
  --extra-connection-attributes "mapBooleanAsBoolean=true" \
  --region <aws-region>
```

`target-settings.json` holds the secret, the role and the foreign key bypass from Decision 6:

```json
{
  "SecretsManagerSecretId": "<target-secret-arn>",
  "SecretsManagerAccessRoleArn": "<dms-secrets-role-arn>",
  "AfterConnectScript": "SET session_replication_role = 'replica';"
}
```

Keep `AfterConnectScript` unless Decision 6 chose another foreign key bypass. Omit it only in that case. The script contains single quotes, and the AWS CLI shorthand syntax rejects it (`Error parsing parameter '--postgre-sql-settings': Expected: ',', received: '''`). Use the JSON file with `file://`.

For RDS PostgreSQL targets, use `--engine-name postgres` instead of `aurora-postgresql`.

> **Important**: `mapBooleanAsBoolean=true` must be set on **both** the source and target endpoints for it to take effect. Without it on both sides, DMS maps PostgreSQL `boolean` to `varchar(5)`.

After any `aws dms modify-endpoint`, run Step 6 again for that endpoint (see Step 6).

### Step 6: Test Endpoint Connections

```bash
aws dms test-connection \
  --replication-instance-arn <replication-instance-arn> \
  --endpoint-arn <source-endpoint-arn> \
  --region <aws-region>

aws dms test-connection \
  --replication-instance-arn <replication-instance-arn> \
  --endpoint-arn <target-endpoint-arn> \
  --region <aws-region>
```

Poll `describe-connections` until both show `Status: successful`:

```bash
aws dms describe-connections \
  --filters Name=endpoint-arn,Values=<endpoint-arn> \
  --query "Connections[0].[Status,LastFailureMessage]" --output text --region <aws-region>
```

If a connection fails, check security groups, network ACLs, and credentials. On an endpoint that uses a secret, suspect the secrets role first: its trust policy (Step 1b), the two secret ARNs in its permission policy, and whether the secret's password still matches the database user (rotation).

**Editing an endpoint invalidates its connection test.** After `aws dms modify-endpoint`, `start-replication-task` fails with `InvalidResourceStateFault`: the connection test for the instance and the endpoint must be successful before the task starts. Run `test-connection` again for every modified endpoint and wait for `successful` before you start or reload a task.

### What to expect (observed on one run)

These times were observed on one run with `dms.t3.small`. They are not guarantees. Poll with `describe-*` calls at intervals of 30 seconds or more, never in a tight loop, and do not conclude that something is stuck before the times below have passed.

| Step | Observed |
| ---- | -------- |
| Replication instance creation (from the create call to status `available`) | About 10 minutes |
| `stop-replication-task` to status `stopped` | 1 to 2 minutes |
| `modify-replication-task` | Needs a stopped task. The status passes through `modifying` |
| Validation with `ValidationQueryCdcDelaySeconds` at 180 | The first validation states are `Pending validation` and `Pending records`. A mismatch showed up about 7 minutes after the load finished |

The reload sequence (stop, truncate, reload, wait for table states) is in `troubleshooting.md` → "Reload after a failed or partial run".

## DMS Validation Setup

For databases > 100 GB, turn DMS validation on by default in the replication task settings. The example task in `03-migrate-phase.md` Path 3 enables it for any size, which matches the tested run:

```json
{
  "ValidationSettings": {
    "EnableValidation": true,
    "ThreadCount": 5,
    "PartitionSize": 10000,
    "FailureMaxCount": 10000,
    "TableFailureMaxCount": 1000,
    "ValidationQueryCdcDelaySeconds": 0
  }
}
```

> Note: `ValidationQueryCdcDelaySeconds` default is 0, and 0 is the value used in the tested run ([Data validation task settings](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Tasks.CustomizingTasks.TaskSettings.DataValidation.html)). The setting delays the first validation query on both sides for each CDC update, which can reduce contention when migration latency is high. Raise it (for example to 180 seconds) when the source is busy and CDC changes need time to apply before validation compares rows. A larger delay also postpones the first validation result (see "What to expect" above).

DMS writes validation failures to a control table on the target (`awsdms_validation_failures_v1`, or `_v2` from engine version 3.6.1). When a table shows `Mismatched records`, read that table yourself first. The schema depends on the task setting `ControlTablesSettings.ControlSchema`, so use the lookup and queries in [Path 3 Step 4 in `03-migrate-phase.md`](03-migrate-phase.md#step-4-monitor-dms-validation) and the decoding steps in [DMS validation: Mismatched records](troubleshooting.md#dms-validation-mismatched-records).

CloudWatch metrics to monitor:
- `ValidationSucceededRecordCount`
- `ValidationFailedOverallCount`
- `ValidationAttemptedRecordCount`
- `ValidationSuspendedOverallCount`
- `ValidationPendingOverallCount`

## Pilot Migration

### When to Run Pilots

**Recommended when:**
- Migrating 10+ databases (validate process before scale)
- Team lacks experience with native replication or AWS DMS
- Complex schemas, large data volumes, or special data types
- Need to establish migration duration baselines
- Testing hybrid approach (native replication + DMS)

**Skip pilots when:**
- Migrating a single database with straightforward schema
- Team has proven PostgreSQL replication / DMS experience
- Small database (< 10 GB) with low risk
- Time constraints require immediate migration

### Database Complexity Classification

Use this matrix to assess migration complexity and select appropriate pilot candidates:

| Factor | Low | Medium | High |
| ------ | --- | ------ | ---- |
| Size | < 50 GB | 50–500 GB | > 500 GB |
| Tables | < 50 | 50–200 | > 200 |
| Extensions | Standard only | Few custom | Many custom / unsupported |
| Connections | < 50 concurrent | 50–200 | > 200 |
| Dependencies | Single app | 2–3 apps | 4+ apps or cross-team |
| Downtime tolerance | Hours | < 1 hour | Near-zero |

### Pilot Table Selection

Run a pilot migration with a small subset of tables to validate the approach before full execution.

Choose 3–5 tables that represent:
- A large table (to test performance and throughput)
- A table with foreign keys (to test dependency handling)
- A table with special data types (JSONB, arrays, ENUM)
- A table with RLS policies (if applicable)
- A table without a primary key (to test the offline/DMS fallback path)

### Pilot Execution Steps

1. Run the chosen migration path (offline, native replication, or DMS) on pilot tables
2. Validate row counts match between source and target
3. Validate data integrity with spot checks on key columns
4. Measure migration throughput (rows/second, MB/second)
5. Estimate full migration duration based on pilot results
6. Document any issues encountered and resolutions

### Pilot Validation Queries

```sql
-- Row count comparison (run on both source and target)
SELECT schemaname, relname, n_live_tup
FROM pg_stat_user_tables
WHERE relname IN ('pilot_table_1', 'pilot_table_2', 'pilot_table_3')
ORDER BY relname;

-- Sample data spot check
SELECT * FROM pilot_table_1 ORDER BY id LIMIT 10;
SELECT * FROM pilot_table_1 ORDER BY id DESC LIMIT 10;
```


## Mobilize Phase Complete — Checklist

Before proceeding to the Migrate phase, confirm:

**Infrastructure (provisioned and tested):**
- [ ] Network assessment done (Decision 0) — connectivity path selected and tested
- [ ] Aurora cluster provisioned in private subnets with correct parameter group
- [ ] DMS infrastructure ready (if using DMS): replication instance, source endpoint, target endpoint — all connections tested
- [ ] Native replication configured and active (if using native path): publication, slot, subscription running

**Schema (migrated and verified):**
- [ ] Schema migrated to target and verified (object counts match)
- [ ] Extensions created on target
- [ ] ENUM types created on target (if applicable)

**Application (branch created, ready for refactoring in Migrate phase):**
- [ ] Migration branch created (`migration/aurora-refactor`)
- [ ] Refactoring approach planned (driver choice, connectivity method identified)

**Validation:**
- [ ] Pilot migration completed for at least one representative table (if applicable per Decision 9)
- [ ] Migration duration estimated from pilot results

**NOT done in Mobilize (deferred to Migrate phase):**
- DMS replication task creation and execution → `03-migrate-phase.md` Path 3
- Application code refactoring → `03-migrate-phase.md` "Application Code Refactoring & Testing"
- Cutover and rollback → `03-migrate-phase.md` "Cutover Procedure"
