# Brainrot IDE / Relay Engine - Architecture Diagram

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           BRAINROT IDE / RELAY ENGINE                             │
│                         (VS Code OSS + Multi-Agent System)                        │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                    ┌───────────────────┴───────────────────┐
                    │                                       │
        ┌───────────▼───────────┐               ┌───────────▼───────────┐
        │   Electron Main      │               │   Workbench Process   │
        │   Process             │               │   (Renderer)          │
        │                       │               │                       │
        │ • Window Management   │◄──────────────►│ • Monaco Editor       │
        │ • IPC Communication   │               │ • UI Components       │
        │ • Session Persistence │               │ • Extension Host      │
        │ • Crash Reporting     │               │ • Webview Panels      │
        └───────────┬───────────┘               └───────────┬───────────┘
                    │                                       │
                    │ IPC                                   │ IPC
                    │                                       │
        ┌───────────▼───────────────────────────────────────▼───────────┐
        │                    AGENT HOST PROCESS                          │
        │                  (Node.js - Separate Process)                   │
        │                                                                  │
        │  ┌─────────────────────────────────────────────────────────┐  │
        │  │              AgentService (Orchestrator)                 │  │
        │  │  • Session ↔ Agent mapping                               │  │
        │  │  • Chat lifecycle management                             │  │
        │  │  • Peer chat catalog persistence                          │  │
        │  │  • Restore flow coordination                              │  │
        │  └────────────────────┬────────────────────────────────────┘  │
        │                       │                                         │
        │  ┌────────────────────▼────────────────────────────────────┐  │
        │  │         AgentHostStateManager (State Management)         │  │
        │  │  • _sessionStates: Map<string, ISessionEntry>            │  │
        │  │  • _chatStates: Map<string, ChatState>                   │  │
        │  │  • _chatProviderData: Map<string, string>                │  │
        │  │  • addChat/restoreChat/removeChat                        │  │
        │  └────────────────────┬────────────────────────────────────┘  │
        │                       │                                         │
        │  ┌────────────────────▼────────────────────────────────────┐  │
        │  │              Agent Harnesses (IAgent)                    │  │
        │  │                                                          │  │
        │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │  │
        │  │  │ ClaudeAgent  │  │ CopilotAgent │  │  CodexAgent  │  │  │
        │  │  │              │  │              │  │              │  │  │
        │  │  │ • Multi-chat │  │ • Multi-chat │  │ • Single-chat│  │  │
        │  │  │ • Fork support│ │ • Fork support│ │ • No fork   │  │  │
        │  │  │ • Subagents  │  │ • Subagents  │  │ • Basic only │  │  │
        │  │  └──────────────┘  └──────────────┘  └──────────────┘  │  │
        │  └──────────────────────────────────────────────────────────┘  │
        │                                                                  │
        │  ┌─────────────────────────────────────────────────────────┐  │
        │  │              MCP Integration Layer                       │  │
        │  │  • Model Context Protocol servers                        │  │
        │  │  • Tool routing and execution                           │  │
        │  │  • Resource management                                  │  │
        │  └─────────────────────────────────────────────────────────┘  │
        └──────────────────────────────────────────────────────────────────┘
                    │
                    │ HTTP/HTTPS
                    │
        ┌───────────▼───────────────────────────────────────────────────┐
        │                    EXTERNAL AI SERVICES                       │
        │                                                               │
        │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
        │  │ Anthropic    │  │ OpenAI       │  │ GitHub       │        │
        │  │ Claude API   │  │ Codex API    │  │ Copilot API  │        │
        │  └──────────────┘  └──────────────┘  └──────────────┘        │
        └───────────────────────────────────────────────────────────────┘
