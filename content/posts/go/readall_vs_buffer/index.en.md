---
title: "Go: io.ReadAll vs io.Copy"
subtitle: ""
date: 2023-12-10T18:33:40+07:00
lastmod: 2023-12-10T21:33:40+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: ""
license: ""
images: []
featuredImage: "featured-image.webp"
featuredImagePreview: "featured-image.webp"

tags: ["Go", "Buffer"]
categories: ["Go"]

lightgallery: true

---

When writing Go, we often encounter situations where we need to read files or responses from API calls. Generally, `io.ReadAll` and `io.Copy` are used for this. This post will explore the differences between them.

<!--more-->

## io.ReadAll
`io.ReadAll` is a function used to read all data from a Reader and return it as a byte slice (`[]byte`) of a fixed size containing all the data read into memory. Using `io.ReadAll` might be suitable for reading small amounts of data because you get a byte slice to work with directly. However, since it loads all the data into memory, it's not suitable for reading large files.
### Example
```go
file, _ := os.OpenFile("small-file.json", os.O_RDONLY, 0664)
defer file.Close()
bodyBytes, err := io.ReadAll(file)
```

## io.Copy
`io.Copy` is used for copying data from a Reader to a Writer without needing to store all the data in memory. When data is received from the Reader, it copies it to the Writer in chunks, allowing it to handle very large amounts of data without burdening memory.
### Example
```go
file, _ := os.OpenFile("small-file.json", os.O_RDONLY, 0664)
defer file.Close()
bytesBuff := new(bytes.Buffer)
_,err := io.Copy(bytesBuff, file)
bodyBytes := bytesBuff.Bytes()
```

## Performance Comparison

### Writing the test
We will test by benchmarking the reading of a JSON file.
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

### Test Results
Test results for reading small 10KB, medium 2.9MB, and large 26MB files.

|Name|Loops Executed|Time Taken per Iteration|Bytes Allocated per Operation|Allocations per Operation|
|---|---|---|---|---|
|BenchmarkReadAllSmall-8|            45658|             24383 ns/op|           46296 B/op|         14 allocs/op|
|BenchmarkCopySmall-8|               64654|             16235 ns/op|           30938 B/op|         11 allocs/op|
|BenchmarkReadAllMedium-8|            1510|            734884 ns/op|        16792061 B/op|         37 allocs/op|
|BenchmarkCopyMedium-8|               3433|            333917 ns/op|         8388372 B/op|         20 allocs/op|
|BenchmarkReadAllLarge-8|              171|           6335655 ns/op|        160741794 B/op|        46 allocs/op|
|BenchmarkCopyLarge-8|                 237|           7064719 ns/op|        67108578 B/op|         22 allocs/op|

We can see that `io.Copy` is, on average, about 40% more performant than `io.ReadAll`. However, this comes at the cost of writing more code to create a buffer to receive the data, which might not appeal to the lazy programmer 🤣🤣🤣
