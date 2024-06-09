---
title: "Linux Sysctl Tuning"
subtitle: ""
date: 2023-07-19T23:56:31+07:00
lastmod: 2023-07-19T23:56:31+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "ปรับตั้งค่าใน sysctl เพื่อให้ Linux server ทำงานได้ราบลื่นเมื่อมีโหลดมากขึ้น"
aliases:
- /posts/linux_sysctl_tuning/
license: ""
images: []
resources:
- name: "featured-image"
  src: "featured-image.webp"

tags: ["Linux", "DevOps", "PVE", "K3S", "K8S"]
categories: ["Linux", "DevOps"]

featuredImage: "featured-image.webp"
featuredImagePreview: "featured-image"
---

เราสามารถปรับตั้งค่าใน sysctl เพื่อให้ Linux server ทำงานได้ราบลื่นเมื่อมีโหลดมากขึ้น โดยปกติแล้ว Linux ในแต่ละ Distro จะมีการตั้งค่า sysctl มาให้กลางอยู่แล้วเช่นสาย RHEL อาจจะปรับมาเพื่อให้บริการเป็นเครื่องแม่ข่ายเป็นพิเศษ DEB อาจจะปรับมาเพื่อให้ทำงานได้อย่างบาลานซ์ เป็นต้น ซึ่งในบทความนี้ผมจะมาแนะนำค่าที่ผมใช้งานอยู่ใน Production ของงานแต่ละประเภทดังนี้ครับ

<!--more-->

## Sysctl สำหรับเซิฟเวอร์ทั่วไป `60-sysctl.conf`
> สามารถนำค่าไปไว้ที่ `/etc/sysctl.d/60-sysctl.conf`

{{< gist bouroo bc52ad58a6e75d44e5235b229e9ca988 60-sysctl.conf >}}

## Sysctl เพิ่มเติมสำหรับ Promox VE `80-pve.conf`
> สามารถนำค่าไปไว้ที่ `/etc/sysctl.d/80-pve.conf`

{{< gist bouroo bc52ad58a6e75d44e5235b229e9ca988 80-pve.conf >}}

## Sysctl เพิ่มเติมสำหรับ K3S, K8S `80-k8s-ipvs.conf`
> สามารถนำค่าไปไว้ที่ `/etc/sysctl.d/80-k8s-ipvs.conf`

{{< gist bouroo bc52ad58a6e75d44e5235b229e9ca988 80-k8s-ipvs.conf >}}

## Apply ค่า Sysctl
```bash
sysctl --system
```
> สำหรับเครื่องที่มีการใช้งาน containerd, k3s, k8s อยู่ต้อง restart containerd, k3s, k8s service ด้วย