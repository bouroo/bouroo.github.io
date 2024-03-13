---
title: "GOMAXPROCS, GOMEMLIMIT กับ Kubernetes"
subtitle: ""
date: 2024-03-13T00:45:21+07:00
lastmod: 2024-03-13T00:45:21+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: ""
license: ""
images: []

tags: ["Go", "kubernetes", "DevOps"]
categories: ["Go"]

featuredImage: "go-featured-image.webp"
featuredImagePreview: "go-featured-image.webp"

lightgallery: true
---
[Go runtime](https://pkg.go.dev/runtime) มีค่า ENV ที่สามารถตั้งค่าได้อยู่ไม่กี่ตัว แต่มันจะมีอยู่ 2 ค่าที่ชาว DevOps จะเอามาใช้เป็นค่าเริ่มต้นเลย คือ **GOMAXPROCS** และ **GOMEMLIMIT** โดยแต่ละค่าจะใช้ทำอะไรได้บ้างเรามาลองดูกัน
<!--more-->

## Recap
จาก [Go runtime](https://pkg.go.dev/runtime) ได้อธิบายไว้ว่า

### GOMEMLIMIT (Go 1.19+)
{{< admonition note "Note" >}}
The `GOMEMLIMIT` variable sets a soft memory limit for the runtime. This memory limit includes the Go heap and all other memory managed by the runtime, and excludes external memory sources such as mappings of the binary itself, memory managed in other languages, and memory held by the operating system on behalf of the Go program. `GOMEMLIMIT` is a numeric value in bytes with an optional unit suffix. The supported suffixes include B, KiB, MiB, GiB, and TiB. These suffixes represent quantities of bytes as defined by the IEC 80000-13 standard. That is, they are based on powers of two: KiB means 2^10 bytes, MiB means 2^20 bytes, and so on. The default setting is [math.MaxInt64](https://pkg.go.dev/runtime/internal/math#MaxInt64), which effectively disables the memory limit. [runtime/debug.SetMemoryLimit](https://pkg.go.dev/runtime/debug#SetMemoryLimit) allows changing this limit at run time.
{{< /admonition >}}

ซึ่งก็คือค่าที่บอก Go runtime ว่าเรามี memory ให้ใช้งานเท่าไหร่ ซึ่งค่า default จะเป็นไม่เปิดใช้ memory limit นั่นเอง

### GOMAXPROCS
{{< admonition note "Note" >}}
The `GOMAXPROCS` variable limits the number of operating system threads that can execute user-level Go code simultaneously. There is no limit to the number of threads that can be blocked in system calls on behalf of Go code; those do not count against the `GOMAXPROCS` limit. This package's [GOMAXPROCS](https://pkg.go.dev/runtime#GOMAXPROCS) function queries and changes the limit.
{{< /admonition >}}

จากข้างต้นและ **([GOMAXPROCS เพื่อนที่ดีของ DevOps]({{< ref "/posts/go/automaxprocs" >}} "GOMAXPROCS เพื่อนที่ดีของ DevOps"))** แล้วจะมีจุดสังเกตุว่าถ้าเราตั้ง limit cpu ของ pod เป็น **1 CPU** แต่ทำงานอยู่บนเครื่องที่มี CPU **16 core** ค่าเริ่มต้นของแอปเราจะทำงานด้วย `GOMAXPROCS=16` ส่งผลให้ performance ตกแบบไม่ตั้งใจเมื่อมีการทำงานที่เกิน limit

## การตั้งค่าเมื่อทำงานกับ kubernetes
จากบทความก่อนหน้าได้แนะนำ package [automaxprocs](https://github.com/uber-go/automaxprocs) ไป ซึ่งทำให้เราไม่ต้องมานั่งปรับค่า `GOMAXPROCS` เอง แต่ว่าตอนนี้เรายังต้องมานั่งปรับ `GOMEMLIMIT` เองอยู่ ซึ่งพอเรา deploy ใน kube เราสามารถใช้ค่าจาก `resourceFieldRef` ได้ ด้วยตัวอย่างด้านล่างนี้

```yaml
...
    resources:
      requests:
        cpu: 50m
        memory: 64Mi
      limits:
        cpu: 1000m
        memory: 512Mi
    env:
    - name: GOMEMLIMIT
      valueFrom:
        resourceFieldRef:
          resource: limits.memory
    - name: GOMAXPROCS
      valueFrom:
        resourceFieldRef:
          resource: limits.cpu
```

ซึ่งค่าตัวอย่าง ในแอปเราจะโดนคำนวนเอาไปใส่ใน ENV `GOMAXPROCS` และ `GOMEMLIMIT` ให้เลยตามนี้
```yaml
GOMAXPROCS: 1
GOMEMLIMIT: 536870912
```
ซึ่งค่าที่ตั้งเข้าไปจะถูกนำไปใช้กับ Go runtime ให้เอง โดย Go runtime จะใช้ค่า `0` 
(zero value) เป็น default เมื่อไม่ได้มีการส่งค่าเข้าไป ทำให้ถึงเราจะไม่ได้ตั้ง resources limit ให้กับ Pod ไว้ Go runtime ก็จะ fallback ไปใช้ค่า default ให้เอง เราก็ไม่ต้องกลัวว่า service จะรันไม่ขึ้น ถถถ