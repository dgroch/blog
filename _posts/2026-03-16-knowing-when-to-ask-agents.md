---
layout: post
title: "Knowing when to ask: the boundary between autonomy and good judgment"
date: 2026-03-16
description: "A useful agent is neither timid nor reckless. It knows when interruption is the intelligent move."
---

People often talk about autonomous agents as if asking the user a question is evidence that autonomy failed.

That is backwards.

The real failure is asking at the wrong times, or not asking when the consequences clearly demand it.

Knowing When to Ask in OpenClaw treats this as a judgment problem rather than a personality trait. The goal is not to make agents cautious. It is to make them interrupt only when interruption adds value.

The skill divides the world cleanly. Act autonomously when the task is clear, reversible, familiar, and covered by explicit instructions. Ask when the action is irreversible, the confidence on a key assumption is only moderate or low, the instructions are materially ambiguous, the task depends on user preferences, or the consequences of being wrong are significant.

That sounds obvious, but a lot of agent behaviour is broken precisely because those conditions are not represented explicitly. One class of system asks about everything and drains the user’s attention with implementation details they do not care about. Another class asks about nothing and ploughs into decisions that were never really delegated.

The cost-benefit framework in the skill is the right way to think about it. Every question imposes latency, attention tax, and friction. Every unasked question risks rework, wrong outcomes, and irreversible damage. Ask when the expected cost of guessing wrong exceeds the cost of asking.

That is not weakness. That is calibration under consequence.

The “how to ask well” section is especially good because it treats question quality as part of the skill. Lead with your best guess. Batch questions. Be specific about the gap. Explain briefly why you need the answer. Provide an escape hatch like “if you don’t care, I’ll pick the best option.” This is exactly what competent humans do.

There’s a nice contextual nuance too. The ask threshold should be lower in unfamiliar or socially risky settings like group chats, first interactions, or anything involving other people. It should be higher in trusted main-session contexts where user preferences are known and the task is routine. In other words, the right level of autonomy is not universal. It is relational and situational.

The `FUTURE-HOOK.md` makes the architecture even more interesting. It sketches a future `before_tool_call` or `tool:pre` gate that would catch high-risk actions like messaging, destructive file operations, publishing, deployment, force pushes, and dangerous bash patterns such as `rm -rf`, `curl | sh`, or `DELETE FROM` without `WHERE`. That document matters because it makes an important point: some judgment failures should not rely entirely on the agent remembering to be careful.

The proposed gate is rule-based, fast, and complementary to hard exec approvals. Exec approvals answer can this tool run at all. This gate would answer should the agent ask about the specifics before proceeding. That is a much subtler and more useful line.

Imagine an agent drafting an outbound message. The content is agent-composed rather than verbatim user text. The tool is technically available. Nothing in a generic permission model stops it. But good judgment may still require a confirmation question because the action is socially irreversible. That is exactly the class of failure this skill is trying to catch.

The larger thesis is that autonomy is not maximal action without interruption. It is the ability to proceed decisively in the large space of low-risk, reversible, well-understood work while recognizing the smaller set of moments where a pause is the more intelligent move.

A useful agent is not the one that never asks. It is the one that makes being asked feel justified.
