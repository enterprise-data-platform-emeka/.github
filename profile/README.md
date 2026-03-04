# Enterprise Data Platform (EDP)

I built this project to learn and demonstrate production-grade data engineering on AWS. The idea is to take raw data from a database, move it through a series of cleaning and transformation steps, and make it available to business analysts through a dashboard - the same way it works inside real companies.

This is not a simplified demo. I wanted to build the actual thing that data engineers build at work, which means it uses Terraform for infrastructure, multiple AWS services, distributed data processing with Spark, SQL transformations with dbt, orchestration with Airflow, and an AI agent to monitor and recover the pipeline when things go wrong.

I know that sounds like a lot. The whole point of this README is to explain everything clearly and show the logical order to build it so it stops feeling overwhelming.

---

## What this platform does

My data source is a PostgreSQL database. Think of it like a live application database where records are being inserted, updated, and deleted constantly throughout the day.

The problem with using that database directly for analytics is that it was not designed for it. Running heavy queries against a production database slows the application down, and the raw data is messy and hard to work with.

This platform fixes that by doing the following:

1. **Capturing every database change** in real time using a technique called CDC (Change Data Capture), so inserts, updates, and deletes are all tracked
2. **Storing the raw captured data** into an immutable Bronze layer in S3, so the original data is never modified or lost
3. **Cleaning and validating** the raw data with Apache Spark, turning it into structured records in a Silver layer
4. **Sending any bad records** to a Quarantine area instead of silently dropping them, so data quality problems can be diagnosed
5. **Aggregating the clean data** into business-level summaries in a Gold layer using SQL models written in dbt
6. **Making the Gold data available** through Redshift Serverless so BI tools like Tableau, Power BI, or QuickSight can connect and build dashboards
7. **Running the whole pipeline automatically** on a schedule using Apache Airflow
8. **Logging everything** to CloudWatch so I can see what is happening at any point
9. **Monitoring the pipeline with an AI agent** that watches every layer at once, figures out the real cause of failures, and automatically recovers the pipeline when common problems occur

---

## Architecture diagram

```
                ┌──────────────────────────────────────────────────────────────────────────────────────────┐
                │                                    CONTROL PLANE                                         │
                │                                                                                         │
                │  ┌──────────────────────────┐        ┌──────────────────────────┐                      │
                │  │   MWAA Orchestration      │───────►│   CloudWatch Logs        │                      │
                │  │   (Apache Airflow)        │        │   & EventBridge          │                      │
                │  └────────────┬──────────────┘        └─────────────┬────────────┘                      │
                │               │                                     │ events stream down                │
                │               │                  ┌──────────────────┘                                  │
                │               │                  │                                                      │
                │               │   ┌──────────────▼──────────────────────────────────────────────────┐  │
                │               │   │                    AI OPERATIONS AGENT                           │  │
                │               │   │  Ingests CloudWatch events from every layer in real time         │  │
                │               │   │  Reasons across DMS, Glue, Athena, and Redshift together         │  │
                │               │   │  Re-triggers Airflow DAGs with correct backfill arguments        │  │
                │               │   │  Routes bad data batches directly to the Quarantine bucket       │  │
                │               │   │  Sends plain-English incident reports via SNS / Slack            │  │
                │               │   └──────────────────────────────────────────────────────────────────┘  │
                └───────────────┼─────────────────────────────────────────────────────────────────────────┘
                                │ triggers jobs
                                ▼
        ┌───────────────────────────────────────────────────────────────────────────────────┐
        │                                PROCESSING LAYER                                   │
        │                                                                                   │
        │  ┌────────────────────────────────────────┐   ┌────────────────────────────────┐ │
        │  │  Glue Spark Job                        │   │  dbt using Athena              │ │
        │  │  Validation & Reconstruction           │   │  SQL transformations           │ │
        │  └────────────────────────────────────────┘   └────────────────────────────────┘ │
        └───────────────────────────────────────────────────────────────────────────────────┘
                    │ reads Bronze                              │ reads Silver
                    │ writes Silver / Quarantine                │ writes Gold
                    ▼                                           ▼
┌──────────┐  ┌─────────────────────────────────────────────────────────────────────────────┐  ┌──────────────────────────────────┐
│  SOURCE  │  │                              S3 DATA LAKE                                    │  │         SERVING LAYER            │
│  LAYER   │  │                                                                             │  │                                  │
│          │  │  ┌──────────────────┐  ┌─────────────────────────┐  ┌──────────────────┐   │  │  ┌───────────────────────────┐   │
│PostgreSQL│  │  │     BRONZE       │  │         SILVER           │  │      GOLD         │   │  │  │  Redshift Serverless      │   │
│    │     │  │  │   (Immutable)    │─►│  (Reconstructed State)   │─►│  (Business Aggr.) │───┼─►│  │  with Spectrum            │   │
│    ▼     │  │  │   Raw CDC data   │  │                         │  │                  │   │  │  └─────────────┬─────────────┘   │
│  CDC     │─►│  └────────┬─────────┘  └─────────────────────────┘  └──────────────────┘   │  │                │                 │
│ (DMS)    │  │           │                                                                 │  │                ▼                 │
└──────────┘  │           │         ┌───────────────────────────┐                           │  │          BI Dashboard            │
              │           └────────►│        QUARANTINE          │                           │  └──────────────────────────────────┘
              │                     │     (Invalid Records)      │                           │
              │                     └───────────────────────────┘                           │
              └─────────────────────────────────────────────────────────────────────────────┘
```

