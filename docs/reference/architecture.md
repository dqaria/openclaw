# Architecture Overview

This document provides a comprehensive architectural overview of the OpenClaw project, including system-level diagrams, per-module breakdowns, and the full dependency tree.

---

## 1. High-Level System Architecture

```mermaid
graph TB
    subgraph Users["Users & Clients"]
        TG[Telegram]
        DC[Discord]
        SL[Slack]
        SG[Signal]
        IM[iMessage]
        WA[WhatsApp]
        LN[LINE]
        WEB[Web UI]
        EXT_CH["Extension Channels<br/>(Teams, Matrix, Zalo, ...)"]
    end

    subgraph CLI["CLI Layer"]
        ENTRY[entry.ts]
        PROGRAM[Commander Program]
        SUBCMDS[Subcommands<br/>agent, send, config, status, ...]
    end

    subgraph Gateway["Gateway Server"]
        GW_HTTP[Hono HTTP Server]
        GW_WS[WebSocket Server]
        GW_RPC[RPC Method Handlers]
        GW_CHAN_MGR[Channel Manager]
        GW_STATE[Runtime State]
        GW_DISC[mDNS Discovery]
        GW_CRON[Cron Service]
        GW_MODEL[Model Catalog]
    end

    subgraph Agents["Agent Runtime"]
        PI_CORE["Pi Agent Core<br/>(@mariozechner/pi-*)"]
        TOOLS[Tool Registry]
        WORKSPACE[Agent Workspace]
        SESSIONS[Session Manager]
        MEMORY[Memory / Search]
        SANDBOX[Sandbox Environment]
    end

    subgraph Channels["Channel Layer"]
        CH_REG[Channel Registry]
        CH_PLUGINS[Channel Plugins]
        CH_ALLOW[Allowlist / Gating]
        CH_ROUTING[Routing / Bindings]
    end

    subgraph Plugins["Plugin System"]
        PL_DISC[Plugin Discovery]
        PL_LOAD[Plugin Loader]
        PL_REG[Plugin Registry]
        PL_SDK[Plugin SDK]
    end

    subgraph Infra["Infrastructure"]
        CONFIG[Config Manager]
        MEDIA[Media Pipeline]
        LOGGING[Logging]
        SECURITY[Security]
        PROVIDERS[LLM Providers]
        TLS[TLS / Tunneling]
    end

    subgraph Apps["Native Apps"]
        MACOS[macOS Menubar App]
        IOS[iOS App]
        ANDROID[Android App]
    end

    subgraph WebUI["Web Control UI"]
        LIT[Lit Components]
        VITE[Vite Dev Server]
    end

    %% User connections
    TG & DC & SL & SG & IM & WA & LN --> CH_PLUGINS
    EXT_CH --> CH_PLUGINS
    WEB --> GW_HTTP

    %% CLI flow
    ENTRY --> PROGRAM --> SUBCMDS
    SUBCMDS --> Gateway
    SUBCMDS --> Channels

    %% Gateway orchestration
    GW_HTTP --> GW_WS --> GW_RPC
    GW_RPC --> GW_CHAN_MGR
    GW_RPC --> Agents
    GW_CHAN_MGR --> CH_PLUGINS
    GW_RPC --> GW_MODEL

    %% Channel flow
    CH_PLUGINS --> CH_REG
    CH_PLUGINS --> CH_ALLOW
    CH_PLUGINS --> CH_ROUTING
    CH_ROUTING --> Agents

    %% Agent flow
    PI_CORE --> TOOLS
    PI_CORE --> WORKSPACE
    PI_CORE --> SESSIONS
    PI_CORE --> MEMORY
    TOOLS --> SANDBOX

    %% Plugin flow
    PL_DISC --> PL_LOAD --> PL_REG
    PL_SDK --> PL_REG
    PL_REG --> CH_PLUGINS
    PL_REG --> TOOLS
    PL_REG --> GW_RPC

    %% Infrastructure
    CONFIG --> Gateway
    CONFIG --> Channels
    MEDIA --> Channels
    MEDIA --> Agents
    PROVIDERS --> Agents
    LOGGING --> Gateway

    %% Native apps
    MACOS --> GW_WS
    IOS --> GW_WS
    ANDROID --> GW_WS

    %% Web UI
    WebUI --> GW_HTTP
```

