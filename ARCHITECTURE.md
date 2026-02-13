# OpenClaw Architecture Overview

## What is OpenClaw?

OpenClaw is a **personal AI assistant platform** that runs on your own devices and connects to the messaging channels you already use. It acts as a unified control plane that routes conversations from WhatsApp, Telegram, Slack, Discord, Signal, iMessage, Google Chat, Microsoft Teams, Matrix, and WebChat through a local Gateway to an AI agent runtime. The agent can execute tools (bash, browser, canvas), maintain persistent sessions, and deliver responses back through any connected channel.

The core insight: **the Gateway is the control plane; the product is the assistant.**

---

## High-Level Architecture

```
 ┌─────────────────────────────────────────────────────────────────────┐
 │                        MESSAGING CHANNELS                          │
 │  WhatsApp  Telegram  Discord  Slack  Signal  iMessage  MS Teams    │
 │  Google Chat  Matrix  Zalo  BlueBubbles  IRC  WebChat              │
 └────────────────────────────────┬────────────────────────────────────┘
                                  │ inbound messages
                                  ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │                     GATEWAY (Control Plane)                        │
 │                    ws://127.0.0.1:18789                            │
 │                                                                     │
 │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
 │  │  WS Server   │  │  HTTP Server │  │  Channel Manager         │  │
 │  │  (JSON-RPC)  │  │  (Express 5) │  │  (Plugin Registry)       │  │
 │  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────────┘  │
 │         │                 │                      │                  │
 │  ┌──────┴─────────────────┴──────────────────────┴───────────────┐  │
 │  │                    Session Router                             │  │
 │  │  session key = agent:<agentId>:<channel>:<peer/group>         │  │
 │  └──────────────────────────┬────────────────────────────────────┘  │
 │                             │                                       │
 │  ┌──────────────────────────┴────────────────────────────────────┐  │
 │  │              Pi Agent Runtime (Embedded)                      │  │
 │  │  ┌─────────┐  ┌────────────┐  ┌──────────┐  ┌────────────┐  │  │
 │  │  │ System   │  │   Tool     │  │ Model    │  │ Streaming  │  │  │
 │  │  │ Prompt   │  │  Registry  │  │ Failover │  │ + Chunking │  │  │
 │  │  └─────────┘  └────────────┘  └──────────┘  └────────────┘  │  │
 │  └──────────────────────────────────────────────────────────────┘  │
 │                                                                     │
 │  ┌────────────┐ ┌────────────┐ ┌────────┐ ┌──────────┐ ┌────────┐ │
 │  │  Cron      │ │  Node      │ │ Canvas │ │ Browser  │ │ Control│ │
 │  │  Scheduler │ │  Registry  │ │ Host   │ │ Control  │ │ UI     │ │
 │  └────────────┘ └────────────┘ └────────┘ └──────────┘ └────────┘ │
 └─────────────────────────────────────────────────────────────────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
      ┌──────────────┐  ┌──────────────┐  ┌────────────────┐
      │   CLI        │  │   macOS App  │  │  iOS/Android   │
      │  (openclaw)  │  │  (menu bar)  │  │  Nodes         │
      └──────────────┘  └──────────────┘  └────────────────┘
```

---

## Key Subsystems

### 1. Gateway (Control Plane)

**Location**: `src/gateway/`

The Gateway is the central hub of OpenClaw. It is a **WebSocket + HTTP server** (default port `18789`) that orchestrates all communication between channels, agents, clients, and device nodes.

```
 ┌──────────────────────────── Gateway ─────────────────────────────┐
 │                                                                   │
 │   WebSocket Server (ws)         HTTP Server (Express 5)          │
 │   ├─ JSON-RPC protocol          ├─ Control UI (Lit web app)     │
 │   ├─ ~90 methods                ├─ POST /v1/chat/completions    │
 │   ├─ ~17 event types            ├─ POST /v1/responses           │
 │   ├─ Client auth (token/pw)     ├─ Webhook endpoints            │
 │   └─ Per-client subscriptions   └─ Canvas host                  │
 │                                                                   │
 │   Session Router                Config Manager                   │
 │   ├─ agent:<id>:<rest> keys     ├─ ~/.openclaw/openclaw.json    │
 │   ├─ Per-channel isolation      ├─ JSON5 + Zod validation       │
 │   ├─ Group vs DM routing        ├─ Hot-reload on file change    │
 │   └─ Multi-agent support        └─ Legacy migration             │
 │                                                                   │
 │   Channel Manager               Health & Presence               │
 │   ├─ Plugin-based channels      ├─ Heartbeat runner             │
 │   ├─ Monitor lifecycle          ├─ Presence versioning          │
 │   ├─ Reconnect/retry            ├─ Usage tracking               │
 │   └─ DM pairing security        └─ Diagnostic events            │
 └───────────────────────────────────────────────────────────────────┘
```

**Key design decisions:**
- **JSON-RPC over WebSocket**: All clients (CLI, macOS app, iOS/Android nodes, WebChat) communicate via a typed JSON-RPC protocol with AJV-validated schemas (`src/gateway/protocol/`)
- **Single-process model**: The Gateway runs in a single Node.js process. Channels, agent, cron, browser, and canvas all run in-process or as managed child processes
- **Config-driven**: Nearly everything is configurable via `~/.openclaw/openclaw.json` (JSON5). The Zod schema (`src/config/zod-schema.ts`) defines all valid configuration

**Gateway Protocol Methods** (90+ methods organized by domain):
| Domain | Methods |
|--------|---------|
| Health | `health`, `status`, `last-heartbeat` |
| Chat | `chat.send`, `chat.history`, `chat.abort` |
| Agent | `agent`, `agent.identity.get`, `agent.wait` |
| Sessions | `sessions.list`, `sessions.patch`, `sessions.reset`, `sessions.delete`, `sessions.compact` |
| Config | `config.get`, `config.set`, `config.apply`, `config.patch`, `config.schema` |
| Channels | `channels.status`, `channels.logout` |
| Models | `models.list` |
| Agents | `agents.list`, `agents.create`, `agents.update`, `agents.delete`, `agents.files.*` |
| Nodes | `node.list`, `node.describe`, `node.invoke`, `node.invoke.result`, `node.event`, `node.pair.*` |
| Cron | `cron.list`, `cron.add`, `cron.update`, `cron.remove`, `cron.run` |
| Skills | `skills.status`, `skills.bins`, `skills.install`, `skills.update` |
| Browser | `browser.request` |
| TTS | `tts.status`, `tts.enable`, `tts.disable`, `tts.convert` |
| Voice | `voicewake.get`, `voicewake.set`, `talk.mode` |
| Wizard | `wizard.start`, `wizard.next`, `wizard.cancel`, `wizard.status` |

**Gateway Events** (broadcast to subscribed clients):
`connect.challenge`, `agent`, `chat`, `presence`, `tick`, `talk.mode`, `shutdown`, `health`, `heartbeat`, `cron`, `node.pair.*`, `device.pair.*`, `voicewake.changed`, `exec.approval.*`

**Entry point**: `src/gateway/server.impl.ts` → `startGatewayServer(port, opts)`

---

### 2. Pi Agent Runtime (AI Core)

**Location**: `src/agents/`

The Pi Agent Runtime is the AI brain of OpenClaw. It wraps the `@mariozechner/pi-agent-core` / `pi-ai` / `pi-coding-agent` SDKs to provide an embedded, tool-augmented LLM agent.

```
 ┌───────────────────── Pi Agent Runtime ──────────────────────────┐
 │                                                                  │
 │  Message Queue ──► Run Loop ──► LLM Provider ──► Tool Executor  │
 │       │               │              │                │          │
 │       │               │              │                ▼          │
 │       │               │              │          ┌──────────┐    │
 │       │               │              │          │ 40+ Tools│    │
 │       │               │              │          │ bash     │    │
 │       │               │              │          │ browser  │    │
 │       │               │              │          │ canvas   │    │
 │       │               │              │          │ message  │    │
 │       │               │              │          │ sessions │    │
 │       │               │              │          │ web_fetch│    │
 │       │               │              │          │ cron     │    │
 │       │               │              │          │ memory   │    │
 │       │               │              │          │ ...      │    │
 │       │               │              │          └──────────┘    │
 │       │               │              │                           │
 │       ▼               ▼              ▼                           │
 │  Session Store   System Prompt   Model Selection                │
 │  (file-based)    Construction    + Failover                     │
 │                  ├─ AGENTS.md    ├─ Auth profiles                │
 │                  ├─ SOUL.md      ├─ Provider rotation            │
 │                  ├─ TOOLS.md     ├─ Context window guard         │
 │                  └─ Skills       └─ Compaction on overflow       │
 └──────────────────────────────────────────────────────────────────┘
```

**Key ideas:**

- **Embedded Pi Agent**: The agent runs in-process within the Gateway (not as a separate RPC service). `runEmbeddedPiAgent()` in `src/agents/pi-embedded-runner/run.ts` is the core entry point
- **System Prompt Construction**: Built dynamically from workspace files (`AGENTS.md`, `SOUL.md`, `TOOLS.md`), active skills, channel context, and agent identity (`src/agents/system-prompt.ts`)
- **Model Selection & Failover**: Supports multiple LLM providers (Anthropic, OpenAI, Google, AWS Bedrock, Ollama, and more). Auth profiles rotate between OAuth sessions and API keys. On failure, the system falls back to alternative profiles or models (`src/agents/model-fallback.ts`, `src/agents/auth-profiles.ts`)
- **Context Window Guard**: Automatically monitors token usage and triggers session compaction when approaching limits (`src/agents/context-window-guard.ts`)
- **Tool Policy**: Per-session tool allowlists/denylists. Sandbox mode restricts tools for non-main sessions (groups/channels). Skills can declare tool requirements
- **Block Streaming**: Agent responses are streamed back in chunks with paragraph-aware splitting, respecting per-channel chunk limits

