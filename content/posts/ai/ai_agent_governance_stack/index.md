---
title: "AI Agent Governance Stack: ออกแบบระบบควบคุม AI Coding Agent ให้โค้ดมาเร็วโดยไม่เสียคุณภาพ"
subtitle: ""
date: 2026-09-01T09:00:00+07:00
lastmod: 2026-09-01T09:00:00+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "แนวทางการออกแบบ Governance Stack สำหรับ AI Coding Agent ให้ทีมได้ความเร็วโดยไม่เสียเจตนา การตรวจสอบ และการควบคุมที่พิสูจน์ได้ ผ่าน SPDD, Fable gates และ minimum harness"
license: ""
images: []
featuredImage: "featured-image.svg"
featuredImagePreview: "featured-image.svg"
tags: ["AI", "AI-agents", "LLM", "prompt-engineering", "SPDD"]
categories: ["AI"]
lightgallery: true
---

ช่วงไม่กี่ปีที่ผ่านมา ทีมพัฒนาซอฟต์แวร์ที่นำ AI coding agent มาใช้ในงานจริง ต้องเผชิญกับ **ความย้อนแย้ง** 2 ด้านที่วิ่งสวนทางกันอย่างชัดเจน:

1. **ความสามารถในการสร้างโค้ด (generation) เพิ่มขึ้นอย่างก้าวกระโดด:** งานที่เคยใช้เวลาครึ่งวันถูกสร้างเสร็จภายในสิบนาที ปริมาณ pull request ต่อทีมเพิ่มขึ้นหลายเท่า และคอขวดของการเขียนโค้ดก็หายไป
2. **ขีดความสามารถในการตรวจสอบ (alignment, review, audit) แทบไม่ขยับ:** เจตนาที่ตกลงกับ agent ไว้ในหน้าต่างแชทหายไปพร้อมบทสนทนา และ agent สามารถอ้างอิง endpoint ที่ถูกยกเลิกไปแล้วด้วยความมั่นใจได้

ผลลัพธ์คือโค้ดไหลเข้ามาเร็วขึ้น แต่ความสามารถของทีมในการทำให้การเปลี่ยนแปลงนั้น **ควบคุมได้ ตรวจทานได้ และนำกลับมาใช้ใหม่ได้** ไม่ได้เร็วตาม บทความนี้จะสรุปแนวทางการออกแบบ **governance stack** สามชั้นที่วางซ้อนกัน โดยอ้างอิงจากสามแหล่งที่มาจากคนละมุม:

