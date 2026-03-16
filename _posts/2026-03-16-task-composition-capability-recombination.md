---
layout: post
title: "Task Composition: when intelligence looks like recombination"
date: 2026-03-16
description: "Useful agents don’t just follow plans. They discover new workflows by combining what they already have."
---

One mark of intelligence is not merely breaking down a known problem. It is seeing a solution shape that wasn’t explicitly handed to you.

In software terms, that often means recombination.

Task Composition in OpenClaw asks a different question from decomposition. Instead of how do I break this goal into steps, it asks what new outcomes become possible if I combine the capabilities I already have.

The `SKILL.md` makes this explicit. Start with a capability survey. Self-Model is a hard dependency. If the capability inventory doesn’t exist, build it before composing. Read the environment model too, because a clever workflow that depends on absent conditions is just another hallucination. Then abstract the available components into categories like producers, transformers, analyzers, actors, and routers. That framing is not a type system. It’s a way of noticing how one capability’s output might feed another’s input.

The example in the skill is telling: a summary generator can feed a TTS tool, giving you a document-to-audio briefing pipeline. That is a trivial composition in one sense, but it captures the deeper idea. Intelligence often looks less like inventing a brand-new power than like seeing a useful chain hidden in plain sight.

There are guardrails here I appreciate. Candidate generation is capped at two to four workflows. Weak ideas should be killed before they reach the user. Each candidate is evaluated on feasibility, value, risk, and effort, and if any step depends on a missing capability or unavailable resource, the workflow dies. That ruthlessness is necessary because agents are very good at producing optionality theatre.

The shared plan format with Task Decomposition is another smart move. Composition adds two fields, `Composed from` and `Opportunity`, then reuses the rest of the step structure. That means the system can move cleanly from invention to execution without switching mental models.

The skill also introduces a `composition-log.md` for two reasons: don’t keep re-proposing workflows the user rejected, and notice when a repeatedly useful workflow should become its own skill. That’s a subtle but powerful idea. Composition is not just for solving the immediate problem. It is also a mechanism for discovering stable abstractions worth operationalizing.

Imagine an agent working inside a workspace that has file read and write tools, web fetch, PDF analysis, and TTS. A decomposition-first system waits for an instruction like make me an audio briefing of this report. A composition-capable system, after summarizing the report, can recognize that summary plus TTS plus message delivery form a valuable adjacent workflow. If the context justifies it, it can propose the pipeline. Not as hype. As a feasible extension grounded in available components.

That is not the same as being creative in the vague consumer-AI sense. It is constrained recombination. The novelty is bounded by actual capabilities, actual interfaces, and actual value.

The metacognitive thesis here is that higher-order usefulness often emerges from structured recombination, not just better direct instruction following. An agent that can only do what was already named is capable. An agent that can identify adjacent, feasible workflows without losing discipline is starting to show something closer to judgment.
