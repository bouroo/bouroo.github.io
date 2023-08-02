---
title: "เมื่อ Go กับ SOLID มาเจอกัน"
subtitle: ""
date: 2023-07-29T09:16:31+07:00
lastmod: 2023-07-29T09:16:31+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin-vir.pages.dev"
description: "เรามาดูกันว่าเราจะเขียน Go ตามแนวทาง SOLID ได้ยังไงกันนะ"
aliases:
- /posts/go_solid/
license: ""
images: []
resources:
- name: "featured-image"
  src: "featured-image.webp"

tags: ["Go", "SOLID", "Programing"]
categories: ["Programing"]

featuredImage: "featured-image.webp"
featuredImagePreview: "featured-image"

lightgallery: true
---

 การพัฒนาซอฟต์แวร์ขึ้นมาสักชุดนึงนั้นหากเราเน้นเร่งทำขึ้นมาโดยเน้นว่าเสร็จใช้งานได้ไปก่อนแล้วค่อยมาตามเก็บรายละเอียดที่หลัง (ซึ่งส่วนใหญ่ก็ไม่เคยจะเก็บหมดหรอก) สิ่งหนึ่งที่มักจะต้องเจอเลยก็คือ เพิ่มเติมฟีเจอร์ถึงจุดหนึ่งจะเริ่มรู้สึกว่าก็แก้ไขเพิ่มเติมอะไรก็เป็นเรื่องยากลำบาก ติดพันกันไปหมด หรือว่าเมื่อมีความต้องการใช้งานเพิ่มขึ้น กลับไม่สามารถขยายได้ตามความต้องการ ส่วนหนึ่งเพราะการพัฒนาซอฟต์แวร์นั้นไม่ได้มีการวางแผนโครงร่าง/แนวทางที่จะใช้พัฒนาโปรเจคนั่นเอง ซึ่งวันนี้จะมาแนะนำแนวทางนึงในการพัฒานาซอฟต์แวร์ให้มี คุณภาพโค้ดที่ดี, ดูแลรักษาง่าย, สามารถขยายได้ง่ายตามความต้องการที่เพิ่มเข้ามา ซึ่งแนวทางนึงที่คิดว่าค่อนข้างเข้าใจง่ายคือ SOLID นั่นเอง ซึ่งในตัวอย่างวันนี้จะอยู่ในภาษา Go นะครับ (ก็แน่สิผมถนัด Go #ถถถ)

<!--more-->

## มารู้จัก SOLID กันก่อน
ตามที่บอกไปข้างต้นเลยคือ SOLID เป็นแนวทางการพัฒนาซอฟต์แวร์เพื่อให้ ได้คุณภาพโค้ดที่ดี (คนมาอ่านต่อไม่สรรเสริญย้อนหลัง), ดูแลรักษาง่าย (แก้ที่นึงแล้วไม่กระทบเป็นปฏิกริยาลูกโซ่ไปที่อื่น ๆ แบบคาดเดาไม่ได้), สามารถขยายได้ง่ายตามความต้องการที่เพิ่มเข้ามา (ต่อเติมได้แบบไม่ต้องไปกังวลว่าจะทำของเก่าพังทั้งหมด)

## SOLID ประกอบด้วย
### **S**ingle Responsibility (SRP)
หมายความว่าโค้ดในแต่ละชุดควรมีความรับผิดชอบเพียงอย่างเดียวไม่ทำงานหลายอย่าง สิ่งนี้จะช่วยให้โค้ดสะอาดและบำรุงรักษาได้ง่าย เนื่องจากการเปลี่ยนแปลงโครงสร้างสามารถทำได้ในที่เดียว (ก็ใช่สิโค้ดที่ละชุดควรทำงานเฉพาะหน้าที่ของตัวเองอย่างเดียวอยู่แล้ว)

ตัวอย่าง เริ่มจากเรามีตารางของการรับชำระเงิน
```go
type Payment struct {}

func (k Payment) MakePayment {
  // do payment stuff
}

func (k Payment) CreateInvoice {
  // do invoice stuff 
}

func (k Payment) SendBill {
  // do bill mailing stuff 
}
```

จะเห็นว่า `Payment` รับผิดชอบทั้ง `Payment` และ `Invoice` และ `Bill` ฉะนั้นจาก SRP เราจะต้องแยกให้โครงสร้างมีความรับผิดชอบเฉพาะตัวซึ่งจะกลายเป็น

```go
type PaymentInfo struct {}

func (k PaymentInfo) MakePayment {
  // do payment stuff
}

type InvoiceInfo struct {}

func (k InvoiceInfo) CreateInvoice {
  // do invoice stuff 
}

type Billing struct {}

func (k Billing) SendBill {
  // do bill mailing stuff 
}
```

เวลาที่มีการใช้งานก็จะแยกจากกัน ทำให้โค้ดไม่มีการปะปนไปรับผิดชอบงานอื่นที่ไม่เกี่ยวกับตัวเองเลย ซึ่งจะพบเห็นตัวอย่างใกล้ตัวที่สุดและชัดเจนเลยคือ pkg ของ [Go standard library](https://pkg.go.dev/std) นี่แหละ เช่น `hash/crc64` กับ `hash/crc32` ที่แยก package ออกจากกัน แทนที่จะยำรวมมิตรกันใน `hash/crc` นั่นเอง

### **O**pen-Closed (OCP)
หมายความว่าพฤติกรรมของโค้ดส่วนขยายสามารถขยายได้โดยไม่ต้องเปลี่ยนโค้ดเดิม สิ่งนี้ช่วยให้โค้ดมีความยืดหยุ่นและปรับให้เข้ากับความต้องการที่เปลี่ยนแปลงได้โดยไม่ไปทำให้ของเดิมที่มีอยู่พัง (ก็คือใช้ของเดิมที่มีแล้วต่อเติมออกมาได้เท่านั้น)

ตัวอย่าง เริ่มจากเรามีช่องทางรับชำระเงินและกระบวนการคิดเงินโดยต้องมีมาจ่ายเงินผ่านทาง `Pay()`
```go
type PaymentChannel interface {
  Pay()
}

type Payment struct {}

func (k Payment) Process(payCh PaymentChannel) {
  payCh.Pay()
}
```

เมื่อเราต้องการเพิ่มช่องทางระบชำระเงินเช่น KKU PaymentHub

```go
type KkuPayment struct {
  amount float64
}

func (kkupay KkuPayment) Pay() {
  fmt.Printf("Paid %.2f via KKU PaymentHub", kkupay.amount)
}

func main() {
  payment := Payment{}
  kkuPay := KkuPayment{12.23}
  payment.Process(kkuPay)
}
```

ต่อมาวันดีคืนดีอยากให้รับชำระผ่านบัตรเครดิตได้เราก็จะเพิ่มได้แบบนี้

```go
type CreditCard struct {
  amount float64
}

func (cc CreditCard) Pay() {
  fmt.Printf("Paid %.2f via CreditCard", cc.amount)
}

func main() {
  payment := Payment{}
  // kku payment hub
  kkuPay := KkuPayment{60.43}
  p.Process(kkuPay)
  // credit card
  creditPay := CreditCard{11.12}
  payment.Process(creditPay)
}
```

จะเห็นว่าเราสามารถใช้ `Process()` รวมกันได้ทั้ง `KkuPayment` กับ `CreditCard` รวมถึงที่จะเพิ่มเติมในอนาคตได้โดยที่ไม่ต้องรื้อของเดิมที่ทำงานได้อยู่แล้ว

### **L**iskov Substitution (LSP)
หมายความว่าอ็อบเจกต์ของ super class ควรแทนที่ได้ด้วยอ็อบเจ็กต์ของ sub class โดยไม่กระทบต่อความถูกต้องของโปรแกรม สิ่งนี้ช่วยให้แน่ใจว่าความสัมพันธ์ระหว่างคลาสนั้นชัดเจนและสามารถรักษาโครงสร้างเดิมไว้ได้

ตัวอย่าง เริ่มต้นจากเรามีกาแฟ รสชาติขม 😂😂😂
```go
type Coffee struct {}

func (c Coffee) Taste() {
  fmt.Println("Bitter")
}
```

ทีนี่เราก็มีลาเต้เป็นกาแฟที่รสชาติหวาน

```go
type Late struct {
  Coffee
}

func (a Late) Taste() {
  fmt.Println("Sweet")
}
```

จาก LSP ทำให้เราต้องกำหนดว่าเมื่อมีการเขียนทับ function/method ของ super class ได้โดยไม่กระทบกับระบบเดิม

```go
type CoffeeTaste interface {
  Taste()
}

func TasteOf(cTaste CoffeeTaste) {
  cTaste.Taste()
}

coffee := Coffee{}
late := Late{}
TasteOf(a) // Bitter
TasteOf(b) // Sweet
```

จะเห็นได้ว่า `TasteOf` ที่สร้างเพิ่มมาเป็นตัวกลางซึ่งสามารถรับพารามิเตอร์ที่เป็น `CoffeeTaste` ได้ ทำให้ `Coffee` และ sub class อื่นๆ ที่ขยายมาจาก `Coffee` สามารถทำงานได้ใน function เดียวกันเนื่องจากมี function `Taste()` เหมือนกันและเป็นการรับรองว่าตัวแปรที่จะเข้ามาใน function นี้ต้องประกอบด้วย function `Taste()` เสมอ

### **I**nterface Segregation (ISP)
หมายความว่าอินเทอร์เฟซควรได้รับการออกแบบให้มีขนาดเล็กและเฉพาะเจาะจงที่สุดเท่าที่จะเป็นไปได้ สิ่งนี้ช่วยให้โค้ดมีความยืดหยุ่นและหลีกเลี่ยงการมีความสัมพันธ์ระหว่างคลาสโดยไม่จำเป็น

ตัวอย่าง เรามี interface ที่ดูแลด้าน order ประมาณนี้
```go
type Order interface {
  GetOrder()
  CreateOrder()
  GetItems()
  AddItems()
  Pay()
}
```

จะเห็นว่ามีงานหลากหลายประเภทอยู่ใน interface เดียวกันทั้งหมดซึ่งจากนิยาม ISP เราจะแบ่งออกเป็นได้แบบนี้เพื่อให้แต่ละ interface รับผิดชอบงานของตัวเองเท่านั้น
```go
type Order interface {
  GetOrder()
  CreateOrder()
}

type OrderItem interface {
  GetItems()
  AddItems()
}

type Payment interface {
  Pay()
}
```

เมื่อมีการแก้ไขหรือติดตามปัญหาเราก็สามารถแยกได้ตาม interface ที่รับผิดชอบได้ทันที

### **D**ependency Inversion (DIP)
หลักการนี้ระบุว่าโมดูลระดับบนไม่ควรขึ้นอยู่กับโมดูลระดับล่างโดยตรง แต่ทั้งสองอย่างควรขึ้นอยู่กับสิ่งที่เป็นตัวกลางร่วมกัน สิ่งนี้ช่วยลดการมีสัมพันธ์ระหว่างส่วนประกอบต่าง ๆ และทำให้โค้ดมีความยืดหยุ่นและบำรุงรักษาได้มากขึ้น

ตัวอย่าง เมนูร้านน้ำแห่งหนึ่งมีทั้ง ชา และ กาแฟอยู่ในเมนู
```go
type Menu struct {
  TeaList []Tea
  CoffeeList []Coffee
}
```

จะเห็นว่า `Menu` ขึ้นตรงกับโครงสร้างของ `Tea` และ `Coffee` เมื่อต้องการเพิ่มประเภทให้เข้ามาหรือมีการแก้ไข `Tea`, `Coffee` ก็จะทำให้โครงสร้างของ `Menu` เปลี่ยนไป จากนิยามของ DIP เราจะเปลี่ยนได้ด้วยการใช้ Interface ร่วมกันคือ `Drink`

```go
type Menu struct {
  Drinks []Drink
}

type Drink interface {
  GetCatagory() string
  GetName() string
  GetPrice() float64
}

type Coffee struct {
  Catagory string
  Name string
  AddOn float64
  Price float64
}

func(c Coffee) GetCatagory() string {
  return c.Catagory
}

func(c Coffee) GetTitle() string {
  return c.Name
}

func(c Coffee) GetPrice() string {
  return c.Price + c.AddOn
}

type Tea struct {
  Catagory string
  Title string
  Price float64
}

func(t Tea) GetCatagory() string {
  return c.Catagory
}

func(t Tea) GetTitle() string {
  return t.Title
}

func(t Tea) GetPrice() string {
  return t.Price
}
```

จะเห็นได้ว่าเมื่อทำแบบนี้แล้วไม่ว่าจะ `Tea` และ `Coffee` จะมีโครงสร้างภายในแตกต่างกันหรือเปลี่ยนไปยังไง ก็ยังคงใส่ไว้ใน `Menu` ได้ไม่กระทบกันและกันเพราะทั้งคู่ยังคงเป็น `Drink` อยู่นั่นเอง

## สรุป

SOLID เป็นเพียงแนวทางนึงในการพัฒนาซอฟต์แวร์เพื่อไม่ให้เราหลงทางไปสู่วังวน Technical Debt ทางนึง ซึ่งยังมีอีกหลายแนวทางที่ใช้กันแพร่หลายอีกเดี๋ยวค่อยมาต่อในบทความถัดไปแล้วกันนะครับ สวัสดีครับ 🙏