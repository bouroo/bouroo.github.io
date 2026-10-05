---
title: "Cert Pinning Crisis: วิธีรับมือข้อบังคับ ธปท. ในยุคที่ SSL Cert มีอายุสั้นลงเรื่อย ๆ"
date: 2026-10-05T17:45:00+07:00
draft: false
tags: ["Security", "Mobile App", "DevOps", "Bank of Thailand", "SSL Pinning", "TLS", "Android", "iOS"]
categories: ["Engineering", "Security"]
description: "แนวทางการวางสถาปัตยกรรม Certificate Pinning สำหรับ Mobile Banking ให้สอดคล้องตามข้อบังคับ ธปท. (BOT) โดยไม่เกิดปัญหาแอปพัง เมื่อ SSL Certificate มีอายุสั้นลงเรื่อย ๆ"

featuredImage: "featured-image.svg"
featuredImagePreview: "featured-image.svg"
---

ช่วงไม่กี่ปีที่ผ่านมา ทีมพัฒนา Mobile App และ Security Engineer ในไทย โดยเฉพาะสาย FinTech และ ธนาคาร ต้องเผชิญกับ **ความย้อนแย้งครั้งใหญ่** 2 ด้านที่วิ่งสวนทางกันอย่างชัดเจน:

1. **ข้อบังคับจาก ธนาคารแห่งประเทศไทย (ธปท. / BOT):** กำหนดให้แอปพลิเคชัน Mobile Banking / Financial Services ต้องใช้ช่องทางสื่อสารที่ปลอดภัยและทำการ **Certificate Pinning** (หรือวิธีอื่นที่เทียบเท่า) เพื่อป้องกันการโจมตีแบบ Man-in-the-Middle (MITM)
2. **มาตรฐานอุตสาหกรรม SSL/TLS (CA/Browser Forum, Apple, Google):** มีนโยบายบีบให้อายุของ SSL/TLS Certificate สั้นลงเรื่อย ๆ จากเดิมที่เคยใช้อย่างยาวนานหลายปี ปัจจุบันถูกลดลงเหลือเพียงไม่กี่เดือน (และมีแนวโน้มจะสั้นลงอีกในอนาคต)

หากทีมพัฒนาเลือกใช้วิธี **Static Leaf Certificate Pinning** แบบฮาร์ดโค้ดไฟล์ `.crt` หรือ Cert Fingerprint ลงในแอป ผลลัพธ์คือ **"แอปพังบ่อยครั้งตามรอบการหมดอายุของ Certificate"** ผู้ใช้ที่ไม่ยอมกดอัปเดตแอปผ่าน App Store / Play Store จะเข้าใช้งานไม่ได้ทันที

บทความนี้จะสรุปแนวทางการออกแบบสถาปัตยกรรม Certificate Pinning ให้ปลอดภัยตามเกณฑ์ ธปท. และมีความยืดหยุ่นสูง (Resilient) ไม่สะดุดเมื่อ Cert หมดอายุ

---

## 1. กับดักของการ Pinning แบบดั้งเดิม (The Anti-Pattern)

การทำ Certificate Pinning มี 3 ระดับหลัก ๆ ตามโครงสร้างของ Certificate Chain:

```text
[ Root CA Certificate ]       <-- ครอบคลุมสูง ปลอดภัยน้อยสุด (ถ้า Root CA ถูก Compromise ระบบหลุดหมด)
       │
       ├── [ Intermediate CA ] <-- ปลอดภัยปานกลาง ยืดหยุ่นสูง (อาจเปลี่ยนเมื่อ CA อัปเกรดระบบ)
       │
       └── [ Leaf Certificate ] <-- ปลอดภัยสูงสุด แต่เปราะบางที่สุด (เปลี่ยนบ่อยตามอายุขัย Cert)
```

ปัญหาที่พบบ่อยคือ Developer มักนำ **SHA-256 Fingerprint ของ Leaf Certificate** ไปฮาร์ดโค้ดไว้ในแอป

เมื่อ SSL Certificate มีอายุสั้นลง และต้องทำการหมุนเวียน (Rotate) บ่อยขึ้น:
- หาก Certificate เปลี่ยนแปลง Signature หรือ Serial Number
- แอปเวอร์ชันเดิมบนเครื่องผู้ใช้จะไม่รู้จัก Certificate ใหม่ทันที
- เกิดปัญหา **TLS Handshake Failure** ส่งผลกระทบระดับ P0 Outage ที่ผู้ใช้เข้าใช้งานแอปไม่ได้

---

## 2. 3 กลยุทธ์แก้ไขปัญหา (Architectural Solutions)

### Strategy 1: Pin ที่ Public Key Hash (SPKI) แทนตัว Certificate

ทางออกแรกที่เปรียบเสมือน Silver Bullet ของเรื่องนี้คือการทำ **SubjectPublicKeyInfo (SPKI) Pinning** 

> **หลักการ:** เมื่อเราต่ออายุ SSL Certificate (Renewal) เราไม่จำเป็นต้องสร้าง Key Pair ใหม่ทุกครั้ง เราสามารถสร้าง Certificate ใหม่โดยใช้ **Private Key เดิม** ได้

การทำแบบนี้จะทำให้ **Public Key Hash (SHA-256)** ของเซิร์ฟเวอร์มีค่าเดิมเสมอ แม้ว่าวันที่หมดอายุ (Validity Date) หรือ Signature ของ Certificate ใหม่จะเปลี่ยนไปก็ตาม

#### ตัวอย่างการตั้งค่า SPKI Pinning บน Android (Network Security Config)

