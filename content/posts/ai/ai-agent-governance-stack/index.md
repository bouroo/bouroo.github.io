---
title: "ทำยังไงให้ AI Agent เขียนโค้ดแทนเราได้โดยไม่ต้องกลัว"
subtitle: ""
date: 2026-09-01T09:00:00+07:00
lastmod: 2026-09-01T09:00:00+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "หลังจากใช้ AI coding agent มาสักพัก ผมเจอปัญหาเดิมๆ คือโค้ดมาเร็วแต่รีวิวไม่ทัน พอไปเจอสามบทความนี้เข้า ทุกอย่างก็เชื่อมกันเป็นระบบเดียว นั่นคือ SPDD, Fable gates และ minimum harness"
license: ""
images: []
featuredImage: "featured-image.jpeg"
featuredImagePreview: "featured-image.jpeg"
tags: ["AI", "AI-agents", "LLM", "prompt-engineering", "SPDD"]
categories: ["AI"]
lightgallery: true
---

<!--more-->

สวัสดีครับ!

ช่วงนี้ผมใช้ AI coding agent เยอะขึ้นเรื่อยๆ ทั้งในงานและโปรเจกต์ส่วนตัว และเชื่อว่าหลายคนคงเจออาการเดียวกัน ช่วงแรกมันฟีเจอร์เรื่องความเร็วจริง โค้ดตัวที่เคยใช้เวลาครึ่งวัน เดี๋ยวนี้ได้ภายในสิบนาที แต่พอใช้ไปสักพัก เราเริ่มรู้สึกว่าบางอย่างมันไม่ได้เร็วขึ้นจริงๆ

รีวิวโค้ดที่แล่นเข้ามาเป็น PR ใหญ่ๆ ที่แทบอ่านไม่ทัน เจตนาบางอย่างที่เราคุยกับ agent ไว้ในแชทแล้วหายไปพร้อมหน้าต่างบทสนทนา และบางครั้ง agent ก็ "มั่นใจ" กับสิ่งที่มันจำผิด ผมเคยเจอ agent ชี้ endpoint ที่เลิกใช้แล้วให้ โดยที่หน้าตาเหมือนแม่นมาก

เมื่อสัปดาห์ก่อนผมได้ไปอ่านสามบทความที่มาจากคนละมุม แต่พออ่านจบแล้ววางเรียงกัน มันกลับเชื่อมเป็นรูปเดียวกันอย่างน่าประหลาด เลยอยากเล่าให้ฟังครับ

สามบทความคือ:

- [Structured-Prompt-Driven Development (SPDD)](https://martinfowler.com/articles/structured-prompt-driven/) จากทีม Thoughtworks บน martinfowler.com
- [Flowcharts ของ Fable method](https://github.com/Sahir619/fable-method) จาก GitHub
- [Stop Overengineering Your Agent Harness](https://www.oreilly.com/radar/stop-overengineering-your-agent-harness/) จาก O'Reilly Radar

## ประโยคที่ปักอกที่สุด: generation ถูกแล้ว แต่ alignment แพง

SPDD เปิดด้วยประโยคที่ผมว่าตรงมาก เขาเปรียบว่าการซื้อ AI assistant ก็เหมือนซื้อเฟอร์รารีมาขับบนถนนลูกรัง เครื่องยนต์ทรงพลังก็จริง แต่เวลาที่ถึงจุดหมายขึ้นอยู่กับสภาพถนน ไม่ใช่แรงม้า

นั่นแหละคือสิ่งที่เราเจอกัน คอขวดไม่ใช่การเขียนโค้ดอีกต่อไป แต่คือการทำให้การเปลี่ยนแปลงจาก AI มัน *ควบคุมได้ ตรวจทานได้ และเอากลับมาใช้ใหม่ได้*

แต่ทั้งสามบทความตอบคำถามนี้คนละชั้นกัน:

- SPDD ตอบที่ชั้น **เจตนา** เราระบุสิ่งที่ต้องสร้างยังไงให้ชัดและตรวจได้
- Fable ตอบที่ชั้น **กระบวนการ** agent ควรทำงานยังไง ตรวจสอบยังไง หยุดตรงไหน
- O'Reilly ตอบที่ชั้น **runtime** เราต้องห่อโมเดลด้วยกลไกอะไรบ้าง และ (ที่สำคัญกว่า) อะไรที่ *ไม่ต้อง* ห่อ

ผมเรียกมันรวมๆ ว่า governance stack ครับ มาไล่ทีละชั้นกัน

## ชั้นที่ 1: เลิกทิ้ง prompt ในแชท (SPDD)

ไอเดียหลักของ SPDD ง่ายๆ คือ prompt ไม่ใช่ของใช้แล้วทิ้ง แต่ควรเป็น engineering artifact ที่มี version control เหมือนโค้ด

เขาใช้โครงสร้างเดียวเรียกว่า REASONS Canvas ซึ่งเป็น prompt template เจ็ดส่วน:

- **R**equirements พร้อม Definition of Done
- **E**ntities (โดเมนโมเดล)
- **A**pproach (กลยุทธ์ และ trade-off ที่ยอมรับ)
- **S**tructure (การเปลี่ยนแปลงนี้ไปอยู่ตรงไหนของระบบ)
- **O**perations (ขั้นตอนรูปธรรม ละเอียดถึงระดับ method signature)
- **N**orms (มาตรฐานโค้ดของทีม)
- **S**afeguards (เงื่อนไขที่ห้ามต่อรอง เรื่องความปลอดภัย ขอบเขต)

ส่วนที่ผมชอบที่สุดคือกฎทองของเขา: **เมื่อโค้ดคลาดเคลื่อนจากเจตนา ให้แก้ prompt ก่อน แล้วค่อยแก้โค้ด** ไม่ใช่แก้โค้ดเฉยๆ แล้วปล่อยให้ spec เน่าทิ้งไว้ และถ้าเรา refactor โค้ด (พฤติกรรมเดิม) ต้อง sync กลับเข้า canvas ด้วย มันเป็น two-way sync ไม่ใช่ one-way pipeline

ผลข้างเคียงที่ดีคือรีวิวเปลี่ยนไป จาก "หาบั๊กใน diff ใหญ่ๆ" กลายเป็น "ตรวจเจตนาใน canvas" ซึ่งเบากว่ากันเยอะ

แต่เขาก็ไม่ได้ขายฝันนะครับ ตาราง fitness ของเขาชัดเจนว่า SPDD เหมาะกับงาน standardized ซ้ำๆ และงาน compliance หนักๆ (ห้าดาว) แต่กับ hotfix, spike, งานศิลป์อย่าง frontend styling หรือ "context black hole" ที่ใครๆ ก็ยังนิยามโจทย์ไม่ชัด อย่าเสียเวลาเลย หนึ่งดาว

## ชั้นที่ 2: gate ที่ agent เถียงไม่ได้ (Fable)

Fable method คือการเขียน "วิธีทำงานของ agent" ลงเป็น flowchart ที่เป็น pseudocode ที่ปฏิบัติได้จริง ทุกกล่อง trace กลับไปหากฎ ทุกเพชรคือการตัดสินใจที่โมเดลต้องตัดสินจริงๆ

ส่วนที่ผมว่าฉลาดที่สุดคือการบังคับให้ agent **เขียนอะไรบางอย่างออกมาก่อน** ผ่านสิ่งที่เขาเรียกว่า gate สี่ตัวที่ผมว่ายืมมาใช้ได้เลย:

1. **Intent gate** ก่อนแตะพฤติกรรมใดๆ agent ต้องเขียน `INTENT: โค้ดทำ X, เช็คคาดหวัง Y, spec ระบุ Z` ถ้าสามอย่างนี้ไม่ตรงกัน ห้ามแก้ ต้องยกขึ้นมาให้คนตัดสิน ลำดับความสำคัญคือ คำพูดผู้ใช้ > spec > เช็ค > โค้ดปัจจุบัน

2. **Authorization gate** การกระทำที่ย้อนกลับไม่ได้ (push, deploy, ส่งเมล, จ่ายเงิน) ต้องอ้างคำพูดของ user เองแบบคำต่อคำ (`AUTH: user said "..."`) ถ้าไม่มีก็เขียน `PENDING:` แล้วหยุด ผมชอบประโยคนี้มาก **README ไม่ใช่การอนุญาต ความรู้สึกว่างานยังไม่ครบก็ไม่ใช่การอนุญาต**

3. **Recall gate** ข้อเท็จจริงที่ agent "จำ" มา (API signature, endpoint, ราคา) ต้องเปิดจากแหล่งจริงใหม่ ไม่งั้นต้องติดป้าย "จากความจำ ยังไม่ตรวจสอบ" ไว้ในรายงาน นี่แหละคือทางแก้เรื่อง endpoint ที่ agent ชี้มั่นใจๆ ให้ผม

4. **Verification gate** ต้องรันเช็คเอง ถ้าล้มเหลวซ้ำๆ ครบสามรอบ หยุด แล้วส่งกลับมาพร้อมสิ่งที่ลอง output จริง และสมมติฐานปัจจุบัน

และตอนตัดสินว่างาน "เสร็จ" จริงๆ: diff กับ ground truth มีน้ำหนักเหนือรายงาน ทุกการตรวจสอบที่อ้างว่าทำต้องรันซ้ำได้ (รันซ้ำไม่ได้ = ไม่นับ) และคำตัดสินมีแค่สามแบบ **VERIFIED**, **VERIFIED WITH CAVEATS**, **REFUTED**

จริงๆ ตอนแรกผมคิดว่า flowchart แบบนี้คงเป็นงานสายวิชาการ แต่เจ้าของเขาเขียนไว้ว่าทุกกล่องถูกตรวจกับ transcript จริงของ agent ที่รันงานจริง แล้วแก้สามจุดตามที่ observation บอก สไตล์เดียวกับที่เรา debug โค้ดเลยครับ

## ชั้นที่ 3: harness เล็กที่สุดเท่าที่จะไปวันนี้ (O'Reilly)

บทความจาก O'Reilly ตบท้ายด้วยเรื่องที่เรามักทำเกิน สร้างกลไกรอบโมเดลมากเกินความจำเป็น

เขาให้กรอบง่ายๆ สองแกน:

- **Action complexity** ต้องประสานเครื่องมือและการตัดสินใจมากแค่ไหน
- **Context complexity** ต้องเก็บและจำข้อมูลมากแค่ไหน

agent งาน support ที่คุย 1–5 turn จบ อยู่ต่ำทั้งสองแกน ลงทุนที่ routing, เครื่องมือแบบจำกัด, guardrail และ handoff ให้คนก็พอ ไม่ต้องมี memory หรือ compaction ส่วน coding กับ deep-research agent ที่ context โต ถึงค่อยคุยเรื่อง reduce / offload / isolate

และเขายกตัวอย่างว่า coding agent ที่ใช้งานได้จริงเขียนด้วย Python ได้ใน ~131 บรรทัด

จุดที่ผมว่าสำคัญที่สุดในบทความนี้คือ **Kirby effect** (ชื่อตั้งตาม Kirby ที่ดูดความสามารถของศัตรูเข้าตัวเอง): ทุกองค์ประกอบของ harness คือการสมมติว่า "โมเดลทำสิ่งนี้เองไม่ได้" พอโมเดลเก่งขึ้น สมมติฐานหมดอายุ สิ่งที่เราเฝ้าสร้างมาก็กลายเป็นน้ำหนักตาย

เคสจริงที่เขายกมาก็หนักแน่น chain-of-thought prompting กลายเป็น reasoning model ไปแล้ว, โหมด plan กำลังถูกถอดออกเพราะโมเดลเชื่อฟัง "วางแผน แต่อย่าแก้โค้ด" ได้เองแล้ว, Manus ถูก re-architecture ห้าครั้งในปีเดียว แม้แต่ Anthropic ก็ถอดกลไกของ Claude Code ทุกครั้งที่โมเดลรุ่นใหม่ออก

ดังนั้นทุกอย่างที่เราสร้างในชั้นนี้ ควรตั้งงบไว้สำหรับวันถอดมันออกด้วย

## พอวางซ้อนกัน มันเห็นภาพเลย

{{< mermaid >}}
flowchart TD
    REQ["ความต้องการ"] --> CANVAS["REASONS Canvas<br/>ชั้นเจตนา (SPDD)"]
    CANVAS --> GATES["Decision gates<br/>ชั้นกระบวนการ (Fable)"]
    GATES --> LOOP["Agent loop<br/>ชั้น runtime (harness)"]
    LOOP --> DIFF["การเปลี่ยนแปลง + หลักฐาน"]
    DIFF --> JUDGE["Judge pass<br/>VERIFIED / CAVEATS / REFUTED"]
    JUDGE -->|แก้เจตนา| CANVAS
    JUDGE -->|ส่งมอบ| OUT["Release อย่างสบายใจ"]
    LOOP -.->|Kirby effect| LOOP
{{< /mermaid >}}

พอมองเป็น stack จะเห็นว่าแต่ละชั้นช่วยกันเอง:

- Canvas ทำให้ intent gate เขียนง่าย เพราะ spec มีอยู่แล้ว agent แค่เปิดอ่าน ไม่ต้องเดา
- Gate ทำให้ canvas น่าเชื่อ เพราะ prompt↔โค้ด sync กันจริง ไม่ใช่แค่คำสัญญา
- Harness ทำให้ทั้งคู่ audit ได้ เพราะมี trace และ eval ให้ judge pass รันซ้ำ ถ้าไม่มีส่วนนี้ "VERIFIED" ก็แค่ละคร
- และ Kirby effect คุมทั้งสามชั้น: gate กับ canvas ก็ควรโดนทบทวนทุกครั้งที่โมเดลรุ่นใหม่ออกเหมือนกัน ไม่ใช่แค่ harness

## ตัวอย่างสั้นๆ: โจทย์เล็กๆ หนึ่งงานผ่านทั้งสามชั้น

โจทย์: "เพิ่ม retry แบบ exponential backoff ให้ตัวส่ง webhook" ฟังดูต้องจุดโต้งใช่ไหม? จริงๆ จบในจอเดียว

**ชั้นที่ 1 — REASONS Canvas (SPDD)** เจ็ดบรรทัดสั้นๆ ก่อนเขียนโค้ดแม้แต่บรรทัดเดียว:

| REASONS | งานนี้ |
|---|---|
| **R**equirements | webhook ที่ส่งไม่สำเร็จ retry สูงสุด 5 ครั้ง backoff 1s→16s เสร็จเมื่อ: บังคับ fail แล้ว retry จนเข้า dead-letter ได้ |
| **E**ntities | `WebhookDelivery` เพิ่ม `attempt_count`, `next_retry_at` |
| **A**pproach | เกิด retry ที่ delivery worker ไม่ใช่ผู้เรียก ยอมรับ trade-off: queue ช้าลง แต่ API ไม่ช้า |
| **S**tructure | แก้แค่ `internal/webhook/delivery.go` |
| **O**perations | `func (w *Worker) deliverWithRetry(d *Delivery) error` |
| **N**orms | เทสแบบ table-driven ใช้ logger เดิม |
| **S**afeguards | ห้าม retry 4xx ยกเว้น 429 และ delay รวมห้ามเกิน 1 นาที |

**ชั้นที่ 2 — gate (Fable)**: agent เปิดงานด้วย `INTENT: โค้ดเพิ่ม retry, TestDeliveryRetry คาดหวัง 5 attempts, canvas หัวข้อ R ว่าตรงกัน` — สามอย่างตรงกัน แก้ได้ ต่อมามันอยาก push branch: คุณไม่เคยพูดไว้ มันจึงเขียน `PENDING: push awaiting approval` แล้วหยุด

**ชั้นที่ 3 — harness (O'Reilly)**: การรันทิ้งร่องรอยที่ replay ได้ เท่านั้นแหละ — เวอร์ชัน prompt, tool call, output ของเทส ไม่ต้องมีอะไรแพงกว่านั้น

**Judge pass**: `go test ./internal/webhook/ -run TestDeliveryRetry` → ผ่าน 3 เคส แต่เคส 429 จำลองเท่านั้น → คำตัดสิน: **VERIFIED WITH CAVEATS** (ยังไม่เคยเทสกับ 429 จริง)

ทั้ง stack จบแค่นี้เอง รีวิวเลิกเป็นงานขุดดิน: อ่านจอเดียวก็รู้ทันทีว่าเกิดอะไรขึ้น และอะไรยังพิสูจน์ไม่ได้

## สิ่งที่ผมกำลังจะลองทำ

อ่านจบแล้วผมเจอตัวเองอยู่ Level ต้นๆ ของบันไดนี้แหละครับ (Level 0 คือ "vibes" prompt มั่วในแชท, Level 3 คือมี governed intent เต็มระบบ) เลยวางแผนง่ายๆ ไว้แบบนี้:

1. **สัปดาห์แรก: runtime** ไล่ดูว่า agent ที่ใช้อยู่น่าจะอยู่ตรงไหนของสองแกน แล้วลองลบกลไกที่อ้างเหตุผลไม่ได้ออกสักอย่าง
2. **สัปดาห์ที่สอง: gate** เริ่มจาก intent gate (บังคับให้ agent เขียน INTENT ก่อนแก้) กับ authorization gate (AUTH หรือ PENDING) บนงานที่ย้อนกลับไม่ได้ที่สุด
3. **สัปดาห์ที่สามไป: canvas** หาฟีเจอร์นึงที่ scope ชัด ลองเขียน REASONS Canvas เต็มๆ ดู แล้วฝึกกฎทองให้เป็นธรรมชาติ: แก้ prompt ก่อน แก้โค้ดทีหลัง
4. **ต่อเนื่อง: expiry review** ทุกครั้งที่โมเดลรุ่นใหม่ออก ถามตัวเองกับทุกกลไกว่า "อันนี้ตั้งอยู่บนสมมติฐานว่าโมเดลอ่อนเรื่องอะไร และจุดอ่อนนั้นยังอยู่ไหม"

ส่วนของที่ควรระวัง เจอหน้ากากหลายแบบเหมือนกัน: governance theater (ตรา VERIFIED โดยไม่เคยรันซ้ำ), one-way sync (โค้ดเดินหน้า spec เน่า), และการหลงเชื่อว่านั่งร้านที่เราสร้างไว้คือสถาปัตยกรรมถาวร ทั้งที่มันคือ workaround ที่มีวันหมดอายุ

## สรุป

ถ้าให้ย่อทั้งหมดเหลือประโยคเดียว ผมจะย่อว่า: **ย้ายความไม่แน่นอนไปทางซ้ายให้มากที่สุด** ให้การตัดสินใจส่วนใหญ่เกิดขึ้นตอนที่ยังถูกๆ อยู่ใน artifact ที่คนรีวิวได้ ไม่ใช่ตอนที่โค้ดไปถึง production แล้ว

SPDD บอกว่า *เจตนา* ต้องเป็นไฟล์ที่มี version, Fable บอกว่า *กระบวนการ* ต้องมี gate ที่เถียงไม่ได้ และ O'Reilly บอกว่า *กลไก* ที่เราห่อโมเดลไว้ ทุกชิ้นมีวันหมดอายุ

ปิดท้ายด้วยประโยคที่ผมชอบที่สุดจาก SPDD ครับ: ในยุค AI การพัฒนาซอฟต์แวร์ไม่ใช่การประกวด IQ ของโมเดล แต่เป็นการประกวดแบนด์วิดท์การคิดของวิศวกร เราคิดชัดแค่ไหน หล่อปัญหาได้ดีแค่ไหน ตัดสินใจเป็นแค่ไหน

## ลิงก์ที่เกี่ยวข้อง

- [Structured-Prompt-Driven Development (SPDD) martinfowler.com](https://martinfowler.com/articles/structured-prompt-driven/)
- [Fable method flowcharts Sahir619/fable-method (GitHub)](https://github.com/Sahir619/fable-method)
- [Stop Overengineering Your Agent Harness O'Reilly Radar](https://www.oreilly.com/radar/stop-overengineering-your-agent-harness/)
- [openspdd CLI สำหรับรัน workflow ของ SPDD](https://github.com/gszhangwei/open-spdd)