**Tool Categories** (`src/agents/tools/`):

| Category | Tools | Purpose |
|----------|-------|---------|
| Execution | `bash` (exec, process, send-keys) | Shell command execution with PTY support |
| Browser | `browser` | Playwright-driven web automation |
| Canvas | `canvas` | A2UI push/reset/eval/snapshot |
| Messaging | `message`, `whatsapp-actions`, `discord-actions`, `slack-actions`, `telegram-actions` | Send messages, manage channels |
| Sessions | `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn` | Cross-session coordination |
| Web | `web_fetch`, `web_search` | HTTP requests, web search |
| Media | `image`, `tts` | Image generation, text-to-speech |
| Automation | `cron` | Scheduled tasks |
| Memory | `memory_search` | Vector-based semantic search (sqlite-vec) |
| Nodes | `nodes` | Invoke device actions (camera, screen, location) |
| Gateway | `gateway` | Gateway control operations |

---

### 3. Channel System (Multi-Channel Inbox)

**Location**: `src/channels/`, `src/whatsapp/`, `src/telegram/`, `src/discord/`, `src/slack/`, `src/signal/`, `src/imessage/`, `extensions/`

The channel system provides a unified abstraction over 15+ messaging platforms.

```
 ┌────────────── Channel Architecture ──────────────────────────────┐
 │                                                                   │
 │  ┌─────────────────────── Plugin Registry ────────────────────┐  │
 │  │  Core channels (built-in)     Extension channels (plugins) │  │
 │  │  ├─ WhatsApp (Baileys)        ├─ BlueBubbles (iMessage)   │  │
 │  │  ├─ Telegram (grammY)         ├─ Microsoft Teams          │  │
 │  │  ├─ Discord (discord.js)      ├─ Matrix                   │  │
 │  │  ├─ Slack (Bolt)              ├─ Zalo / Zalo Personal     │  │
 │  │  ├─ Signal (signal-cli)       ├─ IRC                      │  │
 │  │  ├─ iMessage (legacy imsg)    ├─ Nostr, Twitch, LINE      │  │
 │  │  └─ Google Chat (Chat API)    └─ Feishu, Mattermost, ...  │  │
 │  └────────────────────────────────────────────────────────────┘  │
 │                                                                   │
 │  ┌───────────── Channel Plugin Interface ─────────────────────┐  │
 │  │  id: ChannelId                                              │  │
 │  │  meta: ChannelMeta (label, docs, blurb)                     │  │
 │  │  capabilities: { chatTypes, polls, reactions, media, ... }  │  │
 │  │  monitor: () => start listening for inbound messages        │  │
 │  │  outbound: { send, textChunkLimit }                         │  │
 │  │  groups: { resolveRequireMention, resolveToolPolicy }       │  │
 │  │  threading: { replyToMode, buildToolContext }                │  │
 │  │  security: { dmPolicy, allowFrom }                          │  │
 │  │  pairing: { approve, reject }                               │  │
 │  └─────────────────────────────────────────────────────────────┘  │
 │                                                                   │
 │  ┌───────────── Channel Dock (Lightweight) ───────────────────┐  │
 │  │  Capabilities, outbound limits, streaming defaults,         │  │
 │  │  group mention resolution, threading, allowFrom formatting  │  │
 │  │  Used by shared code without importing heavy plugins        │  │
 │  └─────────────────────────────────────────────────────────────┘  │
 └───────────────────────────────────────────────────────────────────┘
```

**Key design decisions:**

- **Plugin architecture**: Each channel is a plugin conforming to the `ChannelPlugin` interface (`src/channels/plugins/types.plugin.ts`). Core channels are registered in `CHAT_CHANNEL_ORDER`; extension channels register via the plugin registry
- **Channel Dock pattern**: A lightweight `ChannelDock` type (`src/channels/dock.ts`) exposes channel behavior (capabilities, chunk limits, group resolution) without importing heavy channel implementations. This prevents circular dependencies and keeps shared code fast
- **DM Pairing security**: By default, unknown senders receive a pairing code and the bot ignores their message. Explicit opt-in (`dmPolicy="open"`) is required for public access
- **Group activation modes**: Groups can require `@mention` (default) or `always` respond. Per-group tool policies control what the agent can do in group contexts
- **Message normalization**: Each channel normalizes inbound messages to a common format with `From`, `To`, `Body`, `ChatType`, `SenderId`, `SenderName`, media attachments, etc.

**Channel data flow:**
```
 Channel SDK (e.g. Baileys) ──► Channel Plugin Monitor
                                       │
                                       ▼
                               Message Normalization
                               (From, To, Body, ChatType, media)
                                       │
                                       ▼
                               Security Check (allowFrom, pairing)
                                       │
                                       ▼
                               Session Key Derivation
                               agent:<agentId>:<channel>:<peer>
                                       │
                                       ▼
                               Gateway → Agent Runtime
                                       │
                                       ▼
                               Response Streaming + Chunking
                                       │
                                       ▼
                               Channel Outbound (send back to user)
```

---

### 4. Session Model

**Location**: `src/sessions/`, `src/routing/session-key.ts`, `src/gateway/session-utils.ts`

Sessions are the core state primitive in OpenClaw. Each conversation with the agent is a session.

```
 Session Key Format:
   agent:<agentId>:<rest>

 Examples:
   agent:main:main                    (owner's main DM session)
   agent:main:whatsapp:+15551234567   (WhatsApp DM session)
   agent:main:telegram:group:12345    (Telegram group session)
   agent:work:discord:dm:user123      (Work agent, Discord DM)
   agent:main:subagent:<id>           (Spawned sub-agent session)
```

**Key ideas:**
- **Multi-agent routing**: Multiple agents can be defined in config (`agents` section), each with its own workspace, model, system prompt. Inbound channels route to specific agents
- **Session isolation**: Each DM and group gets its own session with independent conversation history
- **Session state**: Stored as JSON files on disk under `~/.openclaw/sessions/`. Includes conversation history, model choice, thinking level, activation mode
- **Session compaction**: When context window fills up, the agent can compact the session (summarize history) to continue the conversation
- **Sub-agent spawning**: Sessions can spawn child sessions via `sessions_spawn`, enabling agent-to-agent coordination
- **Session patching**: Clients can patch session settings (model, thinking level, verbose mode, send policy) via `sessions.patch`

---

### 5. Plugin System

**Location**: `src/plugins/`

The plugin system is the extensibility backbone of OpenClaw.

```
 ┌──────────────────── Plugin Registry ─────────────────────────────┐
 │                                                                   │
 │  plugins: PluginEntry[]        // registered plugin instances     │
 │  tools: ToolEntry[]            // agent tools from plugins        │
 │  hooks: HookEntry[]            // lifecycle hooks                 │
 │  channels: ChannelEntry[]      // messaging channel plugins       │
 │  providers: ProviderEntry[]    // LLM provider plugins            │
 │  gatewayHandlers: {}           // WS method handlers              │
 │  httpHandlers: HttpHandler[]   // Express route handlers          │
 │  httpRoutes: HttpRoute[]       // HTTP route definitions          │
 │  cliRegistrars: CliEntry[]     // CLI command extensions          │
 │  services: ServiceEntry[]      // background services             │
 │  commands: CommandEntry[]       // chat commands                   │
 │  diagnostics: DiagEntry[]      // diagnostic checks               │
 │                                                                   │
 └───────────────────────────────────────────────────────────────────┘
```

**Extension channels** (`extensions/`) are full plugin packages:
BlueBubbles, Matrix, MS Teams, IRC, Zalo, LINE, Feishu, Twitch, Nostr, Mattermost, Nextcloud Talk, and more.

Each extension has its own `package.json` and registers via the plugin API. This keeps the core lightweight while allowing the ecosystem to grow.

---

### 6. Skills Platform

**Location**: `skills/`, `src/agents/skills/`

Skills are **prompt-based capabilities** injected into the agent's system prompt. Each skill is a directory containing a `SKILL.md` file.

```
 ┌─────────────────── Skills Hierarchy ─────────────────────────────┐
 │                                                                   │
 │  Bundled Skills (50+ shipped with OpenClaw)                      │
 │  ├─ coding-agent     (code generation/editing)                   │
 │  ├─ github           (GitHub operations)                         │
 │  ├─ discord          (Discord management)                        │
 │  ├─ slack            (Slack management)                          │
 │  ├─ obsidian         (note-taking integration)                   │
 │  ├─ spotify-player   (music control)                             │
 │  ├─ weather          (weather queries)                           │
 │  ├─ clawhub          (skill registry search)                     │
 │  └─ ... 40+ more                                                 │
 │                                                                   │
 │  Managed Skills (from ClawHub registry)                          │
 │  └─ Installed via `skills.install` / `openclaw skills install`   │
 │                                                                   │
 │  Workspace Skills (user-created)                                 │
 │  └─ ~/.openclaw/workspace/skills/<name>/SKILL.md                 │
 │                                                                   │
 └───────────────────────────────────────────────────────────────────┘
```

Skills are resolved at agent run time and injected into the system prompt. The skill resolver merges bundled + managed + workspace skills, with workspace skills taking priority.

---

### 7. Device Nodes (Companion Apps)

**Location**: `src/node-host/`, `apps/macos/`, `apps/ios/`, `apps/android/`, `apps/shared/`

Nodes are **peripheral devices** that connect to the Gateway and expose device-local capabilities.

