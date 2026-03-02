---
name: loom-desktop
description: >
  Build desktop applications where Claude Code CLI (`claude -p`) is the runtime,
  using ElectroBun. The Bun process spawns Claude processes that read files, run
  commands, and return structured output. A native desktop interface renders the
  results. Use when a user wants to build a desktop app powered by Claude — file
  analysis tools, code assistants, document processors, local AI agents with a
  native UI. Triggers: "build a desktop app that uses Claude", "native app with
  Claude", "ElectroBun + Claude", "desktop Claude tool", "local Claude app", or
  any desktop application needing Claude's agentic capabilities through a native
  interface. NOT for web apps (use loom), Anthropic API apps, or chat replicas.
---

# Loom Desktop: Native Apps on the Claude Code Runtime

You're helping someone build a desktop application where Claude Code is the
runtime — not a chat wrapper, but a native tool where Claude reads files, runs
commands, and streams structured output to a local UI.

This is the desktop counterpart to web Loom. Where web Loom puts an Express
server between the browser and Claude, desktop Loom removes that layer entirely.
The Bun process spawns `claude -p` directly and pushes output to the webview via
typed RPC. No HTTP. No SSE formatting. No proxy timeouts. No auth.

Desktop means one user on their own machine. That simplifies everything: no
sessions to manage, no credentials to pass per-request, no rate limiting across
users. The architecture collapses to a thin bridge between subprocess and UI.

## Why Desktop

The question to help users explore is: **what would you build if Claude had full
access to your local filesystem, ran as a native app on your dock, and could
stream its reasoning to a custom interface — with zero server infrastructure?**

A desktop Loom app has things web apps don't:

- **Full local file access** — Claude reads and writes files on the user's machine directly. No upload step.
- **Native UX** — System tray, native menus, file dialogs, drag-and-drop. Feels like a real tool, not a browser tab.
- **Offline-first** — The app launches instantly. Claude needs network for inference, but the interface itself has no server dependency.
- **No infrastructure** — No VM to provision, no server to keep running, no auth to maintain. Ship a binary.
- **Privacy** — Files never leave the machine. Claude processes them locally via the CLI.

## Prerequisites

Before building, you need:

