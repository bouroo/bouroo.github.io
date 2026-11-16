---
title: "When Go Meets SOLID"
subtitle: ""
date: 2023-07-29T09:16:31+07:00
lastmod: 2023-07-29T09:16:31+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "Writing Go code that follows SOLID principles, with a before-and-after for each of the five."
aliases:
- /posts/go_solid/
license: ""
images: []

tags: ["Go", "SOLID", "Programming"]
categories: ["Go"]

featuredImage: "featured-image.svg"
featuredImagePreview: "featured-image.svg"

lightgallery: true
---

Writing something quickly for the first version and cleaning it up later, which rarely happens, makes the next feature harder. The code ends up tightly coupled, and without a clear structure it stops scaling when the demand grows.

The SOLID guidelines are the set I use for this. The examples below are in Go, the language I know best.

<!--more-->

## Let's get to know SOLID first
SOLID is a set of software development guidelines for creating high-quality code (making it a pleasure for others to read), easy to maintain (changes in one place don't have unexpected effects on other parts), and easily scalable to meet increasing demands (can be extended without worrying about breaking existing code).

## SOLID consists of

### **S**ingle Responsibility Principle (SRP)
This principle means that each set of code should have only one responsibility and not perform multiple tasks at the same time. This helps keep the code clean and easy to maintain, as structural changes can be made in one place.

Example: Let's start with a payment processing structure:
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

`Payment` is responsible for making payments (`Payment`), creating invoices (`Invoice`), and sending bills (`Bill`). SRP says to separate these responsibilities into their own structures:

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

A `hash/crc64` package that also held `crc32` would mix two checksum algorithms in one package. The Go standard library keeps `hash/crc64` and `hash/crc32` separate instead of one `hash/crc`.

### **O**pen-Closed Principle (OCP)
This principle means that code should be extensible without modifying the original code. This helps make the code flexible and adaptable to changing requirements without breaking existing code.

Example: Let's start with a payment channel and a payment process that requires `Pay()`
```go
type PaymentChannel interface {
  Pay()
}

type Payment struct {}

func (k Payment) Process(payCh PaymentChannel) {
  payCh.Pay()
}
```

When we want to add a new payment channel, such as KKU PaymentHub:
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

Later, if we want to allow payment via credit card, we can do it like this:
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

Both `KkuPayment` and `CreditCard` work through `Process()`, as would any future payment channel, without modifying the existing code.

### **L**iskov Substitution Principle (LSP)
Code written against a `Coffee` should work on anything that substitutes for it, as long as the substitute returns the same `Taste()`.

Example: We have a bitter-tasting coffee:
```go
type Coffee struct {}

func (c Coffee) Taste() {
  fmt.Println("Bitter")
}
```

Now we have a latte, which is a sweet-tasting coffee:
```go
type Late struct {
  Coffee
}

func (a Late) Taste() {
  fmt.Println("Sweet")
}
```

According to the LSP, we can override the superclass's function without affecting the existing system:
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

`TasteOf` accepts a `CoffeeTaste`, so it runs for `Coffee` and for `Late`. Both satisfy the interface with their own `Taste()`; the caller sees `Bitter` and `Sweet`.

### **I**nterface Segregation Principle (ISP)
This principle means that interfaces should be designed to be as small and specific as possible. This helps make the code flexible and avoids unnecessary relationships between classes.

Example: We have an interface that handles ordering:
```go
type Order interface {
  GetOrder()
  CreateOrder()
  GetItems()
  AddItems()
  Pay()
}
```

This interface mixes several tasks. ISP says to split it:
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

Separating them into interfaces that are responsible for their own tasks makes it easier to modify or track down problems.

### **D**ependency Inversion Principle (DIP)
This principle states that high-level modules should not depend directly on low-level modules, but both should depend on a common abstraction. This reduces the coupling between components and makes the code more flexible and maintainable.

Example: A drink shop's menu has both tea and coffee:
```go
type Menu struct {
  TeaList []Tea
  CoffeeList []Coffee
}
```

`Menu` depends directly on the `Tea` and `Coffee` structures. Adding a new type, or changing either one, forces a change to `Menu`. DIP replaces that with a common interface, `Drink`:
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

Now the internal structure of `Tea` or `Coffee` can differ or change, and both still fit in `Menu` because both are `Drink`s.

## Conclusion

SOLID is one set of guidelines that keeps a codebase out of the technical-debt cycle. Other widely used guidelines exist and are worth reading next.
