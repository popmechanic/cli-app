# ElectroBun v1.15.1 Update & loom-desktop Skill Review — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use trycycle-executing to implement this plan task-by-task.

**Goal:** Update the loom-desktop skill to reflect ElectroBun v1.15.1 (released 2026-03-06), fix all factual errors, improve reliability patterns, and make it easier for users to build desktop apps with ElectroBun.

**Architecture:** Four skill files are updated in place. The main SKILL.md gets version bumps, corrected config/import references, `CLAUDE_BIN` consistency fixes across all code examples, and new API mentions. The electrobun-setup.md reference gets a rewritten template table, corrected config patterns, and corrected import paths. No new files are created; no files are deleted.

**Tech Stack:** Markdown skill documents with embedded TypeScript code examples. No executable code to test — verification is by diff review against the v1.15.1 source.

---

### Task 1: Fix version references and import path in electrobun-setup.md

**Files:**
- Modify: `skills/loom-desktop/references/electrobun-setup.md`

**What to change:**

1. **Config import path** (line 69): Change `import type { ElectrobunConfig } from "electrobun/config"` to `import type { ElectrobunConfig } from "electrobun"`.
   - Rationale: The `"electrobun/config"` export path does not exist in v1.15.1. The package.json exports map `"."` → `"./dist/api/bun/index.ts"` which re-exports `ElectrobunConfig`.

**Step 1: Fix the import path**

In `skills/loom-desktop/references/electrobun-setup.md`, find:
```typescript
import type { ElectrobunConfig } from "electrobun/config";
```
Replace with:
```typescript
import type { ElectrobunConfig } from "electrobun";
```

**Step 2: Verify the change**

Visually confirm the diff shows only the import path change.

**Step 3: Commit**

```bash
git add skills/loom-desktop/references/electrobun-setup.md
git commit -m "fix: correct ElectroBun config import path for v1.15.1"
```

---

### Task 2: Update template table in electrobun-setup.md

**Files:**
- Modify: `skills/loom-desktop/references/electrobun-setup.md`

**What to change:**

Replace the 4-template table with the actual 19 templates available in v1.15.1. Group them by category for usability.

**Step 1: Replace the template table**

Find the current table (lines ~22-28):
```markdown
| Template | Description |
|----------|-------------|
| `hello-world` | Minimal app — start here for Loom desktop apps |
| `photo-booth` | Camera integration example |
| `interactive-playground` | API exploration sandbox |
| `multitab-browser` | Multi-tab browser with navigation |
```

Replace with:
```markdown
| Template | Description |
|----------|-------------|
| `hello-world` | Minimal app — **start here for Loom desktop apps** |
| `react-tailwind-vite` | React + Tailwind + Vite — best choice for Loom apps with rich UI |
| `vanilla-vite` | Vanilla TypeScript + Vite |
| `tailwind-vanilla` | Tailwind CSS without a framework |
| `angular` | Angular framework template |
| `solid` | SolidJS framework template |
| `svelte` | Svelte framework template |
| `vue` | Vue framework template |
| `multi-window` | Multiple window management |
| `multitab-browser` | Multi-tab browser with navigation |
| `notes-app` | Note-taking application |
| `photo-booth` | Camera integration example |
| `sqlite-crud` | SQLite database operations |
| `tray-app` | System tray application |
| `wgpu` | Raw WebGPU rendering |
| `wgpu-threejs` | Three.js with native WebGPU |
| `wgpu-babylon` | Babylon.js with native WebGPU |
| `wgpu-mlp` | WebGPU machine learning playground |
| `bunny` | 3D bunny game demo |
```

**Step 2: Verify the change**

Confirm all 19 templates from the repo are listed and `interactive-playground` (which never existed) is removed.

**Step 3: Commit**

```bash
git add skills/loom-desktop/references/electrobun-setup.md
git commit -m "update template table to match ElectroBun v1.15.1 (19 templates)"
```

---

