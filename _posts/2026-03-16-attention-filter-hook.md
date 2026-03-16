---
layout: post
title: "Attention Filter: what the agent sees shapes what it can think"
date: 2026-03-16
description: "Relevance-aware injection is not garnish. It is part of the cognitive architecture."
---

Before an agent reasons, something has already decided what is in front of it.

That sounds trivial. It isn’t.

What the agent sees, in what order, with what framing, and with what lightweight cues about urgency or continuity will shape the reasoning that follows. Attention is not merely an internal property of the model. It is also an architectural choice.

That is what the Attention Filter hook makes explicit.

The `HOOK.md` describes it as an always-on layer that does not remove core context but adds targeted focusing signals. On `agent:bootstrap`, it performs relevance-aware context injection. On `message:preprocessed`, it classifies incoming signals and helps preserve focus when new requests arrive mid-task.

The handler implementation is refreshingly concrete. It uses simple regex patterns for urgency words like urgent, asap, immediately, critical, broken, or outage. It has a heartbeat pattern so it can avoid wasting attention on routine polls. It maps common user-message patterns to relevant artifact pointers: capability questions point toward `skills/self-model/capabilities.md`; failure language points toward the failure log and env-model; planning language points toward the task-decomposition skill.

The file checks are cheap. `fs.existsSync()`. Small reads from `.active-plan.md`. No LLM calls in the hot path. A cap of two artifact pointers per turn. This is important because hooks that fire every turn have to behave like infrastructure, not research demos.

The active-task handling is the core of it. If `.active-plan.md` exists, the hook reads a small head slice, extracts the goal and maybe the current in-progress step, and injects a directive like ACTIVE PLAN: resume this work unless the user redirects. If a checkpoint exists instead, it nudges the agent to consult `.context-checkpoint.md`. During message preprocessing, if active work exists and the incoming message is not urgent, it can attach a note saying the system was working on X and should decide whether to switch or finish first.

That sounds almost too simple. But simplicity is the point. Attention scaffolding should be cheap, legible, and reliable.

A lot of current agent systems dump everything into the prompt and hope the model sorts salience out internally. Sometimes it does. Sometimes it doesn’t. Attention Filter is a direct rejection of that passive design. It says relevance can be shaped mechanically before reasoning begins.

The relationship to the companion Attention Awareness skill is exactly right. The hook shapes what the agent notices. The skill teaches it how to reason about what matters, what is background, what is noise, and when to switch tasks. One is automated scaffolding. The other is conscious discipline.

A concrete example helps. Suppose the agent has an active plan to write a report and receives a new message saying the staging site is down. The hook tags the message urgent. That alone changes the attentional landscape before the model does any deeper reasoning. If instead the new message is just ambient chatter or a low-priority question, the hook can preserve continuity by reminding the agent that current work exists and a switch should be deliberate.

The larger thesis is simple: attention allocation is one of the hidden levers of cognition. If you want more reliable reasoning, don’t just improve the reasoner. Improve the layer that decides what deserves attention in the first place.

What the agent sees shapes what it can think. Treating that as architecture rather than accident is one of the smartest ideas in this repo.