---

## How I'm building this from start to finish

When I first looked at everything this platform needs, it felt like too much to take on at once. There are infrastructure resources to create, code to write, services to connect, and tools I had never used before all at the same time.

The thing that made it manageable was realizing there is a strict logical order to it. You cannot fill a bucket that does not exist yet. You cannot run a Spark job against a database that has not been created. Every piece has a prerequisite, and once I mapped the order out, it became a straightforward checklist instead of a pile of complexity.

I split the work into two phases. Phase 1 is all infrastructure. Phase 2 is all application code on top of that infrastructure.

---

### Phase 1: Infrastructure with Terraform

Terraform is the tool I use to create all the AWS resources. Instead of clicking through the AWS console, I write code that describes what I want, and Terraform creates it. This means I can destroy everything at the end of a session to save money and recreate it exactly the same way next time.

Everything in Phase 1 lives in the `terraform-platform-infra-live` directory, organized as modules. Each module has one job.

**Step 1 - Remote state storage**

Before Terraform can work reliably, it needs a place to store a file called `terraform.tfstate`. This file is how Terraform tracks what it has already created so it does not try to create things twice.

I store this file in S3 with a DynamoDB table handling the locking, so it is safe and consistent. This setup lives in `terraform-bootstrap` and runs once per AWS account before anything else. It is the only thing that has to be done before the main infrastructure.

**Step 2 - Networking**

I create a VPC, which is a private network inside AWS that all my resources will live in. Along with the VPC, I create subnets, route tables, and an S3 VPC endpoint. The S3 endpoint is important because it lets services inside the VPC talk to S3 without going out to the public internet.

Nothing else can be created without a network to put it in, so this always comes first.

**Step 3 - Data lake storage**

I create the five S3 buckets: Bronze, Silver, Gold, Quarantine, and a fifth one for Athena query results. These are just empty buckets at this point. No data is in them yet. But they need to exist before any of the services that write to them can be set up.

**Step 4 - Permissions and metadata**

I create a KMS encryption key that will encrypt all data at rest across the platform. I also create the IAM roles for every service that needs one: Glue, Airflow, Redshift, DMS, and the ECS task for the AI agent. Each role only has permission to do exactly what that service needs.

I also create three Glue Data Catalog databases here, one for Bronze, one for Silver, and one for Gold. These are metadata containers that describe the schema of the data in each layer. Glue and Athena both use these to understand the structure of the data they are reading.

**Step 5 - Data source and CDC**