```

## Multi-Chat Architecture Detail

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        MULTI-CHAT SYSTEM                                │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ UI Layer (Sessions Window)                                               │
│                                                                          │
│ • ISessionsManagementService                                            │
│ • SessionCapabilities (context keys)                                     │
│ • "Add Chat" / "Fork" actions (gated by capabilities)                    │
└────────────────────────────┬────────────────────────────────────────────┘
                             │ IPC (agentHost channel)
                             │ createChat / disposeChat / dispatchAction
┌────────────────────────────▼────────────────────────────────────────────┐
│ Orchestrator Layer (Agent Host Process)                                  │
│                                                                          │
│  AgentService:                                                           │
│  • (session, chat) → (agent, session URI, chat URI) mapping              │
│  • _providers, _sessionToProvider maps                                  │
│  • Dispatches user-driven chat lifecycle                                 │
│  • Persists peer-chat catalog (PEER_CHATS_METADATA_KEY)                 │
│  • Routes harness-spawned chats into catalog                            │
│  • Owns restore flow                                                    │
│                                                                          │
│  AgentHostStateManager:                                                  │
│  • _sessionStates: Map<string, ISessionEntry>                           │
│  • _chatStates: Map<string, ChatState>                                  │
│  • _chatProviderData: Map<string, string> (opaque blobs)                │
│  • _ensureDefaultChat (creates default chat deterministically)          │
│  • addChat/restoreChat/removeChat (single catalog path)                 │
└────────────────────────────┬────────────────────────────────────────────┘
                             │ IAgentChats.* interface
                             │ createChat / fork / sendMessage / getMessages
┌────────────────────────────▼────────────────────────────────────────────┐
│ Agent Harness Layer (IAgent implementations)                            │
│                                                                          │
│  ClaudeAgent:                                                            │
│  • _sessions: DisposableMap<id, ClaudeSessionEntry>                      │
│  • ClaudeSessionEntry extends AgentSessionEntry                          │
│  • _chats: DisposableMap<chatUri, ClaudeAgentSession> (all chats)        │
│  • Each peer chat backed by fresh SDK session                            │
│  • Capabilities: multipleChats: { fork: true }                          │
│                                                                          │
│  CopilotAgent:                                                           │
│  • _sessions: DisposableMap<id, CopilotSessionEntry>                    │
│  • CopilotSessionEntry extends AgentSessionEntry                          │
│  • _chats: DisposableMap<chatUri, CopilotAgentSession> (all chats)       │
│  • _chatBackings: Map<chatUri, IPersistedChat> (in-memory cache)         │
│  • Capabilities: multipleChats: { fork: true }                          │
│                                                                          │
│  CodexAgent:                                                             │
│  • _sessions: Map<id, ICodexSession> (single-chat only)                 │
│  • No peer-chat support                                                  │
│  • Capabilities: (no multipleChats = unsupported)                       │
└─────────────────────────────────────────────────────────────────────────┘
```

## Data Flow: User-Driven Add Chat

```
┌──────────┐
│ Sessions  │
│    UI     │
└─────┬────┘
      │ createChat(session, chatUri, options?)
      ▼
┌───────────────────┐
│  AgentService     │
│  _findProviderFor │
│  Session(session) │
└─────┬─────────────┘
      │ chats.createChat(chatUri, convOptions)
      ▼
┌───────────────────┐
│  IAgent.chats     │
│  (Claude/Copilot) │
└─────┬─────────────┘
      │ IAgentCreateChatResult { providerData?, backingSession? }
      ▼
┌───────────────────┐
│  AgentService     │
│  addChat(session, │
│  chatUri,         │
│  {providerData})  │
└─────┬─────────────┘
      │
      ▼
┌───────────────────┐
│ AgentHostState    │
│ Manager.addChat   │
└─────┬─────────────┘
      │ ActionEnvelope (SessionChatAdded)
      ▼
┌──────────┐
│ Sessions  │
│    UI     │
└──────────┘

      │
      │ (parallel)
      ▼
┌───────────────────┐
│  AgentService     │
│  _persistPeerChat │
│  (enqueues RMW of │
│  PEER_CHATS_      │
│  METADATA_KEY)    │
└───────────────────┘

      │ (if backingSession set)
      ▼
┌───────────────────┐
│  AgentService     │
│  _markPeerChat    │
│  Backing          │
│  (writes marker   │
│  to suppress from │
│  listSessions)    │
└───────────────────┘
```

## Data Flow: Harness-Spawned Chat (Subagent)

```
┌──────────┐
│ Agent    │
│ SDK      │
└─────┬────┘
      │ subagent_started signal
      ▼
┌───────────────────┐
│  IAgent           │
│  onDidSession     │
│  Progress         │
└─────┬─────────────┘
      │ AgentSignal {kind: 'subagent_started'}
      ▼
┌───────────────────┐
│  AgentService     │
│  _onChatSpawned   │
│  (registered      │
│  BEFORE Agent     │
│  SideEffects)     │
└─────┬─────────────┘
      │ addChat(session, chat, {origin: {kind:Tool, toolCallId}})
      ▼
┌───────────────────┐
│ AgentHostState    │
│ Manager.addChat   │
└─────┬─────────────┘
      │ ChatSummary
      ▼
┌───────────────────┐
│ AgentSideEffects  │
│ (fires next, chat │
│  already in       │
│  catalog)         │
└─────┬─────────────┘
      │ dispatch turn lifecycle actions
      ▼
┌───────────────────┐
│ AgentHostState    │
│ Manager           │
│                  │
│ Note: Spawned     │
│ chats are NOT     │
│ persisted to      │
│ PEER_CHATS_       │
│ METADATA_KEY      │
│ (transient,       │
│ re-derived from    │
│ event log on      │
│ restore)          │
└───────────────────┘
```

