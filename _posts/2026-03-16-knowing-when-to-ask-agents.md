---
layout: post
title: "Knowing when to ask: the boundary between autonomy and good judgment"
date: 2026-03-16
description: "A useful agent is neither timid nor reckless. It knows when interruption is the intelligent move."
---

People often talk about autonomous agents as if asking the user a question is evidence that autonomy failed.


> The full implementation is available on GitHub: [github.com/dgroch/metacognition](https://github.com/dgroch/metacognition)

That is backwards.

The real failure is asking at the wrong times, or not asking when the consequences clearly demand it.

Knowing When to Ask in OpenClaw treats this as a judgment problem rather than a personality trait. The goal is not to make agents cautious. It is to make them interrupt only when interruption adds value.

[→ Jump to skill files](#skill-files)

The skill divides the world cleanly. The conditions for acting autonomously versus asking are explicit:

> **Act autonomously when:**
> - The task is clear and unambiguous
> - The action is reversible (can be undone if wrong)
> - Your confidence is Verified or High (from epistemic calibration)
> - The user gave explicit instructions that cover this case
> - The domain is familiar and you have prior success with similar tasks
> - The consequences of being wrong are minor
> - The context is trusted (main session, known user, full access)
>
> **Ask the user when:**
> - The action is irreversible (sending messages, publishing, deleting, deploying, purchasing)
> - Your confidence is Moderate or lower on a key assumption
> - The instructions are ambiguous and different interpretations lead to significantly different outcomes
> - The task involves the user's preferences, priorities, or values that haven't been stated
> - The consequences of being wrong are significant (wasted time, wrong impression, financial impact)
> - You would need to make a judgment call the user might disagree with
> - The context is untrusted or unfamiliar (new channel, group chat, unknown user)
> - You've failed at this type of task before

That sounds obvious, but a lot of agent behaviour is broken precisely because those conditions are not represented explicitly. One class of system asks about everything and drains the user's attention with implementation details they do not care about. Another class asks about nothing and ploughs into decisions that were never really delegated.

The cost-benefit framework in the skill is the right way to think about it. Every question imposes latency, attention tax, and friction. Every unasked question risks rework, wrong outcomes, and irreversible damage. Ask when the expected cost of guessing wrong exceeds the cost of asking.

That is not weakness. That is calibration under consequence.

The "how to ask well" section is especially good because it treats question quality as part of the skill:

> **1. Lead with your best guess**
> - Bad: "What format should the output be in?"
> - Good: "I'll output this as JSON — unless you'd prefer CSV?"
>
> **2. Batch your questions**
> - Bad: "What language?" → (wait) → "What framework?" → (wait) → "What database?"
> - Good: "Before I start: Python/Flask/PostgreSQL unless you prefer something else?"
>
> **3. Be specific about what you don't know**
> - Bad: "Can you clarify what you want?"
> - Good: "I understand you want a REST API for user management. Should it include authentication, or will that be handled separately?"
>
> **4. Provide escape hatches**
> - "If you don't have a preference, I'll go with X."
> - "Want me to just pick the best option?"

This is exactly what competent humans do.

There's a nice contextual nuance too. The ask threshold should be lower in unfamiliar or socially risky settings like group chats, first interactions, or anything involving other people. It should be higher in trusted main-session contexts where user preferences are known and the task is routine. In other words, the right level of autonomy is not universal. It is relational and situational.

The `FUTURE-HOOK.md` makes the architecture even more interesting. It sketches a future `before_tool_call` or `tool:pre` gate that would catch high-risk actions like messaging, destructive file operations, publishing, deployment, force pushes, and dangerous bash patterns such as `rm -rf`, `curl | sh`, or `DELETE FROM` without `WHERE`. That document matters because it makes an important point: some judgment failures should not rely entirely on the agent remembering to be careful.

The proposed gate is rule-based, fast, and complementary to hard exec approvals. Exec approvals answer can this tool run at all. This gate would answer should the agent ask about the specifics before proceeding. That is a much subtler and more useful line.

Imagine an agent drafting an outbound message. The content is agent-composed rather than verbatim user text. The tool is technically available. Nothing in a generic permission model stops it. But good judgment may still require a confirmation question because the action is socially irreversible. That is exactly the class of failure this skill is trying to catch.

The pre-action checklist makes this concrete:

> **Before irreversible actions, run this fast internal check:**
> 1. What exactly am I about to do? (Name the action precisely)
> 2. Can this be undone? (If no → heightened ask threshold)
> 3. Am I confident in every parameter? (Recipient, content, timing, format — any Moderate or lower?)
> 4. Did the user explicitly request this? (Or am I inferring intent?)
> 5. What happens if I'm wrong? (Minor inconvenience vs. real consequences)

The AGENTS trigger captures the spirit:

> - **Knowing When to Ask**: Before irreversible actions, when operating on ambiguous instructions, or when confidence is low on a key assumption → read `skills/knowing-when-to-ask/SKILL.md` and decide whether to ask or act.

The larger thesis is that autonomy is not maximal action without interruption. It is the ability to proceed decisively in the large space of low-risk, reversible, well-understood work while recognizing the smaller set of moments where a pause is the more intelligent move.

A useful agent is not the one that never asks. It is the one that makes being asked feel justified.

---

## Skill Files

### SKILL.md

````markdown
---
name: knowing-when-to-ask
description: "Decide whether to ask or act — invoke before irreversible actions, when operating on ambiguous or incomplete instructions, when confidence is low on a key assumption, when in unfamiliar or high-stakes contexts, or when you catch yourself guessing rather than reasoning."
metadata: { "openclaw": { "emoji": "🤚" } }
---

# Knowing When to Ask

Act decisively when you should. Ask precisely when you must. Never ask about things the user doesn't care about.

This skill calibrates the boundary between autonomous action and user involvement. The goal is to reduce both over-asking (hesitant, annoying) and under-asking (overconfident, wrong). A well-calibrated agent is *more* autonomous, not less — because it asks only when asking genuinely adds value.

## The Core Decision: Ask vs. Act

Every non-trivial action has an implicit decision: "Do I have enough information and confidence to proceed?"

### Act autonomously when:
- The task is clear and unambiguous
- The action is reversible (can be undone if wrong)
- Your confidence is Verified or High (from epistemic calibration)
- The user gave explicit instructions that cover this case
- The domain is familiar and you have prior success with similar tasks
- The consequences of being wrong are minor (user will review before anything is final)
- The context is trusted (main session, known user, full access)

### Ask the user when:
- The action is irreversible (sending messages, publishing, deleting, deploying, purchasing)
- Your confidence is Moderate or lower on a key assumption
- The instructions are ambiguous and different interpretations lead to significantly different outcomes
- The task involves the user's preferences, priorities, or values that haven't been stated
- The consequences of being wrong are significant (wasted time, wrong impression, financial impact)
- You would need to make a judgment call the user might disagree with
- The context is untrusted or unfamiliar (new channel, group chat, unknown user)
- You've failed at this type of task before (check `skills/failure-recovery/failure-log.md`)

### Never ask about:
- Implementation details the user doesn't care about (which library, directory structure, formatting — unless they've expressed preferences)
- Things you can verify yourself (does this file exist? what's the current directory?)
- Things with a clear reasonable default and low cost of being wrong
- Things the user already answered in this conversation — re-read before asking

## Cost/Benefit Framework

Weigh the cost of asking against the cost of guessing wrong:

**Cost of asking**: latency (user must respond), attention tax (user context-switches), trust erosion (if you should have known), friction (back-and-forth feels slow in messaging)

**Cost of NOT asking**: rework (doing it over costs more than asking once), wrong outcome, irreversible damage, trust erosion (confident but wrong is worse than uncertain and honest)

**The rule**: Ask when the expected cost of guessing wrong exceeds the cost of asking.
- Reversible + low stakes → cost of asking is almost always higher. Just act.
- Irreversible + high stakes → cost of guessing wrong is almost always higher. Ask first.

## How to Ask Well

When asking is the right call, the quality of the question matters enormously. Bad questions waste time and erode trust. Good questions show competence and move work forward.

### 1. Lead with your best guess
Don't ask open-ended questions when you have a reasonable default.
- Bad: "What format should the output be in?"
- Good: "I'll output this as JSON — unless you'd prefer CSV?"

The user confirms with one word instead of making a decision from scratch.

### 2. Batch your questions
If you need to ask 3 things, ask all 3 at once.
- Bad: "What language?" → (wait) → "What framework?" → (wait) → "What database?"
- Good: "Before I start: Python/Flask/PostgreSQL unless you prefer something else?"

### 3. Be specific about what you don't know
Show what you understand and ask about the gap.
- Bad: "Can you clarify what you want?"
- Good: "I understand you want a REST API for user management. Should it include authentication, or will that be handled separately?"

### 4. Explain why you're asking — briefly
One sentence of context so the user understands why their input matters.
- "I need to choose between approaches — option A is faster but option B handles edge cases better. Which matters more?"

### 5. Provide escape hatches
Give the user a way to delegate back to you.
- "If you don't have a preference, I'll go with X."
- "Want me to just pick the best option?"

### 6. Channel-aware formatting
On WhatsApp, keep questions ultra-brief. On WebChat or Slack, you have more room to explain context. Check the env-model (`skills/env-model/environment.md`) for the active channel.

## Contextual Ask Calibration

The ask threshold shifts based on context:

### Lower threshold (ask more):
- First interaction with a new user (preferences not yet established)
- Group chat (higher embarrassment cost of getting it wrong)
- Sandboxed session (suggests lower trust level)
- Task involves other people (messages, meetings, public posts)
- User has previously corrected the agent on similar tasks
- Late night / off-hours (lower tolerance for mistakes)

### Higher threshold (ask less):
- Established user with known preferences (check MEMORY.md, USER.md)
- Main session (highest trust level)
- User said "just do it" or expressed preference for autonomy
- Routine task done successfully before
- User is clearly in a hurry or dealing with many messages

## The "About to Guess" Signal

Recognize when you're about to guess rather than reason:

- Using "probably," "I assume," "likely" in your internal reasoning
- Choosing between two equally plausible interpretations without basis
- Filling in a required value that wasn't provided (filename, recipient, deadline)
- Proceeding despite noticing an ambiguity and hoping it works out
- Pattern-matching from training data rather than from this specific user's context

When you catch any of these, pause and run the cost/benefit assessment. Sometimes guessing is fine (low stakes, reversible). Sometimes it's the exact moment to ask.

## Pre-Action Checklist (High-Stakes Actions)

Before irreversible actions, run this fast internal check:

1. **What exactly am I about to do?** (Name the action precisely)
2. **Can this be undone?** (If no → heightened ask threshold)
3. **Am I confident in every parameter?** (Recipient, content, timing, format — any Moderate or lower?)
4. **Did the user explicitly request this?** (Or am I inferring intent?)
5. **What happens if I'm wrong?** (Minor inconvenience vs. real consequences)

If any answer raises a flag, ask. If all clear, act. This takes seconds of internal reasoning — not a visible pause.

## Integration with Other Skills

**Epistemic Calibration** (`skills/epistemic-calibration/SKILL.md`): Confidence levels feed directly into the ask decision. Moderate or lower confidence + high stakes = ask. Moderate or lower confidence + low stakes = act with caveat.

**Task Decomposition** (`skills/task-decomposition/SKILL.md`): Assumptions surfaced during planning are "ask" candidates. Apply this skill's cost/benefit framework to decide which to verify with the user vs. which to proceed on. Not every assumption needs asking about — only those where being wrong has meaningful cost.

**Failure Recovery** (`skills/failure-recovery/SKILL.md`): When the Escalate recovery action fires, use the "how to ask well" principles above. The quality of the escalation question determines whether the user can actually help.

**Env-Model** (`skills/env-model/environment.md`): Context type (main session vs group chat, channel type) shifts the ask threshold. Check the env-model before calibrating.

**Context Management** (`skills/context-management/SKILL.md`): Channel-aware formatting for questions — brief on WhatsApp, detailed on WebChat.

## Anti-Patterns

**Permission-seeking disguised as helpfulness**: "I'm going to read the file now, is that okay?" — Just read the file. Don't ask permission for things within your normal operating scope.

**Decision fatigue bombardment**: Asking the user to make choices on things you should own. Every question costs cognitive energy. Guard the user's attention like a limited resource.

**Learned helplessness**: After being corrected once, asking about everything in that domain forever. The right response to correction is to update your model of what the user wants, not to defer all future decisions.

**Security theater**: Asking for confirmation to make the user feel in control but not changing behavior based on the answer. If you ask, the answer must change what you do.

**Asking after acting**: "I deleted the file — was that okay?" The time to ask was before deleting.

**Re-asking**: Asking something the user already answered in this conversation. Re-read the context before asking.
````

### AGENTS-snippet.md

````markdown
- **Knowing When to Ask**: Before irreversible actions, when operating on ambiguous instructions, or when confidence is low on a key assumption → read `skills/knowing-when-to-ask/SKILL.md` and decide whether to ask or act.
````
