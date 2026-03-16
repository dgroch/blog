---
layout: post
title: "Env-Model: intelligence depends on knowing the world around you"
date: 2026-03-16
description: "Capability is not enough. An agent also needs a working model of the runtime it is standing inside."
---

A system can know exactly what tools it has and still behave stupidly.

That happens whenever intelligence is treated as abstract instead of situated.

An agent may have permission to use a browser in one session and not another. It may be running in a container, on a remote node, in a group chat, in a sub-agent context, on a machine with no network egress, or under a channel with formatting and length constraints. None of that is a side detail. It is the world the cognition has to inhabit.

This is what the Env-Model skill is for.

[→ Jump to skill files](#skill-files)

OpenClaw separates it cleanly from Self-Model, and that separation is more important than it looks. Self-Model tracks what the agent can do. Env-Model tracks what is around the agent: runtime, channels, nodes, user context, session state, and connectivity. That distinction prevents a common category error. Having a tool is not the same thing as being in conditions where the tool is usable.

The `SKILL.md` reflects that. The quick scan block is deliberately cheap:

```bash
echo "--- runtime ---"
uname -srm
hostname
echo "workspace: $(pwd)"
echo "--- sandbox ---"
[[ -f /.dockerenv ]] && echo "container: yes" || echo "container: no"
echo "--- disk ---"
df -h . 2>/dev/null | tail -1
echo "--- openclaw ---"
which openclaw &>/dev/null && openclaw status 2>/dev/null || echo "no openclaw CLI"
echo "--- connectivity ---"
[[ -n "${BROWSER_AVAILABLE:-}" ]] && echo "browser: yes" || echo "browser: no"
```

It checks runtime basics with `uname`, `hostname`, current workspace, whether the process is in a container, local disk availability, OpenClaw status, and whether browser availability is signaled. Then it tells the agent to note model, thinking level, session type, current channel, and timezone from the prompt itself. If more detail is needed, there is a deep scan broken into layers: runtime, OpenClaw, user, and connectivity.

That layered structure is a quiet piece of good design. Most environment failures do not require a forensic expedition. They require the agent to ask the right question at the right depth. Is this a sandbox issue? A network issue? A channel issue? A node issue? A user-context issue? Doing a quick scan by default and probing deeper only by layer keeps the cost low and the diagnosis legible.

The skill also embeds concrete communication constraints. The channel constraints from the deep scan section:

> - WhatsApp: ~65k char limit, no markdown rendering, no code blocks
> - Telegram: markdown supported, 4096 char message limit
> - Slack: mrkdwn format, 40k char limit, threads available
> - CLI/direct: no constraints

These are not UX flourishes. They shape what a correct answer looks like.

That is the larger point: intelligence is partly formatting judgment under constraints.

A lot of agent discourse still assumes cognition happens in a clean vacuum where the model's only job is to produce the right abstract answer. Real systems don't get to live there. They operate in channels, on machines, with permissions, inside service boundaries, under time limits, alongside human expectations. An agent that ignores those conditions can still sound smart while being operationally useless.

The env-model file written to `environment.md` becomes the external memory for those facts. The skill explicitly marks entries as confirmed or inferred, tells the agent to treat the file as last known state, and warns that environmental information decays quickly. That is metacognition in the practical sense: the system is not just storing facts, but storing the reliability class of those facts.

The skill's querying table makes this explicit:

> | Question | How to answer |
> |---|---|
> | "Can I use the browser?" | Check Session section for browser availability. |
> | "Should I send a long formatted response?" | Check Channels section for current channel constraints. |
> | "Why did this command fail?" | Check Runtime section for sandbox/permissions; check Connectivity for network. |
> | "Is now a good time for a proactive message?" | Check User section for timezone and channel context. |

Imagine an agent asked to send a concise update while also launching a lengthy background job. In a desktop shell, the answer may be one kind of message and one kind of execution flow. In Telegram, the answer needs tighter length control and cleaner formatting. In a sub-agent session, the correct behaviour may be to keep the response compact and write details to files instead. Same request. Different world. Different intelligent action.

The AGENTS trigger captures this nicely:

> - **Env-Model**: When encountering unexpected failures, adapting output to a channel, or planning environment-dependent work → read `skills/env-model/SKILL.md` and build or consult your environmental model.

Stop assuming the world is stable just because the task is familiar.

An agent is not intelligent in the abstract. It is intelligent in a runtime, in a channel, on a machine, under constraints. If it cannot model that world, it doesn't really know where it is standing.

---

## Skill Files

### SKILL.md

````markdown
---
name: env-model
description: "Build and consult an environmental model — invoke on session start, when commands fail unexpectedly, when adapting output to a channel, or when planning environment-dependent work."
metadata: { "openclaw": { "emoji": "🌍" } }
---

# Environmental Model Building

You maintain a working model of the world around you — runtime, channels, nodes, user context, session state, and connectivity. This is distinct from the capability inventory (see `skills/self-model/SKILL.md`), which tracks what you *can do*. This model tracks what's *around you*.

Consult the environmental model before acting on assumptions about your runtime, the user's channel, or external connectivity. Update it when the environment shifts.

## When to Build or Refresh

Build from scratch if `{baseDir}/environment.md` does not exist. Refresh when:

- A command or tool fails in a way suggesting environmental constraints (sandbox, permissions, network)
- You're about to do something environment-dependent (send a message, use the browser, access a remote node, run a long task)
- The user context shifts (new channel, group vs DM, different user)
- You're planning work that depends on environmental capabilities

**Staleness:** The environmental model is more volatile than the capability inventory. Treat it as "last known state." Verify critical assumptions (connectivity, node status) before acting on them if more than a few minutes have passed.

## How to Survey the Environment

Use a **quick scan** by default. Run a **deep scan** only when you need detail on a specific layer or when diagnosing a failure.

### Quick Scan (default — run on session start)

Run this single block to capture the essentials:

```bash
echo "--- runtime ---"
uname -srm
hostname
echo "workspace: $(pwd)"
echo "--- sandbox ---"
[[ -f /.dockerenv ]] && echo "container: yes" || echo "container: no"
echo "--- disk ---"
df -h . 2>/dev/null | tail -1
echo "--- openclaw ---"
which openclaw &>/dev/null && openclaw status 2>/dev/null || echo "no openclaw CLI"
echo "--- connectivity ---"
[[ -n "${BROWSER_AVAILABLE:-}" ]] && echo "browser: yes" || echo "browser: no"
```

Then note from your system prompt:
- Current model and thinking level (from the runtime info line)
- Session type (main agent vs sub-agent)
- Current channel (if present in runtime info)
- Current date/time and timezone

### Deep Scan (on demand, by layer)

Run only the layers you need.

**Runtime layer:**
```bash
uname -a
cat /etc/os-release 2>/dev/null | head -3
free -h 2>/dev/null || vm_stat 2>/dev/null | head -5
df -h .
whoami
echo "PATH dirs: $(echo $PATH | tr ':' '\n' | wc -l)"
```

**OpenClaw layer:**
```bash
which openclaw &>/dev/null && {
  openclaw status
  openclaw node list
  openclaw channel list
} || echo "no openclaw CLI — standalone mode"
```

**User layer:**
```bash
cat USER.md 2>/dev/null || echo "no USER.md"
cat SOUL.md 2>/dev/null | head -20 || echo "no SOUL.md"
```

Check communication constraints from the current channel:
- WhatsApp: ~65k char limit, no markdown rendering, no code blocks
- Telegram: markdown supported, 4096 char message limit
- Slack: mrkdwn format, 40k char limit, threads available
- CLI/direct: no constraints

**Connectivity layer:**
```bash
# Check for configured API keys (names only, not values)
env | grep -iE '_(KEY|TOKEN|SECRET)=' | sed 's/=.*/=<set>/' 2>/dev/null
# Check network egress
curl -s --max-time 3 -o /dev/null -w "%{http_code}" https://httpbin.org/get 2>/dev/null || echo "no egress"
```

## Writing the Environmental Model

Write to `{baseDir}/environment.md` using the template at `{baseDir}/environment-template.md`.

**Rules:**
- Always set `last_updated` to current date/time
- Always set `session_id` to match the value used in the capability inventory (or "unknown")
- Keep entries terse — this file is read repeatedly
- Mark inferred values with `(inferred)` and confirmed values with `(confirmed)` in the Confidence column where applicable
- For quick scans, leave deep-scan-only fields as "not probed" rather than empty

## Incremental Updates

Do not rebuild from scratch unless the model is missing. To update:

1. Read the current `{baseDir}/environment.md`
2. Update only the changed section(s)
3. Update `last_updated`
4. If a section becomes unreliable (e.g., node disconnected), flag it as stale rather than deleting

## Querying the Environmental Model

| Question | How to answer |
|---|---|
| "Can I use the browser?" | Check Session section for browser availability. |
| "Should I send a long formatted response?" | Check Channels section for current channel constraints. |
| "Why did this command fail?" | Check Runtime section for sandbox/permissions; check Connectivity for network. |
| "Is now a good time for a proactive message?" | Check User section for timezone and channel context. |
| "Can I reach this external service?" | Check Connectivity section. If stale, re-probe before acting. |
| "What are the constraints for this task?" | Read both this model and the capability inventory (`skills/self-model/capabilities.md`). |

Read `{baseDir}/environment.md` to answer these. Re-probe a specific layer only if the relevant section is stale or missing.

## Output

After building or refreshing, briefly state:
- Key environmental facts discovered (OS, channel, sandbox status)
- Anything surprising or constraining
- Sections left as "not probed"

Do not dump the full model into chat unless the user asks for it.
````

### AGENTS-snippet.md

````markdown
- **Env-Model**: When encountering unexpected failures, adapting output to a channel, or planning environment-dependent work → read `skills/env-model/SKILL.md` and build or consult your environmental model.
````
