---
layout: post
title: "Attention Filter: what the agent sees shapes what it can think"
date: 2026-03-16
description: "Relevance-aware injection is not garnish. It is part of the cognitive architecture."
---

Before an agent reasons, something has already decided what is in front of it.


> The full implementation is available on GitHub: [github.com/dgroch/metacognition](https://github.com/dgroch/metacognition)

That sounds trivial. It isn't.

What the agent sees, in what order, with what framing, and with what lightweight cues about urgency or continuity will shape the reasoning that follows. Attention is not merely an internal property of the model. It is also an architectural choice.

That is what the Attention Filter hook makes explicit.

[→ Jump to hook files](#hook-files)

The `HOOK.md` describes it as an always-on layer that does not remove core context but adds targeted focusing signals. The hook registers for two lifecycle events:

```yaml
events:
  - agent:bootstrap
  - message:preprocessed
```

On `agent:bootstrap`, it performs relevance-aware context injection. On `message:preprocessed`, it classifies incoming signals and helps preserve focus when new requests arrive mid-task.

The handler implementation is refreshingly concrete. The urgency classifier uses simple regex patterns:

```typescript
const URGENT_PATTERNS = /\b(urgent|asap|now|immediately|quickly|emergency|critical|broken|down|outage)\b/i;
const HEARTBEAT_PATTERN = /\b(heartbeat|poll|cron|scheduled|ping)\b/i;
```

It maps common user-message patterns to relevant artifact pointers via a static lookup table:

```typescript
const ARTIFACT_RELEVANCE: Array<{
  pattern: RegExp;
  pointer: string;
  file: string;
}> = [
  {
    pattern: /\b(what can you do|capabilities|what do you know|your skills|your tools)\b/i,
    pointer: "Relevant: capability inventory → `skills/self-model/capabilities.md`",
    file: "skills/self-model/capabilities.md",
  },
  {
    pattern: /\b(not working|error|fail|broke|bug|crash|issue|problem)\b/i,
    pointer: "Relevant: failure log → `skills/failure-recovery/failure-log.md`, env-model → `skills/env-model/environment.md`",
    file: "skills/failure-recovery/failure-log.md",
  },
  {
    pattern: /\b(plan|complex|multi.?step|break.?down|decompos|architect)\b/i,
    pointer: "Relevant: task decomposition → `skills/task-decomposition/SKILL.md`",
    file: "skills/task-decomposition/SKILL.md",
  },
  {
    pattern: /\b(what have you|working on|progress|status|recent|history)\b/i,
    pointer: "Relevant: composition log → `skills/task-composition/composition-log.md`",
    file: "skills/task-composition/composition-log.md",
  },
  {
    pattern: /\b(environment|system|platform|node|connect|network|sandbox)\b/i,
    pointer: "Relevant: env-model → `skills/env-model/environment.md`",
    file: "skills/env-model/environment.md",
  },
];

// Max artifact pointers to inject per turn
const MAX_POINTERS = 2;
```

Capability questions point toward `skills/self-model/capabilities.md`; failure language points toward the failure log and env-model; planning language points toward the task-decomposition skill.

The file checks are cheap. `fs.existsSync()`. Small reads from `.active-plan.md`. No LLM calls in the hot path. A cap of two artifact pointers per turn. This is important because hooks that fire every turn have to behave like infrastructure, not research demos.

The active-task handling is the core of it. If `.active-plan.md` exists, the hook reads a small head slice, extracts the goal and maybe the current in-progress step, and injects a directive:

```typescript
if (fs.existsSync(activePlanPath)) {
  hasActiveTask = true;
  try {
    const head = fs.readFileSync(activePlanPath, "utf-8").slice(0, 500);
    const goalMatch = head.match(/\*\*Goal\*\*:\s*(.+)/);
    const inProgress = head.match(/\[in.progress\].*$/m);
    const goal = goalMatch ? goalMatch[1].trim() : "see .active-plan.md";
    const step = inProgress ? inProgress[0].trim() : "";
    attentionLines.push(`ACTIVE PLAN: ${goal}${step ? ` — current step: ${step}` : ""}`);
    attentionLines.push("Priority: resume this work unless the user redirects.");
  } catch {
    attentionLines.push("ACTIVE PLAN: see `.active-plan.md` for current state.");
  }
}
```

During message preprocessing, if active work exists and the incoming message is not urgent, it attaches a focus preservation note:

```typescript
const focusNote = `[attention-filter] Note: You were working on "${activeTaskSummary}". ` +
  `This appears to be a new request. Decide whether to switch or finish current work first.`;
```

That sounds almost too simple. But simplicity is the point. Attention scaffolding should be cheap, legible, and reliable.

A lot of current agent systems dump everything into the prompt and hope the model sorts salience out internally. Sometimes it does. Sometimes it doesn't. Attention Filter is a direct rejection of that passive design. It says relevance can be shaped mechanically before reasoning begins.

The relationship to the companion [Attention Awareness](https://dangroch.com/2026/03/16/attention-awareness-for-ai-agents/) skill is exactly right. The hook shapes what the agent notices. The skill teaches it how to reason about what matters, what is background, what is noise, and when to switch tasks. One is automated scaffolding. The other is conscious discipline.

A concrete example helps. Suppose the agent has an active plan to write a report and receives a new message saying the staging site is down. The hook tags the message urgent. That alone changes the attentional landscape before the model does any deeper reasoning. If instead the new message is just ambient chatter or a low-priority question, the hook can preserve continuity by reminding the agent that current work exists and a switch should be deliberate.

The larger thesis is simple: attention allocation is one of the hidden levers of cognition. If you want more reliable reasoning, don't just improve the reasoner. Improve the layer that decides what deserves attention in the first place.

What the agent sees shapes what it can think. Treating that as architecture rather than accident is one of the smartest ideas in this repo.

---

## Hook Files

### HOOK.md

````markdown
---
name: attention-filter
description: "Metacognitive attention filtering — relevance-aware context injection, incoming signal prioritization, and focus preservation across turns."
metadata:
  openclaw:
    emoji: "🔍"
    events:
      - agent:bootstrap
      - message:preprocessed
---

# Attention Filter Hook

An always-on hook that shapes what the agent pays attention to on each turn. It doesn't remove core context — it adds targeted focusing signals so the agent knows what's relevant right now.

This is the automated layer. The companion skill (`skills/attention-awareness/SKILL.md`) handles the reasoning the agent controls.

## What This Hook Does

### 1. Relevance-Aware Context Injection (`agent:bootstrap`)

Fires before bootstrap files are injected. Adds brief attention directives based on workspace state:

- **Active task detection**: Checks for `.context-checkpoint.md`, `.active-plan.md`, or `.metacognitive-state.md`. If found, injects a short focusing signal (~50-100 tokens) telling the agent what it was doing and which step it's on.
- **Metacognitive artifact pointers**: Lightweight keyword analysis of the last user message to surface 1-2 relevant skill artifacts (capability inventory, failure log, env-model, etc.) — only when clearly relevant, not on every turn.
- **Session type awareness**: Full pointers in main sessions. Minimal injection in sub-agent sessions. No metacognitive pointers in group chats.

Does NOT remove or modify existing bootstrap files (SOUL.md, AGENTS.md always load). Only adds signals.

### 2. Incoming Signal Classification (`message:preprocessed`)

Fires after message enrichment, before the agent sees the message:

- **Priority tagging**: Classifies incoming messages as Urgent / Normal / Ambient based on keyword patterns.
- **Focus preservation**: If the agent has active work and a new unrelated request arrives, injects a brief note: "You were working on [X]. This appears to be a new request. Decide whether to switch or finish current work first."
- **Heartbeat filtering**: Skips attention directives for heartbeat polls to keep them fast.

## Performance Requirements

Both events fire every turn. Target <30ms per handler:
- Use `fs.existsSync()` checks, not full file reads
- Keyword matching only, no semantic analysis, no LLM calls
- Fail silently on any error — never block the agent

## Coordination

- **Context Guard hook** (`hooks/context-guard/`): Fires on the same `agent:bootstrap` event. Context Guard handles state continuity across compaction. Attention Filter handles relevance-based focusing. They're complementary — hook execution order doesn't matter since both only add to `bootstrapFiles`.
- **Companion skill** (`skills/attention-awareness/SKILL.md`): The hook shapes what the agent sees; the skill teaches the agent how to reason about relevance, focus, and context-switching.

## Configuration

```json
{
  "hooks": {
    "internal": {
      "enabled": true,
      "entries": {
        "attention-filter": { "enabled": true }
      }
    }
  }
}
```
````

### handler.ts

````typescript
import type { HookHandler } from "../../src/hooks/hooks.js";
import fs from "fs";
import path from "path";

// ---------------------------------------------------------------------------
// Attention Filter Hook
// ---------------------------------------------------------------------------
// agent:bootstrap       → relevance-aware context injection
// message:preprocessed  → incoming signal classification + focus preservation
// ---------------------------------------------------------------------------

const LOG_PREFIX = "[attention-filter]";

// ---------------------------------------------------------------------------
// Keyword patterns for message classification and artifact relevance
// ---------------------------------------------------------------------------

const URGENT_PATTERNS = /\b(urgent|asap|now|immediately|quickly|emergency|critical|broken|down|outage)\b/i;
const HEARTBEAT_PATTERN = /\b(heartbeat|poll|cron|scheduled|ping)\b/i;

// Maps keyword patterns in user messages to relevant metacognitive artifacts
const ARTIFACT_RELEVANCE: Array<{
  pattern: RegExp;
  pointer: string;
  file: string; // relative to workspace, for existence check
}> = [
  {
    pattern: /\b(what can you do|capabilities|what do you know|your skills|your tools)\b/i,
    pointer: "Relevant: capability inventory → `skills/self-model/capabilities.md`",
    file: "skills/self-model/capabilities.md",
  },
  {
    pattern: /\b(not working|error|fail|broke|bug|crash|issue|problem)\b/i,
    pointer: "Relevant: failure log → `skills/failure-recovery/failure-log.md`, env-model → `skills/env-model/environment.md`",
    file: "skills/failure-recovery/failure-log.md",
  },
  {
    pattern: /\b(plan|complex|multi.?step|break.?down|decompos|architect)\b/i,
    pointer: "Relevant: task decomposition → `skills/task-decomposition/SKILL.md`",
    file: "skills/task-decomposition/SKILL.md",
  },
  {
    pattern: /\b(what have you|working on|progress|status|recent|history)\b/i,
    pointer: "Relevant: composition log → `skills/task-composition/composition-log.md`",
    file: "skills/task-composition/composition-log.md",
  },
  {
    pattern: /\b(environment|system|platform|node|connect|network|sandbox)\b/i,
    pointer: "Relevant: env-model → `skills/env-model/environment.md`",
    file: "skills/env-model/environment.md",
  },
];

// Max artifact pointers to inject per turn
const MAX_POINTERS = 2;

// ---------------------------------------------------------------------------
// Bootstrap handler — relevance-aware context injection
// ---------------------------------------------------------------------------

function handleBootstrap(event: any): void {
  const workspaceDir: string | undefined = event?.context?.workspaceDir;
  if (!workspaceDir) return;

  // Skip heavy injection for sub-agent sessions
  const isSubAgent = event?.context?.promptMode === "minimal" ||
    event?.context?.isSubAgent === true;
  const isGroupChat = event?.context?.channelType === "group";

  // 1. Active task detection — check for state files (existsSync is fast)
  const attentionLines: string[] = [];

  const checkpointPath = path.join(workspaceDir, ".context-checkpoint.md");
  const activePlanPath = path.join(workspaceDir, ".active-plan.md");
  const metacogStatePath = path.join(workspaceDir, ".metacognitive-state.md");

  let hasActiveTask = false;

  if (fs.existsSync(activePlanPath)) {
    hasActiveTask = true;
    // Read just the first few lines to get plan name/step (fast, small file)
    try {
      const head = fs.readFileSync(activePlanPath, "utf-8").slice(0, 500);
      const goalMatch = head.match(/\*\*Goal\*\*:\s*(.+)/);
      const inProgress = head.match(/\[in.progress\].*$/m);
      const goal = goalMatch ? goalMatch[1].trim() : "see .active-plan.md";
      const step = inProgress ? inProgress[0].trim() : "";
      attentionLines.push(`ACTIVE PLAN: ${goal}${step ? ` — current step: ${step}` : ""}`);
      attentionLines.push("Priority: resume this work unless the user redirects.");
    } catch {
      attentionLines.push("ACTIVE PLAN: see `.active-plan.md` for current state.");
    }
  } else if (fs.existsSync(checkpointPath)) {
    hasActiveTask = true;
    attentionLines.push("RESUMED SESSION: see `.context-checkpoint.md` for prior state.");
  } else if (fs.existsSync(metacogStatePath)) {
    hasActiveTask = true;
    attentionLines.push("METACOGNITIVE CONTINUITY: see `.metacognitive-state.md`.");
  }

  // 2. Metacognitive artifact pointers (skip for sub-agents and group chats)
  if (!isSubAgent && !isGroupChat) {
    const lastMessage = extractLastUserMessage(event);
    if (lastMessage) {
      let pointersAdded = 0;
      for (const rule of ARTIFACT_RELEVANCE) {
        if (pointersAdded >= MAX_POINTERS) break;
        if (rule.pattern.test(lastMessage)) {
          // Only add pointer if the artifact file actually exists
          const fullPath = path.join(workspaceDir, rule.file);
          if (fs.existsSync(fullPath)) {
            attentionLines.push(rule.pointer);
            pointersAdded++;
          }
        }
      }
    }
  }

  // 3. Inject attention directive if we have anything to say
  if (attentionLines.length > 0 && !isSubAgent) {
    const directive = [
      "<!-- attention-filter -->",
      ...attentionLines,
      "<!-- /attention-filter -->",
    ].join("\n");

    if (event?.context?.bootstrapFiles && Array.isArray(event.context.bootstrapFiles)) {
      event.context.bootstrapFiles.push({
        name: "ATTENTION.md",
        content: directive,
      });
    }
  }
}

// ---------------------------------------------------------------------------
// Message preprocessed handler — signal classification + focus preservation
// ---------------------------------------------------------------------------

function handleMessagePreprocessed(event: any): void {
  const workspaceDir: string | undefined = event?.context?.workspaceDir;
  if (!workspaceDir) return;

  const messageText = extractMessageText(event);
  if (!messageText) return;

  // Skip attention directives for heartbeat polls
  if (HEARTBEAT_PATTERN.test(messageText)) {
    return;
  }

  // 1. Priority classification
  const isUrgent = URGENT_PATTERNS.test(messageText);

  // 2. Focus preservation — if active task exists and this looks like a new topic
  const activePlanPath = path.join(workspaceDir, ".active-plan.md");
  const checkpointPath = path.join(workspaceDir, ".context-checkpoint.md");
  const hasActiveWork = fs.existsSync(activePlanPath) || fs.existsSync(checkpointPath);

  if (hasActiveWork && !isUrgent) {
    // Read active task summary for the focus note
    let activeTaskSummary = "an in-progress task";
    if (fs.existsSync(activePlanPath)) {
      try {
        const head = fs.readFileSync(activePlanPath, "utf-8").slice(0, 300);
        const goalMatch = head.match(/\*\*Goal\*\*:\s*(.+)/);
        if (goalMatch) {
          activeTaskSummary = goalMatch[1].trim();
        }
      } catch {
        // use default summary
      }
    }

    // Inject focus preservation note
    const focusNote = `[attention-filter] Note: You were working on "${activeTaskSummary}". ` +
      `This appears to be a new request. Decide whether to switch or finish current work first.`;

    if (event?.context?.metadata) {
      event.context.metadata.attentionNote = focusNote;
    } else if (event?.context) {
      event.context.metadata = { attentionNote: focusNote };
    }
  }

  // Tag priority on the event context for downstream consumers
  if (event?.context) {
    if (!event.context.metadata) event.context.metadata = {};
    event.context.metadata.messagePriority = isUrgent ? "urgent" : "normal";
  }
}

// ---------------------------------------------------------------------------
// Utility helpers
// ---------------------------------------------------------------------------

/** Extract the last user message text from event context. */
function extractLastUserMessage(event: any): string | null {
  // Try various event context shapes
  if (event?.context?.lastUserMessage) {
    return String(event.context.lastUserMessage);
  }
  if (event?.context?.messages && Array.isArray(event.context.messages)) {
    const userMsgs = event.context.messages.filter(
      (m: any) => m?.role === "user" || m?.type === "user"
    );
    if (userMsgs.length > 0) {
      const last = userMsgs[userMsgs.length - 1];
      return typeof last.content === "string"
        ? last.content
        : typeof last.text === "string"
          ? last.text
          : null;
    }
  }
  return null;
}

/** Extract message text from a preprocessed message event. */
function extractMessageText(event: any): string | null {
  if (event?.message?.text) return String(event.message.text);
  if (event?.message?.content) return String(event.message.content);
  if (event?.data?.text) return String(event.data.text);
  return extractLastUserMessage(event);
}

// ---------------------------------------------------------------------------
// Main handler — event router
// ---------------------------------------------------------------------------

const handler: HookHandler = async (event) => {
  try {
    if (event.type === "agent" && event.action === "bootstrap") {
      handleBootstrap(event);
      return;
    }

    if (event.type === "message" && event.action === "preprocessed") {
      handleMessagePreprocessed(event);
      return;
    }
  } catch (err) {
    // Never throw from a hook — fail silently
    console.log(`${LOG_PREFIX} Error: ${err instanceof Error ? err.message : String(err)}`);
  }
};

export default handler;
````

---

**Next in the series:** [Context Guard: preserving the mind across compaction](https://dangroch.com/2026/03/16/context-guard-hook/)
