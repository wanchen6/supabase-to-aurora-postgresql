# Troubleshooting — PostgreSQL Migration

## Overview

Common errors and resolutions encountered during PostgreSQL migrations to Aurora/RDS, organized by category. Reference this file when encountering errors during any migration phase.

## Generic Database Connection Errors

### SSL/TLS Connection Failures

**Symptom**: `SSL connection is required` or `SSL error: certificate verify failed` when connecting to Aurora.

**Root Cause**: Aurora enforces SSL by default via the `rds.force_ssl` parameter. Clients must use `sslmode=require` (or stricter) and require the RDS CA certificate bundle.

**SSL Modes Summary**:

| Mode | Encrypted | CA Verified | Hostname Verified |
| ---- | --------- | ----------- | ----------------- |
| `disable` | ❌ | ❌ | ❌ |
| `allow` | maybe | ❌ | ❌ |
| `prefer` | maybe | ❌ | ❌ |
| `require` | ✅ | ❌ | ❌ |
| `verify-ca` | ✅ | ✅ | ❌ |
| `verify-full` | ✅ | ✅ | ✅ |

**Resolution**:
1. Verify the Aurora parameter group has `rds.force_ssl = 1` (this is the default and recommended setting — do NOT disable it)
2. Update the connection string to include `sslmode=require` at minimum:
   ```
   psql "host=<aurora-endpoint> port=5432 dbname=<db> user=postgres sslmode=require"
   ```
3. For `sslmode=verify-ca` or `sslmode=verify-full` (recommended for production), download the RDS CA bundle and set `sslrootcert`:
   ```bash
   wget https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
   psql "host=<aurora-endpoint> port=5432 dbname=<db> user=postgres sslmode=verify-full sslrootcert=global-bundle.pem"
   ```
4. For application connection pools, set the SSL option in the pool config:
   ```javascript
   // Node.js (pg) — sslmode=require equivalent
   const pool = new Pool({ ssl: { rejectUnauthorized: false } })

   // Node.js (pg) — sslmode=verify-full equivalent
   const pool = new Pool({ ssl: { rejectUnauthorized: true, ca: fs.readFileSync('global-bundle.pem') } })
   ```

### pg_hba.conf / Authentication Errors

**Symptom**: `FATAL: no pg_hba.conf entry for host` or `FATAL: password authentication failed`.

**Root Cause**: The client IP is not allowed to connect, or the credentials are incorrect.

**Resolution**:
1. On Aurora/RDS, `pg_hba.conf` is managed via security groups — verify the security group allows inbound TCP 5432 from the client IP or security group:
   ```bash
   aws ec2 describe-security-groups --group-ids <aurora-sg-id> --region <region> \
     --query "SecurityGroups[0].IpPermissions[?FromPort==\`5432\`]"
   ```
2. Verify the password is correct — retrieve from Secrets Manager:
   ```bash
   aws secretsmanager get-secret-value --secret-id <secret-id> --region <region> \
     --query SecretString --output text | python3 -c "import sys,json; print(json.load(sys.stdin)['password'])"
   ```
3. Verify the username exists on the target: `SELECT usename FROM pg_user;`
4. For source databases (Supabase, Heroku, etc.), check the platform dashboard for the correct connection credentials

### Connection Timeout

**Symptom**: `connection timed out` or `could not connect to server: Operation timed out`.

**Root Cause**: Network path is blocked — security group, NACL, routing, or the endpoint is unreachable.

**Resolution**:
1. Verify the endpoint is correct (cluster endpoint for writes, reader endpoint for reads)
2. Check security group inbound rules allow TCP 5432 from the source
3. Check VPC route tables — if connecting from outside the VPC, ensure a NAT gateway or internet gateway is configured
4. Test basic connectivity: `nc -zv <endpoint> 5432` (should show "Connection succeeded")
5. If connecting via SSM to an EC2 bastion, verify the bastion's security group allows outbound to Aurora
6. For cross-VPC or cross-account connections, verify VPC peering or Transit Gateway routes


