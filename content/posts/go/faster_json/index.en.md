---
title: "Give JSON in Go a Jet Engine"
subtitle: ""
date: 2023-12-11T10:00:40+07:00
lastmod: 2026-10-06T19:00:40+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "Benchmarks goccy/go-json against Go's standard encoding/json and shows roughly 10x speedups, with a 2025-2026 update on bytedance/sonic and the experimental encoding/json/v2."
license: ""
images: []
featuredImage: "featured-image.svg"
featuredImagePreview: "featured-image.svg"

tags: ["Go", "JSON"]
categories: ["Go"]

lightgallery: true

---

A common task when working with REST APIs is converting JSON back and forth between services. Typically, `encoding/json` is used, which is the standard library provided in Go. But now, there's something new to try: `goccy/go-json`, which will make our services faster without any extra cost.

<!--more-->

## encoding/json
`encoding/json` is the standard library in Go that can be used to convert data between Go data structures (structs, slices, maps) and the JSON we are all familiar with.

## goccy/go-json
[goccy/go-json](https://github.com/goccy/go-json) is a library developed to provide high speed and efficiency in handling JSON in Go. It is capable of handling larger data and is [optimized to increase the speed of data conversion with a jet engine](https://github.com/goccy/go-json#how-it-works), while still being compatible with `encoding/json`.

### Writing the test
We will test by benchmarking the reading of a JSON file (you can expand to see what is being tested).
```go
package main_test

import (
	"encoding/json"
	"os"
	"testing"

	// You can actually use "github.com/goccy/go-json" directly
	// to replace "encoding/json"
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

### Test Results
Test results for reading a small 10KB file.

|Name|Loops Executed|Time Taken per Iteration|Bytes Allocated per Operation|Allocations per Operation|
|---|---|---|---|---|
|BenchmarkGoSTDUnmarshal-8|          65835|             24631 ns/op|           10872 B/op|         11 allocs/op|
|BenchmarkGoCcyUnmarshal-8|         103197|             10113 ns/op|           20973 B/op|         11 allocs/op|
|BenchmarkGoSTDDecoder-8|            27882|             39565 ns/op|           31440 B/op|         17 allocs/op|
|BenchmarkGoCcyDecoder-8|          1219350|               890.3 ns/op|           863 B/op|          9 allocs/op|


Test results for reading a medium 2.9MB file.

|Name|Loops Executed|Time Taken per Iteration|Bytes Allocated per Operation|Allocations per Operation|
|---|---|---|---|---|
|BenchmarkGoSTDUnmarshal-8|            225|           4509768 ns/op|         2925207 B/op|         11 allocs/op|
|BenchmarkGoCcyUnmarshal-8|           5428|            219836 ns/op|         5849958 B/op|         13 allocs/op|
|BenchmarkGoSTDDecoder-8|              232|           4739039 ns/op|         8387297 B/op|         26 allocs/op|
|BenchmarkGoCcyDecoder-8|          1248626|               890.3 ns/op|           871 B/op|          9 allocs/op|

Test results for reading a large 26MB file.

|Name|Loops Executed|Time Taken per Iteration|Bytes Allocated per Operation|Allocations per Operation|
|---|---|---|---|---|
|BenchmarkGoSTDUnmarshal-8|             13|          77841000 ns/op|        26150382 B/op|         16 allocs/op|
|BenchmarkGoCcyUnmarshal-8|            264|           4654911 ns/op|        52298626 B/op|         13 allocs/op|
|BenchmarkGoSTDDecoder-8|               16|          63935862 ns/op|        67107820 B/op|         33 allocs/op|
|BenchmarkGoCcyDecoder-8|          1200399|               877.8 ns/op|           863 B/op|          9 allocs/op|

## Current landscape (2025-2026)

The JSON library landscape has kept moving since this benchmark was written:

- **[bytedance/sonic](https://github.com/bytedance/sonic)** the fastest option for large payloads on **amd64**, thanks to its SIMD/JIT path (inspired by simdjson); it is a drop-in replacement for `encoding/json`, but on arm64 it falls back to a compatibility implementation (see the 2026 re-test below).
- **[goccy/go-json](https://github.com/goccy/go-json)** still a solid, easy drop-in; the benchmarks above remain representative.
- **[encoding/json/v2](https://github.com/golang/go/issues/71707)** Go's own rewrite. It shipped as an **experimental** preview in Go 1.25 (enable with `GOEXPERIMENT=jsonv2`) and is still not stable in Go 1.26. Once finalized it will narrow the gap with the third-party libraries from inside the standard library.

Pick by workload: `sonic` for the absolute fastest large-payload path, `goccy/go-json` for a zero-config speedup, and keep an eye on `encoding/json/v2` so you can drop the dependency entirely once it stabilizes.

### Actually testing sonic (2026)

Run on an Apple M4 Pro (arm64), 14 threads, Go 1.26.8, sonic v1.15.4, goccy v0.11.2 — same payload sizes (10KB / 2.9MB / 26MB) and the same parallel structure as 2023, but with the 2023 flaw fixed: that run opened files with `defer file.Close()` inside the `RunParallel` loop and ignored every error. The 2023 Decoder rows that came in under 1µs on MB-sized files were a file-descriptor-leak artifact, not real numbers, so **do not compare them** with this new set. Here every file is closed per iteration and every error is checked (`b.Fatal`).

The only additions to the original snippet (sonic side):

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

Actual results for the small 10KB file.

|Name|Loops Executed|Time Taken per Iteration|Bytes Allocated per Operation|Allocations per Operation|
|---|---|---|---|---|
|BenchmarkSTDUnmarshal_Small-14|21969|51945 ns/op|61447 B/op|1706 allocs/op|
|BenchmarkGoccyUnmarshal_Small-14|28645|41651 ns/op|59430 B/op|299 allocs/op|
|BenchmarkSonicUnmarshal_Small-14|41011|30200 ns/op|97643 B/op|149 allocs/op|
|BenchmarkSTDDecoder_Small-14|16906|66874 ns/op|81319 B/op|1712 allocs/op|
|BenchmarkGoccyDecoder_Small-14|18993|63769 ns/op|91726 B/op|310 allocs/op|
|BenchmarkSonicDecoder_Small-14|61672|18963 ns/op|70603 B/op|148 allocs/op|

Actual results for the medium 2.9MB file.

|Name|Loops Executed|Time Taken per Iteration|Bytes Allocated per Operation|Allocations per Operation|
|---|---|---|---|---|
|BenchmarkSTDUnmarshal_Medium-14|346|2977370 ns/op|16746951 B/op|471641 allocs/op|
|BenchmarkGoccyUnmarshal_Medium-14|778|1399864 ns/op|14628227 B/op|76679 allocs/op|
|BenchmarkSonicUnmarshal_Medium-14|1208|943860 ns/op|36031420 B/op|36307 allocs/op|
|BenchmarkSTDDecoder_Medium-14|355|3132189 ns/op|22086104 B/op|471655 allocs/op|
|BenchmarkGoccyDecoder_Medium-14|754|1395477 ns/op|21115091 B/op|76591 allocs/op|
|BenchmarkSonicDecoder_Medium-14|846|1193419 ns/op|45980991 B/op|36325 allocs/op|

Actual results for the large 26MB file.

|Name|Loops Executed|Time Taken per Iteration|Bytes Allocated per Operation|Allocations per Operation|
|---|---|---|---|---|
|BenchmarkSTDUnmarshal_Large-14|33|30685900 ns/op|152241174 B/op|4244135 allocs/op|
|BenchmarkGoccyUnmarshal_Large-14|105|14059384 ns/op|132670200 B/op|689879 allocs/op|
|BenchmarkSonicUnmarshal_Large-14|98|14073981 ns/op|367612325 B/op|326560 allocs/op|
|BenchmarkSTDDecoder_Large-14|30|34076865 ns/op|191421457 B/op|4244152 allocs/op|
|BenchmarkGoccyDecoder_Large-14|108|12868000 ns/op|134777234 B/op|688851 allocs/op|
|BenchmarkSonicDecoder_Large-14|66|15304603 ns/op|435962738 B/op|326583 allocs/op|

What the numbers say: on this arm64 machine sonic runs the compatibility/optdec path, not the JIT, so it lands at roughly goccy level — a small win on small-to-medium payloads, but on the 26MB file it ties or loses. The large README lead comes from the JIT/SIMD path that only exists on amd64; deploy on amd64 servers to get the full benefit.

You will find that `goccy/go-json` is on average 10X++ more performant than `encoding/json`, especially when using `Encoder/Decoder` instead of `Marshal/Unmarshal`, which is not even comparable.
