---
layout: post
title: "Context Guard: preserving the mind across compaction"
date: 2026-03-16
description: "Some forms of metacognition shouldn’t depend on the agent remembering to invoke them."
---

One of the hardest things about long-running agent systems is that memory loss is not binary.

Before a hard context reset, there is gradual drift. Then comes compaction, which preserves some things, compresses others, and quietly drops details the model may still need. If the agent has to remember, in that moment, to preserve its own critical state, you are already gambling.

That is why the Context Guard hook matters.

It is not another reasoning prompt. It is infrastructure.

The `HOOK.md` lays out the contract clearly. On `session:compact:before`, the hook writes a `.context-checkpoint.md` file to the workspace root. That file captures active task state, metacognitive state, key decisions, recently changed files, and next steps. On `agent:bootstrap`, it checks for a recent checkpoint, injects a concise recovery block into bootstrap files, and renames the checkpoint to `.context-checkpoint.recovered.md` so the state is not replayed forever.

This is exactly the kind of metacognitive function that should be automated. The agent is most vulnerable to forgetting what matters at the point where the context window is already under stress. Asking it to self-manage perfectly in that moment is wishful thinking.

The TypeScript handler is practical in all the right ways. It uses a ten-minute freshness window for recovery. It caps the injected recovery block at about two thousand characters to stay within a small token budget. It scans for recently modified files within the last thirty minutes, bounded by shallow recursion. It looks for `.active-plan.md`, a failure log, and model artifacts like `capabilities.md` and `environment.md` to estimate the freshness of the broader metacognitive state.

I especially like the design principle that it never throws. All operations are wrapped defensively. If artifacts are missing, sections are skipped. The hook is additive, not brittle. That is exactly how background cognition support should behave.

Look at what the checkpoint actually preserves. Not just generic session summary, but plan summary, last failure, pending assumptions, self-model freshness, env-model freshness, changed files, and next steps. Those are the things ordinary compaction summaries often miss because they are operationally important but not always conversationally prominent.

That distinction matters. A standard summary may remember the topic. Context Guard tries to remember the state of mind.

Imagine an agent halfway through a multi-step migration. It has already made one key decision about tool choice, logged a recent environment-related failure, and written two files. Compaction hits. Without a hook like this, the resumed system may know that it was “working on the migration” and almost nothing else. With Context Guard, it can recover not just the headline task but the active plan, the failure context, and the next action. That is a much more coherent restart.

This is the larger lesson of the hook layer in the suite. Some metacognitive disciplines are too important to remain voluntary habits. Preserving state across compaction is one of them.

If we want agents that operate for long enough to matter, we need to build for the moments when their memory compresses, shifts, or partially disappears. Context Guard is a practical answer to that problem. Not philosophical continuity. Operational continuity.
