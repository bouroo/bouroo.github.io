---
title: "Let's Try to Understand Zero-Allocation Programming in Go"
subtitle: ""
date: 2024-12-21T11:47:24+07:00
lastmod: 2024-12-21T11:47:24+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "Techniques for reducing heap allocations in Go to ease the garbage collector's workload, using arrays, sync.Pool, slice capacity management, strings.Builder, and stack-based returns."
license: ""
images: []

tags: []
categories: []

featuredImage: "featured-image.jpg"
featuredImagePreview: "featured-image.jpg"

lightgallery: true
---

One of the features of Go is its garbage collector (GC), which saves us developers from the headache of having to manage memory. However, this convenience comes at a cost: the time it takes for the GC to run. Although it's not a full stop-the-world event, [you can learn more about the Go GC here](https://tip.golang.org/doc/gc-guide). But for some latency-critical tasks, we need to help the GC work less, which means reducing heap allocations. This is where the question of how to reduce heap allocations without going overboard comes in.

<!--more-->

## Why avoid heap allocation?
Although Go's GC is designed for efficient and fast memory management, there are some things to keep in mind:
- Latency: Every GC cycle always has a slight delay, which can be a problem in systems that require very consistent response times.
- CPU resource: The GC needs the CPU to run, so some resources will be allocated to the GC.
- ~~Stop-The-World~~: Even though it's not a full-blown stop-the-world event, there will still be a slight delay, which can affect complex or very large systems.

We can help reduce the GC's workload by ~~not littering~~ reducing heap allocations.

## How to reduce heap allocation (as far as I know)
### 1. Use an array instead of a slice if you know the size in advance
Dynamic memory allocation (e.g., during runtime) often leads to heap allocations, which the GC will eventually have to reclaim. Instead of creating new slices or buffers at runtime, pre-allocating a reusable array can help reduce heap allocations.

{{< admonition example >}}
```go
package main

// Force inputs to have 10 elements and create an output with 10 elements
func doubleTenValue(inputs [10]int) (result [10]int) {
	for i, input := range inputs {
		// Process the buffer
		result[i] = input * 2
	}

	return
}

func main() {
	inputs := [...]int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	result := doubleTenValue(inputs)

	// Print the result
	for _, v := range result {
		println(v)
	}
}
```
{{< /admonition >}}
#### Benefits:
- Reduces repeated heap allocations.
- The buffer is reused, not recreated, which reduces the garbage collector's workload.

### 2. Use sync.Pool for reuse
sync.Pool is an effective tool for managing expensive, temporary objects that are frequently used and discarded. By using a pool, we can reuse them and reduce the amount of garbage that needs to be collected.

{{< admonition example >}}
```go
package main

import (
	"bytes"
	"sync"
)

var bufPool = sync.Pool{
	New: func() interface{} {
		return new(bytes.Buffer)
	},
}

func buildString(s ...string) string {
	// Get a buffer from the pool
	buf := bufPool.Get().(*bytes.Buffer)
	// Return it to the pool when done
	defer func() {
		buf.Reset()
		bufPool.Put(buf)
	}()

	for _, str := range s {
		buf.WriteString(str)
	}

	return buf.String()
}

func main() {
	result := buildString("a", "b", "c")
	println(result)
}
```
{{< /admonition >}}
#### Benefits:
- Significantly reduces the number of memory allocations based on the frequency of creation.
- The buffer is reused, which can greatly reduce the GC's workload.

### 3. Manage slice capacity
Slices in Go are very useful, but their ability to grow dynamically often results in repeated memory allocations and moves to accommodate the slice's size at runtime. By managing the slice's capacity to avoid unnecessary resizing, we can keep slices on the stack instead of the heap.

{{< admonition example >}}
```go
package main

func appendData(inputs []int) []int {
    // Create a slice with enough capacity in advance
	result := make([]int, 0, len(inputs))

	for _, val := range inputs {
        // Add a value to the slice without expanding the capacity
		result = append(result, val * 2)
	}

	return result
}

func main() {
	inputs := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	result := appendData(inputs)

	// Print the result
	for _, v := range result {
		println(v)
	}
}
```
{{< /admonition >}}
#### Benefits:
- By initializing the result with a pre-defined capacity (len(inputs)),
- we avoid new memory allocations and reduce the chance of allocations from an increasing heap size.

### 4. Strings in Go are immutable
This means that every time a string is modified, a new string is created. To avoid frequent memory allocations for strings, we should:
- Use strings.Builder for string concatenation.
- Avoid using `+` for concatenating multiple strings in a loop.

{{< admonition example >}}
```go
package main

import "strings"

func buildMessage(parts []string) string {
	var builder strings.Builder
	// Allocate enough memory for the intended use
	builder.Grow(len(parts))

	for _, part := range parts {
		builder.WriteString(part)
	}

	return builder.String()
}

func main() {
	result := buildMessage([]string{"Hello", " ", "World"})
	println(result)
}
```
{{< /admonition >}}
#### Benefits:
- strings.Builder replaces string concatenation with `+`, which helps reduce the expansion of memory behind the string.
- Pre-allocating space for the Builder helps avoid unnecessary new memory allocations.

### 5. Use stack returns when possible
One of Go's most effective optimizations is the [compiler's analysis](https://go.dev/doc/faq#stack_or_heap) that determines which variables can be returned on the stack or need to be placed on the heap. I recommend watching [Understanding Allocations: the Stack and the Heap - GopherCon SG 2019](https://youtu.be/ZMZpH4yT7M0?si=S6ECgkU8mDkGhNFD) for a better understanding.

{{< admonition example >}}
```go
type Data struct {
    value int
}

func processData() Data {
    d := Data{value: 42}
    // Return the value on the stack
    return d
}

func processStackPointer(d *Data) err {
    // Use the original pointer on the stack
    d = &Data{value: 42}
    return nill
}

func processPointer() *Data {
    d := Data{value: 42}
    // Return the value on the heap, which requires memory allocation
    return &d
}
```
{{< /admonition >}}
#### Guidelines:
- Avoid returning pointers to internal variables unless necessary. See more at ([Go and the Age-Old Question: Value or Pointer?]({{< ref "/posts/go/value_or_pointer" >}} "Go and the Age-Old Question: Value or Pointer?"))
- Prioritize values over pointers when the size of the returned object is small, such as primitive data types.

### 6. Reduce repeated variable creation in each operation
In code sections that are executed frequently (e.g., request handlers, loops), eliminating memory allocations will have a performance impact that increases with the number of calls.

{{< admonition example >}}
```go
package main

func sumAll(inputs []int) int {
	sum := 0
	for _, val := range inputs {
		sum += val // Use the same memory for the operation
	}
	return sum
}

func main() {
	sum := sumAll([]int{1, 2, 3, 4, 5})
	println(sum)
}
```
{{< /admonition >}}
#### Explanation:
- We declare the `sum` variable in advance and use it within the loop without having to declare a new variable every time the loop iterates, which helps save on additional memory allocations.

## Conclusion
Zero allocation is an important technique for writing efficient Go programs. Avoiding frequent new memory allocations helps reduce the garbage collector's workload and makes programs run faster. However, the choice of techniques depends on the context of the problem and the system's constraints.

**Note**: The use of zero allocation should also consider the complexity and readability of the code. Using overly complex techniques can make the code difficult to read and a burden for future generations.

Additional: [Zero Allocations And Benchmarking In Golang](https://youtu.be/QFGbTOsk-Bk?si=1bejzN__HsLH7fmu)
