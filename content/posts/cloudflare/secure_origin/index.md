---
title: "ซ่อนเว็บเซอร์วิสไว้ข้างหลัง Cloudflare"
date: 2023-07-15T21:57:40+07:00
lastmod: 2023-07-16T14:45:40+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin-vir.pages.dev"
description: "บทความนี้จะแนะนำวิธีการซ่อนเว็บของเราไว้ข้างหลัง cloudflare อย่างมิดชิด"
aliases:
- /posts/go_solid/
images: []
resources:
- name: "featured-image"
  src: "featured-image.webp"
- name: "featured-image-preview"
  src: "featured-image.webp"

tags: ["Cloudflare", "DevOps", "DevOpsSec"]
categories: ["DevOps"]

hiddenFromHomePage: false
hiddenFromSearch: false
twemoji: false
lightgallery: true
ruby: true
fraction: true
fontawesome: true
linkToMarkdown: true
rssFullText: false

toc:
  enable: true
  auto: true
code:
  copy: true
  maxShownLines: 50
math:
  enable: false
  # ...
mapbox:
  # ...
share:
  enable: true
  # ...
comment:
  enable: true
  # ...
library:
  css:
    # someCSS = "some.css"
    # located in "assets/"
    # Or
    # someCSS = "https://cdn.example.com/some.css"
  js:
    # someJS = "some.js"
    # located in "assets/"
    # Or
    # someJS = "https://cdn.example.com/some.js"
seo:
  images: []
  # ...
---

<!--more-->

## Cloudfalre คืออะไร ?

Cloudflare เป็นเครือข่ายเซิร์ฟเวอร์ทั่วโลก เมื่อเราเพิ่มแอปพลิเคชันของเราไปยัง Cloudflare เราจะใช้เครือข่ายนี้เพื่อทำหน้าที่ระหว่างคำขอและเซิร์ฟเวอร์ต้นทางของเรา

Cloudflare อยู่ระหว่างคำขอและเซิร์ฟเวอร์ต้นทางของเรา
ตำแหน่งนี้ช่วยให้เราทำสิ่งต่าง ๆ ได้หลายอย่าง เช่น เพิ่มความเร็วในการจัดส่งเนื้อหาและประสบการณ์ของผู้ใช้ (CDN) ปกป้องเว็บไซต์ของเราจากกิจกรรมที่เป็นอันตราย (DDoS, ไฟร์วอลล์) กำหนดเส้นทางการรับส่งข้อมูล (โหลดบาลานซ์ ห้องรอคิว) และอื่น ๆ

## Cloudflare ทำงานอย่างไร ?

ด้วย Cloudflare หมายความว่าโดเมนหรือโดเมนย่อยของเรากำลังใช้ระเบียน DNS พร็อกซี การค้นหา DNS สำหรับ URL ของแอปพลิเคชันของเราจะแปลงเป็น Cloudflare Anycast IP แทนเป้าหมาย DNS ดั้งเดิม

### ก่อนใช้งานผ่าน cloudflare proxy

{{< mermaid >}}
flowchart LR
  user(Users)
  srv(Servers)
    user <-- http request --> srv
{{< /mermaid >}}

### หลังจากใช้งานผ่าน cloudflare proxy

{{< mermaid >}}
flowchart LR
  user(Users)
  cf(Cloudflare)
  srv(Servers)
    user <-- http request --> cf
    cf <-- http request --> srv
{{< /mermaid >}}

## ซ่อนเว็บเซอร์วิสไว้ข้างหลัง Cloudflare อย่างจริงจัง

จากภาพด้านบนเหมือนว่าเว็บเราจะปลอดภัยจากโลกภายนอกแล้วใช่ไหมครับ แต่เดี๋ยวก่อนถ้าหากสังเกตดี ๆ จะพบว่าถ้าก่อนหน้านี้มีคนรู้ IP ของเซิฟเวอร์เราไปแล้วละ เว็บเราก็ก็ยังโดนยิงเข้ามาได้ถูกไหมครับ เพราะฉะนั้นวิธีที่ง่ายที่สุดคือ เราก็เปิดให้เฉพาะ Cloudflare เข้าถึงเว็บเซอร์วิสของเราได้โดยตรงแต่เพิ่งผู้เดียวไปเลยสิ โดยเราจะอาศัยไฟร์วอลของระบบปฏิบัติการนี่แหละ

{{< mermaid >}}
flowchart LR
  user(Users)
  cf(Cloudflare)
  srv(Servers)
    user <-- http request --> cf
    user -- http request --x srv
    cf <-- http request --> srv
{{< /mermaid >}}

## เปิดให้เฉพาะ Cloudflare เข้าถึงเว็บเรา

โดยจะขอยกตัวอย่างจากของ Debian 12 นะครับ
> ต้องใช้สิทธิ์ `root` นะครับ

### ติดตั้ง ipset และผองเพื่อน
```bash
# install ipset
apt -y install ipset iptables-persistent ipset-persistent curl
```

### เพิ่ม IPs ของ Cloudflare เข้า ipset
```bash
# get cloudflare IPs
mkdir -p /etc/zones/
curl -L -s https://www.cloudflare.com/ips-v4 -o /etc/zones/cf-ips-v4
curl -L -s https://www.cloudflare.com/ips-v6 -o /etc/zones/cf-ips-v6

# add cloudflare IPs to ipset
ipset -N cloudflare-ips-v4 hash:net
for i in $(cat /etc/zones/cf-ips-v4 ); do ipset -A cloudflare-ips-v4 $i; done
ipset -N cloudflare-ips-v6 hash:net family inet6
for i in $(cat /etc/zones/cf-ips-v6 ); do ipset -A cloudflare-ips-v6 $i; done
```

### เปิด http,https ให้เฉพาะ Cloudflare
```bash
# allow only http/https from cloudflare ipv4
iptables -I INPUT -p tcp -m set --match-set cloudflare-ips-v4 src -m multiport --dports 80,443,8080,8443 -j ACCEPT
iptables -I INPUT -p udp -m set --match-set cloudflare-ips-v4 src -m multiport --dports 80,443,8080,8443 -j ACCEPT
iptables -A INPUT -p tcp -m multiport --dports 80,443,8080,8443 -j DROP
iptables -A INPUT -p udp -m multiport --dports 80,443,8080,8443 -j DROP

# allow only http/https from cloudflare ipv6
ip6tables -I INPUT -p tcp -m set --match-set cloudflare-ips-v6 src -m multiport --dports 80,443,8080,8443 -j ACCEPT
ip6tables -I INPUT -p udp -m set --match-set cloudflare-ips-v6 src -m multiport --dports 80,443,8080,8443 -j ACCEPT
ip6tables -A INPUT -p tcp -m multiport --dports 80,443,8080,8443 -j DROP
ip6tables -A INPUT -p udp -m multiport --dports 80,443,8080,8443 -j DROP
```

### บันทึก
```bash
# save ipset configurations to /etc/iptables/ipsets
dpkg-reconfigure ipset-persistent

# save iptables and ip6tables configurations to /etc/iptables/rules.v{4|6}
dpkg-reconfigure iptables-persistent
```

## ทดสอบ
```bash
curl -L -v -H "Host: your.domain" your.server.ip.address
```

## Bash script `behind_cloudflare.sh`
{{< gist bouroo 6877eb823c2eee859e50ffeef67ea19e behind_cloudflare.sh >}}