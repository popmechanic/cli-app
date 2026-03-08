# Audit Auth Guidance for Login with Anthropic

> **For Claude:** REQUIRED SUB-SKILL: Use trycycle-executing to implement this plan task-by-task.

## Goal

Improve the reliability of the "Login with Anthropic" feature in the Loom skill by consolidating scattered auth guidance, fixing ambiguous env function references, making implicit dependencies explicit, and documenting the complete bidirectional 401 flow. The neko-chat reference implementation is the authoritative working example.

## Problem Analysis

The Loom skill's auth guidance is split across three files (SKILL.md, oauth-reference.md, cli-runtime-reference.md) with these concrete failure modes:

1. **`cleanEnv()` vs `spawnEnvForUser()` confusion** — SKILL.md's code patterns all use `cleanEnv()` directly, which produces an env without `CLAUDE_CODE_OAUTH_TOKEN`. The oauth-reference.md defines `spawnEnvForUser()` which calls `cleanEnv()` then injects the token. But the SKILL.md patterns never use `spawnEnvForUser()` — they use `cleanEnv()` bare. Claude following SKILL.md's patterns will produce servers where auth tokens are never injected.

2. **Scattered auth guidance across 3 files** — SKILL.md's "Authentication Setup" section says "see oauth-reference.md" but the code patterns in SKILL.md use `cleanEnv()` without auth integration. A reader following SKILL.md's SSE/WebSocket/REST patterns gets a working server structure but with no auth wiring. The oauth-reference.md has the complete auth story but it's disconnected from the patterns.

3. **Cookie parser dependency implicit** — SKILL.md's "Server Setup" block shows `app.use(cookieParser())` but never mentions that `cookie-parser` is a required npm dependency for auth. The "What to Generate" section's package.json list includes it, but someone building incrementally could miss it.

4. **Profile fetch after token exchange sometimes omitted** — The oauth-reference.md shows the profile fetch, and the "What to Generate" section mentions it, but the SKILL.md "Authentication Setup" checklist item #3 is easily overlooked. When Claude generates code, it sometimes omits the profile fetch, leaving `session.profile` undefined — the frontend shows "CONNECTED" instead of the user's email.

5. **401 bidirectional flow not well-documented** — The oauth-reference.md has a "401 Handling" section and SKILL.md mentions 401 in the First-Run Reliability Checklist, but neither describes the full round-trip: server restart → browser has stale cookie → 401 on protected endpoint → show setup screen → re-auth → return to app. The neko-chat implementation does this correctly (see `response.status === 401` handler in `send()` function).

6. **Scope string currency/versioning** — The OAuth scope string `user:profile user:inference user:sessions:claude_code user:mcp_servers` appears in both oauth-reference.md and neko-chat. If Anthropic changes scopes, it needs updating in two places. No note about scope versioning or what each scope does.

7. **`hasCompletedOnboarding` flag for deployment scenarios** — For deployed apps where sessions are wiped on restart, there's no guidance on distinguishing "never authenticated" from "session expired." Both show the setup screen, but the UX should differ (welcome vs. "reconnect").

## Worktree

All edits target: `/Users/marcusestes/Websites/loom/.worktrees/audit-auth-guidance/`

## Changed Files

- `skills/loom/SKILL.md`
- `skills/loom/references/oauth-reference.md`

## Tasks

### Task 1: Replace `cleanEnv()` with `spawnEnvForUser()` in SKILL.md patterns

**Why:** Every server pattern in SKILL.md currently shows `env: cleanEnv()` — this produces an environment WITHOUT the user's OAuth token. When someone builds an app following these patterns and also adds auth from the oauth-reference.md, the patterns silently fail because the token is never injected. The fix is to show `spawnEnvForUser(req.userSession)` in authenticated patterns and keep `cleanEnv()` only where it's used as a building block.

**Files:** `skills/loom/SKILL.md`

**Changes:**

