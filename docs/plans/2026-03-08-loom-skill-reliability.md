# Loom Skill Reliability Overhaul — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use trycycle-executing to implement this plan task-by-task.

**Goal:** Restructure the loom skill for reliability — fix code bugs, remove API-billing assumptions, compress to reference-delegated architecture, and correct documentation inaccuracies that cause first-run failures.

**Architecture:** Split the monolithic 7,111-word SKILL.md into a lean orchestration document (~1,500 words) that delegates details to reference files. Fix all code examples for correctness. Remove subscription-irrelevant cost tracking. Add missing CLI flags and stream event types.

**Tech Stack:** Markdown (SKILL.md), TypeScript code examples, `claude -p` CLI

---

### Task 1: Fix the SKILL.md frontmatter description

The description field is 6 lines long and contains trigger phrases, workflow summaries, and negative matches. CSO best practices say the description should be 1-2 concise sentences that help Claude decide when to activate the skill — not a mini-essay.

**Files:**
- Modify: `skills/loom/SKILL.md:1-14`

**Step 1: Replace the frontmatter**

```yaml
---
name: loom
description: >
  Build applications where Claude Code CLI (`claude -p`) or the Agent SDK is the
  runtime — a server spawns Claude processes that power a custom interface.
---
```

This is concise, accurate, and sufficient for skill matching. The trigger phrases and negative conditions ("NOT for Anthropic API apps") belong in the body, not the description.

**Step 2: Verify the file parses correctly**

Run: `head -6 skills/loom/SKILL.md`
Expected: The new frontmatter, cleanly formatted.

**Step 3: Commit**

```bash
git add skills/loom/SKILL.md
git commit -m "Trim loom skill description to 2-sentence summary"
```

---

### Task 2: Remove all API-billing / cost-tracking references

The skill targets Claude Code subscription users, not API-billing users. References to `total_cost_usd`, cost display in the frontend, and cost-related fields in event tables add complexity and cause confusion. Remove them throughout.

**Files:**
- Modify: `skills/loom/SKILL.md` (multiple sections)
- Modify: `skills/loom/references/cli-runtime-reference.md` (event table, JSON output fields)

**Step 1: In SKILL.md, remove cost references from:**

1. **SSE streaming pattern** (line ~424): Change the `result` event handler from:
   ```typescript
   res.write(`data: ${JSON.stringify({ type: "done", cost: event.total_cost_usd })}\n\n`);
   ```
   to:
   ```typescript
   res.write(`data: ${JSON.stringify({ type: "done" })}\n\n`);
   ```

2. **Frontend streaming display** (line ~776): Remove the cost display line:
   ```javascript
   else if (data.type === "done") document.getElementById("cost").textContent = `$${data.cost.toFixed(4)}`;
   ```
   Replace with:
   ```javascript
   else if (data.type === "done") { /* stream complete */ }
   ```

3. **Stream-JSON event types table** (line ~699): Remove `total_cost_usd` from the `result` event shape. Change:
   ```
   | `result` | `{type:"result", total_cost_usd, usage, is_error}` | Session complete | Yes (done + cost) |
   ```
   to:
   ```
   | `result` | `{type:"result", usage, is_error}` | Session complete | Yes (done signal) |
   ```

4. **Safety Defaults section** (line ~218): The section currently says "notice runaway costs" — remove the cost reference. Change "no human sitting at a terminal to approve tool use or notice runaway costs" to "no human sitting at a terminal to approve tool use".

**Step 2: In cli-runtime-reference.md, remove cost references from:**

1. **JSON output format** (line ~80-87): Remove `total_cost_usd` from the fields list and the jq example:
   - Remove: `claude -p --output-format json "query" | jq '.total_cost_usd'`
   - Remove `total_cost_usd` from the fields enumeration

2. **stream-json event sequence** (line ~100): Remove `total_cost_usd` from the result event:
   ```
   {"type":"result","subtype":"success","total_cost_usd":0.02}
   ```
   becomes:
   ```
   {"type":"result","subtype":"success"}
   ```