---

## 2. Message Flow Architecture

The core data flow for receiving and responding to messages:

```mermaid
sequenceDiagram
    participant User as User (Telegram/Discord/...)
    participant Channel as Channel Monitor
    participant Router as Routing Engine
    participant Agent as Pi Agent
    participant Tools as Tool Registry
    participant LLM as LLM Provider
    participant Send as Channel Sender

    User->>Channel: Incoming message
    Channel->>Channel: Parse event, extract text/media
    Channel->>Router: resolveAgentRoute(channel, peer, account)
    Router->>Router: Check bindings (peer → guild → account → channel)
    Router-->>Agent: agentId + sessionKey

    Agent->>Agent: Load session context
    Agent->>LLM: Send prompt + history
    LLM-->>Agent: Response (may include tool calls)

    opt Tool Calls
        Agent->>Tools: Execute tool
        Tools-->>Agent: Tool result
        Agent->>LLM: Send tool result
        LLM-->>Agent: Final response
    end

    Agent->>Send: sendMessage(channel, peer, response)
    Send->>User: Deliver reply
```

---

## 3. Module Architecture Diagrams

### 3.1 CLI Module (`src/cli/`)

```mermaid
graph TB
    subgraph Entry["Entry Point"]
        ENTRY_TS[entry.ts<br/>Node respawner]
        INDEX_TS[index.ts<br/>buildProgram]
    end

    subgraph Program["Program Registration"]
        REG[register.subclis.ts<br/>Lazy subcommand loader]
        HELP[configureProgramHelp]
        HOOKS[registerPreActionHooks]
    end

    subgraph Commands["Command Groups"]
        AGENT[agent<br/>TUI, CLI, RPC modes]
        MSG[message / send<br/>Send to channels]
        GW_CMD[gateway<br/>run, register, discover]
        STATUS[status / health<br/>System status]
        CONFIG_CMD[config<br/>Get, set, edit]
        ONBOARD[onboard / setup<br/>First-time wizard]
        CHANNELS[channels<br/>Login, logout, status]
        MODELS[models<br/>Configure LLM models]
        PLUGINS_CMD[plugins<br/>Install, list, remove]
        SKILLS[skills<br/>List, enable, disable]
        BROWSER[browser<br/>Browser tools]
        NODES[nodes<br/>Remote node mgmt]
        MEMORY_CMD[memory<br/>Search memory]
        TUI_CMD[tui<br/>Terminal UI chat]
    end

    subgraph Deps["Dependency Injection"]
        CLI_DEPS[createDefaultDeps<br/>CliDeps factory]
        SEND_DEPS[createOutboundSendDeps]
    end

    ENTRY_TS --> INDEX_TS
    INDEX_TS --> Program
    REG --> Commands
    CLI_DEPS --> SEND_DEPS
    Commands --> CLI_DEPS
```

### 3.2 Gateway Module (`src/gateway/`)

```mermaid
graph TB
    subgraph Init["Initialization"]
        IMPL[server.impl.ts<br/>Main orchestrator]
        LOAD_CFG[loadConfig]
        LOAD_PLUGINS[loadGatewayPlugins]
        LOAD_MODELS[loadGatewayModelCatalog]
    end

    subgraph Server["HTTP/WS Server"]
        HONO[Hono HTTP App]
        WS[WebSocket Upgrade]
        AUTH[Device Auth]
        HEALTH[Health Endpoint]
    end

    subgraph Methods["RPC Methods"]
        CHAT[chat]
        TOOL_RES[tool_result]
        PROBE[probe]
        CHAN_CTRL[channel control<br/>start, stop, status]
        PLUGIN_M[plugin-contributed methods]
    end

    subgraph Managers["Service Managers"]
        CHAN_MGR[Channel Manager<br/>start/stop/status per account]
        NODE_MGR[Node Subscription Manager]
        APPROVAL[Exec Approval Handlers]
        RUNTIME[Gateway Runtime State]
        SIDECARS[Sidecars<br/>Browser, Canvas]
    end

    subgraph Protocol["Protocol"]
        ENCODE[Protocol Encoding]
        DECODE[Protocol Decoding]
        SCHEMA[Message Schema]
    end

    IMPL --> LOAD_CFG --> LOAD_PLUGINS --> LOAD_MODELS
    IMPL --> Server
    HONO --> WS --> Methods
    Methods --> Managers
    CHAN_MGR --> Protocol
    WS --> Protocol
```