## Key Architecture Invariants

1. **providerData is opaque** - Never parsed by orchestrator, round-tripped verbatim
2. **sessionUri and chatChannelUri are never overloaded** - Distinct schemes
3. **Default chat's backing SDK session IS the session** - Derived deterministically
4. **Single catalog path** - Both user-driven and harness-spawned chats use addChat
5. **Orchestrator peer-chat catalog is restore source of truth** - With one-time legacy migration
6. **_findProviderForSession not _sessionToProvider** - Falls back to URI scheme for restored sessions
7. **Peer chat's backing SDK session must never surface as top-level session** - Marked with peerChatBacking

## Component Directory Structure

```
Relay/
├── src/
│   ├── main.ts                          # Electron main process entry point
│   ├── cli.ts                           # CLI entry point
│   ├── vs/
│   │   ├── base/                        # Common utilities (browser, common, node, parts)
│   │   ├── code/                        # VS Code core (browser, electron-main, node)
│   │   ├── editor/                      # Monaco editor components
│   │   ├── platform/                    # Platform services
│   │   │   ├── agentHost/               # Multi-agent orchestrator
│   │   │   │   ├── common/              # Shared interfaces (IAgent, IAgentService)
│   │   │   │   ├── node/                # Node.js agent host process
│   │   │   │   │   ├── claude/          # Claude agent implementation
│   │   │   │   │   ├── copilot/         # Copilot agent implementation
│   │   │   │   │   ├── codex/           # Codex agent implementation
│   │   │   │   │   ├── agentService.ts  # Orchestrator service
│   │   │   │   │   └── agentHostStateManager.ts
│   │   │   │   └── browser/             # Browser-side agent host
│   │   │   ├── mcp/                     # Model Context Protocol integration
│   │   │   └── [other platform services]
│   │   ├── workbench/                   # VS Code workbench
│   │   │   ├── contrib/                 # Workbench contributions
│   │   │   │   ├── chat/                # Chat UI and services
│   │   │   │   ├── mcp/                 # MCP UI integration
│   │   │   │   ├── agentsVoice/         # Voice agent features
│   │   │   │   ├── terminal/            # Terminal integration
│   │   │   │   └── [other contributions]
│   │   │   ├── api/                     # Workbench API
│   │   │   ├── browser/                 # Browser workbench
│   │   │   ├── common/                  # Common workbench
│   │   │   ├── services/                # Workbench services
│   │   │   └── workbench.*.main.ts      # Workbench entry points
│   │   └── [other VS Code modules]
│   └── [bootstrap files]
├── extensions/                          # Bundled VS Code extensions
│   ├── copilot/                         # GitHub Copilot extension
│   ├── [language extensions]
│   └── [theme extensions]
├── build/                              # Build system
├── scripts/                            # Utility scripts
└── package.json                         # Dependencies and scripts
```

## Technology Stack

