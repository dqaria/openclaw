# 架构概览

本文档提供 OpenClaw 项目的完整架构概览，包括系统级架构图、各模块架构图以及完整的依赖树。

---

## 1. 系统顶层架构

```mermaid
graph TB
    subgraph Users["用户与客户端"]
        TG[Telegram]
        DC[Discord]
        SL[Slack]
        SG[Signal]
        IM[iMessage]
        WA[WhatsApp]
        LN[LINE]
        WEB[Web UI]
        EXT_CH["扩展频道<br/>(Teams, Matrix, Zalo, ...)"]
    end

    subgraph CLI["CLI 层"]
        ENTRY[entry.ts<br/>入口]
        PROGRAM[Commander 程序]
        SUBCMDS["子命令<br/>agent, send, config, status, ..."]
    end

    subgraph Gateway["网关服务器"]
        GW_HTTP[Hono HTTP 服务器]
        GW_WS[WebSocket 服务器]
        GW_RPC[RPC 方法处理器]
        GW_CHAN_MGR[频道管理器]
        GW_STATE[运行时状态]
        GW_DISC[mDNS 发现]
        GW_CRON[定时任务服务]
        GW_MODEL[模型目录]
    end

    subgraph Agents["Agent 运行时"]
        PI_CORE["Pi Agent 核心<br/>(@mariozechner/pi-*)"]
        TOOLS[工具注册表]
        WORKSPACE[Agent 工作区]
        SESSIONS[会话管理器]
        MEMORY["记忆 / 搜索"]
        SANDBOX[沙箱环境]
    end

    subgraph Channels["频道层"]
        CH_REG[频道注册表]
        CH_PLUGINS[频道插件]
        CH_ALLOW["白名单 / 门控"]
        CH_ROUTING["路由 / 绑定"]
    end

    subgraph Plugins["插件系统"]
        PL_DISC[插件发现]
        PL_LOAD[插件加载器]
        PL_REG[插件注册表]
        PL_SDK[插件 SDK]
    end

    subgraph Infra["基础设施"]
        CONFIG[配置管理器]
        MEDIA[媒体管道]
        LOGGING[日志系统]
        SECURITY[安全模块]
        PROVIDERS[LLM 提供方]
        TLS["TLS / 隧道"]
    end

    subgraph Apps["原生应用"]
        MACOS[macOS 菜单栏应用]
        IOS[iOS 应用]
        ANDROID[Android 应用]
    end

    subgraph WebUI["Web 控制台"]
        LIT[Lit 组件]
        VITE[Vite 开发服务器]
    end

    %% 用户连接
    TG & DC & SL & SG & IM & WA & LN --> CH_PLUGINS
    EXT_CH --> CH_PLUGINS
    WEB --> GW_HTTP

    %% CLI 流程
    ENTRY --> PROGRAM --> SUBCMDS
    SUBCMDS --> Gateway
    SUBCMDS --> Channels

    %% 网关编排
    GW_HTTP --> GW_WS --> GW_RPC
    GW_RPC --> GW_CHAN_MGR
    GW_RPC --> Agents
    GW_CHAN_MGR --> CH_PLUGINS
    GW_RPC --> GW_MODEL

    %% 频道流程
    CH_PLUGINS --> CH_REG
    CH_PLUGINS --> CH_ALLOW
    CH_PLUGINS --> CH_ROUTING
    CH_ROUTING --> Agents

    %% Agent 流程
    PI_CORE --> TOOLS
    PI_CORE --> WORKSPACE
    PI_CORE --> SESSIONS
    PI_CORE --> MEMORY
    TOOLS --> SANDBOX

    %% 插件流程
    PL_DISC --> PL_LOAD --> PL_REG
    PL_SDK --> PL_REG
    PL_REG --> CH_PLUGINS
    PL_REG --> TOOLS
    PL_REG --> GW_RPC

    %% 基础设施
    CONFIG --> Gateway
    CONFIG --> Channels
    MEDIA --> Channels
    MEDIA --> Agents
    PROVIDERS --> Agents
    LOGGING --> Gateway

    %% 原生应用
    MACOS --> GW_WS
    IOS --> GW_WS
    ANDROID --> GW_WS

    %% Web UI
    WebUI --> GW_HTTP
```

---

## 2. 消息流转架构

收发消息的核心数据流：

