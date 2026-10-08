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

{{< mermaid >}}
sequenceDiagram
    autonumber
    participant App as Mobile Banking App
    participant Pin as Pin Config API
    Note over Pin: Pin list ถูกเซ็นล่วงหน้าด้วย Ed25519<br/>(Private Key เก็บใน HSM แบบ Offline)
    App->>Pin: GET /v2/pins
    Pin-->>App: pins[] + Ed25519 Signature
    Note over App: ตรวจ Signature ด้วย Public Key<br/>ที่ฝังไว้ในแอปเสมอ ก่อนนำไปใช้
    App->>App: อัปเดต In-Memory TrustStore
{{< /mermaid >}}

#### ข้อควรระวังของ Dynamic Pinning:
- **ห้ามดึง Pin List มาตรง ๆ ผ่าน HTTP/HTTPS ธรรมดา:** เพราะหากฝั่งตรงข้ามทำ MITM ได้ เขาก็สามารถแก้ Payload Pin List ที่ส่งลงมาได้เช่นกัน
- **ต้องใช้ Asymmetric Signature (เช่น Ed25519 หรือ ECDSA):** ฝั่ง Server ต้องเซ็นกำกับ Payload ด้วย Private Key ที่เก็บไว้แบบ Offline / HSM และให้ Mobile App ใช้ Public Key ที่ฮาร์ดโค้ดไว้ในแอปในการตรวจสอบความถูกต้องก่อนอัปเดต Pin List

---

## 3. ตัวอย่างสถาปัตยกรรมแบบครบวงจร (Reference Architecture)

เมื่อนำทั้ง 3 กลยุทธ์มาประกอบร่างเข้ากับระบบ Backend ทั้งหมดจะหน้าตาประมาณนี้:

{{< mermaid >}}
flowchart LR
    subgraph APP["Mobile Banking App"]
        TS["In-Memory TrustStore<br/>Active SPKI Pin + Backup Pin"]
        VK["Ed25519 Public Key<br/>(hardcoded ใน binary)"]
    end

    subgraph EDGE["Edge / API Gateway"]
        LB["TLS Termination<br/>Leaf Cert หมุนเวียนถี่<br/>แต่ Public Key คงเดิม"]
    end

    subgraph PINSVC["Pin Distribution Service"]
        PC["Pin List JSON<br/>เซ็นด้วย Ed25519"]
    end

    subgraph OPS["Security Operations (Offline)"]
        KMS["KMS / HSM<br/>Active Key • Backup Keys<br/>Ed25519 Signing Key"]
        CICD["CI/CD + cert-manager<br/>Auto-renew แบบ Reuse Key"]
        MON["SPKI Drift Monitor<br/>เตือนล่วงหน้า 30 วัน"]
    end

    APP -- "① TLS Handshake<br/>(ตรวจ SPKI ทุกครั้ง)" --> LB
    APP -- "② ดึง Signed Pin List" --> PINSVC
    KMS -. "เซ็น payload" .-> PC
    CICD -- "③ ต่ออายุ Cert (Key เดิม)" --> LB
    MON -- "④ เทียบ Live SPKI กับ Pin ในแอป" --> LB
    MON -. "⑤ Alert" .-> CICD
{{< /mermaid >}}

| Component | บทบาทหน้าที่ |
| :--- | :--- |
| **In-Memory TrustStore** | เก็บ Active + Backup SPKI pins และตรวจทุก TLS connection ฝั่งแอป |
| **Ed25519 Public Key** | ฝังในแอปตั้งแต่ build ใช้ตรวจ signature ของ pin list ที่ดึงมาทาง OTA |
| **Edge / API Gateway** | ยุติ TLS ด้วย leaf cert ที่หมุนเวียนบ่อย แต่ยังออกจาก key pair ตัวเดิมเสมอ |
| **Pin Distribution Service** | แจก pin list เวอร์ชันล่าสุดแบบเซ็นกำกับ ผ่านหลาย CDN/endpoint |
| **KMS / HSM** | เก็บ active key, backup keys และ signing key แบบ offline |
| **CI/CD + cert-manager** | ต่ออายุ cert อัตโนมัติด้วย private key ตัวเดิม ทำให้ SPKI hash ไม่เปลี่ยน |
| **SPKI Drift Monitor** | เทียบ SPKI จริงบน production กับ pin set ที่แอปถืออยู่ แล้วเตือนก่อนเกิด incident |

