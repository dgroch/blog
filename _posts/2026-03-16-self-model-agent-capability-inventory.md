---
layout: post
title: "Self-Model: the agent should know what it can actually do"
date: 2026-03-16
description: "Capability inventory is the difference between useful autonomy and polished delusion."
---

A surprising amount of agent failure begins with a simple lie: the system behaves as if having a language model means having a capability.

It doesn't.

An agent can describe a browser flow without having browser access. It can propose a git operation without `git` installed. It can speak confidently about connected nodes, external APIs, image generation, or local binaries that are absent, stale, or broken. This is one of the most dangerous default behaviours in LLM systems because it looks like competence right until execution time.

The Self-Model skill in OpenClaw exists to make that bluff harder.

[→ Jump to skill files](#skill-files)

Its job is straightforward: build and maintain a structured capability inventory of what the agent can do right now. Not in theory. Not in training data. Not on some other machine. Right now, in this environment.

The `SKILL.md` is admirably literal about how to do it. Survey installed skills by listing `skills/*/SKILL.md`. Extract tools from the system prompt. Probe for connected nodes with `openclaw node list` if the CLI exists. Check for useful binaries in one pass with a loop over things like `git`, `python3`, `node`, `ffmpeg`, `jq`, `docker`, `cargo`, and `gh`. Fold in known failures from `capability-gaps.md`. Then write the result to `capabilities.md` using a template designed to be cheap to reread.

The binary probe is a single shell loop — deliberate in what it checks, cheap to run:

```bash
for cmd in git uv pip python3 node npm docker docker-compose ffmpeg jq yq curl wget gh rg fd make gcc cargo rustc go java mvn; do
  command -v "$cmd" &>/dev/null && echo "✓ $cmd: $(command -v "$cmd")"
done
```

That sounds boring. It's not. It's the foundation of non-delusional agency.

Look at what the skill is really doing. It forces the agent to separate four things language models habitually blur together: what I can describe, what I can infer, what I can do in principle, and what I can execute in this session. Only the last one should drive action.

The AGENTS snippet that triggers Self-Model is short but sharp: use it when unsure what you can do, when planning multi-step work, or when a tool or skill fails unexpectedly. That last condition matters. A failure is often not just a local exception. It is evidence that the agent's internal model of itself was wrong.

The full trigger condition reads:

> - **Self-Model**: When unsure what you can do, when planning multi-step work, or when a tool/skill fails unexpectedly → read `skills/self-model/SKILL.md` and build or consult your capability inventory.

This is the deeper value of explicit capability inventory. It gives the system a place to update. Without that, every failure is temporary theatre. The agent says sorry, then continues reasoning from the same false assumptions.

The repo design reinforces this. Self-Model writes to disk. `capabilities.md` survives session boundaries. `capability-gaps.md` records repeated absences in a simple append-only format: what failed, why, and when. That turns embarrassment into memory. A missing binary stops being a one-turn accident and becomes part of the system's self-knowledge.

New gaps are logged with a minimal structure:

```
- <what failed> | <why> | <date>
```

There's also a subtle choice here I like a lot: the skill distinguishes rebuilding from incremental refresh. If the inventory exists, don't rescan the universe. Update changed sections, bump `last_updated`, append gaps. That mirrors how cognition should work. You do not rediscover your whole body every morning. You notice what changed.

The staleness heuristic is explicit:

> **Staleness heuristic:** If the inventory timestamp is from a different session or if you have installed/removed skills or tools since the last update, treat it as stale.

Suppose an agent is asked to summarize a PDF and turn the summary into audio. A model without a self-model may leap straight to writing a script because code is the generic shape of possible action. An agent with a capability inventory can reason more cleanly. It may know it has a native PDF analysis tool and a TTS tool. Or it may discover it lacks PDF support but has `python3` and `ffmpeg`. Or it may learn that none of those exist and stop pretending otherwise.

That isn't just better execution. It is a more honest form of thought.

The README describes the broader problem well: agents act on instinct, reaching for familiar tools and stating uncertain things with false confidence. Self-Model attacks both habits. If the agent has to consult a concrete inventory before making capability claims, it becomes harder to hallucinate affordances into existence.

This matters because capability hallucination is not a narrow tooling bug. It corrupts planning, confidence, user trust, and recovery. A plan built on nonexistent abilities is fake. Confidence in that plan is fake. Subsequent retries are fake. The whole stack inherits the original self-deception.

If you want more autonomous agents, this is where to start. Not with bolder prompting. With a disciplined answer to the question: what are you, concretely, capable of right now?

Metacognition begins there. Before an agent can monitor its reasoning, it has to know whether the reasoning terminates in reality.

---

## Skill Files

### SKILL.md

````markdown
---
name: self-model
description: "Build and consult a capability inventory — invoke when uncertain what you can do, when planning multi-step tasks, when a tool or skill fails unexpectedly, or at session start for complex work."
metadata: { "openclaw": { "emoji": "🪞" } }
---

# Self-Model: Capability Inventory

You maintain a structured inventory of your current capabilities — skills, tools, system resources, connected nodes, and known gaps. This inventory is the ground truth for what you can and cannot do *right now*. Consult it before planning, and update it when the environment changes.

## When to Build or Refresh

Build the inventory from scratch if `{baseDir}/capabilities.md` does not exist. Refresh incrementally when:

- A new skill or tool is installed
- A capability gap is discovered (tool call fails, skill missing, binary not found)
- The session environment has changed (new node connected, different machine)
- You are planning a complex multi-step task and the inventory is stale

**Staleness heuristic:** If the inventory timestamp is from a different session or if you have installed/removed skills or tools since the last update, treat it as stale.

## How to Survey Capabilities

Run these discovery steps. Skip any step whose section is already current.

### 1. Installed Skills

```bash
# List all skill directories
ls -1 skills/*/SKILL.md 2>/dev/null
```

For each skill found, read its frontmatter to extract `name` and `description`. Record the skill name, one-line description, and what it produces (outputs/side effects) if apparent from the description.

### 2. Available Tools

Check your system prompt for the tools list. You already have this in context — extract tool names and summarize each in a few words. Common tool sets: Read, Write, Edit, Bash, Glob, Grep, WebFetch, WebSearch, Agent, TodoWrite, NotebookEdit.

### 3. Connected Nodes

```bash
# Check node status if openclaw CLI is available
which openclaw &>/dev/null && openclaw node list 2>/dev/null || echo "No openclaw CLI found"
```

If no CLI is available, note "standalone mode — no connected nodes."

### 4. System Binaries

Probe PATH for commonly useful CLIs. Run a single check:

```bash
for cmd in git uv pip python3 node npm docker docker-compose ffmpeg jq yq curl wget gh rg fd make gcc cargo rustc go java mvn; do
  command -v "$cmd" &>/dev/null && echo "✓ $cmd: $(command -v "$cmd")"
done
```

### 5. Known Gaps

```bash
cat {baseDir}/capability-gaps.md 2>/dev/null || echo "No gaps file found"
```

If the file exists, incorporate its contents into the Gaps section of the inventory.

## Writing the Inventory

Write the inventory to `{baseDir}/capabilities.md` using the template at `{baseDir}/capabilities-template.md`. Fill in each section from your survey results. The format is designed to be both human-readable and easy for you to parse on subsequent reads.

**Rules:**
- Always set `last_updated` to the current date/time
- Always set `session_id` to a short identifier for the current session (use the first 8 chars of any session ID you have, or "unknown")
- Keep descriptions terse — this file costs tokens every time you read it
- For the Gaps section, note *what* you can't do and *why* (missing binary, no API key, no skill, etc.)

## Incremental Updates

Do not rebuild from scratch unless the inventory is missing. To update:

1. Read the current `{baseDir}/capabilities.md`
2. Update only the changed section(s)
3. Update the `last_updated` timestamp
4. If adding a gap, append to the Gaps section

To log a new gap at any time:

1. Append to `{baseDir}/capability-gaps.md` with the format: `- <what failed> | <why> | <date>`
2. Update the Gaps section in `{baseDir}/capabilities.md` if it exists

## Querying the Inventory

When making decisions, consult the inventory:

| Question | How to answer |
|---|---|
| "Can I do X?" | Check Skills + Tools + System sections. If X requires something not listed, the answer is no — log the gap. |
| "What's the best way to do X?" | List all approaches from available capabilities, compare trade-offs. |
| "What would I need to do X?" | Identify missing capabilities, feed into skill acquisition. |
| "What changed since last check?" | Compare `last_updated` and `session_id` to current session. |

Read `{baseDir}/capabilities.md` to answer these. Do not re-survey unless the inventory is stale.

## Output

After building or refreshing, briefly state what changed:
- New capabilities found
- Gaps discovered
- Sections that were already current (skipped)

Do not dump the full inventory into chat unless the user asks for it.
````

### AGENTS-snippet.md

````markdown
## Metacognitive Protocols

Before acting on complex or ambiguous tasks, apply these protocols:

- **Self-Model**: When unsure what you can do, when planning multi-step work, or when a tool/skill fails unexpectedly → read `skills/self-model/SKILL.md` and build or consult your capability inventory.
````
