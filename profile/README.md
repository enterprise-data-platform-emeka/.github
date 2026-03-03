
# Enterprise Data Platform (EDP)

A production-grade, cloud-native data platform built on AWS.
It moves data from a source database through automated pipelines into analytics-ready datasets that power business dashboards.

---

## What This Platform Does  

Imagine a company has a PostgreSQL database that records customer orders, transactions, and events all day long.

The problem: that raw database data is not in a shape that analysts or dashboards can easily use. Analysts need clean, pre-aggregated data. They should not run queries directly against the production database.

This platform solves that by:

1. **Capturing every change** made to the source database (inserts, updates, deletes) in real time using CDC (Change Data Capture)
2. **Landing that raw data** into an immutable storage layer (Bronze) so nothing is ever lost
3. **Validating and cleaning** the raw data using Apache Spark (Bronze → Silver)
4. **Routing bad records** to a Quarantine bucket for investigation rather than silently dropping them
5. **Aggregating clean data** into business-level summaries using dbt and SQL (Silver → Gold)
6. **Serving that Gold data** through Redshift Serverless so BI tools like Tableau, Power BI, or QuickSight can connect and build dashboards
7. **Orchestrating the whole pipeline** automatically using Apache Airflow (MWAA) on a schedule
8. **Logging everything** to CloudWatch for observability and alerting

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                             CONTROL PLANE                                       │
│                                                                                 │
│          ┌─────────────────────┐          ┌─────────────────────┐               │
│          │  MWAA Orchestration │────────► │   CloudWatch Logs   │               │
│          └──────────┬──────────┘          └─────────────────────┘               │
│                     │ triggers                                                  │
└─────────────────────┼───────────────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                             PROCESSING LAYER                                    │
│                                                                                 │
│   ┌──────────────────────────────┐     ┌─────────────────────────────────┐      │
│   │  Glue Spark Job              │     │  dbt using Athena               │      │
│   │  Validation & Reconstruction │     │  SQL transformations            │      │
│   └──────────────┬───────────────┘     └─────────────────┬───────────────┘      │
│                  │ reads/writes                          │ reads/writes          │
└──────────────────┼────────────────────────────────────────┼────────────────────┘
                   │                                        │
                   ▼                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                             S3 DATA LAKE                                        │
│                                                                                 │
│  ┌──────────────┐   ┌──────────────────┐   ┌──────────────────┐                │
│  │    BRONZE    │   │     SILVER        │   │      GOLD         │               │
│  │  (Immutable) │──►│ (Reconstructed   │──►│  (Business        │               │
│  │  Raw CDC     │   │   State)          │   │   Aggregations)   │               │
│  └──────────────┘   └──────────────────┘   └──────────────────┘                │
│         │                    │                                                  │
│         │         ┌──────────────────┐                                          │
│         └────────►│   QUARANTINE     │                                          │
│                   │ (Invalid Records) │                                          │
│                   └──────────────────┘                                          │
└───────────────────────────────────────────────────────────────┬────────────────┘
                                                                │