```mermaid
sequenceDiagram
    participant User as 用户 (Telegram/Discord/...)
    participant Channel as 频道监听器
    participant Router as 路由引擎
    participant Agent as Pi Agent
    participant Tools as 工具注册表
    participant LLM as LLM 提供方
    participant Send as 频道发送器

    User->>Channel: 收到消息
    Channel->>Channel: 解析事件，提取文本/媒体
    Channel->>Router: resolveAgentRoute(channel, peer, account)
    Router->>Router: 检查绑定 (peer -> guild -> account -> channel)
    Router-->>Agent: agentId + sessionKey

    Agent->>Agent: 加载会话上下文
    Agent->>LLM: 发送 prompt + 历史
    LLM-->>Agent: 返回响应（可能包含工具调用）

    opt 工具调用
        Agent->>Tools: 执行工具
        Tools-->>Agent: 工具结果
        Agent->>LLM: 发送工具结果
        LLM-->>Agent: 最终响应
    end

    Agent->>Send: sendMessage(channel, peer, response)
    Send->>User: 投递回复
```

---

## 3. 分模块架构图

### 3.1 CLI 模块 (`src/cli/`)

```mermaid
graph TB
    subgraph Entry["入口"]
        ENTRY_TS["entry.ts<br/>Node 重启器"]
        INDEX_TS["index.ts<br/>buildProgram"]
    end

    subgraph Program["程序注册"]
        REG["register.subclis.ts<br/>懒加载子命令"]
        HELP[configureProgramHelp]
        HOOKS[registerPreActionHooks]
    end

    subgraph Commands["命令组"]
        AGENT["agent<br/>TUI / CLI / RPC 模式"]
        MSG["message / send<br/>发送消息"]
        GW_CMD["gateway<br/>run, register, discover"]
        STATUS["status / health<br/>系统状态"]
        CONFIG_CMD["config<br/>读取、设置、编辑"]
        ONBOARD["onboard / setup<br/>初始化向导"]
        CHANNELS["channels<br/>登录、登出、状态"]
        MODELS["models<br/>配置 LLM 模型"]
        PLUGINS_CMD["plugins<br/>安装、列表、移除"]
        SKILLS["skills<br/>列表、启用、禁用"]
        BROWSER["browser<br/>浏览器工具"]
        NODES["nodes<br/>远程节点管理"]
        MEMORY_CMD["memory<br/>记忆搜索"]
        TUI_CMD["tui<br/>终端 UI 聊天"]
    end

    subgraph Deps["依赖注入"]
        CLI_DEPS["createDefaultDeps<br/>CliDeps 工厂"]
        SEND_DEPS[createOutboundSendDeps]
    end

    ENTRY_TS --> INDEX_TS
    INDEX_TS --> Program
    REG --> Commands
    CLI_DEPS --> SEND_DEPS
    Commands --> CLI_DEPS
```

### 3.2 网关模块 (`src/gateway/`)

```mermaid
graph TB
    subgraph Init["初始化"]
        IMPL["server.impl.ts<br/>主编排器"]
        LOAD_CFG[loadConfig]
        LOAD_PLUGINS[loadGatewayPlugins]
        LOAD_MODELS[loadGatewayModelCatalog]
    end

    subgraph Server["HTTP/WS 服务器"]
        HONO[Hono HTTP 应用]
        WS[WebSocket 升级]
        AUTH[设备认证]
        HEALTH[健康检查端点]
    end

    subgraph Methods["RPC 方法"]
        CHAT[chat]
        TOOL_RES[tool_result]
        PROBE[probe]
        CHAN_CTRL["频道控制<br/>start, stop, status"]
        PLUGIN_M[插件贡献的方法]
    end

    subgraph Managers["服务管理器"]
        CHAN_MGR["频道管理器<br/>按账号 start/stop/status"]
        NODE_MGR[节点订阅管理器]
        APPROVAL[执行审批处理器]
        RUNTIME[网关运行时状态]
        SIDECARS["边车服务<br/>Browser, Canvas"]
    end

    subgraph Protocol["协议"]
        ENCODE[协议编码]
        DECODE[协议解码]
        SCHEMA[消息 Schema]
    end

    IMPL --> LOAD_CFG --> LOAD_PLUGINS --> LOAD_MODELS
    IMPL --> Server
    HONO --> WS --> Methods
    Methods --> Managers
    CHAN_MGR --> Protocol
    WS --> Protocol
```

### 3.3 频道层 (`src/channels/` + `src/telegram/`、`src/discord/` 等)

