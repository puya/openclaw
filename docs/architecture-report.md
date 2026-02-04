# Moltbot Architecture Report

A comprehensive technical overview of the Moltbot repository: structure, core components, data flow, and how they fit together.

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Repository Metrics](#repository-metrics)
3. [High-Level Architecture](#high-level-architecture)
4. [Repository Structure](#repository-structure)
5. [Core Components](#core-components)
   - [Gateway](#1-gateway-srcgateway)
   - [Agents (AI Runtime)](#2-agents-ai-runtime-srcagents)
   - [Auto-Reply Pipeline](#3-auto-reply-pipeline-srcauto-reply)
   - [Channels](#4-channels)
   - [CLI](#5-cli-srccli--srccommands)
   - [Configuration](#6-configuration-srcconfig)
6. [Agent Internals (Deep Dive)](#agent-internals-deep-dive)
   - [Execution Engine](#1-execution-engine-pi-embedded-runner)
   - [Streaming & Events](#2-streaming--event-subscription)
   - [Bash / Code Execution](#3-bash--code-execution)
   - [System Prompt Assembly](#4-system-prompt-assembly)
   - [Model Layer](#5-model-layer)
   - [Auth & Credentials](#6-auth--credentials)
   - [Tools](#7-tools)
   - [Skills](#8-skills-system)
   - [Memory](#9-memory-system)
   - [Sub-Agents](#10-sub-agents-multi-agent)
   - [Sandbox](#11-sandbox)
   - [Compaction](#12-compaction)
7. [AI Model Integration](#ai-model-integration)
8. [Browser Automation](#browser-automation)
9. [Email / Webhooks](#email--webhooks)
10. [Extensions & Plugins](#extensions--plugins)
11. [Native Apps](#native-apps)
12. [Supporting Infrastructure](#supporting-infrastructure)
13. [Comparison to Coding-Only Tools](#comparison-to-coding-only-tools)
14. [Appendix: File Counts by Directory](#appendix-file-counts-by-directory)

---

## Executive Summary

Moltbot is a **personal AI assistant platform** that connects to 20+ messaging channels (Telegram, Discord, Slack, WhatsApp, Signal, iMessage, etc.) and wraps multiple AI model providers (Anthropic Claude, OpenAI GPT, Google Gemini, and more) behind a unified runtime.

It is not a single-purpose coding tool. It is closer to a **personal AI operating system**: a persistent, always-on gateway that routes messages from any channel to AI agents equipped with tools (code execution, web browsing, image generation, messaging, memory search, etc.) and instructional skills (52 bundled skill packages).

The codebase spans ~520,000 lines of TypeScript, ~72,000 lines of Swift (macOS/iOS apps), and ~11,000 lines of Kotlin (Android app), with 29 extension plugins and 298 documentation files.

---

## Repository Metrics

| Metric | Value |
|--------|-------|
| **TypeScript (src/)** | 412,917 LOC across 2,500 files |
| **TypeScript (extensions/)** | 70,756 LOC |
| **TypeScript source files** | 1,615 |
| **TypeScript test files** | 885 |
| **Swift (macOS + iOS apps)** | 70,862 LOC |
| **Kotlin (Android app)** | 10,900 LOC |
| **Web UI** | 22,002 LOC |
| **Documentation** | 50,056 LOC across 298 files |
| **Skills** | 52 skill packages (4,877 LOC) |
| **Extensions** | 29 plugins |
| **Scripts** | 78 build/release/dev scripts |
| **Total estimated LOC** | ~600,000+ |

---

## High-Level Architecture

```
 User Messages
 (Telegram, Discord, Slack, WhatsApp, Signal, iMessage,
  Matrix, MS Teams, Line, Web, etc.)
                    |
                    v
 ┌──────────────────────────────────────────────────────────────┐
 │                    GATEWAY (src/gateway/)                    │
 │        WebSocket RPC + HTTP API + OpenAI-compatible          │
 │     Manages sessions, channels, tools, routing, hooks        │
 └──────────────────────────────────────────────────────────────┘
                    |
          ┌─────────┴──────────┐
          v                    v
 ┌─────────────────┐  ┌─────────────────┐
 │  Auto-Reply      │  │    Webhooks      │
 │  Pipeline        │  │  (Gmail, Cron)   │
 │  (src/auto-reply)│  │  (src/hooks)     │
 └────────┬─────────┘  └────────┬─────────┘
          |                     |
          v                     v
 ┌──────────────────────────────────────────────────────────────┐
 │                    AGENTS (src/agents/)                      │
 │  ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌────────────┐  │
 │  │  Models  │  │  Tools   │  │  Skills   │  │  Memory    │  │
 │  │ Claude   │  │ 25+ tools│  │ 52 skills │  │ vectors +  │  │
 │  │ GPT      │  │ bash,web │  │ SKILL.md  │  │ MEMORY.md  │  │
 │  │ Gemini   │  │ browser  │  │ files     │  │ daily logs │  │
 │  │ 20+ more │  │ message  │  │           │  │            │  │
 │  └──────────┘  └──────────┘  └───────────┘  └────────────┘  │
 └──────────────────────────────────────────────────────────────┘
                    |
                    v
 ┌──────────────────────────────────────────────────────────────┐
 │                    SESSION STORAGE                            │
 │    ~/.clawdbot/agents/<agentId>/sessions/*.jsonl              │
 │    ~/.clawdbot/agents/<agentId>/workspace/MEMORY.md           │
 │    ~/.clawdbot/memory/<id>.sqlite (vector embeddings)         │
 └──────────────────────────────────────────────────────────────┘
                    |
                    v
 ┌──────────────────────────────────────────────────────────────┐
 │                    NATIVE APPS                               │
 │          macOS (menubar) │ iOS │ Android │ Web UI            │
 └──────────────────────────────────────────────────────────────┘
```

---

## Repository Structure

```
moltbot/
├── src/                    # Core TypeScript codebase (412k LOC)
│   ├── agents/             # AI runtime: models, tools, skills, memory
│   ├── auto-reply/         # Inbound message → agent dispatch pipeline
│   ├── gateway/            # WebSocket/HTTP server, RPC
│   ├── commands/           # CLI command implementations
│   ├── cli/                # CLI framework, prompts, progress
│   ├── infra/              # HTTP, crypto, storage, process utils
│   ├── config/             # Config schema, validation, types
│   ├── channels/           # Shared channel abstractions
│   ├── telegram/           # Telegram bot
│   ├── discord/            # Discord bot
│   ├── slack/              # Slack bot
│   ├── signal/             # Signal messenger
│   ├── imessage/           # iMessage
│   ├── web/                # WhatsApp Web / web provider
│   ├── browser/            # Playwright browser automation
│   ├── memory/             # Vector memory, embeddings
│   ├── hooks/              # Gmail webhooks, hook system
│   ├── plugins/            # Plugin loader, SDK, registry
│   ├── cron/               # Scheduled tasks
│   ├── media/              # Image/video processing
│   ├── tui/                # Terminal UI components
│   ├── terminal/           # Table rendering, ANSI helpers
│   ├── tts/                # Text-to-speech
│   ├── security/           # Security policies
│   ├── routing/            # Message routing
│   ├── pairing/            # Device pairing
│   ├── sessions/           # Session management helpers
│   ├── providers/          # Provider abstractions
│   ├── canvas-host/        # Canvas/drawing tool host
│   ├── wizard/             # Setup wizard
│   └── ...                 # Other utility modules
│
├── apps/                   # Native applications
│   ├── macos/              # macOS menubar app (53k LOC Swift)
│   ├── ios/                # iOS app (7k LOC Swift)
│   └── android/            # Android app (11k LOC Kotlin)
│
├── extensions/             # 29 plugins (channels + features)
│   ├── telegram/           # Telegram (plugin variant)
│   ├── discord/            # Discord (plugin variant)
│   ├── slack/              # Slack (plugin variant)
│   ├── matrix/             # Matrix protocol
│   ├── msteams/            # Microsoft Teams
│   ├── googlechat/         # Google Chat
│   ├── line/               # LINE messenger
│   ├── whatsapp/           # WhatsApp (plugin variant)
│   ├── signal/             # Signal (plugin variant)
│   ├── bluebubbles/        # BlueBubbles (iMessage bridge)
│   ├── imessage/           # iMessage (plugin variant)
│   ├── mattermost/         # Mattermost
│   ├── nextcloud-talk/     # Nextcloud Talk
│   ├── twitch/             # Twitch chat
│   ├── nostr/              # Nostr protocol
│   ├── tlon/               # Tlon/Urbit
│   ├── zalo/               # Zalo OA
│   ├── zalouser/           # Zalo user
│   ├── voice-call/         # Voice call handling
│   ├── memory-core/        # Memory backend (core)
│   ├── memory-lancedb/     # Memory backend (LanceDB vectors)
│   ├── llm-task/           # Background LLM tasks
│   ├── diagnostics-otel/   # OpenTelemetry tracing
│   ├── copilot-proxy/      # GitHub Copilot auth proxy
│   ├── google-antigravity-auth/
│   ├── google-gemini-cli-auth/
│   ├── qwen-portal-auth/
│   ├── lobster/            # UI theming
│   └── open-prose/         # Prose tools
│
├── skills/                 # 52 skill packages
│   ├── 1password/          # 1Password CLI
│   ├── github/             # GitHub operations
│   ├── canvas/             # Drawing/visualization
│   ├── coding-agent/       # Code assistance
│   ├── notion/             # Notion integration
│   ├── bear-notes/         # Bear Notes
│   ├── obsidian/           # Obsidian vaults
│   ├── spotify-player/     # Spotify control
│   ├── weather/            # Weather lookups
│   └── ...                 # (47 more)
│
├── docs/                   # 298 Mintlify documentation files
├── scripts/                # 78 build/release/dev scripts
├── test/                   # E2E and integration tests
├── ui/                     # Web UI (22k LOC)
├── packages/               # Shared workspace packages
├── vendor/                 # Vendored dependencies
├── Swabble/                # Swift utility package
└── patches/                # pnpm dependency patches
```

---

## Core Components

### 1. Gateway (`src/gateway/`)

**187 files, 36,213 LOC**

The gateway is the central control plane. It runs as a persistent process (typically in the macOS menubar app) and manages everything.

**Responsibilities:**
- WebSocket server for real-time communication with clients
- HTTP server exposing an OpenAI-compatible API (`/v1/chat/completions`)
- JSON-RPC methods for all operations (chat, models, agents, config)
- Channel lifecycle management (start/stop/status for each messaging platform)
- Session routing (which message goes to which agent and session)
- Webhook endpoint for hooks (Gmail, cron, external triggers)
- Plugin loading and extension registration

**Key RPC methods:**
- `chat.send` — Send a message and stream the response
- `chat.history` — Retrieve conversation history
- `chat.abort` — Cancel a running agent
- `chat.inject` — Inject messages for transcript management
- `models.list` — List available models
- `agent.run` — Direct agent execution

**OpenAI-compatible endpoint** (`src/gateway/openai-http.ts`):
Exposes `/v1/chat/completions` so any OpenAI-compatible client can interact with Moltbot's agents. Supports both streaming and non-streaming responses, converting between OpenAI message format and Moltbot's internal agent commands.

---

### 2. Agents (AI Runtime) (`src/agents/`)

**435 files, 70,900 LOC**

The largest and most complex subsystem. Covered in detail in [Agent Internals (Deep Dive)](#agent-internals-deep-dive).

---

### 3. Auto-Reply Pipeline (`src/auto-reply/`)

**206 files, 39,400 LOC**

Handles the flow from an inbound message to agent dispatch and reply delivery. This is where:
- Messages are parsed (attachments, images, commands)
- Session context is loaded (history, resets)
- Agent dispatch occurs (`dispatchInboundMessage`)
- Reply formatting and delivery happen
- Channel-specific delivery rules are applied

---

### 4. Channels

Channels are messaging platform integrations. Each has its own directory with platform-specific API wiring, message formatting, and delivery logic.

**Built-in channels (in `src/`):**

| Channel | Files | LOC | Notes |
|---------|-------|-----|-------|
| `telegram/` | 84 | 18,348 | Telegram Bot API |
| `discord/` | 61 | 12,027 | Discord.js |
| `slack/` | 65 | 8,827 | Slack Bolt |
| `signal/` | 24 | 3,646 | Signal CLI bridge |
| `imessage/` | 16 | 2,380 | AppleScript / BlueBubbles |
| `web/` | 77 | 12,906 | WhatsApp Web + web provider |
| `channels/` | 101 | 10,510 | Shared abstractions |

**Extension channels (in `extensions/`):**
Matrix, MS Teams, Google Chat, LINE, Mattermost, Nextcloud Talk, Twitch, Nostr, Tlon, Zalo, BlueBubbles, WhatsApp, and more.

Each channel integration handles:
- Connecting to the platform's API/WebSocket
- Receiving and parsing inbound messages
- Formatting and delivering outbound replies
- Media attachments (images, audio, video, documents)
- Platform-specific features (reactions, threads, edits, deletions)

---

### 5. CLI (`src/cli/` + `src/commands/`)

**392 files, 61,747 LOC combined**

Commander.js-based CLI that provides the `moltbot` command.

Key commands:
- `moltbot gateway run` — Start the gateway server
- `moltbot agent --message "..."` — Run agent directly
- `moltbot config set ...` — Configure settings
- `moltbot channels status` — Show channel status
- `moltbot message send` — Send messages
- `moltbot webhooks gmail setup` — Set up Gmail integration
- `moltbot doctor` — Diagnose issues
- `moltbot login` — Authenticate with providers

---

### 6. Configuration (`src/config/`)

**130 files, 18,212 LOC**

Manages the configuration schema using Zod for validation and TypeBox for JSON schema generation.

Config sources (in precedence order):
1. CLI flags
2. Environment variables
3. `moltbot.json` (project-local)
4. `~/.moltbot/config.json5` (user-global)
5. Built-in defaults

Key config sections:
- `agents.defaults.model` — Primary model + fallbacks
- `tools` — Tool profile, allow/deny lists
- `skills` — Skill enable/disable
- `hooks` — Webhook configuration (Gmail, cron)
- `channels` — Per-channel settings
- `sandbox` — Docker sandbox settings

---

## Agent Internals (Deep Dive)

The `src/agents/` directory is the brain of the application. Here is a detailed breakdown of its subsystems.

### 1. Execution Engine (`pi-embedded-runner/`)

The core agent loop — takes a user message, runs it through an AI model, handles tool calls, and produces a response.

```
src/agents/pi-embedded-runner/
├── run.ts              (679 LOC)  # Top-level orchestrator
├── run/
│   ├── attempt.ts      (884 LOC)  # Single model invocation with retry/auth rotation
│   ├── images.ts       (423 LOC)  # Image attachment processing
│   ├── payloads.ts     (229 LOC)  # Request payload construction
│   ├── params.ts       ( 98 LOC)  # Parameter resolution
│   └── types.ts                   # Shared types
├── compact.ts          (488 LOC)  # Context compaction when history is too long
├── model.ts            (101 LOC)  # Model resolution for a given run
├── history.ts          ( 85 LOC)  # History loading and trimming
├── system-prompt.ts    ( 81 LOC)  # System prompt injection
├── extensions.ts       ( 92 LOC)  # Plugin extension hooks
├── session-manager-*.ts           # Session file initialization/caching
├── tool-split.ts                  # SDK tool format splitting
├── abort.ts                       # Abort signal handling
└── types.ts                       # Type definitions
```

**Execution flow:**
1. `run.ts` receives a message and resolved configuration
2. Resolves which model to use (primary or fallback)
3. Resolves auth profile (API key) for that model
4. Builds the request payload (system prompt + history + message)
5. `attempt.ts` executes the API call with streaming
6. Streams events through the subscription layer
7. If tool calls are returned, executes them and loops
8. On failure, rotates auth profiles or falls over to next model
9. Writes completed turn to session transcript (`.jsonl`)

---

### 2. Streaming & Event Subscription

Handles real-time processing of tokens streaming from the AI model.

| File | LOC | Purpose |
|------|-----|---------|
| `pi-embedded-subscribe.ts` | 499 | Main event subscriber |
| `pi-embedded-subscribe.handlers.messages.ts` | ~260 | Text chunking, fenced code block handling, reply splitting |
| `pi-embedded-subscribe.handlers.tools.ts` | ~190 | Tool call/result event processing |
| `pi-embedded-subscribe.handlers.lifecycle.ts` | ~70 | Session lifecycle events |
| `pi-embedded-subscribe.tools.ts` | ~160 | Tool execution dispatch |
| `pi-embedded-subscribe.types.ts` | ~40 | Type definitions |

Key behaviors:
- Splits streaming text at paragraph boundaries for progressive delivery
- Re-opens fenced code blocks when splitting mid-block
- Handles reply suppression (for internal tool chatter)
- Manages compaction retries when context overflows mid-stream
- Emits reasoning/thinking as separate messages when enabled

---

### 3. Bash / Code Execution

The most complex tool — command execution with full terminal emulation.

| File | LOC | Purpose |
|------|-----|---------|
| `bash-tools.exec.ts` | **1,495** | Command execution: PTY allocation, timeouts, approval gates, background processes |
| `bash-tools.process.ts` | 654 | Long-running process management (send keys, read output, signal) |
| `bash-tools.shared.ts` | ~190 | Output truncation, sanitization helpers |
| `bash-process-registry.ts` | ~190 | Registry tracking active background processes |
| `pty-keys.ts` | ~160 | PTY key sequence handling |
| `pty-dsr.ts` | ~15 | Device Status Report detection |

Features:
- PTY (pseudo-terminal) allocation for interactive commands
- Configurable timeouts with graceful fallback
- Approval ID system for gated commands
- Background process support with output streaming
- Process registry for cleanup
- Output size limiting and truncation

---

### 4. System Prompt Assembly

| File | LOC | Purpose |
|------|-----|---------|
| `system-prompt.ts` | 591 | Builds the full system prompt from components |
| `system-prompt-params.ts` | ~75 | Parameter resolution for prompt templates |
| `system-prompt-report.ts` | ~130 | Debug report of system prompt composition |

The system prompt is **dynamically assembled** from:
- Agent identity (name, persona)
- Available tools (descriptions, schemas)
- Active skills (conditional on OS, binaries, config)
- Memory context (MEMORY.md summary)
- Date/time information
- Channel context (which platform the message came from)
- Sandbox restrictions (if running in Docker)

---

### 5. Model Layer

Manages discovery, selection, authentication, and failover across 20+ AI model providers.

| File | LOC | Purpose |
|------|-----|---------|
| `model-selection.ts` | 363 | Pick primary model + resolve fallbacks |
| `model-fallback.ts` | 370 | Failover logic on model errors |
| `model-auth.ts` | 358 | Resolve API key for a given model |
| `model-scan.ts` | 467 | Scan and validate available models |
| `model-catalog.ts` | ~120 | Load the full model catalog from pi-ai |
| `model-compat.ts` | ~20 | Model compatibility flags |
| `models-config.providers.ts` | 503 | Provider definitions (Ollama, Bedrock, Moonshot, Venice, etc.) |
| `models-config.ts` | ~120 | Write merged models.json |
| `failover-error.ts` | ~140 | Error classification for failover decisions |
| `bedrock-discovery.ts` | ~170 | AWS Bedrock auto-discovery |
| `venice-models.ts` | ~270 | Venice AI model catalog |
| `opencode-zen-models.ts` | ~220 | OpenCode Zen model catalog |
| `synthetic-models.ts` | ~110 | Test/simulation models |

**Supported model APIs:**
- `openai-completions` — OpenAI-compatible chat completions
- `openai-responses` — OpenAI response format
- `anthropic-messages` — Anthropic Messages API
- `google-generative-ai` — Google Generative AI
- `github-copilot` — GitHub Copilot API
- `bedrock-converse-stream` — AWS Bedrock Converse

**Supported providers:**
- **Anthropic**: Claude Opus 4.5, Claude Sonnet (default)
- **OpenAI**: GPT-5.2 and variants
- **Google**: Gemini 3 Pro/Flash
- **OpenRouter**: xAI, Groq, Cerebras, Mistral, and more
- **GitHub Copilot**: Via OAuth token exchange
- **AWS Bedrock**: Auto-discovery by region
- **Ollama**: Local models via HTTP discovery
- **MiniMax**, **Moonshot (Kimi)**, **Qwen**, **Venice AI**, **Z.AI/GLM**, **Vercel AI Gateway**

**Failover chain:**
When a model fails (auth error, rate limit, context overflow, timeout), the system automatically tries the next model in the fallback list. Auth profiles are rotated before switching models entirely.

---

### 6. Auth & Credentials

| File | LOC | Purpose |
|------|-----|---------|
| `cli-credentials.ts` | 499 | Interactive credential setup |
| `auth-profiles/store.ts` | | Persistent auth profile storage (`~/.moltbot/agents/auth-profiles/auth.json`) |
| `auth-profiles/order.ts` | | Profile ordering and priority |
| `auth-profiles/oauth.ts` | | OAuth flows (GitHub Copilot, Qwen, etc.) |
| `auth-profiles/doctor.ts` | | Auth health checks |
| `auth-profiles/types.ts` | | Type definitions |
| `auth-health.ts` | ~170 | Validate API keys are still working |
| `chutes-oauth.ts` | ~150 | Chutes provider OAuth flow |

Supports multiple API keys per provider, with round-robin rotation and "last good" tracking. Cooldown periods are applied to failed profiles.

---

### 7. Tools

Tools are JSON-schema functions that the AI model can call during a conversation. Defined in `src/agents/tools/` (56 files).

**Coding tools:**
| Tool | File | Purpose |
|------|------|---------|
| `exec` / `bash` | `bash-tools.exec.ts` | Execute shell commands |
| `read` | `pi-tools.read.ts` | Read file contents |
| `write` | (in pi-tools) | Write/create files |
| `edit` | (in pi-tools) | Edit existing files |

**Web tools:**
| Tool | File | Purpose |
|------|------|---------|
| `web_search` | `web-search.ts` | Search via Brave API or Perplexity |
| `web_fetch` | `web-fetch.ts` | Fetch URL content (fetch + Readability, Firecrawl fallback) |
| `browser` | `browser-tool.ts` | Full Playwright browser automation |

**Messaging tools:**
| Tool | File | Purpose |
|------|------|---------|
| `message` | `message-tool.ts` | Send/react/delete across channels |
| `discord_*` | `discord-actions.ts` | Discord-specific actions (guilds, moderation) |
| `slack_*` | `slack-actions.ts` | Slack-specific actions |
| `telegram_*` | `telegram-actions.ts` | Telegram-specific actions |
| `whatsapp_*` | `whatsapp-actions.ts` | WhatsApp-specific actions |

**Session/agent tools:**
| Tool | File | Purpose |
|------|------|---------|
| `sessions_list` | `sessions-list-tool.ts` | List active sessions |
| `sessions_send` | `sessions-send-tool.ts` | Send to other agent sessions |
| `sessions_spawn` | `sessions-spawn-tool.ts` | Spawn sub-agent sessions |
| `session_status` | `session-status-tool.ts` | Get session metadata |
| `agents_list` | `agents-list-tool.ts` | List available agents |

**Media tools:**
| Tool | File | Purpose |
|------|------|---------|
| `image` | `image-tool.ts` | Image generation (DALL-E, etc.) |
| `tts` | `tts-tool.ts` | Text-to-speech |
| `canvas` | `canvas-tool.ts` | Drawing/visualization |

**Other tools:**
| Tool | File | Purpose |
|------|------|---------|
| `memory_search` | `memory-tool.ts` | Search vector memory |
| `cron` | `cron-tool.ts` | Schedule recurring tasks |
| `gateway` | `gateway-tool.ts` | Gateway control (restart, config) |
| `nodes` | `nodes-tool.ts` | Multi-step node execution |

**Tool policy** (`pi-tools.policy.ts`, `tool-policy.ts`):
Tools are filtered through multiple policy layers:
1. Profile: `"coding"`, `"messaging"`, or `"full"`
2. Global allow/deny lists
3. Agent-specific policies
4. Sandbox restrictions
5. Plugin-contributed tool permissions

---

### 8. Skills System

Skills are **instructional packages** — not executable tools. They teach the agent how to use external capabilities by injecting guidance into the system prompt.

**Structure of a skill:**
```
skills/<skill-name>/
└── SKILL.md       # YAML frontmatter + markdown instructions
```

**Example `SKILL.md` frontmatter:**
```yaml
---
name: 1password
description: Use 1Password CLI (op) to manage secrets
metadata:
  moltbot:
    emoji: "lock"
    requires:
      bins: ["op"]
      platform: ["darwin", "linux"]
---
Instructions for using 1Password CLI...
```

**Skill loading code:**

| File | Purpose |
|------|---------|
| `skills.ts` | Entry point |
| `skills/workspace.ts` | Load from workspace, bundled, and managed dirs |
| `skills/frontmatter.ts` | Parse YAML frontmatter from SKILL.md |
| `skills/config.ts` | Enable/disable configuration |
| `skills/env-overrides.ts` | Inject env vars for skills |
| `skills/refresh.ts` | Hot-reload on changes |
| `skills/plugin-skills.ts` | Skills contributed by plugins |
| `skills-install.ts` | Install skills from registry |
| `skills-status.ts` | Show skill status |

**Loading precedence:**
1. `<workspace>/skills/` (highest priority)
2. `~/.clawdbot/skills/` (user-managed)
3. Bundled skills in repo `skills/` (lowest)

**Gating:** Skills are conditionally loaded based on:
- Required binaries (e.g., `op` for 1Password)
- Operating system (`darwin`, `linux`, `win32`)
- Environment variables
- Config flags (`skills.entries.<name>.enabled`)

**52 bundled skills include:**
1password, apple-notes, apple-reminders, bear-notes, bird, blogwatcher, blucli, bluebubbles, camsnap, canvas, clawdhub, coding-agent, discord, eightctl, food-order, gemini, gifgrep, github, gog, goplaces, himalaya, imsg, local-places, mcporter, model-usage, nano-banana-pro, nano-pdf, notion, obsidian, openai-image-gen, openai-whisper, openai-whisper-api, openhue, oracle, ordercli, peekaboo, sag, session-logs, sherpa-onnx-tts, skill-creator, slack, songsee, sonoscli, spotify-player, summarize, things-mac, tmux, trello, video-frames, voice-call, wacli, weather

---

### 9. Memory System

Moltbot has a multi-layered memory architecture:

| Layer | Storage | Purpose |
|-------|---------|---------|
| **Session Transcript** | `~/.clawdbot/agents/<id>/sessions/*.jsonl` | Full conversation history, append-only JSONL |
| **Semantic Memory** | `~/.clawdbot/agents/<id>/workspace/MEMORY.md` | Curated long-term memory (agent-maintained) |
| **Daily Logs** | `workspace/memory/YYYY-MM-DD.md` | Running notes, indexed for search |
| **Vector Index** | `~/.clawdbot/memory/<id>.sqlite` | Embeddings for semantic search |
| **RAM Cache** | In-memory LRU | Last ~50 messages per session for quick access |

**Memory code:**

| File | LOC | Purpose |
|------|-----|---------|
| `src/memory/` (33 files) | 6,801 | Vector storage, embedding generation, search |
| `memory-search.ts` | 291 | Memory search tool implementation |
| `extensions/memory-core/` | | Core memory backend plugin |
| `extensions/memory-lancedb/` | | LanceDB vector backend |

---

### 10. Sub-Agents (Multi-Agent)

Moltbot supports spawning and managing multiple concurrent agents.

| File | LOC | Purpose |
|------|-----|---------|
| `subagent-registry.ts` | 371 | Track spawned sub-agents |
| `subagent-announce.ts` | 471 | Format and deliver sub-agent results to parent |
| `subagent-announce-queue.ts` | ~140 | Queue announcements for delivery |
| `subagent-registry.store.ts` | ~100 | Persistent sub-agent state |

**Capabilities:**
- Run multiple named agents with different configurations
- Spawn child agents from within a session (`sessions_spawn` tool)
- Route different channels to different agents
- Track sub-agent lifecycle and results
- Announce sub-agent completion to the parent agent
- Cross-agent session communication (`sessions_send` tool)

---

### 11. Sandbox

Docker-based sandboxing for untrusted code execution.

```
src/agents/sandbox/
├── config.ts           # Sandbox configuration resolution
├── docker.ts           # Docker container management
├── manage.ts           # Sandbox lifecycle (create, start, stop)
├── registry.ts         # Active sandbox tracking
├── context.ts          # Sandbox context resolution
├── workspace.ts        # Workspace mounting
├── browser.ts          # Browser-in-sandbox support
├── browser-bridges.ts  # Browser bridge configuration
├── prune.ts            # Clean up old sandbox containers
├── tool-policy.ts      # Sandbox tool restrictions
├── types.ts            # Type definitions
└── constants.ts        # Default settings
```

---

### 12. Compaction

When a conversation approaches the model's context window limit, compaction kicks in.

| File | LOC | Purpose |
|------|-----|---------|
| `compaction.ts` | 345 | Summarize old conversation history |
| `pi-embedded-runner/compact.ts` | 488 | Compaction orchestration during runs |
| `context-window-guard.ts` | ~65 | Monitor context usage |
| `pi-extensions/compaction-safeguard.ts` | | Runtime safeguards |

**How it works:**
1. Monitor token count as conversation grows
2. When approaching the limit, take older messages
3. Summarize them into a compact representation
4. Keep recent messages intact
5. Replace old messages with the summary
6. Continue the conversation with reduced context

---

## AI Model Integration

Models are abstracted through the `@mariozechner/pi-ai` and `@mariozechner/pi-coding-agent` libraries.

### Model Discovery Flow

```
Configuration Loading (ensureMoltbotModelsJson)
    |
    ├── Load explicit providers from moltbot.json
    ├── Resolve implicit providers from env vars / auth profiles
    ├── Discover AWS Bedrock models (if configured)
    ├── Discover Ollama local models via HTTP
    └── Merge all into ~/.moltbot/agents/models.json
           |
           v
Model Registry (loadModelCatalog)
    |
    ├── Query pi-coding-agent discoverModels()
    ├── Filter by authentication availability
    └── Return sorted catalog with metadata
           |
           v
Model Resolution (resolveModel)
    |
    ├── Check pi-ai built-in catalog
    ├── Fall back to inline models.providers config
    └── Return typed Model<Api> object
```

### Defaults

```
Default provider:  anthropic
Default model:     claude-opus-4-5
Context window:    200,000 tokens
```

### Thinking Levels

Models that support extended thinking can be configured with:
```
off | minimal | low | medium | high | xhigh
```

---

## Browser Automation

Full browser control via Playwright (`src/browser/`, 81 files, 15,615 LOC).

### How the LLM Sees Web Pages

The LLM receives **two complementary representations**:

**1. Accessibility Tree Snapshot (primary)**

Playwright captures the browser's ARIA accessibility tree and formats it as structured text with interactive element references:

```
- navigation "Main"
  - link "Home" [ref=e1]
  - link "Products" [ref=e2]
- main
  - heading "Welcome" [ref=e3]
  - textbox "Search" [ref=e4]
  - button "Submit" [ref=e5]
```

The LLM reads this tree, picks a reference (e.g., `e5`), and says "click e5". Moltbot resolves the reference to the actual DOM element and performs the action.

**2. Screenshots (visual)**

Playwright takes screenshots, which are resized (max 2000px) and compressed to JPEG (under 5MB) before being sent to the LLM as image content. This gives multimodal models (Claude, GPT-4V) visual context.

### Browser Modes

- **Chrome Extension Relay**: Takes over existing Chrome tabs via a browser extension
- **Isolated Browser**: Moltbot launches and controls its own Playwright browser instance

### Key Browser Files

| File | Purpose |
|------|---------|
| `pw-role-snapshot.ts` | Accessibility tree processing and ref assignment |
| `screenshot.ts` | Screenshot capture, resize, compression |
| `pw-tools-core.ts` | Core Playwright operations |
| `pw-tools-core.interactions.ts` | Click, type, scroll, hover |
| `pw-tools-core.snapshot.ts` | Page snapshot capture |
| `pw-tools-core.state.ts` | Browser state management |
| `pw-session.ts` | Playwright session lifecycle |
| `server.ts` | Browser control HTTP server |
| `client.ts` | Browser control client |

---

## Email / Webhooks

Email is a **built-in core feature** (not a plugin), but it is limited to **Gmail only** and uses a **webhook-based** (not IMAP) architecture.

### How It Works

```
Gmail inbox
    |
    v
Google Cloud Pub/Sub (push notification)
    |
    v
gog CLI webhook server (local, port 8788)
    |
    v
Moltbot hooks endpoint (/hooks/gmail)
    |
    v
Agent wakes up and processes the email
```

**Gmail Watch** monitors the inbox via the Gmail API. When a new email arrives, Google Pub/Sub pushes a notification to a local webhook endpoint. Moltbot processes the email and can wake an agent or deliver a summary to a messaging channel.

### Key Files

| File | Purpose |
|------|---------|
| `src/hooks/gmail.ts` | Configuration defaults |
| `src/hooks/gmail-ops.ts` | Setup and run functions |
| `src/hooks/gmail-watcher.ts` | Auto-starts on gateway boot, auto-renews watch |
| `src/hooks/gmail-setup-utils.ts` | Dependency install, GCP auth, Pub/Sub setup |
| `src/cli/webhooks-cli.ts` | CLI commands (`moltbot webhooks gmail setup/run`) |

### Requirements

- `gcloud` (Google Cloud SDK)
- `gog` (Gmail API CLI tool)
- `tailscale` (for public endpoint via Tailscale Funnel)

### Limitations

- Gmail only (no Outlook, Yahoo, ProtonMail, etc.)
- Receiving via Pub/Sub only (no IMAP polling)
- Subject + body snippet (no full attachment support)
- No general SMTP sending capability

---

## Extensions & Plugins

Extensions are workspace packages under `extensions/`. They extend Moltbot with additional channels, auth providers, memory backends, and features.

### Plugin Architecture

| File | Purpose |
|------|---------|
| `src/plugins/` (37 files, 6,884 LOC) | Plugin loader, SDK, registry |
| `src/plugin-sdk/` | SDK exposed to plugins |

Plugins are loaded at runtime by the gateway. Each plugin can:
- Register new channels (messaging platforms)
- Add tools
- Contribute skills
- Provide auth mechanisms
- Add memory backends

### Extension Categories

**Messaging channels (15):**
`telegram`, `discord`, `slack`, `signal`, `whatsapp`, `matrix`, `msteams`, `googlechat`, `line`, `mattermost`, `nextcloud-talk`, `twitch`, `nostr`, `tlon`, `zalo`/`zalouser`, `bluebubbles`, `imessage`

**Auth providers (4):**
`copilot-proxy`, `google-antigravity-auth`, `google-gemini-cli-auth`, `qwen-portal-auth`

**Memory backends (2):**
`memory-core`, `memory-lancedb`

**Features (4):**
`voice-call`, `llm-task`, `diagnostics-otel`, `lobster`, `open-prose`

---

## Native Apps

### macOS Menubar App (`apps/macos/`)

**295 Swift files, ~53,300 LOC**

The primary way to run Moltbot on macOS. Lives in the menu bar and:
- Runs the gateway as a background process
- Provides a native UI for configuration
- Handles channel status and controls
- Manages the agent lifecycle

### iOS App (`apps/ios/`)

**47 Swift files, ~7,200 LOC**

Mobile companion app for iPhone/iPad.

### Android App (`apps/android/`)

**63 Kotlin files, ~10,900 LOC**

Mobile companion app for Android devices.

### Web UI (`ui/`)

**22,002 LOC**

Browser-based interface for interacting with Moltbot.

---

## Supporting Infrastructure

| Directory | Files | LOC | Purpose |
|-----------|-------|-----|---------|
| `src/infra/` | 182 | 28,654 | HTTP clients, crypto, file storage, process management, key-value stores |
| `src/cron/` | 30 | 4,384 | Scheduled task system |
| `src/security/` | 9 | 4,718 | Security policies, content filtering |
| `src/media/` | 19 | 2,707 | Image/video processing, resizing, format conversion |
| `src/tts/` | | | Text-to-speech pipeline |
| `src/tui/` | 37 | 4,938 | Terminal UI components |
| `src/terminal/` | 11 | 805 | Table rendering, ANSI color helpers, palette |
| `src/pairing/` | 5 | 685 | Device pairing for mobile apps |
| `src/routing/` | 4 | 768 | Message routing logic |
| `src/wizard/` | 9 | 2,010 | Interactive setup wizard |
| `src/logging/` | | | Structured logging |

---

## Comparison to Coding-Only Tools

Moltbot shares its agent core architecture with tools like Claude Code and OpenCode, but diverges significantly in scope.

### Shared (~15% of codebase)

The inner agent loop is structurally identical:
- System prompt + history -> LLM API call -> tool execution -> loop
- Bash/code execution with PTY, timeouts, approvals
- File read/write/edit tools
- Context compaction on overflow
- Session transcripts (JSONL)
- Model abstraction across providers

### Unique to Moltbot (~85% of codebase)

| Capability | Claude Code / OpenCode | Moltbot |
|------------|----------------------|---------|
| **Interface** | Terminal only | 20+ messaging channels + native apps |
| **Runtime** | Invoked on demand | Always-on gateway server |
| **Agents** | Single agent | Multi-agent with sub-agent spawning |
| **Skills** | Hardcoded prompt | 52 pluggable skill packages |
| **Non-coding tools** | ~5 (read, write, edit, bash, search) | 25+ (messaging, browser, image gen, TTS, memory, cron, etc.) |
| **Memory** | Current session only | Multi-layer persistent memory |
| **Plugins** | None | 29 extension plugins |
| **Native apps** | None | macOS, iOS, Android |
| **Email/Webhooks** | None | Gmail Pub/Sub integration |
| **Size** | ~30-40k LOC | ~600k LOC |

---

## Appendix: File Counts by Directory

**`src/` subdirectories, sorted by LOC:**

| Directory | LOC | Files | Description |
|-----------|-----|-------|-------------|
| `agents/` | 70,900 | 435 | AI runtime: models, tools, skills, memory |
| `auto-reply/` | 39,400 | 206 | Message → agent dispatch pipeline |
| `commands/` | 36,897 | 223 | CLI command implementations |
| `gateway/` | 36,213 | 187 | WebSocket/HTTP server, RPC |
| `infra/` | 28,654 | 182 | Utilities: HTTP, crypto, storage |
| `cli/` | 24,850 | 169 | CLI framework, prompts, progress |
| `config/` | 18,212 | 130 | Config schema, validation, types |
| `telegram/` | 18,348 | 84 | Telegram bot integration |
| `browser/` | 15,615 | 81 | Playwright browser automation |
| `web/` | 12,906 | 77 | WhatsApp Web / web provider |
| `discord/` | 12,027 | 61 | Discord bot integration |
| `channels/` | 10,510 | 101 | Shared channel abstractions |
| `slack/` | 8,827 | 65 | Slack bot integration |
| `plugins/` | 6,884 | 37 | Plugin loader, SDK, registry |
| `memory/` | 6,801 | 33 | Vector memory, embeddings |
| `hooks/` | 5,688 | 33 | Gmail webhooks, hook system |
| `tui/` | 4,938 | 37 | Terminal UI components |
| `security/` | 4,718 | 9 | Security policies |
| `cron/` | 4,384 | 30 | Scheduled tasks |
| `signal/` | 3,646 | 24 | Signal messenger integration |
| `media/` | 2,707 | 19 | Image/video processing |
| `imessage/` | 2,380 | 16 | iMessage integration |
| `wizard/` | 2,010 | 9 | Setup wizard |
| `providers/` | 1,169 | 8 | Provider abstractions |
| `canvas-host/` | 917 | 3 | Canvas drawing host |
| `utils/` | 876 | 13 | Shared utilities |
| `terminal/` | 805 | 11 | Table rendering, ANSI |
| `routing/` | 768 | 4 | Message routing |
| `pairing/` | 685 | 5 | Device pairing |
| `sessions/` | 328 | 7 | Session helpers |
| `types/` | 165 | 9 | Shared type definitions |
| `shared/` | 61 | 1 | Shared text utilities |