I create the RDS PostgreSQL database that acts as my source system. This is the database that the CDC process will read changes from.

I also create the DMS replication instance and replication task. DMS connects to PostgreSQL, reads its write-ahead log (the internal journal PostgreSQL keeps of every change), and converts those changes into Parquet files that land in the Bronze S3 bucket. This is where data ingestion begins.

One important thing: after the first Terraform apply, the RDS instance needs a reboot to activate logical replication mode. This is a one-time manual step. The DMS task also needs to be started manually after the infrastructure is applied. Both of these are documented in the ingestion module README.

**Step 6 - Processing configuration**

I create the Glue security configuration, which controls how Glue encrypts data while it is working, and the Athena workgroup, which controls where Athena writes query results and how much data queries are allowed to scan.

These are settings containers. The actual Glue PySpark code and the dbt SQL models come in Phase 2. At this point I'm just setting up the environment they will run in.

**Step 7 - Serving layer**

I create the Redshift Serverless namespace and workgroup. This is the data warehouse that analysts will eventually query. At this point it is empty and has no data in it, but it exists and it can read directly from the Gold S3 bucket using Redshift Spectrum once data starts flowing.

**Step 8 - Orchestration**

I create the MWAA environment, which is the managed Airflow service on AWS. Airflow is what runs the pipeline on a schedule and controls the order that things run in. I also create the ECS cluster and task definition for the AI Operations Agent.

One note on MWAA: it costs about $0.49 per hour, which adds up quickly. I only spin this up when I need to test orchestration or record a demo. For day-to-day development of DAGs, I run Airflow locally.

Once all eight infrastructure steps are done, everything is in place. The VPC exists, the buckets exist, the database exists, DMS is set up, the processing environment is configured, Redshift is ready, and Airflow is running. No data is flowing yet but the whole scaffolding is there.

---

### Phase 2: Application code

This is where I write the code that actually moves and transforms data. The infrastructure from Phase 1 is the foundation. Everything in Phase 2 runs on top of it.

**Step 9 - PySpark transformation jobs (platform-glue-jobs)**

I write the PySpark code that runs inside AWS Glue. This job reads Parquet files from Bronze, validates each record against the expected schema, applies the CDC operations correctly (so a record that was inserted and then deleted ends up being deleted in Silver, not duplicated), and writes the clean result to Silver. Any record that fails schema validation or has data quality issues gets written to Quarantine instead.

This is the Bronze to Silver transformation. It is the most complex piece of code in the project because it has to handle all the different CDC operation types correctly.

**Step 10 - SQL transformation models (platform-dbt-analytics)**

I write dbt models, which are just SQL files with some extra structure. These read from Silver through Athena and produce business-level aggregations that go into Gold. Things like daily revenue by product, monthly active users, weekly order volumes.

dbt handles dependency ordering between models automatically, so if one model depends on another, dbt figures out which one to run first. It also generates documentation and can run data quality tests on the output.

**Step 11 - Airflow DAGs (platform-orchestration-mwaa-airflow)**

I write the DAGs that orchestrate the whole sequence. A DAG is a Python file that defines a set of tasks and the order they run in. My main pipeline DAG looks roughly like this:

```
start_dms_check -> trigger_glue_job -> wait_for_glue -> trigger_dbt -> verify_gold -> notify_success
```

If any task fails, the DAG stops and an alert fires. Airflow handles retries, scheduling, and the task dependency graph automatically.

**Step 12 - CDC simulator (platform-cdc-simulator)**

I write a Python script that generates realistic PostgreSQL changes: inserts of new orders, updates to existing records, and deletes. This lets me test the full pipeline without needing a real application connected to the database. I can control the volume and type of changes to test edge cases.

**Step 13 - AI Operations Agent (platform-ops-agent)**

I build the Python application that monitors the pipeline and handles incident recovery. I cover this in its own section below.

---

### Phase 3: End-to-end validation

**Step 14 - Full pipeline test in dev**