### 3.3 Channel Layer (`src/channels/` + `src/telegram/`, `src/discord/`, etc.)

```mermaid
graph TB
    subgraph Registry["Channel Registry"]
        REG[registry.ts<br/>CHAT_CHANNEL_ORDER]
        PLUGIN_TYPE[ChannelPlugin interface]
    end

    subgraph Shared["Shared Infrastructure"]
        ALLOW[allowlist/<br/>Access control]
        GATING[command-gating.ts<br/>mention-gating.ts]
        ACK[ack-reactions.ts]
        SENDER[sender-identity.ts]
        LABEL[conversation-label.ts]
        PREFIX[reply-prefix.ts]
    end

    subgraph CoreChannels["Core Channel Implementations"]
        TG_MOD[telegram/<br/>monitor, send, probe]
        DC_MOD[discord/<br/>monitor, send, probe]
        SL_MOD[slack/<br/>monitor, send, probe]
        SG_MOD[signal/<br/>monitor, send, probe]
        IM_MOD[imessage/<br/>monitor, send, probe]
        WA_MOD[web/ (WhatsApp)<br/>monitor, send, probe]
        LN_MOD[line/<br/>monitor, send, probe]
    end

    subgraph ExtChannels["Extension Channel Plugins"]
        TEAMS[msteams]
        MATRIX[matrix]
        GCHAT[googlechat]
        ZALO[zalo]
        NOSTR[nostr]
        TLON[tlon]
        MORE[...]
    end

    REG --> CoreChannels
    REG --> ExtChannels
    PLUGIN_TYPE --> CoreChannels
    PLUGIN_TYPE --> ExtChannels
    Shared --> CoreChannels
    Shared --> ExtChannels
```

### 3.4 Agent Runtime (`src/agents/`)

```mermaid
graph TB
    subgraph Core["Agent Core"]
        PI[Pi Agent Core<br/>@mariozechner/pi-agent-core]
        PI_AI[Pi AI<br/>@mariozechner/pi-ai]
        PI_CODING[Pi Coding Agent<br/>@mariozechner/pi-coding-agent]
    end

    subgraph Runners["Agent Runners"]
        CLI_RUN[cli-runner/<br/>CLI agent execution]
        EMBED_RUN[pi-embedded-runner/<br/>Embedded runtime]
        EMBED_HELP[pi-embedded-helpers/<br/>Sanitization, utils]
        EMBED_SUB[pi-embedded-subscribe/<br/>Stream handling]
    end

    subgraph ToolSystem["Tool System"]
        TOOL_REG[tools/<br/>Tool definitions]
        TOOL_POLICY[Tool policy & approval]
        BASH_TOOL[Bash tool execution]
        SKILL_TOOLS[Skill-contributed tools]
        PLUGIN_TOOLS[Plugin-contributed tools]
    end

    subgraph State["State Management"]
        AUTH_PROF[auth-profiles/<br/>Auth profile mgmt]
        SESSIONS_MGR[Session management]
        SCHEMA_VAL[schema/<br/>Schema validators]
        SANDBOX_ENV[sandbox/<br/>Sandbox config]
    end

    subgraph Skills["Skills System"]
        SKILL_REG[skills/<br/>Skills registry]
        SKILL_55["55 skill packages<br/>(apple-notes, spotify, github, ...)"]
    end

    Core --> Runners
    Runners --> ToolSystem
    Runners --> State
    ToolSystem --> Skills
    SKILL_REG --> SKILL_55
    PLUGIN_TOOLS --> ToolSystem
```

### 3.5 Plugin System (`src/plugins/`)

