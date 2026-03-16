---
layout: post
title: "Task Decomposition: planning without pretending the plan is the work"
date: 2026-03-16
description: "Good agent planning breaks goals into executable steps without drifting into fake productivity."
---

There are two common failures in agent planning.

The first is not planning at all. The model jumps from request to action, misses dependencies, and discovers halfway through that step three required information from step one.

The second is more insidious: elaborate planning theatre. The agent produces a crisp multi-section plan that feels intelligent, then fails because the plan was never checked against actual tools, actual constraints, or the cost of being wrong.

Task Decomposition in OpenClaw is trying to sit exactly between those failures.

Its core claim is simple: plans are useful when they make execution more feasible, not when they merely look organized.

The `SKILL.md` starts with a triage heuristic rather than a universal mandate. Decompose when the goal is ambiguous, multi-step, dependent, risky, or likely to take more than a handful of tool calls. Skip it for obvious one-shot work. That restraint matters. A lot of reasoning systems quietly become slower versions of instinct because they insist on ceremonial planning for trivial tasks.

Then the skill introduces adaptive depth: light, standard, and deep. Light decomposition stays implicit. Standard decomposition writes a concrete plan and executes. Deep decomposition includes explicit feasibility checks against the self-model and env-model and is meant for costly mistakes.

The plan template itself is unusually practical. It asks for a concrete goal, a done condition, explicit assumptions, a chosen depth, numbered steps in verb-plus-object form, dependencies, a tool or skill for each step, a risk note, and a flags section for capability gaps, environmental constraints, missing user input, and irreversible actions. This is not documentation prose. It is a compact execution model.

The best part is the feasibility check. For each step, the agent is told to ask two questions: can I do this, and does the environment support it? If `capabilities.md` or `environment.md` don’t exist, the skill says to note that feasibility is unchecked. That one instruction cuts against one of the worst habits in agent systems: smuggling unverified assumptions into plans as if they were stable facts.

That is why the title tension matters. Planning is not the work. The actual work of decomposition is not turning one goal into six bullet points. It is turning one goal into six bullet points that can survive contact with a real environment.

Suppose the user asks an agent to create a blog series from a repo, match an existing house style, and package everything for review. A naive planner might write: research, draft, edit, deliver. That’s neat and almost useless. A disciplined decomposition would say: read voice anchors, read repo overview, map post structure to source files, draft hub post, draft each skill post from its minimum sources, write a manifest, verify filenames and frontmatter, then stop short of publication. Each step has dependencies. Each maps to a file or tool action. That is the difference between choreography and execution.

The re-planning section is just as important. The skill explicitly says not to throw away the whole plan when one step fails. Re-plan from the failed step forward unless the goal itself changed. That is exactly what good operators do and exactly what many agents don’t.

The shared plan format also matters beyond this one skill. Task Composition and Failure Recovery are designed to interoperate with it. A composed workflow can later be decomposed for execution, and a failed plan can be surgically updated with a step `3b` instead of rewritten as if nothing happened.

The failure mode this corrects is the language-model instinct to mistake narrative coherence for operational coherence. A model is very good at producing a sequence that sounds like a plan. It is much worse at respecting hidden dependencies unless something forces them into the open.

Planning is not intelligence. But intelligence without planning becomes brittle surprisingly quickly. The trick is to make plans answerable to reality.