┌──────────────────────────────┐                                ▼
│        SOURCE LAYER          │   ┌──────────────────────────────────────────────┐
│                              │   │              SERVING LAYER                   │
│  PostgreSQL ──► CDC ─────────┤   │                                              │
│   Database   (DMS)           │   │  Redshift Serverless + Spectrum ──► Dashboard│
└──────────────────────────────┘   └──────────────────────────────────────────────┘
```

---

## The Medallion Architecture Explained

The data lake uses a pattern called **Medallion Architecture**. Each layer is a separate S3 bucket with a specific purpose.

### Bronze — The Immutable Raw Layer

**What lands here:** Every CDC event from DMS exactly as it arrived. No changes, no filtering.

**Why immutable:** If a bug is discovered in the Silver or Gold transformation logic, engineers can re-process from Bronze without losing any original data. Bronze is the source of truth.

**Format:** Parquet files partitioned by date, one folder per source table.

### Silver — The Reconstructed State

**What lands here:** Clean, validated, deduplicated records. Each row represents the current state of a record (the CDC deletes and updates have been resolved into a consistent snapshot).

**How it gets here:** An AWS Glue PySpark job reads Bronze, applies schema validation, deduplicates records, handles CDC operation types (INSERT/UPDATE/DELETE), and writes clean data to Silver. Any record that fails validation goes to Quarantine instead.

### Gold — Business Aggregations

**What lands here:** Pre-aggregated, business-level datasets. Examples: daily revenue by product, monthly active users, weekly inventory levels.

**How it gets here:** dbt (data build tool) runs SQL models that read from Silver via Athena and write aggregated results to Gold.

### Quarantine — Invalid Records

**What lands here:** Any record that failed Silver validation (missing required fields, wrong data types, referential integrity violations, etc.).

**Why keep bad records:** So engineers can diagnose data quality problems at the source rather than silently dropping data.

### Athena Results

**What lands here:** Query result files from Amazon Athena SQL queries. This bucket acts as the designated output location for all Athena workgroup queries.

---

## Data Flow — Step by Step

```
Step 1: Source
  PostgreSQL database has new activity (insert/update/delete)

Step 2: CDC Capture
  AWS DMS (Database Migration Service) captures that change
  and writes it as a Parquet file to the Bronze S3 bucket
  The file includes: the changed row + the operation type (I/U/D) + timestamp

Step 3: Orchestration triggers
  MWAA (Airflow) detects it is time to process Bronze data
  It starts a Glue Spark job

Step 4: Glue Spark processes Bronze → Silver
  The Glue job reads the Bronze Parquet files
  For each record:
    - Validates schema and required fields
    - Resolves CDC operations (merge inserts/updates/deletes into current state)
    - If valid → writes to Silver bucket
    - If invalid → writes to Quarantine bucket

Step 5: Airflow triggers dbt
  After Glue completes, MWAA starts the dbt transformation

Step 6: dbt transforms Silver → Gold
  dbt runs SQL models using Athena as the query engine
  Silver data is aggregated into business-level summaries
  Results are written to the Gold bucket

Step 7: Redshift reads Gold
  Redshift Serverless uses Redshift Spectrum to query Gold S3 data
  as if it were internal Redshift tables (no data loading required)

Step 8: BI Dashboard connects to Redshift
  Tableau / Power BI / QuickSight connects to Redshift endpoint
  Analysts query Gold data and build dashboards

Step 9: Monitoring
  Every step writes logs to CloudWatch
  Alerts can be configured for pipeline failures
