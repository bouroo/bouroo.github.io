---
title: "How to Let AI Agents Write Code Without Losing Sleep"
subtitle: ""
date: 2026-09-01T09:00:00+07:00
lastmod: 2026-09-01T09:00:00+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "After months with AI coding agents, the same problems kept showing up: code arrives fast, review can't keep up, and intent dies in chat history. Then three articles from three different angles clicked into one picture SPDD, Fable gates, and the minimum harness."
license: ""
images: []
featuredImage: "featured-image.jpeg"
featuredImagePreview: "featured-image.jpeg"
tags: ["AI", "AI-agents", "LLM", "prompt-engineering", "SPDD"]
categories: ["AI"]
lightgallery: true
---

<!--more-->

Hello!

I've been leaning on AI coding agents a lot lately, both at work and in side projects, and I suspect many of you are feeling the same thing I am. The first weeks are pure speed code that used to take half a day lands in ten minutes. But after a while, something starts to feel off.

Reviews turn into huge PRs you can barely read in time. The intent you discussed with the agent lives in a chat window and dies there. And sometimes the agent is *confidently wrong* I've had one hand me a deprecated endpoint with a completely straight face.

Last week I read three articles that approach this from three different angles, and when I laid them side by side, they snapped into a single picture. That's what this post is about.

The three articles:

