---
title: "Living with SQL: How to Make It Work for You"
subtitle: ""
date: 2023-09-05T21:30:43+07:00
lastmod: 2023-09-05T22:10:43+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "Shares practical query and table design techniques to keep RDBMS performance from degrading as data grows."
license: ""
images: []
featuredImage: "featured-image.webp"
featuredImagePreview: "featured-image.webp"

tags: ["Database", "SQL"]
categories: ["SQL"]

lightgallery: true
---

When working with Relational Database Management Systems (RDBMS), a common issue is that as data grows, performance slows down. Why isn't it as fast as it was during development? Today, let's look at how to design queries and tables to achieve the performance they should have.

<!--more-->

## Orderable/Sortable Indexes
Keys and indexes must always be sortable. Simply put, they shouldn't be random.

## Primary Keys
Always use primary keys as the main key to separate data in rows. Oh, and don't be lazy and use `auto_increment`, because if you scale up to a distributed database, life will be difficult. Use ULID, Snowflake IDs, KSUID, or UUIDv7 instead.

## Field Length and Data Type
Use appropriate data types and sizes for the data being used. For example, a good index shouldn't be longer than 64 characters. Don't just use `varchar(255)` for everything. If you're using JSON, [index the fields within the JSON](https://www.postgresql.org/docs/current/datatype-json.html#JSON-INDEXING) that will be used in the WHERE clause so you don't have to use `WHERE LIKE` in JSON.

## Data Quantity and Retention
Only access/query the data you need. Don't sweep the entire table when you only use a few dozen fields.

## Searchable Arguments (SARGable Queries)
Always use Searchable Arguments (SARGable) in your queries. What is SARGable? Simply put, they are operators that can use indexes, which will make your queries faster (because they don't have to scan the entire table to check). Operations that effectively utilize B+Tree indexes include:
  - Equals, IN (=): For example, `WHERE indexed_column = value`
  - Inequality operators (<, <=, >, >=): Comparison operators like `WHERE indexed_column > value`. This does not include `<>` or `!=`.
  - BETWEEN: Compares a range, for example, `WHERE indexed_column BETWEEN low_value AND high_value`
  - LIKE (only `prefix%`): For example, `WHERE indexed_column LIKE 'prefix%'`. `%suffix` is not SARGable. If you need to use `LIKE '%key word%'`, you should consider using a database for full-text search.
  - IS NULL and IS NOT NULL: These are operations, for example, `WHERE indexed_column IS NULL`. `ISNULL(column)` (as a function) is not SARGable.
  - DISTINCT: Always use with **unique** indexed_column values.
  - EXISTS and NOT EXISTS: Always use with indexed_column values.
  - CASE: If used for direct comparisons, such as `CASE WHEN indexed_column = value THEN result END`, it can be SARGable.
  - Avoid using non-**MATH functions** in the WHERE clause, such as `WHERE YEAR(column) BETWEEN low_value AND high_value` or `WHERE LEFT(column, 1) = 'K'`.
  - JOIN ON, USING: Always with the conditions mentioned above.

## Concurrent Activities
In systems with concurrent operations, if you design it such that reads and writes occur on the same field simultaneously, it will naturally be slow. If there's no other way to design it, using a queue can help mitigate the issue.

## Data Staging
If you have a huge query that causes timeouts, breaking down large queries into smaller ones and then combining them can, in some cases, not only make them easier to understand but also improve performance. However, if a query looks like program code, it's probably not a good SQL query and should be moved into your application code.

## Flexible Column
If your data frequently changes its table structure or if a column in the table is excessively long, it's more suitable to use a **NoSQL** database.
