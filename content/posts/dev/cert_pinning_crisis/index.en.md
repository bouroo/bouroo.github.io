---
title: "Cert Pinning Crisis: Handling Short SSL Lifespans Without Breaking Your Mobile App"
date: 2026-10-05T17:50:00+07:00
draft: false
tags: ["Security", "Mobile App", "DevOps", "Bank of Thailand", "SSL Pinning", "TLS", "Android", "iOS"]
categories: ["Engineering", "Security"]
description: "How to architect resilient Certificate Pinning for mobile banking apps to comply with Bank of Thailand (BOT) mandates while adapting to shorter SSL certificate lifespans."

featuredImage: "featured-image.svg"
featuredImagePreview: "featured-image.svg"
---

Over the past few years, mobile development and security engineering teams in Thailand—especially in FinTech and banking—have faced a major operational paradox:

1. **Bank of Thailand (BOT) Regulatory Mandate:** Mobile banking and financial apps are required to enforce **Certificate Pinning** (or equivalent security controls) to mitigate Man-in-the-Middle (MITM) attacks.
2. **Evolving WebPKI Standards (CA/Browser Forum, Apple, Google):** Industry-wide policy shifts have squeezed SSL/TLS certificate lifespans down from years to increasingly shorter validity periods.

If an engineering team relies on traditional **Static Leaf Certificate Pinning**—hardcoding the `.crt` file or fingerprint directly into the client app—the result is predictable: **frequent outages** for users who haven't enabled auto-updates on the App Store or Google Play Store.

This article outlines how to design a resilient Certificate Pinning architecture that complies with BOT standards without sacrificing system availability or user experience.

---

## 1. The Anti-Pattern: Hardcoded Leaf Certificate Pinning

Certificate Pinning can target three different levels of the Certificate Chain:

```text
[ Root CA Certificate ]       <-- Broadest coverage, least secure (Compromised CA trusts all issued certs)
       │
       ├── [ Intermediate CA ] <-- Moderate security, higher flexibility (Risk if CA rotates intermediates)
       │
       └── [ Leaf Certificate ] <-- Maximum security, highest fragility (Rotates frequently)
```

The most common anti-pattern occurs when developers hardcode the **SHA-256 Fingerprint of the Leaf Certificate** into the application binary.

As certificates expire on shorter lifespans and rotate more frequently:
- The Certificate Signature or Serial Number changes.
- Unupdated mobile app clients reject the new certificate during the **TLS Handshake**.
- A **Severity 1 Outage** occurs, locking active users out of the application.

---

## 2. Three Architectural Strategies for Resilience

### Strategy 1: SubjectPublicKeyInfo (SPKI) Pinning

Instead of pinning the entire certificate, pin the **SubjectPublicKeyInfo (SPKI)** hash of the certificate's Public Key.

> **Key Concept:** When renewing an SSL/TLS certificate, you do not need to generate a new Key Pair. You can issue a new certificate using the **existing Private Key**.

By reusing the Private Key during renewal, the **Public Key Hash (SHA-256)** remains identical, allowing legacy client apps to establish secure TLS connections seamlessly even after the certificate itself has been renewed.

#### Android Example (`res/xml/network_security_config.xml`)

```xml
<network-security-config>
    <domain-config>
        <domain includeSubdomains="true">api.yourbank.com</domain>
        <pin-set expiration="2027-12-31">
            <!-- Primary Public Key Hash (Active Key) -->
            <pin digest="SHA-256">cUPcRizA9ELLptM3a8258DAAR2DAA3k5hQAAA...</pin>
            <!-- Backup Public Key Hash (Cold Storage CSR) -->
            <pin digest="SHA-256">jQJT223a8258DAAR2DAA3k5hQAAAcUPcRizA9EL...</pin>
        </pin-set>
    </domain-config>
</network-security-config>
```

---

### Strategy 2: Mandatory Backup Pins

Never configure a single pin in your mobile client. The **OWASP Certificate Pinning Guidelines** mandate maintaining at least two pins:

1. **Active Pin:** SHA-256 hash of the currently active Public Key.
2. **Backup Pin(s):** SHA-256 hash of an offline, pre-generated Public Key stored securely in a Key Management Service (KMS) or hardware security module (HSM).

If an emergency key rotation is required (e.g., active key compromise or infrastructure migration), the server can instantly switch to the backup key without requiring an emergency app store release.

