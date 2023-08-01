---
title: "Go กับการ re-use memory ด้วย sync.Pool"
subtitle: ""
date: 2023-08-01T20:31:34+07:00
lastmod: 2023-08-01T20:31:34+07:00
draft: true
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin-vir.pages.dev"
description: "ลดการจอง memoy ใน Go ด้วยการ re-use memory โดยใช้ sync.Pool"
license: ""
images: []
resources:
- name: "featured-image"
  src: "go-featured-image.webp"

tags: ["Go", "Memory Pooling"]
categories: ["Programing"]

featuredImage: "go-featured-image.webp"
featuredImagePreview: "featured-image"

lightgallery: true
---


<!--more-->

## หน้าตาของ sync.Pool{}
```go
type SomeStruct struct {
	Arg1 int // 8 bytes
}

var somePool = sync.Pool{
	New: func() any {
		// create new SomeStruct instance
		return &SomeStruct{}
	},
}
```

### สาธิตวิธีการใช้งาน
```go
func DoSomethingWithPool() {
	var sample *SomeStruct

	for round := 0; round < 10000; round++ {
		// get instance from pool
		sample = somePool.Get().(*SomeStruct)
		// set some value
		sample.AutoIncrease()
		// return instance to the pool
		somePool.Put(sample)
	}

  _ = fmt.sprint("%v", sample)
}

func DoSomethingWithOutPool() {
	var sample *SomeStruct

	for round := 0; round < 10000; round++ {
		// init new pointer
		sample = new(SomeStruct)
		// set some value
		sample.AutoIncrease()
	}

}
```
ผลงานเปรียบเทียบ


## ข้อควรระวัง
ถ้าไม่ `put` กลับคืน pool จะทำให้ทุกครั้งที่ `get` จะเป็นการสร้าง instance ใหม่ไปเรื่อย ๆ ทำให้ระบบโดยรวมสิ้นเปลืองมากขึ้น (เพราะต้องไป allocate heap เพิ่มขึ้นเรื่อย ๆ) สามรถใช้ `defer put` ทันทีไว้หลังจาก `get` ช่วยให้ไม่ลืม `put` กลับได้

## สรุป
- sync.Pool เหมาะกับ
  - งานที่ทำงานซ้ำๆ และต้องการ concurrent safe