```mermaid
graph TB
    subgraph Discovery["Discovery"]
        DISC[discovery.ts<br/>Scan extensions/, node_modules]
        MANIFEST[manifest.ts<br/>Parse plugin metadata]
    end

    subgraph Loading["Loading"]
        LOADER[loader.ts<br/>Require + register]
        BUNDLED[bundled-dir/<br/>Built-in plugins]
    end

    subgraph Registry["Registry"]
        REG[registry.ts<br/>Central registration store]
        TOOL_R[Tool Registrations]
        CLI_R[CLI Registrations]
        CHAN_R[Channel Registrations]
        PROV_R[Provider Registrations]
        HOOK_R[Hook Registrations]
        HTTP_R[HTTP Route Registrations]
        GW_R[Gateway Handler Registrations]
    end

    subgraph SDK["Plugin SDK (exports)"]
        SDK_CHAN[Channel types & adapters]
        SDK_API[Plugin API]
        SDK_GW[Gateway types]
        SDK_CFG[Config schemas]
    end

    subgraph Runtime["Runtime"]
        RT[runtime/<br/>Plugin lifecycle]
        SVC[Service management]
    end

    DISC --> MANIFEST --> LOADER
    LOADER --> REG
    REG --> TOOL_R & CLI_R & CHAN_R & PROV_R & HOOK_R & HTTP_R & GW_R
    SDK --> LOADER
    REG --> Runtime
```

### 3.6 Media Pipeline (`src/media/`)

```mermaid
graph LR
    subgraph Input["Input"]
        FETCH[fetch.ts<br/>Download from URL]
        PARSE[parse.ts<br/>MIME detection]
        INPUT_F[input-files.ts<br/>File handling]
    end

    subgraph Processing["Processing"]
        AUDIO[audio.ts<br/>Codec, tags]
        IMAGE[image-ops.ts<br/>Resize via Sharp]
        OPTIMIZE[Optimize to JPEG]
    end

    subgraph Storage["Storage & Serving"]
        STORE[store.ts<br/>Local storage]
        HOST[host.ts<br/>Ensure hosted URL]
        SERVER[HTTP media server]
    end

    subgraph Understanding["Understanding"]
        MEDIA_U[media-understanding/<br/>Vision, OCR]
        LINK_U[link-understanding/<br/>Link preview]
    end

    subgraph ChannelAdapters["Channel-Specific"]
        TG_M[Telegram: file_id cache]
        DC_M[Discord: attachment URL]
        SL_M[Slack: file upload]
        WA_M[WhatsApp: base64]
        SG_M[Signal: binary upload]
    end

    Input --> Processing --> Storage
    Storage --> ChannelAdapters
    Input --> Understanding
```

### 3.7 Routing Engine (`src/routing/`)

```mermaid
graph TB
    subgraph Input["Route Input"]
        CHAN_ID[Channel ID]
        ACCOUNT[Account ID]
        PEER[Peer / Sender]
        GUILD[Guild / Team ID]
    end

    subgraph Resolution["Route Resolution"]
        RESOLVE[resolve-route.ts<br/>resolveAgentRoute]
        BINDINGS[bindings.ts<br/>Config-based bindings]
        SESSION_KEY[session-key.ts<br/>Build session keys]
    end

    subgraph Output["Route Output"]
        AGENT_ID[Agent ID]
        SESS_KEY[Session Key]
        MATCH[Match Source<br/>binding.peer / binding.guild / default]
    end

    subgraph Binding["Binding Priority"]
        B1[1. Peer binding]
        B2[2. Guild / Team binding]
        B3[3. Account binding]
        B4[4. Channel binding]
        B5[5. Default agent]
    end

    Input --> RESOLVE
    RESOLVE --> BINDINGS
    BINDINGS --> Binding
    RESOLVE --> SESSION_KEY
    RESOLVE --> Output
```

### 3.8 Infrastructure (`src/infra/`)

```mermaid
graph TB
    subgraph Network["Network"]
        PORTS[ports.ts<br/>Port inspection]
        NET[net/<br/>Networking utils]
        OUTBOUND[outbound/<br/>HTTP client]
        TLS_MOD[tls/<br/>TLS management]
        TAILSCALE[Tailscale integration]
        SSH[SSH tunneling]
    end

    subgraph Runtime["Runtime"]
        ENV[Shell environment]
        EXEC[Process execution]
        PATHS[Path management]
        ERRORS[Error handling]
        UPDATES[Update checker]
    end

    subgraph Data["Data"]
        ARCHIVE[Archive management]
        MIGRATIONS[State migrations]
        PROVIDER_USAGE[Provider usage tracking]
    end

    subgraph Config["Config System"]
        CFG_LOAD[config.ts<br/>Load JSON5]
        CFG_SCHEMA[Schema validation]
        CFG_MIGRATE[Legacy migration]
    end

    Network --> Runtime
    Runtime --> Data
    Config --> Runtime
```

