# Bundled Hooks

This directory contains hooks that ship with OpenClaw. These hooks are automatically discovered and can be enabled/disabled via CLI or configuration.

## Available Hooks

### 💾 session-memory

Automatically saves session context to memory when you issue `/new` or `/reset`.

**Events**: `command:new`, `command:reset`
**What it does**: Creates a dated memory file with LLM-generated slug based on conversation content.
**Output**: `<workspace>/memory/YYYY-MM-DD-slug.md` (defaults to `~/.openclaw/workspace`)

**Enable**:

```bash
openclaw hooks enable session-memory
```

### 📎 bootstrap-extra-files

Injects extra bootstrap files (for example monorepo `AGENTS.md`/`TOOLS.md`) during prompt assembly.

**Events**: `agent:bootstrap`
**What it does**: Expands configured workspace glob/path patterns and appends matching bootstrap files to injected context.
**Output**: No files written; context is modified in-memory only.

**Enable**:

```bash
openclaw hooks enable bootstrap-extra-files
```

### 📝 command-logger

Logs all command events to a centralized audit file.

**Events**: `command` (all commands)
**What it does**: Appends JSONL entries to command log file.
**Output**: `~/.openclaw/logs/commands.log`

**Enable**:

```bash
openclaw hooks enable command-logger
```

### 🚀 boot-md

Runs `BOOT.md` whenever the gateway starts (after channels start).

**Events**: `gateway:startup`
**What it does**: Executes BOOT.md instructions via the agent runner.
**Output**: Whatever the instructions request (for example, outbound messages).

**Enable**:

```bash
openclaw hooks enable boot-md
```

## Hook Structure

Each hook is a directory containing:

- **HOOK.md**: Metadata and documentation in YAML frontmatter + Markdown
- **handler.ts**: The hook handler function (default export)

Example structure:

```
session-memory/
├── HOOK.md          # Metadata + docs
└── handler.ts       # Handler implementation
```

## HOOK.md Format

```yaml
---
name: my-hook
description: "Short description"
homepage: https://docs.openclaw.ai/automation/hooks#my-hook
metadata:
  { "openclaw": { "emoji": "🔗", "events": ["command:new"], "requires": { "bins": ["node"] } } }
---
# Hook Title

Documentation goes here...
```

### Metadata Fields

- **emoji**: Display emoji for CLI
- **events**: Array of events to listen for (e.g., `["command:new", "session:start"]`)
- **requires**: Optional requirements
  - **bins**: Required binaries on PATH
  - **anyBins**: At least one of these binaries must be present
  - **env**: Required environment variables
  - **config**: Required config paths (e.g., `["workspace.dir"]`)
  - **os**: Required platforms (e.g., `["darwin", "linux"]`)
- **install**: Installation methods (for bundled hooks: `[{"id":"bundled","kind":"bundled"}]`)

## Creating Custom Hooks

To create your own hooks, place them in:

- **Workspace hooks**: `<workspace>/hooks/` (highest precedence)
- **Managed hooks**: `~/.openclaw/hooks/` (shared across workspaces)

Custom hooks follow the same structure as bundled hooks.

## Managing Hooks

List all hooks:

```bash
openclaw hooks list
```

Show hook details:

```bash
openclaw hooks info session-memory
```

Check hook status:

```bash
openclaw hooks check
```

Enable/disable:

```bash
openclaw hooks enable session-memory
openclaw hooks disable command-logger
```

## Configuration

Hooks can be configured in `~/.openclaw/openclaw.json`:

```json
{
  "hooks": {
    "internal": {
      "enabled": true,
      "entries": {
        "session-memory": {
          "enabled": true
        },
        "command-logger": {
          "enabled": false
        }
      }
    }
  }
}
```

## Event Types

Currently supported events:

- **command**: All command events
- **command:new**: `/new` command specifically
- **command:reset**: `/reset` command
- **command:stop**: `/stop` command
- **agent:bootstrap**: Before workspace bootstrap files are injected
- **gateway:startup**: Gateway startup (after channels start)
- **message**: All message events
- **message:received**: When an inbound message arrives (observational, fire-and-forget)
- **message:sent**: When an outbound message is sent (observational, fire-and-forget)
- **message:enrich**: Inject custom metadata into inbound messages before the agent sees them (see below)

