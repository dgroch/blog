---
layout: post
title: "Failure Recovery: real agents need more than retries"
date: 2026-03-16
description: "Robust systems don’t just fail less. They fail with diagnosis, adaptation, and memory."
---

Blind retries are not resilience. They are just faster repetition of the same mistake.

Failure Recovery in OpenClaw starts from that premise and builds a much healthier response to things going wrong.

The skill first expands what counts as failure. Not only explicit tool errors, but silent failures, behavioural failures, and cascading failures. A command that exits zero but produces empty output is a failure. A user saying “that’s not what I meant” is a failure. A plan that completes step by step but ends in an incoherent result is a failure. That broader definition matters because agents are otherwise too willing to treat tool success as task success.

Then comes diagnosis. The skill classifies failures into transient failure, environmental constraint, input or data problem, capability gap, logic or approach error, goal misunderstanding, and fundamental impossibility. The order is practical: check common causes first, and check the `failure-log.md` to see if the pattern is already known.

That log file is more important than it looks. Without it, every failure is local. With it, a system can accumulate operational memory about what tends to go wrong and what tends to fix it. The template is plain: context, failure type, symptoms, diagnosis, recovery, outcome, lesson. That turns recovery into something that can actually improve the next attempt.

The recovery actions are simple and strong: retry, pivot, escalate, abort.

Retry is allowed only for genuinely transient cases, and something must change between attempts. Pivot means choose a different approach to the same goal, consulting the self-model for alternatives and possibly using Task Composition to build a new path. Escalate means ask the user for something specific. Abort means stop because the task is impossible or continuing would make things worse.

The escalation ladder is the right kind of strict: retry, then pivot, then escalate, then abort. Don’t keep recovering from the recovery. One meta-level is enough.

This is where the suite starts to feel like practical cognition rather than a bag of prompts. Failure Recovery does not live alone. It reads from Self-Model to distinguish capability gaps from environment problems. It can trigger Task Decomposition when the plan itself is wrong. It works with the shared plan format so a failed step can become step `3b` without rewriting everything from scratch. It updates understanding rather than just apologizing.

That last part is the heart of it. Most agent failure modes are bad not because they happen, but because the system learns almost nothing from them.

Suppose an agent tries a browser-based approach, hits a permissions problem, then keeps nudging the same path because browser automation was the first plan it formed. A system using this skill should instead classify the issue as environmental, consult the env-model, pivot to a direct API call or CLI tool if one exists, update the plan, and log the lesson. That is qualitatively different behaviour.

The anti-pattern list reads like a short obituary for current agent robustness: blind retry loops, silent failure absorption, scope creep in recovery, apologize-and-quit, blaming the user, recursive recovery. If you eliminate even half of those, the system stops feeling magical and flaky and starts feeling like a serious operator.

There is also a deeper metacognitive point here. Intelligence is not just about succeeding on the first pass. It is about preserving coherence when the first pass breaks. Diagnosis, adaptation, and explicit learning are what make failure survivable.

That is why robust systems do not merely fail less. They fail better. And failing better is one of the clearest signs that a system is reasoning about its own reasoning rather than just producing another confident guess.