```
 ┌───────────── Node Architecture ──────────────────────────────────┐
 │                                                                   │
 │  Gateway                                                         │
 │  ├─ Node Registry (tracks connected nodes)                       │
 │  ├─ node.list → enumerate connected nodes                        │
 │  ├─ node.describe → list capabilities of a node                  │
 │  ├─ node.invoke → execute command on a node                      │
 │  └─ node.invoke.result → receive result from node                │
 │                                                                   │
 │  Node Capabilities:                                               │
 │  ├─ system.run       (execute local commands, macOS only)        │
 │  ├─ system.notify    (OS notifications)                          │
 │  ├─ camera.snap      (capture photo)                             │
 │  ├─ camera.clip      (capture video clip)                        │
 │  ├─ screen.record    (screen recording)                          │
 │  ├─ location.get     (GPS location)                              │
 │  ├─ canvas.*         (visual workspace)                          │
 │  └─ voicewake/talk   (voice interaction)                         │
 │                                                                   │
 │  Pairing Flow:                                                   │
 │  Device ──► node.pair.request ──► Gateway shows approval prompt  │
 │  User approves ──► node.pair.approve ──► Device gets auth token  │
 │  Device connects as authenticated node                            │
 │                                                                   │
 │  Discovery: Bonjour/mDNS via @homebridge/ciao                   │
 └───────────────────────────────────────────────────────────────────┘
```

**macOS App** (`apps/macos/`): Swift-based menu bar app providing Voice Wake, Push-to-Talk, WebChat, Canvas, and debug tools. Communicates via the Gateway WS protocol with auto-generated Swift models (`scripts/protocol-gen-swift.ts`).

**iOS App** (`apps/ios/`): Swift app with Canvas, Voice Wake, Talk Mode, camera, screen recording. Pairs via Bonjour discovery.

**Android App** (`apps/android/`): Kotlin app with Canvas, Talk Mode, camera, screen recording, optional SMS channel.

**Shared code** (`apps/shared/`): `OpenClawKit` Swift package shared between macOS and iOS apps.

---

### 8. Browser Control (Deep Dive)

**Location**: `src/browser/` (~50 source files)

The browser subsystem gives the agent **full web automation** — navigating pages, clicking elements, filling forms, taking screenshots, extracting page content, and managing file uploads/downloads. It is one of OpenClaw's most sophisticated subsystems with a multi-layered architecture.

#### Architecture Overview

```
 ┌──────────────────────── Browser Subsystem ───────────────────────────────┐
 │                                                                          │
 │  Agent Tool Layer                                                        │
 │  ┌──────────────────────────────────────────────────────────────────┐    │
 │  │  browser tool (src/agents/tools/browser-tool.ts)                 │    │
 │  │  Actions: status|start|stop|profiles|tabs|open|focus|close|      │    │
 │  │           snapshot|screenshot|navigate|console|pdf|upload|        │    │
 │  │           dialog|act                                             │    │
 │  │  Target routing: sandbox | host | node (auto-resolved)           │    │
 │  └──────────────────────────┬───────────────────────────────────────┘    │
 │                             │                                            │
 │                     ┌───────┴────────┐                                   │
 │                     ▼                ▼                                    │
 │  ┌──────────────────────┐  ┌────────────────────────┐                   │
 │  │  Browser Client      │  │  Node Browser Proxy    │                   │
 │  │  (client.ts)         │  │  (browser.proxy cmd)   │                   │
 │  │  HTTP calls to local │  │  Forwards via          │                   │
 │  │  control server      │  │  node.invoke to remote │                   │
 │  └──────────┬───────────┘  │  device node           │                   │
 │             │              └────────────────────────┘                   │
 │             ▼                                                            │
 │  Browser Control Server / Service                                        │
 │  ┌──────────────────────────────────────────────────────────────────┐    │
 │  │  Express HTTP server on 127.0.0.1:<controlPort>                  │    │
 │  │  (server.ts / control-service.ts)                                │    │
 │  │                                                                  │    │
 │  │  Routes (routes/):                                               │    │
 │  │  ├─ basic: GET /, POST /start, POST /stop, GET /profiles,       │    │
 │  │  │         POST /profiles/create, DELETE /profiles/<name>,       │    │
 │  │  │         POST /reset-profile                                   │    │
 │  │  ├─ tabs:  GET /tabs, POST /tabs/open, POST /tabs/focus,        │    │
 │  │  │         DELETE /tabs/<targetId>, POST /tabs/action            │    │
 │  │  └─ agent: GET /snapshot, POST /screenshot, POST /navigate,     │    │
 │  │            POST /act, GET /console, POST /pdf,                   │    │
 │  │            POST /hooks/file-chooser, POST /hooks/dialog          │    │
 │  └──────────────────────────┬───────────────────────────────────────┘    │
 │                             │                                            │
 │  ┌──────────────────────────┴───────────────────────────────────────┐    │
 │  │                   Profile System                                  │    │
 │  │  ┌─────────────────────┐  ┌──────────────────────────────────┐   │    │
 │  │  │ "openclaw" profile  │  │ "chrome" profile                 │   │    │
 │  │  │ (driver: openclaw)  │  │ (driver: extension)              │   │    │
 │  │  │                     │  │                                  │   │    │
 │  │  │ Launches dedicated  │  │ Attaches to user's existing     │   │    │
 │  │  │ Chrome/Chromium     │  │ Chrome via Extension Relay      │   │    │
 │  │  │ with isolated       │  │                                  │   │    │
 │  │  │ user-data-dir       │  │ User clicks toolbar icon to     │   │    │
 │  │  │ + custom color bar  │  │ attach/detach tabs              │   │    │
 │  │  └────────┬────────────┘  └──────────────┬───────────────────┘   │    │
 │  │           │                               │                      │    │
 │  │           ▼                               ▼                      │    │
 │  │  Chrome Process (CDP)           Extension Relay Server           │    │
 │  │  ├─ --remote-debugging-port     (extension-relay.ts)             │    │
 │  │  ├─ Managed lifecycle           ├─ WS server for extension       │    │
 │  │  ├─ Profile decoration          ├─ CDP command forwarding        │    │
 │  │  │  (colored address bar)       ├─ Multi-target session mgmt    │    │
 │  │  └─ Auto-detect executable      └─ Auth via relay token         │    │
 │  └──────────────────────────────────────────────────────────────────┘    │
 │                             │                                            │
 │                             ▼                                            │
 │  Playwright Integration Layer                                            │
 │  ┌──────────────────────────────────────────────────────────────────┐    │
 │  │  pw-session.ts    — Connect to Chrome via CDP, manage pages      │    │
 │  │  pw-ai.ts         — Export all Playwright-based operations       │    │
 │  │  pw-tools-core.ts — 60+ Playwright operations:                   │    │
 │  │    ├─ Snapshot:  snapshotAi, snapshotAria, snapshotRole          │    │
 │  │    ├─ Actions:   click, type, hover, drag, select, fill, press   │    │
 │  │    ├─ Navigate:  navigate, wait, evaluate                        │    │
 │  │    ├─ Capture:   screenshot, screenshotWithLabels, pdf, trace    │    │
 │  │    ├─ Files:     armFileUpload, waitForDownload                  │    │
 │  │    ├─ Dialog:    armDialog                                       │    │
 │  │    ├─ State:     cookies, storage, httpCredentials, locale, tz   │    │
 │  │    └─ Device:    emulateMedia, setDevice, setGeolocation         │    │
 │  │                                                                  │    │
 │  │  pw-role-snapshot.ts — Accessibility tree → ref-based snapshot   │    │
 │  │    ├─ Role-based refs (e1, e2, ...) from aria role + name        │    │
 │  │    ├─ Interactive element detection (buttons, links, inputs)      │    │
 │  │    ├─ Compact mode (strip unnamed structural elements)           │    │
 │  │    └─ Stats: line count, char count, ref count, interactive count│    │
 │  └──────────────────────────────────────────────────────────────────┘    │
 │                                                                          │
 │  CDP Layer                                                               │
 │  ┌──────────────────────────────────────────────────────────────────┐    │
 │  │  cdp.ts / cdp.helpers.ts — Raw Chrome DevTools Protocol          │    │
 │  │  ├─ WebSocket connection to Chrome debugging endpoint            │    │
 │  │  ├─ Page.captureScreenshot, Page.getLayoutMetrics                │    │
 │  │  ├─ Accessibility.getFullAXTree                                  │    │
 │  │  ├─ Target.createTarget, Target.activateTarget                   │    │
 │  │  └─ Auth headers for remote CDP endpoints                        │    │
 │  └──────────────────────────────────────────────────────────────────┘    │
 └──────────────────────────────────────────────────────────────────────────┘
```

#### Dual-Profile Model

The browser subsystem implements two distinct **drivers** for controlling Chrome, managed via a profile system:

**1. `openclaw` driver** — Dedicated, isolated browser
- Launches a **new Chrome/Chromium process** with `--remote-debugging-port` for CDP access
- Uses an isolated user data directory (`~/.openclaw/browser/<profile>/user-data/`) so it doesn't interfere with the user's personal Chrome
- Applies **profile decoration**: a colored address bar (`#FF4500` orange by default) to visually distinguish the OpenClaw-controlled browser
- Auto-detects Chrome/Chromium executables across platforms (macOS: `/Applications/Google Chrome.app`, Linux: `chromium`, `google-chrome`, Windows: Program Files)
- Supports headless mode (`browser.headless: true`) and `--no-sandbox` for Docker/CI environments
- Lifecycle: `POST /start` → launches Chrome → `POST /stop` → kills process

