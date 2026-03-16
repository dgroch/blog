---
layout: post
title: "Attention Awareness: intelligence is also about ignoring the right things"
date: 2026-03-16
description: "Better reasoning is not just more reasoning. It is better allocation of cognitive focus."
---

There is a naïve theory of intelligence that says more attention is always better.

Read more. Track more. Consider more. Invoke more reasoning steps. Keep more context alive.

That theory fails quickly in real systems. Better cognition is not just about adding more thought. It is about allocating thought well.

That is what Attention Awareness is for.

The skill begins with a simple claim: not everything in context matters right now. The agent should distinguish active information, background information, dormant information, and noise. It should reason deeply with what the current step requires and let the rest fall back. This is not a literal tagging workflow. It is a mental model for preserving signal.

That matters because agents often fail by over-processing. They drag old diagnostics, irrelevant side conversations, superseded assumptions, and verbose tool output forward into later steps as if all context were equally important. It isn’t. Some information deserves attention. Some deserves memory. Some deserves neither.

The context-switch logic is also good. When a new request arrives, assess priority, the state of current work, whether the task can be paused cleanly, and whether the new request actually supersedes the old one. That sounds like project management, but it is really attentional triage. Urgent messages may justify switching. Ambient ones should not. Explicit user redirection overrides the queue. That is not profound. It is just the difference between a system that keeps a thread and one that thrashes.

The skill also contains a useful warning against over-metacognition. Simple tasks should skip the full suite. Medium tasks should use one or two relevant skills. Complex tasks can use more, but in sequence and only as needed. If you are making three or more tool calls of pure thinking without producing work, you are overdoing it. That line alone could rescue a lot of agents from self-inflicted paralysis.

The companion hook shows why this belongs in architecture, not just habit. `attention-filter` runs on `agent:bootstrap` and `message:preprocessed`. It looks for active work in files like `.active-plan.md` or `.context-checkpoint.md`, classifies incoming messages with cheap urgency patterns, skips heartbeat clutter, and injects notes like you were working on X, decide whether to switch or finish first. It also adds lightweight pointers to relevant artifacts such as the capability inventory, failure log, or task decomposition skill based on simple keyword matches.

No deep semantic engine. No mystical attention mechanism. Just system-level scaffolding that shapes what the agent sees as salient.

That is an important architectural point. Attention is not only an internal property of the model. It can be supported, directed, and constrained by the software around the model.

A concrete example makes this clearer. Suppose the agent is halfway through a multi-step plan and a new message arrives that contains the words urgent and broken. The hook marks the message as urgent. The skill then treats that as a possible switch, but not a guaranteed one; it still asks whether the current work is at a safe breakpoint. If the new message is ambient, the note may simply remind the agent not to abandon the in-progress task. This is what sane focus looks like under interruption.

The broader thesis is that intelligence is partly subtractive. It is not only the ability to think about many things. It is the ability to not think about the wrong things at the wrong time.

Agents that cannot filter noise end up spending cognition on residue. Agents that over-invoke metacognition disappear into commentary about commentary. Attention awareness is the discipline that keeps reasoning pointed at the task rather than at the spectacle of reasoning.
