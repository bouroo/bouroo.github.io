---
title: "รับ interface แล้วคืนค่า struct"
subtitle: ""
date: 2025-11-08T12:00:00+07:00
lastmod: 2025-11-08T12:00:00+07:00
draft: false
author: "กวิน วิริยะประสพสุข"
authorLink: "https://kawin.dev"
description: "ชวนคุยแนวคิดออกแบบฟังก์ชันใน Go แบบรับ interface เป็น input แล้วคืนค่า struct เป็น output ช่วยให้เทสง่าย ลดการผูกติดกันของโค้ด และรองรับการเปลี่ยนแปลงได้ดีขึ้น"
aliases:
- /posts/go_interface_struct/
license: ""
images: []

tags: ["Go", "Interfaces", "Structs", "Design Patterns"]
categories: ["Go"]

featuredImage: "featured-image.png"
featuredImagePreview: "featured-image.png"

lightgallery: true
---

ในโลกของการเขียนโปรแกรมด้วย Go การออกแบบฟังก์ชันให้ยืดหยุ่นและดูแลง่ายคือกุญแจสำคัญของการสร้างแอปที่แข็งแรง หนึ่งในแพทเทิร์นที่ทรงพลังมากคือ

> รับค่าเป็น **interface** แล้วคืนค่าเป็น **struct**

แนวคิดนี้ช่วยให้เทสง่ายขึ้น ลดการผูกติด (coupling) ระหว่างส่วนต่าง ๆ ในระบบ และทำให้โค้ดของเราปรับตัวตาม requirement ที่เปลี่ยนไปได้ดีขึ้นมาก

<!--more-->

## ทำไมควรใช้ Interface เป็น Input?

เวลาเราทำให้ฟังก์ชัน “รับ interface” เป็นพารามิเตอร์ หมายความว่าฟังก์ชันนั้นสามารถทำงานกับ type อะไรก็ได้ที่ “implements interface นั้น” อยู่

{{< admonition note "หลักการออกแบบที่เกี่ยวข้อง" >}}
แนวคิดนี้สอดคล้องกับ **Dependency Inversion Principle** และ **Open-Closed Principle** ช่วยให้ระบบยืดหยุ่นและขยายได้ง่ายขึ้น
{{< /admonition >}}

ลองนึกภาพเคสที่เราต้องประมวลผลเอกสารหลายประเภท ปกติอาจเผลอไปเขียนฟังก์ชันแยกเป็น `ProcessPDF()`, `ProcessWord()` ฯลฯ แต่จริง ๆ แล้วเราสามารถนิยาม interface กลางขึ้นมาชุดเดียวแล้วใช้ร่วมกันได้

{{< admonition example >}}
```go
package main

import "fmt"

// DocumentProcessor กำหนดสัญญา (contract) สำหรับตัวที่เอาไว้ประมวลผลเอกสาร
type DocumentProcessor interface {
	Process() string
	GetContent() string
}

// PDFDocument implements DocumentProcessor สำหรับไฟล์ PDF
type PDFDocument struct {
	Content string
}

func (p PDFDocument) Process() string {
	return fmt.Sprintf("Processing PDF document with content: %s", p.Content)
}

func (p PDFDocument) GetContent() string {
	return p.Content
}

// WordDocument implements DocumentProcessor สำหรับไฟล์ Word
type WordDocument struct {
	Text string
}

func (w WordDocument) Process() string {
	return fmt.Sprintf("Processing Word document with text: %s", w.Text)
}

func (w WordDocument) GetContent() string {
	return w.Text
}

// HandleDocument รับค่าอะไรก็ได้ที่ implements DocumentProcessor
func HandleDocument(processor DocumentProcessor) {
	fmt.Println(processor.Process())
}

func main() {
	pdf := PDFDocument{Content: "Go documentation"}
	word := WordDocument{Text: "Software design patterns"}

	HandleDocument(pdf)
	HandleDocument(word)
}
```
{{< /admonition >}}

ในตัวอย่างนี้ `HandleDocument` ไม่สนใจเลยว่ากำลังทำงานกับ `PDFDocument` หรือ `WordDocument` สิ่งเดียวที่มันสนใจคือ object ที่ส่งเข้ามา “ต้อง” implements `DocumentProcessor` ซึ่งการันตีว่าอย่างน้อยมีเมธอด `Process()` ให้เรียกแน่นอน

{{< admonition tip "ข้อดีของการใช้ Interface" >}}
ทำให้ `HandleDocument` นำกลับมาใช้ซ้ำได้สูง และเทสง่ายมาก เพราะเราสามารถสร้าง mock ที่ implements `DocumentProcessor` ขึ้นมาทดแทนของจริงได้เลย
{{< /admonition >}}