```mermaid
graph TB
    subgraph Registry["频道注册表"]
        REG["registry.ts<br/>CHAT_CHANNEL_ORDER"]
        PLUGIN_TYPE[ChannelPlugin 接口]
    end

    subgraph Shared["共享基础设施"]
        ALLOW["allowlist/<br/>访问控制"]
        GATING["command-gating.ts<br/>mention-gating.ts"]
        ACK[ack-reactions.ts]
        SENDER[sender-identity.ts]
        LABEL[conversation-label.ts]
        PREFIX[reply-prefix.ts]
    end

    subgraph CoreChannels["核心频道实现"]
        TG_MOD["telegram/<br/>monitor, send, probe"]
        DC_MOD["discord/<br/>monitor, send, probe"]
        SL_MOD["slack/<br/>monitor, send, probe"]
        SG_MOD["signal/<br/>monitor, send, probe"]
        IM_MOD["imessage/<br/>monitor, send, probe"]
        WA_MOD["web/ (WhatsApp)<br/>monitor, send, probe"]
        LN_MOD["line/<br/>monitor, send, probe"]
    end

    subgraph ExtChannels["扩展频道插件"]
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

### 3.4 Agent 运行时 (`src/agents/`)

```mermaid
graph TB
    subgraph Core["Agent 核心"]
        PI["Pi Agent Core<br/>@mariozechner/pi-agent-core"]
        PI_AI["Pi AI<br/>@mariozechner/pi-ai"]
        PI_CODING["Pi Coding Agent<br/>@mariozechner/pi-coding-agent"]
    end

    subgraph Runners["Agent 运行器"]
        CLI_RUN["cli-runner/<br/>CLI Agent 执行"]
        EMBED_RUN["pi-embedded-runner/<br/>内嵌运行时"]
        EMBED_HELP["pi-embedded-helpers/<br/>清洗、工具函数"]
        EMBED_SUB["pi-embedded-subscribe/<br/>流处理"]
    end

    subgraph ToolSystem["工具系统"]
        TOOL_REG["tools/<br/>工具定义"]
        TOOL_POLICY["工具策略与审批"]
        BASH_TOOL[Bash 工具执行]
        SKILL_TOOLS[技能贡献的工具]
        PLUGIN_TOOLS[插件贡献的工具]
    end

    subgraph State["状态管理"]
        AUTH_PROF["auth-profiles/<br/>认证配置管理"]
        SESSIONS_MGR[会话管理]
        SCHEMA_VAL["schema/<br/>Schema 校验器"]
        SANDBOX_ENV["sandbox/<br/>沙箱配置"]
    end

    subgraph Skills["技能系统"]
        SKILL_REG["skills/<br/>技能注册表"]
        SKILL_55["55 个技能包<br/>(apple-notes, spotify, github, ...)"]
    end

    Core --> Runners
    Runners --> ToolSystem
    Runners --> State
    ToolSystem --> Skills
    SKILL_REG --> SKILL_55
    PLUGIN_TOOLS --> ToolSystem
```

### 3.5 插件系统 (`src/plugins/`)

```mermaid
graph TB
    subgraph Discovery["发现"]
        DISC["discovery.ts<br/>扫描 extensions/, node_modules"]
        MANIFEST["manifest.ts<br/>解析插件元数据"]
    end

    subgraph Loading["加载"]
        LOADER["loader.ts<br/>加载 + 注册"]
        BUNDLED["bundled-dir/<br/>内置插件"]
    end

    subgraph Registry["注册表"]
        REG["registry.ts<br/>中央注册存储"]
        TOOL_R[工具注册]
        CLI_R[CLI 注册]
        CHAN_R[频道注册]
        PROV_R[提供方注册]
        HOOK_R[钩子注册]
        HTTP_R[HTTP 路由注册]
        GW_R[网关处理器注册]
    end

    subgraph SDK["插件 SDK（导出）"]
        SDK_CHAN["频道类型与适配器"]
        SDK_API[插件 API]
        SDK_GW[网关类型]
        SDK_CFG[配置 Schema]
    end

    subgraph Runtime["运行时"]
        RT["runtime/<br/>插件生命周期"]
        SVC[服务管理]
    end

    DISC --> MANIFEST --> LOADER
    LOADER --> REG
    REG --> TOOL_R & CLI_R & CHAN_R & PROV_R & HOOK_R & HTTP_R & GW_R
    SDK --> LOADER
    REG --> Runtime