---

### Strategy 3: Dynamic Pinning via Signed Remote Configuration

For maximum agility, organizations can deploy **Dynamic Pinning**, allowing client applications to update their pin lists Over-The-Air (OTA):

{{< mermaid >}}
sequenceDiagram
    autonumber
    participant App as Mobile Banking App
    participant Pin as Pin Config API
    Note over Pin: Pin list is pre-signed with Ed25519<br/>(Private Key kept offline in an HSM)
    App->>Pin: GET /v2/pins
    Pin-->>App: pins[] + Ed25519 Signature
    Note over App: Always verify the Signature first<br/>with the Public Key bundled in the app
    App->>App: Update In-Memory TrustStore
{{< /mermaid >}}

#### Critical Security Constraints for Dynamic Pinning:
- **Never fetch pin lists over unauthenticated HTTP/HTTPS:** An attacker performing a MITM attack could intercept and manipulate the incoming payload.
- **Enforce Asymmetric Signature Verification (e.g., Ed25519 / ECDSA):** The backend must cryptographically sign the configuration payload using an offline Private Key. The mobile app verifies the signature using a hardcoded Public Key before updating its in-memory TrustStore.

---

## 3. Reference Architecture

Putting all three strategies together with the surrounding backend, the full picture looks like this:

{{< mermaid >}}
flowchart LR
    subgraph APP["Mobile Banking App"]
        TS["In-Memory TrustStore<br/>Active SPKI Pin + Backup Pin"]
        VK["Ed25519 Public Key<br/>(hardcoded in binary)"]
    end

    subgraph EDGE["Edge / API Gateway"]
        LB["TLS Termination<br/>Short-lived leaf certs<br/>rotated on the same key pair"]
    end

    subgraph PINSVC["Pin Distribution Service"]
        PC["Pin List JSON<br/>signed with Ed25519"]
    end

    subgraph OPS["Security Operations (Offline)"]
        KMS["KMS / HSM<br/>Active Key • Backup Keys<br/>Ed25519 Signing Key"]
        CICD["CI/CD + cert-manager<br/>Auto-renew with Key Reuse"]
        MON["SPKI Drift Monitor<br/>30-day pre-expiry alert"]
    end

    APP -- "① TLS Handshake<br/>(SPKI check on every call)" --> LB
    APP -- "② Fetch Signed Pin List" --> PINSVC
    KMS -. "signs payload" .-> PC
    CICD -- "③ Renew Cert (same key)" --> LB
    MON -- "④ Compare live SPKI vs app pins" --> LB
    MON -. "⑤ Alert" .-> CICD
{{< /mermaid >}}

| Component | Role |
| :--- | :--- |
| **In-Memory TrustStore** | Holds the Active + Backup SPKI pins and validates every TLS connection the app makes |
| **Ed25519 Public Key** | Compiled into the app; verifies the signature of OTA-fetched pin lists |
| **Edge / API Gateway** | Terminates TLS with frequently rotated leaf certs issued from the same key pair |
| **Pin Distribution Service** | Serves the latest signed pin list across multiple CDN endpoints |
| **KMS / HSM** | Stores the active key, backup keys, and signing key offline |
| **CI/CD + cert-manager** | Auto-renews certificates with the same private key, keeping the SPKI hash stable |
| **SPKI Drift Monitor** | Compares the live production SPKI against the pins shipped in apps and alerts before an incident |

---

## 4. Implementation Guide, Step by Step

### Step 1 — Extract the SPKI Hashes

From the live server (yields the **active pin**):

```bash
openssl s_client -connect api.yourbank.com:443 -servername api.yourbank.com </dev/null 2>/dev/null \
  | openssl x509 -pubkey -noout \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256 -binary | base64
```

From a pre-generated offline backup key (yields the **backup pin**):

```bash
openssl pkey -in backup-2027.key -pubout -outform DER \
  | openssl dgst -sha256 -binary | base64
```

The base64 output of these two commands is what you embed in the app — not the fingerprint of the whole certificate, which changes on every renewal.

### Step 2 — Android

Use `network_security_config.xml` (see Strategy 1) for the platform HTTP stack. If the app manages its own connections with OkHttp/Retrofit, pin in code as well:

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

To let QA debug through a proxy, open a debug-build-only escape hatch with `debug-overrides`:

