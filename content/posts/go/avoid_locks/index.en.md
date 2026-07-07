---
title: "How to Write Go Code That Accesses Shared Resources Without Using Exclusive Locks"
subtitle: ""
date: 2024-12-21T18:07:21+07:00
lastmod: 2024-12-21T18:07:21+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "Explains why to avoid exclusive mutex locks in Go and shows four lock-free alternatives: channels, sync.Map, atomic operations, and sync.RWMutex."
license: ""
images: []

tags: []
categories: []

featuredImage: "featured-image.jpg"
featuredImagePreview: "featured-image.jpg"

lightgallery: true
---

In Go, using a Mutex (Mutual Exclusion) is one way to manage access to shared data across multiple goroutines to prevent race conditions, which can lead to data corruption. However, sometimes using a Mutex can cause performance issues and program complexity, and it can hurt performance. So let's try to write code that avoids unnecessary Mutex locks and see how it's done.

<!--more-->

## Reasons to Avoid Exclusive Locks
- Deadlock: Occurs when two or more goroutines are waiting for each other to release a misplaced mutex, causing the program to lock up and stop working.
- Livelock: Occurs when two or more goroutines repeatedly release and lock a mutex, preventing any goroutine from making progress.
- Overhead: Locking and unlocking a mutex has a cost in terms of context switching and managing the state of the mutex, which can affect program performance, especially in cases of high contention.
- Complexity: Managing mutexes in complex programs can make the code harder to read and understand.
- Additional: [Dmitry Vyukov — Go scheduler: Implementing language with lightweight concurrency](https://youtu.be/-K11rY57K7k?si=t8vKOjBWpcJ7YwJA)

## Ways to Avoid Exclusive Locks (as far as I know)
### 1. Use Channels Instead of Mutexes

Channels are the primary tool for communication between goroutines in Go and can be used instead of mutexes to safely manage data access.

{{< admonition example >}}
```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	const (
		items      		= int[]{1, 2, 3, 4, 5, 6, 7}
		producerSleep   = 100 * time.Millisecond
		consumerSleep   = 150 * time.Millisecond
		channelBuffer   = 2 // The size of the channel buffer for the producer and consumer
	)

	dataChannel := make(chan int, channelBuffer)
	var wg sync.WaitGroup

	wg.Add(2) // Create a wait queue for 2 goroutines

	go producer(dataChannel, items, producerSleep, &wg)
	go consumer(dataChannel, consumerSleep, &wg)

	wg.Wait() // Wait until both goroutines are done
}

func veryLongTask(input int, sleepDuration time.Duration) (output int, err error) {
	output = input * 2
	time.Sleep(sleepDuration)
	return
}

func producer(ch chan<- int, items []int, sleepDuration time.Duration, wg *sync.WaitGroup) {
	defer wg.Done()
	for _, item := range items {
		// Send to work
		output, _ := veryLongTask(item, sleepDuration)
		ch <- output // Send data to the channel after the work is done
		fmt.Println("Produced:", item)
	}
	close(ch) // Close the channel after all data has been sent
}

func consumer(ch <-chan int, sleepDuration time.Duration, wg *sync.WaitGroup) {
	defer wg.Done()
	for data := range ch {
		// Receive data from the channel
		fmt.Println("Consumed:", data)
		time.Sleep(sleepDuration)
	}
}
```
{{< /admonition >}}

### 2. Use Lock-Free Data Structures

Using data structures designed to avoid mutexes, such as `sync.Map`, allows for safe data access without having to manage mutexes yourself.

{{< admonition example >}}
```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var m sync.Map

	// Store data
	m.Store("key1", "value1")
	m.Store("key2", "value2")

	// Read data
	m.Range(func(key, value interface{}) bool {
		fmt.Println(key, value)
		return true
	})
}
```
{{< /admonition >}}

### 3. Use Atomic Operations

Go has the `sync/atomic` package, which provides functions for atomic operations (similar to the Atomicity property in databases), allowing for safe incrementing or decrementing of variables from concurrent access by multiple goroutines.

{{< admonition example >}}
```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var counter int64
	var wg sync.WaitGroup

	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			atomic.AddInt64(&counter, 1) // Increment the counter variable
		}()
	}

	wg.Wait() // Wait for all goroutines to finish
	fmt.Println("Final Counter:", counter)
}
```
{{< /admonition >}}

### 4. Use a Read/Write Mutex

If you want to allow multiple goroutines to read data but only one goroutine to write data, and you need to use a mutex, you can use `sync.RWMutex`, which allows for concurrent reads without waiting for writes.

{{< admonition example >}}
```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type SafeData struct {
	mu sync.RWMutex
	data int
}

func (s *SafeData) Read() int {
	s.mu.RLock() // Use RLock for reading
	defer s.mu.RUnlock()
	return s.data
}

func (s *SafeData) Write(value int) {
	s.mu.Lock() // Use Lock for writing
	defer s.mu.Unlock()
	s.data = value
}

func main() {
	safeData := SafeData{}

	var wg sync.WaitGroup

	// Goroutine for writing data
	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 0; i < 5; i++ {
			safeData.Write(i)
			fmt.Println("Written:", i)
			time.Sleep(100 * time.Millisecond)
		}
	}()

	// Goroutines for reading data
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			fmt.Println("Read:", safeData.Read())
			time.Sleep(50 * time.Millisecond)
		}()
	}

	wg.Wait() // Wait for all goroutines to finish
}
```
{{< /admonition >}}

## Conclusion

Avoiding the use of Mutex locks in Go can be done in several ways, such as using channels, lock-free data structures, atomic operations, and read/write mutexes. Choosing the right method will help improve performance and reduce the complexity of your program, making it run smoothly and safely from concurrent data access (race conditions).

{{< admonition type=quote title="Andrew Gerrand" >}}
Do not communicate by sharing memory; instead, share memory by communicating.
{{< /admonition >}}

Additional: [How To Avoid Locks (Mutex) In Your Golang Programs?](https://youtu.be/Ya5KRFrwPug?si=_DaVJYNj3uJGq7nz)