```

---

## Technology Stack

| Technology | What It Is | Role in This Platform |
|---|---|---|
| **Terraform** | Infrastructure-as-Code tool | Provisions all AWS resources reproducibly |
| **AWS S3** | Cloud object storage | Hosts the Bronze/Silver/Gold/Quarantine data lake |
| **PostgreSQL on RDS** | Relational database | The source system (simulates a real application DB) |
| **AWS DMS** | Database Migration Service | Captures CDC events from PostgreSQL and writes to Bronze |
| **AWS Glue** | Managed Apache Spark service | Runs PySpark jobs to transform Bronze → Silver |
| **Apache Spark (PySpark)** | Distributed data processing engine | The runtime inside Glue jobs |
| **dbt** | Data Build Tool | Runs SQL transformations Silver → Gold using Athena |
| **Amazon Athena** | Serverless SQL query engine | Executes dbt SQL models against S3 data |
| **Redshift Serverless** | Serverless data warehouse | Serves analytics queries via JDBC/ODBC |
| **Redshift Spectrum** | Redshift feature | Queries S3 Gold data as external tables |
| **Amazon MWAA** | Managed Apache Airflow | Orchestrates and schedules the pipeline |
| **Apache Airflow** | Workflow orchestration | DAGs define pipeline order and dependencies |
| **AWS KMS** | Key Management Service | Encrypts data at rest across all services |
| **AWS IAM** | Identity and Access Management | Controls who/what can access each service |
| **AWS Glue Catalog** | Metadata catalog | Stores table schemas for Bronze/Silver/Gold |
| **CloudWatch** | AWS monitoring service | Logs and metrics for all platform components |
| **VPC** | Virtual Private Cloud | Private network that isolates all compute |

---

## Repository Structure

This project is split across multiple repositories, each with a single responsibility:

```
enterprise-data-platform/
│
├── README.md                          ← You are here
│
├── terraform-bootstrap/               ← STEP 1: Run this first
│   Purpose: Creates remote Terraform state storage (S3 + DynamoDB)
│   for each AWS account before any other infrastructure is provisioned.
│
├── terraform-platform-infra-live/     ← STEP 2: Run this second
│   Purpose: Provisions all AWS infrastructure (VPC, S3 buckets, IAM,
│   RDS, DMS, Glue, Redshift, MWAA) across dev/staging/prod environments.
│
├── platform-glue-jobs/                ← STEP 3: Deploy after infra
│   Purpose: PySpark code that runs inside AWS Glue.
│   Handles Bronze → Silver transformation and quarantine logic.
│
├── platform-dbt-analytics/            ← STEP 4: Deploy after Glue
│   Purpose: dbt SQL models that transform Silver → Gold.
│   Uses Athena as the query engine.
│
├── platform-orchestration-mwaa-airflow/  ← STEP 5: Deploy after dbt
│   Purpose: Apache Airflow DAGs that orchestrate the full pipeline.
│   Controls when Glue jobs run and when dbt runs.
│
├── platform-cdc-simulator/            ← Used for testing
│   Purpose: Generates synthetic PostgreSQL change events
│   to test the pipeline end-to-end without a real application.
│
└── platform-docs/                     ← Documentation
    Purpose: Additional architecture docs, runbooks, and diagrams.
```

---

## AWS Account Strategy

This platform runs across **three separate AWS accounts**:

| Account | Purpose | AWS CLI Profile |
|---|---|---|
| **dev** | Development and testing. Safe to break things. | `dev-admin` |
| **staging** | Pre-production validation. Mirrors prod config. | `staging-admin` |
| **prod** | Live production. Protected. Limited access. | `prod-admin` |

Each account has its own:
- Terraform remote state bucket (from `terraform-bootstrap`)
- VPC and networking
- S3 data lake buckets
- IAM roles and KMS keys
- Glue, MWAA, Redshift environments

**Why separate accounts?**
Account-level isolation is the strongest security boundary in AWS. A misconfiguration in dev cannot affect prod. A cost spike in staging does not impact prod billing. This is standard enterprise practice.

---

## VPC CIDR Allocation

Each environment has its own non-overlapping IP range:

| Environment | VPC CIDR | Private Subnet A | Private Subnet B |
|---|---|---|---|
| dev | 10.10.0.0/16 | 10.10.16.0/20 | 10.10.32.0/20 |
| staging | 10.20.0.0/16 | 10.20.16.0/20 | 10.20.32.0/20 |
| prod | 10.30.0.0/16 | 10.30.16.0/20 | 10.30.32.0/20 |

Non-overlapping ranges allow future VPC peering between accounts if needed.

---

## Infrastructure Module Map

Inside `terraform-platform-infra-live`, the infrastructure is organized as reusable modules:

```
modules/
├── networking/       Creates VPC, subnets, route tables, S3 endpoint
├── data-lake/        Creates all 5 S3 medallion buckets
├── iam-metadata/     Creates KMS key, IAM roles, Glue Catalog databases
├── ingestion/        Creates RDS PostgreSQL + DMS for CDC
├── processing/       Creates Glue security config, Glue connection, Athena workgroup
├── serving/          Creates Redshift Serverless namespace + workgroup
└── orchestration/    Creates MWAA environment, DAGs bucket, CloudWatch log groups
```

Each environment (dev/staging/prod) calls all of these modules with environment-specific values.

The modules depend on each other in this order:

```
networking  ──────────────────────────────────────────────────────┐
                                                                   │