```xml
<!-- res/xml/network_security_config.xml -->
<network-security-config>
    <domain-config>
        <domain includeSubdomains="true">api.yourbank.com</domain>
        <pin-set expiration="2027-12-31">
            <!-- Primary Public Key Hash (Active) -->
            <pin digest="SHA-256">cUPcRizA9ELLptM3a8258DAAR2DAA3k5hQAAA...</pin>
            <!-- Backup Public Key Hash 1 (Cold Storage CSR) -->
            <pin digest="SHA-256">jQJT223a8258DAAR2DAA3k5hQAAAcUPcRizA9EL...</pin>
        </pin-set>
    </domain-config>
</network-security-config>
```

---

### Strategy 2: การกำหนด Backup Pins (Mandatory Rule)

ไม่ว่าจะใช้เทคนิคใด **ห้ามระบุ Pin เพียงค่าเดียวเด็ดขาด** (OWASP Certificate Pinning Cheat Sheet บังคับให้มีอย่างน้อย 2 Pins):

1. **Active Pin:** SHA-256 ของ Public Key ปัจจุบันที่ใช้งานอยู่
2. **Backup Pin (At least 1-2 Pins):** SHA-256 ของ Public Key สำรองที่เตรียมไว้ในสภาพแวดล้อมที่ปลอดภัย (เช่น ออกแบบ CSR ทิ้งไว้แบบ Offline หรือเก็บไว้ใน Key Management Service)

หากเกิดเหตุฉุกเฉิน เช่น Private Key ปัจจุบันหลุด (Compromised) หรือต้องการสลับไปใช้ Certificate สำรอง ระบบฝั่ง Server สามารถสลับไปใช้ Key สำรองที่แอปถือสำรองไว้อยู่แล้วได้ทันที โดยไม่ต้องรอให้ผู้ใชัปเดตแอป

---

### Strategy 3: Dynamic Pinning ผ่าน Remote Config แบบ Signed Payload

หากต้องการความยืดหยุ่นระดับสูงสุด และไม่ต้องคอย Release แอปเวอร์ชันใหม่ผ่าน Store สามารถเลือกใช้ **Dynamic Pinning** ได้ แต่ต้องทำอย่างถูกต้องเพื่อไม่ให้ขัดกับหลัก Security:

```text
+------------------+       1. Request Pin List       +-------------------+
|                  | ------------------------------> |                   |
|  Mobile Banking  |                                 | Remote Config API |
|       App        | <------------------------------ |                   |
|                  |     2. Return Signed Pins       +-------------------+
+------------------+     (Payload + Ed25519 Sig)
         │
         ├── 3. Verify Signature using Public Key inside App
         └── 4. Update In-Memory TrustStore
```

#### ข้อควรระวังของ Dynamic Pinning:
- **ห้ามดึง Pin List มาตรง ๆ ผ่าน HTTP/HTTPS ธรรมดา:** เพราะหากฝั่งตรงข้ามทำ MITM ได้ เขาก็สามารถแก้ Payload Pin List ที่ส่งลงมาได้เช่นกัน
- **ต้องใช้ Asymmetric Signature (เช่น Ed25519 หรือ ECDSA):** ฝั่ง Server ต้องเซ็นกำกับ Payload ด้วย Private Key ที่เก็บไว้แบบ Offline / HSM และให้ Mobile App ใช้ Public Key ที่ฮาร์ดโค้ดไว้ในแอปในการตรวจสอบความถูกต้องก่อนอัปเดต Pin List

---

## 3. ปรับกระบวนการฝั่ง DevOps & Certificate Lifecycle

เทคนิคข้างต้นจะสำเร็จได้ ทีม Infra / DevOps ต้องทำงานสอดคล้องกับทีม Mobile App ด้วย:

| กระบวนการ | แนวทางปฏิบัติที่แนะนำ |
| :--- | :--- |
| **Cert Renewal Policy** | กำหนดในระบบอัตโนมัติ (เช่น Let's Encrypt / Cert-Manager / AWS ACM) ให้ใช้ **Key Reuse Strategy** (คงใช้ Key Pair เดิมในการหมุนเวียนใบรับรองอัตโนมัติ) |
| **Key Rotation Schedule** | หากต้องการ Rotate Public Key ตามนโยบายความปลอดภัย ให้ทำตามรอบ **App Release Cycle** เช่น แปะ Backup Pin ใหม่ลงในแอปตั้งแต่วันนี้ และเปิดใช้งาน Key นั้นจริงในอีก 6-12 เดือนข้างหน้า |
| **Monitoring & Alerting** | ตั้งระบบเตือนล่วงหน้า 30 วันก่อน Cert หมดอายุ เพื่อตรวจสอบว่า Key Pair ปัจจุบันตรงกับ Pin ในแอปเวอร์ชันล่าสุดหรือไม่ |

---

## สรุป Checklists สำหรับ Engineering Team

- [ ] เลิกใช้ Leaf Certificate Pinning เปลี่ยนมาใช้ **SPKI (Public Key Hash) Pinning**
- [ ] ฝั่ง Mobile App มีการใส่ **Backup Pin** อย่างน้อย 1-2 ค่าเสมอ
- [ ] ฝั่ง DevOps มีการตั้งค่า Auto-renewal โดย **Reuse Private Key**
- [ ] มีขั้นตอน Playbook ชัดเจนหากเกิด Emergency Key Rotation
- [ ] (Optional) ทำระบบ Dynamic Pinning โดยมี **Asymmetric Signature Verification** เพื่ออัปเดต Pin OTA ได้ปลอดภัย

การปฏิบัติตามข้อบังคับของ ธปท. ไม่จำเป็นต้องแลกมาด้วย UX ที่แย่หรือปัญหา App Outage หากเราออกแบบระบบ Key Management และเลือกเทคนิค Pinning ให้เหมาะสมตั้งแต่สถาปัตยกรรมระดับฐานราก