### Task 3: Fix `copy` config format in electrobun-setup.md

**Files:**
- Modify: `skills/loom-desktop/references/electrobun-setup.md`

**What to change:**

The current skill shows `copy` as an array inside `views.mainview`. In v1.15.1, `copy` is a `{ [sourcePath: string]: string }` object at the `build` level.

**Step 1: Fix the copy config in the Configuration section**

The current config example shows:
```typescript
build: {
    bun: {
      entrypoint: "src/bun/index.ts",
    },
    views: {
      mainview: {
        entrypoint: "src/mainview/index.ts",
      },
    },
  },
```

Update this to include `copy` at the correct level and add comments about new options:
```typescript
build: {
    bun: {
      entrypoint: "src/bun/index.ts",
    },
    views: {
      mainview: {
        entrypoint: "src/mainview/index.ts",
      },
    },
    copy: {
      "src/mainview/index.html": "views/mainview/index.html",
    },
  },
```

**Step 2: Update the Key Sections list**

After the config example, update the `build.views` description to include `build.copy`:
- **`build.copy`** — Files to copy to the build output. Keys are source paths, values are destination paths. Use this for HTML files, static assets, etc.

**Step 3: Verify the change**

Confirm `copy` is at the `build` level as `{ source: dest }` format.

**Step 4: Commit**

```bash
git add skills/loom-desktop/references/electrobun-setup.md
git commit -m "fix: correct copy config format (build-level object, not view-level array)"
```

---

### Task 4: Fix the copy directive in Gotcha #10 of SKILL.md

**Files:**
- Modify: `skills/loom-desktop/SKILL.md`

**What to change:**

Gotcha #10 ("Missing index.html") shows the old array-based `copy` inside `views.mainview`. Fix to match v1.15.1's `build.copy` as `{ source: dest }`.

**Step 1: Fix the copy directive**

Find (around line 1822-1829):
```typescript
build: {
  views: {
    mainview: {
      entrypoint: "src/mainview/index.ts",
      copy: ["src/mainview/index.html"],
    },
  },
},
```

Replace with:
```typescript
build: {
  views: {
    mainview: {
      entrypoint: "src/mainview/index.ts",
    },
  },
  copy: {
    "src/mainview/index.html": "views/mainview/index.html",
  },
},
```

**Step 2: Verify**

Confirm the gotcha now shows the correct `build.copy` format.

**Step 3: Commit**

```bash
git add skills/loom-desktop/SKILL.md
git commit -m "fix: correct copy config format in gotcha #10"
```

---

### Task 5: Update version references in SKILL.md

**Files:**
- Modify: `skills/loom-desktop/SKILL.md`

**What to change:**

Update all version references from v1.14.4 to v1.15.1.

**Step 1: Replace version strings**