### IPv6 / Dual-Stack Connectivity

**Symptom**: DMS replication instance cannot connect to a source database that resolves to an IPv6 address (AAAA record only), or connections time out despite correct security group rules.

**Root Cause**: Some managed database providers (e.g., Supabase) expose endpoints that resolve to IPv6 addresses only. By default, DMS replication instances and Aurora clusters use IPv4-only networking. If the source endpoint is IPv6-only, the DMS replication instance cannot reach it unless Dual-stack mode is enabled.

**How to check**: Verify whether the source host resolves to IPv6:
```bash
# Check DNS resolution
dig AAAA db.<project-ref>.supabase.co
nslookup -type=AAAA db.<project-ref>.supabase.co

# If you see an AAAA record (e.g., 2a05:d014:...), the host is IPv6
```

**Resolution**: Follow the complete IPv6 and dual-stack setup steps in `supabase-prerequisites-and-connectivity.md` → "IPv6 and Dual-Stack Connectivity (Preferred for Supabase)" and → "Online Migration from IPv6-Only Supabase Sources".

### Subscription worker cannot connect to an IPv6-only publisher

**Symptom**: An Aurora `CREATE SUBSCRIPTION` to an IPv6-only Supabase host stays enabled, but the Aurora PostgreSQL log shows `could not connect to the publisher ... port 5432 failed: Connection timed out` on every retry, and the Supabase side shows no connection.

**First check**: Open a control connection from another resource in the same subnets, security group and route table, such as a temporary Lambda function or EC2 instance. A reply to the PostgreSQL SSL request (`S`) on port 5432 shows the network path and the Supabase side work, which narrows the question to the connection the subscription worker starts.

**Route**: Use DMS in dual-stack mode with CDC, or the IPv4 add-on path, for the online migration. A direct subscription to an IPv6-only publisher is not yet validated. The Verification status table in `supabase-prerequisites-and-connectivity.md` (under "Online Migration from IPv6-Only Supabase Sources") has the evidence and the re-test steps.

## DMS-Specific Errors

### Reading DMS Task Logs

DMS task logs are available in CloudWatch Logs under the log group `dms-tasks-<replication-instance-id>`, in a stream named `dms-task-<task-external-id>` (the external id is the last part of the task ARN). DMS assigns both names and you cannot set them ([logging task settings](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Tasks.CustomizingTasks.TaskSettings.Logging.html)). Key log signatures:

- `]E:` — Error messages (critical, usually cause task failure)
- `]W:` — Warning messages (may indicate data issues but task continues)
- `]I:` — Informational messages (normal operation)

**Troubleshooting workflow**:

1. Check the DMS task status in the console or via CLI:

   ```bash
   aws dms describe-replication-tasks \
     --filters Name=replication-task-id,Values=<task-id> \
     --region <region> \
     --query "ReplicationTasks[0].{Status:Status,StopReason:StopReason,LastError:ReplicationTaskStats}"
   ```

2. Search CloudWatch Logs for errors:

   ```bash
   aws logs filter-log-events \
     --log-group-name dms-tasks-<replication-instance-id> \
     --filter-pattern "]E:" \
     --region <region> \
     --query "events[*].message" --output text
   ```