- **Framework**: Electron (Chromium + Node.js)
- **Editor**: Monaco Editor (VS Code's editor component)
- **Language**: TypeScript
- **AI SDKs**:
  - @anthropic-ai/sdk (Claude)
  - @openai/codex (Codex)
  - @github/copilot (Copilot)
- **Build Tools**: Gulp, esbuild, TypeScript
- **Testing**: Mocha, Playwright

## Key Design Principles

1. **"Represent, don't orchestrate"** - Agent harness creates/drives SDK chats; orchestrator records and routes
2. **Composition over inheritance** - All harnesses share membership/persistence/restore paths
3. **Single catalog path** - All chats enter catalog through addChat
4. **Capability-based UI** - Features gated by AgentCapabilities, not provider IDs
5. **Opaque provider data** - Orchestrator never parses agent-specific blobs

## Brainrot Social Feed Features

### Supported Platforms

#### YouTube Shorts
- **Embedding**: Converts `/shorts/VIDEOID` to `/embed/VIDEOID` format for iframe embedding
- **Login**: Optional Google OAuth for personalized recommendations
- **Controls**: Auto-play next, manual scroll buttons, pause/resume
- **Compliance**: Official YouTube embed player with privacy-enhanced mode (youtube-nocookie.com)
- **Restrictions**: Age-restricted videos cannot play in embeds
- **Session**: Persistent cookies via Electron session, token refresh

#### Instagram Reels
- **Embedding**: Instagram oEmbed API or embedded webview for full Reels experience
- **Login**: OAuth flow via Facebook app (Creator/Business account required for Graph API)
- **Controls**: Swipe gestures, "Next Reel" buttons, auto-scroll simulation
- **Compliance**: oEmbed for frontend display only (no data extraction/scraping)
- **Session**: Persistent webview session, secure token storage
- **Note**: Instagram Basic Display API retired Dec 2024 - requires Creator/Business account

#### X (Twitter)
- **Embedding**: Twitter oEmbed API for tweets and timeline embeds
- **Login**: OAuth 2.0 flow with persistent session
- **Controls**: Timeline scroll, tweet interactions (like, retweet via webview)
- **Content**: Home timeline, search results, trending topics
- **Compliance**: Official Twitter embed widgets, follows X API Terms of Service
- **Session**: OAuth tokens stored securely, automatic refresh

#### Hinge
- **Embedding**: Embedded webview for full Hinge app experience
- **Login**: OAuth flow via Hinge's authentication system
- **Controls**: Profile browsing, like/pass gestures, message viewing
- **Content**: Discover feed, matches, conversations
- **Compliance**: User-driven webview only (no automation/scraping)
- **Session**: Persistent webview session, secure credential storage
- **Note**: Dating app integration - requires careful privacy handling

### Unified Feed Management

#### Feed Controller
- **Activation**: Feeds auto-activate during AI generation, pause on completion
- **Layout**: Tabbed interface or split-panel view for multiple platforms
- **Priority**: User can set preferred feed or rotate through all
- **Volume Control**: Mute/unmute per feed, global mute during focus mode
- **Time Tracking**: Entertainment time logged per platform (displayed in sidebar)

#### Session Management
- **Unified Login**: Single setup flow for all platforms (OAuth + webview)
- **Credential Storage**: OS keychain integration (keytar) for tokens
- **Session Persistence**: Electron defaultSession for cookies, survives restarts
- **Privacy Mode**: Option to clear sessions on exit, incognito mode
- **Multi-Account**: Support for multiple accounts per platform

### UX Integration

#### Brainrot Mode
- **Trigger**: Activates when AI agent starts processing
- **Behavior**: Opens feed panels, starts auto-scrolling content
- **Indicator**: Progress bar showing "Watching [platform] while we code…"
- **Smart Selection**: Learns user preferences for platform/content timing
- **Emergency Stop**: Manual override to pause feeds immediately

#### Completion Flow
- **Detection**: Listens for agent completion signal
- **Action**: Pauses all feeds, minimizes panels
- **Celebration**: Confetti animation, "Code ready! 🎉" message
- **Transition**: Smooth focus back to code editor with generated results
- **Stats Update**: Updates entertainment time, unlocks achievements

#### Achievement System
- **Time-Based**: "30min Brainrot Master", "Hour of Doomscrolling"
- **Platform-Specific**: "Reels Addict", "Shorts Scholar", "X Expert", "Hinge Pro"
- **Milestones**: "First Generated Code", "10th Session", "Token Saver"
- **Mini-Games**: Unlock Snake/Flappy Bird on major achievements
- **Leaderboard**: Optional comparison with other users (opt-in)

### Technical Implementation

#### Webview Architecture
- **Isolation**: Each platform in separate `<webview>` or BrowserView
- **Communication**: IPC for controls (scroll, pause, next)
- **Performance**: Lazy loading, memory management for multiple webviews
- **Security**: Sandboxed webviews, node context disabled
- **Fallback**: Graceful degradation if platform unavailable

#### Plugin System
- **Platform Plugins**: `plugins/brainrot/youtube`, `plugins/brainrot/instagram`, etc.
- **Unified Interface**: All platforms implement `IBrainrotFeed` interface
- **Extensibility**: Easy to add new platforms (TikTok, Reddit, etc.)
- **Configuration**: Per-platform settings in user config
- **Updates**: Auto-update feed plugins independently of core

### Privacy & Compliance

#### Data Handling
- **No Collection**: User content never sent to third parties beyond platform APIs
- **Local Storage**: All sessions stored locally, encrypted
- **Opt-In Telemetry**: Optional anonymous usage statistics
- **GDPR Compliance**: No unconsented data storage, right to deletion
- **Age Restrictions**: Respects platform age requirements

#### Platform Terms
- **YouTube**: Follows YouTube API Terms of Service and embed policies
- **Instagram**: Uses oEmbed only for display, no scraping (per Instagram policies)
- **X (Twitter)**: Complies with X API Terms and Developer Agreement
- **Hinge**: User-driven webview only, respects Hinge's terms and privacy policy
- **General**: Personal use only, no commercial automation

#### Security
- **Credential Protection**: OS keychain for all tokens/passwords
- **Encryption**: At-rest encryption for sensitive data
- **Network Security**: HTTPS only, certificate pinning where possible
- **Sandboxing**: Webviews isolated from main process
- **Audit**: Regular security audits of feed integrations