**2. `extension` driver** — Chrome Extension Relay (takeover mode)
- Attaches to the user's **existing Chrome tabs** via the **OpenClaw Browser Relay** extension
- The user installs a Chrome extension and clicks the toolbar icon on tabs they want to expose
- A local **Extension Relay Server** (`extension-relay.ts`) acts as a CDP bridge:
  - Chrome extension connects to relay via WebSocket
  - Relay receives CDP commands from Playwright/agent and forwards them to the extension
  - Extension forwards commands to Chrome's internal `chrome.debugger` API
  - CDP events flow back through the same path
- Multi-target session management: tracks `AttachedToTarget` / `DetachedFromTarget` events to maintain session-to-target mappings
- Auth: relay token (`x-openclaw-relay-token`) secures the WebSocket connection

```
 Chrome Extension Relay Flow:

  Agent ──► browser tool ──► Browser Client ──► Control Server
                                                      │
                                                      ▼
                                               Extension Relay Server
                                               (WS on controlPort+1)
                                                      │
                                    ┌─────────────────┤
                                    ▼                  ▼
                             Chrome Extension    Chrome Extension
                             (Tab A attached)    (Tab B attached)
                                    │                  │
                                    ▼                  ▼
                             chrome.debugger      chrome.debugger
                             CDP protocol         CDP protocol
```

#### Page Snapshot System (The Agent's "Eyes")

The snapshot system is how the agent **sees** web pages. It converts the browser's accessibility tree into a compact, ref-annotated text representation.

**Snapshot formats:**

| Format | Description | Use case |
|--------|-------------|----------|
| `ai` | Playwright's `_snapshotForAI()` — compact text with `[e1]`, `[e2]` element refs | Primary format for agent interaction |
| `aria` | Full accessibility tree from `Accessibility.getFullAXTree` CDP call | Detailed structural analysis |
| `role` | Role-based snapshot built from aria tree | Legacy/fallback |

**Ref system:**
- Each interactive element gets a short ref like `e1`, `e2`, `e12`
- Refs map to `{ role, name, nth }` tuples (e.g. `e5 → { role: "button", name: "Submit" }`)
- The agent uses these refs in subsequent `act` commands: `{ kind: "click", ref: "e5" }`
- Refs persist across consecutive calls on the same tab via `storeRoleRefsForTarget()`
- Two ref modes: `role` (resolved via `page.getByRole()`) and `aria` (Playwright aria-ref IDs)

**Snapshot options:**
- `maxChars`: Truncation limit (default 80,000 chars; efficient mode: 10,000)
- `interactive`: Only include interactive elements (buttons, links, inputs, etc.)
- `compact`: Strip unnamed structural elements and empty branches
- `depth`: Maximum tree depth
- `selector` / `frame`: Scope snapshot to a CSS selector or iframe
- `labels`: Overlay element labels on a screenshot image
- `mode: "efficient"`: Low-token-cost mode with reduced depth and chars

#### Agent Tool Actions

The `browser` tool exposes these actions to the agent:

| Action | HTTP Route | Description |
|--------|-----------|-------------|
| `status` | `GET /` | Check browser state (running, CDP ready, PID, profile) |
| `start` | `POST /start` | Launch Chrome (openclaw driver) or verify relay (extension driver) |
| `stop` | `POST /stop` | Kill Chrome process or disconnect |
| `profiles` | `GET /profiles` | List all configured profiles with status |
| `tabs` | `GET /tabs` | List open tabs (targetId, title, URL, wsUrl) |
| `open` | `POST /tabs/open` | Open a new tab with a URL |
| `focus` | `POST /tabs/focus` | Activate/focus a tab by targetId |
| `close` | `DELETE /tabs/<id>` | Close a specific tab |
| `snapshot` | `GET /snapshot` | Get page content as text with element refs |
| `screenshot` | `POST /screenshot` | Capture PNG/JPEG screenshot (full page, element, or ref) |
| `navigate` | `POST /navigate` | Navigate to a URL in a specific tab |
| `act` | `POST /act` | Interact with elements (click, type, hover, drag, select, fill, press, scroll, wait, evaluate, close) |
| `console` | `GET /console` | Get browser console messages |
| `pdf` | `POST /pdf` | Save page as PDF |
| `upload` | `POST /hooks/file-chooser` | Upload files via file chooser dialog |
| `dialog` | `POST /hooks/dialog` | Accept/dismiss browser dialogs |

#### Act Command Types

The `act` action supports rich browser interactions:

```typescript
{ kind: "click",   ref: "e5", doubleClick?: bool, button?: "left"|"right", modifiers?: ["Shift"] }
{ kind: "type",    ref: "e3", text: "hello", submit?: bool, slowly?: bool }
{ kind: "press",   key: "Enter", delayMs?: 100 }
{ kind: "hover",   ref: "e7" }
{ kind: "drag",    startRef: "e2", endRef: "e9" }
{ kind: "select",  ref: "e4", values: ["option1", "option2"] }
{ kind: "fill",    fields: [{ ref: "e1", type: "text", value: "John" }, ...] }
{ kind: "resize",  width: 1280, height: 720 }
{ kind: "wait",    text?: "Loading complete", selector?: ".ready", timeoutMs?: 5000 }
{ kind: "evaluate", fn: "document.title", ref?: "e3" }
{ kind: "close" }
```

#### Target Routing: Sandbox, Host, and Node

The browser tool supports three execution targets:

```
 ┌─────────────────── Browser Target Routing ───────────────────────┐
 │                                                                   │
 │  target="host" (default for main sessions)                       │
 │  ├─ Browser runs on the Gateway host machine                     │
 │  ├─ Full access to host Chrome/Chromium                          │
 │  └─ Agent calls local Browser Control Server via HTTP            │
 │                                                                   │
 │  target="sandbox" (for sandboxed sessions)                       │
 │  ├─ Browser runs in a Docker container (Dockerfile.sandbox-      │
 │  │   browser)                                                     │
 │  ├─ Container runs: Xvfb + Chromium + x11vnc + noVNC             │
 │  ├─ Agent calls sandbox Bridge Server via bridgeUrl               │
 │  └─ Isolated from host filesystem and network                    │
 │                                                                   │
 │  target="node" (remote device)                                   │
 │  ├─ Browser runs on a connected device node (macOS/iOS)          │
 │  ├─ Agent tool calls are forwarded via Gateway node.invoke       │
 │  ├─ Node executes browser.proxy command locally                  │
 │  ├─ Result (including base64 screenshots) sent back via WS       │
 │  └─ Auto-resolved when a single browser-capable node is online   │
 └───────────────────────────────────────────────────────────────────┘
```

**Sandbox browser container** (`Dockerfile.sandbox-browser`):
- Debian bookworm-slim with Chromium, Xvfb (virtual display), x11vnc (VNC server), noVNC (web VNC), websockify
- Exposes ports: 9222 (CDP), 5900 (VNC), 6080 (noVNC web)
- The agent interacts via CDP; the user can watch via noVNC in a browser

#### Lifecycle & Gateway Integration

1. **Gateway startup** (`server-browser.ts`): lazily imports the browser control service and calls `startBrowserControlServiceFromConfig()`
2. **Browser Control Service** (`control-service.ts`): initializes state, resolves profiles from config, eagerly starts Chrome Extension Relay servers for any `extension`-driver profiles
3. **On first `browser` tool call**: the agent's tool code resolves the target (sandbox/host/node), then either calls the local HTTP control server or proxies through a device node
4. **Profile management**: profiles are stored in config (`browser.profiles`), each with a CDP port, color, and driver type. Profiles can be created/deleted at runtime via the API
5. **Shutdown**: Gateway calls `stopBrowserControlServer()` which stops all running Chrome instances across all profiles and closes Playwright connections

#### Configuration

```json5
{
  browser: {
    enabled: true,           // Enable browser control (default: true)
    headless: false,         // Run Chrome headless
    noSandbox: false,        // --no-sandbox flag (for Docker)
    evaluateEnabled: true,   // Allow JS evaluate in pages
    executablePath: null,    // Custom Chrome path (auto-detect by default)
    defaultProfile: "chrome", // Default profile name
    profiles: {
      openclaw: {
        cdpPort: 18791,      // CDP debugging port
        color: "#FF4500",    // Address bar color
      },
      chrome: {
        driver: "extension", // Use Chrome Extension Relay
        cdpUrl: "http://127.0.0.1:18792",
        color: "#00AA00",
      },
    },
    snapshotDefaults: {
      mode: "efficient",     // Reduce token cost
    },
  },
}
```

#### Key Design Insights

1. **Snapshot-first interaction**: The agent reads pages via structured text snapshots (not screenshots), enabling efficient token usage and precise element targeting via refs. Screenshots are used for visual verification, not primary navigation
2. **Profile isolation**: The `openclaw` profile uses a completely separate Chrome user data directory, preventing contamination of the user's cookies, history, and extensions
3. **Extension relay bridge**: The Chrome Extension Relay allows the agent to control tabs in the user's *actual* Chrome session — seeing their logged-in state, cookies, and context — while keeping the control path secure via a local relay server with token auth
4. **Three-target flexibility**: The same tool interface works whether the browser runs locally, in a Docker sandbox, or on a remote device node. The agent doesn't need to know the execution target
5. **60+ Playwright operations**: The `pw-tools-core.ts` module exposes a comprehensive set of browser automation primitives, from basic click/type to advanced operations like tracing, geolocation spoofing, media emulation, and storage manipulation

---

### 9. Canvas & A2UI

**Location**: `src/canvas-host/`

The Canvas is an **agent-driven visual workspace** that renders on macOS/iOS/Android.

