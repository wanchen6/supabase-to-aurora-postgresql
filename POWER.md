---
name: "supabase-to-apg-migration-power"
displayName: "Supabase to Aurora PostgreSQL Migration"
description: "Migrate PostgreSQL databases from Supabase to Amazon Aurora PostgreSQL — supports offline (pg_dump/pg_restore) and online (native logical replication, AWS DMS with CDC) migration paths, application code refactoring (Supabase SDK to direct PostgreSQL), Aurora best practices, migration validation, and cost analysis."
keywords: ["supabase", "aurora", "postgresql", "migration", "database", "dms", "pg_dump", "pg_restore", "aws", "rds", "logical-replication", "cdc", "validation", "cost-analysis"]
author: "AWS Startup Database Team"
---

# Supabase to Aurora PostgreSQL Migration

## Overview

This power automates migrating a PostgreSQL database from Supabase to Amazon Aurora PostgreSQL. It covers the full migration lifecycle — assessment, target provisioning, data migration (offline and online), validation, application cutover, and post-migration optimization.

The power supports three migration paths:

1. **Offline**: pg_dump/pg_restore — simple, requires downtime
2. **Online (preferred)**: PostgreSQL native logical replication — zero-downtime CDC
3. **Online (fallback)**: AWS DMS with CDC and built-in validation — for tables where native replication doesn't work

**Rule of thumb for online migration**: Try native PostgreSQL logical replication first. If it doesn't work for a specific table (e.g., missing primary key, unsupported data type), use DMS for that table.

The agent uses up to four MCP servers: Supabase MCP (OAuth-authenticated, read-only, for source database discovery), Aurora PostgreSQL MCP (pgwire + Secrets Manager, for target operations; optional, the AWS CLI and the Aurora Data API cover the same work), AWS Knowledge MCP (for documentation lookups), and AWS Pricing MCP (for cost analysis). The user only needs to provide details and confirm at critical decision points.

**Important**: The Supabase MCP server in this power's `mcp.json` is read-only because its URL carries the `read_only=true` parameter ([Supabase MCP documentation](https://supabase.com/docs/guides/getting-started/mcp)). Any write operation on Supabase (creating publications, replication slots, schema changes) is a command the user runs. The agent provides the exact command and waits for confirmation before proceeding. The agent never edits MCP configuration: if an entry needs to change, it shows the entry and asks the user to apply it.

## Migration Paths

**Fast Path** (database < 1 GB AND brief downtime acceptable):
Skip Decisions 1-11 in `02-mobilize-phase.md` (no DMS/replication infrastructure needed). Still read `agent-behavioral-rules.md` and follow the branch/validation rules below. For databases this small, there are two execution methods:

- **If shell access is available** (EC2 bastion, CloudShell): use `pg_dump` from source → `pg_restore` to Aurora. Takes seconds. If no bastion exists, provision one per `supabase-prerequisites-and-connectivity.md` → "EC2 Bastion Setup".
- **If no shell access** (agent running from local Kiro without VPC connectivity): copy data via Supabase MCP (read) → Aurora Data API or MCP (write). Equivalent result for small databases. See `03-migrate-phase.md` → Path 1 → "Execution Method Decision".

Either way, follow these steps before touching any code:

1. Create a migration branch: `git checkout -b migration/aurora-refactor`
2. Verify git user email is a public address (Vercel rejects internal/corporate domains): `git config user.email` — set with `git config user.email "<public-email>"` if needed
3. Read `supabase-migration-feature-mapping.md` → "Vercel + Aurora Connectivity" before writing any connection helper or `db.ts` file

Then proceed to application refactoring after data is verified.

**Full Path** (database > 1 GB OR zero-downtime required):
Follow Decision 0-11 in `02-mobilize-phase.md`.

| Path | When | Details In |
| ---- | ---- | ---------- |
| Offline (pg_dump/pg_restore) | Downtime OK, < 100 GB | `03-migrate-phase.md` → Path 1 |
| Native Logical Replication | Zero-downtime, source has IPv4 | Setup: `02-mobilize-phase.md` → Native Replication. Execution: `03-migrate-phase.md` → Path 2 |
| AWS DMS with CDC | Zero-downtime, IPv6-only source, or need validation | Infrastructure: `02-mobilize-phase.md` → AWS DMS Setup. Task execution: `03-migrate-phase.md` → Path 3 |
| Hybrid | Mixed requirements | `03-migrate-phase.md` → Path 4 |

**Key constraint**: Supabase direct hosts are often IPv6-only. For an online migration from an IPv6-only source the verified route is AWS DMS in dual-stack mode with CDC. A direct subscription from Aurora to an IPv6-only publisher is not yet validated (see the Verification status table in `supabase-prerequisites-and-connectivity.md`). Native replication works when the source is reachable over IPv4.

