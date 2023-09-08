---
title: "เมื่อต้องใช้ชีวิตกับ SQL ก็ต้องอยู่ให้เป็น"
subtitle: ""
date: 2023-09-05T21:30:43+07:00
lastmod: 2023-09-05T22:10:43+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin-vir.pages.dev"
description: ""
license: ""
images: []
resources:
- name: "featured-image"
  src: "featured-image.webp"
- name: "featured-image-preview"
  src: "featured-image.webp"

tags: ["Database", "SQL"]
categories: ["SQL"]

hiddenFromHomePage: false
hiddenFromSearch: false
twemoji: false
lightgallery: true
ruby: true
fraction: true
fontawesome: true
linkToMarkdown: true
rssFullText: false

toc:
  enable: true
  auto: true
code:
  copy: true
  maxShownLines: 50
math:
  enable: false
  # ...
mapbox:
  # ...
share:
  enable: true
  # ...
comment:
  enable: true
  # ...
library:
  css:
    # someCSS = "some.css"
    # located in "assets/"
    # Or
    # someCSS = "https://cdn.example.com/some.css"
  js:
    # someJS = "some.js"
    # located in "assets/"
    # Or
    # someJS = "https://cdn.example.com/some.js"
seo:
  images: []
  # ...
---

เมื่อเราต้องทำงานกับ Relational database (RDBMS) สิ่งที่พบบ่อย ๆ เลยคือเมื่อข้อมูลมาก ขึ้นทำไมมันถึงได้ช้าลง ทำไมมันถึงไม่เร็วเหมือนตอน Dev กันนะ วันนี้มาดูวิธีออกแบบ Query และ Table ให้สามารถ Access ได้เร็วอย่างที่ควรจะเป็นกัน

<!--more-->

## Orderable/Sortable Indexes
Keys, Indexes ต้องสามารถเรียงลำดับกันได้เสมอ เอาง่าย ๆ ว่าไม่สุ่มนั่นแหละ

## Primary Keys
ใช้ primary keys เป็นคีย์หลักแยกข้อมูลในแถวเสมอ อ้อ อย่าไปมักง่ายใช้ **auto_increment** นะ เพราะถ้าได้สเกลขึ้นไปใช้ distributed database ชีวิตจะลำบากเอา ไปใช้พวก ULID, Snowflake IDs, KSUID, UUIDv7 เถิด

## Field Length and Data Type
ใช้ประเภทของข้อมูลและขนาดของข้อมูลให้เหมาะ / พอดีกับข้อมูลที่ใช้งาน เช่น Index ให้ดีก็อย่าให้ยาวเกิน 64 ตัว, ไม่ใช่เอะอะก็ varchar(255), ถ้าใช้ JSON ก็[ทำ Index จากฟิลล์ ใน JSON](https://www.postgresql.org/docs/current/datatype-json.html#JSON-INDEXING) ที่จะเอาไปใช้ใน WHERE clause ด้วยจะได้ไม่ต้อง WHERE LIKE ใน JSON อะไรแบบนี้

## Data Quantity and Retention
เลือก access / query เฉพาะ ข้อมูลที่ต้องการใช้งานเท่านั้น ไม่ใช่กวาดไปทั้งตาราง ทั้ง ๆ ที่ใช้อยู่ไม่กี่สิบฟิลล์

## Searchable Arguments (SARGable Queries)
ใช้ Searchable arguments (SARGable) ใน query เสมอ แล้ว SARGable มันคืออัลไล ง่าย ๆ ก็คือตัวดำเนินการ (operators) ที่สามารถใช้งาน Indexes ได้ ซึ่งจะช่วยใน query เราเร็วขึ้น (เพราะมันไม่ต้องไล่กวาดข้อมูลทั้งตารางมาเช็คอะ) ดำเนินการที่ใช้ประโยชน์จาก B+Tree index ได้ดี ยกตัวอย่าง
  - Equals, IN (=): ก็เท่ากับนั่นแหละแหละ เช่น `WHERE indexed_column = value`
  - Inequality operators (<, <=, >, >=): ตัวดำเนินการแนวเปรียบเทียบ เช่น `WHERE indexed_column > value` ไม่รวม **<>, !=** นะ
  - BETWEEN: เปรียบเทียบช่วงระหว่าง เช่น `WHERE indexed_column BETWEEN low_value AND high_value`
  - LIKE (เฉพาะ `prefix%` นะ): เช่น `WHERE indexed_column LIKE 'prefix%'` ส่วน `%suffix` ไม่ใช่นะ ถ้าจะใช้ `LIKE '%key word%'` ก็ควรย้ายไปใช้พวก database สำหรับ full text search ไป
  - IS NULL กับ IS NOT NULL: ที่เป็นดำเนินการ เช่น `WHERE indexed_column IS NULL` ส่วน `ISNULL(column)` ที่เป็น fn อันนี้ไม่ใช่นะ
  - DISTINCT: ใช้กับ **unique** indexed_column values เสมอนะ
  - EXISTS กับ NOT EXISTS: ใช้กับ indexed_column values เสมอนะ
  - CASE: ถ้าใช้กับการเปรียบเทียบตรง ๆ เช่น `CASE WHEN indexed_column = value THEN result END` เป็นใช้ได้
  - หลีกเลี่ยงการใช้ฟังก์ชันที่ไม่ใช่ **MATH fn** ใน WHERE clause เช่น `WHERE YEAR(column) BETWEEN low_value AND high_value` หรือ `WHERE LEFT(column, 1) = 'K'`
  - JOIN ON, USING: ด้วยเงื่อนไขที่ว่ามาข้างต้นเสมอ

## Concurrent Activities
ในระบบที่มีการทำงานพร้อม ๆ กัน ถ้าหลงไปออกแบบแล้วมีการอ่านเขียนลงใน ฟิลล์เดียวกันพร้อม ๆ กับจะช้าเป็นธรรมดา ถ้าไม่สามารถออกแบบอื่นได้แล้วจริง ๆ ก็ใช้ queue มาช่วยบรรเทาได้

## Data Staging
ถ้ามี query ที่ใหญ่เบิ้มจน query time out การแบ่งซอย query ใหญ่ ๆ ให้เล็กลงมาแล้วค่อยเอามารวมกัน ในบางกรณีนอกจากจะอ่านเข้าใจง่ายแล้วยังช่วยให้ได้ประสิทธิภาพเพิ่มขึ้นด้วยนะเอ้อ แต่จริง ๆ ถ้า query มันดูแล้วเหมือนจะเป็น code program แสดงว่ามันไม่ใช่แล้วอะ เอาออกมาทำเป็น program เถอะ

## Flexible Column
ถ้าข้อมูลที่มีการปรับเปลี่ยนโครงสร้างตารางบ่อย ๆ หรือ Column ในตารางมันยาวสุดลูกหูลูกตา ก็ย้ายไปใช้ **NoSQL** จะเหมาะกว่า