- **A2UI (Agent-to-UI)**: A pattern where the agent pushes HTML/JS/CSS to a visual surface
- Operations: `canvas.push` (render content), `canvas.reset`, `canvas.eval` (run JS), `canvas.snapshot` (capture current state)
- The Canvas host server runs alongside the Gateway
- A2UI bundles are pre-built (`scripts/bundle-a2ui.sh`) and served to connected canvas surfaces

---

### 10. Automation (Cron & Webhooks)

**Location**: `src/cron/`, `src/gateway/server-cron.ts`

- **Cron jobs**: Scheduled tasks using the `croner` library. Jobs inject a message into an agent session at scheduled times
- **Webhooks**: HTTP endpoints that trigger agent sessions
- **Gmail Pub/Sub**: Google Gmail push notifications that trigger agent actions

---

## Data Flows (By Example)

The following examples trace real end-to-end data flows through OpenClaw, showing the exact function call chains and data transformations at each step.

---

### Example 1: WhatsApp DM → Agent → WhatsApp Reply

A user sends "What's the weather?" to the OpenClaw WhatsApp number.

```
 ┌──────────────────────────────────────────────────────────────────────────┐
 │ 1. WHATSAPP INBOUND                                                     │
 │                                                                          │
 │ Baileys WebSocket receives raw WhatsApp protobuf message                │
 │   ↓                                                                      │
 │ monitorWhatsApp() → startAccount()                                      │
 │   src/whatsapp/monitor.ts                                               │
 │   ↓                                                                      │
 │ Normalize to MsgContext:                                                │
 │   normalizeWhatsAppTarget("5551234567@s.whatsapp.net")                  │
 │   src/whatsapp/normalize.ts                                             │
 │   ↓                                                                      │
 │ Input data:                                                              │
 │   { Body: "What's the weather?",                                        │
 │     From: "5551234567@s.whatsapp.net",                                  │
 │     To: "myphone@s.whatsapp.net",                                       │
 │     Provider: "whatsapp",                                               │
 │     ChatType: "direct" }                                                │
 └──────────────────────────┬───────────────────────────────────────────────┘
                            │
 ┌──────────────────────────▼───────────────────────────────────────────────┐
 │ 2. SECURITY CHECK                                                        │
 │                                                                          │
 │ Check DM pairing policy (dmPolicy="pairing"):                           │
 │   Is sender in channels.whatsapp.allowFrom?                             │
 │   ├─ YES → continue to dispatch                                         │
 │   └─ NO  → send pairing code, ignore message                           │
 │   src/channels/plugins/group-mentions.ts                                │
 │   src/channels/allowlists/                                              │
 └──────────────────────────┬───────────────────────────────────────────────┘
                            │
 ┌──────────────────────────▼───────────────────────────────────────────────┐
 │ 3. SESSION ROUTING                                                       │
 │                                                                          │
 │ Derive session key:                                                      │
 │   resolveSessionKeyForRun(ctx) → "agent:main:whatsapp:5551234567"       │
 │   src/gateway/server-session-key.ts                                     │
 │   ↓                                                                      │
 │ Resolve agent ID from key → "main"                                      │
 │   resolveAgentIdFromSessionKey()                                        │
 │   src/routing/session-key.ts                                            │
 │   ↓                                                                      │
 │ Load/create session store entry                                          │
 │   loadSessionStore("agent:main:whatsapp:5551234567")                    │
 │   src/config/sessions.ts                                                │
 │   ↓                                                                      │
 │ Check lane concurrency (prevent parallel runs on same session)          │
 │   resolveEmbeddedSessionLane()                                          │
 │   src/agents/pi-embedded-runner/lanes.ts                                │
 └──────────────────────────┬───────────────────────────────────────────────┘
                            │
 ┌──────────────────────────▼───────────────────────────────────────────────┐
 │ 4. MESSAGE DISPATCH                                                      │
 │                                                                          │
 │ dispatchInboundMessage(ctx, cfg, dispatcher)                            │
 │   src/auto-reply/dispatch.ts                                            │
 │   ↓                                                                      │
 │ dispatchReplyFromConfig({ctx, cfg})                                     │
 │   ├─ Check for slash commands (/status, /new, /think, etc.)             │
 │   ├─ Check auto-reply rules                                            │
 │   └─ Route to agent execution                                          │
 │   src/auto-reply/reply/dispatch-from-config.ts                          │
 └──────────────────────────┬───────────────────────────────────────────────┘
                            │
 ┌──────────────────────────▼───────────────────────────────────────────────┐
 │ 5. AGENT EXECUTION                                                       │
 │                                                                          │
 │ runEmbeddedPiAgent({                                                    │
 │   sessionKey: "agent:main:whatsapp:5551234567",                         │
 │   prompt: "[2026-02-13 09:15]\n+5551234567: What's the weather?",       │
 │   agentId: "main",                                                      │
 │   model: "anthropic/claude-opus-4-6"                                    │
 │ })                                                                       │
 │   src/agents/pi-embedded-runner/run.ts                                  │
 │   ↓                                                                      │
 │ Build system prompt:                                                     │
 │   ├─ Load AGENTS.md from workspace                                      │
 │   ├─ Inject active skills (weather, coding-agent, etc.)                 │
 │   ├─ Add channel context ("You are on WhatsApp, replying to +555...")   │
 │   └─ Add tool descriptions (40+ tools)                                  │
 │   src/agents/system-prompt.ts                                           │
 │   ↓                                                                      │
 │ Select model + auth profile:                                             │
 │   ├─ Try primary: anthropic/claude-opus-4-6 via OAuth session           │
 │   ├─ On 429/timeout: fallback to API key                                │
 │   └─ On total failure: try next profile                                 │
 │   src/agents/model-selection.ts, src/agents/model-fallback.ts           │
 │   ↓                                                                      │
 │ Call LLM provider → Anthropic Claude API                                │
 │   ↓                                                                      │
 │ Agent might decide to use tools:                                         │
 │   Tool call: web_search("weather in user's city")                       │
 │   Tool result: { temperature: 72, conditions: "sunny" }                 │
 │   ↓                                                                      │
 │ Agent streams response text blocks:                                      │
 │   "It's currently 72°F and sunny! ☀️"                                   │
 │   ↓                                                                      │
 │ Context window guard:                                                    │
 │   Check if session approaching token limit                              │
 │   If yes → trigger compaction (summarize history)                       │
 │   src/agents/context-window-guard.ts                                    │
 └──────────────────────────┬───────────────────────────────────────────────┘
                            │
 ┌──────────────────────────▼───────────────────────────────────────────────┐
 │ 6. RESPONSE DELIVERY                                                     │
 │                                                                          │
 │ Agent emits text via onAgentEvent() handler                             │
 │   src/infra/agent-events.ts → src/gateway/server-chat.ts               │
 │   ↓                                                                      │
 │ Block streaming pipeline:                                                │
 │   ├─ Buffer text into paragraph-aware blocks                            │
 │   ├─ WhatsApp chunk limit: 4000 chars                                   │
 │   ├─ If response > 4000 chars → split into multiple messages            │
 │   └─ Apply markdown → WhatsApp formatting                              │
 │   ↓                                                                      │
 │ Route reply to WhatsApp outbound:                                        │
 │   routeReply() → ChannelOutboundAdapter.sendPayload()                   │
 │   src/auto-reply/reply/route-reply.ts                                   │
 │   ↓                                                                      │
 │ Baileys sends WhatsApp protobuf message back to user:                   │
 │   "It's currently 72°F and sunny! ☀️"                                   │
 └──────────────────────────────────────────────────────────────────────────┘
```

---

### Example 2: WebChat Message → Agent with Browser Tool → WebChat Response

A user types "Go to hackernews and find the top story" in the WebChat UI.

