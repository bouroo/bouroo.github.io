---
title: "Simple Tricks for Writing Code That Won't Annoy Your Ancestors"
subtitle: ""
date: 2024-06-09T11:01:28+07:00
lastmod: 2024-06-09T11:01:28+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "Shares simple, team-friendly tricks for writing more readable Go code: flattening nested ifs, extracting complex logic into functions, removing duplication, and using names everyone understands."
license: ""
images: []

tags: ["dev"]
categories: ["dev"]

featuredImage: "featured-image.webp"
featuredImagePreview: "featured-image.webp"

lightgallery: true
---
When writing code as a team, a common headache is **"Who wrote this code?!"** To minimize friction, it's essential to have shared principles or team agreements on how to write code. Here are some simple tips that can make our lives better; please consider them.
<!--more-->
## The Problematic Code
Let's start with the problematic code that resulted from a patchwork, "it works" approach.
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

## Avoid Deeply Nested Code Blocks
This reduces the complexity of the code when reading it, so you don't have to remember which scope you're in.

### Flatten nested if statements with inversion
From the original code, you can see many nested `if` statements. Most of the time, if you're not doing separate work between `if` and `else`, you can flatten the `if` statements like this:
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

### Extract complex logic into functions
Take difficult-to-read, long logic and put it into a function with a name that describes what the logic does.
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

## Make code reusable, reduce code duplication
If there's code that's repeatedly created but doesn't perform differently, consolidate it so it can be reused from a single location.
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

## Don't use names that only you understand
When naming things in code, use terms that customers understand, teammates understand, and then we'll all understand each other.
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
    receipt["totalPrice"] := SumProductPrice(orderProducts, receipt["tax"])
    receiptMutex.Unlock()

    PrintReceipt(orderProducts, totalPrice)
    ...
}

func SumProductPrice(orderProducts []Product, tax float64) total float64 {
    for _, orderItem := range orderProducts {
        switch orderItem.Category {
            case "food":
                total += orderItem.Price + orderItem.GasPrice
            case "drink":
                total += orderItem.Price + orderItem.PackagePrice
            case "alcohol":
                total += orderItem.Price + tax
            default:
                total += orderItem.Price
        }
    }
}
```

## Conclusion
I hope these tips help you write better code and reduce the number of times people complain about it when they have to work on it. Happy coding!
