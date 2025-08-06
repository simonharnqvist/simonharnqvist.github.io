---
layout: post
title: Bridging Spark and Dask with Apache Iceberg: A Practical Workaround
date: 2025-08-06 16:03 +0100
---
# Bridging Spark and Dask with Apache Iceberg: A Practical Workaround

As part of a science platform project, I worked on deploying a Jupyter code-to-data solution to enable scientists to analyse petabyte-scale datasets. One of the core challenges was supporting **both Spark and Dask** users accessing the same storage layer efficiently — specifically, data stored in **Apache Iceberg**.

---

## Project Context


The project had several hard constraints:

- Data needed to be **geographically partitioned**, based on access patterns.
- We had to support **fast, partition-aligned joins** — without requiring additional sorting or shuffling.
- Users should be able to analyse data with either **Spark** or **Dask**.

Another team had already selected [**Apache Iceberg**](https://iceberg.apache.org/) as the core table format. This made sense; Iceberg supports logical partitioning and sorting guarantees, which let Spark users benefit from optimized queries without re-sorting the data. Without it, we found that Spark would insist on sorting data, which in our case took considerable resources.

However, Dask presented a major challenge.

---

## The Problem: Iceberg Has No Real Support in Dask

While Spark integrates well with Iceberg — more or less, point at the catalog and shoot — Dask does not yet have native support for Iceberg.

Yes, there's [Daskberg](https://github.com/martindurant/daskberg), an experimental connector by Martin Durant, but it hasn’t seen development in over three years. As of writing, there appears to be limited movement in the Dask community toward Iceberg support.

This left me with a core question:

> **How can we enable both Spark and Dask users to access the same Iceberg data — while preserving the ability to perform fast, storage-partitioned joins?**

---

## The Workaround: Letting the Lakehouse Lie

Iceberg’s underlying storage format is Parquet — an open standard. At first glance, there’s no major issue; Dask can read those files directly using `dask.dataframe.read_parquet()`. And this works — but only up to a point.

The problem is **partition awareness**.

Iceberg stores **logical partitions**, meaning partitioning is encoded in metadata, not in the physical directory layout. Dask, on the other hand, relies on **Hive-style partitioning**, which uses actual directory structures (e.g. `region=EU/`) to guide partition-aware reads and joins.

So I asked myself: Iceberg or Hive-style physical partitioning?

The answer, I eventually realised, was **both**.

---

## The Insight: Dual Reality, Carefully Maintained

By aligning the physical layout of our data with Hive-style partitioning (e.g. `region=EU` folders), and registering that data with Iceberg using logical partition metadata, we had a solution where:

- **Spark sees an Iceberg table**, and benefits from its metadata, partitioning, and optimizations.
- **Dask sees a Hive-partitioned Parquet dataset**, and can read partition-aware subsets without needing Iceberg support.

The catch? **This setup must be strictly read-only**. Any writes from Dask would bypass Iceberg metadata. Any writes from Spark would update Iceberg but break the physical layout Dask relies on. The best case scenario would then be that data loading fails, allowing us to fix the problem. Far more worringly, the two distributed computing packages could present two diverging datasets to users when there should only be one.

---

## Reflections

There is no doubt that this is a somewhat fragile design. Ensuring that read-only data is only ever read is not in itself a challenging or unusual requirement; but the need to  repartition the dataset in its entirety should we ever need to update or add as little as a single row is. Particularly so, since the major selling point of Iceberg is its ability to handle schema evolution and time travel; it is made for dynamic datasets.

The solution presented here is a hack if I’ve ever seen one.

But for our use case, it met the core requirements: support for Spark and Dask, fast joins, and compatibility with the data engineering decisions already in place.

There are trade-offs: we forgo Iceberg’s schema evolution and ACID guarantees altogether, and we must strictly control access (more so than usual) to avoid accidental writes.

Still, this solution enables our scientists to access petabyte-scale data via the tools they prefer — without duplicating storage or workflows.

Until Dask introduces official Iceberg support, or Spark stops requiring re-sorting, this hybrid approach holds — as long as everyone involved in managing the lakehouse remembers: **read-only means read-only.**