1. Line 53: `v1.14.4 stable (v1.14.5-beta.0 in beta)` → `v1.15.1 stable`
2. Line 1753 (Gotcha #7): `v1.14.4 stable (March 2026)` → `v1.15.1 stable (March 2026)`

**Step 2: Update Gotcha #7 content**

The gotcha lists known platform issues. Update to reflect v1.15.1 status:

Find:
```markdown
### 7. ElectroBun beta status

ElectroBun is at v1.14.4 stable (March 2026). Known platform issues:

- **Linux**: Self-extracting binary issues on some distributions
- **Windows**: WebView2 preload script truncation in some configurations
- **No `saveFileDialog()`** — use `Bun.write()` with a known path as workaround
- **Menu support**: Full on macOS, basic on Windows, not supported on Linux
```

Replace with:
```markdown
### 7. ElectroBun version status

ElectroBun is at v1.15.1 stable (March 2026). Known platform issues:

- **Linux**: Self-extracting binary issues on some distributions
- **Windows**: WebView2 preload script truncation in some configurations
- **No `saveFileDialog()`** — use `Bun.write()` with a known path as workaround
- **Menu support**: Full on macOS, basic on Windows, not supported on Linux

New in v1.15.x: `--watch` race condition fixed, WebGPU/WGPU native rendering,
`GpuWindow`, `GlobalShortcut`, `Screen`, `Session`, `Socket`, `BuildConfig` APIs.
```

**Step 3: Commit**

```bash
git add skills/loom-desktop/SKILL.md
git commit -m "update version references to ElectroBun v1.15.1"
```

---

### Task 6: Fix `CLAUDE_BIN` consistency — all `Bun.spawn` calls must use it

**Files:**
- Modify: `skills/loom-desktop/SKILL.md`

**What to change:**

The skill defines `resolveClaudePath()` and `CLAUDE_BIN` in the Shared Utilities section, but several code examples still use the string `"claude"` instead of `CLAUDE_BIN`. This is a reliability bug — apps launched from Finder won't find the CLI.

**Step 1: Fix Pattern 1 (Synchronous)**

Find the `Bun.spawnSync` call around line 454:
```typescript
        const proc = Bun.spawnSync([
            "claude", "-p",
```
Replace with:
```typescript
        const proc = Bun.spawnSync([
            CLAUDE_BIN, "-p",
```

**Step 2: Fix Pattern 2 (Streaming)**

Find the `Bun.spawn` call around line 566:
```typescript
  const proc = Bun.spawn(["claude", ...args], {
```
Replace with:
```typescript
  const proc = Bun.spawn([CLAUDE_BIN, ...args], {
```

**Step 3: Fix the startup CLI check**

Find around line 861:
```typescript
    const result = Bun.spawnSync(["claude", "--version"]);
```
Replace with:
```typescript
    const result = Bun.spawnSync([CLAUDE_BIN, "--version"]);
```

**Step 4: Add a note after the Shared Utilities section**

After `const CLAUDE_BIN = resolveClaudePath();` (line 264), add:

```markdown
**Use `CLAUDE_BIN` everywhere.** Every `Bun.spawn()` and `Bun.spawnSync()` call
in your app must use `CLAUDE_BIN` instead of the string `"claude"`. The patterns
below all reference `CLAUDE_BIN`.
```

**Step 5: Verify**

Search the file for any remaining `"claude"` strings in `Bun.spawn` calls. There should be none.

**Step 6: Commit**

```bash
git add skills/loom-desktop/SKILL.md
git commit -m "fix: use CLAUDE_BIN consistently in all spawn calls"
```

---

### Task 7: Fix stale `--continue` reference in Pattern 3 notes

**Files:**
- Modify: `skills/loom-desktop/SKILL.md`

**What to change:**

Pattern 3's "Don't do this" section and the note above it reference `--continue` as the flag for follow-up turns. The actual pattern in `spawnClaude()` uses `--resume <sessionId>` (not `--continue`). The `--continue` flag resumes the *most recent* session, not a specific one — it's wrong for multi-session apps.

**Step 1: Fix the note after the conversational handler**

Find (around lines 965-968):
```markdown
Note: `spawnClaude` from Pattern 2 already appends `--session-id <id>
--continue` when `opts.sessionId` is provided. The first turn omits
`--continue` — set that logic in `spawnClaude` by checking if the session
has prior turns.
```

Replace with:
```markdown
Note: `spawnClaude` from Pattern 2 already handles session flags: it uses
`--session-id <id>` on the first turn and `--resume <id>` on follow-up
turns. The `isFirstTurn` flag controls which flag is used.
```

**Step 2: Fix the "Don't do this" bullets**

Find (around lines 1084-1089):
```markdown
**Don't do this:**
- Don't spawn concurrent Claude processes on the same session — one at a time.
  `--continue` resumes the most recent turn; parallel spawns create race
  conditions.
- Don't forget `--continue` on follow-up turns. Without it, Claude starts a
  fresh conversation even with the same `--session-id`.
```

Replace with:
```markdown
**Don't do this:**
- Don't spawn concurrent Claude processes on the same session — one at a time.
  `--resume` resumes a specific session; parallel spawns create race conditions.
- Don't forget `--resume` on follow-up turns. Without it, Claude starts a
  fresh conversation. Use `--session-id` only on the first turn.
```

**Step 3: Commit**

```bash
git add skills/loom-desktop/SKILL.md
git commit -m "fix: replace stale --continue references with --resume in Pattern 3"
```

---

### Task 8: Add new ElectroBun APIs to electrobun-setup.md Key Concepts

**Files:**
- Modify: `skills/loom-desktop/references/electrobun-setup.md`

**What to change:**

Add brief mentions of the most relevant new v1.15.1 APIs that Loom desktop builders might use. These go in the Key Concepts section after the existing BrowserView/RPC/etc entries.

**Step 1: Add new API subsections**

After the "Navigation Rules" subsection (before "## Next Steps"), add:

```markdown
### GlobalShortcut

Register system-wide keyboard shortcuts:

```typescript
import { GlobalShortcut } from "electrobun/bun";

GlobalShortcut.register("CommandOrControl+Shift+Space", () => {
  // Activate your app from anywhere
  win.focus();
});
```

### Screen

Query display information (resolution, scaling, positioning):

```typescript
import { Screen } from "electrobun/bun";
// Use for window positioning and multi-monitor support
```

### Session

Manage browser sessions and cookies:

```typescript
import { Session } from "electrobun/bun";
// Cookie management, storage partitioning
```

### BuildConfig

Access build-time configuration at runtime:

```typescript
import { BuildConfig } from "electrobun/bun";

const config = await BuildConfig.get();
// config.runtime contains values from electrobun.config.ts runtime section
```

### Context Menus

Right-click context menus:

```typescript
import { ContextMenu } from "electrobun/bun";
// Define context menus for webview content
```
```

**Step 2: Add new config options to the Configuration section**

After the existing Key Sections list, add a brief note:

```markdown
Additional config options (see [ElectroBun docs](https://blackboard.sh/electrobun/docs/) for details):
- **`app.urlSchemes`** — Register custom URL schemes for deep linking (macOS only)
- **`build.useAsar`** — Pack assets into ASAR archive
- **`build.mac.bundleWGPU`** / **`build.win.bundleWGPU`** / **`build.linux.bundleWGPU`** — Bundle WebGPU for native GPU rendering
- **`build.targets`** — Build for specific platforms
- **`scripts`** — Pre/post build hooks (`preBuild`, `postBuild`, `postWrap`, `postPackage`)
- **`runtime`** — Arbitrary key-value pairs accessible at runtime via `BuildConfig.get()`
```

**Step 3: Commit**

```bash
git add skills/loom-desktop/references/electrobun-setup.md
git commit -m "add new v1.15.1 APIs and config options to setup reference"
```

---

### Task 9: Fix `--watch` flag documentation

**Files:**
- Modify: `skills/loom-desktop/references/electrobun-setup.md`

**What to change:**

The `--watch` flag had a race condition that was fixed in v1.14.5-beta.0 (commit `7a119ebe`). The current documentation already mentions `--watch` but doesn't note the fix. Since it now works reliably, no change is needed to the text — but verify this.

**Step 1: Verify the --watch documentation**

Read lines 107-113 of `electrobun-setup.md` and confirm `--watch` is already documented as working. It is — the current text says:

```
Add `--watch` for auto-rebuild on source changes:
```

This is correct for v1.15.1. No change needed.

**Step 2: Commit (skip — no changes)**

No commit needed for this task.

---

### Task 10: Improve reliability — add error recovery pattern to SKILL.md

**Files:**
- Modify: `skills/loom-desktop/SKILL.md`

**What to change:**

Add a retry/recovery pattern for when Claude CLI processes crash or timeout. This is a common reliability issue in real desktop apps.

**Step 1: Add a new Gotcha #16**

After Gotcha #15 (No cross-compilation), add:

```markdown
### 16. Process crash recovery

Claude processes can fail for transient reasons — network timeouts, rate
limits, or API errors. Desktop apps should handle this gracefully:

```typescript
async function spawnWithRetry(
  taskId: string,
  prompt: string,
  opts: any,
  rpc: any,
  maxRetries = 2
) {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const proc = spawnClaude(taskId, prompt, opts, rpc);
      const exitCode = await proc.exited;
      if (exitCode === 0) return;
      if (attempt < maxRetries) {
        console.warn(`Claude exited ${exitCode}, retrying (${attempt + 1}/${maxRetries})...`);
        await new Promise(r => setTimeout(r, 1000 * (attempt + 1)));
      }
    } catch (err) {
      if (attempt === maxRetries) throw err;
    }
  }
}
```

Don't retry on user-initiated aborts (`SIGTERM`). Only retry on unexpected
exits (non-zero exit code without an abort request).
```