## Try It

1. Ensure the Supabase MCP server is connected (Kiro MCP Servers panel). The Aurora PostgreSQL MCP server is optional
2. Ensure your AWS CLI is configured: `aws sts get-caller-identity`
3. Say: *"I want to migrate my Supabase database to Aurora PostgreSQL"*
4. The agent asks for your Supabase project URL, plan, and AWS region (Steps 0–3 gather the details)
5. Review and confirm the migration plan including cost estimate (Step 4 confirms the plan)
6. The agent asks before each write and confirms with you again before the live cutover

## Agent Behavioral Rules

**Before Step 0**: read `agent-behavioral-rules.md` now (call `readSteering` with `steeringFile="agent-behavioral-rules.md"`). All mandatory security, user interaction, MCP usage, and phase execution rules are there. Do not skip this.

**CRITICAL security rule enforced here**: NEVER ask the user for `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` for Vercel deployments. Always use **Vercel OIDC federation** for Aurora access — it provides short-lived credentials automatically with no static keys. See `supabase-migration-feature-mapping.md` → "Vercel + Aurora Connectivity" → "Vercel OIDC + AWS Setup" for the one OIDC recipe. If OIDC federation cannot be enabled for the project, tell the user and ask how to proceed; do not fall back to static keys.

## Getting Started — MANDATORY Intake Flow

**Flow summary**: Step 0 → verify MCP servers connected | Step 1 → auto-discover source (ask only project URL if not provided) | Step 2 → infer app stack and refactoring scope | Step 3 → propose target config | Step 4 → confirm plan with cost estimate → execute data migration → app refactoring → cutover.

**CRITICAL: You MUST complete Steps 0–4 before executing any migration. Auto-discover as much as possible via MCP and context. Only ask the user for information you genuinely cannot infer. Present what you've found, ask for confirmation, then proceed.**

**CUTOVER GATE**: Do NOT announce "ready for cutover" until BOTH are true: (1) data replication/copy is complete and validated, AND (2) application refactoring is complete and tested against Aurora on a preview/staging deployment. "I don't know your application stack" is not an acceptable state at cutover time — the app stack must be established in Step 2.

### Step 0: Verify MCP Server Configuration (MANDATORY)

**The agent never edits MCP configuration.** When an entry needs to change, show the user the exact entry and ask them to apply it and reload the MCP server. Then verify with a read-only call.

1. **Supabase MCP server** (required): Attempt to call `list_projects` or `list_tables`.

   - **If it fails entirely**: Ask the user to complete the OAuth flow in the Kiro MCP Servers panel.
   - **If it succeeds but returns the wrong project**: Do NOT edit the configuration. Tell the user which project came back and which one is expected, show the corrected URL `https://mcp.supabase.com/mcp?read_only=true&project_ref=<correct-ref>`, and ask the user to update the existing Supabase entry of this power (not to add a second entry in the workspace config, which was observed to cause auth errors). When the user says it is saved, retry `list_tables` and verify the project. If auth is needed after the change, ask the user to complete the OAuth flow. See `supabase-prerequisites-and-connectivity.md` → "Wrong Project Connected" for the steps.
   - **If it succeeds and returns the correct project**: Proceed to Step 1.