- [Structured-Prompt-Driven Development (SPDD)](https://martinfowler.com/articles/structured-prompt-driven/) from Thoughtworks, on martinfowler.com
- The [Fable method flowcharts](https://github.com/Sahir619/fable-method) on GitHub
- [Stop Overengineering Your Agent Harness](https://www.oreilly.com/radar/stop-overengineering-your-agent-harness/) from O'Reilly Radar

## The line that hooked me: generation is cheap, alignment is expensive

SPDD opens with a metaphor that nails it: buying an AI assistant is like buying a Ferrari and driving it on muddy roads. The engine is powerful, but your arrival time is set by the road, not the horsepower.

That's exactly what we're all running into. The bottleneck isn't writing code anymore it's making AI-generated change *governable, reviewable, and reusable*.

But the three articles answer that question at different layers:

- SPDD answers at the **intent** layer how do we specify what to build so it's precise and inspectable?
- Fable answers at the **process** layer how should the agent work: gather evidence, authorize actions, verify claims, and stop?
- O'Reilly answers at the **runtime** layer what machinery should wrap the model, and (more importantly) what should *not*?

I've started thinking of them together as a governance stack. Let's walk it layer by layer.

## Layer 1: Stop leaving prompts in chat (SPDD)

SPDD's core move is simple: a prompt is not a disposable message. It's an engineering artifact with version control, just like code.

Their standard structure is the REASONS Canvas a seven-part prompt template:

- **R**equirements, with a Definition of Done
- **E**ntities (the domain model)
- **A**pproach (strategy, and the trade-offs you accepted)
- **S**tructure (where the change fits in the system)
- **O**perations (concrete steps, down to method signatures)
- **N**orms (team coding standards)
- **S**afeguards (non-negotiable invariants, security, limits)

The part I like most is their golden rule: **when the code diverges from intent, fix the prompt first, then the code.** Don't patch the code and let the spec rot. And when you refactor (same behavior), sync it back into the canvas. It's a two-way sync, not a one-way pipeline.

A nice side effect: review changes shape. Instead of "hunt the bug in a giant diff," it becomes "check the intent in the canvas" a much lighter job.

They're not selling a dream, though. Their fitness table is blunt: SPDD earns five stars for standardized, repeatable, compliance-heavy work, and one star for hotfixes, spikes, aesthetic work like frontend styling, or "context black holes" where nobody can define the problem precisely enough to constrain the model. Don't bother.

## Layer 2: Gates the agent can't argue with (Fable)

The Fable method encodes "how an agent should work" as flowcharts that are executable pseudocode every box traces back to a rule, every diamond is a decision the model must actually make.

The clever part is forcing the agent to **write something down before it proceeds**, through what they call gates. Four of them are immediately stealable:

1. **Intent gate** before touching any behavior, the agent writes `INTENT: code does X, check expects Y, spec says Z`. If the three disagree, no editing. Escalate to a human. The authority order is: user statement > spec > checks > current code.

2. **Authorization gate** irreversible actions (push, deploy, send email, pay) require a verbatim quote of the user's own words (`AUTH: user said "..."`). No quote? Write `PENDING:` and stop. I love this line: **a README is not authorization, and "the task feels incomplete" is not authorization.**

3. **Recall gate** anything the agent "remembers" (an API signature, an endpoint, a price) must be re-opened from a live source, or explicitly labeled "from memory, unverified" in the report. This is the direct cure for the confident deprecated-endpoint incident.

4. **Verification gate** run the check yourself. If it fails three times in a row, stop, and hand back what you tried, the actual output, and your current hypothesis.

And when judging whether work is really "done": the diff against ground truth outranks the report, every claimed verification must be re-runnable (can't re-run it = doesn't count), and the verdict is exactly one of three: **VERIFIED**, **VERIFIED WITH CAVEATS**, or **REFUTED**.

I'll admit I first assumed flowcharts like these were academic cosplay. But the author writes that every box was checked against real transcripts of agents running real problems, and three boxes got corrected by observation. Same spirit as debugging code, really.

## Layer 3: The smallest harness that ships today (O'Reilly)

The O'Reilly piece closes the loop with the thing we tend to overdo building too much machinery around the model.

Their framing is two axes:

- **Action complexity** how many tools and decisions must the agent coordinate?
- **Context complexity** how much must it gather and retain?

A support agent that finishes in 1–5 turns sits low on both routing, bounded tools, guardrails, and human handoffs are enough. No memory, no compaction. Coding and deep-research agents with big contexts are where reduce / offload / isolate conversations start.

For perspective: a working coding agent can be ~131 lines of Python.

The most important idea in the article is the **Kirby effect** (named after Kirby, who absorbs enemies' powers): every harness component is a bet that "the model can't do this by itself." As models improve, the bet expires, and the component becomes dead weight.

The receipts are heavy: chain-of-thought prompting became reasoning models. Plan modes are being removed because models now obey "plan, don't edit" on their own. Manus was re-architected five times in one year. Even Anthropic strips Claude Code's harness every time a new model generation ships.

So whatever we build at this layer, budget for the day we rip it out.

## Stack the layers and the picture appears

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

Once you see it as a stack, each layer pays for the others:

- The canvas makes the intent gate cheap the spec already exists, the agent just opens it instead of guessing.
- The gates make the canvas trustworthy prompt↔code sync is real, not a promise.
- The harness makes both auditable traces and evals are what the judge pass re-runs. Without this layer, "VERIFIED" is theater.
- And the Kirby effect disciplines all three gates and canvas sections should be re-reviewed on every model generation too, not just harness features.

## What I'm going to try

Reading these, I located myself at the bottom of the ladder (Level 0 is "vibes" ad hoc prompts in chat; Level 3 is fully governed intent assets). So here's my simple plan:

1. **Week 1: runtime** map the agents I use on the two axes, then delete one piece of machinery I can't justify.
2. **Week 2: gates** start with the intent gate (INTENT line before any edit) and the authorization gate (AUTH or PENDING) on my most irreversible workflow.
3. **Week 3 on: canvas** pick one well-bounded feature, write a full REASONS Canvas, and drill the golden rule until it's reflex: prompt first, code second.
4. **Ongoing: expiry review** every new model release, ask of every mechanism: "what model weakness does this assume, and does that weakness still exist?"

And the traps to watch for there are a few familiar masks: governance theater (VERIFIED stamps with nothing re-run), one-way sync (code moves, spec rots), and believing the scaffolding we built is permanent architecture when it's a workaround with an expiry date.

## Wrapping up

If I had to compress all of this into one sentence: **push uncertainty as far left as possible.** Make most of the decisions while they're still cheap, inside artifacts humans can review not after the code reaches production.

SPDD says *intent* should be a versioned file. Fable says *process* needs gates that can't be argued with. O'Reilly says every piece of *machinery* we wrap around the model has an expiry date.

I'll close with my favorite line from SPDD: in the AI era, software development isn't a contest of model IQ. It's a contest of engineer cognitive bandwidth how clearly we think, how well we frame problems, how deliberately we decide.

## Related links

- [Structured-Prompt-Driven Development (SPDD) martinfowler.com](https://martinfowler.com/articles/structured-prompt-driven/)
- [Fable method flowcharts Sahir619/fable-method (GitHub)](https://github.com/Sahir619/fable-method)
- [Stop Overengineering Your Agent Harness O'Reilly Radar](https://www.oreilly.com/radar/stop-overengineering-your-agent-harness/)
- [openspdd the CLI that runs the SPDD workflow](https://github.com/gszhangwei/open-spdd)
