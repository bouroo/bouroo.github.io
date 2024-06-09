---
title: "ติดไอพ่นให้ JSON ใน Go"
subtitle: ""
date: 2023-12-11T10:00:40+07:00
lastmod: 2023-12-11T10:00:40+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: ""
license: ""
images: []
featuredImage: "/img/go-featured-image.webp"
featuredImagePreview: "/img/go-featured-image.webp"

tags: ["Go", "JSON"]
categories: ["Go"]

lightgallery: true

---

สิ่งที่ต้องเจอบ่อย ๆ เวลาทำงานกับ REST API นั่นคือการแปลง JSON ไปมาระหว่าง services โดบปกติแล้วก็จะใช้ `encoding/json` กันซึ่งเป็นไลบรารีมาตรฐานที่มีให้ใน Go แต่ตอนนี้มีของจะมาแนะนำให้ลองกัน นั่นคือ `goccy/go-json` ที่จะทำให้ services เราเร็วส์ขึ้นแบบไม่ต้องจ่ายตังเพิ่ม

<!--more-->

## encoding/json
`encoding/json` เป็นไลบรารีมาตรฐานที่มีให้ใน Go ที่สามารถใช้ในการแปลงข้อมูลระหว่างโครงสร้างข้อมูล Go (structs, slices, maps) กับ JSON ที่เราคุ้นเคยกันดี

## goccy/go-json
[goccy/go-json](https://github.com/goccy/go-json) เป็นไลบรารีที่ถูกพัฒนาขึ้นเพื่อให้ความเร็วและประสิทธิภาพสูงในการจัดการ JSON ใน Go โดยเฉพาะ มีความสามารถในการจัดการกับข้อมูลที่ใหญ่มากขึ้น และมีการ [Optimize เพื่อเพิ่มประสิทธิภาพในการแปลงข้อมูลแบบเร็วส์ติดไอพ่น](https://github.com/goccy/go-json#how-it-works) โดยที่ยัง complatible กับ `encoding/json` อยู่

### เขียนการทดสอบ
โดยจะทดสอบด้วยการทำ Benchmark เทียบการอ่านไฟล์ JSON (กดขยายดูได้นะว่าทดสอบไรมั้ง)
```go
package main_test

import (
	"encoding/json"
	"os"
	"testing"

	// จริง ๆ ใช้ "github.com/goccy/go-json" เฉย ๆ
	// แทนที่ "encoding/json" ได้เลย
	goccy "github.com/goccy/go-json"
)

func BenchmarkGoSTDUnmarshal(b *testing.B) {

	b.ReportAllocs()
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			resp := make(map[string]interface{})
			file, _ := os.ReadFile("file.json")
			json.Unmarshal(file, &resp)
		}
	})
}

func BenchmarkGoCcyUnmarshal(b *testing.B) {

	b.ReportAllocs()
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			resp := make(map[string]interface{})
			file, _ := os.ReadFile("file.json")
			goccy.Unmarshal(file, &resp)
		}
	})
}

func BenchmarkGoSTDDecoder(b *testing.B) {

	b.ReportAllocs()
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			resp := make(map[string]interface{})
			file, _ := os.Open("file.json")
			defer file.Close()
			json.NewDecoder(file).Decode(&resp)
		}
	})
}

func BenchmarkGoCcyDecoder(b *testing.B) {

	b.ReportAllocs()
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			resp := make(map[string]interface{})
			file, _ := os.Open("file.json")
			defer file.Close()
			goccy.NewDecoder(file).Decode(&resp)
		}
	})
}
```

### ผลการทดสอบ
ผลการทดสอบอ่านไฟล์ small 10KB

|Name|Loops Executed|Time Taken per Iteration|Bytes Allocated per Operation|Allocations per Operation|
|---|---|---|---|---|
|BenchmarkGoSTDUnmarshal-8|          65835|             24631 ns/op|           10872 B/op|         11 allocs/op|
|BenchmarkGoCcyUnmarshal-8|         103197|             10113 ns/op|           20973 B/op|         11 allocs/op|
|BenchmarkGoSTDDecoder-8|            27882|             39565 ns/op|           31440 B/op|         17 allocs/op|
|BenchmarkGoCcyDecoder-8|          1219350|               890.3 ns/op|           863 B/op|          9 allocs/op|


ผลการทดสอบอ่านไฟล์ medium 2.9MB

|Name|Loops Executed|Time Taken per Iteration|Bytes Allocated per Operation|Allocations per Operation|
|---|---|---|---|---|
|BenchmarkGoSTDUnmarshal-8|            225|           4509768 ns/op|         2925207 B/op|         11 allocs/op|
|BenchmarkGoCcyUnmarshal-8|           5428|            219836 ns/op|         5849958 B/op|         13 allocs/op|
|BenchmarkGoSTDDecoder-8|              232|           4739039 ns/op|         8387297 B/op|         26 allocs/op|
|BenchmarkGoCcyDecoder-8|          1248626|               890.3 ns/op|           871 B/op|          9 allocs/op|

ผลการทดสอบอ่านไฟล์ large 26MB

|Name|Loops Executed|Time Taken per Iteration|Bytes Allocated per Operation|Allocations per Operation|
|---|---|---|---|---|
|BenchmarkGoSTDUnmarshal-8|             13|          77841000 ns/op|        26150382 B/op|         16 allocs/op|
|BenchmarkGoCcyUnmarshal-8|            264|           4654911 ns/op|        52298626 B/op|         13 allocs/op|
|BenchmarkGoSTDDecoder-8|               16|          63935862 ns/op|        67107820 B/op|         33 allocs/op|
|BenchmarkGoCcyDecoder-8|          1200399|               877.8 ns/op|           863 B/op|          9 allocs/op|

จะพบว่า `goccy/go-json` จะประสิทธิภาพดีกว่า `encoding/json` เฉลี่ยที่ 10X++ เลยทีเดียว โดยเฉพาะการใช้ `Encoder/Decoder` แทนการใช้ `Marshal/Unmarshal` ที่สามารถเรียกได้ว่าเทียบกันกันไม่ติดเลย 🤣🤣🤣