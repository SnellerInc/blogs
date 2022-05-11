# Introducing sneller

## TL;DR

Sneller is a cloud-native, serverless (i.e no dedicated storage) SQL engine designed specifically for JSON. With Sneller, you can run interactive SQL on large TB-sized datasets of deeply nested semi-structured JSON. Our target use cases are event data pipelines such as Security, Observability, Ops, User Events and Sensor/IoT.

If you use Elasticsearch for these use cases today, you can replace the Elasticsearch engine (the 'E' in the ELK stack) with Sneller via an included Elastic adapter while providing improved unlimited storage scaling and significantly better performance, up to 10x faster on complex aggregate queries over multi-TB log datasets (Note: Unlike Elastic, Sneller is **not** a **free text search** engine).

If you're instead using a Data Warehouse/Data Lake and SQL for these use cases, Sneller can both simplify and speed up your existing pipeline. You no longer need ETL or ELT just to transform or reshape JSON to an alternative format. You can run plain SQL directly against deeply nested JSON without any non-standard, product-specific SQL extensions. Sneller’s extensive use of vectorization built on AVX-512 SIMD capabilities allow you to run sub second SQL queries against billions of event records.

[Sneller](https://github.com/SnellerInc/sneller) is open-source under the AGPLv3 license.

## Introduction

In this blog, we’ll talk about
- Event Data - what it is, where it comes from and the specific challenges it poses for data management
- Why existing solutions, whether unstructured data platforms (like Elastic/ELK) or structured data (analytic DW/DBs that support SQL) are not a great fit for semi-structured event data
- How Sneller addresses these problems with Event Data and how you can sign up to be a beta user

## Event Data - What it is, where it comes from

A few months ago, Erik Bernhardsson, formerly of Spotify and someone who’s well known on data twitter, asked:

For a while now, we’ve been living in the world of event data that Erik is referring to. Events, in their simplest and most general form are timestamped (and increasingly, location-stamped) data about any activity of interest.

Some of the earliest and well known types of event data were (and still are) from **Security/SIEM use cases** - spanning security events from applications, operating systems and network infrastructure among other event types. Another big consumer of event data is **Observability** (often shortened to ‘o11y’), which covers application **logs**, infrastructure **metrics** (e.g. CPU load, memory utilization) and **traces** - which are data that show the detailed path that a request (e.g. an API call or a user action) takes through the entire technology stack from end to end. A third well known class of event data is **Product analytics** used by growth orgs, based on app instrumentation. At its heart event data is a time-ordered stream of well-defined user action events.

Event Data are seen in many areas beyond security, observability and product/service analytics. Real-time **sensor data** from the rapidly expanding world of connected devices, vehicles and industrial equipment are basically event data. Finally, in the rapidly emerging world of Ops workflows (DevOps, DataOps, MLOps/AIOps), the core telemetry data from each stage in an Ops pipeline is again a form of event data.

Of note is [OpenTelemetry](https://opentelemetry.io), an emerging standardization effort for event data that aims to provide a common vocabulary for event data domains to avoid needless redundancy/reinvention.

## Why do we a new data platform specifically for event data?

Since there’s been a Cambrian explosion of tools for every kind of data pipeline or use case, we have to justify why we felt the need to add another one to the mix. To justify this, here (in our view) are the three key characteristics of event data and the needs of an event data platform.

## Why do today’s data platforms fall short for event data?

Here’s a quick summary of an ideal data platform for event data:
1. Cost effective, even when faced with essentially unbounded retention of data at data rates easily approaching multiple terabytes a day or more
2. Ability to support both interactive monitoring and deeper ad-hoc, long-range investigation of usage patterns on the same platform - without over-provisioning infrastructure or added complexity
3. Handle schema evolution without huge sunk cost in setting up and maintaining data transformation/reshaping (aka ETL/ELT) pipelines
4. Optimized for SQL, to allow integration with the broadest ecosystem of data tools in existence

**Text Search platforms:** Tools like Elasticsearch, OpenSearch, ChaosSearch and Quickwit belong in this category. They were designed first and foremost to handle querying of unstructured data such as text. Because they were built to extract structure from text (often by building an [inverted index](https://en.wikipedia.org/wiki/Inverted_index)), they've become a natural choice for handling the semi-structured nature of event data (such as structured logging).  Of these, Elastic has the advantage of a long-established open source solution with the ELK stack for observability and security use cases.

**Structured data platforms:** Every data warehouse, data lake or analytic database that supports SQL as its primary API, belongs in this category. The obvious advantage here is unmatched familiarity of SQL, and the sheer scale of the supporting ecosystem that has developed over decades.
When we consider the three characteristics of an ideal event data platform, these alternatives fall short along typically one or more dimensions.

## Sneller: Built for event data

This is what we set out to solve at Sneller. We wanted to provide:
-	The ideal combination of performance and economics at the typical scale of semi-structured event data (GB-TB/day or more)
-	Comprehensive SQL support, without non-standard types or custom extensions
-	Far simpler pipelines for JSON and semi-structured data - no more brittle ETL/ELT only for data reshaping, and no more headaches due to schema evolution

We will get into details of how we achieved this in subsequent blog posts. For now, here’s an illustration of the complete Sneller Cloud user experience today (technical details to follow in subsequent blog posts).

Sneller’s core value compared to existing alternatives, is **simplicity at scale**. You simply load your JSON-encoded event data directly into S3 (or other cloud object stores in the near future), and proceed to query it with standard SQL. There's no need to provision dedicated capacity for indexing, build ETL/ELT pipelines, or handle schema evolution, or worry about how to age your data out to cheaper storage. This comes from four key design decisions:

### Storage/compute separation (aka 'Serverless')

As the dominant storage mechanism in a cloud-native world, centralized object storage services such as S3, GCS or Azure Blob are battle-tested, secure and cost-effective at petabyte scale. They are the natural choice for the storage economics of unbounded event data.

Sneller is foundationally built to use object storage as its **primary** storage. A major problem with data infrastructure today is over/under-provisioning - because storage and compute are not separate, you always end up paying for compute you don't need while scaling storage (or vice versa). Sneller lets you dial compute capacity independently up or down based on how your workload changes (or your SLAs), without worrying about scaling or availability of the storage subsystem. As a result, Sneller is far more cost effective than comparable solutions - you only pay based on the work you do, not the volume of stored data, or an 'always-on' infrastructure size you provisioned upfront.

Moreover, unlike many databases/data warehouses, Sneller does not ‘copy’ or require you to load your data into a custom format or data layout, columnar or otherwise (we’ll elaborate on this admittedly controversial tradeoff in subsequent posts). As a result, in Sneller Cloud, your data - source data, intermediate data and outputs, are **always in your control**, in your own account in S3 (and soon GCS and Azure blob).

Finally, Sneller Cloud compute nodes do not have any locally attached storage, which considerably simplifies scaling but also means that we don't persist any customer data (even inadvertently) within our service. In memespeak,

### Schema-agnostic

Sneller is completely [schema-on-read](https://www.techopedia.com/definition/30153/schema-on-read). This is a huge advantage for semi-structured JSON data which is naturally built to be flexible. It avoids the need for brittle, hard to maintain ETL/ELT pipelines whose sole purpose is to reshape JSON to ingest it into a table.

This radically simplifies your JSON event data pipeline compared to both search platforms like Elastic that require dedicated indexing capacity, or to data warehouses that need a dedicated transformation pipeline for JSON. You can add new fields, change field types (and even handle type ambiguity issues) in your source data without the need to manage [schema evolution](https://en.wikipedia.org/wiki/Schema_evolution), which is a significant problem for event data in particular.

### Speed

With Sneller, you can run low latency, interactive SQL directly on TBs or more of semi-structured JSON data.

We did this by writing our engine from the ground up, to be [SIMD-aware](https://en.wikipedia.org/wiki/Single_instruction,_multiple_data). Specifically, we use AVX-512 extensions available on modern Intel server CPUs (also available in AMD when Zen4 arrives). This allows us up to 16x the throughput for core kernels in our engine without requiring specialized accelerator hardware. The ‘done right’ part is the trick though - we had to develop a library of assembly-level AVX-512 routines integrated into a Golang-based execution engine for this purpose.

### SQL

While SQL is far and away the most popular API (or UX) for data, it is closely tied to the relational (row/column) data model. On the other hand, JSON data allows for far richer structure including nested objects and complex field types. To use SQL with JSON, databases often resort to workarounds like [specialized variant types](https://docs.snowflake.com/en/sql-reference/data-types-semistructured.html) to store JSON and/or dedicated, [non-standard JSON](https://cloud.google.com/bigquery/docs/reference/standard-sql/json_functions) helper functions.

By contrast, Sneller leverages [PartiQL](https://partiql.org), an extended dialect of SQL originally designed by Amazon specifically for semi-structured data (and supported by many products such as DynamoDB, Glue within the AWS data platform ecosystem). This gives you the full power and flexibility of SQL for complex nested event data structures without requiring [explicit ELT/ETL](https://docs.snowflake.com/en/user-guide/json-basics-tutorial-flatten.html) into a tabular model, or product-specific extensions to handle this kind of data.

Taken together, Sneller offers the familiarity and capability of SQL for semi-structured JSON data, without the cost or complexity of either search platforms like Elastic, or data warehouse pipelines with dedicated ETL/ELT stages for transformation/reshaping JSON. In our benchmarks we have seen upto a 50-80% cost advantage over Elastic Cloud while obtaining a 3-10x (or more) increase in query performance for typical queries on billion-event datasets that contain complex nested JSON. Watch out for an upcoming blog that dives into great detail on how we built our benchmarking infrastructure (which is also soon to be open sourced).

## Getting started with Open Source

We have [open sourced](https://github.com/SnellerInc/sneller) Sneller’s core engine under the AGPLv3 license. You can download, build and try Sneller on your own JSON data. To get you started, we’ve provided canned datasets and also a conversion utility that creates ion (the compressed binary JSON format we use) from your own JSON data. You’ll need access to an AVX-512 capable server - these are widely available on all cloud providers.

Additionally, we also have instructions on how you can set up and run [Sneller with Kubernetes](https://docs.sneller.io/kubernetes.html).

Since we’re in early days, we would love any feedback - issues, stars, comments all appreciated!

## Sign up/Learn more

If that piqued your interest, please drop by our [website](https://sneller.io) and sign up to try Sneller Cloud for free on your own data. 

Alternatively, you can head over to our [GitHub page](https://github.com/SnellerInc) and build and try Sneller for yourself. Over the next few weeks, we’ll have many more deep dives into Sneller’s core technology and engineering, as well as how we benchmarked Sneller, so please stay tuned!