I run the CDC simulator to generate data, trigger the Airflow DAG, and watch data move from PostgreSQL all the way to a queryable result in Redshift. When this works end to end, the project is functionally complete.

**Step 15 - Staging and prod confirmation**

I deploy to staging and then prod just to confirm the Terraform modules work correctly in those environments. I destroy both immediately after confirming. Active development only happens in dev.

---

That is the full build. Thirteen steps across two phases, plus a final validation. Written out it looks like a lot but each step is self-contained, has a clear purpose, and follows directly from the previous one. The reason I wrote it this way is because looking at the architecture diagram without this ordering made the whole thing feel like a wall. With the ordering, it becomes a checklist.

---

## The AI Operations Agent

When data moves through five separate services (DMS, Glue, dbt, Redshift, Airflow) and something goes wrong, the failure often shows up in a different place than where it started. Here is a real example of what that looks like:

```
DMS replication lag spikes
  and Glue reads empty Bronze files
    and Silver never gets updated
      and dbt runs on stale Silver data
        and Gold aggregations are wrong
          and the dashboard shows incorrect numbers
```

CloudWatch fires an alarm in DMS. Airflow retries the Glue task. But neither of them knows that the Glue failure and the DMS alarm are related. An engineer looking at these alerts in isolation would probably start debugging Glue first, which is the wrong place.

The AI Operations Agent solves this. It watches every layer at once and reasons across all of them before deciding what to do.

### How it works

The agent is a Python application running as an ECS Fargate container. It subscribes to EventBridge rules that fire whenever something notable happens in DMS, Glue, Athena, Airflow, or Redshift.

When an event fires, the agent does not just react to that single event. It calls the AWS SDK across multiple services at the same time to get the full picture before taking any action.

Here is what it does when a Glue job failure event arrives:

```
Event received: Glue job failed

Agent checks simultaneously:
  - DMS: what is the current replication lag?
  - S3: how many files actually landed in Bronze for this time window?
  - CloudWatch: has this happened before in the last 24 hours?
  - Glue: what did the last 5 job runs look like?

Conclusion: Bronze had zero files because DMS paused 47 minutes ago.
            The Glue job did not fail because of a code bug.
            The root cause is a DMS health issue.

Action: Pause downstream Glue jobs to stop them running pointlessly.
        Alert specifically on DMS, not Glue.
        Schedule a Glue retry for when DMS recovers.
```

This cross-service reasoning is what makes the agent useful. No individual AWS alarm or Airflow retry can do this because they each only see one part of the picture.

### What the agent does in each scenario

| Situation | What the agent does |
|---|---|
| DMS lag is the root cause | Pauses downstream Glue jobs to prevent empty runs |
| A bad Bronze file caused Glue to fail | Moves the bad file to Quarantine, re-triggers Glue on the rest |
| dbt ran on incomplete Silver data | Delays dbt, schedules a backfill once Silver catches up |
| Transient network error in Glue | Triggers a single retry through the Airflow REST API |
| Unknown failure pattern | Escalates with full context, does not attempt auto-recovery |

Every incident produces a plain-English report sent via SNS:

```
INCIDENT REPORT - 2024-03-15 02:17 UTC
Root cause: DMS task edp-dev-replication paused (lag: 47 min)
Impact: Bronze partition 2024-03-15/orders has 0 files
Action taken: Glue job paused until DMS recovers
Action taken: Glue retry scheduled for 03:00 UTC
Manual action needed: Check DMS task - may need RDS WAL retention increase
Confidence: HIGH
```

### What the agent does not do

The agent is deliberately limited to operations and recovery. It does not modify Terraform infrastructure, change Glue code or dbt models, or attempt auto-recovery on anything it cannot explain with high confidence. Everything it does is logged with a clear reason.

---

## The data lake layers (Medallion Architecture)

The data lake follows a pattern called Medallion Architecture. Each layer is a separate S3 bucket and has a specific job.

### Bronze: raw and immutable

