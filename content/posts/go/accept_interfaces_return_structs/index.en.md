---
title: "Accept interfaces, return structs"
subtitle: ""
date: 2025-11-08T12:00:00+07:00
lastmod: 2025-11-08T12:00:00+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "How accepting an interface as a function input and returning a struct as output lets new document types plug into one handler and new result fields ship without changing a function signature."
aliases:
- /posts/go_interface_struct/
license: ""
images: []

tags: ["Go", "Interfaces", "Structs", "Design Patterns"]
categories: ["Go"]

featuredImage: "featured-image.svg"
featuredImagePreview: "featured-image.svg"

lightgallery: true
---

A function that accepts an interface works on every type that implements it; a function that returns a struct hands back named fields instead of a tuple. In the example below, `HandleDocument` takes `PDFDocument` and `WordDocument` through one signature, and adding a field to `ProcessResult` does not change that signature.

<!--more-->

## Why Interface as Input?

When a function accepts an interface as an input parameter, it means the function can operate on any type that implements that interface.
{{< admonition note "Design Principles" >}}
This adheres to the **Dependency Inversion Principle** and **Open-Closed Principle**, allowing for greater flexibility and extensibility.
{{< /admonition >}}

Consider a scenario where you need to process different types of documents. Instead of writing separate functions for each document type, you can define an interface.

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
	return fmt.Sprintf("Processing PDF document with content: %s", p.Content)
}

func (p PDFDocument) GetContent() string {
	return p.Content
}

// WordDocument implements DocumentProcessor for Word files.
type WordDocument struct {
	Text string
}

func (w WordDocument) Process() string {
	return fmt.Sprintf("Processing Word document with text: %s", w.Text)
}

func (w WordDocument) GetContent() string {
	return w.Text
}

// HandleDocument accepts any type that implements the DocumentProcessor interface.
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

In this example, `HandleDocument` doesn't care whether it's dealing with a `PDFDocument` or a `WordDocument`. It only cares that the input implements the `DocumentProcessor` interface, which guarantees the `Process()` method is available.
{{< admonition tip "Benefit of Interfaces" >}}
This makes `HandleDocument` highly reusable and easy to test with mock implementations of `DocumentProcessor`.
{{< /admonition >}}

## Why Struct as Output?

Returning a struct as output provides a clear and organized way to convey multiple pieces of information from a function. Unlike returning multiple bare values, a struct gives semantic meaning to the grouped data and makes the function signature easier to understand and maintain.

Let's extend our document processing example to return structured results.

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
		Status:       "Success",
		Message:      "PDF processed successfully",
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
		Status:       "Success",
		Message:      "Word document processed successfully",
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

By returning `ProcessResult`, we encapsulate all relevant information about the processing operation.
{{< admonition tip "Structured Output Benefits" >}}
This makes the return value self-describing and easy to extend if more information needs to be added in the future without changing the function signature (`HandleDocument`'s return type remains `(ProcessResult, error)`).
{{< /admonition >}}

## Conclusion

{{< admonition summary "Key Takeaway" >}}
Accepting an interface means `HandleDocument` runs for any type that implements `DocumentProcessor`, so a new document type needs no edit to the handler. Returning `ProcessResult` means callers read `Status` and `Message` by name, and a new field goes in without touching the function signature.
{{< /admonition >}}