---
title: "เปิดเว็บเซอร์วิสสู่ชาวโลกด้วย K3S + Cloudflare Tunnel"
subtitle: ""
date: 2023-07-20T19:49:54+07:00
lastmod: 2023-07-20T19:49:54+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "บทความนี้จะแนะนำวิธีการนำเสนอเว็บเซอร์วิสของเราไปสู่โลกภายนอกด้วย K3S + Cloudflare Tunnel แบบไม่ต้องง้อ External Public IP"
aliases:
- /posts/behind_cloudflare/
license: ""
images: []
resources:
- name: "featured-image"
  src: "featured-image.webp"
- name: "featured-image-preview"
  src: "featured-image.webp"

tags: ["Cloudflare", "DevOps", "K3S", "K8S"]
categories: ["DevOps"]

lightgallery: true
---

หากเรามี home lab หรือ service ที่อยู่ใน k3s/k8s อยากเปิดให้ชาวโลกได้เข้ามาใช้งาน แต่ไม่มี public IP จะทำอย่างไรได้นะ ยิ่งในโลกที่ทุกวันนี้ ISP แจก IP แบบ Carrier-grade NAT หรือเรียกติดปากกันว่า large-scale NAT (LSN) ทำให้จะใช้ DDNS ก็ลำบากอัปเดต IP กันอีก จึงเป็นที่มาของพระเอกในบทความนี้ครับ

<!--more-->

## cloudflare tunnel คืออะไร
![Access Diagram](img/Access_Diagram.webp "Access Diagram")
Cloudflare tunnel ตามชื่อเลยคือการสร้าง tunnel จากเราเข้าไปที่ cloudflare network ทำให้ cloudflare กลายมาเป็นหน้าด่านให้ระหว่างเราและโลกภายนอก

## สร้าง cloudflare tunnel
> เราต้องมีโดเมนที่ผูกไว้กับ cloudflare ก่อนอย่างน้อยหนึ่งชื่อนะครับ

เริ่มต้นจากเปิดใช้งานที่ https://one.dash.cloudflare.com/
แล้วเข้าเมนู Access > Tunnels > Create a tunnel
![create_tunnel](img/create_tunnel.webp "create_tunnel")

ตั้งชื่อ Tunnel
![naming_tunnel](img/naming_tunnel.webp "naming_tunnel")

เอาค่า tunnel token เพื่อไปใส่ใน kube secret ตอน deploy
![tunnel_token](img/tunnel_token.webp "tunnel_token")

## deploy cloudflare tunnel บน k3s/k8s

สร้างไฟล์ `cloudflared-daemonset.yml` หน้าตาประมาณนี้ โดยเอาค่า tunnel token ไปแปลงเป็น base64 แล้วใส่ไว้ใน secret ชื่อ `cf_tunnel_token`
{{< gist bouroo 624c6cd6d515c5e0af54904dba60f073 cloudflared-daemonset.yml >}}

จากนั้น deploy ด้วย
```bash
kubectl apply -f cloudflared-daemonset.yml
```

## expose service สู่โลกภายนอก
เช็คสถานะใน cloudflare one dashboard ว่า tunnel เราเชื่อมต่อได้แล้ว
![tunnel_config](img/tunnel_config.webp "tunnel_config")

เสมือนว่าได้ kube-proxy + load balancer กันเลยทีเดียว
![kube_proxy](img/kube_proxy.webp "kube_proxy")

แล้วเราก็สามารถทำ reverse proxy เข้าไปหา servic ใน cluster เราได้เลยในเช่น `service_name.namespace` เช่น `https` ไปที่ `rancher` บน namspace `cattle-system` จะได้ตามภาพ
![ingress](img/ingress.webp "ingress")
กรณีที่ service ใช้งาน self sign SSL เราต้องตั้งให้ cloudflare skip verify ไปด้วยครับ
![tls_skip_verify](img/tls_skip_verify.webp "tls_skip_verify")


## ทดสอบเข้า service
![rancher](img/rancher.webp "rancher")
![daemonset](img/daemonset.webp "daemonset")