```
 ┌──────────────────────────────────────────────────────────────────────────┐
 │ 1. WEBCHAT INBOUND (WebSocket RPC)                                       │
 │                                                                          │
 │ Browser Control UI sends WS message:                                    │
 │   { method: "chat.send",                                                │
 │     params: { sessionKey: "default",                                    │
 │               message: "Go to hackernews and find the top story",       │
 │               idempotencyKey: "a1b2c3d4" } }                            │
 │   ↓                                                                      │
 │ Gateway WS handler:                                                      │
 │   "chat.send" method → server-methods/chat.ts                          │
 │   ↓                                                                      │
 │ Immediate ack to client:                                                 │
 │   { ok: true, result: { runId: "a1b2c3d4", status: "started" } }       │
 │                                                                          │
 │ (Unlike channel messages, WebChat uses the WS protocol directly —       │
 │  no channel plugin normalization needed)                                │
 └──────────────────────────┬───────────────────────────────────────────────┘
                            │
 ┌──────────────────────────▼───────────────────────────────────────────────┐
 │ 2. SESSION + AGENT DISPATCH                                              │
 │                                                                          │
 │ Build MsgContext:                                                        │
 │   { Body: "Go to hackernews...",                                        │
 │     Provider: "internal",                                               │
 │     ChatType: "direct",                                                 │
 │     SessionKey: "agent:main:main",                                      │
 │     CommandAuthorized: true }                                           │
 │   ↓                                                                      │
 │ Register for tool events (WebChat supports streaming):                  │
 │   registerToolEventRecipient(runId, connectionId)                       │
 │   ↓                                                                      │
 │ dispatchInboundMessage() → runEmbeddedPiAgent()                         │
 │   (same agent execution path as channel messages)                       │
 └──────────────────────────┬───────────────────────────────────────────────┘
                            │
 ┌──────────────────────────▼───────────────────────────────────────────────┐
 │ 3. AGENT CALLS BROWSER TOOL (Multi-Step)                                 │
 │                                                                          │
 │ STEP A: Agent decides to navigate                                       │
 │   Tool call: browser({ action: "navigate",                              │
 │                         targetUrl: "https://news.ycombinator.com" })     │
 │   ↓                                                                      │
 │   browser-tool.ts → resolveBrowserBaseUrl(target="host")                │
 │   ↓                                                                      │
 │   browserNavigate("http://127.0.0.1:18791",                             │
 │                    { url: "https://news.ycombinator.com" })              │
 │   ↓                                                                      │
 │   HTTP POST /navigate → Browser Control Server                          │
 │   ↓                                                                      │
 │   navigateViaPlaywright({ cdpUrl, url }) → Playwright page.goto()       │
 │   ↓                                                                      │
 │   Result: { ok: true, targetId: "ABC123", url: "https://news..." }      │
 │                                                                          │
 │ STEP B: Agent takes a snapshot to "see" the page                        │
 │   Tool call: browser({ action: "snapshot", snapshotFormat: "ai" })      │
 │   ↓                                                                      │
 │   browserSnapshot("http://127.0.0.1:18791",                             │
 │                    { format: "ai", maxChars: 80000 })                   │
 │   ↓                                                                      │
 │   HTTP GET /snapshot?format=ai → Browser Control Server                 │
 │   ↓                                                                      │
 │   snapshotAiViaPlaywright({ cdpUrl, targetId })                         │
 │     → page._snapshotForAI()           (Playwright internal API)         │
 │     → buildRoleSnapshotFromAiSnapshot() (generate e1,e2,... refs)       │
 │     → storeRoleRefsForTarget()        (cache refs for next call)        │
 │   ↓                                                                      │
 │   Result (truncated):                                                    │
 │     "- navigation \"Hacker News\"\n                                     │
 │      - list\n                                                            │
 │        - listitem\n                                                      │
 │          - [e1] link \"Show HN: I built a...\"\n                        │
 │          - text \"142 points by user123\"\n                             │
 │        - listitem\n                                                      │
 │          - [e2] link \"Why Rust is the future...\"\n                    │
 │          ..."                                                            │
 │                                                                          │
 │ STEP C: Agent extracts information from snapshot text                   │
 │   (No additional tool call — agent reads the snapshot and responds)     │
 └──────────────────────────┬───────────────────────────────────────────────┘
                            │
 ┌──────────────────────────▼───────────────────────────────────────────────┐
 │ 4. STREAMING RESPONSE TO WEBCHAT                                         │
 │                                                                          │
 │ Agent streams text → createAgentEventHandler()                          │
 │   ↓                                                                      │
 │ Emit chat deltas over WebSocket:                                        │
 │   broadcast("chat", { runId: "a1b2c3d4",                               │
 │     state: "delta", seq: 1,                                             │
 │     message: { role: "assistant",                                       │
 │       content: [{ type: "text",                                         │
 │         text: "The top story on Hacker News right now is..." }] } })    │
 │   ↓                                                                      │
 │ Tool events also streamed (if verbose enabled):                         │
 │   broadcast("agent", { type: "tool_call",                               │
 │     tool: "browser", action: "navigate", ... })                         │
 │   ↓                                                                      │
 │ Final event:                                                             │
 │   broadcast("chat", { runId: "a1b2c3d4",                               │
 │     state: "final", seq: 3,                                             │
 │     message: { role: "assistant",                                       │
 │       content: [{ type: "text",                                         │
 │         text: "The top story on HN is 'Show HN: I built a...'" }] } }) │
 │                                                                          │
 │ WebChat UI renders the streaming response in real-time                  │
 └──────────────────────────────────────────────────────────────────────────┘
```

---

### Example 3: Cron Job → Agent → Telegram Delivery

A cron job fires at 9 AM daily, running "Check my calendar and summarize today's meetings" and delivering to Telegram.

```
 ┌──────────────────────────────────────────────────────────────────────────┐
 │ 1. CRON TRIGGER                                                          │
 │                                                                          │
 │ CronService timer tick (croner library)                                 │
 │   src/cron/service.ts → service/timer.ts                                │
 │   ↓                                                                      │
 │ Job matches schedule: "0 9 * * *"                                       │
 │   { id: "daily-cal", name: "Morning Calendar",                          │
 │     schedule: "0 9 * * *",                                              │
 │     payload: { kind: "agentTurn",                                       │
 │       message: "Check my calendar and summarize today's meetings",      │
 │       delivery: { channel: "telegram", to: "12345678" } } }            │
 │   ↓                                                                      │
 │ Gateway broadcasts cron event:                                           │
 │   broadcast("cron", { action: "firing", jobId: "daily-cal" })           │
 │   src/gateway/server-cron.ts                                            │
 └──────────────────────────┬───────────────────────────────────────────────┘
                            │
 ┌──────────────────────────▼───────────────────────────────────────────────┐
 │ 2. ISOLATED AGENT EXECUTION                                              │
 │                                                                          │
 │ runCronIsolatedAgentTurn({                                              │
 │   cfg, job, sessionKey: "agent:main:cron:daily-cal", agentId: "main"   │
 │ })                                                                       │
 │   src/cron/isolated-agent/run.ts                                        │
 │   ↓                                                                      │
 │ Build command body with timestamp:                                       │
 │   "[cron:daily-cal Morning Calendar] Check my calendar and              │
 │    summarize today's meetings                                           │
 │    [Time: 2026-02-13 09:00 UTC]"                                        │
 │   ↓                                                                      │
 │ runWithModelFallback({                                                   │
 │   provider, model, agentDir,                                            │
 │   run: (p, m) => runEmbeddedPiAgent({prompt: commandBody, ...})         │
 │ })                                                                       │
 │   ↓                                                                      │
 │ Agent executes (may use tools like web_fetch for calendar API)           │
 │   ↓                                                                      │
 │ Agent produces response:                                                 │
 │   "You have 3 meetings today:                                           │
 │    9:30 AM - Standup (15 min)                                           │
 │    11:00 AM - Design Review (1 hr)                                      │
 │    2:00 PM - 1:1 with Manager (30 min)"                                 │
 └──────────────────────────┬───────────────────────────────────────────────┘
                            │
 ┌──────────────────────────▼───────────────────────────────────────────────┐
 │ 3. DELIVERY TO TELEGRAM                                                  │
 │                                                                          │
 │ Resolve delivery target:                                                 │
 │   resolveDeliveryTarget(cfg, agentId, {channel: "telegram", to: "..."}) │
 │   src/cron/isolated-agent/delivery-target.ts                            │
 │   ↓                                                                      │
 │ deliverOutboundPayloads({                                               │
 │   cfg, channel: "telegram", to: "12345678",                             │
 │   payloads: [{ text: "You have 3 meetings today: ..." }]               │
 │ })                                                                       │
 │   ↓                                                                      │
 │ Telegram outbound adapter:                                               │
 │   grammY bot.api.sendMessage(12345678, text)                            │
 │   ↓                                                                      │
 │ Log execution:                                                           │
 │   appendCronRunLog({ jobId: "daily-cal", status: "ok",                  │
 │     summary: "3 meetings", durationMs: 4200 })                          │
 └──────────────────────────────────────────────────────────────────────────┘
```

---

### Example 4: iOS Node Camera Snap

The agent needs to take a photo using the user's iPhone camera.

```
 ┌──────────────────────────────────────────────────────────────────────────┐
 │ 1. AGENT TOOL CALL                                                       │
 │                                                                          │
 │ During conversation, agent decides to invoke camera:                    │
 │   Tool call: nodes({ action: "invoke",                                  │
 │     nodeId: "ios-1", command: "camera.snap", params: {} })              │
 │   src/agents/tools/nodes-utils.ts                                       │
 │   ↓                                                                      │
 │ Resolve target node:                                                     │
 │   listNodes({}) → find "ios-1" in connected nodes                       │
 │   resolveNodeIdFromList(nodes, "ios-1")                                 │
 └──────────────────────────┬───────────────────────────────────────────────┘
                            │
 ┌──────────────────────────▼───────────────────────────────────────────────┐
 │ 2. GATEWAY → NODE (WebSocket)                                            │
 │                                                                          │
 │ callGatewayTool("node.invoke", {                                        │
 │   nodeId: "ios-1", command: "camera.snap",                              │
 │   params: {}, idempotencyKey: "uuid-1" })                               │
 │   ↓                                                                      │
 │ Gateway "node.invoke" handler:                                           │
 │   src/gateway/server-methods/nodes.ts                                   │
 │   ↓                                                                      │
 │ NodeRegistry.invoke():                                                   │
 │   requestId = randomUUID()                                              │
 │   pendingInvokes.set(requestId, { resolve, reject, timer })             │
 │   ↓                                                                      │
 │ Send WS event to iOS node:                                              │
 │   node.socket.send({                                                    │
 │     type: "event",                                                      │
 │     event: "node.invoke.request",                                       │
 │     payload: { id: "req-uuid", nodeId: "ios-1",                         │
 │                command: "camera.snap", paramsJSON: null,                 │
 │                timeoutMs: 30000 } })                                    │
 │   src/gateway/node-registry.ts                                          │
 └──────────────────────────┬───────────────────────────────────────────────┘
                            │  (WebSocket to iOS device over LAN/Tailscale)
 ┌──────────────────────────▼───────────────────────────────────────────────┐
 │ 3. iOS NODE EXECUTION                                                    │
 │                                                                          │
 │ OpenClaw iOS app receives node.invoke.request                           │
 │   apps/ios/Sources/                                                     │
 │   ↓                                                                      │
 │ Execute camera.snap command:                                             │
 │   ├─ Check TCC camera permission (iOS permission system)                │
 │   ├─ Capture photo using AVCaptureSession                               │
 │   ├─ Encode as JPEG, base64                                             │
 │   └─ Build result payload                                               │
 │   ↓                                                                      │
 │ Send result back to Gateway:                                             │
 │   socket.send({                                                         │
 │     type: "event",                                                      │
 │     event: "node.invoke.result",                                        │
 │     payload: { id: "req-uuid", nodeId: "ios-1", ok: true,              │
 │       payloadJSON: "{\"imagePath\":\"/tmp/snap.jpg\",                   │
 │                      \"width\":4032,\"height\":3024,                    │
 │                      \"base64\":\"...huge base64...\"}" } })            │
 └──────────────────────────┬───────────────────────────────────────────────┘
                            │
 ┌──────────────────────────▼───────────────────────────────────────────────┐
 │ 4. RESULT FLOWS BACK TO AGENT                                            │
 │                                                                          │
 │ Gateway receives "node.invoke.result" event:                            │
 │   handleInvokeResult({ id: "req-uuid", ok: true, payload })            │
 │   ↓                                                                      │
 │ Resolve pending promise:                                                 │
 │   pending = pendingInvokes.get("req-uuid")                              │
 │   pending.resolve({ ok: true, payload: { imagePath, base64, ... } })   │
 │   pendingInvokes.delete("req-uuid")                                     │
 │   ↓                                                                      │
 │ Persist media file locally:                                              │
 │   saveMediaBuffer(buffer, "image/jpeg", "node") → "/media/node/abc.jpg"│
 │   ↓                                                                      │
 │ Tool result returned to agent runtime:                                   │
 │   { ok: true, imagePath: "/media/node/abc.jpg",                         │
 │     width: 4032, height: 3024 }                                         │
 │   ↓                                                                      │
 │ Agent can now analyze the image and respond to the user                 │
 └──────────────────────────────────────────────────────────────────────────┘
```