### 3.9 Native Apps Architecture

```mermaid
graph TB
    subgraph macOS["macOS App (Swift/SwiftUI)"]
        MENU[Menubar App]
        GW_CTRL[Gateway Control]
        AUDIO_CAP[Audio Capture]
        VOICE_WAKE[Voice Wake]
        SETTINGS[Settings UI]
        CRON_UI[Cron Jobs UI]
        NOTIF[Notifications]
        ONBOARD_MAC[Onboarding]
    end

    subgraph iOS["iOS App (Swift/SwiftUI)"]
        CHAT_UI[Chat UI]
        CAMERA[Camera Module]
        LOCATION[Location Module]
        VOICE_IOS[Voice Module]
        SCREEN[Screen Module]
        STATUS_IOS[Status Display]
        SETTINGS_IOS[Settings]
    end

    subgraph Android["Android App (Kotlin)"]
        ANDROID_MAIN[Main Activity]
        ANDROID_SVC[Background Service]
    end

    subgraph Shared["Shared (Swift Packages)"]
        KIT[OpenClawKit<br/>Shared framework]
        CHAT_SHARED[OpenClawChatUI<br/>Shared chat components]
        PROTO[OpenClawProtocol<br/>Protocol definitions]
    end

    macOS --> Shared
    iOS --> Shared
    macOS & iOS & Android -->|WebSocket| GW_WS[Gateway WS Server]
```

---

## 4. Workspace Package Dependency Tree

```mermaid
graph TB
    subgraph Root["openclaw (root)"]
        ROOT_PKG["openclaw v2026.2.1<br/>Main CLI + Gateway"]
    end

    subgraph Shims["Compatibility Shims"]
        CLAWDBOT["clawdbot<br/>→ openclaw (workspace:*)"]
        MOLTBOT["moltbot<br/>→ openclaw (workspace:*)"]
    end

    subgraph UI["Web UI"]
        UI_PKG["openclaw-control-ui<br/>(Lit + Vite)"]
    end

    subgraph Extensions["Extensions (30 packages)"]
        subgraph ChannelExt["Channel Extensions (no runtime deps)"]
            E_DC["@openclaw/discord"]
            E_TG["@openclaw/telegram"]
            E_SL["@openclaw/slack"]
            E_SG["@openclaw/signal"]
            E_IM["@openclaw/imessage"]
            E_WA["@openclaw/whatsapp"]
            E_LN["@openclaw/line"]
            E_BB["@openclaw/bluebubbles"]
            E_MT["@openclaw/msteams"]
            E_MX["@openclaw/matrix"]
            E_MM["@openclaw/mattermost"]
            E_GC["@openclaw/googlechat"]
            E_NC["@openclaw/nextcloud-talk"]
            E_TL["@openclaw/tlon"]
            E_TW["@openclaw/twitch"]
            E_ZA["@openclaw/zalo"]
            E_ZU["@openclaw/zalouser"]
            E_NO["@openclaw/nostr"]
            E_LO["@openclaw/lobster"]
        end

        subgraph FeatureExt["Feature Extensions"]
            E_MC["@openclaw/memory-core<br/>(no deps)"]
            E_ML["@openclaw/memory-lancedb<br/>→ @lancedb/lancedb, openai"]
            E_VC["@openclaw/voice-call<br/>→ ws, typebox, zod"]
            E_LT["@openclaw/llm-task<br/>(no deps)"]
            E_OP["@openclaw/open-prose<br/>(no deps)"]
        end

        subgraph AuthExt["Auth Extensions (no runtime deps)"]
            E_CP["@openclaw/copilot-proxy"]
            E_GA["@openclaw/google-antigravity-auth"]
            E_GG["@openclaw/google-gemini-cli-auth"]
            E_MP["@openclaw/minimax-portal-auth"]
            E_QP["@openclaw/qwen-portal-auth"]
        end

        subgraph ObsExt["Observability"]
            E_OT["@openclaw/diagnostics-otel<br/>→ @opentelemetry/* (10 pkgs)"]
        end
    end

    ROOT_PKG -.->|"exports plugin-sdk"| Extensions
    CLAWDBOT -->|"workspace:*"| ROOT_PKG
    MOLTBOT -->|"workspace:*"| ROOT_PKG
    Extensions -.->|"devDependencies"| ROOT_PKG
    UI_PKG -.->|"independent"| ROOT_PKG
```

