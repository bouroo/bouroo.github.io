---
title: "รู้หรือไม่ ว่าการเรียง column ใน DB ก็สำคัญนะจร๊ะส์"
subtitle: ""
date: 2024-02-15T19:41:41+07:00
lastmod: 2024-02-15T19:41:41+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: ""
license: ""
images: []

tags: ["Database", "SQL", "PostgreSQL"]
categories: ["SQL"]

featuredImage: "featured-image.webp"
featuredImagePreview: "featured-image.webp"

lightgallery: true
---
เวลาสร้างตารางในฐานข้อมูลเคยสังเกตไหมครับว่า ทำไมเราต้องมานั่งเลือกขนาดของข้อมูล เลยจะมาลองดูกันว่ามันสำคัญยังไงไอ้การเรียง column เนี้ย โดยตัวอย่างที่ยกมาจะเป็น PGSQL เด้อ
<!--more-->

## การจัดเรียงข้อมูลบน PGSQL
อันดับแรกเรามาดูใน document กันว่ามีอะไรที่เกี่ยวข้องกับประเภทของข้อมูลบน DB กันบ้าง

{{< admonition note "Note" >}}
`typalign` char

`typalign` is the alignment required when storing a value of this type. It applies to storage on disk as well as most representations of the value inside PostgreSQL. **<ins>When multiple values are stored consecutively, such as in the representation of a complete row on disk, padding is inserted before a datum of this type so that it begins on the specified boundary</ins>**. The alignment reference is the beginning of the first datum in the sequence. Possible values are:

- c = char alignment, i.e., no alignment needed.
- s = short alignment (2 bytes on most machines).
- i = int alignment (4 bytes on most machines).
- d = double alignment (8 bytes on many machines, but by no means all).
{{< /admonition >}}
ที่มา [pg_type](https://www.postgresql.org/docs/current/catalog-pg-type.html) 

จะเห็นว่าแต่ละประเภทข้อมูลจะใช้พื้นที่ไม่เท่ากันตามแต่ละประเภท ซึ่งขนาดบน storage ที่ใช้งานจริงจะขึ้นอยู่กับการเรียงลำดับของ column ด้วย

{{< admonition note "Note" >}}
Interpreting the actual data can only be done with information obtained from other tables, mostly `pg_attribute`. **<ins>The key values needed to identify field locations are `attlen` and `attalign`</ins>**. There is no way to directly get a particular `attribute`, except when there are only fixed width fields and no null values. All this trickery is wrapped up in the functions `heap_getattr`, `fastgetattr` and `heap_getsysattr`.
{{< /admonition >}}
ที่มา [Table Row Layout](https://www.postgresql.org/docs/current/storage-page-layout.html#STORAGE-TUPLE-LAYOUT)

จากข้อความที่เน้นไว้จะพบว่าการเรียง column ให้ดี ตอนที่สร้างตารางไม่ได้มีผลแค่ storage เท่านั้นยังมีผลต่อ memory กับ cpu ที่ใช้งานตารางนั้นด้วย

## ว่าด้วยเรื่อง Padding
หลังจากอ่าน Docs แล้วยัง งง กันอยู่เรามาลองดูว่าพอใช้งานจริง ๆ มันจะเป็นยังไงนะ

เริ่มจากสร้างตารางมา 2 อัน
```sql
CREATE TABLE tb_fragment (col_1 smallint, col_2 bigint, col_3 int, col_4 bigint);

CREATE TABLE tb_continuous (col_1 bigint, col_2 bigint, col_3 int, col_4 smallint);
```

แล้วก็มาส่องดูว่าหน้าตาบน storage เป็นยังไง
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

จากที่เห็นก็คือ
- `BIGINT` ใช้ 8 bytes
- `INT` ใช้ 4 bytes
- `SMALLINT` ใช้ 2 bytes

เราคำนวณออกมาได้ว่าทั้ง 2 ตารางใช้ก็น่าจะใช้ 22 bytes ต่อหนึ่ง row แต่แล้วเรามาดูความเป็นจริง

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

อ้าวเห้ย ทำไมไม่เท่ากันละ ถถถ

มาถึงตรงนี้ถ้าสังเกตจาก `typlen` คนที่เคยอ่าน ([ออกแบบ Go struct ด้วยความรู้วิชา Computer Architecture และ Data Structure]({{< ref "/posts/go/struct_memory" >}} "ออกแบบ Go struct ด้วยความรู้วิชา Computer Architecture และ Data Structure")) ก็จะพบว่ามันคล้าย ๆ กันเลยนะ ใช่ครับมันคือสิ่งเดียวกันเลย สิ่งที่เกิดขึ้นคือ padding ระหว่าง filed ที่ออกมานั่นเองครับ

ซึ่งเราสามารถใช้ extension `pageinspect` เพื่ออธิบายสิ่งที่เกิดขึ้นได้ตามนี้

```sql
SELECT t_data FROM heap_page_items(get_raw_page('tb_fragment','main',0))gx
-[ RECORD 1 ]--------------------------------------------------------------
t_data | xff7f000000000000ffffffffffffff7fffffff7f00000000ffffffffffffff7f

SELECT t_data FROM heap_page_items(get_raw_page('tb_continuous','main',0))gx
-[ RECORD 1 ]------------------------------------------
t_data | xffffffffffffff7fffffffffffffff7fffffff7fff7f
```

จะเห็นว่าในตาราง `tb_fragment` เวลาดึงข้อมูลออกมานั้นจะมี zero padding จนเต็ม 8 bytes แล้วค่อยเป็น column ถัดไปเพราะ **col_1(2 bytes) + col_2(8 bytes) เกิน 8 bytes ทำให้โดนแยกออกจากกัน** ซึ่งจะต่างจาก `tb_continuous` ที่ **col_3(4 bytes) + col_4(2 bytes) ไม่เกิน 8 bytes ทำให้รวมกันออกมาด้วย word เดียวกันได้เลย**

## สรุป
การวางแผนออกแบบโครงสร้างโดยพิจารณาสิ่งที่จะเกิดขึ้นระดับ low-level นอกจากจะช่วยให้เราไม่สูญเสีย storage ไปเปล่า ๆ แล้วยังช่วยเพิ่มประสิทธิภาพในการใช้งาน memory กับ cpu ได้อีกด้วย เนื่องจากไม่ต้องเสียไปกับการ load data ที่ไม่จำเป็น แต่ว่าก็ต้องแลกมากับลำดับความสัมพันของ column ที่จะดูไม่รู้เรื่อง เราก็เลยอาจจะเว้นไว้สำหรับ PKEY หรือ index อื่น ๆ ที่จะทำให้เราอ่านตารางได้ง่าย ให้มาอยู่ column แรก ๆ แบบที่ชาวโลกปกติทำกัน โดยไม่ต้องไปฝืนเรียงเพื่อเอา perfomance เพียงอย่างเดียว
