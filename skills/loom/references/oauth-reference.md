# Anthropic OAuth Reference

Loom apps need valid Anthropic credentials before the Claude CLI can function.
This reference implements **"Bring Your Own Claude"** — users authenticate with
their own Anthropic account through a one-time OAuth setup. No API keys to
manage, no cost on your side. Users bring their own Claude subscription.

## Table of Contents

1. [User Flow](#user-flow)
2. [Constants](#constants)
3. [PKCE Utilities](#pkce-utilities)
4. [Server Endpoints](#server-endpoints)
5. [Token Lifecycle](#token-lifecycle)
6. [Frontend Setup Screen](#frontend-setup-screen)
7. [Integration Notes](#integration-notes)

---

## User Flow

The authentication is a four-step process:

1. User clicks **"Sign in with Anthropic"** in the setup screen
2. User authorizes the app on **claude.ai**
3. User copies the **authentication code** from the callback page
4. User pastes the code in the app, which exchanges it for tokens

```
┌──────────┐   GET /api/oauth/start   ┌──────────┐
│  Browser  │ ───────────────────────► │  Server  │
│  (Setup)  │ ◄─────────────────────── │          │
│           │   { authUrl, state }     │          │
└─────┬─────┘                          └──────────┘
      │
      │  Opens popup to authUrl
      ▼
┌──────────────────┐
│  claude.ai       │
│  /oauth/authorize│
│                  │
│  User authorizes │
│  and copies code │
└────────┬─────────┘
         │
         │  User pastes code
         ▼
┌──────────┐  POST /api/oauth/exchange  ┌──────────┐  POST token endpoint  ┌───────────────┐
│  Browser  │ ────────────────────────► │  Server  │ ──────────────────── │  Anthropic    │
│  (Setup)  │ ◄─────────────────────── │          │ ◄────────────────── │  OAuth        │
│           │   { ok: true }           │          │   { access_token,   │               │
└──────────┘                           │          │     refresh_token } │               │
                                       └─────┬────┘                     └───────────────┘
                                             │
                                             │  Writes tokens to
                                             ▼
                                       ~/.claude/.credentials.json
                                             │
                                             │  Claude CLI auto-discovers
                                             ▼
                                       claude -p starts successfully
```

### Why PKCE?

PKCE (Proof Key for Code Exchange, RFC 7636) prevents authorization code
interception attacks. The server generates a random `code_verifier` and
sends its SHA-256 hash (`code_challenge`) with the authorization request.
When exchanging the code for tokens, the server proves it initiated the
request by providing the original verifier. Anthropic's OAuth requires PKCE.

---

## Constants

```typescript
const OAUTH_CLIENT_ID = "9d1c250a-e61b-44d9-88ed-5944d1962f5e";
const OAUTH_AUTHORIZE_URL = "https://claude.ai/oauth/authorize";
const OAUTH_TOKEN_URL = "https://platform.claude.com/v1/oauth/token";
const OAUTH_REDIRECT_URI = "https://platform.claude.com/oauth/code/callback";
const OAUTH_SCOPES = "user:profile user:inference user:sessions:claude_code user:mcp_servers";
```

---

## PKCE Utilities

Pure Node.js crypto, no framework dependency:

```typescript
import { createHash, randomBytes } from "crypto";

function base64url(buf: Buffer): string {
  return buf.toString("base64").replace(/\+/g, "-").replace(/\//g, "_").replace(/=+$/g, "");
}

function generateCodeVerifier(): string {
  return base64url(randomBytes(32));
}

function generateCodeChallenge(verifier: string): string {
  return base64url(createHash("sha256").update(verifier).digest());
}
```

---

## Server Endpoints

### `GET /api/oauth/start`

Generates a PKCE challenge and returns an authorization URL for the frontend
to open in a popup.

- Generates random state (32 bytes hex) and PKCE verifier/challenge pair
- Stores `{state -> {verifier, createdAt}}` in an in-memory Map with 10-minute TTL
- Builds the authorization URL with all required parameters
- Returns `{authUrl, state}` to the frontend

```typescript
const pendingPKCE = new Map<string, { verifier: string; createdAt: number }>();
const PKCE_TTL_MS = 10 * 60 * 1000;

// Cleanup expired PKCE flows every 60s
setInterval(() => {
  const now = Date.now();
  for (const [key, val] of pendingPKCE) {
    if (now - val.createdAt > PKCE_TTL_MS) pendingPKCE.delete(key);
  }
}, 60_000);

app.get("/api/oauth/start", (req, res) => {
  const state = randomBytes(32).toString("hex");
  const verifier = generateCodeVerifier();
  const challenge = generateCodeChallenge(verifier);

  const authUrl = `${OAUTH_AUTHORIZE_URL}?` + new URLSearchParams({
    client_id: OAUTH_CLIENT_ID,
    code_challenge: challenge,
    code_challenge_method: "S256",
    redirect_uri: OAUTH_REDIRECT_URI,
    scope: OAUTH_SCOPES,
    state,
    response_type: "code",
  }).toString();

  pendingPKCE.set(state, { verifier, createdAt: Date.now() });
  res.json({ authUrl, state });
});
```

### `POST /api/oauth/exchange`

Exchanges an authorization code for tokens using the stored PKCE verifier.

- Validates state exists in pendingPKCE map (returns 400 if expired/missing)
- Strips `#state` suffix from code (handles callback URL fragments)
- POSTs to Anthropic's token endpoint with the code, verifier, and other params
- On success, calls `writeCredentials()` with the returned tokens
- Returns `{ok: true}` to the frontend

```typescript
app.post("/api/oauth/exchange", async (req, res) => {
  const { code, state } = req.body;
  if (!code || !state) return res.status(400).json({ error: "code and state required" });

  const pending = pendingPKCE.get(state);
  if (!pending) return res.status(400).json({ error: "Invalid or expired state" });
  pendingPKCE.delete(state);

  const cleanCode = code.split("#")[0]; // Strip callback fragment

  try {
    const resp = await fetch(OAUTH_TOKEN_URL, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        grant_type: "authorization_code",
        code: cleanCode,
        redirect_uri: OAUTH_REDIRECT_URI,
        client_id: OAUTH_CLIENT_ID,
        code_verifier: pending.verifier,
        state,
      }),
    });

    if (!resp.ok) {
      const err = await resp.text();
      return res.status(502).json({ error: `Token exchange failed (${resp.status}): ${err}` });
    }

    const tokens = await resp.json();
    writeCredentials(tokens.access_token, tokens.refresh_token, tokens.expires_in);
    res.json({ ok: true });
  } catch (err) {
    res.status(500).json({ error: "Token exchange failed" });
  }
});
```

---

## Token Lifecycle

### `writeCredentials()`

Persists OAuth tokens in the format the Claude CLI expects. Sets file
permissions to `0o600` (user read/write only).

```typescript
import { existsSync, mkdirSync, writeFileSync, readFileSync } from "fs";
import { join } from "path";
import { homedir } from "os";

const CLAUDE_CREDS_PATH = join(homedir(), ".claude", ".credentials.json");

function writeCredentials(accessToken: string, refreshToken: string, expiresIn: number) {
  const dir = join(homedir(), ".claude");
  if (!existsSync(dir)) mkdirSync(dir, { recursive: true });
  const creds = {
    claudeAiOauth: {
      accessToken,
      refreshToken,
      expiresAt: Date.now() + expiresIn * 1000,
      scopes: ["user:inference", "user:mcp_servers", "user:profile", "user:sessions:claude_code"],
    },
  };
  writeFileSync(CLAUDE_CREDS_PATH, JSON.stringify(creds, null, 2) + "\n", { mode: 0o600 });
}
```

### `loadCredentials()`

Checks whether valid credentials exist. Call on server startup to determine
whether to show the setup screen or the main app.

```typescript
function loadCredentials(): { accessToken: string; expiresAt: number } | null {
  if (!existsSync(CLAUDE_CREDS_PATH)) return null;
  try {
    const data = JSON.parse(readFileSync(CLAUDE_CREDS_PATH, "utf-8"));
    const token = data?.claudeAiOauth?.accessToken;
    const expiresAt = data?.claudeAiOauth?.expiresAt ?? 0;
    if (typeof token === "string" && token.length > 0) return { accessToken: token, expiresAt };
    return null;
  } catch { return null; }
}
```

### `refreshTokenIfNeeded()`

Proactively refreshes the access token if it expires within 30 minutes.
Call before spawning Claude subprocesses to prevent mid-session auth failures.

**Important:** The refresh token rotates on every refresh — always persist
the new refresh token from the response.

```typescript
async function refreshTokenIfNeeded(): Promise<boolean> {
  const creds = loadCredentials();
  if (!creds?.accessToken) return false;

  let data: any;
  try { data = JSON.parse(readFileSync(CLAUDE_CREDS_PATH, "utf-8")); } catch { return false; }

  const refreshToken = data?.claudeAiOauth?.refreshToken;
  if (!refreshToken) return false;

  const thirtyMinutes = 30 * 60 * 1000;
  if (creds.expiresAt > Date.now() + thirtyMinutes) return false;

  try {
    const resp = await fetch(OAUTH_TOKEN_URL, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        grant_type: "refresh_token",
        refresh_token: refreshToken,
        client_id: OAUTH_CLIENT_ID,
        scope: OAUTH_SCOPES,
      }),
    });
    if (!resp.ok) return false;

    const tokens = await resp.json();
    writeCredentials(tokens.access_token, tokens.refresh_token, tokens.expires_in);
    return true;
  } catch { return false; }
}
```

---

## Integration Notes

These functions fit into the server lifecycle at specific points:

- **`loadCredentials()`** on server startup — determines whether to show the
  setup screen or the main app
- **`refreshTokenIfNeeded()`** before every `spawn("claude", ...)` call —
  prevents mid-session auth failures
- **`/api/health` endpoint** — returns `{ needsSetup: !loadCredentials() }`
  for the frontend to poll after OAuth exchange

```typescript
app.get("/api/health", (req, res) => {
  res.json({ needsSetup: !loadCredentials() });
});
```

---

## Frontend Setup Screen

### State Machine

```
States: idle → starting → idle (step 2) → exchanging → polling → complete
         ↓                   ↓                ↓
       error              error            error
```

State variables:

- `step`: 1 (authorize button) or 2 (paste code)
- `status`: `idle | starting | exchanging | polling | error`
- `state`: PKCE state string from `/api/oauth/start`
- `code`: authorization code pasted by user
- `error`: error message string

### `handleOAuthStart()`

Opens the Anthropic authorization page. The popup is opened **synchronously**
before any async work to avoid popup blockers.

```javascript
const handleOAuthStart = async () => {
  setStatus('starting');
  setError('');
  const popup = window.open('about:blank', '_blank');
  try {
    const res = await fetch('/api/oauth/start');
    const data = await res.json();
    if (!res.ok) { if (popup) popup.close(); setError(data.error); setStatus('error'); return; }
    setState(data.state);
    if (popup) popup.location.href = data.authUrl;
    else window.open(data.authUrl, '_blank');
    setStep(2);
    setStatus('idle');
  } catch (err) {
    if (popup) popup.close();
    setError(err.message);
    setStatus('error');
  }
};
```

### `handleOAuthExchange()`

Sends the authorization code to the server and polls `/api/health` until
credentials are confirmed.

```javascript
const handleOAuthExchange = async () => {
  if (!code.trim() || !state) return;
  setStatus('exchanging');
  setError('');
  try {
    const res = await fetch('/api/oauth/exchange', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ code: code.trim(), state }),
    });
    const data = await res.json();
    if (!res.ok) {
      if (data.error?.includes('expired')) { setStep(1); setCode(''); setError('Session expired. Start over.'); setStatus('idle'); }
      else { setError(data.error); setStatus('error'); }
      return;
    }
    setStatus('polling');
    for (let i = 0; i < 20; i++) {
      await new Promise(r => setTimeout(r, 1000));
      try {
        const h = await fetch('/api/health');
        const hd = await h.json();
        if (!hd.needsSetup) { onComplete(); return; }
      } catch {}
    }
    setError('Claude process did not start. Check server logs.');
    setStatus('error');
  } catch (err) { setError(err.message); setStatus('error'); }
};
```

### `<SetupScreen>` Component

A complete, self-contained React component with a two-step OAuth flow.
Neutral styling — clean and minimal, easy to customize for any app.

Props: `onComplete` callback, fired when credentials are confirmed.

```jsx
function SetupScreen({ onComplete }) {
  const [step, setStep] = React.useState(1);
  const [status, setStatus] = React.useState('idle');
  const [state, setState] = React.useState('');
  const [code, setCode] = React.useState('');
  const [error, setError] = React.useState('');

  const handleOAuthStart = async () => {
    setStatus('starting');
    setError('');
    const popup = window.open('about:blank', '_blank');
    try {
      const res = await fetch('/api/oauth/start');
      const data = await res.json();
      if (!res.ok) { if (popup) popup.close(); setError(data.error); setStatus('error'); return; }
      setState(data.state);
      if (popup) popup.location.href = data.authUrl;
      else window.open(data.authUrl, '_blank');
      setStep(2);
      setStatus('idle');
    } catch (err) {
      if (popup) popup.close();
      setError(err.message);
      setStatus('error');
    }
  };

  const handleOAuthExchange = async () => {
    if (!code.trim() || !state) return;
    setStatus('exchanging');
    setError('');
    try {
      const res = await fetch('/api/oauth/exchange', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ code: code.trim(), state }),
      });
      const data = await res.json();
      if (!res.ok) {
        if (data.error?.includes('expired')) { setStep(1); setCode(''); setError('Session expired. Start over.'); setStatus('idle'); }
        else { setError(data.error); setStatus('error'); }
        return;
      }
      setStatus('polling');
      for (let i = 0; i < 20; i++) {
        await new Promise(r => setTimeout(r, 1000));
        try {
          const h = await fetch('/api/health');
          const hd = await h.json();
          if (!hd.needsSetup) { onComplete(); return; }
        } catch {}
      }
      setError('Claude process did not start. Check server logs.');
      setStatus('error');
    } catch (err) { setError(err.message); setStatus('error'); }
  };

  const containerStyle = {
    display: 'flex', flexDirection: 'column', alignItems: 'center',
    justifyContent: 'center', minHeight: '100vh', fontFamily: 'system-ui, sans-serif',
    padding: '2rem', backgroundColor: '#fafafa',
  };
  const cardStyle = {
    backgroundColor: '#fff', borderRadius: '12px', padding: '2.5rem',
    maxWidth: '420px', width: '100%', boxShadow: '0 1px 3px rgba(0,0,0,0.1)',
  };
  const headingStyle = { margin: '0 0 0.5rem', fontSize: '1.5rem', fontWeight: 600 };
  const textStyle = { color: '#666', lineHeight: 1.6, margin: '0 0 1.5rem' };
  const buttonStyle = {
    width: '100%', padding: '0.75rem 1.5rem', fontSize: '1rem', fontWeight: 500,
    border: 'none', borderRadius: '8px', cursor: 'pointer',
    backgroundColor: '#1a1a1a', color: '#fff',
  };
  const inputStyle = {
    width: '100%', padding: '0.75rem', fontSize: '1rem', border: '1px solid #ddd',
    borderRadius: '8px', marginBottom: '1rem', boxSizing: 'border-box',
  };
  const errorStyle = {
    backgroundColor: '#fef2f2', color: '#dc2626', padding: '0.75rem',
    borderRadius: '8px', marginBottom: '1rem', fontSize: '0.875rem',
  };
  const secondaryStyle = {
    ...buttonStyle, backgroundColor: 'transparent', color: '#666',
    border: '1px solid #ddd', marginTop: '0.5rem',
  };

  return React.createElement('div', { style: containerStyle },
    React.createElement('div', { style: cardStyle },
      error && React.createElement('div', { style: errorStyle }, error),

      step === 1 ? React.createElement(React.Fragment, null,
        React.createElement('h1', { style: headingStyle }, 'Connect Your Account'),
        React.createElement('p', { style: textStyle },
          'Sign in with your Anthropic account to get started. ',
          'You\'ll be redirected to claude.ai to authorize access.'
        ),
        React.createElement('button', {
          style: { ...buttonStyle, opacity: status === 'starting' ? 0.6 : 1 },
          onClick: handleOAuthStart,
          disabled: status === 'starting',
        }, status === 'starting' ? 'Opening...' : 'Sign in with Anthropic')
      ) : React.createElement(React.Fragment, null,
        React.createElement('h1', { style: headingStyle }, 'Paste Your Code'),
        React.createElement('p', { style: textStyle },
          'After authorizing, you\'ll see a code on the Anthropic page. ',
          'Copy it and paste it below.'
        ),
        React.createElement('input', {
          style: inputStyle, type: 'text', placeholder: 'Paste authorization code here',
          value: code, onChange: (e) => setCode(e.target.value),
          disabled: status === 'exchanging' || status === 'polling',
        }),
        React.createElement('button', {
          style: { ...buttonStyle, opacity: !code.trim() || status !== 'idle' ? 0.6 : 1 },
          onClick: handleOAuthExchange,
          disabled: !code.trim() || status !== 'idle',
        }, status === 'exchanging' ? 'Connecting...' : status === 'polling' ? 'Verifying...' : 'Connect'),
        React.createElement('button', {
          style: secondaryStyle,
          onClick: () => { setStep(1); setCode(''); setError(''); },
          disabled: status === 'exchanging' || status === 'polling',
        }, 'Back')
      )
    )
  );
}
```
