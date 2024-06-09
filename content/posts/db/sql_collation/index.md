---
title: "COLLATE ใน Database มันคืออิหยังกันนะ แล้วมันทำอะไรได้"
subtitle: ""
date: 2023-09-08T10:24:29+07:00
lastmod: 2023-09-08T10:24:29+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: ""
license: ""
images: []
featuredImage: "featured-image.webp"
featuredImagePreview: "featured-image.webp"

tags: ["Database", "SQL"]
categories: ["SQL"]

lightgallery: true
---

<!--more-->

## COLLATE คือ
เวลาสร้าง Database หรือ Table เราจะกำหนด CHARACTER SET และ COLLATE ยกตัวอย่างของ MariaDB เช่น `utf8mb4_unicode_ci` ประกอบด้วย

### Character set
utf8mb4 หมายถึงใช้ character set ที่ encoding เป็น UTF-8 แบบเต็ม 4 bytes (utf8 เฉย ๆ ของรุ่นเก่า ๆ คือ UTF-8 แบบ 3 bytes ซึ่งจะไม่รองรับ emoji กับ unicode แบบเต็มรูปแบบ)

### Collation Algorithm
unicode หมายถึงใช้การเรียงลำดับ (ค่าของ) ตัวอักษรตาม unicode

### Diacritical sensitivity
- ai (ancient insensitive) ตัวอย่าง a == ᾰ
- as (ancient sensitive) ตัวอย่าง a != ᾰ
- ci (case insensitive) ตัวอย่าง A == a
- cs (case sensitive) ตัวอย่าง A != a

## Usecase
ยกตัวอย่างที่ง่ายที่สุดน่าจะเป็นการค้นหาชื่อไม่ว่าจะคนหรือสถานที่เช่น

```sql
SELECT
  users._id,
  users.first_name,
  users.last_name
FROM
  users
WHERE
  LOWER(users.first_name) = 'kawin';
```

ผลที่ได้

```json
[
  {
    "_id": 5,
    "first_name": "KAWIN",
    "last_name": "eiei"
  },
  {
    "_id": 4,
    "first_name": "kawin",
    "last_name": "naja"
  },
  {
    "_id": 3,
    "first_name": "Kawin",
    "last_name": "Viriyaprasopsook"
  }
]
```

เราก็จะได้มาทั้ง `Kawin`, `kawin`, `KAWIN` ซึ่งดูเหมือนจะได้ผลตามปกติ แต่ถ้าลอง EXPLAIN ดู

```sql
EXPLAIN FORMAT=JSON
SELECT
  users._id,
  users.first_name,
  users.last_name
FROM
  users
WHERE
  LOWER(users.first_name) = 'kawin';
```

จะพบว่า query เราอ่านไปทั้งหมด 1000 rows ในการค้นหาเลย

```json
{
  "query_block": {
    "select_id": 1,
    "nested_loop": [
      {
        "table": {
          "table_name": "users",
          "access_type": "index",
          "key": "udx_full_name",
          "key_length": "518",
          "used_key_parts": ["first_name", "last_name"],
          "rows": 1000,
          "filtered": 100,
          "attached_condition": "lcase(users.first_name) = 'kawin'",
          "using_index": true
        }
      }
    ]
  }
}
```

สิ่งนี้เรียกว่า **index obscure** ก็คือ database มองว่า where จาก ผลของ fn(column) ทำให้ต้อง**ไล่ LOWER() ไปจนสุดตาราง** ซึ่งถ้าในตอนที่เราสร้างตาราง/คอลัมน์ เป็นแบบ case insensitive อยู่แล้วเราก็สามารถใช้ `=` ได้เลย

```sql
EXPLAIN FORMAT=JSON
SELECT
  users._id,
  users.first_name,
  users.last_name
FROM
  users
WHERE
  users.first_name = 'kawin';
```

ซึง EXPLAIN ออกมาจะได้ตามนี้ อ่านไปแค่ 3 rows แทนที่จะอ่านจากทั้งตาราง ซึ่งก็ได้ผลออกมาเหมือนกับใช้ LOWER() ครอบเลย

```json
{
  "query_block": {
    "select_id": 1,
    "nested_loop": [
      {
        "table": {
          "table_name": "users",
          "access_type": "ref",
          "possible_keys": ["udx_full_name"],
          "key": "udx_full_name",
          "key_length": "259",
          "used_key_parts": ["first_name"],
          "ref": ["const"],
          "rows": 3,
          "filtered": 100,
          "attached_condition": "users.first_name = 'kawin'",
          "using_index": true
        }
      }
    ]
  }
}
```

{{< admonition warning >}}
เนื่องจากจะตัวเล็กจะตัวใหญ่ มีค่าเท่ากันต้องระวังในการใช้ unique เพราะ database จะมองว่า Kawin ซ้ำกับ kawin นั่นเอง
{{< /admonition >}}