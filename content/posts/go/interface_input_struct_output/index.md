---
title: "Go Functions: Interface as Input, Struct as Output"
subtitle: ""
date: 2025-11-08T12:00:00+07:00
lastmod: 2025-11-08T12:00:00+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "สำรวจประโยชน์ของการออกแบบฟังก์ชัน Go ด้วยอินเทอร์เฟซสำหรับอินพุตและ Struct สำหรับเอาต์พุต"
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

ในโลกของการเขียนโปรแกรม Go การออกแบบฟังก์ชันที่ยืดหยุ่นและดูแลรักษาง่ายเป็นสิ่งสำคัญมากสำหรับการสร้างแอปพลิเคชันที่แข็งแกร่งและทำงานได้ดี รูปแบบหนึ่งที่เจ๋งมากๆ คือการใช้ Interface เป็นอินพุตของฟังก์ชัน และ Struct เป็นเอาต์พุตของฟังก์ชัน วิธีนี้จะช่วยให้โค้ดของเราทดสอบได้ง่ายขึ้น ลดการยึดติดกันของส่วนต่างๆ และปรับเปลี่ยนได้ง่ายขึ้นในอนาคต

<!--more-->

## ทำไมต้องส่ง Interface เข้าไปเป็น Input?

พอฟังก์ชันรับ Interface เป็นพารามิเตอร์อินพุต มันหมายความว่าฟังก์ชันนั้นสามารถทำงานกับข้อมูลประเภทไหนก็ได้ที่ implements อินเทอร์เฟซนั้นๆ เหมือนกับสัญญาที่บอกว่าถ้าจะเข้ามาได้จะต้องมีอะไรให้เรียกใช้บ้าง ส่วนเรียกแล้วจะทำงานอย่างไรต่อก็ขึ้นอยู่กับต้นทาง ทำให้การปรับเพิ่มลดตาม Requirement เป็นส่วน ๆ ได้ง่าย
{{< admonition note "หลักการออกแบบ" >}}
ซึ่งสอดคล้องกับหลักการ **Dependency Inversion Principle** และ **Open-Closed Principle** ทำให้โค้ดของเรายืดหยุ่นและขยายได้ง่ายขึ้นเยอะเลย
{{< /admonition >}}

ลองนึกภาพสถานการณ์ที่เราต้องประมวลผลเอกสารหลายประเภท แทนที่จะเขียนฟังก์ชันแยกสำหรับเอกสารแต่ละแบบ เราก็สามารถกำหนดอินเทอร์เฟซขึ้นมาอันเดียวได้เลย

{{< admonition example >}}
```go
package main

import "fmt"

// DocumentProcessor defines the contract for processing documents.
type DocumentProcessor interface {
	Process() string
	GetContent() string
}

// PDFDocument implements DocumentProcessor for PDF files.
type PDFDocument struct {
	Content string
}

func (p PDFDocument) Process() string {
	return fmt.Sprintf("กำลังประมวลผลเอกสาร PDF ที่มีเนื้อหา: %s", p.Content)
}

func (p PDFDocument) GetContent() string {
	return p.Content
}

// WordDocument implements DocumentProcessor for Word files.
type WordDocument struct {
	Text string
}

func (w WordDocument) Process() string {
	return fmt.Sprintf("กำลังประมวลผลเอกสาร Word ที่มีข้อความ: %s", w.Text)
}

func (w WordDocument) GetContent() string {
	return w.Text
}

// HandleDocument accepts any type that implements the DocumentProcessor interface.
func HandleDocument(processor DocumentProcessor) {
	fmt.Println(processor.Process())
}

func main() {
	pdf := PDFDocument{Content: "เอกสาร Go"}
	word := WordDocument{Text: "รูปแบบการออกแบบซอฟต์แวร์"}

	HandleDocument(pdf)
	HandleDocument(word)
}
```
{{< /admonition >}}

ในตัวอย่างนี้ `HandleDocument` ไม่ได้สนใจเลยว่ามันกำลังจัดการกับ `PDFDocument` หรือ `WordDocument` แต่มันสนใจแค่ว่า input ที่เข้ามานั้น implements `DocumentProcessor` interface ซึ่งรับประกันว่ามีเมธอด `Process()` อยู่แล้ว
{{< admonition tip "ประโยชน์ของ Interface" >}}
ทำให้ `HandleDocument` สามารถนำไปใช้ซ้ำได้ง่ายมากๆ และยังทดสอบได้ง่ายด้วย mock implementations ของ `DocumentProcessor` ด้วยอีกต่างหาก
{{< /admonition >}}

## แล้วทำไมต้องส่ง Struct ออกไปเป็น Output?

