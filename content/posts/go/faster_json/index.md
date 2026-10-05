---
title: "ติดไอพ่นให้ JSON ใน Go"
subtitle: ""
date: 2023-12-11T10:00:40+07:00
lastmod: 2026-10-06T19:00:40+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "ทำ benchmark เทียบ goccy/go-json กับ encoding/json ของ standard library พบว่าเร็วขึ้นราว 10 เท่า พร้อมอัปเดตสถานการณ์ 2025-2026 ของ bytedance/sonic และ encoding/json/v2 ที่ยังเป็น experimental"
license: ""
images: []
featuredImage: "featured-image.svg"
featuredImagePreview: "featured-image.svg"

tags: ["Go", "JSON"]
categories: ["Go"]

lightgallery: true

---

สิ่งที่ต้องเจอบ่อย ๆ เวลาทำงานกับ REST API นั่นคือการแปลง JSON ไปมาระหว่าง services โดบปกติแล้วก็จะใช้ `encoding/json` กันซึ่งเป็นไลบรารีมาตรฐานที่มีให้ใน Go แต่ตอนนี้มีของจะมาแนะนำให้ลองกัน นั่นคือ `goccy/go-json` ที่จะทำให้ services เราเร็วขึ้นโดยไม่ต้องจ่ายตังเพิ่ม

<!--more-->

## encoding/json
`encoding/json` เป็นไลบรารีมาตรฐานที่มีให้ใน Go ที่สามารถใช้ในการแปลงข้อมูลระหว่างโครงสร้างข้อมูล Go (structs, slices, maps) กับ JSON ที่เราคุ้นเคยกันดี