**Step 2: Commit**

```bash
git add skills/loom-desktop/SKILL.md
git commit -m "add process crash recovery gotcha for reliability"
```

---

### Task 11: Add `sandbox` consideration to electrobun-setup.md

**Files:**
- Modify: `skills/loom-desktop/references/electrobun-setup.md`

**What to change:**

The BrowserWindow/BrowserView can use a `sandbox` option for webview isolation. Mention this in the Key Concepts section since it affects security posture for Loom apps.

Actually, after checking the source more carefully, `sandbox` is a property on BrowserView options but not a top-level config option. The current skill doesn't mention it. Let me check if it's relevant enough.

**Step 1: Skip — sandbox is a BrowserView constructor option, already covered implicitly**

The skill already documents `BrowserView` creation. Adding sandbox here would be scope creep. Skip this task.

---

### Task 12: Update the "What to Generate" checklist in SKILL.md

**Files:**
- Modify: `skills/loom-desktop/SKILL.md`

**What to change:**

The checklist references `deriveAndSendRPC()` as a separate function. In the actual code patterns, this logic is inlined in `spawnClaude()`. Clean this up and add `resolveClaudePath()` to the checklist since it's critical.

**Step 1: Update the checklist**

Find the current checklist (around line 1946-1955):
```markdown
- [ ] `src/bun/claude-manager.ts` — `cleanEnv()`, `createStreamParser()`, `spawnClaude()`, `abort()`, heartbeat, `deriveAndSendRPC()`
```

