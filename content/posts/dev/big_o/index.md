---
title: "ทำความเข้าใจ Big O Notation"
subtitle: ""
date: 2024-08-07T23:13:16+07:00
lastmod: 2024-08-07T23:13:16+07:00
draft: true
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "แนะนำ Big O Notation แบบใช้งานได้จริง ครอบคลุม O(1), O(n), O(log n), O(n²), O(n log n) และ O(2ⁿ) พร้อมตัวอย่างสั้น ๆ ในภาษา Go สำหรับแต่ละรูปแบบ"
license: ""
images: []

tags: []
categories: []

featuredImage: "/img/featured-image.webp"
featuredImagePreview: "/img/featured-image.webp"

lightgallery: true
---
## ทำไมเราถึงต้องสนใจ Big O Notation?

ลองนึกภาพ Big O Notation เป็นเหมือน "เกจวัดเชื้อเพลิง" สำหรับประสิทธิภาพของโค้ดของคุณ มันจะช่วยบอกเราว่าประสิทธิภาพของโปรแกรมจะเปลี่ยนแปลงไปอย่างไรเมื่อขนาดของข้อมูลเพิ่มขึ้น ซึ่งเป็นสิ่งสำคัญอย่างยิ่งในการสร้างแอปพลิเคชันที่ทำงานได้อย่างราบรื่น แม้จะต้องจัดการกับข้อมูลจำนวนมหาศาล

<!--more-->

## มาทำความเข้าใจ Big O Notation กันอย่างละเอียด

* **Big O Notation บ่งบอกถึงสถานการณ์ที่เลวร้ายที่สุด.** เป็นวิธีการแสดงว่าจำนวนการดำเนินการที่โปรแกรมของคุณทำจะเพิ่มขึ้นอย่างไรเมื่อเทียบกับขนาดของข้อมูล 
* **เราใช้สัญลักษณ์ เช่น O(n), O(n^2), O(log n) เป็นต้น** เพื่อแสดงถึงอัตราการเติบโตที่แตกต่างกัน

## Big O Notation ทั่วไป

### O(1) - เวลาคงที่
เวลาที่ใช้ในการดำเนินการไม่ขึ้นอยู่กับขนาดของข้อมูล 

```go
package main

import "fmt"

func main() {
    slice := []int{1, 2, 3, 4, 5}
    fmt.Println(slice[2]) // O(1)
}
```

### O(n) - เวลาเชิงเส้น
เวลาที่ใช้ในการดำเนินการจะเพิ่มขึ้นตามขนาดของข้อมูล

```go
package main

import "fmt"

func main() {
    slice := []int{1, 2, 3, 4, 5}
    for i := 0; i < len(slice); i++ { // O(n)
        fmt.Println(slice[i])
    }
}
```

### O(log n) - เวลาลอการิทึม
เวลาที่ใช้ในการดำเนินการจะเพิ่มขึ้นแบบลอการิทึม ตามขนาดของข้อมูล 

```go
package main

import "fmt"

func main() {
    slice := []int{1, 3, 5, 7, 9}
    target := 7
    left := 0
    right := len(slice) - 1
    for left <= right { // O(log n)
        mid := (left + right) / 2
        if slice[mid] == target {
            fmt.Println("Found!")
            break
        } else if slice[mid] < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }
}
```

### O(n^2) - เวลากำลังสอง
เวลาที่ใช้ในการดำเนินการจะเพิ่มขึ้นแบบกำลังสอง ตามขนาดของข้อมูล

```go
package main

import "fmt"

func main() {
    slice := []int{1, 2, 3, 4, 5}
    for i := 0; i < len(slice); i++ { // O(n)
        for j := 0; j < len(slice); j++ { // O(n)
            fmt.Println(slice[i], slice[j])
        }
    }
}
```

### O(n log n) - เวลาล็อกเชิงเส้น

```go
package main

import "fmt"

func main() {
    slice := []int{5, 2, 4, 6, 1, 3}
    for i := 0; i < len(slice)-1; i++ { // O(n)
        for j := i + 1; j < len(slice); j++ { // O(log n)
            if slice[i] > slice[j] {
                temp := slice[i]
                slice[i] = slice[j]
                slice[j] = temp
            }
        }
    }
    fmt.Println(slice) 
}
```

* **O(2^n) - เวลาเลขชี้กำลัง:** 

```go
package main

import "fmt"

func main() {
    n := 5
    result := 0
    for i := 0; i < n; i++ { // O(2^n)
        for j := 0; j < n; j++ {
            result += 1
        }
    }
    fmt.Println(result)
}
```

## สรุป
* การทำความเข้าใจ Big O Notation เป็นสิ่งสำคัญอย่างยิ่งสำหรับการเขียนโค้ดที่มีประสิทธิภาพ
* เราควรพยายามใช้อัลกอริธึมที่มีความซับซ้อนของเวลาต่ำกว่า โดยเฉพาะเมื่อต้องจัดการกับชุดข้อมูลขนาดใหญ่
* การใช้เครื่องมือ เช่น การสร้างโปรไฟล์โค้ด จะช่วยวิเคราะห์ประสิทธิภาพของโค้ดและระบุจุดที่อาจเกิดปัญหาได้