```xml
<debug-overrides>
    <trust-anchors>
        <!-- Accept user CAs (Charles/Burp) in debug builds only -->
        <certificates src="user" />
    </trust-anchors>
</debug-overrides>
```

### Step 3 — iOS

Apple ships no declarative pinning config; the common approach is the **TrustKit** library:

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

Then call `TrustKit.sharedInstance().pinningValidator.handle(_:challenge:completionHandler:)` inside `urlSession(_:didReceive:completionHandler:)`. Without a dependency, you can also compare the SPKI hash yourself in the session delegate (`SecTrustCopyPublicKey` + `SHA256.hash`).

### Step 4 — Server Side: Auto-Renew with Key Reuse (cert-manager)

The crux is `rotationPolicy: Never` — every renewed certificate is issued from the same private key, so the SPKI hash never changes and every app version keeps connecting:

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
    rotationPolicy: Never   # reuse the same key on every renewal -> stable SPKI hash
```

ACME supports CSR reuse out of the box. Note that fully managed AWS ACM certificates rotate keys on their own schedule; on AWS, import your self-issued certificate into ACM instead of using an ACM-issued one.

### Step 5 — Dynamic Pinning: Payload, Signing, and Verification

The payload you distribute:

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

Sign offline, once per pin change (the signing key never touches a server):

```bash
# One-time setup; keep the private key in an HSM/offline only
openssl genpkey -algorithm ed25519 -out pin-signing.key
openssl pkey -in pin-signing.key -pubout   # embed this public key in the app

# Every time the pins change
openssl pkeyutl -sign -inkey pin-signing.key -rawin -in pins.json -out pins.json.sig
base64 < pins.json.sig   # ship alongside the payload
```

Client-side verification (Kotlin):

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

(`Ed25519` in `java.security` requires Android API 33+ / Java 15+; use Tink or BouncyCastle to support older versions.)

The most commonly missed point: serve this endpoint over plain CA-validated HTTPS — do **not** pin this channel — so the app can fetch fresh pins even when the old pins are stale. The payload's trust comes from the signature, not the channel.

### Step 6 — SPKI Drift Monitoring

An hourly cron comparing the live production SPKI against the current pin set:

```bash
LIVE=$(openssl s_client -connect api.yourbank.com:443 -servername api.yourbank.com </dev/null 2>/dev/null \
  | openssl x509 -pubkey -noout | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256 -binary | base64)

grep -q "$LIVE" pins.json \
  || curl -fsS -X POST "$ALERT_WEBHOOK" \
       -H 'Content-Type: application/json' \
       -d "{\"text\":\"P0: SPKI drift — live server key matches no pin in the app\"}"
```

Wire it into CI as well: if a new app build is about to ship and the live server SPKI matches neither the active nor the backup pin, fail the pipeline immediately.

---

## 5. Aligning DevOps & Certificate Lifecycle Management

Technical controls on the mobile client must be supported by automated backend infrastructure workflows:

| Area | Recommended Practice |
| :--- | :--- |
| **Cert Renewal Policy** | Configure ACME clients (`cert-manager`, Let's Encrypt, AWS ACM) to enforce a **Key Reuse Strategy** during automated certificate renewals. |
| **Key Rotation Schedule** | Synchronize Public Key rotations with planned **App Release Cycles** (e.g., bundle new backup pins 6–12 months before activating the corresponding backend key). |
| **Monitoring & Alerting** | Implement automated monitoring 30 days prior to certificate expiration to verify that the active server public key matches the deployed mobile pin sets. |

---

## Summary Checklist for Engineering Teams

- [ ] Transition from Leaf Certificate Pinning to **SPKI (Public Key Hash) Pinning**.
- [ ] Enforce **Active + Backup Pins** across all mobile configurations.
- [ ] Automate certificate renewals in DevOps pipelines using **Private Key Reuse**.
- [ ] Maintain an emergency **Key Rotation Playbook**.
- [ ] Run **SPKI Drift Monitoring** in Cron/CI, comparing the live server key against the app pin set.
- [ ] (Optional) Deploy **Signed Dynamic Pinning** for OTA pin updates.

Complying with Bank of Thailand security regulations does not have to result in operational disruption. By implementing SPKI pinning and proper key lifecycle management, engineering teams can maintain high security standards while seamlessly accommodating shorter certificate lifespans.
