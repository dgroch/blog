---
layout: post
title: "Task Composition: when intelligence looks like recombination"
date: 2026-03-16
description: "Useful agents don't just follow plans. They discover new workflows by combining what they already have."
---

One mark of intelligence is not merely breaking down a known problem. It is seeing a solution shape that wasn't explicitly handed to you.


> The full implementation is available on GitHub: [github.com/dgroch/metacognition](https://github.com/dgroch/metacognition)

In software terms, that often means recombination.

Task Composition in OpenClaw asks a different question from decomposition. Instead of how do I break this goal into steps, it asks what new outcomes become possible if I combine the capabilities I already have.

[→ Jump to skill files](#skill-files)

The `SKILL.md` makes this explicit. Start with a capability survey. [Self-Model](https://dangroch.com/2026/03/16/self-model-agent-capability-inventory/) is a hard dependency. If the capability inventory doesn't exist, build it before composing. Read the environment model too, because a clever workflow that depends on absent conditions is just another hallucination. Then abstract the available components into categories:

> Think about capabilities in these abstract categories (a thinking aid, not a type system):
> - **Producers**: generate content (text generation, data fetching, image creation)
> - **Transformers**: change content form (summarization, translation, format conversion)
> - **Analyzers**: extract information (code review, data analysis, search)
> - **Actors**: affect the external world (sending messages, creating files, API calls)
> - **Routers**: direct flow (conditional logic, user confirmation, channel selection)
>
> Typical composed workflows chain: Producer → Transformer(s) → Actor, or Analyzer → Transformer → Actor.

The example in the skill is telling: a summary generator can feed a TTS tool, giving you a document-to-audio briefing pipeline. That is a trivial composition in one sense, but it captures the deeper idea. Intelligence often looks less like inventing a brand-new power than like seeing a useful chain hidden in plain sight.

There are guardrails here I appreciate. Candidate generation is capped at two to four workflows. Weak ideas should be killed before they reach the user. Each candidate is evaluated on feasibility, value, risk, and effort, and if any step depends on a missing capability or unavailable resource, the workflow dies. That ruthlessness is necessary because agents are very good at producing optionality theatre.

The `SKILL.md` is explicit about the bar:

> Be ruthless. Proposing useless compositions erodes trust faster than proposing nothing.

The shared plan format with [Task Decomposition](https://dangroch.com/2026/03/16/task-decomposition-for-ai-agents/) is another smart move. Composition adds two fields, then reuses the rest of the step structure:

```markdown
## Plan: [Workflow name]

**Composed from**: [list of capabilities combined]
**Opportunity**: [why this combination is valuable — what it produces that individual capabilities don't]
**Goal**: [concrete description of what the workflow produces]
**Done when**: [acceptance criteria]
**Assumptions**: [list anything assumed but not confirmed]
**Depth**: light | standard | deep

### Steps
...
```

The `Composed from` and `Opportunity` fields distinguish composition plans from decomposition plans. Everything else — step structure, dependency annotations, risk flags — is identical. That means the system can move cleanly from invention to execution without switching mental models.

The skill also introduces a `composition-log.md` for two reasons: don't keep re-proposing workflows the user rejected, and notice when a repeatedly useful workflow should become its own skill. That's a subtle but powerful idea. Composition is not just for solving the immediate problem. It is also a mechanism for discovering stable abstractions worth operationalizing.

> After a composed workflow has been executed (now or in prior sessions), evaluate:
> 1. Has this workflow been useful more than once?
> 2. Is it likely to be useful again?
> 3. Is it complex enough that re-composing from scratch each time is wasteful?
>
> If yes to all three: flag it as a candidate for consolidation into a new skill.

Imagine an agent working inside a workspace that has file read and write tools, web fetch, PDF analysis, and TTS. A decomposition-first system waits for an instruction like make me an audio briefing of this report. A composition-capable system, after summarizing the report, can recognize that summary plus TTS plus message delivery form a valuable adjacent workflow. If the context justifies it, it can propose the pipeline. Not as hype. As a feasible extension grounded in available components.

That is not the same as being creative in the vague consumer-AI sense. It is constrained recombination. The novelty is bounded by actual capabilities, actual interfaces, and actual value.

The AGENTS trigger is grounded in exactly those conditions:

> - **Task Composition**: When exploring what's possible, when an opportunity exists to exceed the literal request, or when asking "what can I build with what I have?" → read `skills/task-composition/SKILL.md` and compose from available capabilities.

The metacognitive thesis here is that higher-order usefulness often emerges from structured recombination, not just better direct instruction following. An agent that can only do what was already named is capable. An agent that can identify adjacent, feasible workflows without losing discipline is starting to show something closer to judgment.

---

## Skill Files

### SKILL.md

````markdown
---
name: task-composition
description: "Compose novel workflows by combining available capabilities forward — invoke when exploring what's possible, when an opportunity exists to exceed the literal request, or when the user asks what the agent can do."
metadata: { "openclaw": { "emoji": "🔗" } }
---

# Task Composition

Reason forward from capabilities to possibilities. Where Task Decomposition breaks a goal into steps, composition asks: "What new outcomes can I create by combining what I already have?"

This is where novel capability emerges. Use it sparingly and deliberately — every proposed composition must be feasible, valuable, and grounded in concrete capabilities.

## When to Compose

**Compose when:**
- The user asks "what can you do?" or "what's possible?" — survey and combine
- You've completed a task and recognize an adjacent opportunity ("I just did X — I could also chain it with Y to produce Z")
- The user describes a problem without prescribing a solution — compose an approach from available capabilities
- You're stuck on a task and could try a different combination of tools/skills
- During decomposition, a sub-task could be solved by a novel capability chain

**Don't compose when:**
- The user gave explicit instructions — follow them
- The task is straightforward and well-served by a single skill or tool
- Speed matters more than exploration
- The composed workflow would require capabilities you don't have (flag the gap instead)
- You'd be interrupting focused execution to say "I could also..." — finish first

The value bar for proposing a composition is high. Nobody wants an agent constantly suggesting alternatives when they just want the task done.

## Composition Framework

### Step 1 — Capability Survey

Read your current building blocks:
- **Self-model inventory**: Read `skills/self-model/capabilities.md`. This is a hard dependency — if it doesn't exist, invoke the self-model skill to build it before composing.
- **Environment model**: Read `skills/env-model/environment.md`. If it doesn't exist, note that feasibility checks will be incomplete and proceed with caution.
- **Capability gaps**: Check `skills/self-model/capability-gaps.md` if it exists — avoid composing workflows that rely on known gaps.

Focus on capabilities relevant to the current context, not the entire inventory. Build a mental model of available building blocks, each with its inputs, outputs, and side effects.

Think about capabilities in these abstract categories (a thinking aid, not a type system):
- **Producers**: generate content (text generation, data fetching, image creation)
- **Transformers**: change content form (summarization, translation, format conversion)
- **Analyzers**: extract information (code review, data analysis, search)
- **Actors**: affect the external world (sending messages, creating files, API calls)
- **Routers**: direct flow (conditional logic, user confirmation, channel selection)

Typical composed workflows chain: Producer → Transformer(s) → Actor, or Analyzer → Transformer → Actor.

### Step 2 — Interface Analysis

For each relevant capability, identify:
- What does it take as input?
- What does it produce as output?
- What are its side effects?

Map natural pairings — where one capability's output matches another's input. Example: "The summarize skill produces text. TTS takes text and produces audio. Chain: document → summary → audio briefing."

Look for capabilities that transform, filter, combine, or route data between others. These are the connective tissue of composed workflows.

### Step 3 — Candidate Generation

Given the current context, generate candidate workflows:
- Each candidate chains 2+ capabilities with clear data flow
- Each must produce an outcome meaningfully different from any single capability alone
- Name each descriptively (e.g., "Inbox-to-Calendar Conflict Detector", "Multi-Format Content Pipeline")
- **Limit to 2-4 candidates max.** Quality over quantity. Kill weak ideas before they reach the user.

### Step 4 — Feasibility and Value Evaluation

For each candidate, assess:

| Criterion | Question | Fail condition |
|---|---|---|
| **Feasibility** | Can every step execute right now? Check self-model and env-model. | Any step requires a missing capability or unavailable resource → kill |
| **Value** | Does this produce something the user would actually want? Is it better than simpler alternatives? | Marginal improvement over a single-skill approach → kill |
| **Risk** | Are there irreversible steps? What could go wrong? | Unacceptable risk without clear value → kill |
| **Effort** | How many tool calls / how much time? | Overhead exceeds the value of the composed result → kill |

Be ruthless. Proposing useless compositions erodes trust faster than proposing nothing.

### Step 5 — Present or Execute

- **In response to "what can you do?"**: Present top candidates with brief descriptions of what each produces and why it's useful.
- **As part of active work (low-risk, clearly valuable)**: Execute the best candidate directly.
- **As part of active work (any uncertainty)**: Propose briefly — "I could also do X by combining Y and Z — want me to?"

Keep proposals concrete and brief. "I can cross-reference your inbox with your calendar to flag scheduling conflicts" — not "I theorize that a synthesis of capabilities A through D might yield..."

### Step 6 — Consolidation Signal

After a composed workflow has been executed (now or in prior sessions), evaluate:
1. Has this workflow been useful more than once?
2. Is it likely to be useful again?
3. Is it complex enough that re-composing from scratch each time is wasteful?

If yes to all three: flag it as a candidate for consolidation into a new skill. Log the signal in `{baseDir}/composition-log.md` and reference the skill acquisition intake path (`[SKILL_ACQUISITION_INTAKE_PATH]` — wire this when the skill-acquisition skill is built).

## Composed Plan Format

Composed workflows use the same plan format as Task Decomposition (`skills/task-decomposition/SKILL.md`) with two additional metadata fields. This keeps the format interoperable — Failure Recovery and other downstream skills can parse both plan types identically.

```markdown
## Plan: [Workflow name]

**Composed from**: [list of capabilities combined]
**Opportunity**: [why this combination is valuable — what it produces that individual capabilities don't]
**Goal**: [concrete description of what the workflow produces]
**Done when**: [acceptance criteria]
**Assumptions**: [list anything assumed but not confirmed]
**Depth**: light | standard | deep

### Steps

1. [Step name] — [brief description]
   - Depends on: none
   - Tool/skill: [what you'll use]
   - Risk: none

2. [Step name] — [brief description]
   - Depends on: step 1
   - Tool/skill: [what you'll use]
   - Risk: [brief description if any]

### Flags

- [Capability gaps from feasibility check]
- [Environmental constraints]
- [Items needing user input]
- [Irreversible steps requiring confirmation]
- [Consolidation candidate: yes/no]
```

The `Composed from` and `Opportunity` fields distinguish composition plans from decomposition plans. Everything else — step structure, dependency annotations, risk flags — is identical.

## Composition Log

Maintain `{baseDir}/composition-log.md` to track compositions across sessions. The agent creates this file on first use from the template at `{baseDir}/composition-log.md`.

Log entries serve two purposes:
1. Avoid re-proposing compositions the user has already rejected
2. Surface consolidation candidates for skill acquisition

Keep entries brief. If the log exceeds ~50 entries, prune old rejected proposals (keep accepted and consolidation candidates).

## Coordination Notes

- **Self-Model (skill 1)** is a hard dependency. Don't compose without reading the inventory first. If `skills/self-model/capabilities.md` doesn't exist, build it.
- **Env-Model (skill 2)** constrains feasibility. A composed workflow requiring browser + macOS node is only viable if both are available per `skills/env-model/environment.md`.
- **Task Decomposition (skill 3)** is the complement. Use decomposition to plan a chosen composition. The shared plan format means you can compose a workflow, then decompose its execution — the formats nest naturally.
- **Skill Acquisition** is the downstream consumer of consolidation signals. When it's built, wire `{baseDir}/composition-log.md` consolidation entries to its intake mechanism.
````

### AGENTS-snippet.md

````markdown
- **Task Composition**: When exploring what's possible, when an opportunity exists to exceed the literal request, or when asking "what can I build with what I have?" → read `skills/task-composition/SKILL.md` and compose from available capabilities.
````