---

## 5. External Dependency Tree

### 5.1 Core Runtime Dependencies

```
openclaw v2026.2.1
│
├─── AI / Agent Framework
│    ├── @mariozechner/pi-agent-core (0.51.1)
│    ├── @mariozechner/pi-ai (0.51.1)
│    ├── @mariozechner/pi-coding-agent (0.51.1)
│    ├── @mariozechner/pi-tui (0.51.1)
│    └── @agentclientprotocol/sdk (0.13.1)
│
├─── Web / HTTP
│    ├── hono (4.11.7)              — HTTP framework (gateway)
│    ├── express (5.2.1)            — Legacy HTTP support
│    ├── undici (7.20.0)            — HTTP client
│    └── ws (8.19.0)                — WebSocket
│
├─── Chat Platform SDKs
│    ├── grammy (1.39.3)            — Telegram Bot API
│    ├── discord-api-types (0.38.38)— Discord types
│    ├── @slack/bolt (4.6.0)        — Slack framework
│    ├── @slack/web-api (7.13.0)    — Slack Web API
│    ├── @line/bot-sdk (10.6.0)     — LINE Messaging API
│    └── @whiskeysockets/baileys (7.0.0-rc.9) — WhatsApp Web
│
├─── Media Processing
│    ├── sharp (0.34.5)             — Image processing
│    ├── pdfjs-dist (5.4.624)       — PDF parsing
│    ├── @napi-rs/canvas            — Canvas rendering (peer)
│    └── jszip (3.10.1)             — ZIP handling
│
├─── CLI / Terminal UI
│    ├── commander (14.0.3)         — CLI framework
│    ├── chalk (5.6.2)              — Terminal colors
│    ├── @clack/prompts (1.0.0)     — Interactive prompts
│    └── osc-progress (0.3.0)       — Progress bars
│
├─── Schema / Validation
│    ├── @sinclair/typebox (0.34.48)— JSON schema builder
│    └── zod (4.3.6)                — Schema validation
│
├─── Cloud / Providers
│    └── @aws-sdk/client-bedrock (3.981.0) — AWS Bedrock
│
├─── Database
│    └── sqlite-vec (0.1.7-alpha.2) — Vector SQLite
│
├─── Content Processing
│    ├── markdown-it (14.1.0)       — Markdown parser
│    ├── linkedom (0.18.12)         — DOM parser
│    └── yaml (2.8.2)               — YAML parser
│
├─── Runtime
│    ├── jiti (2.6.1)               — Runtime TS loader
│    ├── dotenv (17.2.3)            — Environment vars
│    ├── proper-lockfile (4.1.2)    — File locking
│    └── playwright-core (1.58.1)   — Browser automation
│
└─── Peer Dependencies (optional)
     ├── @napi-rs/canvas (^0.1.89)
     └── node-llama-cpp (3.15.1)    — Local LLM
```

### 5.2 Development Dependencies

```
openclaw (dev)
│
├─── Build
│    ├── tsdown (0.20.1)            — TypeScript bundler
│    ├── rolldown (1.0.0-rc.2)      — Module bundler
│    └── tsx (4.21.0)               — TS executor
│
├─── Type System
│    ├── typescript (5.9.3)
│    ├── @typescript/native-preview (7.0.0-dev)
│    └── @types/* (node, express, ws, ...)
│
├─── Lint / Format
│    ├── oxlint (1.43.0)            — Rust-based linter
│    └── oxfmt (0.28.0)             — Rust-based formatter
│
└─── Testing
     ├── vitest (4.0.18)            — Test framework
     └── @vitest/coverage-v8        — Coverage provider
```

### 5.3 Extension Dependencies