## Handler API

Hook handlers receive an `InternalHookEvent` object:

```typescript
interface InternalHookEvent {
  type: "command" | "session" | "agent" | "gateway" | "message";
  action: string; // e.g., 'new', 'reset', 'stop', 'received', 'enrich'
  sessionKey: string;
  context: Record<string, unknown>;
  timestamp: Date;
  messages: string[]; // Push messages here to send to user
}
```

Example handler:

```typescript
import type { HookHandler } from "../../src/hooks/hooks.js";

const myHandler: HookHandler = async (event) => {
  if (event.type !== "command" || event.action !== "new") {
    return;
  }

  // Your logic here
  console.log("New command triggered!");

  // Optionally send message to user
  event.messages.push("✨ Hook executed!");
};

export default myHandler;
```

## Enrichment Hooks (`message:enrich`)

Enrichment hooks inject custom per-message metadata into the agent's context **without modifying the system prompt** (preserving prompt cache). This is useful for context-aware behavior like:

- Injecting GPS location/velocity so the agent can reply with voice when the user is driving
- Adding calendar availability or device state
- Attaching external sensor data or API results

Unlike regular hooks (fire-and-forget), enrichment handlers **return metadata** that gets merged into the per-message `UntrustedContext` block — the same place conversation info, sender info, and reply context live.

### How it works

1. An inbound message arrives
2. `message:received` fires (observational, as before)
3. `message:enrich` fires — handlers can return `{ metadata: Record<string, unknown> }`
4. Returned metadata is merged and appended to `UntrustedContext`
5. The agent sees the enriched context alongside conversation info

Zero overhead when no enrich hooks are registered (`hasEnrichHooks()` guard skips the entire path).

### Why not system prompt injection?

Workspace context files (AGENTS.md, TOOLS.md, etc.) are injected into the **system prompt**, which is cached across messages. Writing dynamic data there (e.g., current velocity) would change the system prompt on every message, breaking the cache. With models like Opus at $5/MTok input, this gets expensive fast.

Enrichment hooks inject into **user-role context** instead, so the system prompt stays stable and cacheable.

### Example: enrichment hook handler

```typescript
// handler.ts
import type { EnrichHookHandler } from "../../src/hooks/hooks.js";

const handler: EnrichHookHandler = async (event) => {
  // Call any external API, read a file, etc.
  const resp = await fetch("https://my-api.example.com/context");
  const data = await resp.json();

  return {
    metadata: {
      velocity_mps: data.velocity,
      latitude: data.lat,
      longitude: data.lon,
      source: "my-tracker",
    },
  };
};

export default handler;
```

```yaml
# HOOK.md
---
name: my-enrich-hook
description: "Inject custom context into inbound messages"
metadata:
  { "openclaw": { "emoji": "📍", "events": ["message:enrich"] } }
---
```

The agent will see this as an enriched context block in its prompt:

```
Enriched context (hook-injected metadata):
{
  "velocity_mps": 12.5,
  "latitude": 47.70,
  "longitude": -122.20,
  "source": "my-tracker"
}
```

### Configuration

Enable an enrichment hook in `openclaw.json`:

```json
{
  "hooks": {
    "internal": {
      "entries": {
        "my-enrich-hook": { "enabled": true }
      }
    }
  }
}
```

Place the hook in `~/.openclaw/hooks/my-enrich-hook/` or `<workspace>/hooks/my-enrich-hook/`.

## Testing

Test your hooks by:

1. Place hook in workspace hooks directory
2. Restart gateway: `pkill -9 -f 'openclaw.*gateway' && pnpm openclaw gateway`
3. Enable the hook: `openclaw hooks enable my-hook`
4. Trigger the event (e.g., send `/new` command)
5. Check gateway logs for hook execution

## Documentation

Full documentation: https://docs.openclaw.ai/automation/hooks