```

### 3.6 媒体管道 (`src/media/`)

```mermaid
graph LR
    subgraph Input["输入"]
        FETCH["fetch.ts<br/>URL 下载"]
        PARSE["parse.ts<br/>MIME 检测"]
        INPUT_F["input-files.ts<br/>文件处理"]
    end

    subgraph Processing["处理"]
        AUDIO["audio.ts<br/>编解码、标签"]
        IMAGE["image-ops.ts<br/>Sharp 缩放"]
        OPTIMIZE["优化为 JPEG"]
    end

    subgraph Storage["存储与服务"]
        STORE["store.ts<br/>本地存储"]
        HOST["host.ts<br/>确保托管 URL"]
        SERVER[HTTP 媒体服务器]
    end

    subgraph Understanding["理解"]
        MEDIA_U["media-understanding/<br/>视觉、OCR"]
        LINK_U["link-understanding/<br/>链接预览"]
    end

    subgraph ChannelAdapters["频道适配"]
        TG_M["Telegram: file_id 缓存"]
        DC_M["Discord: 附件 URL"]
        SL_M["Slack: 文件上传"]
        WA_M["WhatsApp: base64"]
        SG_M["Signal: 二进制上传"]
    end

    Input --> Processing --> Storage
    Storage --> ChannelAdapters
    Input --> Understanding
```

### 3.7 路由引擎 (`src/routing/`)

```mermaid
graph TB
    subgraph Input["路由输入"]
        CHAN_ID["频道 ID"]
        ACCOUNT["账号 ID"]
        PEER["Peer / 发送者"]
        GUILD["Guild / Team ID"]
    end

    subgraph Resolution["路由解析"]
        RESOLVE["resolve-route.ts<br/>resolveAgentRoute"]
        BINDINGS["bindings.ts<br/>基于配置的绑定"]
        SESSION_KEY["session-key.ts<br/>构建会话键"]
    end

    subgraph Output["路由输出"]
        AGENT_ID[Agent ID]
        SESS_KEY["会话键"]
        MATCH["匹配来源<br/>binding.peer / binding.guild / default"]
    end

    subgraph Binding["绑定优先级"]
        B1["1. Peer 绑定"]
        B2["2. Guild / Team 绑定"]
        B3["3. Account 绑定"]
        B4["4. Channel 绑定"]
        B5["5. 默认 Agent"]
    end

    Input --> RESOLVE
    RESOLVE --> BINDINGS
    BINDINGS --> Binding
    RESOLVE --> SESSION_KEY
    RESOLVE --> Output
```

### 3.8 基础设施 (`src/infra/`)

```mermaid
graph TB
    subgraph Network["网络"]
        PORTS["ports.ts<br/>端口检测"]
        NET["net/<br/>网络工具"]
        OUTBOUND["outbound/<br/>HTTP 客户端"]
        TLS_MOD["tls/<br/>TLS 管理"]
        TAILSCALE[Tailscale 集成]
        SSH[SSH 隧道]
    end

    subgraph Runtime["运行时"]
        ENV[Shell 环境]
        EXEC[进程执行]
        PATHS[路径管理]
        ERRORS[错误处理]
        UPDATES[更新检查器]
    end

    subgraph Data["数据"]
        ARCHIVE[归档管理]
        MIGRATIONS[状态迁移]
        PROVIDER_USAGE[提供方用量追踪]
    end

    subgraph Config["配置系统"]
        CFG_LOAD["config.ts<br/>加载 JSON5"]
        CFG_SCHEMA[Schema 校验]
        CFG_MIGRATE[旧版迁移]
    end

    Network --> Runtime
    Runtime --> Data
    Config --> Runtime