- [Structured-Prompt-Driven Development (SPDD)](https://martinfowler.com/articles/structured-prompt-driven/) จากทีม Thoughtworks บน martinfowler.com ตอบที่ชั้น **เจตนา**
- [Flowcharts ของ Fable method](https://github.com/Sahir619/fable-method) จาก GitHub ตอบที่ชั้น **กระบวนการ**
- [Stop Overengineering Your Agent Harness](https://www.oreilly.com/radar/stop-overengineering-your-agent-harness/) จาก O'Reilly Radar ตอบที่ชั้น **runtime**

## 1. กับดักของความเร็วที่ไร้การควบคุม (The Anti-Pattern)

SPDD เปิดด้วยการเปรียบเทียบที่ตรงประเด็น: การซื้อ AI assistant ก็เหมือนซื้อเฟอร์รารีมาขับบนถนนลูกรัง เครื่องยนต์ทรงพลังก็จริง แต่เวลาที่ถึงจุดหมายขึ้นอยู่กับสภาพถนน ไม่ใช่แรงม้า

คอขวดของการพัฒนาซอฟต์แวร์จึงไม่ใช่การเขียนโค้ดอีกต่อไป แต่คือการทำให้การเปลี่ยนแปลงจาก AI *ควบคุมได้ ตรวจทานได้ และนำกลับมาใช้ใหม่ได้* กับดักที่พบบ่อยมีสามรูปแบบ: **เจตนาตายในแชท (intent death)**, **โมเดลมั่นใจในสิ่งที่จำผิด (confident recall)** และ **การรีวิวกลายเป็นงานขุดดิน (review as archaeology)** การแก้ปัญหาทั้งสามต้องอาศัยสามชั้นที่ทำงานร่วมกัน ชั้นเจตนา (SPDD) ชั้นกระบวนการ (Fable) และชั้น runtime (harness)

## 2. ชั้นเจตนา: SPDD และ REASONS Canvas (The Intent Layer)

ไอเดียหลักของ SPDD ง่ายและตรงไปตรงมา: prompt ไม่ใช่ข้อความใช้แล้วทิ้ง แต่เป็น **engineering artifact** ที่มี version control เหมือนโค้ด โครงสร้างมาตรฐานคือ **REASONS Canvas** ซึ่งเป็น prompt template เจ็ดส่วน:

- **R**equirements พร้อม Definition of Done
- **E**ntities (โดเมนโมเดล)
- **A**pproach (กลยุทธ์ และ trade-off ที่ยอมรับ)
- **S**tructure (การเปลี่ยนแปลงนี้ไปอยู่ตรงไหนของระบบ)
- **O**perations (ขั้นตอนรูปธรรม ละเอียดถึงระดับ method signature)
- **N**orms (มาตรฐานโค้ดของทีม)
- **S**afeguards (เงื่อนไขที่ห้ามต่อรอง เรื่องความปลอดภัย ขอบเขต)

กฎทองของ SPDD คือ: **เมื่อโค้ดคลาดเคลื่อนจากเจตนา ให้แก้ prompt ก่อน แล้วค่อยแก้โค้ด** ไม่ใช่แก้โค้ดเฉยๆ แล้วปล่อยให้ spec เน่าทิ้งไว้ และเมื่อ refactor โค้ด (พฤติกรรมเดิม) ต้อง sync เจตนากลับเข้า canvas ด้วย เป็น **two-way sync ไม่ใช่ one-way pipeline** ผลข้างเคียงที่ดีคือรูปแบบการรีวิวเปลี่ยนไป จาก "หาบั๊กใน diff ใหญ่ๆ" กลายเป็น "ตรวจเจตนาใน canvas" ซึ่งเบากว่ามาก

SPDD ไม่ได้แนะนำให้ใช้กับทุกงาน ตาราง fitness ของเขาระบุชัดเจนว่า:

| ประเภทงาน | ความเหมาะสม |
|---|---|
| งาน standardized ซ้ำๆ และงาน compliance หนักๆ | สูง (ห้าดาว) |
| hotfix, spike, งานศิลป์อย่าง frontend styling | ต่ำ (หนึ่งดาว) |
| "context black hole" ที่ยังนิยามโจทย์ไม่ชัด | ต่ำ (หนึ่งดาว) |

## 3. ชั้นกระบวนการ: Fable gates ที่ agent เถียงไม่ได้ (The Process Layer)

Fable method คือการเขียน "วิธีทำงานของ agent" ลงเป็น flowchart ที่เป็น pseudocode ซึ่งปฏิบัติได้จริง ทุกกล่อง trace กลับไปหากฎ และทุกเพชรคือการตัดสินใจที่โมเดลต้องตัดสินจริงๆ กลไกที่ฉลาดที่สุดคือการบังคับให้ agent **เขียนบางอย่างออกมาก่อน** ผ่านสิ่งที่เรียกว่า gate สี่ตัว:

1. **Intent gate** ก่อนแตะพฤติกรรมใดๆ agent ต้องเขียน `INTENT: โค้ดทำ X, เช็คคาดหวัง Y, spec ระบุ Z` ถ้าสามอย่างนี้ไม่ตรงกัน ห้ามแก้ ต้องยกขึ้นมาให้คนตัดสิน ลำดับความสำคัญคือ คำพูดผู้ใช้ > spec > เช็ค > โค้ดปัจจุบัน
2. **Authorization gate** การกระทำที่ย้อนกลับไม่ได้ (push, deploy, ส่งเมล, จ่ายเงิน) ต้องอ้างคำพูดของ user เองแบบคำต่อคำ (`AUTH: user said "..."`) ถ้าไม่มีก็เขียน `PENDING:` แล้วหยุด หลักการคือ **README ไม่ใช่การอนุญาต และความรู้สึกว่างานยังไม่ครบก็ไม่ใช่การอนุญาต**
3. **Recall gate** ข้อเท็จจริงที่ agent "จำ" มา (API signature, endpoint, ราคา) ต้องเปิดจากแหล่งจริงใหม่ทุกครั้ง ไม่งั้นต้องติดป้าย "จากความจำ ยังไม่ตรวจสอบ" ไว้ในรายงาน นี่คือทางแก้โดยตรงของปัญหา endpoint ที่ถูกยกเลิกแล้ว
4. **Verification gate** ต้องรันเช็คเอง ถ้าล้มเหลวซ้ำๆ ครบสามรอบ ให้หยุด แล้วส่งกลับมาพร้อมสิ่งที่ลองไป, output จริง และสมมติฐานปัจจุบัน

การตัดสินว่างาน "เสร็จ" จริงมีหลักสามข้อ: diff กับ ground truth มีน้ำหนักเหนือรายงาน, ทุกการตรวจสอบที่อ้างว่าทำต้องรันซ้ำได้ (รันซ้ำไม่ได้ = ไม่นับ) และคำตัดสินมีแค่สามแบบ **VERIFIED**, **VERIFIED WITH CAVEATS** และ **REFUTED** หลักฐานว่า flowchart นี้ไม่ใช่งานสายวิชาการ: เจ้าของเขียนไว้ว่าทุกกล่องถูกตรวจกับ transcript จริงของ agent ที่รันงานจริง และมีสามกล่องถูกแก้ตาม observation สไตล์เดียวกับการ debug โค้ด

## 4. ชั้น runtime: harness ที่เล็กที่สุดเท่าที่ไปได้ (The Runtime Layer)

บทความจาก O'Reilly ปิดท้ายด้วยสิ่งที่ทีมมักทำเกิน: การสร้างกลไกห่อโมเดลมากเกินความจำเป็น กรอบที่เขาให้มีสองแกน:

- **Action complexity** ต้องประสานเครื่องมือและการตัดสินใจมากแค่ไหน
- **Context complexity** ต้องเก็บและจำข้อมูลมากแค่ไหน

agent งาน support ที่จบใน 1-5 turn อยู่ต่ำทั้งสองแกน ลงทุนแค่ routing, เครื่องมือแบบจำกัด, guardrail และ handoff ให้คนก็พอ ไม่ต้องมี memory หรือ compaction ส่วน coding และ deep-research agent ที่ context ใหญ่ จึงค่อยคุยเรื่อง reduce / offload / isolate เพื่อให้เห็นขนาดที่แท้จริง: coding agent ที่ใช้งานได้จริงเขียนด้วย Python ได้ในประมาณ **131 บรรทัด**

แนวคิดที่สำคัญที่สุดในบทความนี้คือ **Kirby effect** (ชื่อตั้งตาม Kirby ที่ดูดความสามารถของศัตรูเข้าตัวเอง): ทุกองค์ประกอบของ harness คือการสมมติว่า "โมเดลทำสิ่งนี้เองไม่ได้" พอโมเดลเก่งขึ้น สมมติฐานนั้นหมดอายุ สิ่งที่สร้างไว้ก็กลายเป็นน้ำหนักตาย หลักฐานในโลกจริงหนักแน่น: chain-of-thought prompting กลายเป็น reasoning model ไปแล้ว, โหมด plan กำลังถูกถอดออกเพราะโมเดลเชื่อฟังคำสั่ง "วางแผน แต่อย่าแก้โค้ด" ได้เอง, Manus ถูก re-architecture ห้าครั้งในปีเดียว และแม้แต่ Anthropic ก็ถอดกลไกของ Claude Code ทุกครั้งที่โมเดลรุ่นใหม่ออก ดังนั้นทุกอย่างที่สร้างในชั้นนี้ ควรตั้งงบไว้สำหรับวันที่จะถอดมันออกด้วย

## 5. ประกอบเป็น governance stack (The Stack)

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

เมื่อมองเป็น stack จะเห็นว่าแต่ละชั้นช่วยกันเอง:

- **Canvas ทำให้ intent gate เขียนง่าย** เพราะ spec มีอยู่แล้ว agent แค่เปิดอ่าน ไม่ต้องเดา
- **Gate ทำให้ canvas น่าเชื่อถือ** เพราะ prompt และโค้ด sync กันจริง ไม่ใช่แค่คำสัญญา
- **Harness ทำให้ทั้งคู่ audit ได้** เพราะมี trace และ eval ให้ judge pass รันซ้ำ ถ้าไม่มีส่วนนี้ "VERIFIED" ก็เป็นแค่ละคร
- **Kirby effect คุมทั้งสามชั้น** เพราะ gate กับ canvas ก็ควรถูกทบทวนทุกครั้งที่โมเดลรุ่นใหม่ออกเช่นกัน ไม่ใช่แค่ harness

## 6. ตัวอย่างจริง: งานเดียวผ่านทั้งสามชั้น (Worked Example)

โจทย์: "เพิ่ม retry แบบ exponential backoff ให้ตัวส่ง webhook" ชั้นที่ 1 เจตนา (REASONS Canvas) เจ็ดบรรทัดก่อนเขียนโค้ดแม้แต่บรรทัดเดียว:

| REASONS | งานนี้ |
|---|---|
| **R**equirements | webhook ที่ส่งไม่สำเร็จ retry สูงสุด 5 ครั้ง backoff 1s→16s เสร็จเมื่อ: บังคับ fail แล้ว retry จนเข้า dead-letter ได้ |
| **E**ntities | `WebhookDelivery` เพิ่ม `attempt_count`, `next_retry_at` |
| **A**pproach | เกิด retry ที่ delivery worker ไม่ใช่ผู้เรียก ยอมรับ trade-off: queue ช้าลง แต่ API ไม่ช้า |
| **S**tructure | แก้แค่ `internal/webhook/delivery.go` |
| **O**perations | `func (w *Worker) deliverWithRetry(d *Delivery) error` |
| **N**orms | เทสแบบ table-driven ใช้ logger เดิม |
| **S**afeguards | ห้าม retry 4xx ยกเว้น 429 และ delay รวมห้ามเกิน 1 นาที |

ชั้นที่ 2 กระบวนการ (Fable): agent เปิดงานด้วย `INTENT: โค้ดเพิ่ม retry, TestDeliveryRetry คาดหวัง 5 attempts, canvas หัวข้อ R ว่าตรงกัน` สามอย่างตรงกัน จึงแก้ได้ ต่อมา agent ต้องการ push branch แต่ไม่มีการอนุญาตจากผู้ใช้ จึงเขียน `PENDING: push awaiting approval` แล้วหยุด

ชั้นที่ 3 runtime (harness): การรันทิ้งร่องรอยที่ replay ได้ ได้แก่ เวอร์ชัน prompt, tool call และ output ของเทส ไม่ต้องมีอะไรแพงกว่านั้น

Judge pass: รัน `go test ./internal/webhook/ -run TestDeliveryRetry` ผ่าน 3 เคส แต่เคส 429 จำลองเท่านั้น คำตัดสินจึงเป็น **VERIFIED WITH CAVEATS** (ยังไม่เคยเทสกับ 429 จริง) ทั้ง stack จบแค่นี้ การรีวิวเลิกเป็นงานขุดดิน อ่านจอเดียวก็รู้ว่าเกิดอะไรขึ้น และอะไรยังพิสูจน์ไม่ได้

## 7. แผนนำไปใช้และข้อควรระวัง (Adoption Roadmap)

การนำ stack นี้ไปใช้ควรไล่จากล่างขึ้นบนทีละชั้น:

| ระยะเวลา | แนวทางปฏิบัติ |
|---|---|
| สัปดาห์ที่ 1: runtime | วาง agent ที่ใช้อยู่ลงบนสองแกน (action / context complexity) แล้วลบกลไกที่อ้างเหตุผลไม่ได้ออกอย่างน้อยหนึ่งอย่าง |
| สัปดาห์ที่ 2: gate | เริ่มจาก intent gate (บังคับ INTENT ก่อนแก้) และ authorization gate (AUTH หรือ PENDING) บนงานที่ย้อนกลับไม่ได้มากที่สุด |
| สัปดาห์ที่ 3 เป็นต้นไป: canvas | เลือกฟีเจอร์หนึ่งที่ scope ชัด เขียน REASONS Canvas เต็มชุด และฝึกกฎทองจนเป็นนิสัย: แก้ prompt ก่อน แก้โค้ดทีหลัง |
| ต่อเนื่อง: expiry review | ทุกครั้งที่โมเดลรุ่นใหม่ออก ให้ถามกับทุกกลไกว่า "อันนี้ตั้งอยู่บนสมมติฐานว่าโมเดลอ่อนเรื่องอะไร และจุดอ่อนนั้นยังอยู่ไหม" |

กับดักที่ต้องระวัง: **governance theater** (ติดตรา VERIFIED โดยไม่เคยรันเช็คซ้ำ), **one-way sync** (โค้ดเดินหน้า แต่ spec เน่าอยู่ข้างหลัง) และ **permanent scaffolding** (เชื่อว่านั่งร้านที่สร้างไว้คือสถาปัตยกรรมถาวร ทั้งที่มันคือ workaround ที่มีวันหมดอายุ)

## สรุป Checklists สำหรับ Engineering Team

- [ ] ย้ายเจตนาออกจากหน้าต่างแชท เขียนเป็น REASONS Canvas ที่มี version control
- [ ] ใช้กฎทอง two-way sync: แก้ prompt ก่อน แล้วจึงแก้โค้ด
- [ ] บังคับ gate ก่อนแก้: `INTENT:` สำหรับการเปลี่ยนแปลง และ `AUTH:` หรือ `PENDING:` สำหรับการกระทำที่ย้อนกลับไม่ได้
- [ ] เปิดข้อเท็จจริงจากแหล่งจริงเสมอ หรือติดป้าย "จากความจำ ยังไม่ตรวจสอบ"
- [ ] ตัดสินงานด้วย diff และการรันซ้ำได้ ไม่ใช่รายงาน และใช้คำตัดสินเพียง VERIFIED / VERIFIED WITH CAVEATS / REFUTED
- [ ] ตั้งงบสำหรับการถอดกลไกออก (expiry review) ทุกครั้งที่โมเดลรุ่นใหม่ออก

เป้าหมายของทั้ง stack สรุปได้เป็นประโยคเดียว: **ย้ายความไม่แน่นอนไปทางซ้ายให้มากที่สุด** ให้การตัดสินใจส่วนใหญ่เกิดขึ้นตอนที่ยังถูก อยู่ใน artifact ที่คนรีวิวได้ ไม่ใช่ตอนที่โค้ดไปถึง production แล้ว SPDD บอกว่า *เจตนา* ต้องเป็นไฟล์ที่มี version, Fable บอกว่า *กระบวนการ* ต้องมี gate ที่เถียงไม่ได้ และ O'Reilly บอกว่า *กลไก* ที่ห่อโมเดลไว้ ทุกชิ้นมีวันหมดอายุ ในยุค AI การพัฒนาซอฟต์แวร์ไม่ใช่การประกวด IQ ของโมเดล แต่เป็นการประกวดแบนด์วิดท์การคิดของวิศวกร วิศวกรคิดชัดแค่ไหน หล่อปัญหาได้ดีแค่ไหน ตัดสินใจเป็นแค่ไหน

## ลิงก์ที่เกี่ยวข้อง

- [Structured-Prompt-Driven Development (SPDD) martinfowler.com](https://martinfowler.com/articles/structured-prompt-driven/)
- [Fable method flowcharts Sahir619/fable-method (GitHub)](https://github.com/Sahir619/fable-method)
- [Stop Overengineering Your Agent Harness O'Reilly Radar](https://www.oreilly.com/radar/stop-overengineering-your-agent-harness/)
- [openspdd CLI สำหรับรัน workflow ของ SPDD](https://github.com/gszhangwei/open-spdd)