Replace with:
```markdown
- [ ] `src/bun/claude-manager.ts` — `resolveClaudePath()`, `cleanEnv()`, `createStreamParser()`, `spawnClaude()`, `abort()`, heartbeat
```

**Step 2: Commit**

```bash
git add skills/loom-desktop/SKILL.md
git commit -m "update What to Generate checklist (add resolveClaudePath, remove deriveAndSendRPC)"
```

---

### Task 13: Final review — verify all factual claims

**Files:**
- Read: All four skill files

**What to do:**

1. Search all files for `"electrobun/config"` — should be zero occurrences.
2. Search all files for `v1.14` — should be zero occurrences.
3. Search all files for `"claude"` in `Bun.spawn` contexts — should all be `CLAUDE_BIN`.
4. Search for `interactive-playground` — should be zero occurrences.
5. Search for `copy: [` (array format) — should be zero occurrences.
6. Verify `--continue` is not referenced as the session flag (only `--resume`).

**Step 1: Run searches**

```bash
cd /Users/marcusestes/Websites/loom/.worktrees/update-electrobun-skill
grep -rn 'electrobun/config' skills/loom-desktop/
grep -rn 'v1\.14' skills/loom-desktop/
grep -rn 'interactive-playground' skills/loom-desktop/
grep -rn 'copy: \[' skills/loom-desktop/
```

**Step 2: Fix any remaining issues found**

**Step 3: Final commit if needed**

```bash
git add -A
git commit -m "final cleanup: verify all factual claims against v1.15.1"
```