```

### 3.9 原生应用架构

```mermaid
graph TB
    subgraph macOS["macOS 应用 (Swift/SwiftUI)"]
        MENU[菜单栏应用]
        GW_CTRL[网关控制]
        AUDIO_CAP[音频采集]
        VOICE_WAKE[语音唤醒]
        SETTINGS[设置界面]
        CRON_UI[定时任务界面]
        NOTIF[通知]
        ONBOARD_MAC[引导流程]
    end

    subgraph iOS["iOS 应用 (Swift/SwiftUI)"]
        CHAT_UI[聊天界面]
        CAMERA[相机模块]
        LOCATION[位置模块]
        VOICE_IOS[语音模块]
        SCREEN[屏幕模块]
        STATUS_IOS[状态展示]
        SETTINGS_IOS[设置]
    end

    subgraph Android["Android 应用 (Kotlin)"]
        ANDROID_MAIN[主 Activity]
        ANDROID_SVC[后台服务]
    end

    subgraph Shared["共享层 (Swift Package)"]
        KIT["OpenClawKit<br/>共享框架"]
        CHAT_SHARED["OpenClawChatUI<br/>共享聊天组件"]
        PROTO["OpenClawProtocol<br/>协议定义"]
    end

    macOS --> Shared
    iOS --> Shared
    macOS & iOS & Android -->|WebSocket| GW_WS[网关 WS 服务器]
```

---

## 4. 工作区包依赖树

```mermaid
graph TB
    subgraph Root["openclaw（根包）"]
        ROOT_PKG["openclaw v2026.2.1<br/>主 CLI + 网关"]
    end

    subgraph Shims["兼容性垫片"]
        CLAWDBOT["clawdbot<br/>-> openclaw (workspace:*)"]
        MOLTBOT["moltbot<br/>-> openclaw (workspace:*)"]
    end

    subgraph UI["Web UI"]
        UI_PKG["openclaw-control-ui<br/>(Lit + Vite)"]
    end

    subgraph Extensions["扩展（30 个包）"]
        subgraph ChannelExt["频道扩展（无运行时依赖）"]
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

        subgraph FeatureExt["功能扩展"]
            E_MC["@openclaw/memory-core<br/>（无依赖）"]
            E_ML["@openclaw/memory-lancedb<br/>-> @lancedb/lancedb, openai"]
            E_VC["@openclaw/voice-call<br/>-> ws, typebox, zod"]
            E_LT["@openclaw/llm-task<br/>（无依赖）"]
            E_OP["@openclaw/open-prose<br/>（无依赖）"]
        end

        subgraph AuthExt["认证扩展（无运行时依赖）"]
            E_CP["@openclaw/copilot-proxy"]
            E_GA["@openclaw/google-antigravity-auth"]
            E_GG["@openclaw/google-gemini-cli-auth"]
            E_MP["@openclaw/minimax-portal-auth"]
            E_QP["@openclaw/qwen-portal-auth"]
        end

        subgraph ObsExt["可观测性"]
            E_OT["@openclaw/diagnostics-otel<br/>-> @opentelemetry/* (10 个包)"]
        end
    end

    ROOT_PKG -.->|"导出 plugin-sdk"| Extensions
    CLAWDBOT -->|"workspace:*"| ROOT_PKG
    MOLTBOT -->|"workspace:*"| ROOT_PKG
    Extensions -.->|"devDependencies"| ROOT_PKG
    UI_PKG -.->|"独立"| ROOT_PKG
```

---

## 5. 外部依赖树

### 5.1 核心运行时依赖

```
openclaw v2026.2.1
│
├─── AI / Agent 框架
│    ├── @mariozechner/pi-agent-core (0.51.1)
│    ├── @mariozechner/pi-ai (0.51.1)
│    ├── @mariozechner/pi-coding-agent (0.51.1)
│    ├── @mariozechner/pi-tui (0.51.1)
│    └── @agentclientprotocol/sdk (0.13.1)
│
├─── Web / HTTP
│    ├── hono (4.11.7)              — HTTP 框架（网关）
│    ├── express (5.2.1)            — 旧版 HTTP 支持
│    ├── undici (7.20.0)            — HTTP 客户端
│    └── ws (8.19.0)                — WebSocket
│
├─── 聊天平台 SDK
│    ├── grammy (1.39.3)            — Telegram Bot API
│    ├── discord-api-types (0.38.38)— Discord 类型
│    ├── @slack/bolt (4.6.0)        — Slack 框架
│    ├── @slack/web-api (7.13.0)    — Slack Web API
│    ├── @line/bot-sdk (10.6.0)     — LINE Messaging API
│    └── @whiskeysockets/baileys (7.0.0-rc.9) — WhatsApp Web
│
├─── 媒体处理
│    ├── sharp (0.34.5)             — 图像处理
│    ├── pdfjs-dist (5.4.624)       — PDF 解析
│    ├── @napi-rs/canvas            — Canvas 渲染（peer）
│    └── jszip (3.10.1)             — ZIP 处理
│
├─── CLI / 终端 UI
│    ├── commander (14.0.3)         — CLI 框架
│    ├── chalk (5.6.2)              — 终端颜色
│    ├── @clack/prompts (1.0.0)     — 交互式提示
│    └── osc-progress (0.3.0)       — 进度条
│
├─── Schema / 校验
│    ├── @sinclair/typebox (0.34.48)— JSON Schema 构建器
│    └── zod (4.3.6)                — Schema 校验
│
├─── 云服务 / 提供方
│    └── @aws-sdk/client-bedrock (3.981.0) — AWS Bedrock
│
├─── 数据库
│    └── sqlite-vec (0.1.7-alpha.2) — 向量 SQLite
│
├─── 内容处理
│    ├── markdown-it (14.1.0)       — Markdown 解析器
│    ├── linkedom (0.18.12)         — DOM 解析器
│    └── yaml (2.8.2)               — YAML 解析器
│
├─── 运行时
│    ├── jiti (2.6.1)               — TS 运行时加载器
│    ├── dotenv (17.2.3)            — 环境变量
│    ├── proper-lockfile (4.1.2)    — 文件锁
│    └── playwright-core (1.58.1)   — 浏览器自动化
│
└─── Peer 依赖（可选）
     ├── @napi-rs/canvas (^0.1.89)
     └── node-llama-cpp (3.15.1)    — 本地 LLM