1. In the **Shared Utilities** section, after the `cleanEnv()` definition and the `createStreamParser()` definition, add `spawnEnvForUser()` as a third shared utility:

   Add after the `createStreamParser()` closing backticks and its explanation paragraph:

   ```
   **`spawnEnvForUser()`** — Inject the requesting user's OAuth token into the spawn environment.

   Every authenticated spawn needs the user's `CLAUDE_CODE_OAUTH_TOKEN` in the
   environment — without it, `claude -p` starts but can't authenticate with
   Anthropic's servers. This wraps `cleanEnv()` with the token injection.
   See `references/oauth-reference.md` for the full session store and
   `requireAuth` middleware that provides `req.userSession`.

   ```typescript
   function spawnEnvForUser(session: UserSession): NodeJS.ProcessEnv {
     const env = cleanEnv();
     env.CLAUDE_CODE_OAUTH_TOKEN = session.accessToken;
     return env;
   }
   ```

   For local-only single-user development where you're using your own CLI
   credentials, `cleanEnv()` alone works. For any deployed or multi-user app,
   always use `spawnEnvForUser()`.
   ```

2. In the **SSE Streaming** pattern, change `{ env: cleanEnv() }` to `{ env: spawnEnvForUser(req.userSession) }`. Also add `requireAuth,` to the route middleware and `await refreshSessionIfNeeded(req.userSession);` before the spawn. The route line changes from:
   - Old: `app.post("/api/stream", (req, res) => {`
   - New: `app.post("/api/stream", requireAuth, async (req: any, res) => {`
   And add the refresh call after the `const { task } = req.body;` line.

3. In the **WebSocket Session** pattern, add a comment noting that auth must be validated on connection upgrade and token injected. Change `{ env: cleanEnv() }` to `{ env: spawnEnvForUser(userSession) }`. Add a comment before the spawn showing where the session comes from.

4. In the **Background Job** pattern, same change: `cleanEnv()` → `spawnEnvForUser(req.userSession)` with auth middleware.

5. In the **Parallel Analysis** pattern, same change.

6. In the **Persistent Session** pattern, same change with a comment about obtaining the session.

7. In the **REST Endpoint** pattern, same change.

8. In the **Structured Extraction** pattern, add a `session` parameter: `async function extract<T>(prompt: string, schema: object, session: UserSession, timeoutMs = 30000)` and change `cleanEnv()` to `spawnEnvForUser(session)`.

**Verification:** Search SKILL.md for `env: cleanEnv()` — should only appear inside the `cleanEnv()` definition itself and `spawnEnvForUser()` definition. All spawn sites should use `spawnEnvForUser()`.

### Task 2: Add cookie-parser to Server Setup block with explicit dependency note

**Why:** The Server Setup block shows `app.use(cookieParser())` but a reader adding auth incrementally may not realize this is a required npm install. The package.json mention in "What to Generate" is too far away.

**Files:** `skills/loom/SKILL.md`

**Changes:**

In the **Server Setup** section, add a comment in the code block after the import:

Change:
```typescript
import cookieParser from "cookie-parser";
```

To:
```typescript
import cookieParser from "cookie-parser";  // npm i cookie-parser @types/cookie-parser
```

**Verification:** The dependency is now visible at point of use.

### Task 3: Strengthen profile fetch guidance in SKILL.md

**Why:** The profile fetch is critical for the frontend to show who's logged in, but it's buried in bullet point #3 of the "Authentication Setup" section. Claude sometimes skips it, leaving `session.profile` undefined.

**Files:** `skills/loom/SKILL.md`

**Changes:**

In the **Authentication Setup** section, strengthen item #3 from a bare list item to an emphasized warning. Change:

```
3. After token exchange, fetch the user's profile from `https://api.anthropic.com/v1/me`
   and store it in the session — the frontend needs this to show who's logged in
```

To:

```
3. **After token exchange, fetch the user's profile** from `https://api.anthropic.com/v1/me`
   using the new `access_token` as a Bearer token, and store `{name, email}` in the
   session. Without this, the frontend can't show who's logged in — it falls back to
   a generic "CONNECTED" label. This is the most commonly omitted step.