- **Claude CLI** installed and on PATH — `claude --version` should return a version string
- **Bun** v1.1+ — [bun.sh](https://bun.sh)
- **ElectroBun** — `bun add electrobun` — [github.com/blackboardsh/electrobun](https://github.com/blackboardsh/electrobun)
- **Platform**: macOS 14+, Windows 11+, or Ubuntu 22.04+ (other Linux with gtk3 & webkit2gtk-4.1)

ElectroBun is currently at v1.14.4 stable (v1.14.5-beta.0 in beta). It's
actively maintained but still evolving — check the issue tracker for
platform-specific concerns.

## The Architecture: Thin Bridge

Every Loom desktop app has this shape:

```
┌─────────────────────────────────────────────────────┐
│                    ElectroBun App                     │
│                                                       │
│  ┌──────────────┐    Typed RPC     ┌──────────────┐  │
│  │   Webview     │◄───messages────►│  Bun Process  │  │
│  │  (React/HTML) │                 │               │  │
│  │               │  rpc.request    │  Claude       │  │
│  │  Renders UI   │──startTask()──►│  Manager      │  │
│  │  Shows tokens │  rpc.request    │               │  │
│  │  Tool status  │──abort()──────►│  Spawns       │  │
│  │               │                 │  claude -p    │  │
│  │               │◄─rpc.send      │               │  │
│  │               │  .token()       │  Parses       │  │
│  │               │  .toolUse()     │  stdout       │  │
│  │               │  .done()        │  stream       │  │
│  │               │  .error()       │               │  │
│  │               │  .status()      │               │  │
│  └──────────────┘                 └──────┬───────┘  │
│                                          │           │
│                                    stdio │           │
│                                          ▼           │
│                                   ┌──────────┐       │
│                                   │ claude -p │       │
│                                   │ subprocess│       │
│                                   └──────────┘       │
└─────────────────────────────────────────────────────┘
```

**Webview**: The custom UI. React, vanilla HTML, whatever fits the app. Runs in
a system webview (WKWebView on macOS, WebView2 on Windows, WebKit2GTK on Linux).

**Bun Process**: The bridge. Receives requests from the webview via typed RPC,
spawns `claude -p` subprocesses, parses their NDJSON stdout, and pushes results
back as typed RPC messages. This is the Claude Manager — typically 80-100 lines
of TypeScript.

**Claude Subprocess**: The intelligence. `claude -p` with the right flags. This
is where the agentic work happens — reading files, running commands, generating
structured output.

### Why RPC Instead of HTTP

Web Loom uses Express + SSE because browsers need HTTP. Desktop Loom skips that:

| Concern | Web Loom (HTTP/SSE) | Desktop Loom (RPC) |
|---------|--------------------|--------------------|
| Transport | HTTP + SSE streams | WebSocket (encrypted) |
| Type safety | Manual JSON parsing | Compile-time typed schema |
| Proxy timeouts | Heartbeat hacks needed | Not applicable |
| Auth | OAuth per-request | Not needed (single user) |
| Complexity | ~200 lines of plumbing | ~20 lines of plumbing |

The typed RPC contract is defined once and shared between Bun and webview. See
`@references/rpc-schema-reference.md` for the full schema.

## Project Setup

Scaffold an ElectroBun project:

```bash
npx electrobun init hello-world
cd myapp
bun install
electrobun dev
```

This gives you a running app with a Bun process and webview. From here, you'll
add the Claude Manager to the Bun side and the task UI to the webview.

For the full setup guide — project structure, configuration, dev workflow — see
`@references/electrobun-setup.md`.

For Claude CLI flags and output formats, see `@references/cli-runtime-reference.md`.

## The Conversation

When someone comes to you with a desktop app idea, walk through these design
questions. Don't dump them all at once — have a natural conversation. But cover
this ground before you start building.

### 1. What does this app do?

Get concrete about the interface. What does someone see when they open it? What
do they click? What appears when Claude is working? What does the output look
like?

Desktop apps have a stronger "tool" identity than web apps. A web app might be a
dashboard someone visits. A desktop app is something someone installs and
reaches for. What's the tool? What problem does it solve?

Ask: "Walk me through opening this app. What do you see? What do you do first?"

### 2. What Claude capabilities does it need?

Map the app's features to what Claude does behind the scenes:

| Need | Claude Configuration |
|------|---------------------|
| Read and analyze files | `--tools "Read,Glob,Grep"` |
| Modify code or files | `--tools "Read,Edit,Write,Glob,Grep,Bash"` |
| Pure reasoning, no file access | `--tools ""` |
| Web research | `--tools "WebSearch,WebFetch"` |
| Structured data extraction | `--json-schema '{...}'` |
| Custom persona | `--system-prompt "You are..."` |

The tools list determines what Claude can do. Tighter is safer — only grant what
the app actually needs.

### 3. What's the interaction model?

How does the person's action translate to a Claude invocation?

| Interaction | Pattern |
|-------------|---------|
| Click a button, get a result | **Synchronous** — spawn, collect, return |
| Watch progress in real-time | **Streaming** — tokens flow as RPC messages |
| Multi-step conversation with context | **Conversational** — `--session-id` + `--continue` |
| Long task, minimize to tray | **Background** — task registry + tray status |

Most desktop apps start with Streaming. Add Conversational if the user needs
follow-up questions. Add Background for tasks that take minutes.

### 4. What desktop features matter?

This is where desktop Loom diverges from web Loom. Ask which native features
the app needs:

- **File drag-and-drop** — Drop files onto the window to feed them to Claude
- **Native file dialogs** — "Open File..." to select input, save results to disk
- **System tray** — Minimize during long tasks, show progress in tooltip
- **Native menus** — App menu bar with New Task, Abort, Model Selection, Settings
- **Notifications** — Native OS notification when a background task completes

Not every app needs all of these. A simple analysis tool might only need
drag-and-drop. A code assistant might need the full set.

### 5. What are the cost and safety boundaries?

Desktop simplifies auth (no OAuth needed) but cost and permissions still matter:

- **Budget**: `--max-budget-usd` caps spending per task. What's appropriate? $0.50 for quick analysis, $5 for deep codebase review.
- **Model**: Haiku for fast/cheap, Sonnet for balanced, Opus for best quality. `--fallback-model haiku` for reliability.
- **Permissions**: Use `--permission-mode dontAsk` with explicit `--tools` list. This auto-denies anything not whitelisted.
- **Turn limit**: `--max-turns` prevents runaway agent loops.
- **Filesystem scope**: Consider constraining Claude to specific directories via `--allowedTools "Read(/path/**)"` patterns.

## Building It

Default to **TypeScript** for both the Bun process and the webview. Use
**React** for the UI unless the person prefers vanilla HTML.

### Shared Utilities

Every pattern below uses these helpers. Define them once in a shared module
(e.g., `src/bun/claude-manager.ts`).

**`cleanEnv()`** — Remove nesting guards so `claude -p` can start.

When you develop inside Claude Code (which is common), two environment
variables — `CLAUDECODE` and `CLAUDE_CODE_ENTRYPOINT` — tell Claude it's
already running and block nested processes. Remove exactly these two. Do NOT
filter all `CLAUDE*` vars — that kills auth tokens
(`CLAUDE_CODE_OAUTH_TOKEN`) and feature flags.

If running inside **cmux** (the Claude terminal app), `CMUX_*` surface
identifiers trigger nesting detection. These are terminal-state vars, not
auth — safe to remove.

```typescript
function cleanEnv(): NodeJS.ProcessEnv {
  const env = { ...process.env };
  delete env.CLAUDECODE;
  delete env.CLAUDE_CODE_ENTRYPOINT;
  // cmux terminal sets CMUX_* vars that trigger nesting detection.
  // These are terminal-state identifiers, not auth tokens — safe to remove.
  if (env.CMUX_SURFACE_ID) {
    delete env.CMUX_SURFACE_ID;
    delete env.CMUX_PANEL_ID;
    delete env.CMUX_TAB_ID;
    delete env.CMUX_WORKSPACE_ID;
    delete env.CMUX_SOCKET_PATH;
  }
  return env;
}
```

**`createStreamParser()`** — Buffer stdout chunks into complete JSON lines.

TCP delivers data in arbitrary chunks. A JSON line can split across two read
events. Without buffering, the first half fails `JSON.parse` and gets silently
discarded.

```typescript
function createStreamParser(onEvent: (event: any) => void) {
  const decoder = new TextDecoder();
  let buffer = "";

  return (chunk: Buffer | Uint8Array) => {
    buffer += decoder.decode(chunk, { stream: true });
    const lines = buffer.split("\n");
    buffer = lines.pop() || "";
    for (const line of lines) {
      if (!line.trim()) continue;
      try {
        onEvent(JSON.parse(line));
      } catch (err) {
        console.warn("[claude stdout] JSON parse error:", (err as Error).message, line.slice(0, 200));
      }
    }
  };
}
```

Use `TextDecoder` with `{ stream: true }` — not `chunk.toString()` — to handle
multi-byte UTF-8 characters that split across chunk boundaries.

### Event Type Mapping

`claude -p --output-format stream-json --verbose` emits newline-delimited JSON.
Each event maps to an RPC message:

| Stream Event | RPC Message | Notes |
|---|---|---|
| `system` (subtype: init) | `status` (state: "running") | Session started |
| `assistant` (text content block) | `token` | Complete text for thinking models |
| `stream_event` (content_block_delta) | `token` | Incremental for non-thinking models |
| `assistant` (tool_use content block) | `toolUse` | Tool invocation |
| `tool_result` | `toolResult` | Tool output |
| `result` (subtype: success) | `done` | Session complete with cost |
| parse error / process crash | `error` | Process failure |

### `deriveAndSendRPC()`

This function reads a parsed stream event and fires the appropriate RPC message.
It also tracks state for the heartbeat system.

```typescript
// State for heartbeat
let currentState: "spawning" | "running" | "thinking" | "tool_use" | "idle" = "spawning";
let lastToolName = "";
let lastOutputTime = Date.now();

function deriveAndSendRPC(
  taskId: string,
  event: any,
  rpc: { send: LoomRPC["webview"]["messages"] }
) {
  lastOutputTime = Date.now();

  switch (event.type) {
    case "system":
      currentState = "running";
      break;

    case "assistant": {
      const msg = event.message;
      if (!msg?.content) break;
      for (const block of msg.content) {
        if (block.type === "text") {
          rpc.send.token({ taskId, text: block.text });
          currentState = "running";
        } else if (block.type === "tool_use") {
          rpc.send.toolUse({ taskId, tool: block.name, input: block.input });
          currentState = "tool_use";
          lastToolName = block.name;
        }
      }
      break;
    }

    case "stream_event": {
      const delta = event.event?.delta;
      if (delta?.text) {
        rpc.send.token({ taskId, text: delta.text });
        currentState = "running";
      }
      if (delta?.type === "input_json_delta") {
        // Tool input streaming — ignore, wait for full assistant message
      }
      break;
    }

    case "tool_result": {
      const content = event.content ?? event.message?.content;
      const text = typeof content === "string"
        ? content
        : Array.isArray(content)
          ? content.map((b: any) => b.text ?? "").join("")
          : JSON.stringify(content);
      rpc.send.toolResult({
        taskId,
        tool: lastToolName,
        output: text.slice(0, 10000),
        isError: !!event.is_error,
      });
      currentState = "running";
      break;
    }

    case "result":
      rpc.send.done({
        taskId,
        cost: event.total_cost_usd ?? 0,
        duration: event.duration_ms ?? 0,
      });
      currentState = "idle";
      break;
  }
}
```

## Patterns

### Pattern 1: Synchronous (Quick Extraction)

The simplest pattern. The webview sends a `startTask` request, Bun spawns
Claude, collects the full output, and returns the result. No streaming — the
webview waits for the response.

Use this for fast, structured extraction: pull metadata from a file, classify
a document, extract TODOs from a codebase.

**Bun-side handler:**

```typescript
import { BrowserView } from "electrobun/bun";

const rpc = BrowserView.defineRPC<LoomRPC>({
  handlers: {
    startTask: async ({ prompt, model, tools, maxBudget }) => {
      const taskId = crypto.randomUUID();

      const schema = JSON.stringify({
        type: "object",
        properties: {
          result: { type: "string" },
          items: { type: "array", items: { type: "object" } },
        },
        required: ["result"],
      });

      try {
        const proc = Bun.spawnSync([
          "claude", "-p",
          "--output-format", "json",
          "--permission-mode", "dontAsk",
          "--tools", (tools || []).join(","),
          "--model", model || "sonnet",
          "--max-budget-usd", String(maxBudget || 1),
          "--json-schema", schema,
          "--no-session-persistence",
          prompt,
        ], { env: cleanEnv(), timeout: 60000 });

        if (proc.exitCode !== 0) {
          const stderr = proc.stderr.toString().trim();
          rpc.send.error({ taskId, message: stderr || `Exit code ${proc.exitCode}` });
          return { taskId };
        }

        const parsed = JSON.parse(proc.stdout.toString());
        if (parsed.is_error) {
          rpc.send.error({ taskId, message: parsed.result });
        } else {
          rpc.send.token({ taskId, text: parsed.result });
          rpc.send.done({
            taskId,
            cost: parsed.total_cost_usd ?? 0,
            duration: parsed.duration_ms ?? 0,
          });
        }
      } catch (err: any) {
        rpc.send.error({ taskId, message: err.message });
      }

      return { taskId };
    },
    abort: async () => ({ success: false }), // Can't abort sync
  },
});
```

The `--json-schema` flag gives you typed `structured_output` alongside the
narrative `result`. Design the schema to match the UI components you want to
render — the schema IS your API contract between Claude and the frontend.

**Don't do this:**
- Don't use string interpolation to build the command — use an args array.
  Shell strings break on special characters and are injection-vulnerable.
- Don't read `structured_output` without checking `is_error` first — when
  Claude hits a budget cap or tool failure, `structured_output` is `null`.

### Pattern 2: Streaming (Primary Pattern)

The Thin Bridge. This is the core pattern for most Loom desktop apps. Tokens
flow from Claude's stdout through the Bun process to the webview as typed RPC
messages, with status heartbeats every 2 seconds.

**Bun-side: Claude Manager**

The complete `spawnClaude()` function. This is the heart of the bridge:

```typescript
import { BrowserView } from "electrobun/bun";

// Active tasks for abort support
const activeTasks = new Map<string, {
  proc: ReturnType<typeof Bun.spawn>;
  heartbeat: ReturnType<typeof setInterval>;
}>();

function spawnClaude(
  taskId: string,
  prompt: string,
  opts: {
    model?: string;
    tools?: string[];
    maxBudget?: number;
    sessionId?: string;
  },
  rpc: any
) {
  // Heartbeat state
  let currentState: LoomRPC["webview"]["messages"]["status"]["state"] = "spawning";
  let lastToolName = "";
  let lastOutputTime = Date.now();
  const startTime = Date.now();

  // Build args
  const args = [
    "-p",
    "--output-format", "stream-json", "--verbose",
    "--permission-mode", "dontAsk",
    "--tools", (opts.tools || []).join(","),
    "--model", opts.model || "sonnet",
    "--max-budget-usd", String(opts.maxBudget || 1),
    "--no-session-persistence",
  ];
  if (opts.sessionId) {
    args.push("--session-id", opts.sessionId, "--continue");
  }
  args.push(prompt);

  // Spawn
  const proc = Bun.spawn(["claude", ...args], {
    stdout: "pipe",
    stderr: "pipe",
    env: cleanEnv(),
  });

  // Status heartbeat — every 2 seconds while the task runs
  const heartbeat = setInterval(() => {
    rpc.send.status({
      taskId,
      state: currentState,
      detail: currentState === "tool_use" ? lastToolName : undefined,
      elapsedMs: Date.now() - startTime,
      lastActivityMs: Date.now() - lastOutputTime,
    });
  }, 2000);

  // Track for abort
  activeTasks.set(taskId, { proc, heartbeat });

  // Parse stdout stream
  const parse = createStreamParser((event) => {
    lastOutputTime = Date.now();

    switch (event.type) {
      case "system":
        currentState = "running";
        break;

      case "assistant": {
        const msg = event.message;
        if (!msg?.content) break;
        for (const block of msg.content) {
          if (block.type === "text") {
            rpc.send.token({ taskId, text: block.text });
            currentState = "running";
          } else if (block.type === "tool_use") {
            rpc.send.toolUse({ taskId, tool: block.name, input: block.input });
            currentState = "tool_use";
            lastToolName = block.name;
          }
        }
        break;
      }

      case "stream_event": {
        const delta = event.event?.delta;
        if (delta?.text) {
          rpc.send.token({ taskId, text: delta.text });
          currentState = "running";
        }
        break;
      }

      case "tool_result": {
        const content = event.content ?? event.message?.content;
        const text = typeof content === "string"
          ? content
          : Array.isArray(content)
            ? content.map((b: any) => b.text ?? "").join("")
            : JSON.stringify(content);
        rpc.send.toolResult({
          taskId,
          tool: lastToolName,
          output: text.slice(0, 10000),
          isError: !!event.is_error,
        });
        currentState = "running";
        break;
      }

      case "result":
        rpc.send.done({
          taskId,
          cost: event.total_cost_usd ?? 0,
          duration: event.duration_ms ?? 0,
        });
        cleanup();
        break;
    }
  });

  function cleanup() {
    clearInterval(heartbeat);
    activeTasks.delete(taskId);
    currentState = "idle";
  }

  // Pump stdout reader
  (async () => {
    const reader = proc.stdout.getReader();
    try {
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        parse(value);
      }
    } catch (err) {
      // Reader closed — process exited
    }

    // Process finished — check for errors
    const exitCode = await proc.exited;
    if (exitCode !== 0 && currentState !== "idle") {
      const stderr = await new Response(proc.stderr).text();
      rpc.send.error({
        taskId,
        message: stderr.trim() || `Claude exited with code ${exitCode}`,
      });
    }
    cleanup();
  })();

  return proc;
}
```

**Bun-side: Request handlers**

Wire the RPC to `spawnClaude`:

```typescript
const rpc = BrowserView.defineRPC<LoomRPC>({
  handlers: {
    startTask: async ({ prompt, model, tools, maxBudget, sessionId }) => {
      const taskId = crypto.randomUUID();
      spawnClaude(taskId, prompt, { model, tools, maxBudget, sessionId }, rpc);
      return { taskId };
    },
    abort: async ({ taskId }) => {
      const task = activeTasks.get(taskId);
      if (!task) return { success: false };

      task.proc.kill("SIGTERM");
      clearInterval(task.heartbeat);
      activeTasks.delete(taskId);
      rpc.send.error({ taskId, message: "Task aborted by user" });
      return { success: true };
    },
  },
});
```

**Webview-side: Receiving messages**

The frontend registers handlers for each message type. Here's a minimal React
component that renders streaming tokens with status:

```tsx
import { useState, useEffect, useRef } from "react";
import { Electroview } from "electrobun/view";

const electrobun = new Electroview<LoomRPC>();

function App() {
  const [output, setOutput] = useState("");
  const [status, setStatus] = useState<string>("idle");
  const [taskId, setTaskId] = useState<string | null>(null);
  const [cost, setCost] = useState<number | null>(null);
  const [tools, setTools] = useState<string[]>([]);
  const outputRef = useRef<HTMLPreElement>(null);

  useEffect(() => {
    electrobun.rpc.on.token(({ taskId: tid, text }) => {
      setOutput((prev) => prev + text);
    });

    electrobun.rpc.on.toolUse(({ tool }) => {
      setTools((prev) => [...prev, tool]);
    });

    electrobun.rpc.on.toolResult(({ tool, isError }) => {
      setTools((prev) => prev.filter((t) => t !== tool));
    });

    electrobun.rpc.on.status(({ state, detail, elapsedMs }) => {
      const secs = Math.round(elapsedMs / 1000);
      setStatus(detail ? `${state}: ${detail} (${secs}s)` : `${state} (${secs}s)`);
    });

    electrobun.rpc.on.done(({ cost: c, duration }) => {
      setCost(c);
      setStatus(`Done in ${Math.round(duration / 1000)}s`);
      setTaskId(null);
    });

    electrobun.rpc.on.error(({ message }) => {
      setStatus(`Error: ${message}`);
      setTaskId(null);
    });
  }, []);

  useEffect(() => {
    // Auto-scroll output
    if (outputRef.current) {
      outputRef.current.scrollTop = outputRef.current.scrollHeight;
    }
  }, [output]);

  async function runTask(prompt: string) {
    setOutput("");
    setCost(null);
    setStatus("spawning...");
    const { taskId: tid } = await electrobun.rpc.request.startTask({
      prompt,
      model: "sonnet",
      tools: ["Read", "Glob", "Grep"],
      maxBudget: 1,
    });
    setTaskId(tid);
  }

  async function abort() {
    if (taskId) {
      await electrobun.rpc.request.abort({ taskId });
    }
  }

  return (
    <div className="app">
      <header>
        <input
          type="text"
          placeholder="What should Claude do?"
          onKeyDown={(e) => {
            if (e.key === "Enter") runTask(e.currentTarget.value);
          }}
          disabled={!!taskId}
        />
        <button onClick={abort} disabled={!taskId}>Abort</button>
      </header>

      <div className="status-bar">
        {status}
        {tools.length > 0 && <span className="tools">Using: {tools.join(", ")}</span>}
        {cost !== null && <span className="cost">${cost.toFixed(4)}</span>}
      </div>

      <pre className="output" ref={outputRef}>{output}</pre>
    </div>
  );
}

export default App;
```

**Startup CLI check**

On app launch, verify Claude is accessible before showing the main UI:

```typescript
// In src/bun/index.ts, before creating the window:
function checkClaudeCLI(): boolean {
  try {
    const result = Bun.spawnSync(["claude", "--version"]);
    return result.exitCode === 0;
  } catch {
    return false;
  }
}

const claudeAvailable = checkClaudeCLI();

const win = new BrowserWindow({
  title: "My Claude App",
  frame: { width: 1200, height: 800 },
  url: claudeAvailable
    ? "views://mainview/index.html"
    : "views://mainview/setup.html",  // Show install instructions
  rpc,
});
```

If Claude isn't found, show a setup screen with install instructions. Don't
silently fail — the user needs to know why the app isn't working.

### Pattern 3: Conversational (Multi-Turn)

Multi-turn sessions where Claude retains context across messages. Uses
`--session-id` and `--continue` flags. The Bun process maintains a session map.

Use this for: code assistants where you ask follow-up questions, interactive
debugging sessions, multi-step analysis that builds on previous answers.

**Bun-side: Session management**

```typescript
// Session registry
const sessions = new Map<string, {
  sessionId: string;
  turns: number;
  lastActivity: number;
}>();

// Clean up stale sessions periodically (optional)
setInterval(() => {
  const staleMs = 30 * 60 * 1000; // 30 minutes
  const now = Date.now();
  for (const [key, session] of sessions) {
    if (now - session.lastActivity > staleMs) {
      sessions.delete(key);
    }
  }
}, 60000);
```

**Bun-side: Conversational request handler**

Replace the `startTask` handler with session awareness:

```typescript
const rpc = BrowserView.defineRPC<LoomRPC>({
  handlers: {
    startTask: async ({ prompt, model, tools, maxBudget, sessionId }) => {
      const taskId = crypto.randomUUID();

      let actualSessionId: string;
      if (sessionId && sessions.has(sessionId)) {
        // Follow-up turn — reuse session
        actualSessionId = sessionId;
        const session = sessions.get(sessionId)!;
        session.turns++;
        session.lastActivity = Date.now();
      } else {
        // First turn — create new session
        actualSessionId = crypto.randomUUID();
        sessions.set(actualSessionId, {
          sessionId: actualSessionId,
          turns: 1,
          lastActivity: Date.now(),
        });
      }

      // spawnClaude already handles --session-id and --continue
      // when opts.sessionId is provided
      spawnClaude(taskId, prompt, {
        model, tools, maxBudget,
        sessionId: actualSessionId,
      }, rpc);

      return { taskId };
    },
    abort: async ({ taskId }) => {
      const task = activeTasks.get(taskId);
      if (!task) return { success: false };
      task.proc.kill("SIGTERM");
      clearInterval(task.heartbeat);
      activeTasks.delete(taskId);
      rpc.send.error({ taskId, message: "Task aborted by user" });
      return { success: true };
    },
  },
});
```

Note: `spawnClaude` from Pattern 2 already appends `--session-id <id>
--continue` when `opts.sessionId` is provided. The first turn omits
`--continue` — set that logic in `spawnClaude` by checking if the session
has prior turns.

**Webview-side: Chat UI**

A minimal chat component that maintains conversation context:

```tsx
import { useState, useEffect, useRef } from "react";
import { Electroview } from "electrobun/view";

const electrobun = new Electroview<LoomRPC>();

type Message = {
  role: "user" | "assistant";
  content: string;
  taskId?: string;
};

function ChatApp() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState("");
  const [sessionId, setSessionId] = useState<string | null>(null);
  const [streaming, setStreaming] = useState(false);
  const messagesEnd = useRef<HTMLDivElement>(null);

  useEffect(() => {
    electrobun.rpc.on.token(({ taskId, text }) => {
      setMessages((prev) => {
        const last = prev[prev.length - 1];
        if (last?.taskId === taskId && last.role === "assistant") {
          return [...prev.slice(0, -1), { ...last, content: last.content + text }];
        }
        return [...prev, { role: "assistant", content: text, taskId }];
      });
    });

    electrobun.rpc.on.done(() => setStreaming(false));
    electrobun.rpc.on.error(({ message }) => {
      setStreaming(false);
      setMessages((prev) => [...prev, { role: "assistant", content: `Error: ${message}` }]);
    });
  }, []);

  useEffect(() => {
    messagesEnd.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages]);

  async function send() {
    if (!input.trim() || streaming) return;
    const prompt = input.trim();
    setInput("");
    setStreaming(true);

    setMessages((prev) => [...prev, { role: "user", content: prompt }]);

    const { taskId } = await electrobun.rpc.request.startTask({
      prompt,
      model: "sonnet",
      tools: ["Read", "Glob", "Grep", "Bash"],
      maxBudget: 2,
      sessionId: sessionId ?? undefined,
    });

    // After first turn, store session ID for follow-ups
    if (!sessionId) setSessionId(taskId); // Use taskId as proxy, or track via RPC
  }

  return (
    <div className="chat">
      <div className="messages">
        {messages.map((msg, i) => (
          <div key={i} className={`message ${msg.role}`}>
            <pre>{msg.content}</pre>
          </div>
        ))}
        <div ref={messagesEnd} />
      </div>
      <div className="input-row">
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => e.key === "Enter" && send()}
          placeholder="Ask a follow-up question..."
          disabled={streaming}
        />
        <button onClick={send} disabled={streaming}>Send</button>
      </div>
    </div>
  );
}
```

**Don't do this:**
- Don't spawn concurrent Claude processes on the same session — one at a time.
  `--continue` resumes the most recent turn; parallel spawns create race
  conditions.
- Don't forget `--continue` on follow-up turns. Without it, Claude starts a
  fresh conversation even with the same `--session-id`.

### Pattern 4: Background (Long-Running)

For tasks that take minutes — analyzing an entire codebase, batch-processing
documents, comprehensive code review. The user can navigate away or minimize
to the system tray. Status messages continue flowing.

**Bun-side: Task registry**

```typescript
type TaskRecord = {
  proc: ReturnType<typeof Bun.spawn>;
  heartbeat: ReturnType<typeof setInterval>;
  status: "running" | "completed" | "error";
  result?: { cost: number; duration: number };
  error?: string;
  startTime: number;
};

const taskRegistry = new Map<string, TaskRecord>();
```

Spawning is identical to Pattern 2 — the difference is lifecycle management.
Background tasks survive webview navigation and can be listed, polled, or
cancelled from any view.

**Bun-side: Task list handler**

Add a request to list active tasks (extend the RPC schema):

```typescript
// Add to your RPC schema's bun.requests:
listTasks: {
  params: {};
  response: {
    tasks: Array<{
      taskId: string;
      status: "running" | "completed" | "error";
      elapsedMs: number;
    }>;
  };
};
```

```typescript
// In the handler:
listTasks: async () => {
  const tasks = Array.from(taskRegistry.entries()).map(([taskId, record]) => ({
    taskId,
    status: record.status,
    elapsedMs: Date.now() - record.startTime,
  }));
  return { tasks };
},
```

**System tray integration**

Minimize to the system tray during long tasks. Show progress in the tooltip:

```typescript
import { Tray } from "electrobun/bun";

const tray = new Tray({
  title: "Claude",
  image: "views://assets/tray-icon.png",
  width: 22,
  height: 22,
});

tray.setMenu([
  { label: "Show Window", action: "show" },
  { type: "separator" },
  { label: "No active tasks", action: "status", enabled: false },
  { type: "separator" },
  { label: "Quit", role: "quit" },
]);

// Listen for tray clicks
tray.on("tray-clicked", () => {
  win.focus();
});

tray.on("tray-item-clicked", (e) => {
  if (e.data.action === "show") win.focus();
});

// Update tray when tasks change
function updateTrayStatus() {
  const running = Array.from(taskRegistry.values())
    .filter((t) => t.status === "running");

  if (running.length === 0) {
    tray.setMenu([
      { label: "Show Window", action: "show" },
      { type: "separator" },
      { label: "No active tasks", action: "status", enabled: false },
      { type: "separator" },
      { label: "Quit", role: "quit" },
    ]);
  } else {
    const items = running.map((t, i) => ({
      label: `Task ${i + 1}: ${Math.round((Date.now() - t.startTime) / 1000)}s`,
      action: `task-${i}`,
      enabled: false,
    }));
    tray.setMenu([
      { label: "Show Window", action: "show" },
      { type: "separator" },
      ...items,
      { type: "separator" },
      { label: "Quit", role: "quit" },
    ]);
  }
}
```

**Completion notification**

Send a native notification when a background task finishes:

```typescript
import { Utils } from "electrobun/bun";

// In the "result" handler of spawnClaude, after cleanup:
Utils.showNotification({
  title: "Task Complete",
  body: `Finished in ${Math.round(duration / 1000)}s — $${cost.toFixed(4)}`,
});
updateTrayStatus();
```

**Webview-side: Task list component**

```tsx
function TaskList() {
  const [tasks, setTasks] = useState<Array<{
    taskId: string;
    status: string;
    elapsedMs: number;
  }>>([]);

  async function refresh() {
    const { tasks } = await electrobun.rpc.request.listTasks({});
    setTasks(tasks);
  }

  useEffect(() => {
    refresh();
    const interval = setInterval(refresh, 3000);
    return () => clearInterval(interval);
  }, []);

  return (
    <div className="task-list">
      <h3>Background Tasks</h3>
      {tasks.length === 0 && <p>No active tasks</p>}
      {tasks.map((task) => (
        <div key={task.taskId} className={`task ${task.status}`}>
          <span className="task-id">{task.taskId.slice(0, 8)}</span>
          <span className="task-status">{task.status}</span>
          <span className="task-time">{Math.round(task.elapsedMs / 1000)}s</span>
          {task.status === "running" && (
            <button onClick={() => electrobun.rpc.request.abort({ taskId: task.taskId })}>
              Cancel
            </button>
          )}
        </div>
      ))}
    </div>
  );
}
```

## Desktop Features

These features distinguish desktop Loom from web Loom. Each one is optional —
pick what the app needs.

### File Drag-and-Drop

Users drop files onto the window. ElectroBun provides the file paths. The Bun
process feeds them to Claude as context.

**Webview-side: Drop handler**

```tsx
function DropZone({ onFilesDropped }: { onFilesDropped: (paths: string[]) => void }) {
  const [dragging, setDragging] = useState(false);

  return (
    <div
      className={`drop-zone ${dragging ? "active" : ""}`}
      onDragOver={(e) => { e.preventDefault(); setDragging(true); }}
      onDragLeave={() => setDragging(false)}
      onDrop={(e) => {
        e.preventDefault();
        setDragging(false);
        const paths = Array.from(e.dataTransfer.files).map((f) => f.path);
        if (paths.length > 0) onFilesDropped(paths);
      }}
    >
      Drop files here to analyze
    </div>
  );
}
```

**Bun-side: Incorporating dropped files into prompts**

When the webview sends file paths via `startTask`, build a prompt that
references them:

```typescript
// In the startTask handler:
const fileContext = paths.length > 0
  ? `\n\nAnalyze these files:\n${paths.map((p) => `- ${p}`).join("\n")}`
  : "";
const fullPrompt = prompt + fileContext;
```

Or for read-only analysis, use `--allowedTools "Read(${path})"` patterns to
scope Claude's access to only the dropped files.

### Native File Dialogs

Open files for input, save results to disk.

**Bun-side: Open file dialog**

```typescript
import { Utils } from "electrobun/bun";

// Add to RPC schema bun.requests:
// openFile: { params: {}; response: { path: string | null } };

// Handler:
openFile: async () => {
  const result = await Utils.openFileDialog({
    title: "Select a file to analyze",
    filters: [
      { name: "All Files", extensions: ["*"] },
      { name: "Code", extensions: ["ts", "js", "py", "rs", "go"] },
      { name: "Documents", extensions: ["md", "txt", "pdf"] },
    ],
  });
  return { path: result ?? null };
},
```

**Saving results to disk**

ElectroBun doesn't have `saveFileDialog()` yet. Use `Bun.write()` with
a known path, or ask the user for a filename via RPC:

```typescript
// Add to RPC schema bun.requests:
// saveResult: { params: { content: string; filename: string }; response: { saved: boolean } };

// Handler:
saveResult: async ({ content, filename }) => {
  const desktopPath = `${process.env.HOME}/Desktop/${filename}`;
  await Bun.write(desktopPath, content);
  return { saved: true };
},
```

For a better UX, use `Utils.showItemInFolder()` after saving to open Finder
with the file highlighted:

```typescript
await Bun.write(desktopPath, content);
Utils.showItemInFolder(desktopPath);
```

### System Tray

Already shown in Pattern 4 above. Key points:

- `new Tray({ title, image, width, height })` — create tray icon
- `tray.setMenu(items)` — update menu items dynamically
- `tray.on("tray-clicked", handler)` — handle tray icon click
- `tray.on("tray-item-clicked", handler)` — handle menu item click
- Use `views://` scheme for icon paths to reference bundled assets

### Native Menus

Define the app menu bar with keyboard shortcuts:

```typescript
import { ApplicationMenu } from "electrobun/bun";

ApplicationMenu.setApplicationMenu([
  {
    label: "File",
    submenu: [
      { label: "New Task", action: "new-task", accelerator: "n" },
      { label: "Open File...", action: "open-file", accelerator: "o" },
      { type: "separator" },
      { label: "Quit", role: "quit" },
    ],
  },
  {
    label: "Edit",
    submenu: [
      { label: "Undo", role: "undo" },
      { label: "Redo", role: "redo" },
      { type: "separator" },
      { label: "Cut", role: "cut" },
      { label: "Copy", role: "copy" },
      { label: "Paste", role: "paste" },
      { label: "Select All", role: "selectAll" },
    ],
  },
  {
    label: "Task",
    submenu: [
      { label: "Abort Current", action: "abort", accelerator: "." },
      { type: "separator" },
      { label: "Model: Haiku", action: "model-haiku" },
      { label: "Model: Sonnet", action: "model-sonnet", checked: true },
      { label: "Model: Opus", action: "model-opus" },
    ],
  },
]);

// Handle menu actions
ApplicationMenu.on("application-menu-clicked", (e) => {
  switch (e.data.action) {
    case "new-task":
      // Focus the input field via RPC or reload the view
      break;
    case "open-file":
      // Trigger file dialog
      break;
    case "abort":
      // Abort current task
      break;
    case "model-haiku":
    case "model-sonnet":
    case "model-opus":
      // Update model preference
      break;
  }
});
```

The `accelerator` property maps to Cmd+key on macOS and Ctrl+key on Windows.
Single-character accelerators are supported on all platforms. The `role` property
provides built-in OS clipboard and window management.

Note: Application menu is fully supported on macOS. Windows supports basic
accelerators. Linux does not currently support application menus.

### Local File Access Configuration

Claude's tool access determines what it can do on the user's machine. Match
the permission set to the app's purpose:

| Use Case | `--tools` Value | Notes |
|---|---|---|
| Read-only analysis | `"Read,Glob,Grep"` | Safe default for file inspection |
| Code modification | `"Read,Edit,Write,Glob,Grep,Bash"` | Full dev toolkit |
| Pure reasoning | `""` (empty string) | No filesystem access at all |
| Web research | `"WebSearch,WebFetch"` | Internet access, no local files |
| Scoped read | `--allowedTools "Read(/src/**)"` | Only specific directories |

Always pair `--permission-mode dontAsk` with an explicit `--tools` list.
`dontAsk` auto-denies anything not whitelisted — without a tools list, Claude
can't use any tools at all.