data-lake   ──────────────────────────────────────────────────────┤
                                                                   │
iam-metadata  (uses bucket names from data-lake) ─────────────────┤
                                                                   ├──► ingestion
ingestion   (uses vpc, subnets, kms, buckets)                      ├──► processing
                                                                   ├──► serving
processing  (uses vpc, subnets, kms, glue role, buckets)           └──► orchestration
serving     (uses vpc, subnets, kms, redshift role)
orchestration (uses vpc, subnets, kms, mwaa role)
```

---

## Prerequisites — What You Need Before Starting

### 1. AWS Accounts

You need three AWS accounts. In a real enterprise, these are created under an AWS Organization.
For learning, a single AWS account can be used by changing the environment name.

### 2. AWS CLI v2

```bash
aws --version
# Must show: aws-cli/2.x.x
```

Install from: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

### 3. Terraform

```bash
terraform --version
# Must show: Terraform v1.6.0 or higher
```

Install from: https://developer.hashicorp.com/terraform/install

### 4. AWS SSO Profiles

You need three CLI profiles configured (one per account):

```bash
# Configure dev profile
aws configure sso --profile dev-admin

# Configure staging profile
aws configure sso --profile staging-admin

# Configure prod profile
aws configure sso --profile prod-admin
```

See `terraform-bootstrap/README.md` for the full SSO setup walkthrough.

### 5. Python 3.9+

Required for local testing of Glue PySpark jobs.

```bash
python --version
```

### 6. dbt CLI

Required for running dbt models locally.

```bash
pip install dbt-athena-community
dbt --version
```

---

## Getting Started — Deployment Order

Follow this exact order. Each step depends on the previous one.

### Step 1 — Bootstrap Remote State

```bash
cd terraform-bootstrap
# Read: terraform-bootstrap/README.md for full instructions
aws sso login --profile dev-admin
cd environments/dev
terraform init
terraform apply
```

Repeat for staging and prod.

**What this creates:** S3 bucket + DynamoDB table in each account to store Terraform state remotely.

### Step 2 — Deploy Platform Infrastructure

```bash
cd terraform-platform-infra-live
# Read: terraform-platform-infra-live/README.md for full instructions
aws sso login --profile dev-admin

