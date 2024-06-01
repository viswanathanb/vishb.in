---
title: Data Platforms
sidebar_position: 99
hide_title: true
categories: data, platform
---
# Building a Data Platform

You've been tasked with building a "Data Platform" to enable Data & AI capabilities. The company wants to be data-driven. Here's what typically unfolds — and if you've been through it, you'll recognize the pattern immediately.

### Cloud

This decision is usually made for you. Someone senior signed a multi-year enterprise agreement, and for good reason — it simplifies procurement and consolidates spend. In my experience, **Azure** dominates in enterprises (the sales team is remarkably effective), and I've used it for the better part of a decade with no complaints. You'll likely land on **ADLS Gen2** for storage. The same patterns apply to GCP (GCS) and AWS (S3) — no major surprises across providers.

### Data Platform (the actual one)

Given Azure, the next decision is which compute layer: **Databricks**, **Snowflake**, or the hyperscaler's native offering (**BigQuery**, **Fabric**, etc.). Typically, one vendor's sales team secures an _advance credits_ deal for the next year, and then goes quiet until renewal. The platform choice has long-term implications for tooling, talent, and lock-in — but the initial decision is often driven by commercial terms rather than technical evaluation.

### Access Management

This is less of a technical problem and more of an organizational one. Every company of any tenure has some form of Identity and Access Management — usually tied to **Azure AD** (now Entra ID) given O365's dominance. The security team determines granularity: workspace-level isolation, row-level security, data classification policies. On Azure, **MyAccess** (Entitlement Management) can replace home-grown approval workflows if your security team permits it.

### Infrastructure as Code

It's 2024 — IAC is table stakes. Most teams use **Terraform**. Despite the HashiCorp licensing change, the ecosystem remains stable and I haven't encountered meaningful `OpenTofu` adoption in production yet. The technical implementation isn't complex, but the organizational questions are: who owns what, should everything be codified, and how do teams retain autonomy over their workspaces and compute?

Both Databricks and Snowflake can be operated without IAC — teams can get started quickly through the UI. The problems surface when governance requirements emerge: RBAC consistency, FinOps visibility, and audit compliance across a growing number of **data developers** (engineers, analysts, ML practitioners).

On Azure specifically, the combination of Subscriptions, Resource Groups, and Cost Management — along with metadata from the data platform — is sufficient for FinOps and operational monitoring.

### Integration

The ideal state: every team publishes well-structured data to a Kafka topic with schema governance and testing. Reality: most organizations have data scattered across RDBMS, NoSQL, mainframes, APIs, message queues, FTP/SFTP, and spreadsheets.

The core decision is **build vs. buy**. SaaS options (Fivetran, Airbyte, Meltano) offer speed at a cost — their pricing models scale with volume and can become expensive. Self-managed solutions require a dedicated team, and this role tends to see churn, leading to knowledge loss. [DltHub](https://www.dlthub.com) is an open-source Python library trying to reduce that pain.

Where possible, **shift-left** ingestion — push data quality responsibility to the producing team, as explained [here](https://www.youtube.com/watch?v=FiZmyl1Npg0).

### Orchestration

The options are plentiful: Orchestra, Kestra, Prefect, Airflow, Dagster, Argo Workflows, Databricks Workflows, Azure Data Factory. And for many workloads, the most effective orchestrator remains **cron**.

If workflows are independent and handle state well, cron is sufficient. Additional requirements that justify a dedicated orchestrator:

- Concurrent execution
- DAG-based dependency management
- Upstream/downstream coordination
- Materialization triggers
- Event-driven or polling-based scheduling
- Retry and delayed execution

Many orchestration tools are expanding into data governance territory, overlapping with capabilities that Snowflake and Databricks already provide natively via system tables and UI.

### PySpark vs SQL

This depends on the team's expertise, though the **"everything SQL"** trend is gaining ground — and for good reason. On the Python side, `pyspark` remains dominant, with `polars` and `pandas` as alternatives for smaller-scale workloads. `DuckDB` challenges the need for distributed compute entirely with the thesis that [Big Data is Dead](https://motherduck.com/blog/big-data-is-dead/).

On the SQL side, **dbt** is the clear winner for transformation workflows. **SQLMesh** and **SDF** are emerging alternatives that offer near drop-in compatibility with existing dbt projects.

### Monitoring and Observability

A requirement in any cloud deployment, and the market is crowded: Azure Application Insights, Datadog, Grafana Cloud, Grafana OSS, and many more. Every team has preferences — pick one that integrates well with your stack and commit to it.

### Optimization and FinOps

Inefficient queries, over-provisioned compute, runaway workflows, and cost overruns are inevitable. If leadership cares (and eventually they will), the response is either to mandate team-level optimization or to procure a third-party tool. There's a growing ecosystem of FinOps services, but adoption faces both psychological and organizational barriers — engineering teams rarely volunteer to instrument cost controls. More often, it's the CFO's office that drives implementation.

### Governance and Data Catalog

Once the platform is running and engineering has delivered, the **business** stakeholders arrive. Data governance and catalog vendors have sales teams every bit as effective as the cloud providers — with their own narratives and success stories. A platform is eventually chosen (Collibra, Atlan, Datahub, Unity Catalog, etc.), governance policies are defined, and the "Data Platform" is declared complete.

Whether everyone lives happily ever after depends on ongoing investment.

### BI and Data Visualization

The usual suspects — **Power BI** or **Tableau** — arrive via procurement. For teams with appetite for alternatives:

- **BI as Code**: [Evidence.dev](https://evidence.dev)
- **AI-Powered**: [Hex.tech](https://hex.tech)
- **Notebooks + Viz**: [Observable](https://observablehq.com/)
- **Research-Grade**: [Mosaic](https://idl.uw.edu/mosaic/)

---

### The Two Paths

In practice, data platforms fall into one of two camps:

- **Fully OSS** — Airflow, Kubernetes, Argo, Grafana, Datahub — requires a dedicated team to build and maintain
- **Fully Vendor-Managed** — Astronomer, Dagster Cloud, Fivetran, etc. — requires procurement and vendor management

**Optimization and FinOps** deserve the most attention but are often the most neglected.

The fundamental question: **pay the engineers or pay the vendors?**

---

_Next: How to build and maintain a lean OSS solution with Azure and Snowflake/Databricks._