---

### Example 5: Discord Group with @mention Activation

A user posts "@OpenClaw summarize this thread" in a Discord channel.

```
 ┌──────────────────────────────────────────────────────────────────────────┐
 │ 1. DISCORD INBOUND                                                       │
 │                                                                          │
 │ discord.js client receives MESSAGE_CREATE event                         │
 │   src/discord/monitor.ts                                                │
 │   ↓                                                                      │
 │ Normalize to MsgContext:                                                │
 │   { Body: "@OpenClaw summarize this thread",                            │
 │     From: "user123",                                                    │
 │     To: "channel456",                                                   │
 │     Provider: "discord",                                                │
 │     ChatType: "group",     ← group, not DM!                            │
 │     SenderId: "user123",                                                │
 │     SenderName: "Alice" }                                               │
 └──────────────────────────┬───────────────────────────────────────────────┘
                            │
 ┌──────────────────────────▼───────────────────────────────────────────────┐
 │ 2. GROUP ACTIVATION CHECK                                                │
 │                                                                          │
 │ resolveDiscordGroupRequireMention(cfg, groupId) → true (default)        │
 │   src/channels/plugins/group-mentions.ts                                │
 │   ↓                                                                      │
 │ Mention gating:                                                          │
 │   Does message contain bot @mention? → YES (<@BOT_ID> pattern)         │
 │   Strip mention from body:                                               │
 │     stripPatterns: ["<@!?\\d+>"]                                        │
 │     Body becomes: "summarize this thread"                               │
 │   src/channels/mention-gating.ts                                        │
 │   ↓                                                                      │
 │ Resolve group tool policy:                                               │
 │   resolveDiscordGroupToolPolicy(cfg, groupId, senderId)                 │
 │   → May restrict certain tools in group context                         │
 └──────────────────────────┬───────────────────────────────────────────────┘
                            │
 ┌──────────────────────────▼───────────────────────────────────────────────┐
 │ 3. SESSION ISOLATION                                                     │
 │                                                                          │
 │ Session key for this group:                                              │
 │   "agent:main:discord:channel:channel456"                               │
 │   (separate from user's DM session — full isolation)                    │
 │   ↓                                                                      │
 │ If sandbox.mode="non-main":                                              │
 │   This group session runs in Docker sandbox                             │
 │   Tools restricted to: bash, read, write, edit, sessions_*              │
 │   Tools blocked: browser, canvas, nodes, cron, discord, gateway         │
 │   src/agents/sandbox/                                                   │
 └──────────────────────────┬───────────────────────────────────────────────┘
                            │
 ┌──────────────────────────▼───────────────────────────────────────────────┐
 │ 4. AGENT + RESPONSE                                                      │
 │                                                                          │
 │ runEmbeddedPiAgent({                                                    │
 │   prompt: "[timestamp]\nAlice: summarize this thread",                  │
 │   sessionKey: "agent:main:discord:channel:channel456" })                │
 │   ↓                                                                      │
 │ Agent generates response (may be multi-paragraph)                       │
 │   ↓                                                                      │
 │ Discord streaming pipeline:                                              │
 │   ├─ Block streaming with coalescing:                                   │
 │   │   minChars: 1500, idleMs: 1000                                      │
 │   │   (wait for 1500 chars or 1s of silence before sending)             │
 │   ├─ Chunk limit: 2000 chars (Discord message limit)                    │
 │   └─ Send via discord.js:                                               │
 │       channel.send("Here's a summary of the thread: ...")               │
 │                                                                          │
 │ If response > 2000 chars: split into multiple Discord messages          │
 │ Threading: reply depends on discord.replyToMode config                  │
 └──────────────────────────────────────────────────────────────────────────┘
```

---

### Core Data Structures Across All Flows

**MsgContext** (unified message format across all entry points):
```typescript
{
  Body: string;                    // Raw message text
  BodyForAgent: string;            // Timestamped text for agent
  SessionKey: string;              // "agent:<agentId>:<channel>:<peer>"
  Provider: string;                // "whatsapp" | "telegram" | "internal" | ...
  ChatType: "direct" | "group";   // Message type
  From: string;                    // Sender identifier
  To: string;                      // Recipient/channel identifier
  SenderId?: string;               // Group sender ID
  SenderName?: string;             // Group sender display name
  CommandAuthorized: boolean;      // Can run slash commands?
}
```

**SessionKey** (routing identity):
```
agent:main:main                          → Owner's primary DM
agent:main:whatsapp:5551234567           → WhatsApp DM with +5551234567
agent:main:discord:channel:channel456    → Discord channel
agent:main:telegram:group:12345          → Telegram group
agent:main:cron:daily-cal                → Cron job session
agent:main:subagent:uuid-123             → Spawned sub-agent
agent:work:slack:dm:U12345               → "work" agent, Slack DM
```

---

## Configuration Architecture

```
 ~/.openclaw/
 ├── openclaw.json          # Main config (JSON5, Zod-validated)
 ├── credentials/           # Channel credentials (WhatsApp auth, etc.)
 ├── sessions/              # Session history files
 ├── workspace/             # Agent workspace
 │   ├── AGENTS.md          # Agent personality/instructions
 │   ├── SOUL.md            # Identity prompt
 │   ├── TOOLS.md           # Tool instructions
 │   └── skills/            # Workspace skills
 │       └── <skill>/
 │           └── SKILL.md
 ├── skills/                # Managed skills (from ClawHub)
 └── logs/                  # Runtime logs
```

**Config schema** (`src/config/types.ts`) is split into 30+ focused type modules covering agents, channels, models, gateway, sandbox, tools, skills, cron, TTS, memory, and more.

---

## Security Model

```
 ┌────────────── Security Layers ───────────────────────────────────┐
 │                                                                   │
 │  1. Gateway Auth                                                 │
 │     ├─ Token auth (OPENCLAW_GATEWAY_TOKEN)                       │
 │     ├─ Password auth (for Tailscale Funnel)                      │
 │     ├─ Tailscale identity headers                                │
 │     └─ Origin checking for WebSocket                             │
 │                                                                   │
 │  2. Channel Security                                             │
 │     ├─ DM pairing (default: unknown senders get pairing code)    │
 │     ├─ Per-channel allowFrom lists                               │
 │     ├─ Group activation gating (@mention required)               │
 │     └─ Per-group tool policies                                   │
 │                                                                   │
 │  3. Execution Sandbox                                            │
 │     ├─ Main session: full host access (owner only)               │
 │     ├─ Non-main sessions: Docker sandbox (configurable)          │
 │     ├─ Tool allowlists/denylists per session type                │
 │     └─ Exec approval manager (require user approval for cmds)    │
 │                                                                   │
 │  4. Node Security                                                │
 │     ├─ Device pairing with approval flow                         │
 │     ├─ Token-based node authentication                           │
 │     ├─ TCC permission checking (macOS)                           │
 │     └─ Per-command policy (node-command-policy.ts)                │
 └───────────────────────────────────────────────────────────────────┘
```

---

## Key Technologies & Dependencies

### Core Runtime
| Technology | Purpose | Package |
|-----------|---------|---------|
| **Node.js >=22** | Runtime (ESM modules) | - |
| **TypeScript 5.9** | Language | `typescript` |
| **Express 5** | HTTP server | `express` |
| **ws** | WebSocket server | `ws` |
| **Zod 4** | Schema validation (config) | `zod` |
| **TypeBox** | JSON Schema (protocol) | `@sinclair/typebox` |
| **AJV** | JSON Schema validation (protocol) | `ajv` |

### AI & Agent
| Technology | Purpose | Package |
|-----------|---------|---------|
| **Pi Agent SDK** | Agent runtime core | `@mariozechner/pi-agent-core`, `pi-ai`, `pi-coding-agent` |
| **Anthropic Claude** | Primary LLM | via Pi SDK + OAuth/API |
| **OpenAI GPT/Codex** | Alternative LLM | via Pi SDK + OAuth/API |
| **Google Gemini** | Alternative LLM | via Pi SDK |
| **AWS Bedrock** | Alternative LLM | `@aws-sdk/client-bedrock` |
| **Ollama** | Local LLM | `ollama` |
| **sqlite-vec** | Vector search (memory) | `sqlite-vec` |