# Using the Makefile:
make init dev
make plan dev
make apply dev
```

The Makefile targets available are:

| Command | What it does |
|---|---|
| `make init dev` | Runs `terraform init` in environments/dev |
| `make plan dev` | Runs `terraform plan` in environments/dev |
| `make apply dev` | Runs `terraform apply` in environments/dev |
| `make destroy dev` | Runs `terraform destroy` in environments/dev |

Replace `dev` with `staging` or `prod` for other environments.

**What this creates:** VPC, all 5 S3 buckets, KMS key, all IAM roles, RDS, DMS, Glue config, Redshift, MWAA.

### Step 3 — Deploy Glue Jobs

```bash
cd platform-glue-jobs
# Coming soon — see platform-glue-jobs/README.md
```

### Step 4 — Deploy dbt Models

```bash
cd platform-dbt-analytics
# Coming soon — see platform-dbt-analytics/README.md
```

### Step 5 — Deploy Airflow DAGs

```bash
cd platform-orchestration-mwaa-airflow
# Coming soon — see platform-orchestration-mwaa-airflow/README.md
```

---

## Key Concepts for Beginners

### What is CDC (Change Data Capture)?

CDC is a technique for tracking every change (insert, update, delete) made to a database table.

Instead of copying the entire table every time (expensive), CDC only captures what changed.

AWS DMS (Database Migration Service) connects to PostgreSQL and reads its Write-Ahead Log (WAL) — an internal PostgreSQL journal of every change — and converts those entries into files that land in S3.

### What is a DAG (Directed Acyclic Graph)?

A DAG is how Airflow represents a pipeline. It is a set of tasks where each task depends on the previous one completing successfully.

Example DAG:
```
start_glue_job → wait_for_glue → trigger_dbt → verify_gold → notify_success
```

Airflow executes each task in order and retries failed tasks automatically.

### What is Terraform State?

When Terraform creates resources (like an S3 bucket), it stores a record of what it created in a file called `terraform.tfstate`.

This file is critical — it is how Terraform knows what already exists so it does not create duplicates.

We store this file in S3 (remote state) so that the whole team shares the same view of infrastructure and concurrent applies are prevented by DynamoDB locking.

### What is a Terraform Module?

A module is a reusable block of Terraform code. Instead of copying the same S3 bucket configuration for dev, staging, and prod, we write it once in a module and call it three times with different inputs.

```hcl
module "data_lake" {
  source      = "../../modules/data-lake"
  environment = "dev"          # This is the only thing that changes
}
```

### What is Parquet?

Parquet is a column-oriented file format designed for analytical workloads.

Instead of storing data row-by-row (like a CSV), Parquet stores data column-by-column. This means a query like `SELECT revenue FROM orders` only reads the `revenue` column, skipping all other columns. This is dramatically faster and cheaper for analytics.

All data in this platform is stored as Parquet (compressed with Snappy or GZIP).

---

## Security Architecture

Every component in this platform follows the principle of **least privilege** — each service can only access exactly what it needs, nothing more.

| Security Control | Where Applied |
|---|---|
| KMS encryption at rest | All S3 buckets, RDS, DMS |
| IAM roles (not users) | Glue, MWAA, Redshift, DMS each have dedicated roles |
| VPC private subnets | All compute runs in private subnets with no internet access |
| S3 public access blocked | All 5 data lake buckets block all public access |
| S3 versioning enabled | All buckets retain previous versions for recovery |
| RDS deletion protection | Enabled in staging and prod |
| Terraform prevent_destroy | State buckets protected from accidental deletion |
| No static AWS access keys | All authentication uses temporary SSO credentials |

---

## Cost Considerations

This platform uses serverless and pay-per-use services wherever possible:

| Service | Cost Model |
|---|---|
| Redshift Serverless | Pay per RPU-hour (scales to zero when idle) |
| AWS Glue | Pay per DPU-hour (only when jobs run) |
| Amazon Athena | Pay per TB of data scanned |
| Amazon MWAA | Pay per environment-hour (always on when active) |
| AWS DMS | Pay per replication instance-hour |
| Amazon S3 | Pay per GB stored + requests |

**For development:** Use `make destroy dev` after each session to avoid costs on RDS and MWAA (which are always-on services).

---

## Naming Convention

All resources follow this pattern:

```
{name_prefix}-{environment}-{account_id}-{resource_type}
```

Example S3 bucket: `edp-dev-123456789012-bronze`

The account ID is included in bucket names because S3 bucket names are globally unique across all AWS accounts. Including the account ID prevents naming conflicts.

IAM roles and other non-globally-unique resources follow:

```
{name_prefix}-{environment}-{role_name}
```

Example: `edp-dev-glue-role`

---

## What Is Built vs What Is Planned

| Component | Status |
|---|---|
| Terraform remote state (bootstrap) | **Complete** |
| VPC and networking | **Complete** |
| S3 medallion data lake (all 5 buckets) | **Complete** |
| KMS key, IAM roles, Glue Catalog | **In Progress** |
| RDS PostgreSQL + DMS CDC pipeline | **In Progress** |
| Glue security config + Athena workgroup | **In Progress** |
| Redshift Serverless | **In Progress** |
| MWAA (Airflow) environment | **In Progress** |
| Glue PySpark jobs (Bronze → Silver) | **Planned** |
| dbt models (Silver → Gold) | **Planned** |
| Airflow DAGs | **Planned** |
| CDC Simulator | **Planned** |

---

## Who Maintains This

This platform is maintained as an enterprise data engineering reference implementation.

All infrastructure is managed exclusively through Terraform. No resources are created manually in the AWS Console.

If you find something that was created manually, bring it under Terraform control using `terraform import`.

---

## Claude Code — Authentication Reference

This project is developed using Claude Code as the AI engineering assistant. This section documents how authentication works so you can resume work correctly after a session limit or restart.

---

### The Two Authentication Modes

Claude Code supports two distinct modes. Which one you are in determines your limits and how you are billed.

| Mode | How it works | Session limits | Billing |
|---|---|---|---|
| **Claude Pro** | Logged in with your Anthropic account | Yes — resets on a rolling window | Covered by subscription |
| **API Key** | Authenticated with an API key | None | Pay per token |

When working on intensive sessions (long Terraform plans, full repository audits, multi-file builds), the Pro session limit can be reached quickly. Switching to API key mode removes the session cap entirely.

---

### How to Tell Which Mode You Are In

Check the banner at the top of the Claude Code session:

```
Sonnet 4.6 · Claude Pro · your-email@...        ← Pro mode (subscription limits apply)
Sonnet 4.6 · API Usage Billing                  ← API key mode (no session cap)
```

Or run from your shell at any time:

```bash
claude auth status
```

---

### Pro Mode — Normal Login

When you launch Claude Code with a standard account login:

```bash
claude
```

Claude Code shows the Pro banner and consumes your subscription quota. When the session usage reaches 100%, Claude stops responding until the window resets.

To log in with your account:

```bash
claude auth login
```

---

### Switching to API Key Mode

Use this when you have hit the Pro session limit or want uninterrupted long sessions.

**Step 1 — Exit the current session**

Press `Ctrl+C` inside Claude, or close the terminal entirely if it is unresponsive.

**Step 2 — Log out of Pro**

```bash
claude auth logout
```

**Step 3 — Export your API key**

Generate a key from the Anthropic Console if you do not have one. Then export it into your shell:

```bash
export ANTHROPIC_API_KEY="sk-ant-xxxxxxxxxxxxxxxx"
```

To confirm the key is active in your shell:

```bash
echo $ANTHROPIC_API_KEY
```

**Step 4 — Launch Claude Code**

```bash
claude
```

**Step 5 — Log in inside the session**

```bash
/login
```

Select API key authentication when prompted. The banner will change to:

```
Sonnet 4.6 · API Usage Billing
```

You are now in API mode. No session cap. Billing is per token until your API credit runs out.

---

### Switching Back to Pro Mode

```bash
# Inside the Claude session
/logout

