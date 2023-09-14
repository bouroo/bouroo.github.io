---
title: "เมื่อต้องทำ pagination มันมีอะไรซุ่มรอเราอยู่"
subtitle: ""
date: 2023-09-08T10:26:39+07:00
lastmod: 2023-09-08T10:26:39+07:00
draft: true
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

lightgallery: true
---

<!--more-->

## Offset / Limit
เริ่มจากท่า classic ที่ทุกคนน่าจะเคยใช้กันอยู่แล้วคือ Offset / Limit นั่นเอง
```sql
SELECT
  *
FROM
  users
ORDER BY
  created_at
LIMIT
  20 OFFSET 800;
```

## Cursor
ต่อมาคือการใช้ Index Cursor
```sql
SELECT
  *
FROM
  users
WHERE
  _id > 800
ORDER BY
  created_at
LIMIT
  20;
```

## Offset / Limit + Deferred Joins
เนื่องจาก Offset / Limit ใช้งานสะดวกแต่ก็แลกมาด้วยความเปลือง เราสามารถลดความเปลืองลงได้ด้วยการลดขนาดการโยนข้อมูลทิ้งได้ด้วยการใช้ Deferred Joins นั่นเอง
```sql
SELECT
  *
FROM
  users
  INNER JOIN (
    SELECT
      _id
    FROM
      users
    ORDER BY
      created_at
    LIMIT
      20 OFFSET 800
  ) AS sub_users USING (_id)
ORDER BY
  created_at;
```