```
Extensions
│
├─── @openclaw/memory-lancedb
│    ├── @lancedb/lancedb (0.23.0)
│    ├── @sinclair/typebox (0.34.48)
│    └── openai (6.17.0)
│
├─── @openclaw/voice-call
│    ├── @sinclair/typebox (0.34.48)
│    ├── ws (8.19.0)
│    └── zod (4.3.6)
│
├─── @openclaw/diagnostics-otel
│    ├── @opentelemetry/api (~1.9.0)
│    ├── @opentelemetry/sdk-trace-base
│    ├── @opentelemetry/sdk-metrics
│    ├── @opentelemetry/sdk-logs
│    ├── @opentelemetry/exporter-trace-otlp-http
│    ├── @opentelemetry/exporter-metrics-otlp-http
│    ├── @opentelemetry/exporter-logs-otlp-http
│    ├── @opentelemetry/resources
│    └── @opentelemetry/semantic-conventions
│
├─── @openclaw/copilot-proxy .......... (no runtime deps)
├─── @openclaw/google-*-auth .......... (no runtime deps)
├─── @openclaw/memory-core ............ (no runtime deps)
├─── @openclaw/llm-task ............... (no runtime deps)
├─── @openclaw/open-prose ............. (no runtime deps)
└─── All 19 channel extensions ........ (no runtime deps)
```

### 5.4 Web UI Dependencies

```
openclaw-control-ui
│
├─── Runtime
│    ├── lit (3.3.2)                — Web components
│    ├── marked (17.0.1)            — Markdown rendering
│    ├── dompurify (3.3.1)          — HTML sanitization
│    └── @noble/ed25519 (3.0.0)     — Cryptography
│
└─── Dev
     ├── vite (7.3.1)              — Build tool
     ├── vitest (4.0.18)           — Testing
     └── playwright (1.58.1)       — Browser testing
```

---

## 6. Internal Module Dependency Map

Shows how `src/` modules depend on each other (simplified, major edges only):

```mermaid
graph TB
    entry[entry.ts] --> cli

    subgraph HighLevel["High-Level Modules"]
        cli[cli/]
        gateway[gateway/]
        commands[commands/]
        tui[tui/]
    end

    subgraph Core["Core Services"]
        agents[agents/]
        channels[channels/]
        routing[routing/]
        plugins[plugins/]
        providers[providers/]
    end

    subgraph ChannelImpl["Channel Implementations"]
        telegram[telegram/]
        discord[discord/]
        slack[slack/]
        signal[signal/]
        imessage[imessage/]
        web_wa[web/ WhatsApp]
        line_ch[line/]
    end

    subgraph Support["Supporting Modules"]
        config[config/]
        infra[infra/]
        media[media/]
        media_u[media-understanding/]
        logging[logging/]
        security[security/]
        sessions[sessions/]
        memory[memory/]
        hooks[hooks/]
        utils[utils/]
        terminal[terminal/]
    end

    %% High-level deps
    cli --> commands
    cli --> gateway
    cli --> agents
    cli --> channels
    cli --> config
    cli --> terminal

    gateway --> agents
    gateway --> channels
    gateway --> plugins
    gateway --> routing
    gateway --> providers
    gateway --> config
    gateway --> media
    gateway --> infra
    gateway --> logging

    commands --> config
    commands --> channels
    commands --> infra

    tui --> agents
    tui --> terminal

    %% Core deps
    agents --> plugins
    agents --> sessions
    agents --> memory
    agents --> providers
    agents --> infra

    channels --> ChannelImpl
    channels --> config
    channels --> routing

    routing --> config
    routing --> sessions

    plugins --> config
    plugins --> infra

    %% Channel impl deps
    telegram --> media
    telegram --> infra
    discord --> media
    discord --> infra
    slack --> media
    slack --> infra
    signal --> media
    signal --> infra
    imessage --> media
    imessage --> infra
    web_wa --> media
    web_wa --> infra
    line_ch --> media
    line_ch --> infra

    %% Support deps
    media --> infra
    media_u --> media
    logging --> config
    hooks --> config
```

---

## 7. Skills Ecosystem

