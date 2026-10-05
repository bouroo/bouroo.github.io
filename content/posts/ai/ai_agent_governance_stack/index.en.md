---
title: "AI Agent Governance Stack: Designing Controls for AI Coding Agents So Code Ships Fast Without Losing Reviewability"
subtitle: ""
date: 2026-09-01T09:00:00+07:00
lastmod: 2026-09-01T09:00:00+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "How to design a governance stack for AI coding agents that preserves intent, reviewability, and verifiable control while delivering speed, through SPDD, Fable gates, and the minimum harness."
license: ""
images: []
featuredImage: "featured-image.svg"
featuredImagePreview: "featured-image.svg"
tags: ["AI", "AI-agents", "LLM", "prompt-engineering", "SPDD"]
categories: ["AI"]
lightgallery: true
---

Over the past few years, software teams that put AI coding agents into real workflows have faced a **major operational paradox**, two forces pulling in opposite directions:

1. **Code generation capacity has grown dramatically:** work that once took half a day now lands in ten minutes, the volume of pull requests per team has multiplied, and the code-writing bottleneck has disappeared.
2. **Alignment, review, and audit capacity has barely moved:** intent agreed with the agent dies with the chat window, and an agent can confidently cite an endpoint that was retired months ago.

The result is that code arrives faster, but the team's ability to make that change **governable, reviewable, and reusable** does not speed up with it. This article outlines a three-layer **governance stack** that keeps AI speed without losing reviewability, drawn from three sources that approach the problem from different angles:

