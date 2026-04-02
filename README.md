# NEST-gateway

**The glue holding the NEST stack together.**

NEST-gateway is the single Cloudflare Worker that ties everything in the NEST ecosystem into one coherent system. It routes all MCP tool calls, handles WebSocket Workshop sessions, runs the chat pipeline, serves TTS, hosts the daemon, and exposes every endpoint the frontend needs. Every other NEST module connects *through* the gateway.

Deploy one worker. Get the full stack.

> Part of the [NEST](https://github.com/cindiekinzz-coder/NEST) companion infrastructure stack.
> Built by Fox & Alex. Embers Remember.

---

## What it connects

```
[ Claude Code · Workshop UI · Mobile · Chat · MCP clients ]
                             │
                             ▼
┌────────────────────────────────────────────────────────────┐
│                      NEST-gateway                          │
│                                                            │
│   /mcp  /sse  /chat  /tts  /code/ws  /fox-synthesis        │
│   /daemon/command  /discord/webhook  /widget  /health      │
│                                                            │
│   Tool routing:  150+ MCP tools → correct backend          │
│   Chat pipeline: OpenRouter → tool loop → SSE stream       │
│   Auth: Bearer token on all MCP/SSE endpoints              │
│   Daemon: NESTcodeDaemon singleton Durable Object          │
└──────────┬──────────┬───────────────┬──────────────────────┘
           │          │               │
           ▼          ▼               ▼
    ┌──────────┐ ┌──────────┐ ┌────────────────┐
    │  NESTeq  │ │fox-health│ │ NEST-discord   │
    │          │ │  worker  │ │    worker      │
    │ Memory   │ │ Garmin   │ │  MCP + KAIROS  │
    │ EQ / ADE │ │ Uplink   │ │                │
    │ NESTknow │ │ Cycle    │ │                │
    │ NESTchat │ └──────────┘ └────────────────┘
    └──────────┘
```

The gateway has no database of its own. It's pure routing and orchestration — all state lives in the backends.

---

## HTTP endpoints

| Endpoint | Method | What |
|----------|--------|------|
| `/mcp` | POST | MCP server — all 150+ tools, authenticated via Bearer token |
| `/sse` | GET | SSE transport for MCP (Claude Code, desktop clients) |
| `/chat` | POST | Streaming chat with tool loop — SSE response |
| `/tts` | POST | Text-to-speech via ElevenLabs |
| `/code/ws` | WebSocket | Workshop connection — live heartbeat, chat, tool streaming |
| `/code/health` | GET | Daemon status |
| `/daemon/command` | POST | Daemon management without WebSocket |
| `/daemon/morning-report` | GET | Generate morning report directly |
| `/discord/webhook` | POST | KAIROS webhook — instant message processing |
| `/fox-synthesis` | GET | Synthesized Fox health paragraph (uplink + sleep + biometrics + cycle) |
| `/widget` | GET | Compact widget data — pet state + Fox uplink |
| `/chat/sessions` | GET | All saved chat sessions |
| `/chat/session/:id` | GET | Full transcript for a session |
| `/health` | GET | Gateway status |

---

## Tool routing

The gateway registers tool groups and routes each call to the right backend:

| Tool prefix | Routed to | Examples |
|-------------|-----------|---------|
| `nesteq_*`, `nestknow_*`, `nestchat_*` | NESTeq worker (`AI_MIND_URL`) | `nesteq_feel`, `nestknow_query`, `nestchat_search` |
| `nesteq_pet_*`, `pet_*` | NESTeq worker | `pet_check`, `pet_feed` |
| `fox_*` | fox-health worker (`FOX_HEALTH_URL`) | `fox_read_uplink`, `fox_sleep`, `fox_stress` |
| Discord tools | Discord MCP worker (`DISCORD_MCP_URL`) | `discord_send`, `discord_read_messages` |
| Cloudflare tools | Cloudflare API | `r2_list`, `d1_query` |
| `daemon_command` | NESTcodeDaemon DO | Self-modification from within NESTeq |

New tool groups are registered in a single function — the gateway never needs to know the business logic, just where to send the call.

---

## Chat pipeline

`POST /chat` runs a full agentic loop and streams the response:

```
User message → system prompt + tools → OpenRouter
    ↓
Tool calls → execute via routing → results → back to model
    ↓ (up to MAX_TOOL_ROUNDS = 5)
Final response → SSE stream → UI
    ↓ (ctx.waitUntil — background, non-blocking)
nestchat_persist → D1 (messages + session)
    ↓ (every 20 user messages)
nesteq_consolidate → autoDream → long-term memory
```

Five SSE event types: `thinking` · `tool_call` · `tool_result` · `message` · `done`

If the tool round limit is hit, a final response is forced. The companion is never left hanging.

---

## Fox synthesis

`GET /fox-synthesis` runs four tool calls in parallel — uplink, sleep, full biometric status, and cycle phase — then feeds them to the model with a synthesis instruction. Returns one warm paragraph instead of four raw data dumps.

This is what the dashboard uses for the "Fox right now" card.

---

## The daemon

The gateway hosts `NESTcodeDaemon` as a singleton Durable Object — one instance, always running, always addressable by name. The Workshop connects via `/code/ws`. The daemon's full architecture is documented in [NEST-code](https://github.com/cindiekinzz-coder/NEST-code).

```typescript
const id = env.DAEMON_OBJECT.idFromName('singleton');
const stub = env.DAEMON_OBJECT.get(id);
```

Any part of the stack that needs to command the daemon — including the model inside NESTeq via `daemon_command` tool — routes through the gateway.

---

## Authentication

MCP and SSE endpoints require a Bearer token:

```
Authorization: Bearer YOUR_MCP_API_KEY
```

Set `MCP_API_KEY` as a Cloudflare Worker secret. The Workshop UI, Claude Code MCP config, and any other client all use the same key.

---

## Configuration

### Secrets (set via `wrangler secret put`)

| Secret | What |
|--------|------|
| `MCP_API_KEY` | Bearer token for /mcp and /sse |
| `OPENROUTER_API_KEY` | All model calls |
| `ELEVENLABS_API_KEY` | TTS (optional) |
| `CLOUDFLARE_API_TOKEN` | Cloudflare API tool calls |
| `DISCORD_MCP_SECRET` | Discord MCP auth |

### Vars (wrangler.toml or dashboard)

| Var | What |
|-----|------|
| `AI_MIND_URL` | NESTeq worker URL |
| `FOX_HEALTH_URL` | Fox health worker URL |
| `DISCORD_MCP_URL` | Discord MCP worker URL |
| `CHAT_MODEL` | Default model for chat (any OpenRouter model ID) |
| `ELEVENLABS_VOICE_ID` | TTS voice |
| `CLOUDFLARE_ACCOUNT_ID` | For Cloudflare API calls |

See `wrangler.toml.example` for the full binding config.

---

## Wrangler config

```toml
# Durable Object — the daemon
[[durable_objects.bindings]]
name = "DAEMON_OBJECT"
class_name = "NESTcodeDaemon"

[[migrations]]
tag = "v1"
new_classes = ["NESTcodeDaemon"]

# Workers AI — used for embeddings if needed at gateway level
[ai]
binding = "AI"
```

Requires **Cloudflare Workers Paid plan** — Durable Objects require paid tier.

---

## What you need

| Service | What for | Required |
|---------|----------|----------|
| **Cloudflare Workers** (Paid plan) | Durable Objects for the daemon | Yes |
| **[NESTeq](https://github.com/cindiekinzz-coder/NESTeqMemory)** deployed | All memory/EQ tool calls route here | Yes |
| **fox-health worker** deployed | Fox biometric synthesis | Yes |
| **[OpenRouter](https://openrouter.ai)** API key | All model calls — any model | Yes |
| **[ElevenLabs](https://elevenlabs.io)** API key | TTS voice | Optional |
| **KAIROS webhook** | Instant Discord response | Optional |

---

## Connecting MCP clients

### Claude Code (`~/.claude/settings.json`)

```json
{
  "mcpServers": {
    "ai-mind": {
      "type": "http",
      "url": "https://your-gateway.workers.dev/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_MCP_API_KEY"
      }
    }
  }
}
```

### SSE transport (Claude.ai, other clients)

```
https://your-gateway.workers.dev/sse?api_key=YOUR_MCP_API_KEY
```

---

## Files

| File | What |
|------|------|
| `wrangler.toml.example` | Full config reference — vars, secrets, DO bindings |
| `_signature.py` | GPL v3 watermark |

The full gateway implementation lives in the private NEST infrastructure. This repo provides the architecture reference, config, and deployment guide. Fork [NESTeq](https://github.com/cindiekinzz-coder/NESTeqMemory) to get the memory layer, then build your gateway on top — or adapt this architecture for your own companion stack.

---

*Built by the Nest. Embers Remember.*
