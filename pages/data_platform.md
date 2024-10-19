---
title: Data Platforms
sidebar_position: 99
hide_title: true
categories: data, platform
---
# If you want to build a Data Platform, ...

So you decide to build a "Data Platform" to build Data & AI capabilities because the company wants to be a data driven one. Most of what happens next  ( _in most places , of course there are exceptions _)

You will be able to relate it to _ **If you give Pig a party, .... **_

### Cloud
This is a given and the company has already made decision for you. Usually by someone top up the chain of command and for good reasons. From my experience ( albeit limited ), the **Azure** sales team does a wonderful job of it and I have been using it for most part of the 10 years and I can say - "It works and no complaints whatsoever". So you stick with **ADLS Gen2** . But it applies to GCP and AWS as well. No major surprises.

### Data Platform ( the actual one )
Give it is **Azure** , the next step is the next sales team's job - **Databricks** or **Snowflake** , and the Cloud Hyperscaler's own solutions **BigQuery**, **Fabric** etc. Usually, the company gets sold on _**Advance Credits**_ by one of the sales teams for the next year and they go AWOL until the credits run out or renewal kicks in.

### Access Management
This is not really a technical issue but more of the way things are already done in a company. If the company has been in existence for a while, there is definitely some sort of access management in the company - usually with the words `User` `Access` and `Management` in one or the other order. And it has to be tied to the - `Azure AD`  (given O365's dominance).  And now comes the `Security` folks who will usually determine the granularity of access, isolation etc. For those on Azure , `MyAcess` is an alternative to a home grown solution ( if your security team allows it)

### IAC
Its 2024 and its a taboo to not have IAC, most teams end up doing IAC with **Terraform**. Even though it has changed hands, nothing seems to have affected the users and I have not been in any `OpenTofu` party yet. The technical aspect of the IAC isn't that complicated but one can really complicate and messup any terraform code. What adds to delay is who controls what and should everything be IAC or should the teams have control - Workspace, Warehouses etc. etc.

I must admit that `Databricks` and `Snowflake` can be done without much IAC ( and there are still teams that do that to avoid dependency on the platform team ) and a team can get started real quick using the UI. The real problem kicks in when there has to be governance on the Users, RBAC and FinOps. And when your ** Data Developers ( Data Engineer, Analysts , AI and ML) ** grows in numbers.

Esp. on Azure - Subscriptions, Resource Groups, Cost Management along with metadata from any of these platforms is sufficient for FinOps and Monitoring.

### Integration

The most utopian solution to avoid this problem is that every team sends their well structured data to a kafka topic ( or similar) with schema governance and testing. However , most companies are not in this utopian world. There are numerous sources of data - rdbms , nosql, mainframe , api, MQ , FTP, SFTP etc. 

The main decision here is to Buy Vs Build. There are quite a few SaaS solutions FiveTran, Meltano and so on. Databricks has a recent acquisition that does it. As suspected , this involves the long play with Procurement and usually they are costly ( given their pricing models ). Or maintain a team to build and maintain the solution. This is not a fancy job description and there is likely to be churn in this role and knowledge lost.  [DltHub](https://www.dlthub.com) is an OSS solution in Python trying to make it less painful.

`Shift Left` is preferred if possible as explained [here](https://www.youtube.com/watch?v=FiZmyl1Npg0) 

### Orchestration

Orchestra, Kestra, Prefect, Airflow, Dagster, Argo , Databricks Workflows , ADF - There are a dozen of the tools that does the workflows. The most common method of orchestration would be the good old _** Cron **_. 

If the workflows are independent and can handle state well, _Cron_ is all that you need. But there can be additional needs (or wants ) -
* Concurrent Runs
* DAG
* Dependency ( Upstream and Downstream )
* Auto Run (Materialization)
* Triggers (Eventing or Polling)
* Delayed Runs

Some of the tools provide this and there might or might not be value in many of these features. And probably 90% of use cases do not require it but who can argue against `What an engineer wants ?` . And many of these tools are beginning to run into the solutions of `Data Governance` tools and `Snowflake/Databricks` provides these already (either as UI or system tables ).

### PySpark vs SQL

This is always matter of expertise in the company, though `Everything SQL` seems to be growing trend and I have started to like SQL very much. While the python area seems to quite settled on `pyspark` , there are options on `polars` `pandas` and many other options. `DuckDb` has approach with [BIG DATA IS DEAD](https://motherduck.com/blog/big-data-is-dead/).

On the SQL side, `dbt` seems a outright winner and has made life easy for the SQL developers. `SQLMesh` and `SDF` are the new kids on the block and they are almost drop in replacement for existing `dbt` code.

### Monitoring and Observablity

This is always a requirement in any Cloud solution. There is always more than one solution and there are always haters and lovers for a specific solution and the market is crowded - Azure App Insights, Datadog, Grafana Cloud, Grafana Open Source.

### Optimization and Finops

Given that there **will** be inefficient queries, compute , workflows and cost overrun, if **ANYONE** in the company is worried, they might ask the teams to handle it or go for a third party tool that will do it for you - _at a cost_. There is a ecosystem of cloud services available to optimize your platform, but the COST of using them is major `psychological` and probably `egoistic` barrier in adoption. It is quite possible that the CFO of the company will buy/implement it rather than the engineering teams.

### Governance and Data Catalog 

Now that you have a data platform, and it has been a Engineering and CFO party, now comes the time for the mighty `Business` folks and almost every company ( except the pure tech companies ) is driven by the `Business` folks. The sales and marketing teams of the are no less worthy than then Cloud Platforms - they have their own spin and they are quite successful as well. 


And eventually one of the large platforms gets chosen and that completes your `Data Platform` and everyone gets to live `without meetings` there after ... Whether everyone gets to live `happily` ever after depends .... 

### BI / Data Visualization
This varies from the usual suspect Power BI or Tableau via procurement and if the company has the appetite, there are a lot of new kids on the block - 

* `BI as Code`  [Evidence.dev](https://evidence.dev) 
* AI Powered  [Hex.tech](https://hex.tech)
* [Observable](https://observablehq.com/) 
* [Mosaic](https://idl.uw.edu/mosaic/)

Usually there are two kinds of setups 

* Cloud Platform / Data Platform are decided by the top management.
* Completely OSS
    * Airflow / Kubernetes / Argo / Grafana / Datahub
* Completely Vendor
    * Astronomer / Dagster and the likes ..

`Optimization and FinOps` should get the most love but they are often the ignored.

If you want to build a Data Platform, it is either "OSS Solution"  that needs a full scrum team to maintain it , or "Vendor Management Solution" that involves Finance and Procurement.

_** The question is - Pay the engineer or Pay the vendors ?? **_


Next  - "How to build and maintain a lean OSS Solution with Azure and Snowflake/Databricks"