3. Correlate DMS errors with target database logs by timestamp. When the DMS message points to the target database logs, follow [DMS table error: exitwhy 1 / check target database logs](#dms-table-error-exitwhy-1--check-target-database-logs)
4. Common DMS error patterns:
   - `Cannot find or open table` — table doesn't exist on target; check schema migration
   - `No primary key` — CDC cannot replicate UPDATE/DELETE without a PK; add PK or use full load only
   - `Replication slot does not exist` — slot was dropped or never created; recreate on source
   - `ENUM type does not exist` — create ENUM types on target before starting DMS task

### DMS Endpoint Connection Failures

**Symptom**: `test-connection` returns `failed` status.

**Resolution**:
1. For source endpoints (Supabase, Heroku, etc.): verify the DMS replication instance subnet has internet access (NAT gateway) since source databases are on the public internet
2. For target endpoints (Aurora): verify the DMS replication instance security group can reach Aurora on port 5432
3. Verify credentials are correct — test with psql from an EC2 instance in the same VPC
4. Check that the DMS replication instance is in the correct VPC and subnet group

### DMS table error: exitwhy 1 / check target database logs

**Symptom**: a table shows `Table error` in `describe-table-statistics` and the DMS task log has only lines like `Command failed to load data with exit error code 0 and exitwhy 1. Please check target database logs for more information.` and `Failed to load data from csv file.` The DMS log does not name the cause anywhere in the failure window. The cause is in the target database log.

Do this lookup yourself. Do not ask the user to read the logs.

1. Read the `]E:` and `]W:` lines from the task log group `dms-tasks-<replication-instance-id>` (see "Reading DMS Task Logs" above). Note the failure time and the table name.
2. List the target error log files, newest last, and pick the one that covers the failure time. `<writer>` is the writer instance of the Aurora cluster. `LastWritten` is in epoch milliseconds:

   ```bash
   aws rds describe-db-log-files --db-instance-identifier <writer> --filename-contains error/postgresql --query "DescribeDBLogFiles[].[LogFileName,LastWritten,Size]" --output text --region <region>
   ```

3. Download that file and keep the error lines. Without a starting marker the API returns at most the 10000 most recent lines, and a marker of `0` starts at the beginning of the file ([DownloadDBLogFilePortion](https://docs.aws.amazon.com/AmazonRDS/latest/APIReference/API_DownloadDBLogFilePortion.html)). The CLI passes `--starting-token 0` as that marker. The CLI page ([download-db-log-file-portion](https://docs.aws.amazon.com/cli/latest/reference/rds/download-db-log-file-portion.html)) describes the pagination token. AWS recommends publishing the logs to CloudWatch for large reads:

   ```bash
   aws rds download-db-log-file-portion --db-instance-identifier <writer> --log-file-name <file> --starting-token 0 --output text --region <region> | grep -E "ERROR:|DETAIL:"
   ```

   Look at the `ERROR:` and `DETAIL:` lines that name the target database and the table.

Aurora PostgreSQL rotates the log every 60 minutes by default, and the file name pattern follows the `log_filename` parameter, so a failure near an hour boundary can sit in two files ([log file rotation](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_LogAccess.Concepts.PostgreSQL.overview.parameter-groups.html)). The log download API uses resources on the database instance, so AWS recommends publishing the logs to CloudWatch Logs and reading them there. Aurora PostgreSQL can export the `postgresql` log type to the log group `/aws/rds/cluster/<cluster-id>/postgresql` ([publishing to CloudWatch Logs](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.CloudWatch.html)). Turning the export on is a cluster change (`aws rds modify-db-cluster --cloudwatch-logs-export-configuration` with `EnableLogTypes`), so ask first. It publishes from that point on and does not backfill.

Map the cause:

| Target log line | Cause | Fix |
| --------------- | ----- | --- |
| `duplicate key value violates unique constraint` | The target is not empty. A `DO_NOTHING` load found rows from an earlier run. | [Reload after a failed or partial run](#reload-after-a-failed-or-partial-run) |
| `violates foreign key constraint` | Load order, or the foreign key bypass is missing. | Check `AfterConnectScript` on the target endpoint (Mobilize Decision 6) |
| `cannot truncate a table referenced in a foreign key constraint` | `TRUNCATE_BEFORE_LOAD` on a preloaded schema. | Mobilize Decision 5: `DO_NOTHING` on an empty target |
| `invalid input syntax for type` or `is of type ... but expression is of type` | Type mapping. | Compare the column types on both sides and fix the mapping |

### DMS validation: Mismatched records

**Symptom**: `describe-table-statistics` shows `Mismatched records` (or `Suspended records`) for a table when validation is on.

The target's validation failures table holds the reason. Read it yourself as the first step, before you give the user any explanation. The queries, the control schema lookup and the `_v2` / `_v1` choice are in [`03-migrate-phase.md` Path 3 Step 4](03-migrate-phase.md#step-4-monitor-dms-validation). Do not hard-code the schema.

**What the columns mean** ([DMS data validation](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Validating.html)): `"FAILURE_TYPE"` is `RECORD_DIFF`, `MISSING_SOURCE`, `MISSING_TARGET` or `TABLE_WARNING`. `"KEY"` holds the primary key of the failed record. `"DETAILS"` is a JSON formatted string of the source and target column values that do not match for that key. The page does not say which of the two entries is the source, so do not rely on their order.

**Decode it**:

1. Read `"FAILURE_TYPE"` and `"DETAILS"`. Take the key values from `"KEY"` (`"KEY"::json->'key'`).
2. Run the same `SELECT ... WHERE pk IN (...)` on both endpoints (source read-only, target with psql or the Data API), with `md5(t::text)` per row. Use the key lookup and not the order of the entries in `"DETAILS"`.
3. Compare what the values mean, not how they print. Equal hashes for every failed key mean the rows are identical and the mismatch is a difference in how the validator represents the value.
4. If only the representation differs and the independent counts and checksums match (exact row counts and `md5(string_agg(x::text, ',' ORDER BY pk))` per table, session time zone UTC on both sides), record it as a validation representation difference and continue. If any value differs, treat it as data loss and stop.

To join the failure rows to a target table by key, use `"KEY"::json->'key'->>0 = t.id::text` (a single-column primary key). This follows the AWS suggestion to use the key to compare the record on both sides ([source and target data mismatch](https://repost.aws/knowledge-center/dms-source-target-data-mismatch)). The same page and [DMS task issues](https://repost.aws/knowledge-center/dms-task-issues-during-migration) also list checking that the source data type maps to a compatible target type, and checking character sets and collations.

For `MISSING_SOURCE` and `MISSING_TARGET`, compare the key on both sides: a row that exists on one side only is a data difference until you show otherwise. For `Suspended records`, the source row is likely changing while DMS compares it. Read it again after the writes stop.

The failures table is not cleared by a reload. Group the rows by minute to tell which load wrote them (the grouped query is in `03-migrate-phase.md` Path 3 Step 4).

### DMS validation mismatch on boolean columns

**Symptom**: with validation on, a table that has a boolean column ends as `Mismatched records`, while exact row counts and checksums are equal on both sides.

**Observed once** (one run, full load, PostgreSQL source and Aurora PostgreSQL target, engine version 3.5.4): the failure rows were `RECORD_DIFF`. For the rows where the target boolean was true, `"DETAILS"` was `[[{'<boolean_column>': '\u0001'}, {'<boolean_column>': '1'}],]`. For the rows where it was false, it was `[[{'<boolean_column>': '<null>'}, {'<boolean_column>': '0'}],]` (a NUL byte prints as `<null>`). The two entries in each row are the same value in two representations: a raw boolean byte (0x01 or 0x00) and the text `'1'` or `'0'`. So the mismatch was a datatype representation difference in the validator, not a data difference. This agreed with the independent evidence: row counts and `md5(string_agg(x::text, ',' ORDER BY pk))` checksums were identical on all tables (session time zone UTC on both sides), and the `md5(t::text)` of every failed key was identical on both sides. A task built by hand earlier showed the same. After the boolean settings moved from the extra connection attribute string into `PostgreSQLSettings` on the source, the printed values changed, but the mismatch remained.

**What to do**: follow the decoding steps in [DMS validation: Mismatched records](#dms-validation-mismatched-records). Equal checksums (the counts-and-checksums comparison: exact row counts and `md5(string_agg(x::text, ',' ORDER BY pk))` per table, session time zone UTC on both sides) and equal key hashes are the deciding evidence. Never act on a validation mismatch alone.

Do not skip validation. Record what you saw. The cause of the representation difference inside DMS was not determined, and moving the boolean settings between the extra connection attribute string and `PostgreSQLSettings` did not change the outcome. This is not claimed to be a DMS defect, and it is not claimed that another engine version behaves the same or differently (untested). More detail: [DMS Validation Mismatch on Boolean Columns](03-migrate-phase.md#dms-validation-mismatch-on-boolean-columns).

### DMS Full Load + CDC Foreign Key Handling

**Symptom**: DMS full load fails with foreign key constraint violations, or `TRUNCATE TABLE` / `DROP TABLE` fails with `cannot truncate a table referenced in a foreign key constraint`. The prep mode rule is stated once, in Mobilize [Decision 5](02-mobilize-phase.md#decision-5-dms-target-table-preparation-mode): a preloaded schema uses `DO_NOTHING` on an empty target. Read it first.

**Root Cause**: DMS loads each table one at a time during full load, but table order is not guaranteed. If a child table's rows arrive before the parent table's rows, foreign key constraints cause the task to fail. PostgreSQL also has a failsafe mechanism that prevents truncation even when `session_replication_role` is set to `replica`.

Reference: [AWS DMS — Using PostgreSQL as a target](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Target.PostgreSQL.html)

**Resolution — Method 1: Use `session_replication_role` (recommended)**

Set the `session_replication_role` parameter to `replica` on the DMS target endpoint. This deactivates foreign key triggers during full load so DMS can load tables in any order without FK violations.

You can set this via the DMS endpoint Extra Connection Attribute (ECA):

```
AfterConnectScript=SET session_replication_role = 'replica'
```

Or set it via the Aurora/RDS custom parameter group before starting the DMS task. This is the less preferred way: a parameter group value affects every session, including the application, so use it only if you want triggers and FK checks off for all sessions, and set it back to `origin` afterwards:

1. In the RDS console (or via CLI), modify the custom parameter group for your Aurora cluster
2. Set `session_replication_role` = `replica`
3. Apply the change (no reboot required — this is a dynamic parameter)

```bash
# Via AWS CLI
aws rds modify-db-cluster-parameter-group \
  --db-cluster-parameter-group-name <your-custom-param-group> \
  --parameters "ParameterName=session_replication_role,ParameterValue=replica,ApplyMethod=immediate"
```

**Note**: `ALTER SYSTEM` is not allowed on RDS/Aurora — you must use the parameter group.

After migration cutover is complete and the application is writing directly to Aurora, re-enable FK enforcement:

```bash
aws rds modify-db-cluster-parameter-group \
  --db-cluster-parameter-group-name <your-custom-param-group> \
  --parameters "ParameterName=session_replication_role,ParameterValue=origin,ApplyMethod=immediate"
```

Then verify FK integrity:

```sql
-- Check for orphaned rows (example for a specific FK relationship)
SELECT child.id FROM child_table child
LEFT JOIN parent_table parent ON child.parent_id = parent.id
WHERE parent.id IS NULL;
```

**Resolution — Method 2: Use `DO_NOTHING` target table preparation mode**

If you set the DMS task's target table preparation mode to `DO_NOTHING`, DMS will not attempt to `DROP` or `TRUNCATE` target tables before loading. This avoids the PostgreSQL failsafe that blocks truncation of tables referenced by foreign keys, even when `session_replication_role` is set to `replica`. When to choose it is stated once in Mobilize [Decision 5](02-mobilize-phase.md#decision-5-dms-target-table-preparation-mode).

```json
"FullLoadSettings": {
  "TargetTablePrepMode": "DO_NOTHING"
}
```

**Important**: `DO_NOTHING` keeps every row that is already there, so the target tables must be empty before the load starts. A non-empty target ends in `duplicate key` errors. Do not skip this check.

**A failed first run leaves partial data.** A load that failed on the truncate step had already loaded some tables, and switching to `DO_NOTHING` and starting again made those tables fail with duplicate keys. After any failed or partial run: stop the task, run `TRUNCATE ... CASCADE` on every task table on the target, reload with `aws dms start-replication-task --start-replication-task-type reload-target`, and verify exact counts. The ordered recipe is [Reload after a failed or partial run](#reload-after-a-failed-or-partial-run).

### DMS Truncate/Drop Cascade for Reload Scenarios

**Symptom**: DMS task with `DROP_AND_CREATE` (or with `TRUNCATE_BEFORE_LOAD`, which you should not use on tables that foreign keys reference) target table prep mode fails because PostgreSQL cannot drop or truncate a table that is referenced by foreign key constraints from other tables.

**Root Cause**: PostgreSQL prevents `TRUNCATE` and `DROP` on tables that have dependent foreign keys from other tables. Even with `session_replication_role = 'replica'`, the failsafe mechanism blocks truncation of FK-referenced tables.

**Resolution**:

If you need to reload tables from scratch (e.g., after a failed DMS task), manually truncate with `CASCADE` before restarting the DMS task. `TRUNCATE ... CASCADE` automatically truncates all dependent (child) tables as well.

```sql
-- Truncate the parent table and all tables that reference it via FK
TRUNCATE TABLE parent_table CASCADE;
```

**Critical**: When you use `CASCADE`, all related tables in the FK chain are also truncated. You MUST reload ALL cascaded tables — not just the one you intended to truncate. Otherwise those dependent tables will be empty.

Steps for a clean reload:

1. Identify all tables that will be affected by the cascade:

   ```sql
   -- Find all tables that reference a given table via FK
   SELECT DISTINCT tc.table_schema, tc.table_name
   FROM information_schema.table_constraints tc
   JOIN information_schema.constraint_column_usage ccu
     ON tc.constraint_name = ccu.constraint_name
   WHERE tc.constraint_type = 'FOREIGN KEY'
     AND ccu.table_name = 'parent_table';
   ```

2. Run `TRUNCATE ... CASCADE` on the target:

   ```sql
   TRUNCATE TABLE parent_table CASCADE;
   ```

3. Ensure the DMS task table mappings include ALL cascaded tables so they are all reloaded
4. Reload with `TargetTablePrepMode: DO_NOTHING` (since you already truncated manually), following [Reload after a failed or partial run](#reload-after-a-failed-or-partial-run)

Alternatively, disable FK checks via the parameter group, truncate individually, then re-enable:

```bash
# Disable FK triggers via parameter group
aws rds modify-db-cluster-parameter-group \
  --db-cluster-parameter-group-name <your-custom-param-group> \
  --parameters "ParameterName=session_replication_role,ParameterValue=replica,ApplyMethod=immediate"
```

**Note**: `ALTER SYSTEM` is not allowed on RDS/Aurora — you must use the parameter group.

Then truncate tables in reverse dependency order (children before parents):

```sql
TRUNCATE TABLE child_table;
TRUNCATE TABLE parent_table;
```

After truncation, re-enable FK enforcement:

```bash
aws rds modify-db-cluster-parameter-group \
  --db-cluster-parameter-group-name <your-custom-param-group> \
  --parameters "ParameterName=session_replication_role,ParameterValue=origin,ApplyMethod=immediate"
```

This avoids the cascade but requires you to truncate tables in reverse dependency order (children before parents).

### Reload after a failed or partial run

Use this recipe after any failed, stopped or partial full load, and whenever you need to load every table again. It worked in one test run. Follow the order. Poll the status calls at intervals (30 seconds or more), not in a tight loop.

1. Stop the task: `aws dms stop-replication-task --replication-task-arn <task-arn>`. Wait until `describe-replication-tasks` shows `stopped`. In one run this took 1 to 2 minutes.
2. Only if task settings change: `aws dms modify-replication-task` (the task must be `stopped`). Wait for the status to return to `stopped` after `modifying`.
3. On the target, run `TRUNCATE ... CASCADE` on every task table (the list is in the table mapping), then check that every count is 0. `CASCADE` empties dependent tables too, so the list must cover all of them.
4. If you changed an endpoint, run `aws dms test-connection` for it and wait until `describe-connections` shows `successful`. Editing an endpoint invalidates its earlier test, and `start-replication-task` fails with `InvalidResourceStateFault` until the test is `successful`.
5. Clear old validation failures (see the note below), then start: `aws dms start-replication-task --replication-task-arn <task-arn> --start-replication-task-type reload-target`.
6. Wait until `describe-table-statistics` shows `Table completed` for every table. A table in `Table error` goes back to [DMS table error: exitwhy 1 / check target database logs](#dms-table-error-exitwhy-1--check-target-database-logs).
7. Verify exact counts and checksums against the source (`count(*)` and `md5(string_agg(x::text, ',' ORDER BY pk))` per table, session time zone UTC on both sides). Statistics-view estimates are not valid counts.

**The validation failures table keeps old rows.** `reload-target` and truncating the data tables do not clear `awsdms_validation_failures_v1`. Rows from an earlier run stay. Before a reload, either filter by `"FAILURE_TIME"` when you read it, or delete only this task's rows with `DELETE FROM <control-schema>.<failures-table> WHERE "TASK_NAME" = '<task-external-id>'` (the last part of the task ARN). The value DMS stores in `"TASK_NAME"` was not verified, so run `SELECT DISTINCT "TASK_NAME" FROM <control-schema>.<failures-table>` first, use the value it shows, and check the number of deleted rows. Truncating a `_v1` table is safe only when no other task validates against the same target database. Never drop or truncate a `awsdms_validation_failures_v2` table: it is shared by every task with validation on the target. The control schema is a task setting, so find it first; the queries (quoted uppercase column names, `FAILURE_TIME` ordering and the grouped summary) are in [`03-migrate-phase.md` Path 3 Step 4](03-migrate-phase.md#step-4-monitor-dms-validation).

**What this recipe does not show.** The full-load test that produced it covered the full load only. The CDC insert, update and delete counters were not exercised in that test, so nothing here claims that CDC was verified. A later cutover rehearsal saw CDC changes apply on one run; it is an [observation](03-migrate-phase.md#cutover-rehearsal-on-a-dms-loaded-target-observed-on-one-run), not a verification.

## Aurora Data API Errors

### UUID Type Mismatch

**Symptom**: `operator does not exist: uuid = text` when using Aurora Data API with UUID columns.

**Root Cause**: The RDS Data API treats all string parameters as `text` type by default. When comparing against a `uuid` column, PostgreSQL cannot implicitly cast `text` to `uuid` in the `=` operator.

**Resolution**:

Option A — Add `typeHint: "UUID"` to the SqlParameter:
```json
{
  "name": "id",
  "value": { "stringValue": "550e8400-e29b-41d4-a716-446655440000" },
  "typeHint": "UUID"
}
```

Option B — Cast to `::text` in the SQL query:
```sql
SELECT * FROM users WHERE id::text = :id
```

Option C — Cast the parameter to `::uuid` in the SQL query:
```sql
SELECT * FROM users WHERE id = :id::uuid
```

Option A is preferred because it preserves index usage on the `uuid` column. Options B and C may prevent index scans depending on the query planner.

### Data API Timeout (45 seconds)

**Symptom**: `Communications link failure` or timeout after ~45 seconds.

**Root Cause**: Aurora Data API has a hard 45-second execution timeout per statement.

**Resolution**:
- For long-running queries, use a direct psql connection (via bastion/SSM) instead of Data API
- For large data operations, batch into smaller transactions
- Data API is best suited for small, quick operations (CRUD, validation queries, ad-hoc checks)

## Deployment Platform Errors

### Vercel OIDC / IAM Authentication Failures

**Symptom**: `InvalidIdentityTokenException` or `OpenIDConnect provider's HTTPS certificate doesn't match configured thumbprint` when using Vercel OIDC with AWS.

**Root Cause**: The AWS IAM OIDC provider for Vercel is not configured correctly, or the trust policy doesn't match the Vercel project/team, or the deployment was built before the variables were set.

The recipe is in `supabase-migration-feature-mapping.md` → "Vercel + Aurora Connectivity" → "Vercel OIDC + AWS Setup". It is the only OIDC recipe in the power. Check these in order:

1. **Issuer mode and provider.** Read the existing providers (`aws iam list-open-id-connect-providers`). The provider URL must match the issuer mode shown in the Vercel project (Settings → Security): `https://oidc.vercel.com/<team-slug>` for Team, `https://oidc.vercel.com` for Global. Its audience must include `https://vercel.com/<team-slug>`.
2. **Thumbprint.** If AWS asks for a thumbprint or reports a thumbprint mismatch, compute the current one. It can change, so do not reuse an old value:

   ```bash
   # Get the OIDC provider thumbprint (Global issuer host shown; for Team mode use the same host)
   openssl s_client -servername oidc.vercel.com -showcerts -connect oidc.vercel.com:443 </dev/null 2>/dev/null | \
     openssl x509 -fingerprint -noout | sed 's/://g' | cut -d= -f2 | tr '[:upper:]' '[:lower:]'
   ```

   If the thumbprint error persists, update the provider with the latest value:

   ```bash
   aws iam update-open-id-connect-provider-thumbprint \
     --open-id-connect-provider-arn arn:aws:iam::<account-id>:oidc-provider/<provider-host-and-path> \
     --thumbprint-list <new-thumbprint>
   ```

3. **Role trust policy.** The condition keys use the provider host as the prefix (`oidc.vercel.com/<team-slug>` for Team, `oidc.vercel.com` for Global). The `aud` must be `https://vercel.com/<team-slug>` and the `sub` must match `owner:<team-slug>:project:<project-name>:environment:<environment>`. A `sub` limited to `environment:production` makes preview deployments fail with an auth error, so use a `StringLike` wildcard (or list `production` and `preview`) when previews must work:

   ```json
   {
     "Effect": "Allow",
     "Principal": { "Federated": "arn:aws:iam::<account-id>:oidc-provider/oidc.vercel.com/<team-slug>" },
     "Action": "sts:AssumeRoleWithWebIdentity",
     "Condition": {
       "StringEquals": {
         "oidc.vercel.com/<team-slug>:aud": "https://vercel.com/<team-slug>"
       },
       "StringLike": {
         "oidc.vercel.com/<team-slug>:sub": "owner:<team-slug>:project:<project-name>:environment:*"
       }
     }
   }
   ```

4. **Variables and a new build.** `AWS_ROLE_ARN` and a pinned `AWS_REGION` must exist in the environment of the deployment (for a migration preview: branch-scoped Preview variables, added after the branch exists on the remote). After any variable change, trigger a new build, because a deployment that is already built keeps the old values.

### Vercel / Serverless Platform Cannot Connect to Private Aurora

**Symptom**: Application deployed on Vercel (or similar serverless platform) cannot connect to Aurora in a private VPC.

**Root Cause**: Vercel functions run outside your VPC and cannot reach private Aurora endpoints.

**Resolution options**:
1. **Aurora Data API** — no VPC connectivity needed; access Aurora via HTTPS. Best for simple CRUD operations (< 45s execution time). Requires enabling Data API on the Aurora cluster.
2. **Aurora public access** — enable public accessibility on the Aurora instance and restrict via security group to known IPs. Less secure but simplest.
3. **Vercel Secure Compute** (Enterprise plan) — provides VPC peering from Vercel to your AWS VPC. Most secure but requires Vercel Enterprise.
4. **AWS Amplify Hosting** — alternative to Vercel that runs inside your AWS account. SSR uses Lambda functions that can be configured with VPC access to reach private Aurora. Also supports IAM compute roles and VPC endpoints for private connectivity.
5. **PrivateLink** — create a VPC endpoint service for Aurora and connect from Vercel via PrivateLink (requires Vercel Enterprise).

For the complete OIDC setup, IAM role configuration, and Aurora Data API client implementation, see `supabase-migration-feature-mapping.md` → "Vercel + Aurora Connectivity".