- [Structured-Prompt-Driven Development (SPDD)](https://martinfowler.com/articles/structured-prompt-driven/) from Thoughtworks on martinfowler.com, which answers at the **intent** layer
- The [Fable method flowcharts](https://github.com/Sahir619/fable-method) on GitHub, which answers at the **process** layer
- [Stop Overengineering Your Agent Harness](https://www.oreilly.com/radar/stop-overengineering-your-agent-harness/) from O'Reilly Radar, which answers at the **runtime** layer

## 1. The Anti-Pattern: Speed Without Control

SPDD opens with a metaphor that lands: buying an AI assistant is like buying a Ferrari and driving it on muddy roads. The engine is powerful, but arrival time is set by the road, not the horsepower.

The bottleneck in software development is no longer writing code. It is making AI-generated change *governable, reviewable, and reusable*. The common anti-patterns take three forms: **intent death** (agreements live in a chat window, not a version-controlled artifact), **confident recall** (the agent cites an API signature, endpoint, or price from memory without checking a live source), and **review as archaeology** (a large diff arrives with no surrounding intent, so reviewers work slowly and incompletely). Solving all three requires three layers working together: the intent layer (SPDD), the process layer (Fable), and the runtime layer (harness).

## 2. The Intent Layer: SPDD and the REASONS Canvas

SPDD's core move is simple: a prompt is not a disposable message. It is an **engineering artifact** with version control, just like code. Its standard structure is the **REASONS Canvas**, a seven-part prompt template:

- **R**equirements, with a Definition of Done
- **E**ntities (the domain model)
- **A**pproach (strategy, and the trade-offs accepted)
- **S**tructure (where the change fits in the system)
- **O**perations (concrete steps, down to method signatures)
- **N**orms (team coding standards)
- **S**afeguards (non-negotiable invariants: security, limits)

The golden rule is: **when the code diverges from intent, fix the prompt first, then the code.** Do not patch the code and let the spec rot. And when refactoring (same behavior), sync the intent back into the canvas. It is a **two-way sync, not a one-way pipeline**. A useful side effect is that review changes shape: instead of "hunt the bug in a giant diff," it becomes "check the intent in the canvas," which is far lighter.

SPDD does not claim to fit every task. Its fitness table is explicit:

| Type of work | Fit |
|---|---|
| Standardized, repeatable, compliance-heavy work | High (five stars) |
| Hotfixes, spikes, aesthetic work such as frontend styling | Low (one star) |
| "Context black holes" where the problem cannot be defined precisely | Low (one star) |

## 3. The Process Layer: Fable Gates the Agent Cannot Argue With

The Fable method encodes "how an agent should work" as flowcharts that are executable pseudocode: every box traces back to a rule, and every diamond is a decision the model must actually make. The cleverest mechanism forces the agent to **write something down before it proceeds**, through what are called gates. Four of them are immediately reusable:

1. **Intent gate** before touching any behavior, the agent writes `INTENT: code does X, check expects Y, spec says Z`. If the three disagree, no editing is allowed; escalate to a human. The authority order is: user statement > spec > checks > current code.
2. **Authorization gate** irreversible actions (push, deploy, send email, pay) require a verbatim quote of the user's own words (`AUTH: user said "..."`). Without a quote, the agent writes `PENDING:` and stops. The principle is: **a README is not authorization, and "the task feels incomplete" is not authorization.**
3. **Recall gate** anything the agent "remembers" (an API signature, an endpoint, a price) must be re-opened from a live source; otherwise it must be labeled "from memory, unverified" in the report. This is the direct cure for the retired-endpoint problem.
4. **Verification gate** run the check yourself. If it fails three times in a row, stop and hand back what was tried, the actual output, and the current hypothesis.

Judging whether work is really "done" follows three rules: the diff against ground truth outranks the report, every claimed verification must be re-runnable (not re-runnable means it does not count), and the verdict is exactly one of three: **VERIFIED**, **VERIFIED WITH CAVEATS**, or **REFUTED**. The evidence that these flowcharts are not academic: the author writes that every box was checked against real transcripts of agents running real problems, and three boxes were corrected by observation, in the same spirit as debugging code.

## 4. The Runtime Layer: The Smallest Harness That Ships Today

The O'Reilly piece closes the loop with what teams usually overbuild: machinery wrapped around the model beyond what the job needs. Its framing has two axes:

- **Action complexity** how many tools and decisions must be coordinated
- **Context complexity** how much must be gathered and retained

A support agent that finishes in 1-5 turns sits low on both axes: routing, bounded tools, guardrails, and human handoffs are enough. No memory, no compaction. Coding and deep-research agents with large contexts are where reduce / offload / isolate conversations begin. For a sense of scale: a working coding agent can be written in roughly **131 lines of Python**.

The most important idea in the article is the **Kirby effect** (named after Kirby, who absorbs enemies' powers): every harness component is a bet that "the model cannot do this by itself." As models improve, the assumption expires, and what was built becomes dead weight. The real-world evidence is heavy: chain-of-thought prompting became reasoning models, plan modes are being removed because models now obey "plan, do not edit" on their own, Manus was re-architected five times in one year, and even Anthropic strips Claude Code's harness every time a new model generation ships. So whatever is built at this layer should be budgeted for the day it is removed.

## 5. Stacking the Layers (The Governance Stack)

{{< mermaid >}}
flowchart TD
    REQ["Requirement"] --> CANVAS["REASONS Canvas<br/>intent layer (SPDD)"]
    CANVAS --> GATES["Decision gates<br/>process layer (Fable)"]
    GATES --> LOOP["Agent loop<br/>runtime layer (harness)"]
    LOOP --> DIFF["Change + evidence"]
    DIFF --> JUDGE["Judge pass<br/>VERIFIED / CAVEATS / REFUTED"]
    JUDGE -->|fix intent| CANVAS
    JUDGE -->|ship| OUT["Release with confidence"]
    LOOP -.->|Kirby effect| LOOP
{{< /mermaid >}}

Once seen as a stack, each layer pays for the others:

- **The canvas makes the intent gate cheap:** the spec already exists, so the agent opens it instead of guessing.
- **The gates make the canvas trustworthy:** prompt and code sync is real, not a promise.
- **The harness makes both auditable:** traces and evals are what the judge pass re-runs. Without this layer, "VERIFIED" is theater.
- **The Kirby effect disciplines all three:** gates and canvas sections should be re-reviewed on every model generation too, not just harness features.

## 6. A Worked Example: One Task Through the Whole Stack

Task: "add retry with exponential backoff to the webhook sender." Layer 1, intent (REASONS Canvas), seven lines before any code:

| REASONS | This task |
|---|---|
| **R**equirements | Failed webhooks retry up to 5 times, backoff 1s→16s. Done when: a forced failure retries, then dead-letters. |
| **E**ntities | `WebhookDelivery` gains `attempt_count`, `next_retry_at`. |
| **A**pproach | Retry in the delivery worker, not the caller. Accepted trade-off: slower queue, not slower API. |
| **S**tructure | `internal/webhook/delivery.go` only. |
| **O**perations | `func (w *Worker) deliverWithRetry(d *Delivery) error`. |
| **N**orms | Table-driven tests; existing logger interface. |
| **S**afeguards | Never retry 4xx except 429; total delay capped at 1 minute. |

Layer 2, process (Fable): the agent opens with `INTENT: code adds retries, TestDeliveryRetry expects 5 attempts, canvas section R says the same`. All three agree, so editing is allowed. Later it wants to push the branch, but no authorization was given, so it writes `PENDING: push awaiting approval` and stops.

Layer 3, runtime (harness): the run leaves a replayable trail: prompt version, tool calls, test output. Nothing more expensive than that.

Judge pass: running `go test ./internal/webhook/ -run TestDeliveryRetry` passes 3 cases, but the 429 path was only simulated, so the verdict is **VERIFIED WITH CAVEATS** (not yet tested against a live 429). That is the whole stack. Review stops being archaeology: one screen shows what happened and what remains unproven.

## 7. Adoption Roadmap and Failure Modes

Adopting the stack should move layer by layer, not all at once:

| Phase | Recommended Practice |
|---|---|
| Phase 1: runtime | Map the agents in use onto the two axes (action / context complexity), then remove at least one piece of machinery that cannot be justified. |
| Phase 2: gates | Start with the intent gate (require INTENT before any edit) and the authorization gate (AUTH or PENDING) on the most irreversible workflow. |
| Phase 3 onward: canvas | Pick one well-bounded feature, write a full REASONS Canvas, and drill the golden rule until it is a reflex: prompt first, code second. |
| Ongoing: expiry review | On every new model release, ask of each mechanism: "what model weakness does this assume, and does that weakness still exist?" |

Traps to watch for: **governance theater** (stamping VERIFIED without re-running anything), **one-way sync** (code moves forward while the spec rots behind it), and **permanent scaffolding** (treating the scaffold as permanent architecture when it is a workaround with an expiry date).

## Summary Checklist for Engineering Teams

- [ ] Move intent out of the chat window into a version-controlled REASONS Canvas.
- [ ] Apply the golden rule of two-way sync: fix the prompt first, then the code.
- [ ] Enforce gates before edits: `INTENT:` for changes, and `AUTH:` or `PENDING:` for irreversible actions.
- [ ] Re-open facts from a live source, or label them "from memory, unverified."
- [ ] Judge work by the diff and re-runnability, not the report, using only VERIFIED / VERIFIED WITH CAVEATS / REFUTED.
- [ ] Budget for removal (expiry review) on every new model generation.

The goal of the whole stack compresses to one sentence: **push uncertainty as far left as possible.** Make most decisions while they are still cheap, inside artifacts humans can review, not after the code reaches production. SPDD says *intent* must be a versioned file, Fable says *process* must have gates that cannot be argued with, and O'Reilly says every piece of *machinery* wrapped around the model has an expiry date. In the AI era, software development is not a contest of model IQ; it is a contest of engineer cognitive bandwidth: how clearly the engineer thinks, how well the problem is framed, and how deliberately the decision is made.

## Related links

- [Structured-Prompt-Driven Development (SPDD) martinfowler.com](https://martinfowler.com/articles/structured-prompt-driven/)
- [Fable method flowcharts Sahir619/fable-method (GitHub)](https://github.com/Sahir619/fable-method)
- [Stop Overengineering Your Agent Harness O'Reilly Radar](https://www.oreilly.com/radar/stop-overengineering-your-agent-harness/)
- [openspdd the CLI that runs the SPDD workflow](https://github.com/gszhangwei/open-spdd)
