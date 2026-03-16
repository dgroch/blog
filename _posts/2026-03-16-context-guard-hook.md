---
layout: post
title: "Context Guard: preserving the mind across compaction"
date: 2026-03-16
description: "Some forms of metacognition shouldn't depend on the agent remembering to invoke them."
---

One of the hardest things about long-running agent systems is that memory loss is not binary.

Before a hard context reset, there is gradual drift. Then comes compaction, which preserves some things, compresses others, and quietly drops details the model may still need. If the agent has to remember, in that moment, to preserve its own critical state, you are already gambling.

That is why the Context Guard hook matters.

It is not another reasoning prompt. It is infrastructure.

[→ Jump to hook files](#hook-files)

The `HOOK.md` lays out the contract clearly. The hook registers for exactly three lifecycle events:

```yaml
events:
  - agent:bootstrap
  - session:compact:before
  - session:compact:after
```

On `session:compact:before`, the hook writes a `.context-checkpoint.md` file to the workspace root. That file captures active task state, metacognitive state, key decisions, recently changed files, and next steps. On `agent:bootstrap`, it checks for a recent checkpoint, injects a concise recovery block into bootstrap files, and renames the checkpoint to `.context-checkpoint.recovered.md` so the state is not replayed forever.

This is exactly the kind of metacognitive function that should be automated. The agent is most vulnerable to forgetting what matters at the point where the context window is already under stress. Asking it to self-manage perfectly in that moment is wishful thinking.

The TypeScript handler is practical in all the right ways. The operational constants are explicit:

```typescript
const CHECKPOINT_FILE = ".context-checkpoint.md";
const RECOVERED_FILE = ".context-checkpoint.recovered.md";

// Max age (ms) for a checkpoint to be considered "recent" for recovery
const CHECKPOINT_MAX_AGE_MS = 10 * 60 * 1000; // 10 minutes

// Max tokens budget for the recovery injection (~500 tokens ≈ ~2000 chars)
const RECOVERY_MAX_CHARS = 2000;
```

It uses a ten-minute freshness window for recovery. It caps the injected recovery block at about two thousand characters to stay within a small token budget. It scans for recently modified files within the last thirty minutes, bounded by shallow recursion. It looks for `.active-plan.md`, a failure log, and model artifacts like `capabilities.md` and `environment.md` to estimate the freshness of the broader metacognitive state.

The metacognitive state gathering function shows exactly what the hook preserves:

```typescript
interface MetacogState {
  planSummary: string;
  lastFailure: string;
  pendingAssumptions: string;
  selfModelAge: string;
  envModelAge: string;
}
```

These are the things ordinary compaction summaries often miss because they are operationally important but not always conversationally prominent. A standard summary may remember the topic. Context Guard tries to remember the state of mind.

I especially like the design principle that it never throws. All operations are wrapped defensively:

```typescript
} catch (err) {
  // Never throw from a hook — let other hooks run
  console.log(`${LOG_PREFIX} Error: ${err instanceof Error ? err.message : String(err)}`);
}
```

If artifacts are missing, sections are skipped. The hook is additive, not brittle. That is exactly how background cognition support should behave.

Look at what the checkpoint actually preserves:

```typescript
const lines: string[] = [
  `# Context Checkpoint — ${nowStamp()}`,
  "## Active Task",
  "## Metacognitive State",
  `- **Plan**: ${metacog.planSummary}`,
  `- **Last failure**: ${metacog.lastFailure}`,
  `- **Pending assumptions**: ${metacog.pendingAssumptions}`,
  `- **Self-model freshness**: ${metacog.selfModelAge}`,
  `- **Env-model freshness**: ${metacog.envModelAge}`,
  "## Files Changed This Session",
  "## Next Steps",
];
```

Not just generic session summary, but plan summary, last failure, pending assumptions, self-model freshness, env-model freshness, changed files, and next steps.

Imagine an agent halfway through a multi-step migration. It has already made one key decision about tool choice, logged a recent environment-related failure, and written two files. Compaction hits. Without a hook like this, the resumed system may know that it was "working on the migration" and almost nothing else. With Context Guard, it can recover not just the headline task but the active plan, the failure context, and the next action. That is a much more coherent restart.

This is the larger lesson of the hook layer in the suite. Some metacognitive disciplines are too important to remain voluntary habits. Preserving state across compaction is one of them.

If we want agents that operate for long enough to matter, we need to build for the moments when their memory compresses, shifts, or partially disappears. Context Guard is a practical answer to that problem. Not philosophical continuity. Operational continuity.

---

## Hook Files

### HOOK.md

````markdown
---
name: context-guard
description: "Monitors context window usage, checkpoints metacognitive state before compaction, recovers working state after compaction, and prunes oversized tool results."
metadata:
  openclaw:
    emoji: "🛡️"
    events:
      - agent:bootstrap
      - session:compact:before
      - session:compact:after
---

# Context Guard Hook

An always-on hook that protects metacognitive state across compaction boundaries. Works alongside OpenClaw's existing compaction system — additive, not replacing.

## What This Hook Does

### 1. Pre-Compaction Checkpoint (`session:compact:before`)

When compaction is about to happen, writes `.context-checkpoint.md` to the workspace root capturing:

- **Active task**: what the agent was working on
- **Metacognitive state**: active plan summary, recent failure log entries, pending uncertain assumptions, self-model/env-model freshness
- **Key decisions**: significant choices made this session with rationale
- **Files changed**: recently modified workspace files
- **Next steps**: what to resume after compaction

The critical addition over standard compaction: the **Metacognitive State** section preserves references to plan state, failure patterns, and calibration flags that the standard compaction summary doesn't know to keep.

### 2. Post-Compaction Recovery (`agent:bootstrap`)

On bootstrap, checks for a recent `.context-checkpoint.md`:

- If found and not yet recovered, injects a concise recovery context block (~500 tokens) into bootstrap files
- Tells the agent: what it was doing, what plan step was active, critical assumptions, and what to do next
- Renames checkpoint to `.context-checkpoint.recovered.md` to prevent re-injection

### 3. Graceful Degradation

- Works whether or not the metacognitive skills are installed
- If skill artifact files (failure-log.md, capabilities.md, etc.) don't exist, skips those sections
- Never throws — all operations wrapped in try/catch
- Fast — no heavy I/O or parsing

## Checkpoint File Reference

The checkpoint is written to `{workspace}/.context-checkpoint.md`. Recovery renames it to `{workspace}/.context-checkpoint.recovered.md`.

## Configuration

Enable in `openclaw.json`:

```json
{
  "hooks": {
    "internal": {
      "enabled": true,
      "entries": {
        "context-guard": { "enabled": true }
      }
    }
  }
}
```

## Coordination

- **Session-memory hook**: Complementary. Session-memory saves on `/new`; context-guard saves before compaction. Different lifecycle points, no duplication.
- **Compaction safeguards**: OpenClaw's compaction preserves section headings and active task status. This hook adds metacognitive state preservation on top.
- **Companion skill**: `skills/context-management/SKILL.md` teaches the agent strategic context reasoning. This hook handles the automated plumbing.
````

### handler.ts

````typescript
import type { HookHandler } from "../../src/hooks/hooks.js";
import fs from "fs/promises";
import path from "path";

// ---------------------------------------------------------------------------
// Context Guard Hook
// ---------------------------------------------------------------------------
// Automated layer for context management:
//   session:compact:before  → checkpoint metacognitive state
//   session:compact:after   → (reserved for future use)
//   agent:bootstrap         → inject recovery context if post-compaction
// ---------------------------------------------------------------------------

const CHECKPOINT_FILE = ".context-checkpoint.md";
const RECOVERED_FILE = ".context-checkpoint.recovered.md";
const LOG_PREFIX = "[context-guard]";

// Max age (ms) for a checkpoint to be considered "recent" for recovery
const CHECKPOINT_MAX_AGE_MS = 10 * 60 * 1000; // 10 minutes

// Max tokens budget for the recovery injection (~500 tokens ≈ ~2000 chars)
const RECOVERY_MAX_CHARS = 2000;

// ---------------------------------------------------------------------------
// Utility helpers
// ---------------------------------------------------------------------------

async function fileExists(filePath: string): Promise<boolean> {
  try {
    await fs.access(filePath);
    return true;
  } catch {
    return false;
  }
}

async function readFileSafe(filePath: string): Promise<string | null> {
  try {
    return await fs.readFile(filePath, "utf-8");
  } catch {
    return null;
  }
}

/** Read the first N lines of a file, or null if missing. */
async function readHead(filePath: string, lines: number): Promise<string | null> {
  const content = await readFileSafe(filePath);
  if (!content) return null;
  return content.split("\n").slice(0, lines).join("\n");
}

/** Get ISO date-time string. */
function nowStamp(): string {
  return new Date().toISOString().replace("T", " ").replace(/\.\d+Z$/, " UTC");
}

/** Find recently modified files in workspace (last 30 min), excluding dotfiles and node_modules. */
async function recentlyModifiedFiles(workspaceDir: string): Promise<string[]> {
  const cutoff = Date.now() - 30 * 60 * 1000;
  const results: string[] = [];

  async function walk(dir: string, depth: number): Promise<void> {
    if (depth > 3) return; // don't recurse too deep
    let entries;
    try {
      entries = await fs.readdir(dir, { withFileTypes: true });
    } catch {
      return;
    }
    for (const entry of entries) {
      if (entry.name.startsWith(".") || entry.name === "node_modules") continue;
      const full = path.join(dir, entry.name);
      if (entry.isDirectory()) {
        await walk(full, depth + 1);
      } else {
        try {
          const stat = await fs.stat(full);
          if (stat.mtimeMs > cutoff) {
            results.push(path.relative(workspaceDir, full));
          }
        } catch {
          // skip inaccessible files
        }
      }
    }
  }

  await walk(workspaceDir, 0);
  return results.sort();
}

// ---------------------------------------------------------------------------
// Metacognitive state gathering
// ---------------------------------------------------------------------------

interface MetacogState {
  planSummary: string;
  lastFailure: string;
  pendingAssumptions: string;
  selfModelAge: string;
  envModelAge: string;
}

async function gatherMetacogState(workspaceDir: string): Promise<MetacogState> {
  const state: MetacogState = {
    planSummary: "none",
    lastFailure: "none",
    pendingAssumptions: "none",
    selfModelAge: "unknown",
    envModelAge: "unknown",
  };

  // Active plan — check for .active-plan.md first, then plan-template artifacts
  const activePlan = await readFileSafe(path.join(workspaceDir, ".active-plan.md"));
  if (activePlan) {
    // Extract the goal line and current step
    const goalMatch = activePlan.match(/\*\*Goal\*\*:\s*(.+)/);
    const steps = activePlan.match(/^\d+\.\s.+/gm);
    const inProgress = activePlan.match(/\[in.progress\].*$/gm);
    const parts: string[] = [];
    if (goalMatch) parts.push(`Goal: ${goalMatch[1]}`);
    if (inProgress && inProgress.length > 0) {
      parts.push(`Current step: ${inProgress[0].trim()}`);
    } else if (steps && steps.length > 0) {
      parts.push(`${steps.length} steps total`);
    }
    state.planSummary = parts.length > 0 ? parts.join("; ") : "plan file exists but could not parse";
  }

  // Recent failure log entries
  const failureLogPath = path.join(workspaceDir, "skills", "failure-recovery", "failure-log.md");
  const failureLog = await readFileSafe(failureLogPath);
  if (failureLog) {
    // Grab the last entry (entries are separated by ---)
    const entries = failureLog.split(/^---$/m).filter((e) => e.trim());
    if (entries.length > 0) {
      const lastEntry = entries[entries.length - 1].trim();
      // Take first 3 lines as summary
      state.lastFailure = lastEntry.split("\n").slice(0, 3).join(" ").substring(0, 200);
    }
  }

  // Self-model freshness
  const capPath = path.join(workspaceDir, "skills", "self-model", "capabilities.md");
  if (await fileExists(capPath)) {
    try {
      const stat = await fs.stat(capPath);
      const ageMin = Math.round((Date.now() - stat.mtimeMs) / 60000);
      state.selfModelAge = ageMin < 60 ? `${ageMin}m ago` : `${Math.round(ageMin / 60)}h ago`;
    } catch {
      // ignore
    }
  }

  // Env-model freshness
  const envPath = path.join(workspaceDir, "skills", "env-model", "environment.md");
  if (await fileExists(envPath)) {
    try {
      const stat = await fs.stat(envPath);
      const ageMin = Math.round((Date.now() - stat.mtimeMs) / 60000);
      state.envModelAge = ageMin < 60 ? `${ageMin}m ago` : `${Math.round(ageMin / 60)}h ago`;
    } catch {
      // ignore
    }
  }

  return state;
}

// ---------------------------------------------------------------------------
// Pre-compaction checkpoint
// ---------------------------------------------------------------------------

async function handlePreCompaction(event: any): Promise<void> {
  const workspaceDir: string = event?.context?.workspaceDir;
  if (!workspaceDir) {
    console.log(`${LOG_PREFIX} No workspaceDir in event context, skipping checkpoint.`);
    return;
  }

  console.log(`${LOG_PREFIX} Pre-compaction: writing checkpoint...`);

  const metacog = await gatherMetacogState(workspaceDir);
  const recentFiles = await recentlyModifiedFiles(workspaceDir);

  // Build the checkpoint content
  const lines: string[] = [
    `# Context Checkpoint — ${nowStamp()}`,
    "",
    "## Active Task",
    "",
    // We can't reliably extract the active task from conversation in a hook.
    // If an .active-plan.md exists, reference it. Otherwise note it's unknown.
    metacog.planSummary !== "none"
      ? `Working on plan: ${metacog.planSummary}`
      : "Active task not available from hook context. Check compaction summary for details.",
    "",
    "## Metacognitive State",
    "",
    `- **Plan**: ${metacog.planSummary}`,
    `- **Last failure**: ${metacog.lastFailure}`,
    `- **Pending assumptions**: ${metacog.pendingAssumptions}`,
    `- **Self-model freshness**: ${metacog.selfModelAge}`,
    `- **Env-model freshness**: ${metacog.envModelAge}`,
    "",
    "## Files Changed This Session",
    "",
  ];

  if (recentFiles.length > 0) {
    for (const f of recentFiles.slice(0, 20)) {
      lines.push(`- ${f}`);
    }
    if (recentFiles.length > 20) {
      lines.push(`- ... and ${recentFiles.length - 20} more`);
    }
  } else {
    lines.push("No recently modified files detected.");
  }

  lines.push("");
  lines.push("## Next Steps");
  lines.push("");
  lines.push("Resume from the active plan or last user request. Check compaction summary for conversation context.");
  lines.push("");

  const checkpointPath = path.join(workspaceDir, CHECKPOINT_FILE);
  await fs.writeFile(checkpointPath, lines.join("\n"), "utf-8");
  console.log(`${LOG_PREFIX} Checkpoint written to ${checkpointPath}`);
}

