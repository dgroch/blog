---
layout: post
title: "Attention Awareness: intelligence is also about ignoring the right things"
date: 2026-03-16
description: "Better reasoning is not just more reasoning. It is better allocation of cognitive focus."
---

There is a naïve theory of intelligence that says more attention is always better.


> The full implementation is available on GitHub: [github.com/dgroch/metacognition](https://github.com/dgroch/metacognition)

Read more. Track more. Consider more. Invoke more reasoning steps. Keep more context alive.

That theory fails quickly in real systems. Better cognition is not just about adding more thought. It is about allocating thought well.

That is what Attention Awareness is for.

[→ Jump to skill files](#skill-files)

The skill begins with a simple claim: not everything in context matters right now. The agent should distinguish four categories:

> **Active**: Information directly needed for the current step. Reason with this deeply.
>
> **Background**: Information that informs the work but doesn't need active attention (e.g., SOUL.md personality guidelines while writing code). Present but not primary.
>
> **Dormant**: Information from earlier that isn't relevant now but might resurface (e.g., a superseded conversation topic). Let it sit.
>
> **Noise**: Information that shouldn't influence current reasoning at all (e.g., verbose tool output from a completed diagnostic, old group chat messages). Ignore it.

Don't formally classify every context item. This is a mental model for focusing your reasoning, not a procedure.

That matters because agents often fail by over-processing. They drag old diagnostics, irrelevant side conversations, superseded assumptions, and verbose tool output forward into later steps as if all context were equally important. It isn't. Some information deserves attention. Some deserves memory. Some deserves neither.

The context-switch logic is also good. When a new request arrives, the skill gives a concrete decision framework:

> - Urgent + current work is pausable → save state (`.context-checkpoint.md`), switch
> - Urgent + current work at critical point → complete current step, then switch
> - Normal + current work almost done → finish current work, then address
> - Normal + current work early-stage → ask: "I'm working on X. Should I switch, or finish X first?"
> - Ambient → acknowledge, don't switch, continue current work
> - User explicitly redirects ("stop that, do this") → switch immediately
>
> **Always**: note what you were doing when you switch. Don't silently abandon work.

That sounds like project management, but it is really attentional triage. Urgent messages may justify switching. Ambient ones should not. Explicit user redirection overrides the queue. That is not profound. It is just the difference between a system that keeps a thread and one that thrashes.

The skill also contains a useful warning against over-metacognition. The guidance on when to invoke which skill is explicit:

> **Simple tasks** (single command, quick edit, factual question): Skip metacognitive protocols. Just act.
>
> **Medium tasks** (multi-file changes, unfamiliar domain, moderate complexity): Use 1-2 relevant skills — usually [Task Decomposition](https://dangroch.com/2026/03/16/task-decomposition-for-ai-agents/) + [Resource Selection](https://dangroch.com/2026/03/16/resource-selection-for-agent-tool-use/).
>
> **Complex tasks** (multi-step plans, cross-system work, high stakes): Use the full suite, but sequence efficiently:
> - [Self-Model](https://dangroch.com/2026/03/16/self-model-agent-capability-inventory/) and [Env-Model](https://dangroch.com/2026/03/16/env-model-agent-environment-awareness/) only if you don't already have current data
> - Decomposition before Composition
> - Calibration and Knowing-When-to-Ask as continuous habits, not separate invocations
> - Resource Selection at each plan step
>
> **Recovery situations**: [Failure Recovery](https://dangroch.com/2026/03/16/failure-recovery-for-ai-agents/) is the entry point — it references other skills as needed.

If you are making three or more tool calls of pure thinking without producing work, you are overdoing it. That line alone could rescue a lot of agents from self-inflicted paralysis.

The companion hook shows why this belongs in architecture, not just habit. `attention-filter` runs on `agent:bootstrap` and `message:preprocessed`. It looks for active work in files like `.active-plan.md` or `.context-checkpoint.md`, classifies incoming messages with cheap urgency patterns, skips heartbeat clutter, and injects notes like you were working on X, decide whether to switch or finish first. It also adds lightweight pointers to relevant artifacts such as the capability inventory, failure log, or task decomposition skill based on simple keyword matches.

No deep semantic engine. No mystical attention mechanism. Just system-level scaffolding that shapes what the agent sees as salient.

That is an important architectural point. Attention is not only an internal property of the model. It can be supported, directed, and constrained by the software around the model.

A concrete example makes this clearer. Suppose the agent is halfway through a multi-step plan and a new message arrives that contains the words urgent and broken. The hook marks the message as urgent. The skill then treats that as a possible switch, but not a guaranteed one; it still asks whether the current work is at a safe breakpoint. If the new message is ambient, the note may simply remind the agent not to abandon the in-progress task. This is what sane focus looks like under interruption.

The AGENTS trigger reflects the skill's own anti-pattern about over-metacognition:

> - **Attention Filtering**: When juggling multiple requests, when deciding what information to focus on, or when context feels noisy or overwhelming → read `skills/attention-awareness/SKILL.md` and focus on what matters now.

The broader thesis is that intelligence is partly subtractive. It is not only the ability to think about many things. It is the ability to not think about the wrong things at the wrong time.

Agents that cannot filter noise end up spending cognition on residue. Agents that over-invoke metacognition disappear into commentary about commentary. Attention awareness is the discipline that keeps reasoning pointed at the task rather than at the spectacle of reasoning.

---

## Skill Files

### SKILL.md

````markdown
---
name: attention-awareness
description: "Focus on what matters now — invoke when juggling multiple requests, when new signals arrive during in-progress work, when deciding what information to seek vs. ignore, or when context feels noisy or overwhelming."
metadata: { "openclaw": { "emoji": "🎯" } }
---

# Attention Awareness

Not everything in your context matters right now. Your job is to focus on what does, ignore what doesn't, and know the difference. An `attention-filter` hook shapes what you see each turn — this skill teaches you how to reason about relevance once you're thinking.

The goal: spend your reasoning on the task, not on processing noise. If you're thinking about thinking more than you're working, stop and do the work.

## Relevance Assessment

On each turn, ask internally: "Given what I'm doing right now, what in my context matters?"

**Active**: Information directly needed for the current step. Reason with this deeply.

**Background**: Information that informs the work but doesn't need active attention (e.g., SOUL.md personality guidelines while writing code). Present but not primary.

**Dormant**: Information from earlier that isn't relevant now but might resurface (e.g., a superseded conversation topic). Let it sit.

**Noise**: Information that shouldn't influence current reasoning at all (e.g., verbose tool output from a completed diagnostic, old group chat messages). Ignore it.

Don't formally classify every context item. This is a mental model for focusing your reasoning, not a procedure.

## Context-Switch Decision

When a new request arrives while you're working on something:

### Assess before switching:
1. **Priority of new request?** Urgent / Normal / Ambient
2. **State of current work?** Just started / Mid-progress / Almost done
3. **Can current work be paused cleanly?** Clean breakpoint vs. would lose state
4. **Does the new request supersede the current work?** User changed direction vs. separate task

### Decision:
- Urgent + current work is pausable → save state (`.context-checkpoint.md`), switch
- Urgent + current work at critical point → complete current step, then switch
- Normal + current work almost done → finish current work, then address
- Normal + current work early-stage → ask: "I'm working on X. Should I switch, or finish X first?"
- Ambient → acknowledge, don't switch, continue current work
- User explicitly redirects ("stop that, do this") → switch immediately

**Always**: note what you were doing when you switch. Don't silently abandon work.

## Focus During Multi-Step Work

When executing a plan from Task Decomposition:

- Read the plan to know which step you're on. Focus on that step's inputs, tool, and expected output.
- Don't re-evaluate the entire plan on every step unless something unexpected happens (that's Failure Recovery).
- After completing a step, verify the output briefly, then advance. Don't re-process raw output from completed steps.
- If a step produces large output, extract findings and move on.

## Information Foraging

The opposite of filtering noise: knowing when you don't have enough signal.

**Seek more information when:**
- The task requires facts you don't have and can't infer
- A plan step needs input that hasn't been gathered
- The user references something you don't recognize
- Your env-model or capability inventory is stale and the task depends on current state

**Work with what you have when:**
- You have enough for a useful first attempt
- Seeking more costs more than the quality improvement justifies
- The user will review and can correct (reversible)
- The new information is unlikely to change your approach

This complements Epistemic Calibration: calibration asks "how confident am I?" — attention asks "is it worth seeking more?"

## Group Chat and Multi-Party Focus

- Track the primary thread — who addressed you, about what
- Ignore tangential group conversation unless @mentioned or directly relevant
- Multiple simultaneous requests: prioritize owner/operator, then urgency, then order received
- Don't maintain context on every parallel thread — focus on the active one

## When to Invoke Which Metacognitive Skill

The full suite has 10 triggers in AGENTS.md. Don't over-metacognate — spending more time on metacognitive overhead than actual work is a failure mode.

**Simple tasks** (single command, quick edit, factual question): Skip metacognitive protocols. Just act.

**Medium tasks** (multi-file changes, unfamiliar domain, moderate complexity): Use 1-2 relevant skills — usually Task Decomposition + Resource Selection.

**Complex tasks** (multi-step plans, cross-system work, high stakes): Use the full suite, but sequence efficiently:
- Self-Model and Env-Model only if you don't already have current data
- Decomposition before Composition
- Calibration and Knowing-When-to-Ask as continuous habits, not separate invocations
- Resource Selection at each plan step

**Recovery situations**: Failure Recovery is the entry point — it references other skills as needed.

The AGENTS.md triggers provide conditions. Match conditions to your situation. Invoke only what's relevant. Developing this intuition IS the metacognitive skill.

## Anti-Patterns

**Completionism**: Reading every metacognitive artifact on every turn. The inventory, env-model, failure log, composition log — these are reference material, not required reading.

**Analysis paralysis**: 3+ tool calls of thinking without producing work = over-metacognating. Stop and do the task.

**Attention thrashing**: Switching between threads or tasks rapidly without progress on any. Better to finish one thing than touch five.

**Noise amplification**: Treating every piece of context as equally important. A diagnostic `ls` output from five turns ago doesn't need to influence current reasoning.

**Metacognitive recursion**: Using the attention skill to reason about using the attention skill. If you're thinking about thinking about thinking, stop and work.
````

### AGENTS-snippet.md

````markdown
- **Attention Filtering**: When juggling multiple requests, when deciding what information to focus on, or when context feels noisy or overwhelming → read `skills/attention-awareness/SKILL.md` and focus on what matters now.
````

---

**Next in the series:** [Attention Filter: shaping what the agent sees before it starts thinking](https://dangroch.com/2026/03/16/attention-filter-hook/)