Everything that comes out of DMS lands here exactly as it arrived. I never modify Bronze data. If I find a bug in my Glue transformation code six months from now, I can re-process everything from Bronze without losing any original data. Bronze is the source of truth for the whole platform.

Data is stored as Parquet files, partitioned by date with one folder per source table.

### Silver: clean and structured

This is the output of the Glue PySpark job. It reads Bronze, validates each record, applies CDC operations correctly so the current state of each record is accurate, and writes the result here. Records that fail validation go to Quarantine instead. Silver represents the current real state of the source database.

### Gold: business-ready aggregations

This is what analysts actually query. dbt reads Silver through Athena and runs SQL models that produce things like daily revenue by product, monthly active users, and weekly order volumes. Gold is pre-aggregated so dashboards load fast and analysts are not running expensive queries against raw data.

### Quarantine: the bad records

Any record that fails Silver validation ends up here instead of being silently dropped. Silent data loss is much worse than visible data quality problems. With Quarantine I can go back, see exactly what failed, and fix the problem at the source. The AI agent can also route records here directly during incident recovery.

### Athena results

Athena writes query result files here whenever it runs a SQL query. This is just a designated output location for the Athena workgroup.

---

## How data moves through the system

```
Step 1:  A change happens in the PostgreSQL database (insert, update, or delete)

Step 2:  AWS DMS reads that change from PostgreSQL's internal write-ahead log
         and writes it as a Parquet file to the Bronze S3 bucket
         Each file contains the row data, the operation type, and a timestamp

Step 3:  Airflow detects it is time to run the pipeline and triggers the Glue job

Step 4:  The Glue PySpark job reads Bronze
         Each record is validated:
           - If valid, it is written to Silver
           - If invalid, it is written to Quarantine

Step 5:  Airflow detects the Glue job finished and triggers dbt

Step 6:  dbt runs SQL models using Athena as the query engine
         Silver data is aggregated into Gold summaries

Step 7:  Redshift Serverless uses Spectrum to read Gold directly from S3
         No data loading required - Spectrum queries S3 as if it were a Redshift table

Step 8:  BI tools connect to Redshift and analysts build dashboards on Gold data

Step 9:  Every step above writes logs to CloudWatch
         EventBridge converts log patterns into structured events

Step 10: The AI Operations Agent receives all EventBridge events in real time
         For any failure it diagnoses the root cause, takes the right action,
         and sends a plain-English incident report via SNS
         This runs continuously alongside all other steps
```

---

## Tools and technologies

| Tool | What it is | How I use it |
|---|---|---|
| Terraform | Infrastructure-as-Code | Creates all AWS resources from code |
| AWS S3 | Cloud object storage | Holds all data lake layers |
| PostgreSQL on RDS | Relational database | The data source |
| AWS DMS | Database Migration Service | Captures CDC events and writes to Bronze |
| AWS Glue | Managed Spark service | Runs PySpark jobs for Bronze to Silver |
| Apache Spark (PySpark) | Distributed processing engine | The runtime inside Glue |
| dbt | Data Build Tool | SQL models for Silver to Gold |
| Amazon Athena | Serverless SQL engine | Executes dbt SQL against S3 data |
| Redshift Serverless | Serverless data warehouse | Serves analyst queries |
| Redshift Spectrum | Redshift feature | Queries Gold S3 as external tables |
| Amazon MWAA | Managed Airflow | Orchestrates and schedules the pipeline |
| Apache Airflow | Workflow engine | DAGs define task order and dependencies |
| AWS KMS | Key Management Service | Encrypts all data at rest |
| AWS IAM | Identity and Access Management | Controls what each service is allowed to do |
| AWS Glue Catalog | Metadata catalog | Stores table schemas for Bronze, Silver, Gold |
| CloudWatch and EventBridge | Monitoring | Logs and structured events from every layer |
| ECS Fargate | Serverless containers | Runs the AI Operations Agent |
| Claude (Anthropic) | Large language model | Powers the agent's cross-service reasoning |
| VPC | Virtual Private Cloud | Isolates all compute in a private network |

---

## Repository layout