```

**Verification:** The emphasis and "most commonly omitted" warning makes it harder to skip.

### Task 4: Document the full 401 bidirectional flow in oauth-reference.md

**Why:** The current "401 Handling" section shows only the frontend check. It doesn't describe the complete cycle or how the setup screen re-initializes. The neko-chat implementation does this correctly and serves as the reference.

**Files:** `skills/loom/references/oauth-reference.md`

**Changes:**

Replace the existing "## 401 Handling" section with an expanded version:

```markdown
## 401 Handling: Session Expiry Round-Trip

In-memory sessions don't survive server restarts. When the server redeploys
or crashes, every user's session is gone. Their browser still has the
`loom_session` cookie, but the server returns 401 for any authenticated
endpoint.

### The Full Cycle

```
Server restart → sessions wiped
        ↓
Browser sends request with stale cookie
        ↓
requireAuth returns 401  ←── also triggers on session TTL expiry
        ↓
Frontend catches 401 BEFORE parsing response body
        ↓
Frontend hides main app UI, shows setup screen
        ↓
User re-authenticates via OAuth flow
        ↓
New session created, new cookie set
        ↓
Setup screen calls onComplete / reloads
        ↓
Main app UI restored with fresh session
```

### Frontend Guard (Required on Every Protected Fetch)

Every `fetch()` to a protected endpoint must check for 401 **before**
attempting to parse the response as SSE, JSON, or any other format.
A 401 response body is JSON (`{"error":"Not authenticated"}`) — if
this reaches the SSE parser, it fails silently and the user sees nothing.

```javascript
const response = await fetch("/api/chat", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ message }),
});

if (response.status === 401) {
  // Session expired or server restarted — show setup screen
  // Hide the main app UI and re-initialize the setup flow
  document.getElementById('setup-screen').style.display = 'flex';
  document.querySelector('.main-app').style.display = 'none';
  initSetup();  // Re-initialize OAuth event listeners
  return;
}

// Only now parse as SSE stream
const reader = response.body.getReader();
```

### Key Implementation Details

- **Re-initialize the setup screen**, don't just show/hide it. The OAuth
  state variables (PKCE state, code input value) need to be fresh. Call
  `initSetup()` or remount the `<SetupScreen>` component.
- **Clear any in-progress UI state** (streaming indicators, disabled inputs)
  before showing the setup screen.
- **After re-auth, restore the app view** — the setup screen's `onComplete`
  callback should hide the setup screen and show the main app, then focus
  the primary input.
- **The `/api/health` endpoint does NOT require auth** — it checks for a
  session cookie but returns `{needsSetup: true}` instead of 401 when
  the session is missing. This is intentional: the health check is used
  for initial page load to determine which screen to show.
```

**Verification:** The section now describes the complete cycle, not just the frontend snippet.

### Task 5: Add scope documentation to oauth-reference.md

**Why:** The scope string is a magic constant with no explanation. If scopes change, there's no guidance on what each does or how to update.

**Files:** `skills/loom/references/oauth-reference.md`

**Changes:**

In the **Constants** section, after the scope string definition, add documentation:

After the line:
```typescript
const OAUTH_SCOPES = "user:profile user:inference user:sessions:claude_code user:mcp_servers";
```

Add (outside the code block, as a note):

```markdown
**Scope breakdown:**

| Scope | Purpose | Required? |
|-------|---------|-----------|
| `user:profile` | Read user's name and email via `/v1/me` | Yes — frontend identity display |
| `user:inference` | Make inference requests through the user's account | Yes — core functionality |
| `user:sessions:claude_code` | Create and manage Claude Code sessions | Yes — `claude -p` needs this |
| `user:mcp_servers` | Access user's configured MCP servers | Optional — only if app uses MCP |

