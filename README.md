# Supabase to Aurora PostgreSQL Migration — Kiro Power

A [Kiro Power](https://kiro.dev/docs/powers/) that automates migrating a PostgreSQL database from Supabase to Amazon Aurora PostgreSQL. Covers assessment, target provisioning, data migration (offline and online CDC), application code refactoring, cutover, and post-migration optimization.

## When to Migrate from Supabase

Supabase is a good starting point — fast setup, bundled auth, auto-generated APIs. When teams hit limits when their workload or requirements grow beyond what the platform supports:

- **Compliance requirements** — Supabase documents SOC 2 Type 2 (the report is available to Team and Enterprise plan customers) and HIPAA support through a BAA and a HIPAA add-on request on the Team plan or higher ([SOC 2](https://supabase.com/docs/guides/security/soc-2-compliance), [shared responsibility](https://supabase.com/docs/guides/deployment/shared-responsibility-model)). Regulated industries (healthcare, fintech, government) often need other certifications, for example FedRAMP or GovCloud hosting. Check each provider's current compliance pages for those (check before relying on it; not confirmed here, 2026-10-22).
- **Connection limits** — Supabase's direct connection limit depends on the compute size of the project (check the current Supabase compute and disk documentation for the numbers). High-concurrency apps or Lambda-heavy workloads can exhaust a limit quickly. Aurora instance sizing and connection pooling are configured independently (RDS Proxy, Serverless v2).
- **Regional resilience** — A Supabase project is deployed into the region chosen at creation. A [February 2026 outage](https://supabase.com/blog/supabase-incident-on-february-12-2026) affected all services in us-east-2 for 3 hours 42 minutes, and the incident post says Supabase does not offer automatic cross-region failover for customer Postgres databases (as of that post; check before relying on it, checked 2026-10-22). AWS documents Aurora failover to a replica as typically restoring service in less than 60 seconds and often less than 30 seconds ([Aurora high availability](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html)), and Aurora Global Database for cross-region DR.
- **VPC isolation** — Supabase database hosts are reached over the internet by default. Workloads that require private network access (PCI scope reduction, internal tooling, zero-trust architectures) need a database inside a VPC. Check which private networking options your Supabase plan offers (check before relying on it, 2026-10-22).
- **Cost at production scale** — Supabase's per-project billing for compute, PITR, and read replicas (each inherits full primary disk) adds up. Aurora's Reserved Instances and shared cluster storage change the math at scale. Compare with current prices before deciding (check before relying on it, 2026-10-22).
- **Operational control** — Supabase limits PostgreSQL parameter tuning, extension availability, and major version upgrade options. Aurora exposes a large set of supported extensions ([list](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraPostgreSQLReleaseNotes/AuroraPostgreSQL.Extensions.html)), parameter group control, in-place minor and major upgrades, and blue/green upgrades with near-zero downtime. Check what your Supabase plan allows for parameters and extensions before comparing (check before relying on it, 2026-10-22).
- **AWS ecosystem integration** — If your stack is already on AWS (Lambda, ECS, Secrets Manager, CloudTrail), running the database outside AWS adds network egress costs and operational complexity. Aurora integrates natively with IAM, Secrets Manager, CloudWatch, and zero-ETL pipelines to Redshift.

If none of these apply — your workload is small, no strict security or compliance requirements, no long-term reliability requirement, no AWS dependency, and Supabase's developer experience is valuable — staying on Supabase is a reasonable choice.

## Prerequisites

- [Kiro IDE](https://kiro.dev/downloads/) installed
- [uv](https://docs.astral.sh/uv/getting-started/installation/) installed — required to run the Aurora PostgreSQL and Pricing MCP servers (`pip install uv` or see link)
- AWS CLI configured with a profile that has permissions to create Aurora clusters, IAM roles, and Secrets Manager entries (`aws sts get-caller-identity` to verify)
- A Supabase project with a PostgreSQL database
- An AWS account

## Install

1. Open the Powers panel in Kiro IDE
2. Click **Add Custom Power** → **Import from GitHub**
3. Enter the repository URL and confirm

## Quick Start

1. Verify the Supabase MCP server is connected (Kiro MCP Servers panel). The Aurora PostgreSQL MCP server is optional
2. Open a chat and say: *"I want to migrate my Supabase database to Aurora PostgreSQL"*
3. The agent verifies connectivity, then walks you through a 4-step intake (source details, app details, target proposal, plan confirmation)
4. After you confirm the plan, the agent asks before each write and confirms with you again before the live cutover

## Migration Paths

| Path | When | Downtime |
| ---- | ---- | -------- |
| `pg_dump` / `pg_restore` | Database < 100 GB, downtime acceptable | Hours |
| Native logical replication | Zero-downtime, all tables have PKs, source has IPv4 | Minutes (cutover only) |
| AWS DMS with CDC | IPv6-only source, tables without PKs, or built-in validation needed | Minutes (cutover only) |
| Hybrid | Mixed requirements | Minutes + brief window |

> **IPv6 note**: Supabase endpoints are often IPv6-only. For an online migration from an IPv6-only source the verified route is AWS DMS in dual-stack mode with CDC. A direct subscription from Aurora to an IPv6-only publisher is not yet validated. The agent detects the source address type and routes through DMS, the IPv4 add-on path, or a bastion.

## What the Agent Does

- **Assess** — connects to Supabase via MCP (OAuth, `read_only=true`), discovers schema/extensions/RLS/features, queries actual Aurora pricing, presents a multi-option cost comparison
- **Mobilize** — provisions Aurora cluster, assesses VPC networking, migrates schema, configures replication or DMS infrastructure
- **Migrate** — executes data migration, monitors lag, validates exact row counts and checksums, refactors application code (Supabase SDK → direct PostgreSQL), performs cutover
- **Modernize** — gathers statistics, reviews indexes, configures CloudWatch alarms, re-evaluates cost

## Vercel + Aurora Connectivity

Vercel functions cannot reach a private Aurora endpoint directly. This power configures **Aurora Data API + Vercel OIDC** — no static AWS keys, no VPC configuration needed. The agent walks you through one recipe: it reads your existing IAM OIDC providers first and reuses a matching one, creates a dedicated role limited to the Aurora cluster and its secret, and sets `AWS_ROLE_ARN` and a pinned `AWS_REGION`. It asks before each write and does not use static AWS keys.

## Application Code Refactoring

If your app uses `@supabase/supabase-js`, the agent refactors the data layer to direct PostgreSQL queries (`pg`, Drizzle, or Prisma), creates a migration branch, and deploys it to a Vercel preview for testing before cutover.

## MCP Servers

| Server | Purpose | Auth |
| ------ | ------- | ---- |
| `supabase` | Source database discovery | OAuth (browser flow on first use) |
| `awslabs.postgres-mcp-server` | Aurora target operations (optional; the AWS CLI and the Aurora Data API cover the same work). Started with `--allow_write_query`, so every write through it needs your confirmation | AWS credentials via default profile |
| `aws-knowledge-mcp-server` | Documentation lookups | HTTP (no auth) |
| `awslabs.aws-pricing-mcp-server` | Cost analysis | AWS credentials via default profile |

If the Aurora or Pricing MCP server fails to connect, add your AWS profile and region to the power's MCP config yourself (the agent shows you the entry and never edits MCP configuration):

```json
"env": {
  "FASTMCP_LOG_LEVEL": "ERROR",
  "AWS_PROFILE": "your-profile-name",
  "AWS_REGION": "us-east-1"
}
```

The Supabase entry in `mcp.json` ships with `read_only=true`, so queries run as a read-only Postgres user. If the Supabase MCP connects but returns the wrong project (common with multiple orgs), you scope it to one project yourself by adding `&project_ref=<your-project-ref>` to the URL of the Supabase entry:

```json
"url": "https://mcp.supabase.com/mcp?read_only=true&project_ref=<your-project-ref>"
```

Replace `<your-project-ref>` with the ID from your Supabase URL (e.g., `abcdefghijklmnopqrst` from `https://abcdefghijklmnopqrst.supabase.co`). Saving scopes the MCP connection to the specified project (and will prompt for authentication if required). The Supabase documentation lists `read_only` and `project_ref` as URL parameters that can be combined ([Supabase MCP documentation](https://supabase.com/docs/guides/getting-started/mcp)).

The agent never edits MCP configuration. When an entry needs a change, it shows you the entry and asks you to apply it and reload the MCP server.

## Power Structure

The repository is named `supabase-to-aurora-postgresql`. The power's internal `name` in `POWER.md` stays `supabase-to-apg-migration-power` so that existing installs keep working.

```text
supabase-to-aurora-postgresql/
├── POWER.md                                        # Agent instructions and intake flow
├── mcp.json                                        # MCP server configuration
├── README.md                                       # This file
└── steering/
    ├── agent-behavioral-rules.md                   # Security and execution guardrails (manual inclusion)
    ├── auth-comparison-supabase-vs-aurora.md       # Auth patterns: Supabase vs Aurora, what to replace, security risks
    ├── aurora-capabilities-reference.md            # Aurora value proposition: reliability, scale, compliance, ecosystem
    ├── aurora-pg-best-practices.md                 # Schema, indexing, connection management, post-migration checklist
    ├── database-operations-reference.md            # PostgreSQL query library
    ├── postgresql-migration-overview.md            # Phase index, naming conventions, MCP connection methods
    ├── supabase-migration-feature-mapping.md       # Auth→Cognito, Storage→S3, SDK refactoring, Vercel OIDC
    ├── supabase-migration-strategy-reference.md    # Approach decision tree, cutover checklist
    ├── supabase-prerequisites-and-connectivity.md  # Source config, IPv6/DMS networking, MCP setup
    ├── 01-assess-phase.md                          # Discovery, compatibility, cost analysis
    ├── 02-mobilize-phase.md                        # VPC assessment, Aurora provisioning, DMS setup
    ├── 03-migrate-phase.md                         # Migration execution, cutover, rollback, validation
    ├── 04-modernize-phase.md                       # Optimization, HA/DR, cost re-evaluation
    └── troubleshooting.md                          # Connection errors, DMS errors, Vercel connectivity
```

## Security

- Aurora is always provisioned in a private VPC — public access is never enabled
- All credentials stored in AWS Secrets Manager — never in chat or environment variables
- Vercel connectivity uses OIDC federation — no static AWS access keys
- Supabase MCP is read-only because its URL carries `read_only=true`; source-side writes are commands you run yourself, after you confirm them
- The agent never edits MCP configuration; it shows the entry and asks you to apply it

## Resources

- [Amazon Aurora PostgreSQL](https://aws.amazon.com/rds/aurora/postgresql-features/)
- [AWS Database Migration Service](https://aws.amazon.com/dms/)
- [Aurora Data API](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/data-api.html)
- [Vercel OIDC Federation with AWS](https://vercel.com/docs/oidc/aws)
- [Kiro Powers Documentation](https://kiro.dev/docs/powers/)