```
enterprise-data-platform/
│
├── README.md                              (this file)
│
├── terraform-bootstrap/                   BUILD STEP 1
│   Creates S3 buckets and DynamoDB tables for Terraform remote state.
│   Run this once per AWS account before anything else.
│
├── terraform-platform-infra-live/         BUILD STEPS 2 to 8
│   All AWS infrastructure, organized as Terraform modules.
│   VPC, S3 buckets, RDS, DMS, Glue, Redshift, MWAA, and ECS all live here.
│
├── platform-glue-jobs/                    BUILD STEP 9
│   PySpark code that runs inside AWS Glue.
│   Handles Bronze to Silver transformation and quarantine routing.
│
├── platform-dbt-analytics/                BUILD STEP 10
│   dbt SQL models that transform Silver to Gold.
│   Uses Athena as the query engine.
│
├── platform-orchestration-mwaa-airflow/   BUILD STEP 11
│   Airflow DAGs that orchestrate the full pipeline.
│   Controls when Glue runs, when dbt runs, and what happens on failure.
│
├── platform-cdc-simulator/                BUILD STEP 12
│   Script that generates synthetic PostgreSQL changes for testing.
│
├── platform-ops-agent/                    BUILD STEP 13
│   The AI Operations Agent.
│   Python app that monitors the pipeline and recovers from failures.
│
└── platform-docs/
    Additional diagrams, runbooks, and architecture notes.
```

---

## AWS accounts

I run this across three separate AWS accounts:

| Account | Purpose | CLI profile |
|---|---|---|
| dev | Where I build and test. Safe to break. | dev-admin |
| staging | Used only to confirm staging deploys correctly, then destroyed | staging-admin |
| prod | Used only to confirm prod deploys correctly, then destroyed | prod-admin |

Separate accounts are the strongest isolation AWS offers. A mistake in dev cannot touch prod. Unexpected costs in staging do not affect prod billing.

My active development only happens in dev. I spin up staging and prod occasionally to confirm the Terraform modules deploy cleanly, then destroy them to keep costs near zero.

---

## Network layout

Each environment gets its own IP address range so they never overlap:

| Environment | VPC CIDR | Private Subnet A | Private Subnet B |
|---|---|---|---|
| dev | 10.10.0.0/16 | 10.10.16.0/20 | 10.10.32.0/20 |
| staging | 10.20.0.0/16 | 10.20.16.0/20 | 10.20.32.0/20 |
| prod | 10.30.0.0/16 | 10.30.16.0/20 | 10.30.32.0/20 |

Non-overlapping ranges mean I can peer these networks together in the future if needed.

---

## Terraform module map

Inside `terraform-platform-infra-live`, the infrastructure is split into modules. Each module has one responsibility and can be understood on its own.

```
modules/
├── networking/     VPC, subnets, route tables, S3 VPC endpoint
├── data-lake/      All 5 S3 data lake buckets
├── iam-metadata/   KMS key, IAM roles, Glue Catalog databases
├── ingestion/      RDS PostgreSQL and DMS replication instance and task
├── processing/     Glue security config, Glue VPC connection, Athena workgroup
├── serving/        Redshift Serverless namespace and workgroup
└── orchestration/  MWAA environment, ECS cluster, CloudWatch log groups
```

The modules depend on each other in this order. Each one uses outputs from the ones above it:

```
networking
  └── data-lake
        └── iam-metadata
              ├── ingestion
              ├── processing
              ├── serving
              └── orchestration
```

---

## What I need to run this

- Three AWS accounts (or just one if learning, using dev environment only)
- AWS CLI v2: `aws --version` should show 2.x
- Terraform 1.6 or higher: `terraform --version`
- AWS SSO profiles configured for dev-admin, staging-admin, and prod-admin
- Python 3.9 or higher: `python --version`
- dbt CLI: `pip install dbt-athena-community`
- Anthropic API key for the AI agent, stored as the `ANTHROPIC_API_KEY` environment variable

