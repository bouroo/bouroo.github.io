---
title: "GOMAXPROCS, GOMEMLIMIT and Kubernetes"
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

featuredImage: "featured-image.webp"
featuredImagePreview: "featured-image.webp"

lightgallery: true
---
The [Go runtime](https://pkg.go.dev/runtime) has a few environment variables that can be configured, but there are two that DevOps folks often use as a starting point: **GOMAXPROCS** and **GOMEMLIMIT**. Let's take a look at what each of them does.
<!--more-->

## Recap
From the [Go runtime](https://pkg.go.dev/runtime) documentation:

### GOMEMLIMIT (Go 1.19+)
{{< admonition note "Note" >}}
The `GOMEMLIMIT` variable sets a soft memory limit for the runtime. This memory limit includes the Go heap and all other memory managed by the runtime, and excludes external memory sources such as mappings of the binary itself, memory managed in other languages, and memory held by the operating system on behalf of the Go program. `GOMEMLIMIT` is a numeric value in bytes with an optional unit suffix. The supported suffixes include B, KiB, MiB, GiB, and TiB. These suffixes represent quantities of bytes as defined by the IEC 80000-13 standard. That is, they are based on powers of two: KiB means 2^10 bytes, MiB means 2^20 bytes, and so on. The default setting is [math.MaxInt64](https://pkg.go.dev/runtime/internal/math#MaxInt64), which effectively disables the memory limit. [runtime/debug.SetMemoryLimit](https://pkg.go.dev/runtime/debug#SetMemoryLimit) allows changing this limit at run time.
{{< /admonition >}}

This is the value that tells the Go runtime how much memory we have available. The default is to not enable the memory limit.

### GOMAXPROCS
{{< admonition note "Note" >}}
The `GOMAXPROCS` variable limits the number of operating system threads that can execute user-level Go code simultaneously. There is no limit to the number of threads that can be blocked in system calls on behalf of Go code; those do not count against the `GOMAXPROCS` limit. This package's [GOMAXPROCS](https://pkg.go.dev/runtime#GOMAXPROCS) function queries and changes the limit.
{{< /admonition >}}

From the above and **([GOMAXPROCS, a good friend of DevOps]({{< ref "/posts/go/automaxprocs" >}} "GOMAXPROCS, a good friend of DevOps"))**, a key point to note is that if we set a pod's CPU limit to **1 CPU** but it's running on a machine with **16 cores**, our app will default to `GOMAXPROCS=16`. This can lead to unintentional performance degradation when the workload exceeds the limit.

## Configuration when working with Kubernetes
The previous article introduced the [automaxprocs](https://github.com/uber-go/automaxprocs) package, which saves us from having to manually adjust `GOMAXPROCS`. However, we still need to manually adjust `GOMEMLIMIT`. When we deploy in Kubernetes, we can use the value from `resourceFieldRef` as shown in the example below.

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

In this example, the values will be calculated and put into the `GOMAXPROCS` and `GOMEMLIMIT` environment variables for our app as follows:
```yaml
GOMAXPROCS: 1
GOMEMLIMIT: 536870912
```
These set values will be used by the Go runtime automatically. The Go runtime uses a `0` (zero value) as the default when no value is passed. This means that even if we don't set resource limits for the Pod, the Go runtime will fall back to its default values, so we don't have to worry about the service failing to run lol.