---

## 4. วิธี Implement ทีละขั้นตอน

### ขั้นที่ 1 — สกัดค่า SPKI Hash

ดึงจาก server ที่ใช้งานจริง (ได้ค่า **active pin**):

```bash
openssl s_client -connect api.yourbank.com:443 -servername api.yourbank.com </dev/null 2>/dev/null \
  | openssl x509 -pubkey -noout \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256 -binary | base64
```

สกัดจาก backup key ที่ generate เก็บไว้ล่วงหน้าแบบ offline (ได้ค่า **backup pin**):

```bash
openssl pkey -in backup-2027.key -pubout -outform DER \
  | openssl dgst -sha256 -binary | base64
```

ผลลัพธ์ base64 จากสองคำสั่งนี้คือค่าที่จะฝังลงแอป — อย่าสับสนกับ fingerprint ของ certificate ทั้งใบ ซึ่งเปลี่ยนทุกครั้งที่ renew

### ขั้นที่ 2 — Android

ใช้ `network_security_config.xml` ตามตัวอย่างใน Strategy 1 สำหรับ HTTP stack ของระบบ แต่ถ้าแอปคุม connection เองด้วย OkHttp/Retrofit ให้ pin ในโค้ดด้วย:

```kotlin
val client = OkHttpClient.Builder()
    .certificatePinner(
        CertificatePinner.Builder()
            .add("api.yourbank.com",
                 "sha256/<active-pin-base64>",
                 "sha256/<backup-pin-base64>")
            .build()
    )
    .build()
```

ส่วนทีม QA ที่ต้องดีบักผ่าน proxy เปิดทางให้เฉพาะ debug build ด้วย `debug-overrides`:

```xml
<debug-overrides>
    <trust-anchors>
        <!-- ยอมรับ user CA เช่น Charles/Burp ใน debug build เท่านั้น -->
        <certificates src="user" />
    </trust-anchors>
</debug-overrides>
```

### ขั้นที่ 3 — iOS

ฝั่ง iOS ไม่มี declarative config จาก Apple โดยตรง ทางที่นิยมคือไลบรารี **TrustKit**:

```swift
import TrustKit

TrustKit.initSharedInstance(with: [
    kTSKSwizzleNetworkDelegates: false,
    kTSKPinnedDomains: [
        "api.yourbank.com": [
            kTSKEnforcePinning: true,
            kTSKIncludeSubdomains: true,
            kTSKPublicKeyHashes: [
                "<active-pin-base64>",   // SPKI SHA-256 (base64)
                "<backup-pin-base64>"
            ]
        ]
    ]
])
```

แล้วเรียก `TrustKit.sharedInstance().pinningValidator.handle(_:challenge:completionHandler:)` ภายใน `urlSession(_:didReceive:completionHandler:)` หากไม่อยากพึ่ง dependency เขียนเทียบ SPKI hash เองใน session delegate ก็ได้ (`SecTrustCopyPublicKey` + `SHA256.hash`)

### ขั้นที่ 4 — ฝั่ง Server: Auto-renew แบบ Reuse Key (cert-manager)

หัวใจอยู่ที่ `rotationPolicy: Never` — cert ใหม่ทุกใบถูกออกจาก private key ตัวเดิม SPKI hash จึงคงเดิม แอปเก่าทุกเวอร์ชันจึงเชื่อมต่อได้ต่อเนื่อง:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: api-yourbank-tls
spec:
  secretName: api-yourbank-tls
  dnsNames: ["api.yourbank.com"]
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  privateKey:
    algorithm: ECDSA
    size: 256
    rotationPolicy: Never   # ใช้ key เดิมทุกครั้งที่ renew -> SPKI hash ไม่เปลี่ยน
