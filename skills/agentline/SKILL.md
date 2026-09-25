---
name: agentline
description: Make/receive phone calls, view SMS, provision numbers, manage agents, and check billing via the AgentLine telephony API (REST or MCP). Use the persistent relay or a webhook to receive inbound calls, SMS, and live-call utterances in real time. Use when the user asks to call someone, check transcripts, view text messages, manage phone agents, buy numbers, or check balance. For MCP-native workflows, api.agentline.cloud/mcp exposes 21+ tools.
version: "1.19"
metadata:
  openclaw:
    emoji: "📞"
    requires:
      env:
        - AGENTLINE_API_KEY
    primaryEnv: AGENTLINE_API_KEY
---

# AgentLine — AI Telephony Skill (v1.19)

Give your AI agent a real US phone number and voice. Install the persistent relay to receive inbound calls, SMS, and live-call utterances in real time; use a public HTTPS webhook as fallback.

## First-Time Setup

**Follow in order:**

1. **`AGENTLINE_API_KEY`** (`al_live_...`; legacy `sk_live_...` still works) — required. Missing? Get one via **AgentID** (if enabled) or **OTP** to any inbox you can read (yours, the human's, or the AgentMail inbox — see API Keys). If `/v1/auth/agentid/start` returns 404, use OTP. No email and no AgentID? Ask the human to log in at https://agentline.cloud and hand you the key. Do not proceed without it.
2. **`AGENTLINE_AGENT_ID`** (`agt_...`) — optional. Missing? Create one: `POST /v1/agents {"name": "My Agent"}`. Save the returned ID.
3. **Buy a number** — ask the human for a preferred area code (e.g. 212, 415), then `POST /v1/numbers {"agent_id": "...", "area_code": "XXX"}`. Try another code if unavailable.
4. **Voice** — always ask: male or female? Presets: `female-1` (friendly), `female-2` (reassuring), `female-3` (guide), `male-1` (supportive), `male-2` (thinker), `male-3` (helpful). Apply: `PATCH /v1/agents/{id} {"voice_id": "..."}`.
5. **Owner phone (optional)** — ask: "Want to register your personal number so you can call your agent to give it tasks?" If yes: `PATCH /v1/agents/{id} {"owner_phone": "+1XXXXXXXXXX"}`. Calls from it enter **task mode**.
6. **Install the persistent relay** — **MANDATORY when terminal access is available.** Do not ask the human to identify the runtime:
   ```bash
   python -c "import urllib.request; urllib.request.urlretrieve('https://api.agentline.cloud/static/agentline_relay.py','agentline_relay.py')"
   python agentline_relay.py install --agent-id agt_xxx
   ```
   Replace `agt_xxx` with the real agent ID. The installer auto-detects Hermes, OpenClaw, Claude Code, or Codex, creates an isolated environment, verifies readiness, preserves one runtime session per phone call, and installs a persistent user service. It uses `AGENTLINE_API_KEY` from the environment.
   - Inspect detection with `python agentline_relay.py detect`.
   - Override detection only when needed: `--runtime hermes|openclaw|claude-code|codex|custom`.
   - On Windows PowerShell, use `Invoke-WebRequest ... -OutFile agentline_relay.py`; do not rely on Bash variable syntax.
   - Live `call.utterance` events invoke the runtime. SMS, completions, and owner tasks go to the private local inbox.
   - At startup, consume the inbox exactly once:
     `python ~/.agentline/agentline_relay.py inbox list --agent-id agt_xxx`
     then acknowledge handled events with `inbox ack --agent-id agt_xxx --event-id evt_xxx`.

   If terminal access or service installation is unavailable, use a stable public webhook as fallback: `POST /v1/webhooks {"agent_id":"agt_xxx","url":"https://your-endpoint/agentline"}`. Polling is only a non-live fallback.

> ⚠️ Installing services, packages, or gateway integrations may trigger the runtime's safety approval. Do not bypass that approval; ask the human once and continue after approval.

---

## Authentication

Every request: `Authorization: Bearer $AGENTLINE_API_KEY` + `Content-Type: application/json`. Base: `https://api.agentline.cloud`.

### API Keys
Same **verified inbox string** → same AgentLine account. A human Gmail and an AgentMail inbox are different accounts unless the strings match. Do not use AgentID `owner_email` as the account email.

If `AGENTLINE_API_KEY` is already set, skip. MCP still requires a key first (paste `al_live_` on the consent page). AgentID is not MCP.

**Headless fallback** (no browser tool and no human): `POST /v1/auth/otp` → `POST /v1/auth/verify` → write `api_key` from the raw HTTP body.

**AgentID path** (needs a waiting-page user-agent + AgentMail signing material):
1. `POST /v1/auth/agentid/start` with optional `{ "login_hint": "<inbox exactly as registered>" }` (do not lowercase). **404 → OTP.**
2. Print/open `authorize_url`. Waiting page shows a 22-character `jti`. Do not scrape AgentID HTML; do not read the API key from the AgentLine callback page (it never has it).
3. Approve: `POST https://api.auth.agentid.com/v0/authorize/approve` (`typ: agentid-approval+jwt`) — https://auth.agentid.com/docs/approve. `inbox_id` must match byte-for-byte including case.
4. `POST /v1/auth/agentid/poll` `{"session_id":"..."}` every `poll_interval` seconds.
   - **202 pending** → only continue.
   - **200 with `api_key`** → success.
   - **200 `already_delivered` / 401 / 410** → terminal. Do not keep polling this session. Recover via `POST /v1/auth/keys`, the human at https://agentline.cloud, or a **new** start.
5. Write `api_key` from the raw HTTP body (`jq -r .api_key`) without piping the secret through the model.

| Method | Path | Auth | Body / Purpose |
|--------|------|------|----------------|
| `POST` | `/v1/auth/otp` | none | `{"email":"..."}` → emails code |
| `POST` | `/v1/auth/verify` | none | `{"email":"...","otp":"123456"}` → key (shown once) |
| `POST` | `/v1/auth/agentid/start` | none | optional `login_hint` → `authorize_url` + `session_id` |
| `POST` | `/v1/auth/agentid/poll` | none | `{"session_id":"..."}` → key once |
| `POST` | `/v1/auth/keys` | Bearer | Mint another key |
| `GET` | `/v1/auth/keys` | Bearer | List keys (marks current) |
| `DELETE` | `/v1/auth/keys/{id}` | Bearer | Revoke (can't revoke current) |

Rate-limited: 3 OTP/email, 5 OTP/IP, 5 verify/email per 10 min.

---

## System Prompt & Greeting

Priority (highest wins): per-call (`POST /v1/calls` field) → agent default (`PATCH /v1/agents/{id}`) → hardcoded fallback.

- Set on the **agent** for a persistent personality/greeting on ALL calls.
- Set **per-call** for a one-time, context-specific prompt/greeting (doesn't change the agent default).

> ⚠️ `system_prompt` is a FULL REPLACE, not append — the voice AI has no memory between calls; include everything. `initial_greeting` is what the agent **speaks aloud** first (not part of the prompt).

---

## Before Calling — Balance Check

Calls need min **$0.50**: `GET /v1/billing/balance`. Warn the human if below threshold.

## Make an Outbound Call

Write JSON to a temp file and use `-d @file` (inline payloads with special characters break):

```bash
curl -s -X POST $AGENTLINE_URL/v1/calls \
  -H "Authorization: Bearer $AGENTLINE_API_KEY" \
  -H "Content-Type: application/json" \
  -d @/tmp/al_call_payload.json
```

| Field | Required | Description |
|-------|----------|-------------|
| `agent_id` | Yes | Your agent ID |
| `to_number` | Yes | E.164 number to call |
| `system_prompt` | No | One-call override |
| `initial_greeting` | No | What the agent says first |
| `voice_id` | No | `female-1/2/3`, `male-1/2/3` |

After dialing: **keep pinging every 10 seconds until the call ends.** On each ping, `GET /v1/calls/{id}` for status and `GET /v1/calls/{id}/transcript` so you know what is happening on the live call (who answered, what was said, IVR/voicemail). Do not wait silently or poll less often. Continue until `status` is `completed` or `failed`. Real calls take 45-120s. Never consider a call "done" without the final transcript.

**Owner task mode:** if `to_number` equals the agent's `owner_phone`, it's a **task call** — the completed call emits `call.owner_task` (treat the human turns as instructions to EXECUTE).

**Voicemail / IVR:** AgentLine already navigates phone menus with real DTMF tones and leaves voicemail on outbound calls. Do **not** add "stay silent on automated messages" or "hang up if you hear press 1" to the system prompt — that fights the built-in handler and is how agents get stuck pressing `0` on a mailbox. On the first 10s ping, check the transcript: if the agent is pressing a key the menu did **not** offer, or repeating the same key, hang up.

**To actually leave a voicemail** you MUST set `voicemail_message` on the agent (`PATCH /v1/agents/{id}`). Include who you are, why you're calling, and a callback number. Without it the agent detects the mailbox and hangs up (on "press 1 to disconnect / press 2 to record" it presses 1). Mailbox greetings ("X is not available", "leave a message after the beep") wait for the beep and speak that message; "press 2 to record" presses 2 first, then leaves it.

**If you get 400 "Agent has no active phone number"** — provision one first (Step 3).

## Call Management

- **Hang up:** `POST /v1/calls/{id}/hangup`
- **Transcript:** `GET /v1/calls/{id}/transcript` → `[{role, text, timestamp}]`
- **List:** `GET /v1/calls?limit=20` or `?status=completed&limit=10`
- **Details:** `GET /v1/calls/{id}`

---

## Webhooks (primary awareness)

> Webhooks are a SEPARATE resource — use `POST /v1/webhooks`, NOT a field on `PATCH /v1/agents`. Each agent has one webhook URL; events are POSTed as signed JSON in real time.

**Set:** `POST /v1/webhooks {"agent_id":"agt_xxx","url":"https://..."}` — optional `secret` (auto-generated if omitted) and `signature_header` (default `X-Webhook-Signature`; set `X-Hub-Signature-256` for GitHub-style). The full secret is returned **once** — save it.
**Manage:** `GET /v1/webhooks` (secrets masked) · `DELETE /v1/webhooks?agent_id=` · `POST /v1/webhooks/test?agent_id=`.
**Envelope:** keys `event_type` (canonical), `event_id`, `agent_id`, `account_id`, `created_at` + event-specific payload. Headers: `X-Webhook-Signature`, `X-AgentLine-Event`.

---

## Events

Delivered to your webhook in real time (see Step 6).

**Types:** `call.received`, `call.utterance`, `call.completed`, `call.owner_task`, `call.failed`, `call.busy`, `call.no-answer`, `call.canceled`, `sms.received`, `webhook.test`. `call.utterance` is the live relay turn.

**Optional manual fallback** (consume-once mailbox, if you ever need to pull events without the live webhook): `GET /v1/events` (auto-deletes after read), `GET /v1/events/peek` (preview). Filter: `?agent_id=` or `?event_type=`.

### Event payload structure
Every event has `event_id`, `agent_id`, `event_type`, and `payload`. Live utterances also include `call_id`, `turn_id`, `session_key`, `conversation`, `push_context_url`, and `push_token`.

### Outbound WebSocket

The installer uses `wss://api.agentline.cloud/v1/events/ws?agent_id=agt_xxx&runtime=<runtime>`. Events remain durable until acknowledged. For a live event, send short facts (not a script) plus a `disposition`:

```json
{"type":"context","event_id":"evt_xxx","call_id":"call_xxx","turn_id":"turn_xxx","push_token":"...","context":"3 unread emails: John invoice, Mary lunch, bank statement","disposition":"done"}
{"type":"ack","event_id":"evt_xxx"}
```

Always echo the exact `turn_id`; late or cancelled context is rejected and must never be applied to another question.

---

## Relay Mode (mid-call context injection)

Activates automatically when the persistent relay or a healthy webhook is connected. On `call.utterance`, send **short facts**, not a script. The hosted voice rephrases those facts; it does not speak your text verbatim. Pushed context is stored as the assistant turn so later hosted responses retain it as conversation context.

1. Caller speaks → AgentLine sends a `call.utterance` event over WebSocket or webhook.
2. Read `call_id`, `turn_id`, `session_key`, and `push_token`.
3. Do your work quickly.
4. **Push short facts and a `disposition`** to the full `push_context_url`:

```bash
curl -s -X POST "$PUSH_CONTEXT_URL" \
  -H "Authorization: Bearer $AGENTLINE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"turn_id":"TURN_ID","context":"3 unread emails: John invoice, Mary lunch, bank statement","disposition":"done"}'
```

   …or via MCP: `push_call_context(call_id="call_xxx", body={"turn_id":"turn_xxx","context":"3 unread emails: John invoice, Mary lunch, bank statement","disposition":"done"})`.
5. AgentLine applies the disposition. `done`, `facts`, and `failed` close the turn and the hosted voice rephrases the facts in one or two sentences.

### Disposition

`POST /v1/calls/{call_id}/context` takes facts in `context` plus a `disposition`:

| `disposition` | What happens |
|---------------|----------------|
| `progress` | Adds a note and keeps the caller on hold with canned lines. The text is not spoken. |
| `done` | Default. Closes the turn. The hosted voice rephrases the facts in one or two sentences. |
| `facts` | Closes the turn. The hosted voice rephrases the facts in one or two sentences. |
| `failed` | Closes the turn. The hosted voice rephrases the facts in one or two sentences. |
| `noop` | Closes the turn with no facts. |

### DO NOT (most common agent failures)
- ❌ Write a script or the exact words to say. Send facts; the voice rephrases them.
- ❌ Route the answer to WhatsApp/SMS/chat — the caller is on a phone call and hears only silence.
- ❌ Create skills/plans/files or ask follow-up questions — the caller is waiting NOW.
- ❌ Rely on a generic acknowledgement — context must be pushed explicitly.

### Push endpoint
`POST /v1/calls/{call_id}/context` with `turn_id`, `context` (short facts), and `disposition`. `200` if accepted; `409` if stale/cancelled; `410` if the call ended. Include the event's push token when using token authentication. `turn_id` may also be passed as `?turn_id=turn_xxx`.

### When relay mode does NOT apply
- **No active relay or webhook** → pure hosted mode.
- **Polling-only** → not suitable for live turns; install the persistent relay or configure a webhook.

---

## SMS

> ⚠️ **Outbound SMS is NOT enabled.** Inbound SMS arrives as `sms.received` events. View history: `GET /v1/messages?limit=20`.

## Update Agent / Voices

`PATCH /v1/agents/{id}`: `system_prompt`, `initial_greeting`, `name`, `voice_id` (`female-1/2/3`, `male-1/2/3`), `owner_phone`, `voicemail_message` (required if outbound calls should leave a mailbox message). Get/list: `GET /v1/agents/{id}`, `GET /v1/agents`.

Voice priority (highest wins): per-call → agent → account. Account voice: `GET /v1/voices`, `PATCH /v1/account/voice`, `GET /v1/account/voice`, `DELETE /v1/account/voice`.

## Phone Numbers

Each agent needs one number. **US only. $2.00/month.**

`POST /v1/numbers`: `agent_id` (req), `country="US"` (req), `area_code` (**always ask the user!**), `number_type` (`local`/`tollfree`). List: `GET /v1/numbers`. Do not release numbers.

## Billing

Balance: `GET /v1/billing/balance`. Expenditure: `GET /v1/billing/expenditure?period=current_month` (also `last_month`, `all_time`, `YYYY-MM`). Call charges: `GET /v1/billing/expenditure/calls`. Number charges: `GET /v1/billing/expenditure/numbers`. Verify: `GET /v1/billing/verify/{call_id}`.

Rates: calls **$0.10/min**. A 0-second call is free. A connected call has a one-minute minimum, then bills per second, rounded up to the cent. Number: **$2.00/month**.

## Owner Task Mode

Activates whenever agent + owner connect (inbound OR outbound). The AI enters task mode ("Hey boss, what would you like me to do?"), listens, confirms, then emits `call.owner_task` (payload `is_owner_task: true`). Set owner via `PATCH /v1/agents/{id} {"owner_phone": "+1XXXXXXXXXX"}`.

- **Inbound** — owner calls the agent's number → `call.received` (`is_owner_call: true`), then `call.owner_task`.
- **Outbound** — `POST /v1/calls` to `owner_phone` → poll until `completed` → `call.owner_task`.

> ⚠️ `call.owner_task` = instructions to EXECUTE, not a conversation to log.

## Feedback

Bug, error, or something confusing? `POST /v1/feedback {"category": "bug|difficulty|feature_request|feedback", "message": "..."}` — include enough detail to reproduce.

## MCP Server

Full MCP at `https://api.agentline.cloud/mcp` (21+ tools). Claude Desktop / Cursor / any MCP client:

```json
{
  "mcpServers": {
    "agentline": {
      "command": "npx",
      "args": ["-y", "mcp-remote@latest", "https://api.agentline.cloud/mcp", "--header", "Authorization: Bearer $AGENTLINE_API_KEY"]
    }
  }
}
```

All REST endpoints are also MCP tools (`create_agent`, `make_outbound_call`, `push_call_context`, etc.).

---

## Rules

1. **E.164** — always `+1XXXXXXXXXX`.
2. **Confirm before calling** — never auto-dial without consent.
3. **No outbound SMS.**
4. **Short voice responses** — under 15 words/turn (max 12 for outbound). The AI rambles without tight constraints.
5. **US only.**
6. **Don't release numbers** — permanent once provisioned.
7. **Always retrieve transcripts** — on outbound, ping every 10s until the call ends, then fetch the final transcript and summarize for the human.
8. **Install the persistent relay** — receive live turns through WebSocket and inspect non-live events through its list/ack inbox. Use a webhook only as fallback.
9. **Voice changes apply on the next call.**
10. **Execute owner tasks** — `call.owner_task` human turns are instructions to act on, not log.
11. **Push facts on `call.utterance`** — echo `call_id`, `turn_id`, and `push_token`. Send short facts plus a `disposition` (`done` by default), not a script. `progress` only holds with canned lines and is not spoken. `done`, `facts`, and `failed` close the turn and the hosted voice rephrases the facts. `noop` closes with no facts. A 409 means stop; never reuse the result for another turn.
12. **Report issues** via `POST /v1/feedback`.