These scopes are part of Anthropic's OAuth API surface. If a scope is
rejected during authorization (user sees an error on claude.ai), check
whether the scope string is still current in Anthropic's documentation.
The `OAUTH_CLIENT_ID` is a shared public client ID for CLI-based OAuth
flows — it is not app-specific and does not need to be changed.
```

**Verification:** Each scope is now explained, and there's guidance for debugging scope rejection.

### Task 6: Add onboarding vs. reconnection UX guidance to oauth-reference.md

**Why:** Deployed apps need to distinguish first-time setup from session expiry. Both show the setup screen, but the messaging should differ ("Welcome" vs. "Session expired, reconnect").

**Files:** `skills/loom/references/oauth-reference.md`

**Changes:**

Add a new section after "401 Handling" called "## First Visit vs. Reconnection":

```markdown
## First Visit vs. Reconnection

For deployed apps, users may see the setup screen in two scenarios:

1. **First visit** — Never authenticated before. Show a welcome message.
2. **Session expired** — Previously authenticated, server restarted. Show a reconnection message.

The simplest approach: use `localStorage` to remember that the user has
completed onboarding at least once.

```javascript
// After successful OAuth exchange:
localStorage.setItem('hasCompletedOnboarding', 'true');

// On page load, when /api/health returns needsSetup: true:
const isReconnect = localStorage.getItem('hasCompletedOnboarding') === 'true';
if (isReconnect) {
  showSetupScreen({ message: 'Session expired. Sign in again to reconnect.' });
} else {
  showSetupScreen({ message: 'Sign in with your Anthropic account to get started.' });
}
```

This is a UX nicety, not a security mechanism. The server is the sole
authority on session validity — `localStorage` only affects the welcome
message. For apps where this distinction doesn't matter (e.g., dev tools,
single-user), skip it.
```

**Verification:** The guidance is clearly scoped as optional UX polish, not a security requirement.

### Task 7: Add `cookie-parser` to oauth-reference.md dependency note and add `requireAuth` import note to SKILL.md patterns

**Why:** The oauth-reference.md defines `requireAuth` and the session infrastructure, but SKILL.md's patterns use them without explaining where they come from. A reader following SKILL.md alone gets undefined function errors.

**Files:** `skills/loom/SKILL.md`

**Changes:**

In the **Server Setup** section, after the `app.use(cookieParser())` line and before the PORT/listen section, add a comment:

```typescript
// Auth middleware and session store — see references/oauth-reference.md
// Required: requireAuth, refreshSessionIfNeeded, spawnEnvForUser,
// OAuth endpoints (/api/oauth/start, /api/oauth/exchange, /api/health, /api/logout)
```

**Verification:** The server setup block now points to the exact functions and endpoints that need to be imported/defined from the oauth reference.

### Task 8: Add First-Run Reliability Checklist items for auth-specific failures

**Why:** The existing checklist covers streaming and env issues but doesn't cover auth-specific silent failures.

**Files:** `skills/loom/SKILL.md`

**Changes:**

In the **First-Run Reliability Checklist** section, add these items after the existing list:

```markdown
- [ ] `spawnEnvForUser()` (not bare `cleanEnv()`) is used on every authenticated spawn — without the token, `claude -p` starts but can't reach Anthropic's API
- [ ] `cookie-parser` middleware is applied before any route that reads `req.cookies` — without it, `requireAuth` sees `undefined` and every request returns 401
- [ ] Profile fetch happens in the `/api/oauth/exchange` handler after token exchange — without it, the frontend shows "CONNECTED" instead of the user's email
- [ ] `/api/health` does NOT use `requireAuth` — it must return `{needsSetup: true}` for unauthenticated users, not 401
```

**Verification:** Auth failures now have their own checklist items, making them harder to miss during generation.

## Execution Notes

- All changes are markdown edits — no runtime code to test.
- The neko-chat implementation in the main repo should NOT be modified — it is the working reference.
- Do NOT attempt to run `claude -p` or spawn any Claude processes. This is a documentation-only audit.
- After editing, verify consistency by searching for `cleanEnv()` usage in SKILL.md — it should only appear in the function definition itself and in the `spawnEnvForUser()` definition.