```

### 5.2 开发依赖

```
openclaw（开发）
│
├─── 构建
│    ├── tsdown (0.20.1)            — TypeScript 打包器
│    ├── rolldown (1.0.0-rc.2)      — 模块打包器
│    └── tsx (4.21.0)               — TS 执行器
│
├─── 类型系统
│    ├── typescript (5.9.3)
│    ├── @typescript/native-preview (7.0.0-dev)
│    └── @types/* (node, express, ws, ...)
│
├─── 代码检查 / 格式化
│    ├── oxlint (1.43.0)            — 基于 Rust 的 Linter
│    └── oxfmt (0.28.0)             — 基于 Rust 的 Formatter
│
└─── 测试
     ├── vitest (4.0.18)            — 测试框架
     └── @vitest/coverage-v8        — 覆盖率引擎
```

### 5.3 扩展依赖

```
扩展
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
├─── @openclaw/copilot-proxy .......... （无运行时依赖）
├─── @openclaw/google-*-auth .......... （无运行时依赖）
├─── @openclaw/memory-core ............ （无运行时依赖）
├─── @openclaw/llm-task ............... （无运行时依赖）
├─── @openclaw/open-prose ............. （无运行时依赖）
└─── 全部 19 个频道扩展 ............... （无运行时依赖）
```

### 5.4 Web UI 依赖

```
openclaw-control-ui
│
├─── 运行时
│    ├── lit (3.3.2)                — Web 组件框架
│    ├── marked (17.0.1)            — Markdown 渲染
│    ├── dompurify (3.3.1)          — HTML 消毒
│    └── @noble/ed25519 (3.0.0)     — 密码学
│
└─── 开发
     ├── vite (7.3.1)              — 构建工具
     ├── vitest (4.0.18)           — 测试
     └── playwright (1.58.1)       — 浏览器测试
```

---

## 6. 内部模块依赖关系图

展示 `src/` 各模块之间的依赖关系（简化版，仅保留主要依赖边）：

```mermaid
graph TB
    entry[entry.ts] --> cli

    subgraph HighLevel["上层模块"]
        cli[cli/]
        gateway[gateway/]
        commands[commands/]
        tui[tui/]
    end

    subgraph Core["核心服务"]
        agents[agents/]
        channels[channels/]
        routing[routing/]
        plugins[plugins/]
        providers[providers/]
    end

    subgraph ChannelImpl["频道实现"]
        telegram[telegram/]
        discord[discord/]
        slack[slack/]
        signal[signal/]
        imessage[imessage/]
        web_wa["web/ (WhatsApp)"]
        line_ch[line/]
    end

    subgraph Support["支撑模块"]
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

    %% 上层依赖
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

    %% 核心依赖
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

    %% 频道实现依赖
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

    %% 支撑依赖
    media --> infra
    media_u --> media
    logging --> config
    hooks --> config
```

---

## 7. 技能生态

```
skills/（55 个包）
│
├─── 效率工具
│    ├── apple-notes          — Apple Notes 集成
│    ├── apple-reminders      — Apple Reminders
│    ├── bear-notes           — Bear 笔记应用
│    ├── obsidian             — Obsidian 仓库访问
│    ├── notion               — Notion API
│    ├── trello               — Trello 看板
│    ├── things-mac           — Things 3 (macOS)
│    └── tmux                 — tmux 会话控制
│
├─── 通讯
│    ├── discord              — Discord 工具
│    └── slack                — Slack 工具
│
├─── 娱乐
│    ├── spotify-player       — Spotify 控制
│    ├── songsee              — 歌曲识别
│    ├── gifgrep              — GIF 搜索
│    └── camsnap              — 摄像头抓拍
│
├─── 开发
│    ├── coding-agent         — 代码生成 Agent
│    ├── canvas               — Canvas 渲染
│    ├── github               — GitHub API
│    └── himalaya             — 邮件客户端
│
├─── 服务与 API
│    ├── 1password            — 1Password 集成
│    ├── weather              — 天气数据
│    ├── goplaces             — 地点搜索
│    ├── food-order           — 外卖点餐
│    └── healthcheck          — 服务监控
│
├─── AI / 媒体
│    ├── openai-image-gen     — DALL-E 图像生成
│    ├── openai-whisper       — Whisper 语音转文字
│    ├── openai-whisper-api   — Whisper API
│    ├── sherpa-onnx-tts      — 文字转语音
│    └── video-frames         — 视频帧提取
│
└─── 系统
     ├── session-logs         — 会话日志查看器
     ├── model-usage          — 模型用量统计
     ├── blogwatcher          — 博客监控
     └── bird                 — 系统集成
```

---

## 8. 构建与 CI 流水线

```mermaid
graph LR
    subgraph Dev["开发"]
        CODE[源代码]
        PNPM[pnpm install]
        DEV[pnpm dev]
    end

    subgraph QualityGate["质量门禁"]
        TYPECHECK["TypeScript<br/>pnpm build"]
        LINT["Oxlint<br/>pnpm check"]
        FORMAT["Oxfmt<br/>pnpm check"]
        TEST["Vitest<br/>pnpm test"]
        COV["覆盖率<br/>70% 阈值"]
    end

    subgraph Build["构建产物"]
        TSDOWN[tsdown 打包器]
        DIST[dist/]
        A2UI[Canvas A2UI 包]
        BUILD_INFO[构建信息元数据]
    end

    subgraph Release["发布"]
        NPM[npm publish]
        MAC_PKG[macOS DMG]
        IOS_BLD[iOS 构建]
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

## 9. 部署拓扑

```mermaid
graph TB
    subgraph Local["本地机器"]
        CLI_LOCAL[openclaw CLI]
        GW_LOCAL["网关服务器<br/>端口 18789"]
        MAC_APP[macOS 菜单栏应用]
    end

    subgraph Remote["远程 / 云"]
        VPS["VPS / exe.dev VM"]
        FLY[Fly.io 实例]
        RAILWAY[Railway / Northflank]
    end

    subgraph Platforms["消息平台"]
        TG_API[Telegram API]
        DC_API[Discord API]
        SL_API[Slack API]
        WA_WEB[WhatsApp Web]
        SG_SVC[Signal Service]
        LINE_API[LINE API]
    end

    subgraph LLMs["LLM 提供方"]
        CLAUDE[Anthropic Claude]
        GPT[OpenAI GPT]
        GEMINI[Google Gemini]
        BEDROCK[AWS Bedrock]
        LOCAL_LLM["本地 LLM<br/>node-llama-cpp"]
    end

    subgraph Apps["移动 / 桌面端"]
        IOS_APP[iOS 应用]
        ANDROID_APP[Android 应用]
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

---

## 10. 项目统计概览

| 指标 | 数值 |
|------|------|
| 顶层目录 | 14 个主要目录 |
| src/ 子目录 | 52 个模块 |
| CLI 文件数 | 105 个 |
| 扩展包数 | 30 个 |
| 技能包数 | 55 个 |
| 文档分类 | 24 个 |
| 脚本数量 | 60+ |
| 原生平台 | 3 个（macOS、iOS、Android） |
| 消息频道 | 23+ 个已集成平台 |
| 核心运行时依赖 | 43 个 |
| 开发依赖 | 13 个 |
| 有运行时依赖的扩展 | 3 个（memory-lancedb、voice-call、diagnostics-otel） |
| 无运行时依赖的扩展 | 27 个 |
