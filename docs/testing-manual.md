# GoClaw — Manual Test Guide

## Test Environment Setup

### Standard Edition
```bash
source .env.local && ./goclaw
# API: http://localhost:8080
# Web UI: http://localhost:8080 (or http://localhost:5173 in dev mode)
```

### Desktop Lite Edition
```bash
cd ui/desktop && wails dev -tags sqliteonly
# Desktop window opens, API on localhost:18790
```

### WebSocket Client
```bash
# Install wscat if needed
npm install -g wscat

# Connect
wscat -c "ws://localhost:8080/ws"
# Send connect frame:
{"method":"connect","params":{"token":"YOUR_TOKEN","locale":"en"}}
```

---

## Core Feature Test Plans

### TC-001: Agent Creation

**Preconditions:** Logged in as tenant admin

| Step | Action | Expected Result |
|---|---|---|
| 1 | Navigate to Agents → New Agent | Creation form displayed |
| 2 | Enter name, select model, set system prompt | Fields accept input |
| 3 | Click Save | Agent created, redirected to agent detail |
| 4 | Verify agent appears in list | Agent visible with correct name |
| 5 | Switch to another tenant | Previous agent NOT visible |

**Pass criteria:** Agent created, isolated to correct tenant.
**Edge cases:** Empty name (should fail), name > 255 chars (should truncate or fail), duplicate agent_key (should auto-suffix).

---

### TC-002: Chat via WebSocket

**Preconditions:** Agent created with valid LLM provider configured

| Step | Action | Expected Result |
|---|---|---|
| 1 | Connect via WS: `{"method":"connect","params":{"token":"..."}}` | `{"method":"connect","result":{"user":...}}` |
| 2 | Send chat: `{"method":"chat","params":{"agentKey":"my-agent","message":"Hello"}}` | Streaming events start |
| 3 | Observe `event` frames with content chunks | Text streamed progressively |
| 4 | Wait for final `result` frame | Complete response received |
| 5 | Check session history | Message + response stored |

**Edge cases:** Empty message, extremely long message (> 100k chars), provider timeout.

---

### TC-003: Multi-Tenant Isolation

**Preconditions:** Two tenant accounts: Tenant A and Tenant B

| Step | Action | Expected Result |
|---|---|---|
| 1 | Log in as Tenant A, create agent "Alpha" | Agent created |
| 2 | Log in as Tenant B | Cannot see "Alpha" in agent list |
| 3 | Try to access Tenant A's agent via API with Tenant B token | 403 Forbidden |
| 4 | Verify sessions from Tenant A not visible to Tenant B | Session list empty for B |

**Pass criteria:** No cross-tenant data visible at any access level.

---

### TC-004: Desktop Lite Edition Limits

**Preconditions:** Running Desktop app (SQLite edition)

| Step | Action | Expected Result |
|---|---|---|
| 1 | Create 5 agents | All created successfully |
| 2 | Try to create 6th agent | Error: "Lite edition limit: max 5 agents" |
| 3 | Create 50 sessions across agents | All created |
| 4 | Try to create 51st session | Error: "Lite edition limit: max 50 sessions" |
| 5 | Try to create 2nd team | Error: "Lite edition limit: max 1 team" |

---

### TC-005: Telegram Channel Integration

**Preconditions:** Telegram bot token configured, channel active

| Step | Action | Expected Result |
|---|---|---|
| 1 | Send message to bot in Telegram | Bot responds |
| 2 | Send a markdown table | Table rendered as ASCII in `<pre>` block |
| 3 | Send a message triggering code block | Code in `<pre><code>` HTML |
| 4 | Send long response (> 4096 chars) | Message split into chunks |
| 5 | Send message with special HTML chars | Chars escaped, no HTML injection |

---

### TC-006: Mobile Web UI

**Preconditions:** Device or browser DevTools in mobile emulation (iPhone SE or similar)

| Step | Action | Expected Result |
|---|---|---|
| 1 | Load web UI on mobile viewport | No horizontal scroll on main pages |
| 2 | Tap any text input | No page zoom (font-size ≥ 16px) |
| 3 | Open a dialog | Dialog takes full screen on mobile |
| 4 | Scroll through chat messages | Background page does not scroll (overscroll-contain) |
| 5 | Open a dropdown inside a dialog | Dropdown is clickable (pointer-events-auto) |
| 6 | Rotate to landscape | Top bar reduces padding (landscape-compact) |

---

### TC-007: Provider Failover

**Preconditions:** Primary provider configured with invalid API key; fallback provider configured and valid

| Step | Action | Expected Result |
|---|---|---|
| 1 | Send a chat message | Initial request fails (invalid key) |
| 2 | Observe retry behavior | System retries with backoff |
| 3 | Verify fallback (if configured) | Response from fallback provider |
| 4 | Check error logs | Error logged but not exposed to user |

---

## Security Edge Case Tests

### File Path Traversal
```
GET /api/files?path=../../../../etc/passwd
Expected: 403 Forbidden + security.* log entry
```

### SSRF via Provider URL
```
POST /api/providers {"url": "http://169.254.169.254/latest/meta-data/"}
Expected: 400 Bad Request (SSRF protection)
```

### SQL Injection Probe
```
GET /api/agents?name='; DROP TABLE agents; --
Expected: 200 or 404 (parameterized query, no injection)
```

### Cross-Tenant API Key Reuse
```
Use Tenant A's API key to call Tenant B's endpoint
Expected: 403 Forbidden
```

---

## Regression Checklist for Every Release

- [ ] Agent CRUD (create, read, update, delete)
- [ ] Chat via WebSocket + HTTP `/v1/chat/completions`
- [ ] Multi-tenant isolation (cross-tenant 403 checks)
- [ ] Desktop Lite limits enforced
- [ ] Mobile: no input zoom, dialogs full-screen
- [ ] Channel connectivity (Telegram if configured)
- [ ] Migration ran successfully (`./goclaw migrate version`)
- [ ] Health check responds: `GET /health` → `{"status":"ok"}`
