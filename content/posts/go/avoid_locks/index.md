---
title: "เขียน Go เรียกใช้ resource เดียวกันแต่ไม่อยากใช้ Exclusive Lock มีทางไหนบ้าง 🤔"
subtitle: ""
date: 2024-12-21T18:07:21+07:00
lastmod: 2024-12-21T18:07:21+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: ""
license: ""
images: []

tags: []
categories: []

featuredImage: "/img/featured-image.webp"
featuredImagePreview: "/img/featured-image.webp"

lightgallery: true
---

ในภาษา Go การใช้ Mutex (Mutual Exclusion) เป็นวิธีการหนึ่งในการจัดการกับการเข้าถึงข้อมูลที่ใช้ร่วมกันในหลายๆ โค้ดหรือ Goroutine เพื่อป้องกันการเกิด Race Condition ซึ่งอาจทำให้ข้อมูลเสียหายได้ แต่ในบางครั้งการใช้ Mutex ก็อาจทำให้เกิดปัญหาด้านประสิทธิภาพและความซับซ้อนของโปรแกรม และเสีย performance ดังนั้นเราลองมาเขียนโดยการหลีกเลี่ยง Mutex Lock โดยไม่จำเป็นกันดูว่าทำยังไงบ้าง

<!--more-->

## เหตุผลที่ควรหลีกเลี่ยง Exclusive Lock
- Deadlock: เกิดขึ้นเมื่อ goroutine สองตัวขึ้นไป รอให้กันและกันปลดล็อก mutex ที่วางผิดที่ผิดทางจะทำให้โปรแกรมติด lock แล้วหยุดทำงาน
- Livelock: เกิดขึ้นเมื่อ goroutine สองตัวขึ้นไปสลับกันปลดล็อกและล็อก mutex ซ้ำๆ ทำให้ไม่มี goroutine ใดสามารถดำเนินการต่อได้
- Overhead: การล็อกและปลดล็อก mutex มีค่าใช้จ่ายในระหว่าง context switch และการจัดการสถานะของ mutex ซึ่งอาจส่งผลต่อประสิทธิภาพของโปรแกรมโดยเฉพาะอย่างยิ่งในกรณีที่มีการ contention สูง
- ซับซ้อน: การจัดการ mutex ในโปรแกรมที่ซับซ้อนอาจทำให้โค้ดอ่านยากและเข้าใจยากขึ้น
- เพิ่มเติม: [Dmitry Vyukov — Go scheduler: Implementing language with lightweight concurrency](https://youtu.be/-K11rY57K7k?si=t8vKOjBWpcJ7YwJA)

## วิธีหลีกเลี่ยง Exclusive Lock (เท่าที่ผมรู้)
### 1. ใช้ Channel แทน Mutex

Channel เป็นเครื่องมือหลักในการสื่อสารระหว่าง Goroutine ใน Go และสามารถใช้แทน Mutex เพื่อจัดการการเข้าถึงข้อมูลได้อย่างปลอดภัย

{{< admonition example >}}
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	dataChannel := make(chan int)
	go producer(dataChannel)
	go consumer(dataChannel)

	time.Sleep(2 * time.Second) // รอให้ Goroutine ทำงานเสร็จ
}

func producer(ch chan<- int) {
	for i := 0; i < 5; i++ {
		ch <- i // ส่งข้อมูลไปยัง channel
		fmt.Println("Produced:", i)
		time.Sleep(100 * time.Millisecond)
	}
	close(ch) // ปิด channel เมื่อส่งข้อมูลเสร็จ
}

func consumer(ch <-chan int) {
	for data := range ch {
		fmt.Println("Consumed:", data) // รับข้อมูลจาก channel
		time.Sleep(150 * time.Millisecond)
	}
}
```
{{< /admonition >}}

### 2. ใช้ Data Structure ที่ไม่ต้อง Lock

การใช้ Data Structure ที่ออกแบบมาเพื่อหลีกเลี่ยงการใช้ Mutex เช่น `sync.Map` ช่วยให้เข้าถึงข้อมูลได้อย่างปลอดภัยโดยไม่ต้องใช้ Mutex

{{< admonition example >}}
```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var m sync.Map

	// เก็บข้อมูล
	m.Store("key1", "value1")
	m.Store("key2", "value2")

	// อ่านข้อมูล
	m.Range(func(key, value interface{}) bool {
		fmt.Println(key, value)
		return true
	})
}
```
{{< /admonition >}}

### 3. ใช้ Atomic Operations

Go มีแพคเกจ `sync/atomic` ที่ให้ฟังก์ชันสำหรับการดำเนินการเชิงอะตอมิก ซึ่งช่วยให้สามารถทำการเพิ่มหรือลดค่าของตัวแปรในลักษณะที่ปลอดภัยจากการเข้าถึงพร้อมกัน

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
			atomic.AddInt64(&counter, 1) // เพิ่มค่าตัวแปร counter
		}()
	}

	wg.Wait() // รอให้ Goroutine ทุกตัวทำงานเสร็จ
	fmt.Println("Final Counter:", counter)
}
```
{{< /admonition >}}

### 4. ใช้ Read/Write Mutex

หากต้องการให้มีการอ่านข้อมูลได้หลายๆ Goroutine แต่ต้องการให้เขียนข้อมูลได้เพียง Goroutine เดียว และมีความจำเป็นต้องใช้ mutex เราสามารถใช้ `sync.RWMutex` ซึ่งช่วยให้การอ่านสามารถทำได้พร้อมๆ กันโดยไม่ต้องรอการเขียน

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
	s.mu.RLock() // ใช้ RLock สำหรับการอ่าน
	defer s.mu.RUnlock()
	return s.data
}

func (s *SafeData) Write(value int) {
	s.mu.Lock() // ใช้ Lock สำหรับการเขียน
	defer s.mu.Unlock()
	s.data = value
}

func main() {
	safeData := SafeData{}

	var wg sync.WaitGroup

	// Goroutine สำหรับเขียนข้อมูล
	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 0; i < 5; i++ {
			safeData.Write(i)
			fmt.Println("Written:", i)
			time.Sleep(100 * time.Millisecond)
		}
	}()

	// Goroutine สำหรับอ่านข้อมูล
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			fmt.Println("Read:", safeData.Read())
			time.Sleep(50 * time.Millisecond)
		}()
	}

	wg.Wait() // รอให้ Goroutine ทุกตัวทำงานเสร็จ
}
```
{{< /admonition >}}

## สรุป

การหลีกเลี่ยงการใช้ Mutex Lock ในภาษา Go สามารถทำได้หลายวิธี เช่น การใช้ Channel, Data Structures ที่ไม่ต้อง Lock, Atomic Operations และ Read/Write Mutex การเลือกใช้วิธีการที่เหมาะสมจะช่วยเพิ่มประสิทธิภาพและลดความซับซ้อนในการเขียนโปรแกรม ซึ่งจะทำให้โปรแกรมทำงานได้อย่างราบรื่นและปลอดภัยจากการเข้าถึงข้อมูลพร้อมกัน

{{< admonition type=quote title="Andrew Gerrand" >}}
Do not communicate by sharing memory; instead, share memory by communicating.
{{< /admonition >}}

เพิ่มเติม: [How To Avoid Locks (Mutex) In Your Golang Programs?](https://youtu.be/Ya5KRFrwPug?si=_DaVJYNj3uJGq7nz)