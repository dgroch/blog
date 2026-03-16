---
layout: post
title: "Failure Recovery: real agents need more than retries"
date: 2026-03-16
description: "Robust systems don't just fail less. They fail with diagnosis, adaptation, and memory."
---

Blind retries are not resilience. They are just faster repetition of the same mistake.


> The full implementation is available on GitHub: [github.com/dgroch/metacognition](https://github.com/dgroch/metacognition)

Failure Recovery in OpenClaw starts from that premise and builds a much healthier response to things going wrong.

[→ Jump to skill files](#skill-files)

The skill first expands what counts as failure. Not only explicit tool errors, but silent failures, behavioural failures, and cascading failures:

> **Explicit failures:**
> - Tool call returns an error (non-zero exit code, error message, exception)
> - Output contains error keywords: permission denied, not found, connection refused, timeout, command not found
>
> **Silent failures:**
> - Command succeeds (exit code 0) but output is empty, truncated, or wrong
> - A file was supposed to be created but wasn't, or has wrong content
>
> **Behavioral failures:**
> - You're about to attempt the same action you just tried — loop detected
> - The user says "that's not what I meant" or "that's wrong" — this is a failure even if every tool call succeeded
>
> **Cascading failures:**
> - A step succeeds but its output is malformed, causing the next step to fail
> - An early assumption turns out wrong, invalidating everything downstream

A command that exits zero but produces empty output is a failure. A user saying "that's not what I meant" is a failure. A plan that completes step by step but ends in an incoherent result is a failure. That broader definition matters because agents are otherwise too willing to treat tool success as task success.

Then comes diagnosis. The skill classifies failures into seven types, ordered by likelihood:

> 1. **Transient Failure** — timeout, rate limit, temporary network issue
> 2. **Environmental Constraint** — permission denied, sandbox restriction, missing binary
> 3. **Input/Data Problem** — malformed input, wrong file format, missing data
> 4. **Capability Gap** — command not found, missing skill, task requires something you don't have
> 5. **Logic/Approach Error** — the plan is flawed, wrong step order, missed dependency
> 6. **Goal Misunderstanding** — the user says "that's not what I meant"
> 7. **Fundamental Impossibility** — literally cannot be done with available capabilities

The `failure-log.md` shortcut is practical:

> **Shortcut**: Before diagnosing from scratch, check `{baseDir}/failure-log.md` — "Have I seen this failure before?" If yes, skip to the recovery that worked last time.

That log file is more important than it looks. Without it, every failure is local. With it, a system can accumulate operational memory about what tends to go wrong and what tends to fix it.

The recovery actions are simple and strong:

> **Retry** — same approach with adjustments. Max 2–3 retries. Change something between each retry — never repeat the exact same command.
>
> **Pivot** — different approach to the same goal. Consult `skills/self-model/capabilities.md` for alternative capabilities. Compose a different combination.
>
> **Escalate** — ask the user for help. Be specific: "Step 3 failed because [X]. I tried [Y] and [Z]. I need [specific thing] from you to continue."
>
> **Abort** — stop the current task. Explain what was accomplished, what wasn't, and why continuing isn't viable.

The escalation ladder is the right kind of strict: retry → pivot → escalate → abort. Don't keep recovering from the recovery. One meta-level is enough.

When a decomposed plan has a failing step, the repair is surgical:

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

This is where the suite starts to feel like practical cognition rather than a bag of prompts. Failure Recovery does not live alone. It reads from [Self-Model](https://dangroch.com/2026/03/16/self-model-agent-capability-inventory/) to distinguish capability gaps from environment problems. It can trigger [Task Decomposition](https://dangroch.com/2026/03/16/task-decomposition-for-ai-agents/) when the plan itself is wrong. It works with the shared plan format so a failed step can become step `3b` without rewriting everything from scratch. It updates understanding rather than just apologizing.

That last part is the heart of it. Most agent failure modes are bad not because they happen, but because the system learns almost nothing from them.

Suppose an agent tries a browser-based approach, hits a permissions problem, then keeps nudging the same path because browser automation was the first plan it formed. A system using this skill should instead classify the issue as environmental, consult the env-model, pivot to a direct API call or CLI tool if one exists, update the plan, and log the lesson. That is qualitatively different behaviour.

The anti-pattern list reads like a short obituary for current agent robustness:

> | Anti-pattern | What it looks like | What to do instead |
> |---|---|---|
> | **Blind retry loop** | Same command, same error, three times in a row | Change something between retries. After 2 failures, pivot. |
> | **Silent failure absorption** | Error happens, agent continues as if it didn't | Always address a failure before moving on. |
> | **Scope creep in recovery** | "Fixing" the failure turns into installing 3 tools | That's an escalation to the user, not a recovery. |
> | **Apologize-and-quit** | "Sorry, I can't do that." | Diagnose, explain why, suggest alternatives. |
> | **Recursive recovery** | Recovery fails, agent tries to recover from the recovery | Escalate one level. One meta-level is enough. |

The AGENTS trigger fires on the signal agents most need:

> - **Failure Recovery**: When a tool call errors, output is unexpected, you catch yourself retrying the same approach, or the user corrects your output → read `skills/failure-recovery/SKILL.md` and follow the recovery framework.

There is also a deeper metacognitive point here. Intelligence is not just about succeeding on the first pass. It is about preserving coherence when the first pass breaks. Diagnosis, adaptation, and explicit learning are what make failure survivable.

That is why robust systems do not merely fail less. They fail better. And failing better is one of the clearest signs that a system is reasoning about its own reasoning rather than just producing another confident guess.

---

---

← [Previous: Knowing When to Ask](/2026/03/16/knowing-when-to-ask-agents/) · [Next: Context Management](/2026/03/16/context-management-for-long-running-agents/) →


## Skill Files

### SKILL.md

````markdown
---
name: failure-recovery
description: "Diagnose and recover from failures — invoke when a tool call errors, output is unexpected, you catch yourself retrying the same approach, a plan step fails, or the user corrects your output."
metadata: { "openclaw": { "emoji": "🔧" } }
---

# Failure Recovery

When something goes wrong, don't blindly retry and don't immediately surrender. Diagnose what failed, classify why, choose the right recovery action, and learn from it. Speed matters — classify fast, act decisively.

## Failure Detection

Not all failures are obvious. Check for all four categories:

**Explicit failures:**
- Tool call returns an error (non-zero exit code, error message, exception)
- Output contains error keywords: permission denied, not found, connection refused, timeout, command not found
- An API returns an error status

**Silent failures:**
- Command succeeds (exit code 0) but output is empty, truncated, or wrong
- A file was supposed to be created but wasn't, or has wrong content
- A multi-step process completed but the end result doesn't match the goal

**Behavioral failures:**
- You're about to attempt the same action you just tried — loop detected
- The user says "that's not what I meant" or "that's wrong" — this is a failure even if every tool call succeeded
- A composed workflow produces correct individual steps but an incoherent overall result

**Cascading failures:**
- A step succeeds but its output is malformed, causing the next step to fail
- An early assumption turns out wrong, invalidating everything downstream

Develop the instinct: after acting, verify. "Did this actually work?" is a different question from "did this throw an error?"

## Failure Diagnosis

Classify the failure to determine recovery strategy. Work through these types in order of likelihood — check common causes first:

### 1. Transient Failure

- **Symptoms**: timeout, rate limit, temporary network issue, resource briefly unavailable
- **Signature**: the same action worked before or works for others; nothing fundamental changed
- **Recovery**: retry with backoff (max 2-3 retries), then escalate to Pivot

### 2. Environmental Constraint

- **Symptoms**: permission denied, sandbox restriction, missing binary, network blocked, node disconnected
- **Signature**: the capability exists (check `skills/self-model/capabilities.md`) but the environment prevents it (check `skills/env-model/environment.md`)
- **Recovery**: resolve the constraint if possible (install binary, request elevated mode, wait for reconnection), or find an alternative approach that works within the constraint

### 3. Input/Data Problem

- **Symptoms**: malformed input, wrong file format, missing data, unexpected schema
- **Signature**: the tool/skill works in general but this specific input is problematic
- **Recovery**: fix or transform the input, request corrected input from the user, or skip the problematic item and continue

### 4. Capability Gap

- **Symptoms**: command not found, missing skill, the task requires something you don't know how to do
- **Signature**: `skills/self-model/capabilities.md` confirms the capability doesn't exist; `skills/self-model/capability-gaps.md` may already list it
- **Recovery**: route to skill acquisition if the gap is closable, or escalate to the user. Update `skills/self-model/capability-gaps.md` if this is a new gap.

### 5. Logic/Approach Error

- **Symptoms**: the plan is flawed — wrong step order, missed dependency, wrong tool for the sub-task
- **Signature**: individual tool calls succeed but the overall approach doesn't produce the desired result
- **Recovery**: re-plan by invoking task decomposition (`skills/task-decomposition/SKILL.md`) with the new understanding of why the original approach failed. Consider invoking task composition (`skills/task-composition/SKILL.md`) to find an alternative capability combination.

### 6. Goal Misunderstanding

- **Symptoms**: the user says "that's not what I meant," output is technically correct but doesn't serve the actual need
- **Signature**: failure is in interpretation, not execution
- **Recovery**: clarify the goal with the user. Re-plan from scratch — don't just tweak the output, revisit the goal itself.

### 7. Fundamental Impossibility

- **Symptoms**: after diagnosis, this literally cannot be done with available capabilities and environment
- **Signature**: self-model confirms no capability, env-model confirms no workaround, no learnable skill would close the gap
- **Recovery**: tell the user clearly, explain why, suggest the closest achievable alternative

**Shortcut**: Before diagnosing from scratch, check `{baseDir}/failure-log.md` — "Have I seen this failure before?" If yes, skip to the recovery that worked last time.

## Recovery Actions

After classification, choose one of four actions:

### Retry — same approach with adjustments

- **When**: transient failures, or when a small tweak (different flag, corrected input) is likely to succeed
- **Rules**: max 2-3 retries for the same failure. Change something between each retry — never repeat the exact same command. If retry 2 fails, escalate to Pivot.
- **Always**: note what changed between retries

### Pivot — different approach to the same goal

- **When**: the current approach is fundamentally blocked but the goal is achievable via a different path
- **How**: consult `skills/self-model/capabilities.md` for alternative capabilities. Read `skills/task-composition/SKILL.md` and compose a different combination. Re-plan the failed step or the entire plan.
- **Examples**: direct API call fails → try browser-based approach. Python script fails → try bash one-liner. Local tool fails → try a node-based alternative.

### Escalate — ask the user for help

- **When**: you've exhausted your diagnostic ability, the failure requires information or access you don't have, or the decision has consequences you shouldn't make alone
- **How**: be specific. Don't say "something went wrong." Say: "Step 3 failed because [X]. I tried [Y] and [Z]. I need [specific thing] from you to continue."
- **Include**: what was attempted, what failed, what you think the problem is, what you need from the user

### Abort — stop the current task

- **When**: fundamental impossibility, or continuing would make things worse (e.g., a partially failed destructive action)
- **How**: explain what was accomplished, what wasn't, and why continuing isn't viable. Suggest alternatives if any exist.
- **Note**: aborting a step is not aborting the task. You can abort a failed step and pivot to a different path for the broader task.

**Escalation ladder**: retry → pivot → escalate → abort. If a recovery action itself fails, move one level up. Don't try to recover from the recovery — one level of meta is enough.

## Recovery Execution

When executing a recovery:

1. **State what you're doing and why**: "Step 3 failed due to [X]. Pivoting to [Y approach]."
2. **Update the plan**: if a decomposition/composition plan exists, update it to reflect the change. Don't continue with a silently modified plan.
3. **Verify after recovery**: the immediate problem is fixed — but did the recovery produce correct output for downstream steps?
4. **Don't loop**: if the recovery itself fails, escalate one level. Don't recurse.

## Integration with Decomposition Plans

When a plan (from `skills/task-decomposition/SKILL.md` or `skills/task-composition/SKILL.md`) has a failing step:

1. **Identify** which step failed (by step number/name from the plan)
2. **Classify** the failure using the framework above
3. **Assess downstream impact**: does this failure invalidate other steps? Check dependency annotations (`Depends on: step N`)
4. **Choose recovery** for the failed step
5. **Update the plan** if recovery changes it (new steps, reordered steps, removed steps) — keep the same format:

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

6. **Continue from the recovered step**, not from the beginning

This is where structured plans pay off most — you can surgically address the failure without losing context on the broader task.

## Anti-Patterns

Explicitly avoid these:

| Anti-pattern | What it looks like | What to do instead |
|---|---|---|
| **Blind retry loop** | Same command, same error, three times in a row | Change something between retries. After 2 failures, pivot. |
| **Silent failure absorption** | Error happens, agent continues as if it didn't | Always address a failure before moving on. |
| **Scope creep in recovery** | "Fixing" the failure turns into installing 3 tools and reconfiguring the environment | That's an escalation to the user, not a recovery. |
| **Apologize-and-quit** | "Sorry, I can't do that." | Diagnose, explain why, suggest alternatives. |
| **Blame the user** | "Your instructions were ambiguous" | Frame as clarification need: "I want to make sure I understand — did you mean X or Y?" |
| **Recursive recovery** | Recovery fails, agent tries to recover from the recovery | Escalate one level. One meta-level is enough. |

## Failure Log

Maintain `{baseDir}/failure-log.md` to record failures and resolutions across sessions. Consult it early in diagnosis — recognized patterns should skip straight to the working recovery.

The log feeds back into other skills:
- **Self-model**: recurring capability gaps → update `skills/self-model/capability-gaps.md`
- **Env-model**: environmental learnings (e.g., "network calls fail in sandbox") → update `skills/env-model/environment.md`
- **Skill acquisition**: repeated capability gaps → flag as acquisition candidates (`[SKILL_ACQUISITION_INTAKE_PATH]`)

Keep the log under ~30-40 entries. Prune resolved issues and outdated configurations. Retain durable lessons.
````

### AGENTS-snippet.md

````markdown
- **Failure Recovery**: When a tool call errors, output is unexpected, you catch yourself retrying the same approach, or the user corrects your output → read `skills/failure-recovery/SKILL.md` and follow the recovery framework.
````

