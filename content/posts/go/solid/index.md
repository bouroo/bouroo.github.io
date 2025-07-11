---
title: "เมื่อ Go กับ SOLID มาเจอกัน"
subtitle: ""
date: 2023-07-29T09:16:31+07:00
lastmod: 2023-07-29T09:16:31+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "เรามาดูกันว่าเราจะเขียน Go ตามแนวทาง SOLID ได้ยังไงกันนะ"
aliases:
- /posts/go_solid/
license: ""
images: []

tags: ["Go", "SOLID", "Programming"]
categories: ["Go"]

featuredImage: "featured-image.jpg"
featuredImagePreview: "featured-image.jpg"

lightgallery: true
---

การพัฒนาซอฟต์แวร์ในยุคปัจจุบัน หากเน้นการเร่งทำให้เสร็จเพื่อใช้งานก่อน แล้วค่อยกลับไปเก็บรายละเอียดทีหลัง (ซึ่งส่วนใหญ่ก็ไม่เคยทำได้หมด) มักจะส่งผลให้เมื่อต้องการเพิ่มฟีเจอร์ใหม่ ๆ จะเริ่มรู้สึกว่าการแก้ไขหรือเพิ่มเติมเป็นเรื่องยากลำบาก เพราะโค้ดที่เขียนขึ้นมาอาจจะเชื่อมโยงกันมากเกินไป หรือหากมีความต้องการใช้งานเพิ่มขึ้น ก็อาจจะไม่สามารถขยายระบบได้ตามความต้องการ เนื่องจากการพัฒนาซอฟต์แวร์นั้นขาดการวางแผนหรือโครงสร้างที่ชัดเจน

