---
title: "GO กับปัญหาโลกแตก Value หรือ Pointer"
subtitle: ""
date: 2023-07-31T08:33:40+07:00
lastmod: 2023-07-31T08:33:40+07:00
draft: true
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin-vir.pages.dev"
description: "ในเวลาที่เขียนภาษา GO มักจะมีคำถามนึงโผล่มาเสมอ คือตกลง func นี้จะใช้ Value หรือ Pointer ดีนะ เดี๋ยวเรามาดูความแตกต่างกัน"
license: ""
images: []
resources:
- name: "featured-image"
  src: "go-featured-image.webp"

tags: []
categories: []

featuredImage: "go-featured-image.webp"
featuredImagePreview: "featured-image"

lightgallery: true

---

ในเวลาที่เขียนภาษา GO มักจะมีคำถามนึงโผล่มาเสมอ คือตกลง func นี้จะใช้ Value หรือ Pointer ดีนะ เดี๋ยวเรามาดูความแตกต่างกัน

<!--more-->

## tl;dr
GO เป็นภาษาที่ copy by default เพราะฉะนั้นใช้ pass by value เป็น default ได้เลย (ยกเว้น เคสบาางอย่างซึ่งจะอธิบายต่อด้านล่างครับ)

## ทบทวนความรู้

### stack
คือ โครงสร้างข้อมูลแบบเรียงซ้อนต่อกันซึ่งเวลาทำงานมันก็ทำงานแบบ **มาทีหลัง ออกไปก่อน** (มันถึงได้ชื่อ stack แหละ)

### heap
คือ โครงสร้างข้อมูบแบบลำดับตามความสำคัญซึ่งอาศัย binary tree ในการจัดเรียง

## GO complier

## สรุป
รายมาซะยาวเลย มาดูหลักฐานเชิงประจักษ์กันดีกว่า

### benchmark

### เมื่อไหร่ควรใช้ pointer
- 

### เมื่อไหร่ควรใช้ value
- เมื่อไม่ตรงกับเงื่อนไขของ `เมื่อไหร่ควรใช้ pointer` **ถถถ**