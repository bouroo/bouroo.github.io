---
title: "DDD, TDD, BDD: What Are They?"
subtitle: ""
date: 2024-01-27T13:49:01+07:00
lastmod: 2024-01-27T21:49:01+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: ""
license: ""
images: []

tags: ["dev"]
categories: ["dev"]

featuredImage: "featured-image.webp"
featuredImagePreview: "featured-image.webp"

lightgallery: true
---
Software development methodologies are designed to help developers deliver code smoothly and with fewer defects. Recently, three terms have been frequently discussed: Domain Driven Design (DDD), Test Driven Development (TDD), and Behavior Driven Development (BDD). Although their names are similar, these methodologies have distinct differences. Here's a comparison of DDD, TDD, and BDD from a lazy person's perspective.
<!--more-->

## Domain Driven Design (DDD)
### Principles
DDD focuses on designing business models into domains to reduce the likelihood of software models being designed or implemented in a way that doesn't align with the business domain. A simple example is when, after working for a while, you end up with tangled or unused tables and fields.

- Focuses on the Core Domain and the logic of each domain.
- Developers work with Domain Experts (those who understand the business of the product).
- Creates a development model of the domain.
- Communicates using a "ubiquitous language" that all team members understand.
- Separates complexity through domain boundaries.

### Practices
DDD relies on building domain models as entities, value objects, aggregates, and bounded contexts. Developers must continuously review models and domains to refine their understanding of what each domain is responsible for.

- Start by dividing events for each domain with a domain expert. The trick is that the term used to refer to something will change in each event, e.g., a user can have multiple statuses depending on the domain event, such as visitor -> member -> customer -> debtor -> package recipient.
- Use the simplest language understandable by both business and tech sides.
- Define the boundaries of each domain, e.g., when a user makes a payment in the payment domain, it then enters the delivery domain.
- Explore and review to find subdomains. For example, in the payment domain, there might be subdomains like debit, credit, and reconcile that share payment responsibilities but have different operational responsibilities.

### Use Cases
DDD is most beneficial for complex business domains that require intricate logic. DDD provides a clear structure for separating domain complexity.

#### DDD is suitable for:
- Financial systems
- Insurance application systems
- Healthcare systems
- Other domains with constantly changing business methods

#### Limitations of DDD
- High learning curve regarding patterns and methodological principles. Requires experience.
- Difficult to correctly identify domains, contexts, and layers.
- Modeling requires upfront analysis and iterative review for accuracy and correct understanding.

## Test Driven Development (TDD)
### Principles
TDD integrates into the development loop where requirements are transformed into specific test cases for each function. Code is then written to pass these test cases.

- Write tests before coding any functions.
- Write just enough code to pass the tests. Avoid over-engineering.
- Refactoring code is a normal practice.
- Keep test cases and coding as simple as possible.

### Practices
- Write test cases based on requirements.
- Run tests to see if they work; initially, they should fail (red).
- Write code to pass the tests (green).
- Refactor, comment, optimize as needed (refactor).
- Repeat the cycle.

This might seem like a hellish loop, but if we do it just enough and avoid over-engineering, we'll find that with every addition, we'll have tests to ensure we don't unknowingly break existing functionality.

### Use Cases
TDD integrates well with self-contained functions. The testing process forces developers to review and understand requirements before coding (of course, they're forced to write test cases based on requirements). The side benefits are technical documentation from test cases and higher quality work with fewer defects (assuming the test cases are correct).

#### TDD is suitable for:
- Back-end tasks (APIs, microservices)
- Functions based on business logic
- Anything that is a function that is extended or reused.

#### Limitations of TDD
- Requires time to write tests before coding.
- The quality of work depends on the proficiency in writing tests (can be mitigated by senior developers writing tests first and junior developers writing functions).
- Becomes useless if requirements are not stable, as it will waste time in the loop of writing new tests and code. In this case, you should consider Behavior Driven Development (BDD).

## Behavior Driven Development (BDD)
### Principles
BDD focuses on defining system behavior in a way that is understandable by both business and technical stakeholders.

- Test cases are written in a language understandable by both business and tech.
- Test cases are written based on business outcomes, not technical development details.
- Business and tech teams collaborate on requirements.
- Requirements are transformed into automated regression tests.

### Practices
BDD brings business and technical teams together to define system behavior. They collaborate to write scripts describing the expected system behavior, following this sequence:

- Given [context]
- When [an event occurs]
- Then [expected outcome]

These scripts describe the overall function's behavior rather than just technical details. The tests will be automated regression tests.

### Use Cases
BDD facilitates collaboration between business and technical teams, helping everyone understand requirements upfront before coding begins.

#### BDD is suitable for:
- Web Apps/Mobile Apps or user-facing parts.
- Projects with unstable requirements.
- Projects where not all stakeholders are technically proficient.

#### Limitations of BDD
- Requires more upfront time for collaboration and test writing.
- Can be cumbersome when applied to back-end tasks.

## Conclusion
- DDD focuses on modeling complex business domains into suitable development models.
- TDD focuses on coding and quality, preventing new development from breaking existing functionality.
- BDD focuses on defining system behavior, preventing developers from getting lost.

All three methodologies can be used together, depending on how they are adapted to suit the specific task.
