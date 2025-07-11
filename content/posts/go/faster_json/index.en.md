---
title: "Give JSON in Go a Jet Engine"
subtitle: ""
date: 2023-12-11T10:00:40+07:00
lastmod: 2023-12-11T10:00:40+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: ""
license: ""
images: []
featuredImage: "/img/featured-image.webp"
featuredImagePreview: "/img/featured-image.webp"

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

You will find that `goccy/go-json` is on average 10X++ more performant than `encoding/json`, especially when using `Encoder/Decoder` instead of `Marshal/Unmarshal`, which is not even comparable 🤣🤣🤣
