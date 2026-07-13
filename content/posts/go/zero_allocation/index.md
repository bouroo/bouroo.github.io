---
title: "มาลองรู้จักการพยายามเขียนแบบ zero-allocation บน Go"
subtitle: ""
date: 2024-12-21T11:47:24+07:00
lastmod: 2024-12-21T11:47:24+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "เทคนิคการลด heap allocation ใน Go เพื่อช่วยลดภาระของ garbage collector ด้วยการใช้ array, sync.Pool, การจัดการความจุของ slice, strings.Builder และการคืนค่าผ่าน stack"
license: ""
images: []

tags: []
categories: []

featuredImage: "featured-image.jpg"
featuredImagePreview: "featured-image.jpg"

lightgallery: true
---

คุณสมบัติอย่างหนึ่งของ Go คือ มันมี Garbage collector (GC) ทำให้ชีวิต Dev อย่างเรา ๆ ไม่ต้องมาปวดหัวกับการที่ต้องมาคอยจัดการหน่วยความจำ ซึ่งความสะดวกนี้มันก็มีสิ่งที่ต้องแลกมานั่นคือ ช่วงเวลาที่ต้องให้ GC ทำงานนั่นเอง ถึงจะไม่ขนาด stop-the-world ก็เถอะนะ [เพิ่มเติม Go GC ได้ตามลิ้งก์นี้เลย](https://tip.golang.org/doc/gc-guide) แต่สำหรับบางงานที่ latency คือสิ่งสำคัญเราก็ต้องมาช่วยให้ GC ทำงานน้อยลงนั่นคือลด heap allocation ลงนั่นเอง เป็นที่มาว่าทำยังไงเราจะลด heap allocation ให้ได้มาและไม่ฝืนจนเกินไป

<!--more-->

## ทำไมต้องหลีกเลี่ยง heap allocation?
แม้ว่า GC ของ Go จะถูกออกแบบมาเพื่อความมีประสิทธิภาพในการจัดการการหน่วยความจำได้อย่างรวดเร็วแต่ก็มีข้อสังเกตุอยู่:
- Latency: ทุก ๆ รอบของการทำงาน GC จะมีความหน่วงเล็กน้อยเสมอ ซึ่งอาจเป็นปัญหาในระบบที่ต้องการ response time ที่สม่ำเสมอมาก ๆ
- CPU resource: GC จะทำงานได้ก็ต้องใช้ CPU เพราะฉะนั้นจะมีการแบ่ง resource ไปใช้กับ GC
- ~~Stop-The-World~~: ถึงแม้ว่าจะไม่ถึงขั้น stop-the-world แบบจริงจังแต่มันก็จะมีช่วงที่ต้องหน่วงเล็กน้อยอยู่ดีซึ่งก็จะมีผลกับระบบที่ซับซ้อนหรือมีขนาดใหญ่ มาก ๆ อยู่ดี

ซึ่งเราสามารถช่วยลดภาระงานของ GC ลงได้ด้วยการ~~ไม่ทิ้งขยะเรี่ยราด~~ลดการจอง heap ไปทั่ว

## วิธีลด heap allocation (เท่าที่ผมรู้)
### 1. ใช้ Array แทน Slice ถ้ารู้ขนาดล่วงหน้า
การ allocate หน่วยความจำแบบไดนามิก (เช่น ในระหว่างการทำงาน) มักจะนำไปสู่การ allocate heap ซึ่ง GC จะต้องเรียกคืนในที่สุด แทนที่จะสร้าง slices หรือ buffer ใหม่ในขณะที่ทำงาน การ allocate array ที่นำกลับมาใช้ใหม่ล่วงหน้าจะช่วยลด heap allocation ได้

{{< admonition example >}}
```go
package main

// บังคับว่า inputs ต้องมี 10 ตัว และสร้าง output ไว้รอ 10 ตัว
func doubleTenValue(inputs [10]int) (result [10]int) {
	for i, input := range inputs {
		// ประมวลผล buffer
		result[i] = input * 2
	}

	return
}

func main() {
	inputs := [...]int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	result := doubleTenValue(inputs)

	// แสดงผลลัพธ์
	for _, v := range result {
		println(v)
	}
}
```
{{< /admonition >}}
#### ประโยชน์:
- ลดการ allocate heap ที่เกิดขึ้นซ้ำ ๆ
- บัฟเฟอร์ถูกนำกลับมาใช้ใหม่ ไม่ใช่สร้างใหม่เรื่อย ๆ ซึ่งช่วยลดภาระงานของ garbage collector

### 2. การใช้ sync.Pool สำหรับการนำกลับมาใช้ใหม่
sync.Pool เป็นเครื่องมือที่มีประสิทธิภาพสำหรับการจัดการกับสิ่งที่มีราคาแพงเวลาสร้างแล้วใช้งานแบบชั่วคราวที่ใช้บ่อยและถูกทิ้ง ถ้าเราทำเป็น pool จะเกิดวนนำมาใช้ซ้ำลดขยะที่ต้องเก็บกวาดนั่นเอง

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
	// ดึงเอา buffer มาจาก pool
	buf := bufPool.Get().(*bytes.Buffer)
	// ทำงานเสร็จก็เอากลับคืน pool
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
#### ประโยชน์:
- ลดจำนวนการจองหน่วยความจำอย่างมากตามความบ่อยในการสร้าง
- บัฟเฟอร์ถูกนำกลับมาใช้ใหม่ ซึ่งสามารถลดงานของ GC ได้อย่างมาก

### 3. การจัดการความจุของ Slice
Slices ใน Go มีประโยชน์มาก แต่การขยายได้เรื่อย ๆ ของ slice มักจะส่งผลให้เกิดการจองและย้ายหน่วยความจำไปเรื่อย ๆ เพื่อให้ขนาดของ slice พอกับ runtime โดยการจัดการความจุของ slice ให้หลีกเลี่ยงการปรับขนาดที่ไม่จำเป็น ทำให้เราสามารถเก็บ slices ไว้ใน stack แทนที่จะเป็น heap ได้

{{< admonition example >}}
```go
package main

func appendData(inputs []int) []int {
    // สร้าง slice ที่มีความจุพอใช้งานไว้ล่วงหน้า
	result := make([]int, 0, len(inputs))

	for _, val := range inputs {
        // เพิ่มค่าใน slice โดยไม่มีการขยายความจุ
		result = append(result, val * 2)
	}

	return result
}

func main() {
	inputs := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	result := appendData(inputs)

	// แสดงผลลัพธ์
	for _, v := range result {
		println(v)
	}
}
```
{{< /admonition >}}
#### ประโยชน์:
- โดยการเริ่มต้น result ด้วยความจุที่กำหนดไว้ล่วงหน้า (len(inputs))
- เราหลีกเลี่ยงการจองหน่วยความจำใหม่และลดโอกาสในการ allocate จากขนาดของ heap ที่เพิ่มขึ้น

### 4. String ใน Go เป็น imutable
หมายความว่าการแก้ไข String แต่ละครั้งจะสร้าง String ใหม่ เพื่อเป็นการหลีกเลี่ยงการจองหน่วยความจำภายใต้ String บ่อย ๆ เราควร
- ใช้ strings.Builder สำหรับการเชื่อมต่อ String
- หลีกเลี่ยงการใช้ `+` สำหรับการเชื่อมต่อ String หลาย ๆ ตัวในลูป

{{< admonition example >}}
```go
package main

import "strings"

func buildMessage(parts []string) string {
	var builder strings.Builder
	// allocate memory ให้พอดีกับที่จะใช้งาน
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
#### ประโยชน์:
- strings.Builder ใช้แทนการต่อ string ด้วย `+` ช่วยลดการขยายของหน่วยความจำด้านหลังของ string ได้
- การเตรียมพื้นที่ล่วงหน้าของ Builder ช่วยหลีกเลี่ยงการจองหน่วยความจำใหม่ที่ไม่จำเป็น

### 5. ใช้การส่งค่ากลับใน Stack เมื่อเป็นไปได้
หนึ่งในการเพิ่มประสิทธิภาพที่มีประสิทธิภาพที่สุดของ Go คือ[การวิเคราะห์โดยคอมไพเลอร์](https://go.dev/doc/faq#stack_or_heap)ที่กำหนดว่าตัวแปรใดสามารถส่งกลับได้ใน stack หรือจำเป็นต้องฝากไว้ที่ heap แนะนำดู [Understanding Allocations: the Stack and the Heap - GopherCon SG 2019](https://youtu.be/ZMZpH4yT7M0?si=S6ECgkU8mDkGhNFD) เพิ่มเติมเพื่อให้เข้าใจมากขึ้นครับ

{{< admonition example >}}
```go
type Data struct {
    value int
}

func processData() Data {
    d := Data{value: 42}
    // ส่งค่ากลับทาง stack
    return d
}

func processStackPointer(d *Data) err {
    // ใช้งานจาก pointer เดิมบน stack
    d = &Data{value: 42}
    return nill
}

func processPointer() *Data {
    d := Data{value: 42}
    // ส่งค่ากลับทาง heap ทำให้ต้อง allocation memory
    return &d
}
```
{{< /admonition >}}
#### แนวทางการทำงาน:
- หลีกเลี่ยงการส่งคืน pointers ไปยังตัวแปรภายในเว้นแต่จำเป็นต้องทำดูเพิ่มได้ที่ ([Go กับปัญหาโลกแตก Value หรือ Pointer]({{< ref "/posts/go/value_or_pointer" >}} "Go กับปัญหาโลกแตก Value หรือ Pointer"))
- ให้ความสำคัญกับ value มากกว่า pointers เมื่อขนาดของ Object ที่ return นั้นเล็ก เช่น พวก Primitive data type

### 6. ลดการสร้างตัวแปรซ้ำ ๆ ในแต่ละการทำงาน
ในส่วนของโค้ดที่ถูกดำเนินการซ้ำบ่อย ๆ (เช่น ตัวจัดการคำขอ, การวนลูป) การกำจัดการจองหน่วยความจำ จะมีผลต่อมีประสิทธิภาพเพิ่มขึ้นตามจำนวนการเรียกใช้งาน

{{< admonition example >}}
```go
package main

func sumAll(inputs []int) int {
	sum := 0
	for _, val := range inputs {
		sum += val // ใช้หน่วยความจำเดิมในการทำงาน
	}
	return sum
}

func main() {
	sum := sumAll([]int{1, 2, 3, 4, 5})
	println(sum)
}
```
{{< /admonition >}}
#### คำอธิบาย:
- เราประกาศค่า sum ไว้ล่วงหน้าแล้วนำไปใช้งานภายในลูปโดยที่ไม่ต้องประกาศตัวแปรใหม่ทุกครั้งที่มีการวนลูปช่วยให้ประหยัดการจองหน่วยความจำเพิ่มได้

## สรุป
Zero allocation เป็นเทคนิคที่สำคัญในการเขียนโปรแกรมภาษา Go ที่มีประสิทธิภาพ การหลีกเลี่ยงการจองหน่วยความจำใหม่บ่อยครั้งจะช่วยลดภาระของ garbage collector และทำให้โปรแกรมทำงานได้เร็วขึ้น อย่างไรก็ตาม การเลือกใช้เทคนิคต่าง ๆ ขึ้นอยู่กับบริบทของปัญหาและข้อจำกัดของระบบ

**หมายเหตุ**: การใช้ zero allocation ควรพิจารณาถึงความซับซ้อนของโค้ดและความอ่านง่ายของโค้ดด้วย การใช้เทคนิคที่ซับซ้อนเกินไปอาจทำให้โค้ดอ่านยากและลำบากคนรุ่นหลัง

เพิ่มเติม: [Zero Allocations And Benchmarking In Golang](https://youtu.be/QFGbTOsk-Bk?si=1bejzN__HsLH7fmu)