การคืนค่า Struct เป็นเอาต์พุตเนี่ย เป็นวิธีที่ชัดเจนและเป็นระเบียบมากๆ ในการส่งข้อมูลหลายๆ ส่วนออกจากฟังก์ชัน มันดีกว่าการส่งค่าเปล่าๆ หลายๆ ค่ากลับไป เพราะ Struct จะช่วยให้ข้อมูลที่รวมกลุ่มกันมีความหมายมากขึ้น และทำให้การอ่าน Signature ของฟังก์ชันเข้าใจง่ายและดูแลรักษาง่ายขึ้นด้วย

ลองมาขยายตัวอย่างการประมวลผลเอกสารของเรา เพื่อให้มันคืนค่าผลลัพธ์ที่มีโครงสร้างกันดู

{{< admonition example >}}
```go
package main

import "fmt"

// DocumentProcessor defines the contract for processing documents.
type DocumentProcessor interface {
	Process() (ProcessResult, error)
	GetContent() string
}

// ProcessResult represents the structured output of a document processing operation.
type ProcessResult struct {
	DocumentType string
	Status       string
	Message      string
	ProcessedBy  string
}

// PDFDocument implements DocumentProcessor for PDF files.
type PDFDocument struct {
	Content string
}

func (p PDFDocument) Process() (ProcessResult, error) {
	// Simulate some processing logic
	result := ProcessResult{
		DocumentType: "PDF",
		Status:       "สำเร็จ",
		Message:      "ประมวลผล PDF สำเร็จ",
		ProcessedBy:  "PDFProcessorV1",
	}
	return result, nil
}

func (p PDFDocument) GetContent() string {
	return p.Content
}

// WordDocument implements DocumentProcessor for Word files.
type WordDocument struct {
	Text string
}

func (w WordDocument) Process() (ProcessResult, error) {
	// Simulate some processing logic
	result := ProcessResult{
		DocumentType: "Word",
		Status:       "สำเร็จ",
		Message:      "ประมวลผลเอกสาร Word สำเร็จ",
		ProcessedBy:  "WordProcessorV2",
	}
	return result, nil
}

func (w WordDocument) GetContent() string {
	return w.Text
}

// HandleDocument processes a document and returns a structured result.
func HandleDocument(processor DocumentProcessor) (ProcessResult, error) {
	return processor.Process()
}

func main() {
	pdf := PDFDocument{Content: "เอกสาร Go"}
	word := WordDocument{Text: "รูปแบบการออกแบบซอฟต์แวร์"}

	pdfResult, err := HandleDocument(pdf)
	if err != nil {
		fmt.Printf("ข้อผิดพลาดในการประมวลผล PDF: %v\n", err)
	} else {
		fmt.Printf("ผลการประมวลผล PDF: %+v\n", pdfResult)
	}

	wordResult, err := HandleDocument(word)
	if err != nil {
		fmt.Printf("ข้อผิดพลาดในการประมวลผล Word: %v\n", err)
	} else {
		fmt.Printf("ผลการประมวลผล Word: %+v\n", wordResult)
	}
}
```
{{< /admonition >}}

ด้วยการคืนค่า `ProcessResult` เราจะห่อหุ้มข้อมูลที่เกี่ยวข้องทั้งหมดเกี่ยวกับการประมวลผลเอาไว้ในก้อนเดียว
{{< admonition tip "ประโยชน์ของ Structured Output" >}}
สิ่งนี้ทำให้ค่าที่ส่งคืนสามารถบอกได้ด้วยตัวเองว่าคืออะไร และยังง่ายต่อการขยายเพิ่มข้อมูลในอนาคตโดยไม่ต้องไปเปลี่ยน Signature ของฟังก์ชัน (ประเภทการคืนค่าของ `HandleDocument` ก็ยังคงเป็น `(ProcessResult, error)` เหมือนเดิม) คนที่เอาไปใช้งานต่อก็ไม่ต้องแก้ไขตามต้นทางถ้าตราบใดที่ ฟิลใน Struct ไม่หายไป
{{< /admonition >}}

## สรุป

{{< admonition summary "สรุป" >}}
การรวมอินเทอร์เฟซสำหรับอินพุตฟังก์ชันและ Struct สำหรับเอาต์พุตฟังก์ชันเป็นรูปแบบการออกแบบที่เจ๋งมากๆ ใน Go เลยนะ มันช่วยส่งเสริมให้โค้ดของเราถอดประกอบง่าย ทดสอบได้ง่าย และดูแลรักษาง่าย ทำให้แอปพลิเคชันของเราพัฒนาไปได้เรื่อยๆ อย่างราบรื่นเมื่อความต้องการเปลี่ยนไป โดยการเน้นที่สัญญา (Interface) สำหรับสิ่งที่ฟังก์ชันต้องการ และข้อมูลที่มีโครงสร้าง (Struct) สำหรับสิ่งที่ฟังก์ชันคืนค่าให้ เราจะสามารถเขียนโค้ด Go ที่ชนรุ่นหลังไม่สบถถึงบรรพบุรุษเราได้ ถถถ
{{< /admonition >}}