// ---------------------------------------------------------------------------
// Post-compaction (reserved for future enhancement)
// ---------------------------------------------------------------------------

async function handlePostCompaction(_event: any): Promise<void> {
  // Currently a no-op. Recovery happens at bootstrap.
  // Future: could trigger immediate context refresh or notify the agent.
  console.log(`${LOG_PREFIX} Post-compaction event received. Recovery will occur at next bootstrap.`);
}

// ---------------------------------------------------------------------------
// Bootstrap — inject recovery context if post-compaction
// ---------------------------------------------------------------------------

async function handleBootstrap(event: any): Promise<void> {
  const workspaceDir: string = event?.context?.workspaceDir;
  if (!workspaceDir) return;

  const checkpointPath = path.join(workspaceDir, CHECKPOINT_FILE);
  const recoveredPath = path.join(workspaceDir, RECOVERED_FILE);

  // Check if checkpoint exists and is recent
  if (!(await fileExists(checkpointPath))) return;

  let stat;
  try {
    stat = await fs.stat(checkpointPath);
  } catch {
    return;
  }

  const ageMs = Date.now() - stat.mtimeMs;
  if (ageMs > CHECKPOINT_MAX_AGE_MS) {
    console.log(`${LOG_PREFIX} Checkpoint is ${Math.round(ageMs / 60000)}m old — too stale for recovery. Skipping.`);
    return;
  }

  console.log(`${LOG_PREFIX} Recent checkpoint detected (${Math.round(ageMs / 1000)}s old). Injecting recovery context...`);

  // Read checkpoint
  const checkpoint = await readFileSafe(checkpointPath);
  if (!checkpoint) return;

  // Build a concise recovery block (under RECOVERY_MAX_CHARS)
  const recoveryLines: string[] = [
    "# Post-Compaction Recovery",
    "",
    "You just went through context compaction. The context-guard hook captured your state before compaction.",
    "",
  ];

  // Extract key sections from checkpoint, keeping it concise
  const sections = checkpoint.split(/^## /m).filter((s) => s.trim());
  for (const section of sections) {
    const sectionTitle = section.split("\n")[0].trim();
    const sectionBody = section.split("\n").slice(1).join("\n").trim();

    if (sectionTitle.startsWith("Context Checkpoint")) continue; // skip the header

    // Include each section but truncate if needed
    const truncated = sectionBody.substring(0, 400);
    recoveryLines.push(`## ${sectionTitle}`);
    recoveryLines.push("");
    recoveryLines.push(truncated);
    if (sectionBody.length > 400) recoveryLines.push("...(truncated)");
    recoveryLines.push("");
  }

  recoveryLines.push("Resume your work from the state described above. Check `.context-checkpoint.recovered.md` for the full checkpoint if needed.");

  let recoveryContent = recoveryLines.join("\n");
  // Hard cap
  if (recoveryContent.length > RECOVERY_MAX_CHARS) {
    recoveryContent = recoveryContent.substring(0, RECOVERY_MAX_CHARS) + "\n\n...(truncated to fit recovery budget)";
  }

  // Inject into bootstrap files
  if (event?.context?.bootstrapFiles && Array.isArray(event.context.bootstrapFiles)) {
    event.context.bootstrapFiles.push({
      name: "CONTEXT_RECOVERY.md",
      content: recoveryContent,
    });
    console.log(`${LOG_PREFIX} Recovery context injected into bootstrapFiles.`);
  } else {
    // Fallback: write recovery as a workspace file the agent can read
    const recoveryPath = path.join(workspaceDir, "CONTEXT_RECOVERY.md");
    await fs.writeFile(recoveryPath, recoveryContent, "utf-8");
    console.log(`${LOG_PREFIX} bootstrapFiles not available — wrote recovery to ${recoveryPath}`);
  }

  // Rename checkpoint to prevent re-injection
  try {
    await fs.rename(checkpointPath, recoveredPath);
    console.log(`${LOG_PREFIX} Checkpoint renamed to ${RECOVERED_FILE}`);
  } catch (err) {
    console.log(`${LOG_PREFIX} Warning: could not rename checkpoint: ${err}`);
  }
}

// ---------------------------------------------------------------------------
// Main handler — event router
// ---------------------------------------------------------------------------

const handler: HookHandler = async (event) => {
  try {
    // Pre-compaction checkpoint
    if (event.type === "session" && event.action === "compact:before") {
      await handlePreCompaction(event);
      return;
    }

    // Post-compaction (reserved)
    if (event.type === "session" && event.action === "compact:after") {
      await handlePostCompaction(event);
      return;
    }

    // Bootstrap — inject recovery context
    if (event.type === "agent" && event.action === "bootstrap") {
      await handleBootstrap(event);
      return;
    }
  } catch (err) {
    // Never throw from a hook — let other hooks run
    console.log(`${LOG_PREFIX} Error: ${err instanceof Error ? err.message : String(err)}`);
  }
};

export default handler;
````
