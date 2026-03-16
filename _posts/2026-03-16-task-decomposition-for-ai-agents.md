---
layout: post
title: "Task Decomposition: planning without pretending the plan is the work"
date: 2026-03-16
description: "Good agent planning breaks goals into executable steps without drifting into fake productivity."
---

There are two common failures in agent planning.


> The full implementation is available on GitHub: [github.com/dgroch/metacognition](https://github.com/dgroch/metacognition)

The first is not planning at all. The model jumps from request to action, misses dependencies, and discovers halfway through that step three required information from step one.

The second is more insidious: elaborate planning theatre. The agent produces a crisp multi-section plan that feels intelligent, then fails because the plan was never checked against actual tools, actual constraints, or the cost of being wrong.

Task Decomposition in OpenClaw is trying to sit exactly between those failures.

[→ Jump to skill files](#skill-files)

Its core claim is simple: plans are useful when they make execution more feasible, not when they merely look organized.

The `SKILL.md` starts with a triage heuristic rather than a universal mandate:

> **Decompose when:**
> - The goal is ambiguous or under-specified ("build me a dashboard")
> - Multiple tools or skills are needed in sequence
> - Steps have dependencies on each other's outputs
> - The work involves irreversible actions (sending messages, publishing, deleting data)
> - Your first reaction is uncertainty about where to start
> - The task will take more than ~5 tool calls
>
> **Skip decomposition when:**
> - The task is a single clear action ("what time is it in Tokyo?", "read this file")
> - You already have a well-practiced pattern for this task type
> - The user gave explicit step-by-step instructions (they already decomposed it)
> - Speed matters more than thoroughness (quick chat reply)

That restraint matters. A lot of reasoning systems quietly become slower versions of instinct because they insist on ceremonial planning for trivial tasks.

Then the skill introduces adaptive depth: light, standard, and deep.

| Mode | When | What you do |
|---|---|---|
| **Light** | 2-3 step tasks, clear path | Think through steps mentally. No written plan. Quick sanity check, then act. |
| **Standard** | Multi-step with dependencies or risks | Write a plan using the format below. State it briefly to the user, then execute. |
| **Deep** | Complex, high-stakes, or unfamiliar tasks | Full plan with feasibility checks against self-model and env-model. Present to user for review before starting. |

Light decomposition stays implicit. Standard decomposition writes a concrete plan and executes. Deep decomposition includes explicit feasibility checks against the self-model and env-model and is meant for costly mistakes.

The plan template itself is unusually practical. It asks for a concrete goal, a done condition, explicit assumptions, a chosen depth, numbered steps in verb-plus-object form, dependencies, a tool or skill for each step, a risk note, and a flags section for capability gaps, environmental constraints, missing user input, and irreversible actions:

```markdown
## Plan: [Goal summary]

**Goal**: [concrete restatement]
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
```

This is not documentation prose. It is a compact execution model.

The best part is the feasibility check. For each step, the agent is told to ask two questions: can I do this, and does the environment support it? If `capabilities.md` or `environment.md` don't exist, the skill says to note that feasibility is unchecked. That one instruction cuts against one of the worst habits in agent systems: smuggling unverified assumptions into plans as if they were stable facts.

That is why the title tension matters. Planning is not the work. The actual work of decomposition is not turning one goal into six bullet points. It is turning one goal into six bullet points that can survive contact with a real environment.

Suppose the user asks an agent to create a blog series from a repo, match an existing house style, and package everything for review. A naive planner might write: research, draft, edit, deliver. That's neat and almost useless. A disciplined decomposition would say: read voice anchors, read repo overview, map post structure to source files, draft hub post, draft each skill post from its minimum sources, write a manifest, verify filenames and frontmatter, then stop short of publication. Each step has dependencies. Each maps to a file or tool action. That is the difference between choreography and execution.

The re-planning section is just as important. When a step fails, the plan updates surgically:

```markdown
### Steps (updated after step 3 failure)

1. [Step name] — [description] ✅ completed
2. [Step name] — [description] ✅ completed
3. [Step name] — [FAILED: reason] → pivoted to step 3b
3b. [New step name] — [alternative approach]
   - Depends on: step 2
   - Tool/skill: [new tool]
   - Risk: [updated risk]
4. [Step name] — [description, updated if affected by pivot]
   - Depends on: step 3b (was: step 3)
```

The skill explicitly says not to throw away the whole plan when one step fails. Re-plan from the failed step forward unless the goal itself changed. That is exactly what good operators do and exactly what many agents don't.

The shared plan format also matters beyond this one skill. [Task Composition](https://dangroch.com/2026/03/16/task-composition-capability-recombination/) and [Failure Recovery](https://dangroch.com/2026/03/16/failure-recovery-for-ai-agents/) are designed to interoperate with it. A composed workflow can later be decomposed for execution, and a failed plan can be surgically updated with a step `3b` instead of rewritten as if nothing happened.

The failure mode this corrects is the language-model instinct to mistake narrative coherence for operational coherence. A model is very good at producing a sequence that sounds like a plan. It is much worse at respecting hidden dependencies unless something forces them into the open.

Planning is not intelligence. But intelligence without planning becomes brittle surprisingly quickly. The trick is to make plans answerable to reality.

---

## Skill Files

### SKILL.md

````markdown
---
name: task-decomposition
description: "Decompose complex, ambiguous, or multi-step goals into structured plans — invoke when a request has non-obvious steps, dependencies, or irreversible actions, or when unsure where to start."
metadata: { "openclaw": { "emoji": "🧩" } }
---

# Task Decomposition

Break goals into structured, executable plans before acting. Not every task needs this — use the triage heuristic below to decide. When decomposition is warranted, it prevents wasted work, surfaces blockers early, and produces better results.

## When to Decompose (Triage)

**Decompose when:**
- The goal is ambiguous or under-specified ("build me a dashboard")
- Multiple tools or skills are needed in sequence
- Steps have dependencies on each other's outputs
- The work involves irreversible actions (sending messages, publishing, deleting data)
- Your first reaction is uncertainty about where to start
- The task will take more than ~5 tool calls

**Skip decomposition when:**
- The task is a single clear action ("what time is it in Tokyo?", "read this file")
- You already have a well-practiced pattern for this task type
- The user gave explicit step-by-step instructions (they already decomposed it)
- Speed matters more than thoroughness (quick chat reply)

When in doubt, do a light decomposition (mental model only) rather than skipping entirely.

## Adaptive Depth

Pick the right level of planning for the task:

| Mode | When | What you do |
|---|---|---|
| **Light** | 2-3 step tasks, clear path | Think through steps mentally. No written plan. Quick sanity check, then act. |
| **Standard** | Multi-step with dependencies or risks | Write a plan using the format below. State it briefly to the user, then execute. |
| **Deep** | Complex, high-stakes, or unfamiliar tasks | Full plan with feasibility checks against self-model and env-model. Present to user for review before starting. |

Default to standard. Use light for routine work, deep for anything costly to get wrong.

## Decomposition Framework

### Step 1 — Goal Clarification

- Restate the goal in concrete terms
- Define "done" — what are the acceptance criteria?
- Surface assumptions you're making that haven't been stated
- Identify ambiguities that could change the approach
- Decide: ask the user to resolve ambiguities now, or proceed with reasonable defaults and note the assumptions?

### Step 2 — Sub-task Identification

Break the goal into discrete steps. Right granularity: each step should map to roughly one tool call or one skill invocation.

- Too coarse: "Build the backend"
- Too fine: "Type the letter 'a'"
- Right: "Create database schema", "Fetch API documentation", "Write test for auth endpoint"

Name each step as **verb + object**.

### Step 3 — Dependency Mapping

- Which steps need output from earlier steps?
- Which steps can run in parallel or any order?
- Are there external waits (user input, API responses, background tasks)?

Annotate dependencies inline — a simple list with `depends on: step N` is sufficient. No complex graph notation.

### Step 4 — Feasibility Check

For each step, check:

- **Can I do this?** → Read `skills/self-model/capabilities.md` if it exists. Check for required skills, tools, and binaries.
- **Does the environment support this?** → Read `skills/env-model/environment.md` if it exists. Check for sandbox constraints, connectivity, channel limits.

Flag any step that requires:
- A capability the agent doesn't have → candidate for skill acquisition or user escalation
- An environmental condition not met → address before starting

If the inventory or environment model files don't exist, note that feasibility is unchecked and proceed with reasonable caution.

### Step 5 — Risk Identification

For each step, briefly assess:
- Could this fail? What's the most likely failure mode?
- Is this irreversible? (If yes → require user confirmation)
- Is there a decision point where you might need to re-plan?
- What's the cost of getting it wrong? (Low → just try. High → plan carefully.)

### Step 6 — Present or Execute

- **High stakes / complex**: Present the plan to the user. Show steps, flag assumptions, ask for confirmation. Adapt format to the current channel.
- **Medium stakes**: State the plan briefly ("Here's my approach: X, Y, Z. Starting now.") and begin.
- **Low stakes**: Execute. The plan exists in your reasoning but doesn't need to be shown.

## Plan Format

Use this format for standard and deep decompositions. This format is shared with future skills (Task Composition, Failure Recovery) — keep it stable.

```markdown
## Plan: [Goal summary]

**Goal**: [concrete restatement]
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
```

A plan template is available at `{baseDir}/plan-template.md` for reference.

Keep plans proportional to the task. A 5-step task should take seconds to plan, not minutes. The plan is a tool, not a deliverable.

## Re-planning

Plans break. When they do:

| Situation | Response |
|---|---|
| A step fails and changes downstream assumptions | Re-plan from the failed step forward. Keep completed steps. |
| New information changes the goal or constraints | Re-plan from the top. Reference the original plan and explain what changed. |
| A step succeeds but produces unexpected output | Assess impact on remaining steps. Re-plan only if downstream steps are affected. |
| The user changes direction mid-execution | Re-plan from the top with the new goal. |

When re-planning, don't start from scratch unless the goal itself changed. Reference the original plan, note what broke, and adjust the remaining steps.

## Output

After decomposing:
- For **standard/deep**: show the plan (full or abbreviated based on stakes)
- For **light**: no output — proceed directly to execution
- Always note if feasibility checks were skipped (no inventory or env model available)

Do not over-explain the decomposition process itself. The user cares about the plan and the results, not the meta-reasoning.
````

### AGENTS-snippet.md

````markdown
- **Task Decomposition**: When a request is complex, ambiguous, multi-step, or involves irreversible actions → read `skills/task-decomposition/SKILL.md` and decompose before executing.
````