```

ACME รองรับการ reuse CSR แบบนี้อยู่แล้ว ส่วน AWS ACM แบบ managed จะคุม private key เองทั้งหมด ถ้าจะใช้สถาปัตยกรรมนี้บน AWS ต้อง import certificate ที่ออกเองเข้า ACM แทนการใช้ issued cert จาก ACM โดยตรง

### ขั้นที่ 5 — Dynamic Pinning: Payload, การเซ็น, และการตรวจ

โครงสร้าง payload ที่แจกจ่าย:

```json
{
  "version": 42,
  "generated_at": "2026-10-01T00:00:00Z",
  "pins": [
    { "domain": "api.yourbank.com", "type": "spki-sha256", "value": "<active-pin-base64>" },
    { "domain": "api.yourbank.com", "type": "spki-sha256", "value": "<backup-pin-base64>" }
  ]
}
```

เซ็นแบบ offline ครั้งเดียวต่อการเปลี่ยน pin (signing key ห้ามขึ้น server เด็ดขาด):

```bash
# สร้างครั้งเดียว เก็บ private key ใน HSM/offline เท่านั้น
openssl genpkey -algorithm ed25519 -out pin-signing.key
openssl pkey -in pin-signing.key -pubout   # เอาค่า public key นี้ไปฝังในแอป

# ทุกครั้งที่ pin เปลี่ยน
openssl pkeyutl -sign -inkey pin-signing.key -rawin -in pins.json -out pins.json.sig
base64 < pins.json.sig   # แนบไปกับ payload
```

ฝั่งแอปตรวจก่อนนำไปใช้เสมอ (Kotlin):

```kotlin
fun verifyPinPayload(payload: ByteArray, signature: ByteArray): Boolean {
    val pub = Base64.decode(ED25519_PUBLIC_KEY_B64, Base64.DEFAULT)
    val key = KeyFactory.getInstance("Ed25519")
        .generatePublic(X509EncodedKeySpec(pub))
    return java.security.Signature.getInstance("Ed25519").run {
        initVerify(key)
        update(payload)
        verify(signature)
    }
}
```

(`Ed25519` ใน `java.security` มีตั้งแต่ Android API 33+ / Java 15+ สำหรับแอปที่ต้องรองรับเวอร์ชันต่ำกว่า ใช้ Tink หรือ BouncyCastle)

จุดสำคัญที่มักพลาด: เสิร์ฟ endpoint นี้ผ่าน HTTPS แบบ CA-validated ปกติ (ห้าม pin channel นี้) เพื่อให้แอปดึง pin ใหม่ได้แม้ pin เดิมจะล้าสมัย — ความน่าเชื่อถือของ payload ต้องมาจาก signature ไม่ใช่ช่องทาง

### ขั้นที่ 6 — SPKI Drift Monitoring

Cron ทุกชั่วโมงเทียบ SPKI จริงบน production กับ pin set ปัจจุบัน:

```bash
LIVE=$(openssl s_client -connect api.yourbank.com:443 -servername api.yourbank.com </dev/null 2>/dev/null \
  | openssl x509 -pubkey -noout | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256 -binary | base64)

grep -q "$LIVE" pins.json \
  || curl -fsS -X POST "$ALERT_WEBHOOK" \
       -H 'Content-Type: application/json' \
       -d "{\"text\":\"P0: SPKI drift — live server key matches no pin in the app\"}"
```

ต่อเข้า CI/CD ด้วยก็ได้: ถ้ากำลัง build แอปเวอร์ชันใหม่แล้ว SPKI บน server ไม่ตรงกับ active/backup pin ตัวไหนเลย ให้ fail pipeline ทันที

---

## 5. ปรับกระบวนการฝั่ง DevOps & Certificate Lifecycle

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
- [ ] มีระบบ **SPKI Drift Monitoring** ใน Cron/CI เทียบ key จริงบน server กับ pin set ในแอปอยู่เสมอ
- [ ] (Optional) ทำระบบ Dynamic Pinning โดยมี **Asymmetric Signature Verification** เพื่ออัปเดต Pin OTA ได้ปลอดภัย

การปฏิบัติตามข้อบังคับของ ธปท. ไม่จำเป็นต้องแลกมาด้วย UX ที่แย่หรือปัญหา App Outage หากเราออกแบบระบบ Key Management และเลือกเทคนิค Pinning ให้เหมาะสมตั้งแต่สถาปัตยกรรมระดับฐานราก