```
skills/ (55 packages)
│
├─── Productivity
│    ├── apple-notes          — Apple Notes integration
│    ├── apple-reminders      — Apple Reminders
│    ├── bear-notes           — Bear app notes
│    ├── obsidian             — Obsidian vault access
│    ├── notion               — Notion API
│    ├── trello               — Trello boards
│    ├── things-mac           — Things 3 (macOS)
│    └── tmux                 — tmux session control
│
├─── Communication
│    ├── discord              — Discord utilities
│    └── slack                — Slack utilities
│
├─── Entertainment
│    ├── spotify-player       — Spotify control
│    ├── songsee              — Song recognition
│    ├── gifgrep              — GIF search
│    └── camsnap              — Camera capture
│
├─── Development
│    ├── coding-agent         — Code generation agent
│    ├── canvas               — Canvas rendering
│    ├── github               — GitHub API
│    └── himalaya             — Email client
│
├─── Services & APIs
│    ├── 1password            — 1Password integration
│    ├── weather              — Weather data
│    ├── goplaces             — Places search
│    ├── food-order           — Food ordering
│    └── healthcheck          — Service monitoring
│
├─── AI / Media
│    ├── openai-image-gen     — DALL-E image generation
│    ├── openai-whisper       — Whisper transcription
│    ├── openai-whisper-api   — Whisper API
│    ├── sherpa-onnx-tts      — Text-to-speech
│    └── video-frames         — Video frame extraction
│
└─── System
     ├── session-logs         — Session log viewer
     ├── model-usage          — Model usage stats
     ├── blogwatcher          — Blog monitoring
     └── bird                 — System integration
```

---

## 8. Build & CI Pipeline

```mermaid
graph LR
    subgraph Dev["Development"]
        CODE[Source Code]
        PNPM[pnpm install]
        DEV[pnpm dev]
    end

    subgraph QualityGate["Quality Gate"]
        TYPECHECK[TypeScript<br/>pnpm build]
        LINT[Oxlint<br/>pnpm check]
        FORMAT[Oxfmt<br/>pnpm check]
        TEST[Vitest<br/>pnpm test]
        COV[Coverage<br/>70% threshold]
    end

    subgraph Build["Build Output"]
        TSDOWN[tsdown bundler]
        DIST[dist/]
        A2UI[Canvas A2UI bundle]
        BUILD_INFO[Build info metadata]
    end

    subgraph Release["Release"]
        NPM[npm publish]
        MAC_PKG[macOS DMG]
        IOS_BLD[iOS Build]
        ANDROID_BLD[Android APK]
    end

    CODE --> PNPM --> DEV
    CODE --> QualityGate
    TYPECHECK & LINT & FORMAT & TEST --> Build
    TEST --> COV
    TSDOWN --> DIST
    Build --> Release
```

---

## 9. Deployment Topology

```mermaid
graph TB
    subgraph Local["Local Machine"]
        CLI_LOCAL[openclaw CLI]
        GW_LOCAL[Gateway Server<br/>port 18789]
        MAC_APP[macOS Menubar App]
    end

    subgraph Remote["Remote / Cloud"]
        VPS[VPS / exe.dev VM]
        FLY[Fly.io Instance]
        RAILWAY[Railway / Northflank]
    end

    subgraph Platforms["Messaging Platforms"]
        TG_API[Telegram API]
        DC_API[Discord API]
        SL_API[Slack API]
        WA_WEB[WhatsApp Web]
        SG_SVC[Signal Service]
        LINE_API[LINE API]
    end

    subgraph LLMs["LLM Providers"]
        CLAUDE[Anthropic Claude]
        GPT[OpenAI GPT]
        GEMINI[Google Gemini]
        BEDROCK[AWS Bedrock]
        LOCAL_LLM[Local LLM<br/>node-llama-cpp]
    end

    subgraph Apps["Mobile / Desktop"]
        IOS_APP[iOS App]
        ANDROID_APP[Android App]
    end

    CLI_LOCAL --> GW_LOCAL
    MAC_APP --> GW_LOCAL
    IOS_APP -->|WebSocket| GW_LOCAL
    ANDROID_APP -->|WebSocket| GW_LOCAL

    GW_LOCAL --> Platforms
    GW_LOCAL --> LLMs

    VPS --> Platforms
    VPS --> LLMs
    FLY --> Platforms
```
