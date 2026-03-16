---
layout: post
title: "Self-Model: the agent should know what it can actually do"
date: 2026-03-16
description: "Capability inventory is the difference between useful autonomy and polished delusion."
---

A surprising amount of agent failure begins with a simple lie: the system behaves as if having a language model means having a capability.

It doesn’t.

An agent can describe a browser flow without having browser access. It can propose a git operation without `git` installed. It can speak confidently about connected nodes, external APIs, image generation, or local binaries that are absent, stale, or broken. This is one of the most dangerous default behaviours in LLM systems because it looks like competence right until execution time.

The Self-Model skill in OpenClaw exists to make that bluff harder.

Its job is straightforward: build and maintain a structured capability inventory of what the agent can do right now. Not in theory. Not in training data. Not on some other machine. Right now, in this environment.

The `SKILL.md` is admirably literal about how to do it. Survey installed skills by listing `skills/*/SKILL.md`. Extract tools from the system prompt. Probe for connected nodes with `openclaw node list` if the CLI exists. Check for useful binaries in one pass with a loop over things like `git`, `python3`, `node`, `ffmpeg`, `jq`, `docker`, `cargo`, and `gh`. Fold in known failures from `capability-gaps.md`. Then write the result to `capabilities.md` using a template designed to be cheap to reread.

That sounds boring. It’s not. It’s the foundation of non-delusional agency.

Look at what the skill is really doing. It forces the agent to separate four things language models habitually blur together: what I can describe, what I can infer, what I can do in principle, and what I can execute in this session. Only the last one should drive action.

The AGENTS snippet that triggers Self-Model is short but sharp: use it when unsure what you can do, when planning multi-step work, or when a tool or skill fails unexpectedly. That last condition matters. A failure is often not just a local exception. It is evidence that the agent’s internal model of itself was wrong.

This is the deeper value of explicit capability inventory. It gives the system a place to update. Without that, every failure is temporary theatre. The agent says sorry, then continues reasoning from the same false assumptions.

The repo design reinforces this. Self-Model writes to disk. `capabilities.md` survives session boundaries. `capability-gaps.md` records repeated absences in a simple append-only format: what failed, why, and when. That turns embarrassment into memory. A missing binary stops being a one-turn accident and becomes part of the system’s self-knowledge.

There’s also a subtle choice here I like a lot: the skill distinguishes rebuilding from incremental refresh. If the inventory exists, don’t rescan the universe. Update changed sections, bump `last_updated`, append gaps. That mirrors how cognition should work. You do not rediscover your whole body every morning. You notice what changed.

Suppose an agent is asked to summarize a PDF and turn the summary into audio. A model without a self-model may leap straight to writing a script because code is the generic shape of possible action. An agent with a capability inventory can reason more cleanly. It may know it has a native PDF analysis tool and a TTS tool. Or it may discover it lacks PDF support but has `python3` and `ffmpeg`. Or it may learn that none of those exist and stop pretending otherwise.

That isn’t just better execution. It is a more honest form of thought.

The README describes the broader problem well: agents act on instinct, reaching for familiar tools and stating uncertain things with false confidence. Self-Model attacks both habits. If the agent has to consult a concrete inventory before making capability claims, it becomes harder to hallucinate affordances into existence.

This matters because capability hallucination is not a narrow tooling bug. It corrupts planning, confidence, user trust, and recovery. A plan built on nonexistent abilities is fake. Confidence in that plan is fake. Subsequent retries are fake. The whole stack inherits the original self-deception.

If you want more autonomous agents, this is where to start. Not with bolder prompting. With a disciplined answer to the question: what are you, concretely, capable of right now?

Metacognition begins there. Before an agent can monitor its reasoning, it has to know whether the reasoning terminates in reality.