วันนี้เราจะมาแนะนำแนวทางในการพัฒนาซอฟต์แวร์ที่ช่วยให้โค้ดมีคุณภาพดี, ง่ายต่อการดูแลรักษา และสามารถขยายได้ตามความต้องการ แนวทางที่นิยมและเข้าใจง่ายคือ SOLID ซึ่งในตัวอย่างวันนี้จะใช้ภาษา Go เป็นหลัก (เพราะผมถนัด Go ครับ #ฮา)

<!--more-->

## มารู้จัก SOLID กันก่อน
SOLID เป็นแนวทางการพัฒนาซอฟต์แวร์เพื่อให้ได้โค้ดที่มีคุณภาพดี (ทำให้คนอ่านโค้ดรู้สึกสบายใจ), ง่ายต่อการดูแลรักษา (แก้ไขที่หนึ่งแล้วไม่ส่งผลกระทบไปยังส่วนอื่น ๆ ที่ไม่คาดคิด), และสามารถขยายได้ง่ายตามความต้องการที่เพิ่มขึ้น (สามารถต่อเติมได้โดยไม่ต้องกังวลว่าจะแก้ไขของเก่าพัง)

## SOLID ประกอบด้วย

### **S**ingle Responsibility Principle (SRP)
หลักการนี้หมายถึงโค้ดในแต่ละชุดควรมีความรับผิดชอบเพียงอย่างเดียว ไม่ทำงานหลายอย่างในเวลาเดียวกัน สิ่งนี้จะช่วยให้โค้ดสะอาดและบำรุงรักษาได้ง่าย เนื่องจากการเปลี่ยนแปลงโครงสร้างสามารถทำได้ในที่เดียว

ตัวอย่าง เริ่มจากเรามีโครงสร้างการรับชำระเงิน:
```go
type Payment struct {}

func (k Payment) MakePayment() {
  // do payment stuff
}

func (k Payment) CreateInvoice() {
  // do invoice stuff 
}

func (k Payment) SendBill() {
  // do bill mailing stuff 
}
```

จะเห็นว่า `Payment` รับผิดชอบทั้งการชำระเงิน (`Payment`), การสร้างใบแจ้งหนี้ (`Invoice`), และการส่งบิล (`Bill`) ฉะนั้นจาก SRP เราควรแยกให้โครงสร้างมีความรับผิดชอบเฉพาะตัวดังนี้:

```go
type PaymentInfo struct {}

func (k PaymentInfo) MakePayment() {
  // do payment stuff
}

type InvoiceInfo struct {}

func (k InvoiceInfo) CreateInvoice() {
  // do invoice stuff 
}

type Billing struct {}

func (k Billing) SendBill() {
  // do bill mailing stuff 
}
```

การแยกโครงสร้างเช่นนี้ทำให้โค้ดไม่มีการปะปนกันในการรับผิดชอบงานที่ไม่เกี่ยวข้อง นอกจากนี้ยังเห็นตัวอย่างที่ชัดเจนใน `Go standard library` เช่น `hash/crc64` และ `hash/crc32` ที่แยก package ออกจากกัน แทนที่จะรวมกันใน `hash/crc`

### **O**pen-Closed Principle (OCP)
หลักการนี้หมายถึงโค้ดควรสามารถขยายได้โดยไม่ต้องแก้ไขโค้ดเดิม สิ่งนี้ช่วยให้โค้ดมีความยืดหยุ่นและปรับให้เข้ากับความต้องการที่เปลี่ยนแปลงได้โดยไม่ทำให้โค้ดที่มีอยู่เสียหาย

ตัวอย่าง เริ่มจากเรามีช่องทางการชำระเงินและกระบวนการชำระเงินที่ต้องใช้ `Pay()`
```go
type PaymentChannel interface {
  Pay()
}

type Payment struct {}

func (k Payment) Process(payCh PaymentChannel) {
  payCh.Pay()
}
```

เมื่อเราต้องการเพิ่มช่องทางการชำระเงินใหม่ เช่น KKU PaymentHub:
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

ต่อมา หากเราต้องการให้สามารถชำระผ่านบัตรเครดิตได้ เราสามารถทำได้แบบนี้:
```go
type CreditCard struct {
  amount float64
}

func (cc CreditCard) Pay() {
  fmt.Printf("Paid %.2f via CreditCard", cc.amount)
}

func main() {
  payment := Payment{}
  // KKU PaymentHub
  kkuPay := KkuPayment{60.43}
  payment.Process(kkuPay)
  // Credit Card
  creditPay := CreditCard{11.12}
  payment.Process(creditPay)
}
```

จะเห็นว่าเราสามารถใช้ `Process()` ร่วมกันได้ทั้ง `KkuPayment` และ `CreditCard` รวมถึงช่องทางการชำระเงินที่จะเพิ่มเติมในอนาคต โดยไม่ต้องปรับเปลี่ยนโค้ดที่ทำงานอยู่แล้ว

### **L**iskov Substitution Principle (LSP)
หลักการนี้หมายถึงอ็อบเจกต์ของ super class ควรสามารถแทนที่ด้วยอ็อบเจ็กต์ของ sub class โดยไม่กระทบต่อความถูกต้องของโปรแกรม สิ่งนี้ช่วยให้แน่ใจว่าความสัมพันธ์ระหว่างคลาสนั้นชัดเจนและสามารถรักษาโครงสร้างเดิมไว้ได้

ตัวอย่าง เรามีกาแฟรสชาติขม:
```go
type Coffee struct {}

func (c Coffee) Taste() {
  fmt.Println("Bitter")
}
```

ทีนี้เรามีลาเต้ซึ่งเป็นกาแฟที่รสชาติหวาน:
```go
type Late struct {
  Coffee
}

func (a Late) Taste() {
  fmt.Println("Sweet")
}
```

ตามหลัก LSP เราสามารถเขียนทับฟังก์ชันของ super class ได้โดยไม่กระทบกับระบบเดิม:
```go
type CoffeeTaste interface {
  Taste()
}

func TasteOf(cTaste CoffeeTaste) {
  cTaste.Taste()
}

coffee := Coffee{}
late := Late{}
TasteOf(coffee) // Bitter
TasteOf(late)   // Sweet
```

จะเห็นว่า `TasteOf` ที่สร้างขึ้นเป็นตัวกลางสามารถรับพารามิเตอร์ที่เป็น `CoffeeTaste` ได้ ทำให้ `Coffee` และ sub class อื่น ๆ ที่ขยายมาจาก `Coffee` สามารถทำงานได้ในฟังก์ชันเดียวกัน เนื่องจากมีฟังก์ชัน `Taste()` เหมือนกัน และรับรองว่าตัวแปรที่จะเข้ามาในฟังก์ชันนี้ต้องมีฟังก์ชัน `Taste()` เสมอ

### **I**nterface Segregation Principle (ISP)
หลักการนี้หมายถึงอินเทอร์เฟซควรได้รับการออกแบบให้มีขนาดเล็กและเฉพาะเจาะจงมากที่สุดเท่าที่จะเป็นไปได้ สิ่งนี้ช่วยให้โค้ดมีความยืดหยุ่นและหลีกเลี่ยงความสัมพันธ์ระหว่างคลาสโดยไม่จำเป็น

ตัวอย่าง เรามีอินเทอร์เฟซที่ดูแลด้านการสั่งซื้อ:
```go
type Order interface {
  GetOrder()
  CreateOrder()
  GetItems()
  AddItems()
  Pay()
}
```

จะเห็นว่ามีงานหลากหลายอยู่ในอินเทอร์เฟซเดียวกัน จากนิยาม ISP เราจะแบ่งออกเป็น:
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

การแยกออกเป็นอินเทอร์เฟซที่รับผิดชอบงานของตัวเองจะช่วยให้การแก้ไขหรือติดตามปัญหาทำได้ง่ายขึ้น

### **D**ependency Inversion Principle (DIP)
หลักการนี้ระบุว่าโมดูลระดับบนไม่ควรขึ้นอยู่กับโมดูลระดับล่างโดยตรง แต่ทั้งสองอย่างควรขึ้นอยู่กับสิ่งที่เป็นตัวกลางร่วมกัน สิ่งนี้ช่วยลดความสัมพันธ์ระหว่างส่วนประกอบต่าง ๆ และทำให้โค้ดมีความยืดหยุ่นและบำรุงรักษาได้มากขึ้น

ตัวอย่าง เมนูร้านน้ำแห่งหนึ่งมีทั้งชาและกาแฟอยู่ในเมนู:
```go
type Menu struct {
  TeaList []Tea
  CoffeeList []Coffee
}
```

จะเห็นว่า `Menu` ขึ้นตรงกับโครงสร้างของ `Tea` และ `Coffee` เมื่อต้องการเพิ่มประเภทใหม่หรือมีการแก้ไข `Tea` หรือ `Coffee` ก็จะทำให้โครงสร้างของ `Menu` เปลี่ยนไป จากนิยามของ DIP เราจะเปลี่ยนได้ด้วยการใช้ Interface ร่วมกันคือ `Drink`:
```go
type Menu struct {
  Drinks []Drink
}

type Drink interface {
  GetCategory() string
  GetName() string
  GetPrice() float64
}

type Coffee struct {
  Category string
  Name     string
  AddOn    float64
  Price    float64
}

func (c Coffee) GetCategory() string {
  return c.Category
}

func (c Coffee) GetName() string {
  return c.Name
}

func (c Coffee) GetPrice() float64 {
  return c.Price + c.AddOn
}

type Tea struct {
  Category string
  Name     string
  Price    float64
}

func (t Tea) GetCategory() string {
  return t.Category
}

func (t Tea) GetName() string {
  return t.Name
}

func (t Tea) GetPrice() float64 {
  return t.Price
}
```

จะเห็นได้ว่าเมื่อใช้แนวทางนี้ ไม่ว่า `Tea` หรือ `Coffee` จะมีโครงสร้างภายในแตกต่างกันหรือเปลี่ยนไปอย่างไร ก็ยังสามารถจัดเก็บไว้ใน `Menu` ได้โดยไม่กระทบกัน เพราะทั้งคู่ยังคงเป็น `Drink` อยู่

## สรุป

SOLID เป็นแนวทางหนึ่งในการพัฒนาซอฟต์แวร์เพื่อไม่ให้เราหลงทางไปสู่วังวนของ Technical Debt และยังมีแนวทางอื่น ๆ ที่ใช้กันอย่างแพร่หลาย หากสนใจสามารถติดตามบทความถัดไปได้ สวัสดีครับ 🙏
