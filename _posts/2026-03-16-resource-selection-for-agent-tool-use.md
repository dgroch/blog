---
layout: post
title: "Resource Selection: the best tool is rarely the first one the model remembers"
date: 2026-03-16
description: "Tool choice is not a convenience layer. It is one of the agent’s core cognitive decisions."
---

Many agent failures are not failures of reasoning in the abstract. They are failures of choosing how to act.

The model knows the goal. It even has the tools. But it reaches for the wrong one because it is familiar, overgeneralized, or narratively convenient.

This is what Resource Selection is designed to correct.

The first line of the skill says it well: pause between “I know what to do” and “I’m doing it with X.” That pause is small, but it is doing serious cognitive work.

A lot of current agent design quietly assumes tool use is a mechanical afterthought. Once the model understands the task, surely the right tool will follow. In practice the opposite is often true. The task is underspecified until you know which resource you’re committing to, because different tools have different costs, failure modes, output shapes, and environmental requirements.

The selection framework starts by forcing the sub-task to be framed abstractly. Not “I need to write a Python script to parse this JSON,” but “I need to extract the users array and get the email addresses.” That is such a small instruction and such a good one. Tool-shaped thinking is one of the fastest ways an agent narrows its own options before it has even reasoned about them.

Then it generates a few genuine candidates by consulting `capabilities.md` and `environment.md`. Core tools like bash, read, write, and edit sit alongside installed CLIs, browser automation, sub-agents, connected nodes, external APIs, and the possibility of just asking the user. The presence of “ask the user” in a tool-selection framework is quietly important. Sometimes the best resource is not another layer of automation. It is a ten-second clarification.

The evaluation dimensions are simple and exactly right: fitness, efficiency, reliability, environmental fit, reversibility, and simplicity. No scoring rubric, no fake precision. Just comparative reasoning.

The heuristics file is even more useful because it attacks common biases directly. Bash versus Python. CLI tool versus custom code. Inline work versus sub-agent. Local versus node. Do it yourself versus ask the user. The rule of thumb “if you can express it as a pipeline of existing commands in under three lines, use bash” is the kind of practical guidance that saves real systems from bloated, failure-prone action.

Language models have strong familiarity bias. They often want to write code because code is their native performance medium. But many operational tasks are better solved by `jq`, `curl`, `grep`, `sort`, `uniq`, `git`, or `ffmpeg`. A model without structured resource selection will happily produce forty lines of Python where a one-liner would be faster, cheaper, and more reliable.

That is not a cosmetic inefficiency. It affects failure rates, latency, token use, and trust.

The integration with Failure Recovery is another good touch. If an approach fails, pivot by re-selecting from the remaining candidates with the failed option removed. That sounds obvious, but it’s not how agents naturally behave. They often keep reworking the same doomed path because sunk cost feels like persistence.

Suppose the agent needs to inspect structured JSON in a repo, extract two fields, and write a manifest. The default language-model reflex is often to write a script. Resource Selection asks a better question: is `jq` installed? Can bash do this in one line? Is the output small enough for inline work? Does a sub-agent help, or would that be needless overhead? If the environment lacks `jq`, does Python become the right fallback? The point is not that one tool is always superior. The point is that the choice should be deliberate.

Tool selection is a self-monitoring problem. The agent is not just trying to solve the task. It is deciding which of its own action pathways best fits the task under current conditions. That is one of the places where capable and intelligent start to diverge.
