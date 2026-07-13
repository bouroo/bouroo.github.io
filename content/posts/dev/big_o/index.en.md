---
title: "Understanding Big O Notation"
subtitle: ""
date: 2024-08-07T23:13:16+07:00
lastmod: 2024-08-07T23:13:16+07:00
draft: true
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "A practical, beginner-friendly introduction to Big O notation covering O(1), O(n), O(log n), O(n²), O(n log n), and O(2ⁿ) with short Go examples for each."
license: ""
images: []

tags: []
categories: []

featuredImage: "/img/featured-image.webp"
featuredImagePreview: "/img/featured-image.webp"

lightgallery: true
---
## Why should we care about Big O Notation?

Think of Big O Notation as a "fuel gauge" for your code's performance. It tells us how the performance of a program will change as the size of the data increases. This is crucial for building applications that run smoothly, even when dealing with massive amounts of data.

<!--more-->

## Let's understand Big O Notation in detail

* **Big O Notation indicates the worst-case scenario.** It's a way to express how the number of operations your program performs will grow relative to the size of the input data.
* **We use symbols like O(n), O(n^2), O(log n), etc.** to represent different growth rates.

## Common Big O Notations

### O(1) - Constant Time
The time taken for an operation does not depend on the size of the input data.

```go
package main

import "fmt"

func main() {
    slice := []int{1, 2, 3, 4, 5}
    fmt.Println(slice[2]) // O(1)
}
```

### O(n) - Linear Time
The time taken for an operation increases linearly with the size of the input data.

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

### O(log n) - Logarithmic Time
The time taken for an operation increases logarithmically with the size of the input data.

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

### O(n^2) - Quadratic Time
The time taken for an operation increases quadratically with the size of the input data.

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

### O(n log n) - Linearithmic Time

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

* **O(2^n) - Exponential Time:**

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

## Conclusion
* Understanding Big O Notation is crucial for writing efficient code.
* We should strive to use algorithms with lower time complexity, especially when dealing with large datasets.
* Using tools like code profiling can help analyze code performance and identify potential bottlenecks.