# Then from your shell
claude auth logout
claude auth login
claude
```

The banner will return to `Claude Pro · your-email@...`.

---

### Making the API Key Permanent

`export` only sets the variable for the current terminal session. If you open a new terminal it will be gone. To make it permanent, add it to your shell profile:

```bash
# For zsh (default on macOS)
echo 'export ANTHROPIC_API_KEY="sk-ant-xxxxxxxxxxxxxxxx"' >> ~/.zshrc
source ~/.zshrc
```

---

### API Key Security Rules

- **Never paste an API key into a chat, screenshot, or document.** If you do, revoke it immediately from the Anthropic Console and generate a new one.
- **Never commit an API key to Git.** Add `.env` and any file containing `ANTHROPIC_API_KEY` to `.gitignore`.
- **Revoke and rotate keys regularly.** Treat them like passwords.

If a key is exposed:

1. Go to the Anthropic Console → API Keys
2. Revoke the exposed key immediately
3. Generate a new key
4. Export the new key in your shell

---

### Quick Reference

```bash
# Check current auth mode
claude auth status

# Start in Pro mode
claude auth login && claude

# Start in API key mode
export ANTHROPIC_API_KEY="sk-ant-..."
claude
# then inside: /login → select API key

# Switch modes mid-project
/logout              # inside Claude session
claude auth logout   # from shell
```

---

*Read each repository's own README.md for module-level details and step-by-step deployment instructions.*