## ทำไมควรใช้ Struct เป็น Output?

การคืนค่าเป็น struct ทำให้เราสื่อสาร “หลาย ๆ ข้อมูล” ออกจากฟังก์ชันได้อย่างมีความหมาย ชัดเจน และเป็นระบบมากกว่าแค่คืนค่าหลาย ๆ ตัวแบบกระจัดกระจาย เพราะ struct จะบอก semantic ของข้อมูลแต่ละฟิลด์ได้ชัดเจนกว่า และทำให้ signature ของฟังก์ชันอ่านง่าย ดูแลต่อได้ไม่ปวดหัว

ลองขยายตัวอย่างการประมวลผลเอกสารให้คืนค่าแบบมีโครงสร้างมากขึ้น

{{< admonition example >}}
```go
package main

import "fmt"

// DocumentProcessor กำหนดสัญญา (contract) สำหรับตัวที่เอาไว้ประมวลผลเอกสาร
type DocumentProcessor interface {
	Process() (ProcessResult, error)
	GetContent() string
}

// ProcessResult แทนผลลัพธ์แบบมีโครงสร้างของการประมวลผลเอกสาร
type ProcessResult struct {
	DocumentType string
	Status       string
	Message      string
	ProcessedBy  string
}

// PDFDocument implements DocumentProcessor สำหรับไฟล์ PDF
type PDFDocument struct {
	Content string
}

func (p PDFDocument) Process() (ProcessResult, error) {
	// สมมติว่ามี logic การประมวลผลบางอย่าง
	result := ProcessResult{
		DocumentType: "PDF",
		Status:       "Success",
		Message:      "PDF processed successfully",
		ProcessedBy:  "PDFProcessorV1",
	}
	return result, nil
}

func (p PDFDocument) GetContent() string {
	return p.Content
}

// WordDocument implements DocumentProcessor สำหรับไฟล์ Word
type WordDocument struct {
	Text string
}

func (w WordDocument) Process() (ProcessResult, error) {
	// สมมติว่ามี logic การประมวลผลบางอย่าง
	result := ProcessResult{
		DocumentType: "Word",
		Status:       "Success",
		Message:      "Word document processed successfully",
		ProcessedBy:  "WordProcessorV2",
	}
	return result, nil
}

func (w WordDocument) GetContent() string {
	return w.Text
}

// HandleDocument ประมวลผลเอกสารแล้วคืนผลลัพธ์แบบมีโครงสร้าง
func HandleDocument(processor DocumentProcessor) (ProcessResult, error) {
	return processor.Process()
}

func main() {
	pdf := PDFDocument{Content: "Go documentation"}
	word := WordDocument{Text: "Software design patterns"}

	pdfResult, err := HandleDocument(pdf)
	if err != nil {
		fmt.Printf("Error processing PDF: %v\n", err)
	} else {
		fmt.Printf("PDF Processing Result: %+v\n", pdfResult)
	}

	wordResult, err := HandleDocument(word)
	if err != nil {
		fmt.Printf("Error processing Word: %v\n", err)
	} else {
		fmt.Printf("Word Processing Result: %+v\n", wordResult)
	}
}
```
{{< /admonition >}}

พอเราใช้ `ProcessResult` เป็น struct สำหรับผลลัพธ์ เราก็ห่อข้อมูลทุกอย่างที่เกี่ยวกับการประมวลผลใส่เข้าไปในที่เดียว

{{< admonition tip "ข้อดีของการคืนค่าแบบ Struct" >}}
ค่าที่คืนออกมาจะ “อธิบายตัวเอง” ได้ดี (self-describing) และขยายเพิ่มฟิลด์ใหม่ในอนาคตได้ง่าย โดยไม่ต้องไปแก้ function signature ของ `HandleDocument` (ยังคงเป็น `(ProcessResult, error)` เหมือนเดิม)
{{< /admonition >}}

## สรุป

{{< admonition summary "Key Takeaway" >}}
การใช้ interface เป็น input และ struct เป็น output เป็นแพทเทิร์นที่เวิร์กมากใน Go  
มันช่วยให้โค้ดแยกส่วนกันดี (modular), เทสง่าย, และดูแลรักษาในระยะยาวได้สบายขึ้น พอเราโฟกัสที่ “สัญญา” (interface) ว่าฟังก์ชันต้องการอะไร และใช้ “ข้อมูลแบบมีโครงสร้าง” (struct) บอกสิ่งที่ฟังก์ชันส่งกลับ โค้ดที่ได้จะทั้งยืดหยุดและอ่านง่ายในเวลาเดียวกัน
{{< /admonition >}}