For the full SSO setup walkthrough, see `terraform-bootstrap/README.md`.

---

## Running the infrastructure

Follow the build steps in order. Each depends on everything before it.

```bash
# Step 1: Set up remote state (run once per account)
cd terraform-bootstrap/environments/dev
terraform init
terraform apply

# Steps 2 to 8: Deploy all infrastructure
cd terraform-platform-infra-live
make init dev
make plan dev
make apply dev

# Destroy dev when done to avoid unnecessary costs
make destroy dev
```

Available make commands:

| Command | What it runs |
|---|---|
| `make init dev` | terraform init in environments/dev |
| `make plan dev` | terraform plan in environments/dev |
| `make apply dev` | terraform apply in environments/dev |
| `make destroy dev` | terraform destroy in environments/dev |

Replace `dev` with `staging` or `prod` for those environments.

Steps 9 through 13 have their own deployment instructions in each repository's README.

---

## Concepts explained simply

### What is CDC (Change Data Capture)?

CDC is a way of tracking every change that happens to a database without copying the whole thing each time.

PostgreSQL keeps an internal journal called the write-ahead log that records every insert, update, and delete before it is applied. AWS DMS reads this journal and converts those entries into files. This is much cheaper than dumping the whole database repeatedly and it captures changes in near real time.

### What is a DAG?

A DAG is how Airflow represents a pipeline. DAG stands for Directed Acyclic Graph, which just means a set of tasks where each task depends on the ones before it, and there are no circular dependencies.

My main pipeline DAG looks like this:

```
check_dms_health -> trigger_glue -> wait_for_glue -> trigger_dbt -> verify_gold -> notify_success
```

Airflow runs each task in order, handles retries on failure, and lets me see the status of every run in a web UI.

### What is Terraform state?

When Terraform creates an S3 bucket, it writes a record of that to a file called `terraform.tfstate`. This file is how Terraform knows what already exists so it does not try to create the same bucket again next time.

I store this file in S3 (remote state) so it is shared across sessions and protected. A DynamoDB table prevents two Terraform runs from happening at the same time and corrupting the file.

### What is a Terraform module?

A module is reusable Terraform code. Instead of writing the same S3 bucket configuration three times for dev, staging, and prod, I write it once in a module and call it three times with different inputs.

```hcl
module "data_lake" {
  source      = "../../modules/data-lake"
  environment = "dev"
}
```

### What is Parquet?

Parquet is a file format designed for analytics. Unlike a CSV which stores data row by row, Parquet stores data column by column.

This matters because analytics queries typically read only a few columns from large tables. A query like `SELECT revenue FROM orders` only needs the revenue column. With Parquet, that query only reads the revenue column and skips everything else. This makes queries much faster and much cheaper on Athena, which charges per byte scanned.

All data in this platform is stored as Parquet, compressed with Snappy or GZIP.

### What is an AI agent?

An AI agent is a program that uses a language model to make decisions, not just to generate text.

In a normal program, I would write explicit if/else logic for every possible failure scenario. But the number of ways a five-hop data pipeline can fail is too large to cover exhaustively. New failure patterns appear that were never anticipated.

An agent lets me describe a situation in natural language, have the model reason about it, and then execute whatever action the model recommends. This is useful for cross-service diagnosis because the model can reason about the relationship between a DMS lag event and a Glue failure in the same way a senior engineer would, without me having to write code for every possible combination.

---

## Security

Every service in this platform follows least privilege, meaning each one can only access exactly what it needs and nothing more.

| Control | Where it applies |
|---|---|
| KMS encryption at rest | All S3 buckets, RDS, DMS |
| Dedicated IAM roles | Glue, MWAA, Redshift, DMS, and ECS each have their own role |
| Private subnets only | All compute runs with no direct internet access |
| S3 public access blocked | All five data lake buckets |
| S3 versioning enabled | All buckets keep previous file versions for recovery |
| RDS deletion protection | Enabled in staging and prod |
| Terraform prevent_destroy | State buckets cannot be accidentally deleted |
| No static access keys | All authentication uses temporary SSO credentials |
| AI agent scope limited | Can only write to Quarantine S3 and call the Airflow API |