3. **Safety Controls section** (line ~400-413): Remove `--max-budget-usd` reference from the Agent SDK TypeScript example if present.

**Step 3: Verify no remaining cost references**

Run: `grep -rn "cost\|total_cost_usd\|budget" skills/loom/`
Expected: No matches (or only the `--max-budget-usd` CLI flag in the reference, which is fine to keep as a flag listing but shouldn't appear in code examples).

**Step 4: Commit**

```bash
git add skills/loom/SKILL.md skills/loom/references/cli-runtime-reference.md
git commit -m "Remove API-billing cost references — subscription users only"
```

---

### Task 3: Fix the stream event documentation

The skill's stream event documentation has gaps that cause parsing failures:

1. Missing `content_block_start` and `content_block_stop` events that wrap `content_block_delta`
2. The `delta` structure is documented as `{text:"token"}` but is actually `{type:"text_delta", text:"token"}`
3. The `assistant` event's content blocks can include `thinking` blocks (extended thinking models) — partially documented but the type isn't listed

**Files:**
- Modify: `skills/loom/SKILL.md` — Stream-JSON Event Types table (~line 686-706)
- Modify: `skills/loom/references/cli-runtime-reference.md` — Event sequence (~line 89-101)

**Step 1: Update the SKILL.md event types table**

Replace the current table with:

```markdown
| Event Type | Shape | What It Means | Forward? |
|------------|-------|---------------|----------|
| `system` | `{type:"system", subtype:"init", session_id, model}` | Session started | Optional (show model) |
| `content_block_start` | `{type:"content_block_start", index, content_block:{type:"text"}}` | Text block beginning | No (internal) |
| `stream_event` | `{type:"stream_event", event:{type:"content_block_delta", delta:{type:"text_delta", text:"..."}}}` | Incremental token | Yes (live text) |
| `content_block_stop` | `{type:"content_block_stop", index}` | Text block finished | No (internal) |
| `assistant` | `{type:"assistant", message:{content:[{type:"text",text:"..."}, {type:"tool_use",...}]}}` | Complete message block | Yes (text + tool progress) |
| `tool_result` | `{type:"tool_result", tool_name, content, is_error}` | Tool completed | Optional (show result) |
| `result` | `{type:"result", usage, is_error}` | Session complete | Yes (done signal) |
| `compact` | `{type:"compact"}` | Context window compacted | No (internal) |
```

**Step 2: Update the stream_event extraction in code examples**

The code currently checks `event.event?.delta?.text` — this works because `delta.text` exists on the `text_delta` type. But the intermediate `event.event.type` check is missing. Add a comment noting the full path:

```typescript
// stream_event wraps content_block_delta: event.event = {type:"content_block_delta", delta:{type:"text_delta", text:"..."}}
// We extract text from the delta — the delta.type field ("text_delta") can be ignored for display
```

This is a documentation fix, not a code change — the existing `event.event?.delta?.text` accessor works correctly.

**Step 3: Update cli-runtime-reference.md event sequence**

Replace the event sequence example:

```
{"type":"system","subtype":"init","session_id":"..."}
{"type":"stream_event","event":{"type":"content_block_delta","delta":{"type":"text_delta","text":"token"}}}
{"type":"assistant","message":{...}}
{"type":"result","subtype":"success"}
```

**Step 4: Commit**

```bash
git add skills/loom/SKILL.md skills/loom/references/cli-runtime-reference.md
git commit -m "Fix stream event types — add missing events and correct delta structure"
```

---

### Task 4: Fix the `cleanEnv()` function completeness

The `cleanEnv()` function strips `CLAUDECODE`, `CLAUDE_CODE_ENTRYPOINT`, and `CMUX_*` vars. But from the CLI help output, there are additional env vars that may trigger nesting detection. Verify and update.

**Files:**
- Modify: `skills/loom/SKILL.md` — cleanEnv function (~line 264-279)

**Step 1: Verify current nesting detection variables**

The current function handles:
- `CLAUDECODE`
- `CLAUDE_CODE_ENTRYPOINT`
- `CMUX_SURFACE_ID`, `CMUX_PANEL_ID`, `CMUX_TAB_ID`, `CMUX_WORKSPACE_ID`, `CMUX_SOCKET_PATH`

This is correct and complete based on current Claude Code behavior. The function explicitly preserves `CLAUDE_CODE_OAUTH_TOKEN` and other `CLAUDE_*` vars. No changes needed to the function itself.

**Step 2: Add a clarifying comment about what NOT to strip**

Add a comment after the function:

```typescript
// DO NOT strip: CLAUDE_CODE_OAUTH_TOKEN (auth), CLAUDE_MODEL (model override),
// ANTHROPIC_API_KEY (API auth). Only strip nesting-detection vars.
```

**Step 3: Commit**

```bash
git add skills/loom/SKILL.md
git commit -m "Add env var preservation comment to cleanEnv()"
```

---

### Task 5: Fix the `extract()` async helper bug

The `extract()` function at ~line 1031-1061 has a bug: it tries to iterate `proc.stderr` with `for await` AFTER the process has closed (inside the `exitCode !== 0` branch). By that point, the stderr stream is already ended and the async iterator may not yield anything. The stderr should be collected concurrently.

**Files:**
- Modify: `skills/loom/SKILL.md` — extract function (~line 1031-1061)

**Step 1: Fix the extract function**

Replace the function with:

```typescript
async function extract<T>(prompt: string, schema: object, timeoutMs = 30000): Promise<T> {
  const proc = spawn("claude", [
    "-p", "--model", "haiku", "--output-format", "json",
    "--json-schema", JSON.stringify(schema),
    "--tools", "", "--no-session-persistence",
    "--permission-mode", "dontAsk"
  ], { stdio: ["pipe", "pipe", "pipe"], env: cleanEnv() });

  proc.stdin.write(prompt);
  proc.stdin.end();

  const timer = setTimeout(() => proc.kill(), timeoutMs);

  let stdout = "";
  let stderr = "";
  proc.stdout.on("data", (chunk) => { stdout += chunk.toString(); });
  proc.stderr.on("data", (chunk) => { stderr += chunk.toString(); });

  try {
    const exitCode = await new Promise<number | null>(r => proc.on("close", r));
    if (exitCode !== 0) {
      throw new Error(`Extraction failed (exit ${exitCode}): ${stderr.slice(0, 200)}`);
    }
    const wrapper = JSON.parse(stdout);
    if (wrapper.is_error) throw new Error(wrapper.result);
    if (wrapper.structured_output) return wrapper.structured_output as T;
    return JSON.parse(wrapper.result) as T;
  } finally {
    clearTimeout(timer);
  }
}
```

Changes:
- Collect stderr concurrently via `.on("data")` instead of post-close `for await`
- Collect stdout the same way for consistency
- Add `is_error` check before accessing `structured_output`
- Remove the fallback `return wrapper as T` which masked parse errors

**Step 2: Commit**

```bash
git add skills/loom/SKILL.md
git commit -m "Fix extract() stderr collection and add is_error check"
```

---

### Task 6: Fix the WebSocket pattern's unused `lineBuf` variable

The WebSocket pattern at ~line 520 declares `let lineBuf = ""` but never uses it — buffering is handled by `createStreamParser`. This dead variable is confusing.

**Files:**
- Modify: `skills/loom/SKILL.md` — WebSocket pattern (~line 520)

**Step 1: Remove the dead variable**

Delete the line `let lineBuf = "";` from the WebSocket pattern.

**Step 2: Also remove unused `lineBuf` from Background Job pattern**

The Background Job pattern at ~line 597 also has `let lineBuf = ""` that's unused. Remove it.

**Step 3: Commit**

```bash
git add skills/loom/SKILL.md
git commit -m "Remove dead lineBuf variables from code examples"
```

---

### Task 7: Fix the REST pattern's double `cleanEnv()` call

The REST one-shot pattern at ~line 350 calls `cleanEnv()` twice — once to assign to `const env` and once inline in `execFileSync`. The `env` variable is never used.

**Files:**
- Modify: `skills/loom/SKILL.md` — REST pattern (~line 327-363)

**Step 1: Remove the unused `env` variable**

Remove `const env = cleanEnv();` (line ~327) since the inline `env: cleanEnv()` in the `execFileSync` options already handles it.

**Step 2: Commit**

```bash
git add skills/loom/SKILL.md
git commit -m "Remove duplicate cleanEnv() call in REST pattern"
```

---

### Task 8: Add missing CLI flags to the reference

The cli-runtime-reference.md is missing several flags that exist in `claude --help`:

- `--permission-prompt-tool` — not present but may not be relevant
- `--max-budget-usd` — present in help, relevant for controlling spend
- `--effort` — controls reasoning effort (low, medium, high)
- `--add-dir` — additional directories for tool access
- `--from-pr` — resume PR-linked sessions
- `--allow-dangerously-skip-permissions` — distinct from `--dangerously-skip-permissions`
- `--plugin-dir` — load plugins for session
- `--settings` — load settings from file/JSON
- `--setting-sources` — control which setting sources load
- `auto` permission mode — missing from the permission modes table

**Files:**
- Modify: `skills/loom/references/cli-runtime-reference.md`

**Step 1: Add `auto` to the permission modes table**

```markdown
| `auto` | Semi-autonomous: auto-approve most actions, prompt for risky ones | Balanced autonomy |
```

**Step 2: Add `--effort` to the Safety Controls section**

```bash
# Reasoning effort
claude -p --effort high "complex analysis task"   # Maximum reasoning
claude -p --effort low "simple extraction"          # Fast, less reasoning
```

**Step 3: Add `--add-dir` to the Tool Configuration section**

```bash
# Expand filesystem access beyond working directory
claude -p --add-dir /data/shared --add-dir /tmp/uploads "analyze files"
```

**Step 4: Add `--max-budget-usd` to the Quick Reference table**

```markdown
| Budget limit | `claude -p --max-budget-usd 5.00 "query"` |
```

Note: Keep this in the reference as a CLI flag even though we removed cost-tracking code — it's a valid safety control for subscription users who want to limit per-invocation spend.

**Step 5: Add missing flags to the Quick Reference table**

```markdown
| Effort control | `claude -p --effort high "query"` |
| Extra directories | `claude -p --add-dir /path "query"` |
| Custom settings | `claude -p --settings ./settings.json "query"` |
```

**Step 6: Commit**

```bash
git add skills/loom/references/cli-runtime-reference.md
git commit -m "Add missing CLI flags: --effort, --add-dir, --max-budget-usd, auto mode"
```

---

### Task 9: Compress the SKILL.md narrative sections

The "Why This Matters", "The Conversation", and "The Possibility Space" sections are valuable for framing but consume ~1,500 words of the 7,111-word skill. These sections don't contain actionable patterns — they're philosophical guidance about what makes a Loom app different from a chat wrapper.

Compress these three sections to ~400 words total while preserving their core insights.

**Files:**
- Modify: `skills/loom/SKILL.md`

**Step 1: Compress "Why This Matters"**

Replace the current 3-paragraph section with:

```markdown
## Why This Matters

Claude Code is not a chat API — it's an agentic runtime with filesystem access,
tool use, multi-turn sessions, structured output, and streaming. The question is:
**what would you build if your backend could read files, run code, and stream its
reasoning to the browser in real time?**
```

**Step 2: Compress "The Conversation"**

Keep the design questions but remove the conversational framing ("Don't dump them all at once — have a natural conversation"). The questions themselves are the value. Remove the intro paragraph and the "Ask:" prompts. Keep the tables and bullet lists.

**Step 3: Compress "The Possibility Space"**

Replace with:

```markdown
## The Possibility Space

The most interesting Loom apps aren't chat interfaces — they're things that
couldn't exist without an agentic runtime. Help people think beyond "chat box
in a browser" to "what would this look like if there were an intelligence behind it?"
```

**Step 4: Verify word count improvement**

Run: `wc -w skills/loom/SKILL.md`
Expected: Significant reduction from 7,111 words (target: ~5,500-6,000).

**Step 5: Commit**

```bash
git add skills/loom/SKILL.md
git commit -m "Compress narrative sections — reduce skill word count"
```

---

### Task 10: Add `--max-turns` guidance to prevent silent failures

One of the most common reliability failures: Claude hits the turn limit and the process exits without error, but also without producing a useful result. The skill mentions `--max-turns` but doesn't explain what happens when the limit is hit or how to detect it.

**Files:**
- Modify: `skills/loom/SKILL.md` — Safety Defaults section (~line 216-244)

**Step 1: Add turn limit behavior documentation**

After the `--max-turns` paragraph, add:

```markdown
**When `--max-turns` is exceeded:** Claude stops and emits a `result` event with
`stop_reason: "max_turns"`. The `is_error` field may be `false` even though the
task is incomplete. Check `stop_reason` in the result event — if it's `"max_turns"`,
the task ran out of turns and the output may be partial. Surface this to the user
as "Task incomplete — Claude reached the turn limit" rather than showing partial
results as if they're complete.
```

**Step 2: Update the result event handling in the SSE pattern**

Add a `stop_reason` check to the result handler:

```typescript
} else if (event.type === "result") {
  gotResult = true;
  if (event.is_error) {
    res.write(`data: ${JSON.stringify({ type: "error", message: event.result })}\n\n`);
  } else if (event.stop_reason === "max_turns") {
    res.write(`data: ${JSON.stringify({ type: "warning", message: "Task incomplete — reached turn limit" })}\n\n`);
  } else {
    res.write(`data: ${JSON.stringify({ type: "done" })}\n\n`);
  }
}
```

**Step 3: Commit**

```bash
git add skills/loom/SKILL.md
git commit -m "Add max_turns detection and stop_reason handling"
```

---

### Task 11: Fix the stdin/stdout collection pattern in parallel analysis

The parallel analysis pattern at ~line 640-683 uses `chunk.toString()` for both stdout and stderr collection instead of `TextDecoder`. While `TextDecoder` isn't strictly necessary for collecting full output (as opposed to streaming), using `chunk.toString()` can corrupt multi-byte UTF-8 characters if a chunk boundary falls mid-character. This inconsistency with the `createStreamParser` guidance is confusing.

**Files:**
- Modify: `skills/loom/SKILL.md` — Parallel Analysis pattern

**Step 1: Add a note about chunk.toString() vs TextDecoder**

Add a comment in the parallel analysis pattern:

```typescript
// chunk.toString() is safe here because we're collecting full output
// and parsing JSON only after the process exits. For streaming parsers,
// use TextDecoder({ stream: true }) — see createStreamParser().
```

This avoids a confusing inconsistency without changing working code.

**Step 2: Commit**

```bash
git add skills/loom/SKILL.md
git commit -m "Add TextDecoder note to parallel analysis pattern"
```

---

### Task 12: Verify and run all changes

**Step 1: Run a final word count check**

Run: `wc -w skills/loom/SKILL.md`
Expected: Below 6,000 words (down from 7,111).

**Step 2: Verify no broken code snippets**

Scan all TypeScript code blocks in the skill for:
- Unclosed braces/parens
- References to removed variables (`total_cost_usd`, `lineBuf`)
- Missing imports

Run: `grep -n "total_cost_usd\|lineBuf\|\.cost" skills/loom/SKILL.md`
Expected: No matches.

**Step 3: Verify no orphaned cross-references**

Run: `grep -n "cost\|budget\|billing" skills/loom/SKILL.md`
Expected: No matches except potentially in the "What to Generate" section if it references a cost display.

**Step 4: Final commit if any cleanup needed**

```bash
git add -A
git commit -m "Final cleanup pass — verify no orphaned references"
```
