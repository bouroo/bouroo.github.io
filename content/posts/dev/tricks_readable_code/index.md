---
title: "ทริกง่าย ๆ กับการเขียนโค้ดไม่ให้บรรพบุรุษเดือดร้อน"
subtitle: ""
date: 2024-06-09T11:01:28+07:00
lastmod: 2024-06-09T11:01:28+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: ""
license: ""
images: []

tags: ["dev"]
categories: ["dev"]

featuredImage: "featured-image.webp"
featuredImagePreview: "featured-image.webp"

lightgallery: true
---
ในการเขียนโค้ดกันเป็นทีมมันจะมีเรื่องชวนปวดหัวบ่อย ๆ เลยคือ **ใครเขียนโค้ดนี้ว่ะ 🤬** เลยจะต้องมีสิ่งพึงละลึกร่วมกัน หรือตกลงกันในทีมในการเขียนโค้ดให้การกระทบกระทั้งกันน้อยลง โดยจะเอาวิธีง่าย ๆ ที่ช่วยให้ชีวิตชาวเราดีขึ้นมาเสนอลองพิจารณาดูครับ
<!--more-->
## โค้ดเจ้าปัญหา
เริ่มจากโค้ดเจ้าปัญหาที่เกิดจากการทำงานแบบปะผุ It works
```go
func main() {
    ...
    if isLogedIn {
        if isStaff {
            if isInFinanceDepartment {
                for _, v := range products {
                    switch v.Cat {
                        case "food":
                            rwMutex.Lock()
                            total["price"] += v.Amount + v.AddedAmount
                            rwMutex.Unlock()
                        case "drink":
                            rwMutex.Lock()
                            total["price"] += v.Amount + v.AddedAmount
                            rwMutex.Unlock()
                        case "alcohol":
                            rwMutex.Lock()
                            total["price"] += v.Amount + v.Tax
                            rwMutex.Unlock()
                        default:
                            rwMutex.Lock()
                            total["price"] += v.Amount
                            rwMutex.Unlock()
                    }
                }
            } else {
                slog.Warn("main", "isInFinanceDepartment", false)
                return ...
            }
        } else {
            slog.Warn("main", "isStaff", false)
            return ...
        }
    } else {
        slog.Warn("main", "isLogedIn", false)
        return ... 
    }

    p(products, total)
    ...
}
```

## หลีกเลี่ยงการเขียนโค้ดซ้อนกันหลายชั้นใน 1 บล็อค
ก็คือลดความซับซ้อนของโค้ดเวลาอ่าน จะได้ไม่ต้องจำว่ามันอยู่ในวงไหนกันนะ

### ยุบ nested if ด้วย inversion
จากของเดิมจะเห็นหว่ามี if ครอบอยู่หลายชั้นมาก ซึ่งส่วนมากแล้วถ้าเราไม่ได้ทำงานแยกกันระหว่าง if else เราสามารถยุบ if ได้ตามนี้
```go
func main() {
    ...
    if !isLogedIn {
        slog.Warn("main", "isLogedIn", false)
        return ... 
    }

    if !isStaff || !isInFinanceDepartment {
        slog.Warn("main", "isStaffInFinanceDepartment", false)
        return ...
    }

    for _, v := range products {
        switch v.Cat {
            case "food":
                rwMutex.Lock()
                receipt["price"] += v.Amount + v.AddedAmount
                rwMutex.Unlock()
            case "drink":
                rwMutex.Lock()
                receipt["price"] += v.Amount + v.AddedAmount
                rwMutex.Unlock()
            case "alcohol":
                rwMutex.Lock()
                receipt["price"] += v.Amount + v.Tax
                rwMutex.Unlock()
            default:
                rwMutex.Lock()
                receipt["price"] += v.Amount
                rwMutex.Unlock()
        }
    }

    p(products, receipt)
    ...
}
```

### ย้ายสิ่งที่เป็น logic ซับซ้อนออกมาเป็นฟังก์ชัน
เอา logic ที่อ่านยาก ๆ ยาว ๆ ออกมาเป็นชื่อที่ระบุว่า logic นั้นทำอะไร
```go
func isFinanceStaff() bool {
    return isStaff && isInFinanceDepartment
}

func main() {
    ...
    if !isLogedIn {
        slog.Warn("main", "isLogedIn", false)
        return ... 
    }

    if isFinanceStaff() {
        slog.Warn("main", "isFinanceStaff", false)
        return ...
    }

    for _, v := range products {
        switch v.Cat {
            case "food":
                rwMutex.Lock()
                receipt["price"] += v.Amount + v.AddedAmount
                rwMutex.Unlock()
            case "drink":
                rwMutex.Lock()
                receipt["price"] += v.Amount + v.AddedAmount
                rwMutex.Unlock()
            case "alcohol":
                rwMutex.Lock()
                receipt["price"] += v.Amount + receipt["tax"]
                rwMutex.Unlock()
            default:
                rwMutex.Lock()
                receipt["price"] += v.Amount
                rwMutex.Unlock()
        }
    }

    p(products, receipt)
    ...
}
```

## ทำให้โค้ดใช้ซ้ำได้ ลดความทับซ้อนของโค้ด
โค้ดไหนที่มีการสร้างมาซ้ำ ๆ แต่ไม่ได้ทำงานต่างกันก็ยุบรวมไปให้ใช้งานซ้ำ ๆ กันได้จากที่เดียวกัน
```go
func isFinanceStaff() bool {
    return isStaff && isInFinanceDepartment
}

func main() {
    ...
    if !isLogedIn {
        slog.Warn("main", "isLogedIn", false)
        return ... 
    }

    if isFinanceStaff() {
        slog.Warn("main", "isFinanceStaff", false)
        return ...
    }

    rwMutex.Lock()
    receipt["price"] := SumProductPrice(products, receipt["tax"])
    rwMutex.Unlock()

    p(products, totalPrice)
    ...
}

func SumProductPrice(products []Product, tax float64) total float64 {
    for _, v := range products {
        switch v.Cat {
            case "food":
                total += v.Amount + v.AddedAmount
            case "drink":
                total += v.Amount + v.AddedAmount
            case "alcohol":
                total += v.Amount + tax
            default:
                total += v.Amount
        }
    }
}
```

## อย่าตั้งชื่อให้เข้าใจอยู่คนเดียว
การระบุชื่อในโค้ด ให้ใช้คำที่ลูกค้าเข้าใจ เพื่อนร่วมทีมเข้าใจ แล้วเราก็จะเข้าใจกัน
```go
func isFinanceStaff() bool {
    return isStaff && isInFinanceDepartment
}

func main() {
    ...
    if !isLogedIn {
        slog.Warn("main", "isLogedIn", false)
        return ... 
    }

    if isFinanceStaff() {
        slog.Warn("main", "isFinanceStaff", false)
        return ...
    }

    receiptMutex.Lock()
    receipt["totalPrice"] := SumProductPrice(products, receipt["tax"])
    receiptMutex.Unlock()

    p(products, totalPrice)
    ...
}

func SumProductPrice(products []Product, tax float64) total float64 {
    for _, product := range products {
        switch product.Category {
            case "food":
                total += product.Price + product.GasPrice
            case "drink":
                total += product.Price + product.PackagePrice
            case "alcohol":
                total += product.Price + tax
            default:
                total += product.Price
        }
    }
}
```

## สรุป
ก็ลองไปปรับใช้กันดูนะครับเผื่อจะมีคนพาดพิงน้อยลงเวลาที่มีคนเอาโค้ดเราไปทำงานต่อ Happy coding กันครับ