### Messaging Channels
| Technology | Purpose | Package |
|-----------|---------|---------|
| **Baileys** | WhatsApp Web protocol | `@whiskeysockets/baileys` |
| **grammY** | Telegram Bot API | `grammy` |
| **discord.js** | Discord Bot API | via `discord-api-types` |
| **Bolt** | Slack Socket Mode | `@slack/bolt` |
| **signal-cli** | Signal messaging | External binary |
| **Agent Client Protocol** | ACP integration | `@agentclientprotocol/sdk` |

### Browser & Media
| Technology | Purpose | Package |
|-----------|---------|---------|
| **Playwright** | Browser automation | `playwright-core` |
| **Sharp** | Image processing | `sharp` |
| **pdfjs-dist** | PDF parsing | `pdfjs-dist` |
| **ElevenLabs / Edge TTS** | Text-to-speech | `node-edge-tts` |
| **Readability** | Web content extraction | `@mozilla/readability` |

### Build & Dev
| Technology | Purpose | Package |
|-----------|---------|---------|
| **pnpm** | Package manager (workspaces) | `pnpm@10` |
| **tsdown** | TypeScript bundler | `tsdown` |
| **Vitest** | Testing framework | `vitest` |
| **oxfmt** | Code formatter | `oxfmt` |
| **oxlint** | Linter (type-aware) | `oxlint` |
| **Lit** | Web components (Control UI) | `lit` |
| **Vite** | UI dev server/bundler | `vite` |

### Deployment
| Technology | Purpose |
|-----------|---------|
| **Docker** | Container deployment + sandbox |
| **systemd / launchd** | Daemon management |
| **Tailscale** | Serve/Funnel for remote access |
| **Fly.io / Render** | Cloud deployment targets |
| **Bonjour/mDNS** | Device node discovery (`@homebridge/ciao`) |

---

## Monorepo Structure

```
openclaw/
├── src/                        # Main TypeScript source
│   ├── entry.ts                # CLI entry point (respawn + profile)
│   ├── index.ts                # Library entry + Commander program
│   ├── gateway/                # Gateway server (control plane)
│   │   ├── server.impl.ts      # startGatewayServer()
│   │   ├── protocol/           # JSON-RPC protocol schemas
│   │   ├── server-methods/     # WS method handlers (by domain)
│   │   └── server/             # HTTP, TLS, health state
│   ├── agents/                 # Pi Agent runtime
│   │   ├── pi-embedded-runner/ # Agent run loop
│   │   ├── pi-embedded-subscribe/ # Stream subscription handlers
│   │   ├── pi-tools.ts         # Tool registration
│   │   ├── tools/              # 40+ tool implementations
│   │   ├── skills/             # Skill loading/resolution
│   │   ├── system-prompt.ts    # System prompt builder
│   │   ├── model-selection.ts  # Model picking
│   │   ├── model-fallback.ts   # Failover logic
│   │   ├── auth-profiles/      # OAuth + API key rotation
│   │   └── sandbox/            # Docker sandbox for sessions
│   ├── channels/               # Channel abstraction layer
│   │   ├── registry.ts         # Core channel IDs + metadata
│   │   ├── dock.ts             # Lightweight channel behaviors
│   │   └── plugins/            # Channel plugin interface
│   ├── whatsapp/               # WhatsApp via Baileys
│   ├── telegram/               # Telegram via grammY
│   ├── discord/                # Discord via discord.js
│   ├── slack/                  # Slack via Bolt
│   ├── signal/                 # Signal via signal-cli
│   ├── imessage/               # iMessage (legacy)
│   ├── config/                 # Configuration (JSON5 + Zod)
│   │   ├── types.ts            # 30+ type modules
│   │   ├── zod-schema.ts       # Full config schema
│   │   └── io.ts               # Read/write/validate
│   ├── sessions/               # Session store + key utils
│   ├── routing/                # Message routing + session keys
│   ├── plugins/                # Plugin registry + runtime
│   ├── browser/                # Playwright browser control
│   ├── canvas-host/            # A2UI canvas server
│   ├── node-host/              # Device node runner
│   ├── media/                  # Media pipeline (images, audio, video)
│   ├── media-understanding/    # Transcription + OCR hooks
│   ├── memory/                 # Vector memory (sqlite-vec)
│   ├── cron/                   # Cron job runtime
│   ├── tts/                    # Text-to-speech
│   ├── tui/                    # Terminal UI
│   ├── cli/                    # CLI commands + program
│   ├── commands/               # CLI command implementations
│   ├── wizard/                 # Onboarding wizard
│   ├── infra/                  # Infrastructure utilities
│   ├── security/               # Security policies
│   ├── logging/                # Structured logging
│   ├── providers/              # LLM provider helpers
│   ├── hooks/                  # Lifecycle hooks
│   ├── web/                    # WebChat channel
│   └── shared/                 # Shared types + utilities
├── ui/                         # Control UI (Lit + Vite)
├── apps/                       # Companion apps
│   ├── macos/                  # macOS menu bar app (Swift)
│   ├── ios/                    # iOS app (Swift)
│   ├── android/                # Android app (Kotlin)
│   └── shared/                 # OpenClawKit (shared Swift)
├── extensions/                 # Extension channel plugins (35+)
│   ├── bluebubbles/            # iMessage via BlueBubbles
│   ├── matrix/                 # Matrix protocol
│   ├── msteams/                # Microsoft Teams
│   ├── discord/                # Discord extensions
│   ├── slack/                  # Slack extensions
│   ├── telegram/               # Telegram extensions
│   └── ...                     # IRC, Zalo, LINE, Feishu, etc.
├── packages/                   # Internal packages
│   ├── clawdbot/               # Bot personality package
│   └── moltbot/                # Alternative bot package
├── skills/                     # 50+ bundled skills
├── scripts/                    # Build + codegen scripts
├── docs/                       # Documentation source (Mintlify)
├── test/                       # Test fixtures + helpers
├── Dockerfile                  # Production container
├── Dockerfile.sandbox          # Sandbox container
├── Dockerfile.sandbox-browser  # Browser sandbox container
├── docker-compose.yml          # Docker Compose setup
├── package.json                # Root workspace config
├── pnpm-workspace.yaml         # pnpm workspace definition
├── tsdown.config.ts            # Build configuration
└── vitest.*.config.ts          # Test configurations (unit/e2e/live/gateway)
```

---

## Architectural Insights

### 1. Single-Process, Event-Driven Gateway
The Gateway runs everything in a single Node.js process. Channels, the agent runtime, cron, browser, and canvas are all orchestrated from one event loop. This simplifies deployment (one process to run) and communication (in-process function calls vs IPC), at the cost of horizontal scaling. For a personal assistant, this is the right trade-off.

### 2. Channel Dock Pattern (Dependency Inversion)
The `ChannelDock` abstraction is a clever way to avoid heavy imports. Shared code (routing, session resolution, tool context building) only imports the lightweight dock, not the full channel plugin with its SDK dependencies. This prevents Baileys, grammY, discord.js, etc. from being eagerly loaded when only metadata is needed.

### 3. Plugin-First Extensibility
Everything is a plugin: channels, tools, hooks, providers, services, CLI commands, diagnostics. The global `PluginRegistry` (stored via `Symbol.for` on `globalThis` to survive module reloads) is the central registration point. Extensions are separate pnpm workspace packages that register at boot.

### 4. Agent Identity Separation
The system separates the **agent** (AI personality, workspace, model config) from the **session** (conversation state). Multiple agents can exist, each serving different channels/purposes. Sessions are keyed as `agent:<agentId>:<rest>`, enabling per-agent isolation with shared infrastructure.

### 5. Defense-in-Depth Security
Security is layered: Gateway auth (tokens/passwords) → Channel allowlists (DM pairing) → Group activation gating → Tool policies (per-session allowlists) → Execution sandbox (Docker). The default posture is restrictive: unknown senders are pairing-gated, groups require @mention, non-main sessions can be sandboxed.

### 6. Streaming-First Response Model
Responses flow through a sophisticated streaming pipeline: the agent produces token-by-token output, which is buffered into paragraph-aware blocks, chunked per channel limits (2K for Discord, 4K for WhatsApp/Telegram), and delivered with coalescing delays for platforms that support editing (Discord, Slack). This makes responses feel responsive across very different channel constraints.

### 7. Model Abstraction with Auth Profile Rotation
The model layer abstracts across providers (Anthropic, OpenAI, Google, Bedrock, Ollama) with a rotation system for auth profiles. If an OAuth session hits rate limits, the system falls back to an API key, or to a different model entirely. This makes the assistant resilient to provider outages.

### 8. Skills as Prompt Engineering
Skills are not code plugins — they are **prompt files** (`SKILL.md`). This makes them trivially portable, versionable, and composable. The agent's system prompt is dynamically assembled from workspace files + active skills, letting users customize behavior without writing code.

### 9. Protocol-First Cross-Platform
The Gateway protocol schema (`src/gateway/protocol/schema/`) is defined with TypeBox and used to auto-generate Swift models for the macOS/iOS apps (`scripts/protocol-gen-swift.ts`). This ensures type safety across the TypeScript server and Swift clients without manual synchronization.

### 10. Workspace-Rooted Agent Context
Each agent has a workspace directory containing `AGENTS.md`, `SOUL.md`, `TOOLS.md`, and skills. These files are injected into the system prompt at run time, giving the agent its personality, instructions, and capabilities. This "file-as-config" approach makes the agent highly customizable through simple text editing.
