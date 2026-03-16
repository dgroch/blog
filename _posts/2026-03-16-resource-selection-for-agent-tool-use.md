---
layout: post
title: "Resource Selection: the best tool is rarely the first one the model remembers"
date: 2026-03-16
description: "Tool choice is not a convenience layer. It is one of the agent's core cognitive decisions."
---

Many agent failures are not failures of reasoning in the abstract. They are failures of choosing how to act.


> The full implementation is available on GitHub: [github.com/dgroch/metacognition](https://github.com/dgroch/metacognition)

The model knows the goal. It even has the tools. But it reaches for the wrong one because it is familiar, overgeneralized, or narratively convenient.

This is what Resource Selection is designed to correct.

[→ Jump to skill files](#skill-files)

The first line of the skill says it well: pause between "I know what to do" and "I'm doing it with X." That pause is small, but it is doing serious cognitive work.

A lot of current agent design quietly assumes tool use is a mechanical afterthought. Once the model understands the task, surely the right tool will follow. In practice the opposite is often true. The task is underspecified until you know which resource you're committing to, because different tools have different costs, failure modes, output shapes, and environmental requirements.

The selection framework starts by forcing the sub-task to be framed abstractly. The skill draws a sharp contrast:

> - Bad: "I need to write a Python script to parse this JSON"
> - Good: "I need to extract the 'users' array from this JSON file and get the email addresses"

That is such a small instruction and such a good one. Tool-shaped thinking is one of the fastest ways an agent narrows its own options before it has even reasoned about them.

Then it generates a few genuine candidates by consulting `capabilities.md` and `environment.md`. Core tools like bash, read, write, and edit sit alongside installed CLIs, browser automation, sub-agents, connected nodes, external APIs, and the possibility of just asking the user. The presence of "ask the user" in a tool-selection framework is quietly important. Sometimes the best resource is not another layer of automation. It is a ten-second clarification.

The evaluation dimensions are simple and exactly right:

| Dimension | Question |
|---|---|
| **Fitness** | Does this tool handle the exact requirement, or do I need to wrangle the output? |
| **Efficiency** | How many tokens / tool calls / seconds? A one-liner beats a 40-line script for the same result. |
| **Reliability** | How likely to work on the first try? Check `{baseDir}/failure-log.md` — has this approach failed for similar tasks? |
| **Environmental fit** | Does this work in the current environment? Check `{baseDir}/environment.md` for constraints. |
| **Reversibility** | If wrong, how easy to undo? File writes are more reversible than sent messages or API calls. |
| **Simplicity** | All else equal, prefer simpler. Fewer moving parts = fewer failure modes. One tool beats chaining three. |

The heuristics file is even more useful because it attacks common biases directly. On the bash vs. Python decision:

> - **Bash wins**: simple text processing, file manipulation, piping between CLI tools, one-shot commands, anything naturally a pipeline
> - **Python wins**: complex logic, data structures, error handling matters, requires libraries, structured output
> - **Heuristic**: if you can express it as a pipeline of existing commands in under 3 lines, use bash

And on CLI tools vs. custom code:

> - **CLI tool wins**: one exists that handles the exact task (jq for JSON, ffmpeg for media, imagemagick for images, curl for HTTP, git for version control)
> - **Custom code wins**: domain-specific logic no existing tool covers, or the tool's output needs significant post-processing
> - **Heuristic**: always check if a CLI tool exists before writing code. Many agents default to code when a one-liner would work.

Language models have strong familiarity bias. They often want to write code because code is their native performance medium. But many operational tasks are better solved by `jq`, `curl`, `grep`, `sort`, `uniq`, `git`, or `ffmpeg`. A model without structured resource selection will happily produce forty lines of Python where a one-liner would be faster, cheaper, and more reliable.

That is not a cosmetic inefficiency. It affects failure rates, latency, token use, and trust.

The integration with [Failure Recovery](https://dangroch.com/2026/03/16/failure-recovery-for-ai-agents/) is another good touch. If an approach fails, pivot by re-selecting from the remaining candidates with the failed option removed. That sounds obvious, but it's not how agents naturally behave. They often keep reworking the same doomed path because sunk cost feels like persistence.

The skill names this explicitly as an anti-pattern:

> **Sunk cost**: Sticking with a struggling approach because you already started it. If three tool calls in, the approach is clearly wrong, switch. Don't optimize a bad choice.

Suppose the agent needs to inspect structured JSON in a repo, extract two fields, and write a manifest. The default language-model reflex is often to write a script. Resource Selection asks a better question: is `jq` installed? Can bash do this in one line? Is the output small enough for inline work? Does a sub-agent help, or would that be needless overhead? If the environment lacks `jq`, does Python become the right fallback? The point is not that one tool is always superior. The point is that the choice should be deliberate.

The AGENTS trigger reflects that:

> - **Resource/Tool Selection**: When multiple approaches exist for a sub-task, when the default tool might not be the best fit, or when efficiency and reliability matter → read `skills/resource-selection/SKILL.md` and choose deliberately.

Tool selection is a self-monitoring problem. The agent is not just trying to solve the task. It is deciding which of its own action pathways best fits the task under current conditions. That is one of the places where capable and intelligent start to diverge.

---

## Skill Files

### SKILL.md

````markdown
---
name: resource-selection
description: "Choose the best tool or approach for a sub-task — invoke when multiple viable approaches exist, when the default tool might not be the best fit, when efficiency or reliability matters, or when environmental constraints rule out the obvious option."
metadata: { "openclaw": { "emoji": "⚖️" } }
---

# Resource/Tool Selection

Pause between "I know what to do" and "I'm doing it with X." Choose the best approach, not the first one that comes to mind.

This skill must make you faster, not slower. Most tool choices are obvious — this skill fires only when the choice matters.

## When to Deliberate (and When to Just Act)

**Deliberate when:**
- Multiple viable approaches exist and they differ meaningfully in cost, speed, reliability, or output quality
- Your instinct is to use a complex approach but a simpler one might work
- The sub-task involves unfamiliar territory where tool choice isn't obvious
- Previous attempts at similar tasks failed with the default approach (check `{baseDir}/failure-log.md`)
- Token budget is tight, latency matters, or the operation is expensive
- Environmental constraints rule out the obvious approach (sandbox, no browser, no network)

**Just act when:**
- Only one approach exists or is clearly dominant
- The task is trivial (reading a file, running a simple command)
- You've successfully used this approach for this exact type of task before and conditions haven't changed
- The cost of choosing wrong is negligible (easily reversible, low-stakes)

## The Selection Framework

### Step 1 — Frame the Sub-Task Abstractly

Before considering tools, describe what you're actually trying to accomplish in tool-agnostic terms. This prevents premature commitment.

- Bad: "I need to write a Python script to parse this JSON"
- Good: "I need to extract the 'users' array from this JSON file and get the email addresses"

Abstract framing opens options you'd miss if you jumped straight to implementation.

### Step 2 — Generate Candidates

Consult `{baseDir}/capabilities.md` (self-model) and `{baseDir}/environment.md` (env-model) to identify viable approaches. If the capability inventory doesn't exist, build it first (invoke the self-model skill).

Think across these categories:

**Core tools (Pi's four):**
- **Bash**: shell commands, pipelines, CLI tools. Best for quick data manipulation, file operations, system queries, anything with a good CLI tool.
- **Read**: viewing files and directories. Best for understanding content before acting, checking state.
- **Write**: creating new files. Best for generating output, creating scripts, producing deliverables.
- **Edit**: modifying existing files. Best for targeted changes to existing content.

**Extended capabilities:**
- **Installed CLIs**: jq, ffmpeg, imagemagick, uv, git, curl, etc. Check the self-model. Often the best single-tool answer.
- **Skills**: installed OpenClaw skills may handle the task end-to-end. Check the skill list.
- **Browser**: for web-based tasks, scraping, or interacting with web UIs. Higher overhead but sometimes the only option.
- **Connected nodes**: a macOS or iOS node may have capabilities the local environment doesn't. Check the env-model.
- **Sub-agents**: for long-running or complex coding tasks, spawning a sub-agent may be more efficient. Especially relevant for tasks that would bloat the current context.
- **External APIs**: direct HTTP calls via curl when a service has a REST API.
- **Ask the user**: sometimes the fastest path. Not a failure — it's a valid selection when the user has information or access you don't.

Generate 2–4 genuine candidates, not an exhaustive list. Skip approaches that are obviously worse.

### Step 3 — Evaluate Candidates

Compare candidates across these dimensions (comparative reasoning, not scores):

| Dimension | Question |
|---|---|
| **Fitness** | Does this tool handle the exact requirement, or do I need to wrangle the output? |
| **Efficiency** | How many tokens / tool calls / seconds? A one-liner beats a 40-line script for the same result. |
| **Reliability** | How likely to work on the first try? Check `{baseDir}/failure-log.md` — has this approach failed for similar tasks? |
| **Environmental fit** | Does this work in the current environment? Check `{baseDir}/environment.md` for constraints. |
| **Reversibility** | If wrong, how easy to undo? File writes are more reversible than sent messages or API calls. |
| **Simplicity** | All else equal, prefer simpler. Fewer moving parts = fewer failure modes. One tool beats chaining three. |

### Step 4 — Select and Commit

Choose the best candidate. State the choice briefly: "Using jq for this — it handles the JSON extraction natively."

Once committed, don't second-guess mid-execution. If the approach fails, that's a failure recovery event (skill 5) — re-select from remaining candidates with the failed option eliminated (the Pivot action).

## Common Selection Heuristics

These cover the most frequent decisions. See `{baseDir}/heuristics.md` for a compact reference.

### Bash vs Python
- **Bash wins**: simple text processing, file manipulation, piping between CLI tools, one-shot commands, anything naturally a pipeline
- **Python wins**: complex logic, data structures, error handling matters, requires libraries, structured output
- **Heuristic**: if you can express it as a pipeline of existing commands in under 3 lines, use bash

### CLI Tool vs Custom Code
- **CLI tool wins**: one exists that handles the exact task (jq for JSON, ffmpeg for media, imagemagick for images, curl for HTTP, git for version control)
- **Custom code wins**: domain-specific logic no existing tool covers, or the tool's output needs significant post-processing
- **Heuristic**: always check if a CLI tool exists before writing code. Many agents default to code when a one-liner would work.

### Inline Work vs Sub-Agent
- **Inline wins**: task is quick (< 5–10 tool calls), context already in session, part of a larger interactive flow
- **Sub-agent wins**: task is long-running, requires deep exploration, would bloat context, or can run in background
- **Heuristic**: if you'd need to read more than 5–10 files to do the task, consider a sub-agent

### Local vs Node
- **Local wins**: capability exists locally with no advantage to running elsewhere
- **Node wins**: required capability only exists on the node (macOS-specific tools, camera, iOS features), or the task benefits from a different OS
- **Heuristic**: check `{baseDir}/environment.md` for connected nodes and their capabilities before assuming local-only

### Do It Yourself vs Ask the User
- **Do it yourself**: information is discoverable, action is within capabilities, decision has low stakes
- **Ask the user**: decision involves preferences you can't infer, information isn't discoverable, action has irreversible consequences, or asking takes 10 seconds vs. investigating for 5 minutes
- **Heuristic**: if you're about to spend significant effort guessing what the user wants, it's faster to ask

## Selection in Plans

When working with a decomposed plan (from task-decomposition, skill 3), selection happens at each step:

1. **Review the full plan before selecting tools for individual steps** — a tool choice in step 2 might constrain or enable choices in step 4 (e.g., if step 2 produces JSON, step 4 can use jq)
2. **Prefer consistent tooling across related steps** when the difference is marginal — switching between Python and bash and node for successive steps adds overhead
3. **Populate the "Tool/skill" field** in each plan step with a deliberate choice rather than a default

The plan step format (from skill 3):
```
1. [Verb + object] — [description]
   - Depends on: [step or none]
   - Tool/skill: [deliberate selection goes here]
   - Risk: [description or none]
```

## Learning from Past Selections

Two mechanisms — no separate selection log needed:

1. **Failure log consultation** (`{baseDir}/failure-log.md`): Before selecting, check if similar tasks failed with a particular approach. If approach X failed for "JSON transformation" last week, prefer approach Y.
2. **Session heuristics**: Develop awareness through experience: "For this type of task in this environment, [approach] works well." This happens naturally through the failure log (negative signal) and successful completions (positive signal in session context).

## Coordination with Other Skills

**Self-Model** (`skills/self-model/capabilities.md`): Hard dependency. Candidate generation reads the capability inventory. If the inventory doesn't exist, invoke the self-model skill to build it before doing selection on a complex task.

**Env-Model** (`skills/env-model/environment.md`): Environmental fit evaluation. Disqualify candidates that don't work in the current environment. Check for sandbox restrictions, browser status, connected nodes, network access.

**Task Decomposition** (`skills/task-decomposition/SKILL.md`): Selection populates the "Tool/skill" field in plan steps. Abstract task framing (Step 1) aligns with how decomposition describes steps — both should be tool-agnostic at the description level.

**Task Composition** (`skills/task-composition/SKILL.md`): When no single tool is ideal, selection can invoke composition to chain simpler tools. If candidate generation reveals that two lightweight tools together outperform one heavyweight tool, that's a composition opportunity.

**Failure Recovery** (`skills/failure-recovery/SKILL.md`): The Pivot action is re-selection with the failed option removed. Check `{baseDir}/failure-log.md` for negative signal on similar past tasks before selecting.

**Epistemic Calibration** (`skills/epistemic-calibration/SKILL.md`): When you're uncertain a tool can handle the task, that's moderate or low confidence. Prefer approaches you're confident about, especially for high-stakes steps. For uncertain approaches, consider testing on a small input first.

## Anti-Patterns

**Familiarity bias**: Choosing Python for everything because you have the most training data on it. Bash, jq, awk, sed, and CLI tools often solve the same problem in 1/10th the tokens.

**Complexity bias**: Choosing the most sophisticated approach when a simple one works. Writing a script to do what `grep | sort | uniq` handles.

**Sunk cost**: Sticking with a struggling approach because you already started it. If three tool calls in, the approach is clearly wrong, switch. Don't optimize a bad choice.

**Tool-shaped thinking**: Framing the problem in terms of a specific tool ("I need a Python script that...") instead of the actual goal ("I need to extract..."). Always frame abstractly first.

**Ignoring the environment**: Choosing an approach that requires capabilities not present. Always check `{baseDir}/environment.md`.

**Over-delegating**: Spawning a sub-agent for something that takes two bash commands. Sub-agents have setup overhead — use them for substantial work.

**Under-delegating**: Doing 50 tool calls inline for a task a background sub-agent could handle while you continue the conversation.
````

### AGENTS-snippet.md

````markdown
- **Resource/Tool Selection**: When multiple approaches exist for a sub-task, when the default tool might not be the best fit, or when efficiency and reliability matter → read `skills/resource-selection/SKILL.md` and choose deliberately.
````