---

## Costs

I keep costs low by destroying dev at the end of each session. The two most expensive always-on services are MWAA at about $0.49 per hour and RDS. When I am not actively working, those are gone.

For local DAG development I run Airflow on my machine instead of in MWAA. I only spin MWAA up when I need to test it in the cloud or record a demo.

A typical 3 to 4 hour dev session without MWAA costs under $0.50.

Staging and prod are spun up occasionally to confirm deployments, then destroyed. Each validation run costs under $5.

| Service | How it charges |
|---|---|
| Redshift Serverless | Per RPU-hour, scales to zero when not in use |
| AWS Glue | Per DPU-hour, only charged when jobs are running |
| Athena | Per TB of data scanned |
| MWAA | Per environment-hour while it is running |
| DMS | Per replication instance-hour |
| S3 | Per GB stored plus per request |
| ECS Fargate (AI agent) | Per vCPU and memory-hour while running |
| Claude API (AI agent) | Per token, only charged during incident analysis |

---

## Naming convention

S3 bucket names include the AWS account ID because S3 names are globally unique across all accounts. Including the account ID prevents name collisions:

```
edp-dev-123456789012-bronze
edp-dev-123456789012-silver
edp-dev-123456789012-gold
```

Everything else uses a shorter pattern:

```
edp-dev-glue-role
edp-dev-dms-replication-instance
```

The pattern is always: prefix, environment, then the resource name.

---

## Build status

| Component | Status |
|---|---|
| Terraform remote state (bootstrap) | Done |
| VPC and networking | Done |
| S3 data lake (all 5 buckets) | Done |
| KMS key, IAM roles, Glue Catalog | In progress |
| RDS PostgreSQL and DMS CDC | In progress |
| Glue config and Athena workgroup | In progress |
| Redshift Serverless | Planned |
| MWAA and ECS cluster | Planned |
| Glue PySpark jobs (Bronze to Silver) | Planned |
| dbt models (Silver to Gold) | Planned |
| Airflow DAGs | Planned |
| AI Operations Agent | Planned |
| CDC Simulator | Planned |

---

## Claude Code authentication reference

I use Claude Code as my AI coding assistant throughout this project. This section documents how the authentication works so I can pick up where I left off after a session ends or a limit is hit.

### The two modes

| Mode | How it works | Limits | Billing |
|---|---|---|---|
| Claude Pro | Logged in with Anthropic account | Yes, resets on a rolling window | Covered by subscription |
| API Key | Authenticated with an API key | None | Pay per token |

For intensive sessions with long Terraform plans and multi-file work, the Pro limit can be reached quickly. API key mode removes the cap entirely.

### How to tell which mode I am in

The banner at the top of the session shows it:

```
Sonnet 4.6 · Claude Pro · your-email@...     <- Pro mode
Sonnet 4.6 · API Usage Billing               <- API key mode
```

Or check from the terminal:

```bash
claude auth status
```

### Switching to API key mode

```bash
# 1. Exit the current session (Ctrl+C or close terminal)

# 2. Log out of Pro
claude auth logout

# 3. Export the API key
export ANTHROPIC_API_KEY="sk-ant-..."

# 4. Launch Claude
claude

# 5. Inside the session, log in
/login
# Select API key when prompted
```

The banner changes to `API Usage Billing` and there is no session cap.

### Switching back to Pro

```bash
/logout                # inside the session
claude auth logout     # from the terminal
claude auth login
claude
```

### Making the API key permanent

```bash
echo 'export ANTHROPIC_API_KEY="sk-ant-..."' >> ~/.zshrc
source ~/.zshrc
```

### API key rules

Never paste an API key into a chat, commit it to Git, or include it in a screenshot. If one gets exposed, go to the Anthropic Console, revoke it immediately, and generate a new one.

---

*Each repository has its own README with module-level details and step-by-step instructions specific to that component.*
