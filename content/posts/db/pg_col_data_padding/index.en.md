---
title: "Did you know that column ordering in DBs is also important?"
subtitle: ""
date: 2024-02-15T19:41:41+07:00
lastmod: 2024-02-15T19:41:41+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "Explains how column ordering in PostgreSQL affects tuple alignment and the on-disk row storage size through data padding."
license: ""
images: []

tags: ["Database", "SQL", "PostgreSQL"]
categories: ["SQL"]

featuredImage: "featured-image.webp"
featuredImagePreview: "featured-image.webp"

lightgallery: true
---
When creating tables in a database, have you ever noticed why we have to choose the data size? Let's take a look at how important column ordering is, using PGSQL as an example.
<!--more-->

## Data Alignment in PGSQL
First, let's look at the documentation to see what's related to data types in the DB.

{{< admonition note "Note" >}}
`typalign` char

`typalign` is the alignment required when storing a value of this type. It applies to storage on disk as well as most representations of the value inside PostgreSQL. **<ins>When multiple values are stored consecutively, such as in the representation of a complete row on disk, padding is inserted before a datum of this type so that it begins on the specified boundary</ins>**. The alignment reference is the beginning of the first datum in the sequence. Possible values are:

- c = char alignment, i.e., no alignment needed.
- s = short alignment (2 bytes on most machines).
- i = int alignment (4 bytes on most machines).
- d = double alignment (8 bytes on many machines, but by no means all).
{{< /admonition >}}
Source: [pg_type](https://www.postgresql.org/docs/current/catalog-pg-type.html)

As you can see, each data type uses a different amount of space, and the actual storage size depends on the order of the columns.

{{< admonition note "Note" >}}
Interpreting the actual data can only be done with information obtained from other tables, mostly `pg_attribute`. **<ins>The key values needed to identify field locations are `attlen` and `attalign`</ins>**. There is no way to directly get a particular `attribute`, except when there are only fixed width fields and no null values. All this trickery is wrapped up in the functions `heap_getattr`, `fastgetattr` and `heap_getsysattr`.
{{< /admonition >}}
Source: [Table Row Layout](https://www.postgresql.org/docs/current/storage-page-layout.html#STORAGE-TUPLE-LAYOUT)

From the emphasized text, we can see that good column ordering when creating a table not only affects storage but also the memory and CPU used by that table.

## About Padding
After reading the Docs, if you're still confused, let's see how it actually works.

Start by creating two tables.
```sql
CREATE TABLE tb_fragment (col_1 smallint, col_2 bigint, col_3 int, col_4 bigint);

CREATE TABLE tb_continuous (col_1 bigint, col_2 bigint, col_3 int, col_4 smallint);
```

Then let's examine how they look in storage.
```sql
SELECT a.attname, t.typname, t.typalign, t.typlen
FROM pg_class c
 JOIN pg_attribute a ON (a.attrelid = c.oid)
 JOIN pg_type t ON (t.oid = a.atttypid)
WHERE c.relname = 'tb_fragment' AND a.attnum >= 0;
 attname | typname | typalign | typlen
---------+---------+----------+--------
 col_1   | int2    | s        |      2
 col_2   | int8    | d        |      8
 col_3   | int4    | i        |      4
 col_4   | int8    | d        |      8
(4 rows)

SELECT a.attname, t.typname, t.typalign, t.typlen
FROM pg_class c
 JOIN pg_attribute a ON (a.attrelid = c.oid)
 JOIN pg_type t ON (t.oid = a.atttypid)
WHERE c.relname = 'tb_continuous' AND a.attnum >= 0;
 attname | typname | typalign | typlen
---------+---------+----------+--------
 col_1   | int8    | d        |      8
 col_2   | int8    | d        |      8
 col_3   | int4    | i        |      4
 col_4   | int2    | s        |      2
(4 rows)
```

As you can see:
- `BIGINT` uses 8 bytes
- `INT` uses 4 bytes
- `SMALLINT` uses 2 bytes

We calculate that both tables should use 22 bytes per row, but let's look at the reality.

```sql
select pg_column_size(tb_fragment.*) - 24 as row_size from tb_fragment limit 1;
 row_size
----------
       32
(1 row)

select pg_column_size(tb_continuous.*) - 24 as row_size from tb_continuous limit 1;
 row_size
----------
       22
(1 row)
```

You might wonder why the results differ.

At this point, if you look at `typlen`, those who have read ([Designing Go structs with knowledge of Computer Architecture and Data Structures]({{< ref "/posts/go/struct_memory" >}} "Designing Go structs with knowledge of Computer Architecture and Data Structures")) will find that it's similar. Yes, it's the same thing. What happened is the padding between fields.

We can use the `pageinspect` extension to explain what happened:

```sql
SELECT t_data FROM heap_page_items(get_raw_page('tb_fragment','main',0))gx
-[ RECORD 1 ]--------------------------------------------------------------
t_data | xff7f000000000000ffffffffffffff7fffffff7f00000000ffffffffffffff7f

SELECT t_data FROM heap_page_items(get_raw_page('tb_continuous','main',0))gx
-[ RECORD 1 ]------------------------------------------
t_data | xffffffffffffff7fffffffffffffff7fffffff7fff7f
```

As you can see, in the `tb_fragment` table, when data is retrieved, there will be zero padding until it reaches 8 bytes, and then it moves to the next column because **col_1 (2 bytes) + col_2 (8 bytes) exceeds 8 bytes, causing them to be separated**. This is different from `tb_continuous`, where **col_3 (4 bytes) + col_4 (2 bytes) does not exceed 8 bytes, allowing them to be combined into the same word**.

## Conclusion
Planning and designing the structure by considering low-level aspects not only helps us avoid wasting storage but also improves memory and CPU usage. This is because we don't have to waste time loading unnecessary data. However, this comes at the cost of column order readability. So, we might leave PKEYs or other indexes that make the table easier to read in the first columns, as is commonly done, without forcing an order just for performance.