## goccy/go-json
[goccy/go-json](https://github.com/goccy/go-json) เป็นไลบรารีที่ถูกพัฒนาขึ้นเพื่อให้ความเร็วและประสิทธิภาพสูงในการจัดการ JSON ใน Go โดยเฉพาะ มีความสามารถในการจัดการกับข้อมูลที่ใหญ่มากขึ้น และมีการ [Optimize เพื่อเพิ่มประสิทธิภาพในการแปลงข้อมูลให้เร็วขึ้นอย่างมาก](https://github.com/goccy/go-json#how-it-works) โดยที่ยัง complatible กับ `encoding/json` อยู่

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

## สถานการณ์ปัจจุบัน (2025-2026)

วงการ JSON library เคลื่อนไหวต่อเนื่องตั้งแต่ที่เขียน benchmark นี้:

- **[bytedance/sonic](https://github.com/bytedance/sonic)** เร็วที่สุดสำหรับ payload ขนาดใหญ่บน **amd64** ด้วย SIMD/JIT (ได้แรงบันดาลใจจาก simdjson) เป็น drop-in replacement ของ `encoding/json` — แต่บน arm64 จะ fallback ไปใช้ implementation สำรอง (ดูผลทดสอบจริงปี 2026 ด้านล่าง)
- **[goccy/go-json](https://github.com/goccy/go-json)** ยังเป็นตัวเลือก drop-in ที่ดี ใช้ง่าย benchmark ด้านบนยังเป็นตัวแทนที่เชื่อถือได้
- **[encoding/json/v2](https://github.com/golang/go/issues/71707)** การเขียนใหม่ของ Go เอง ออกมาเป็นตัว **experimental** ใน Go 1.25 (เปิดด้วย `GOEXPERIMENT=jsonv2`) และยังไม่ stable ใน Go 1.26 เมื่อเสร็จสมบูรณ์จะช่วยลดช่องว่างกับ library ของ third-party จากภายใน standard library เลย

เลือกตามลักษณะงาน: `sonic` สำหรับ path ที่เร็วที่สุดบน payload ขนาดใหญ่, `goccy/go-json` สำหรับ speed up แบบ zero-config และติดตาม `encoding/json/v2` เพื่อจะได้ถอด dependency ออกได้เลยเมื่อมัน stable

### ผลทดสอบจริงกับ sonic (2026)

รันบน Apple M4 Pro (arm64) 14 threads, Go 1.26.8, sonic v1.15.4, goccy v0.11.2 — ใช้ payload ขนาด 10KB / 2.9MB / 26MB และโครงสร้าง parallel เหมือนปี 2023 แต่แก้จุดที่ตารางปี 2023 เปิดไฟล์ค้างไว้ด้วย `defer file.Close()` ในลูป `RunParallel` แล้วปล่อย error ทิ้ง ตัวเลข Decoder ที่ต่ำกว่า 1µs บนไฟล์ MB เลยเป็น artifact ของ file descriptor leak ไม่ใช่ของจริง ดังนั้น **อย่านำไปเทียบ** กับตารางชุดใหม่นี้ รอบนี้ปิดไฟล์ทุก iteration และเช็ค error ทุกจุด (`b.Fatal`)

โค้ดส่วนที่เพิ่มจากเดิม (เฉพาะฝั่ง sonic):

```go
import (
	"os"
	"testing"

	"github.com/bytedance/sonic"
)

func BenchmarkSonicUnmarshal(b *testing.B) {
	b.ReportAllocs()
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			resp := make(map[string]interface{})
			file, err := os.ReadFile("file.json")
			if err != nil {
				b.Fatal(err)
			}
			if err := sonic.Unmarshal(file, &resp); err != nil {
				b.Fatal(err)
			}
		}
	})
}

func BenchmarkSonicDecoder(b *testing.B) {
	b.ReportAllocs()
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			resp := make(map[string]interface{})
			file, err := os.Open("file.json")
			if err != nil {
				b.Fatal(err)
			}
			if err := sonic.ConfigDefault.NewDecoder(file).Decode(&resp); err != nil {
				b.Fatal(err)
			}
			file.Close()
		}
	})
}
```

ผลทดสอบจริง small 10KB

|Name|Loops Executed|Time Taken per Iteration|Bytes Allocated per Operation|Allocations per Operation|
|---|---|---|---|---|
|BenchmarkSTDUnmarshal_Small-14|21969|51945 ns/op|61447 B/op|1706 allocs/op|
|BenchmarkGoccyUnmarshal_Small-14|28645|41651 ns/op|59430 B/op|299 allocs/op|
|BenchmarkSonicUnmarshal_Small-14|41011|30200 ns/op|97643 B/op|149 allocs/op|
|BenchmarkSTDDecoder_Small-14|16906|66874 ns/op|81319 B/op|1712 allocs/op|
|BenchmarkGoccyDecoder_Small-14|18993|63769 ns/op|91726 B/op|310 allocs/op|
|BenchmarkSonicDecoder_Small-14|61672|18963 ns/op|70603 B/op|148 allocs/op|

ผลทดสอบจริง medium 2.9MB

|Name|Loops Executed|Time Taken per Iteration|Bytes Allocated per Operation|Allocations per Operation|
|---|---|---|---|---|
|BenchmarkSTDUnmarshal_Medium-14|346|2977370 ns/op|16746951 B/op|471641 allocs/op|
|BenchmarkGoccyUnmarshal_Medium-14|778|1399864 ns/op|14628227 B/op|76679 allocs/op|
|BenchmarkSonicUnmarshal_Medium-14|1208|943860 ns/op|36031420 B/op|36307 allocs/op|
|BenchmarkSTDDecoder_Medium-14|355|3132189 ns/op|22086104 B/op|471655 allocs/op|
|BenchmarkGoccyDecoder_Medium-14|754|1395477 ns/op|21115091 B/op|76591 allocs/op|
|BenchmarkSonicDecoder_Medium-14|846|1193419 ns/op|45980991 B/op|36325 allocs/op|

ผลทดสอบจริง large 26MB

|Name|Loops Executed|Time Taken per Iteration|Bytes Allocated per Operation|Allocations per Operation|
|---|---|---|---|---|
|BenchmarkSTDUnmarshal_Large-14|33|30685900 ns/op|152241174 B/op|4244135 allocs/op|
|BenchmarkGoccyUnmarshal_Large-14|105|14059384 ns/op|132670200 B/op|689879 allocs/op|
|BenchmarkSonicUnmarshal_Large-14|98|14073981 ns/op|367612325 B/op|326560 allocs/op|
|BenchmarkSTDDecoder_Large-14|30|34076865 ns/op|191421457 B/op|4244152 allocs/op|
|BenchmarkGoccyDecoder_Large-14|108|12868000 ns/op|134777234 B/op|688851 allocs/op|
|BenchmarkSonicDecoder_Large-14|66|15304603 ns/op|435962738 B/op|326583 allocs/op|

สรุปจากตัวเลข: บน arm64 นี้ sonic วิ่งบน compatibility/optdec path (ไม่ใช่ JIT) จึงทำได้แค่ระดับเดียวกับ goccy — ชนะเล็กน้อยใน payload เล็ก-กลาง แต่พอไฟล์ระดับ 26MB กลับเสมอกันหรือช้ากว่า ส่วนความได้เปรียบแบบขาดลอยใน README มาจาก JIT/SIMD ที่มีเฉพาะ amd64 เท่านั้น ถ้า deploy บน server amd64 จึงจะได้ประโยชน์เต็ม

จะพบว่า `goccy/go-json` จะประสิทธิภาพดีกว่า `encoding/json` เฉลี่ยที่ 10X++ เลยทีเดียว โดยเฉพาะการใช้ `Encoder/Decoder` แทนการใช้ `Marshal/Unmarshal` ที่สามารถเรียกได้ว่าเทียบกันกันไม่ติดเลย