2. **Aurora PostgreSQL MCP server** (optional): Attempt to call `get_database_connection_info`. If it fails with a profile error, the user may add `AWS_PROFILE` and `AWS_REGION` to the MCP server env config (only needed if their default AWS profile isn't the correct one). Show the user this message; do not edit the configuration yourself:

   ```text
   "The Aurora PostgreSQL MCP server couldn't find your AWS credentials.
   By default it uses your default AWS CLI profile. If you need a different profile or region,
   add AWS_PROFILE and AWS_REGION to the MCP server env settings in the Kiro MCP Servers panel
   or in the power's entry of your MCP configuration. The server reconnects after you save."
   ```

   If the user does not want to configure it, or it stays unavailable, continue. Provisioning and schema work use the AWS CLI recipe in `02-mobilize-phase.md` → "Provision with the AWS CLI (no Aurora MCP needed)" and the Aurora Data API. Tell the user what is lost (convenience queries through the MCP) and that every write needs their confirmation either way.

Proceed to Step 1 once the Supabase MCP returns data from the correct project.

### Step 1: Source Database Details (Supabase)

The user provides their Supabase project URL. **Run auto-discovery immediately — do NOT ask the user for details you can query yourself.**

**Ask the user only for:** the Supabase project URL if not already provided in their message.

**Auto-discover via Supabase MCP** (run without waiting for user input): full discovery from `01-assess-phase.md` → "Source Database Discovery" — PG version, size, tables, row counts, indexes, extensions, RLS, FKs, tables without PKs, data types. Detect Supabase feature usage by querying schema presence (`auth.users`, `storage.objects`, `supabase_functions.functions`, `realtime`). Check IPv4/IPv6 status: `dig AAAA db.<project-ref>.supabase.co`.

Only ask if auto-discovery genuinely can't determine: instance size (online migration only) or IPv4 add-on status (if IPv6 check is ambiguous).

Present a single confirmation summary: what was discovered, what features are in use (or "database-only migration"), then proceed.

### Step 2: Application Details

**Infer first from context**, then ask only what you can't determine:

- If the user provided a Vercel URL → hosting is Vercel, framework is likely Next.js
- If the app is on Vercel and uses Supabase → connection method is almost certainly `@supabase/supabase-js`
- If the workspace has `package.json` → read it to confirm framework and dependencies

**Ask the user only for what you cannot infer:**

- Confirm the framework if uncertain (e.g., "This looks like a Next.js app on Vercel — is that correct?")
- EC2 instance ID — only if app is on EC2 (not relevant for Vercel/serverless)
- Connection pooling details — only if not detectable from code

If the app uses Supabase Auth, Storage, or Realtime, read `auth-comparison-supabase-vs-aurora.md` to determine what needs replacement and what security patterns must NOT carry over to Aurora.

Do NOT present a 7-item questionnaire. State what you've inferred and ask for confirmation:

```text
"Based on your Supabase project and Vercel URL, here's what I can see:
- Next.js app hosted on Vercel
- Uses @supabase/supabase-js for database access
- Database is on Supabase Free plan, <X> MB, <Y> tables
Is that correct? Anything else I should know about the app?"
```

### Step 3: Target Database Details (Aurora PostgreSQL)

Based on the source discovery results from Step 1, **propose a target configuration** with reasoning. Only ask the user for what you cannot infer.

**Ask the user:**

- **AWS Region** — ask: "Which AWS region would you like to use for Aurora?" and confirm their choice
- **AWS profile name** — the AWS CLI profile to use for MCP server authentication (must have Secrets Manager and RDS access)
- **Whether an existing Aurora cluster should be used** — or create new

**Propose with reasoning** (derive from Step 1 discovery):

- **Cluster identifier** — derive from the application/project name
- **Database name** — derive from app's logical database name; default to `postgres` if unclear — do NOT ask open-endedly
- **Aurora PostgreSQL engine version** — match source PG version from Step 1
- **Capacity model** — see `01-assess-phase.md` → Decision 6 (Instance Sizing) for the rule (Serverless v2 vs provisioned)

For all other target configuration decisions (Multi-AZ, ACU range, encryption, backup retention), apply AWS best practices defaults and explain the reasoning. Only ask the user if they want to deviate.

Present the proposed configuration as a clear summary with reasoning for each choice. Ask the user to approve or adjust before proceeding.

### Step 4: Confirm Migration Plan

**MANDATORY: You MUST complete the Cost Analysis from `01-assess-phase.md` before presenting this summary.** Use the AWS Pricing MCP server to query actual Aurora pricing for the proposed target configuration. Do NOT skip cost estimation even if the user doesn't ask for it — it is a required deliverable of the Assess phase.

Present this summary (populated from auto-discovery, user inputs, and cost analysis) and get explicit user confirmation:

```text
Source: Supabase (<plan>) — <project-ref> — PG <version> — <size>
  → <table-count> tables, <index-count> indexes, <trigger-count> triggers, <extension-count> extensions
  → Tables without primary keys: <list or "None">
  → Supabase features in use: <RLS, Auth, Storage, etc. or "None (database only)">
  → Data type compatibility: <all compatible or list issues>
Target: Aurora PostgreSQL <version> — <cluster-id> — <region>
  → <capacity-model> (<instance-type or ACU range>), encrypted, <backup-days>-day backups
Database: <database-name>
Migration Path: <offline | online-native-replication | online-dms | hybrid>
  → Reasoning: <why this path was chosen based on size, downtime tolerance, PK status>
Validation: <DMS online validation | offline spot-check>
Application: <framework> on <hosting>
  → Connection refactoring needed: <yes/no and what>
Cost Estimate (multi-option comparison completed — see details below):
  → Current Supabase cost: $<monthly> (<plan>) — source: <mcp | user_stated | unknown>
     (a Supabase MCP connection scoped to one project cannot read the plan, so ask the user; Free is a $0 baseline, and the migration adds cost; if unknown, say so and do not compute a saving)
  → Recommended Aurora option: <option-name> — $<monthly> (<capacity-model>, <region>)
  → Change vs source: <percentage and direction | "adds cost" for a $0 baseline | "n/a" if the source cost is unknown>
  → Options compared: On-Demand, 1yr RI, 3yr RI, DB Savings Plan, Serverless v2, Serverless v2 + Savings Plan
  → Notes: <why recommended option was chosen, commitment trade-offs>
```

Ask: "Does this look correct? Ready to proceed?"

**Cost interpretation rule**: If Aurora costs more than the source, the agent MUST reference `aurora-capabilities-reference.md` and present the key migration drivers — enterprise reliability (99.99% SLA, automatic Multi-AZ failover), security (VPC isolation, IAM auth, encryption), compliance (FedRAMP, HIPAA, PCI-DSS), scale (15 replicas, 256 TiB, auto-scaling, serverless), and AWS ecosystem integration (Lambda, Secrets Manager, Performance Insights, zero-ETL). Frame Aurora as the production-grade path, then ask whether the user wants to proceed.

Only proceed after the user explicitly confirms.

**Only after the user confirms should you begin execution.**

## Post-Migration

See `03-migrate-phase.md` → "Post-Migration Activities", `04-modernize-phase.md`, and `supabase-migration-feature-mapping.md`.

## Available Steering Files

- **auth-comparison-supabase-vs-aurora** — Auth deep-dive: Supabase GoTrue/JWT/RLS vs Aurora IAM/session variables; when Supabase-style auth is needed and its AWS replacements; when it should NOT be used and security risks
- **agent-behavioral-rules** — Mandatory guardrails: security, user interaction, MCP usage, and phase execution checkpoints. Read before starting any phase.
- **aurora-capabilities-reference** — Aurora PostgreSQL capabilities for value proposition: reliability, scale, compliance, cost predictability, ecosystem integration. Reference when presenting migration recommendations.
- **postgresql-migration-overview** — Central index: phase links, source platform comparison, naming conventions, AWS services, security practices
- **supabase-migration-strategy-reference** — Supabase migration: approach decision tree, validation strategy by size, flow summaries for all 4 paths, cutover checklist
- **supabase-prerequisites-and-connectivity** — Supabase migration: source config, Aurora target access, EC2 bastion, DMS, MCP setup and limitations
- **database-operations-reference** — General-purpose PostgreSQL queries: schema exploration, index analysis, troubleshooting, maintenance
- **aurora-pg-best-practices** — Post-migration Aurora best practices: schema design, indexing, query optimization, connection management, monitoring
- **supabase-migration-feature-mapping** — Supabase migration feature mapping: Auth→Cognito, Storage→S3, Realtime→AppSync, Edge Functions→Lambda, schema exclusions, CLI config, MCP usage, DMS config, application code refactoring (Supabase SDK→direct PostgreSQL), cost comparison
- **01-assess-phase** — Assess: database discovery, compatibility, extensions, platform-specific feature audit, cost analysis
- **02-mobilize-phase** — Mobilize: target provisioning, network assessment, schema migration decisions, DMS infrastructure setup (instance, endpoints, subnet group), native replication configuration, pilot migrations. Deliverable: all infrastructure ready, connections tested, approach validated.
- **03-migrate-phase** — Migrate: DMS task creation and execution, full load monitoring, CDC monitoring, application code refactoring, cutover procedure, rollback, validation, post-migration activities. Deliverable: data migrated, app switched to Aurora, source decommissioned.
- **04-modernize-phase** — Modernize: Aurora optimization, HA/DR, cost optimization (Graviton, Serverless v2, RIs, post-migration cost re-evaluation), security enhancements, monitoring setup, operational runbooks
- **troubleshooting** — Troubleshooting: generic DB connection errors (SSL, auth, timeout), DMS errors (logs, FK handling, truncate cascade), Aurora Data API errors (UUID typeHint, timeout), deployment platform errors (Vercel OIDC, private VPC connectivity)

## MCP Servers

| Server | Purpose | Connection |
| ------ | ------- | ---------- |
| `supabase` | Supabase source database discovery and assessment | HTTP — `https://mcp.supabase.com/mcp?read_only=true` (OAuth login — user selects project and permissions during setup) |
| `awslabs.postgres-mcp-server` | Aurora PostgreSQL target operations (create cluster, run queries, manage connections) | pgwire via AWS Secrets Manager — `--allow_write_query` enabled |
| `aws-knowledge-mcp-server` | AWS documentation search, extension compatibility, best practices lookup | HTTP |
| `awslabs.aws-pricing-mcp-server` | Aurora pricing queries, cost analysis, infrastructure cost comparison | stdio via uvx |

For MCP server configuration, authentication flows (Supabase OAuth, Aurora pgwire + Secrets Manager), available tools, read-only mode behavior, and known limitations, see `supabase-prerequisites-and-connectivity.md`.

## Troubleshooting — Quick Reference

For comprehensive troubleshooting (DB connection errors, SSL/TLS, DMS errors, Aurora Data API, deployment platform connectivity), see `troubleshooting.md`.
