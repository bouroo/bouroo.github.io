---
title: "When Go Meets SOLID"
subtitle: ""
date: 2023-07-29T09:16:31+07:00
lastmod: 2023-07-29T09:16:31+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "Let's see how we can write Go code following the SOLID principles."
aliases:
- /posts/go_solid/
license: ""
images: []

tags: ["Go", "SOLID", "Programming"]
categories: ["Go"]

featuredImage: "featured-image.webp"
featuredImagePreview: "featured-image.webp"

lightgallery: true
---

In modern software development, focusing on getting things done quickly for initial use and then going back to clean up the details later (which is often never fully completed) usually results in difficulties when trying to add new features. This is because the code might be too tightly coupled, or if the demand for use increases, the system might not be able to scale as needed due to a lack of clear planning or structure in the software development process.

Today, we will introduce a set of guidelines for software development that helps create high-quality, easy-to-maintain, and scalable code. A popular and easy-to-understand set of guidelines is SOLID. In today's examples, we will be using the Go language (because I'm comfortable with Go, haha).

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

As you can see, `Payment` is responsible for making payments (`Payment`), creating invoices (`Invoice`), and sending bills (`Bill`). According to SRP, we should separate these responsibilities into their own structures as follows:

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

Separating the structures like this ensures that the code doesn't mix unrelated responsibilities. A clear example of this can also be seen in the `Go standard library`, such as `hash/crc64` and `hash/crc32`, which are separated into different packages instead of being combined in `hash/crc`.

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

As you can see, we can use `Process()` with both `KkuPayment` and `CreditCard`, as well as any future payment channels, without modifying the existing code.

### **L**iskov Substitution Principle (LSP)
This principle means that objects of a superclass should be replaceable with objects of a subclass without affecting the correctness of the program. This helps ensure that the relationship between classes is clear and that the original structure can be maintained.

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

As you can see, the `TasteOf` function, which acts as an intermediary, can accept a parameter of type `CoffeeTaste`. This allows `Coffee` and other subclasses that extend `Coffee` to work in the same function because they all have the `Taste()` function, and it ensures that any variable passed to this function will always have a `Taste()` function.

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

As you can see, there are various tasks in the same interface. According to the ISP, we should split it into:
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

As you can see, `Menu` depends directly on the `Tea` and `Coffee` structures. When a new type is added or `Tea` or `Coffee` is modified, the `Menu` structure will also change. According to the DIP, we can change this by using a common interface, `Drink`:
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

As you can see, with this approach, no matter how the internal structure of `Tea` or `Coffee` differs or changes, they can still be stored in the `Menu` without affecting each other because they are both still `Drink`s.

## Conclusion

SOLID is one of the guidelines for software development to prevent us from getting lost in the cycle of technical debt. There are also other widely used guidelines. If you are interested, you can follow the next article. Thank you. 🙏
