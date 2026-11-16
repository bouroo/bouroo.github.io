---
title: "GO: io.ReadAll vs io.Copy"
subtitle: ""
date: 2023-12-10T18:33:40+07:00
lastmod: 2023-12-10T21:33:40+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "เปรียบเทียบ io.ReadAll กับ io.Copy ใน Go พร้อมผล benchmark ไฟล์ JSON ขนาด 10KB, 2.9MB, 26MB: io.Copy เร็วกว่าไฟล์สองขนาดเล็กแต่ช้ากว่าไฟล์ 26MB และ allocate น้อยกว่าทุกขนาด"
license: ""
images: []
featuredImage: "featured-image.svg"
featuredImagePreview: "featured-image.svg"

tags: ["Go", "Buffer"]
categories: ["Go"]

lightgallery: true

---

การเขียน Go บ่อยครั้งเราจะเจอการที่เราต้องเปิดอ่านไฟล์หรืออ่าน Response จากการดึง API ซึ่งส่วนใหญ่ โดยทั่วไปแล้วก็จะใช้ io.ReadAll และ io.Copy กัน ตอนนี้จะพาไปดูว่ามันแตกต่างกันอย่างไร

<!--more-->

## io.ReadAll
`io.ReadAll` เป็นฟังก์ชันที่ใช้สำหรับอ่านข้อมูลจากที่อ่านได้ (Readers) ทั้งหมดและส่งกลับข้อมูลในรูปแบบของ byte slice (เช่น []byte) ที่มีขนาดคงที่ของข้อมูลทั้งหมดที่ถูกอ่านออกมาในหน่วยความจำ โดยการใช้ `io.ReadAll` อาจเหมาะสำหรับการอ่านข้อมูลที่มีขนาดเล็ก เพราะจะได้ byte slice มาใช้งานเลย แต่เนื่องจากการโยนข้อมูลทั้งหมดเก็บไว้ในหน่วยความจำ ทำให้ไม่เหมาะกับการอ่านข้อมูลขนาดใหญ่
### ตัวอย่าง
```go
file, _ := os.OpenFile("small-file.json", os.O_RDONLY, 0664)
defer file.Close()
bodyBytes, err := io.ReadAll(file)
```

## io.Copy
`io.Copy` ใช้สำหรับการคัดลอกข้อมูลจาก Reader ไปยัง Writer โดยไม่จำเป็นต้องเก็บข้อมูลทั้งหมดในหน่วยความจำ เมื่อมีข้อมูลถูกส่งมาจาก Reader มันจะทำการคัดลอกข้อมูลนั้นไปยัง Writer ทีละส่วน ซึ่งทำให้สามารถทำงานกับข้อมูลที่มีขนาดใหญ่มาก ๆ ได้โดยไม่เป็นภาระต่อหน่วยความจำ
### ตัวอย่าง
```go
file, _ := os.OpenFile("small-file.json", os.O_RDONLY, 0664)
defer file.Close()
bytesBuff := new(bytes.Buffer)
_,err := io.Copy(bytesBuff, file)
bodyBytes := bytesBuff.Bytes()
```

## เปรียบเทียบการทำงาน

### เขียนการทดสอบ
โดยจะทดสอบด้วยการทำ Benchmark เทียบการอ่านไฟล์ JSON
```go
import (
	"os"
	"bytes"
	"testing"
)

func BenchmarkReadAll(b *testing.B) {

	b.ReportAllocs()
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			file, _ := os.Open("file.json")
			defer file.Close()
			io.ReadAll(file)
		}
	})
}

func BenchmarkCopy(b *testing.B) {

	b.ReportAllocs()
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			file, _ := os.Open("file.json")
			defer file.Close()
			bytesBuff := new(bytes.Buffer)
			io.Copy(bytesBuff, file)
		}
	})
}
```

### ผลการทดสอบ
ผลการทดสอบอ่านไฟล์ small 10KB, medium 2.9MB, large 26MB

|Name|Loops Executed|Time Taken per Iteration|Bytes Allocated per Operation|Allocations per Operation|
|---|---|---|---|---|
|BenchmarkReadAllSmall-8|            45658|             24383 ns/op|           46296 B/op|         14 allocs/op|
|BenchmarkCopySmall-8|               64654|             16235 ns/op|           30938 B/op|         11 allocs/op|
|BenchmarkReadAllMedium-8|            1510|            734884 ns/op|        16792061 B/op|         37 allocs/op|
|BenchmarkCopyMedium-8|               3433|            333917 ns/op|         8388372 B/op|         20 allocs/op|
|BenchmarkReadAllLarge-8|              171|           6335655 ns/op|        160741794 B/op|        46 allocs/op|
|BenchmarkCopyLarge-8|                 237|           7064719 ns/op|        67108578 B/op|         22 allocs/op|

`io.Copy` ใช้เวลา 16,235 ns/op เทียบกับ 24,383 ของ `io.ReadAll` บนไฟล์ 10KB (เร็วกว่า 33%) และ 333,917 เทียบกับ 734,884 บนไฟล์ 2.9MB (เร็วกว่า 55%) แต่บนไฟล์ 26MB กลับช้ากว่า 11% (7,064,719 เทียบกับ 6,335,655) ส่วน memory ที่ allocate น้อยกว่าทุกขนาด: 31KB, 8.4MB, 67MB เทียบกับ 46KB, 16.8MB, 161MB ข้อแลกเปลี่ยนคือต้องเขียนโค้ดเพิ่ม Buffer มารับข้อมูล ซึ่งอาจจะไม่ถูกจริตสายขี้เกียจสักเท่าไหร่