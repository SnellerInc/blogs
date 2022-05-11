# Introducing sneller

## TL;DR

Sneller is a cloud-native, serverless (i.e no dedicated storage) SQL engine designed specifically for JSON. With Sneller, you can run interactive SQL on large TB-sized datasets of deeply nested semi-structured JSON. Our target use cases are event data pipelines such as Security, Observability, Ops, User Events and Sensor/IoT.

If you use Elasticsearch for these use cases today, you can replace the Elasticsearch engine (the 'E' in the ELK stack) with Sneller via an included Elastic adapter while providing improved unlimited storage scaling and significantly better performance, up to 10x faster on complex aggregate queries over multi-TB log datasets (Note: Unlike Elastic, Sneller is **not** a **free text search** engine).

If you're instead using a Data Warehouse/Data Lake and SQL for these use cases, Sneller can both simplify and speed up your existing pipeline. You no longer need ETL or ELT just to transform or reshape JSON to an alternative format. You can run plain SQL directly against deeply nested JSON without any non-standard, product-specific SQL extensions. Sneller’s extensive use of vectorization built on AVX-512 SIMD capabilities allow you to run sub second SQL queries against billions of event records.

Sneller is open-source under the AGPLv3 license.

## Introduction

In this blog, we’ll talk about
- Event Data - what it is, where it comes from and the specific challenges it poses for data management
- Why existing solutions, whether unstructured data platforms (like Elastic/ELK) or structured data (analytic DW/DBs that support SQL) are not a great fit for semi-structured event data
- How Sneller addresses these problems with Event Data and how you can sign up to be a beta user

## Event Data - What it is, where it comes from

A few months ago, Erik Bernhardsson, formerly of Spotify and someone who’s well known on data twitter, asked:

For a while now, we’ve been living in the world of event data that Erik is referring to. Events, in their simplest and most general form are timestamped (and increasingly, location-stamped) data about any activity of interest.

Some of the earliest and well known types of event data were (and still are) from **Security/SIEM use cases** - spanning security events from applications, operating systems and network infrastructure among other event types. Another big consumer of event data is **Observability** (often shortened to ‘o11y’), which covers application **logs**, infrastructure **metrics** (e.g. CPU load, memory utilization) and **traces** - which are data that show the detailed path that a request (e.g. an API call or a user action) takes through the entire technology stack from end to end. A third well known class of event data is **Product analytics** used by growth orgs, based on app instrumentation. At its heart event data is a time-ordered stream of well-defined user action events.
