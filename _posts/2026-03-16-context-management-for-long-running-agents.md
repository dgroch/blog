---
layout: post
title: "Context Management: thinking is limited by what you can keep in mind"
date: 2026-03-16
description: "Long-running agents don't just need bigger windows. They need better context hygiene."
---

A lot of agent design still treats context as if it were just more memory. Give the system a bigger window, better retrieval, maybe a summary layer, and the problem goes away.

It doesn't.

Context is not just storage. It is working memory. And working memory is not valuable because it is large. It is valuable because the right things stay legible while the wrong things stay out of the way.

That is the problem Context Management is addressing.

[→ Jump to skill files](#skill-files)

The skill begins with the obvious but often ignored fact that context windows are finite and quality degrades before compaction happens. That warning is important. Many systems behave as if the only failure is a hard context reset. In practice the earlier failure is softer and nastier: responses get less focused, earlier decisions blur, old diagnostics cling to attention, and the agent starts repeating itself.

OpenClaw's answer is partly strategic and partly infrastructural. Strategically, the skill tells the agent when to keep work in the main session and when to spawn a sub-agent. The thresholds are refreshingly concrete:

> **Keep in main session:**
> - Single commands, conversations, quick edits (1–3 files)
> - Tasks requiring user input mid-execution
> - Tasks where conversational context is essential
>
> **Spawn a sub-agent when:**
> - The task involves >5 tool calls and context is above 50%
> - The task involves >2 tool calls and context is above 70%
> - The task produces large outputs (file reads, search results, API responses) that would bloat the main session
> - The task is independent of the current conversation

That is not magical. It is capacity planning.

The output hygiene section is even better because it reads like a set of habits for sane operators:

> - **Extract, don't load**: When reading large files, target what you need rather than loading the entire file
> - **Summarize large outputs**: Before continuing after a large tool result, distill it to what matters rather than carrying raw results forward
> - **Write intermediates to files**: When a plan step produces a large intermediate result, write it to a file and reference the path
> - **Prefer targeted commands**: `grep` for a specific line over `cat` of the whole file
> - **Don't re-read unchanged files**: Avoid reading files you've already read this session unless content may have changed

That is context discipline in plain engineering language.

The skill also explicitly recommends writing active state to `.active-plan.md` before compaction or during long-running work. That file matters because it turns a conversational thread back into a recoverable execution state: current step, decisions made, what comes next. Again, this is a system refusing to treat memory as something the model should just somehow manage internally.

The post-compaction recovery protocol is explicit:

> 1. Check for `.context-checkpoint.md` or `.context-checkpoint.recovered.md` (the context-guard hook may have created one)
> 2. Read today's memory file: `memory/YYYY-MM-DD.md`
> 3. Read `memory.md` for durable facts
> 4. The self-model (`skills/self-model/capabilities.md`) and env-model (`skills/env-model/environment.md`) persist on disk — re-read them, don't rebuild
> 5. Check the failure log (`skills/failure-recovery/failure-log.md`) and composition log (`skills/task-composition/composition-log.md`) — they persist too
> 6. Resume the active plan from `.active-plan.md` if one exists

The hook relationship is where the design gets interesting. The skill says a `context-guard` hook runs in the background, checkpointing metacognitive state before compaction and restoring it after. In other words, the strategic layer and the automated layer cooperate. The agent should manage its context proactively, but the infrastructure is there as a backstop when reality catches up.

That split is right. Some context hygiene belongs to judgment. Some belongs to plumbing.

Imagine a long session involving repo inspection, writing, retries, and cross-references between several files. Without context management, the system ends up carrying raw file contents, stale diagnostics, half-formed plans, and redundant summaries all at once. With it, the agent writes the plan to disk, extracts what matters from each large read, delegates self-contained work when appropriate, and relies on the hook to preserve the metacognitive layer across compaction. Same model. Different behaviour.

There is also an epistemic angle here. After compaction, the skill warns that confidence in prior session context should drop. Re-read persistent artifacts like `capabilities.md`, `environment.md`, and the failure log instead of pretending the compacted summary is lossless. That is a subtle but important move. Context loss is not just an inconvenience. It changes what the system should trust.

The AGENTS trigger reflects exactly this:

> - **Context Management**: When context is getting heavy, when deciding whether to spawn a sub-agent, or when preparing for a long-running task → read `skills/context-management/SKILL.md` and manage context proactively.

This is why I think context management belongs squarely inside metacognition rather than being treated as an infrastructure footnote. A system that cannot reason about what it can keep in mind cannot reason reliably over long horizons, no matter how impressive it looks on short tasks.

Thinking is limited by what stays active, what gets compressed, what is written out, and what is deliberately ignored. Bigger windows help. Better habits help more.

---

## Skill Files

### SKILL.md

````markdown
---
name: context-management
description: "Manage context window proactively — invoke when context is getting heavy, when deciding whether to spawn a sub-agent, when preparing for long-running tasks, or when output quality is degrading from context pressure."
metadata: { "openclaw": { "emoji": "📦" } }
---

# Context Management

Your context window is finite. Every tool call, file read, and conversation turn consumes space. Context pressure degrades output quality before it triggers compaction — be proactive, don't wait for the system to force a reset.

A `context-guard` hook runs in the background to checkpoint metacognitive state before compaction and recover after. But the hook is plumbing — your own context hygiene is the first line of defense.

## Context Awareness

- Check context status via `/status` or `session_status` when you suspect pressure
- Context pressure signs: responses becoming less focused, losing track of earlier decisions, repeating yourself
- The context-guard hook will checkpoint your metacognitive state (plans, failure log, model freshness) before compaction — but you should still be proactive about keeping context lean

## Spawn Decision Framework

### Keep in main session:
- Single commands, conversations, quick edits (1–3 files)
- Tasks requiring user input mid-execution
- Tasks where conversational context is essential

### Spawn a sub-agent when:
- The task involves >5 tool calls and context is above 50%
- The task involves >2 tool calls and context is above 70%
- The task produces large outputs (file reads, search results, API responses) that would bloat the main session
- The task is independent of the current conversation (e.g., "also go fix that bug in the other repo")

### When spawning:
Write detailed task descriptions. Sub-agents have no conversation context — they only know what the task field tells them. Include: what to do, what files matter, what constraints apply, and where to write results.

## Output Hygiene

Be context-efficient:

- **Extract, don't load**: When reading large files, target what you need rather than loading the entire file
- **Summarize large outputs**: Before continuing after a large tool result, distill it to what matters rather than carrying raw results forward
- **Write intermediates to files**: When a plan step produces a large intermediate result, write it to a file and reference the path
- **Prefer targeted commands**: `grep` for a specific line over `cat` of the whole file
- **Don't re-read unchanged files**: Avoid reading files you've already read this session unless content may have changed

## Pre-Compaction Preparation

When you sense context is getting heavy:

1. **Persist important state**: Write key decisions and active task state to files proactively. The context-guard hook checkpoints automatically, but you can also persist critical context to `{workspace}/.active-plan.md`
2. **Complete discrete units**: Finish the current sub-task if possible before compaction hits. Half-done work is hard to resume.
3. **Write plan state**: If a complex plan is in progress, write current progress to `{workspace}/.active-plan.md` — step status, decisions made, what's next
4. **Clean up**: Remove temporary files or intermediate outputs that are no longer needed

## Post-Compaction Recovery

After compaction, recover your working context:

1. Check for `.context-checkpoint.md` or `.context-checkpoint.recovered.md` (the context-guard hook may have created one)
2. Read today's memory file: `memory/YYYY-MM-DD.md`
3. Read `memory.md` for durable facts
4. The self-model (`skills/self-model/capabilities.md`) and env-model (`skills/env-model/environment.md`) persist on disk — re-read them, don't rebuild
5. Check the failure log (`skills/failure-recovery/failure-log.md`) and composition log (`skills/task-composition/composition-log.md`) — they persist too
6. Resume the active plan from `.active-plan.md` if one exists

## Integration with Metacognitive Skills

**Task Decomposition**: When decomposing, consider context cost of each step. Steps that produce large outputs are candidates for sub-agent delegation.

**Resource Selection**: Context cost is a selection criterion. A bash one-liner that returns 3 lines beats a Python script that outputs 50. Prefer lightweight tools.

**Failure Recovery**: Failed tool calls waste context. The retry limit (max 2–3 from skill 5) protects against context burn. After repeated failures, stop and reassess rather than burning more context.

**Self-Model / Env-Model**: These persist on disk and survive compaction. Don't rebuild them — just re-read. Don't keep them in context — reference the file paths.

**Epistemic Calibration**: Post-compaction, your confidence in session context drops. Treat compacted information as lower reliability (compaction artifacts — see epistemic calibration traps). Re-verify anything critical.
````

### AGENTS-snippet.md

````markdown
- **Context Management**: When context is getting heavy, when deciding whether to spawn a sub-agent, or when preparing for a long-running task → read `skills/context-management/SKILL.md` and manage context proactively.
````
