# OpenDroid: Comprehensive Technical Architecture, Subsystems & Security Analysis

> **Project:** OpenDroid (`com.opendroid.aiagent` / `com.opendroid.ai`)  
> **Source Repository:** [JMAN730/opendroid](https://github.com/JMAN730/opendroid) (Fork of `yashab-cyber/opendroid`)  
> **Target Platform:** Android SDK 36 (Android 15/16), Min SDK 26 (Android 8.0)  
> **Toolchain:** Kotlin 2.4.0, Gradle 9.7.0, AGP 9.3.1, Java 21 Toolchain  
> **Date of Analysis:** November 1, 2026  

---

## Table of Contents
1. [Executive Summary & Purpose](#1-executive-summary--purpose)
2. [High-Level Architecture & Design Patterns](#2-high-level-architecture--design-patterns)
3. [The Autonomous Agentic Loop](#3-the-autonomous-agentic-loop)
4. [Action Dispatcher & Device Control Systems](#4-action-dispatcher--device-control-systems)
5. [Perception Engine: Accessibility & Multimodal Vision](#5-perception-engine-accessibility--multimodal-vision)
6. [4-Tier Personal Knowledge Graph & Memory Architecture](#6-4-tier-personal-knowledge-graph--memory-architecture)
7. [Hybrid Inference Pipeline & Model Management](#7-hybrid-inference-pipeline--model-management)
8. [Embedded MCP Server & Privileged Execution Layer](#8-embedded-mcp-server--privileged-execution-layer)
9. [Autonomous Habit Mining & Social Media Suites](#9-autonomous-habit-mining--social-media-suites)
10. [Security Architecture & Cryptographic Boundaries](#10-security-architecture--cryptographic-boundaries)
11. [Security Test Suite Verification & Empirical Results](#11-security-test-suite-verification--empirical-results)
12. [Data Persistence & Room Database Schema (v9)](#12-data-persistence--room-database-schema-v9)
13. [User Interface & Presentation Layer](#13-user-interface--presentation-layer)
14. [Build Toolchain, Quality Gates & Compliance](#14-build-toolchain-quality-gates--compliance)

---

## 1. Executive Summary & Purpose

**OpenDroid** is a production-grade, open-source autonomous AI agent designed natively for Android. Unlike conventional mobile conversational assistants that operate merely as text-in/text-out chatbots, OpenDroid is an **action-oriented agentic runtime**.

It runs on-device, interprets complex multi-action natural language commands, constructs directed acyclic graph (DAG) execution plans, interacts directly with Android OS services and third-party applications, dynamically observes and verifies execution outcomes, and self-heals when steps encounter unexpected UI states or runtime failures.

### Core Value Propositions
- **Privacy-First & Local-First:** Sensitive credentials and user identity are encrypted via the hardware-backed Android Keystore (AES-256-GCM). It can operate completely offline via on-device models without sending user data to the cloud.
- **Deep Operating System Integration:** Direct native APIs, UI Intent routing, Accessibility window-tree scraping, synthetic gesture dispatching, and privileged shell execution via **Shizuku / Sui / Root**.
- **Self-Healing Agentic Loop:** Integrates a formal re-evaluation engine that diagnoses failures and dynamically amends pending execution plans in real time.
- **Embedded MCP (Model Context Protocol) Server:** Operates a loopback JSON-RPC 2.0 endpoint enabling desktop IDEs, CLI tools, or external agents to execute actions and manage persistent terminal sessions on the Android phone.

---

## 2. High-Level Architecture & Design Patterns

OpenDroid is organized under strict **Clean Architecture** and **MVVM** principles, ensuring modularity, decoupled responsibilities, and high testability across all system boundaries.

```
                      ┌─────────────────────────────────────────┐
                      │           Presentation Layer            │
                      │  (Jetpack Compose UI, ViewModels, Theme)│
                      └────────────────────┬────────────────────┘
                                           │ StateFlow & UI Events
                                           ▼
                      ┌─────────────────────────────────────────┐
                      │              Domain Layer               │
                      │  (AgentLoop, PlanManager, ReEvalEngine) │
                      └────────────────────┬────────────────────┘
                                           │ Use Cases & Interfaces
                                           ▼
                      ┌─────────────────────────────────────────┐
                      │               Data Layer                │
                      │ (Room Database v9, DataStore, Keystore) │
                      └────────────────────┬────────────────────┘
                                           │ System Drivers & IPC
                                           ▼
                      ┌─────────────────────────────────────────┐
                      │      System Integration & Services      │
                      │(AccessibilityService, MCP, Shizuku, FGS)│
                      └─────────────────────────────────────────┘
```

### Key Architectural Characteristics
1. **Dependency Injection:** Powered by **Dagger-Hilt 2.60.1** with clear scoping (`@Singleton`, `@ViewModelScoped`, `@ServiceScoped`).
2. **Concurrency & Thread Affinity:**
   - `Dispatchers.Main`: UI events, ViewModels, and Accessibility event dispatchers.
   - `Dispatchers.IO`: Database operations, OkHttp network calls, Keystore encryption, file streaming.
   - `Dispatchers.Default`: Plan JSON parsing, DAG dependency validation, hash verification.
   - Jetpack WorkManager: Long-running background model downloads (`dataSync` foreground service).
3. **Reactive State Delivery:** Internal communication relies on Kotlin `StateFlow` and `SharedFlow`. ViewModels expose strictly read-only `StateFlow<UiState>` streams to Compose components.

---

## 3. The Autonomous Agentic Loop

The central coordination engine resides in `AgentLoop.kt`. When a user submits an instruction via voice or text, the system undergoes a multi-phase lifecycle:

```
[User Input]
     │
     ▼
[IntentClassifier / AliasResolver] ──► (Disambiguate contacts, aliases, compound intents)
     │
     ▼
[LLM Planner (Cloud / LiteRT)] ─────► (Decompose goal into JSON Plan DAG with dependencies)
     │
     ▼
[PlanValidator] ───────────────────► (Check registry, inject default params, map aliases)
     │
     ▼
[ActionSequenceExecutor] ──────────► (Evaluate ActionRisk & AutoApprovalPolicy)
     │
     ├─► If Needs Input ───────────► [awaitUserResponse()] ──► (Interactive prompt in chat)
     │
     ├─► If Approved / Low Risk ───► [ActionDispatcher.execute()]
     │
     ▼
[Execution Verification]
     ├─► Success ──────────────────► Log to TaskHistoryEntity & Episodic Memory
     │
     └─► Failure ──────────────────► [ReEvaluationEngine] ──► (CONTINUE / MODIFY / ABANDON)
```

### Components of the Loop
- **`IntentClassifier` & `AliasResolver`:** Extracts entities, classifies query complexity, handles compound triggers ("open YouTube *and* set volume to 80%"), and performs 4-tier contact resolution (exact match, relationship aliases such as "Mom", fuzzy matching, or prompt fallback).
- **`PlanManager` & `PlanValidator`:** Validates generated plans before execution. It ensures data-producing actions (e.g., `GET_WEATHER`, `WEB_SEARCH`) run before consumers, repairs invalid schemas, and tracks unrecognized actions in `UnknownActionEntity`.
- **`AutoApprovalPolicy` & `ActionRisk`:** Actions are classified into `LOW`, `MEDIUM`, `HIGH`, and `CRITICAL`. Depending on user configuration (`OFF`, `YOLO`, or granular grants), low-impact operations run automatically, while high-risk actions (SMS, calls, payments) pause for user confirmation.
- **`ReEvaluationEngine`:** If an action fails (e.g., UI node not found), the failure context and remaining plan steps are passed back to the LLM to synthesize alternative actions or graceful fallbacks.

---

## 4. Action Dispatcher & Device Control Systems

Centralized in `ActionDispatcher.kt`, OpenDroid registers **over 60 modular actions** categorized across 14 operational suites:

| Category | Handlers | Execution Mechanism |
|:---|:---|:---|
| **System** | `TOGGLE_WIFI`, `TOGGLE_BLUETOOTH`, `SET_VOLUME`, `SET_BRIGHTNESS`, `TOGGLE_TORCH`, `TAKE_SCREENSHOT`, `LOCK_DEVICE`, `OPEN_APP` | Android `WifiManager`, `BluetoothAdapter`, `AudioManager`, `Settings.System`, `AccessibilityService` |
| **Communication** | `MAKE_CALL`, `SEND_SMS`, `SEND_WHATSAPP`, `SEND_TELEGRAM`, `DRAFT_EMAIL`, `FIND_CONTACT` | Android Telephony API, SMS Intents, Accessibility Automators |
| **Calendar & Time** | `SET_ALARM`, `SET_TIMER`, `CREATE_EVENT`, `LIST_CALENDAR_EVENTS`, `CREATE_REMINDER`, `TAKE_NOTE` | `AlarmClock` intents, Android `CalendarContract` ContentProvider |
| **Transport** | `GET_DIRECTIONS`, `BOOK_UBER`, `BOOK_OLA`, `CHECK_TRAFFIC` | Geo-URI intents, deep linking into ride-hailing applications |
| **Information** | `WEB_SEARCH`, `GET_WEATHER`, `GET_NEWS`, `TRANSLATE`, `CURRENCY_CONVERT`, `SUMMARIZE_URL`, `FACT_CHECK` | REST APIs, DuckDuckGo, Open-Meteo, OkHttp |
| **Media & Audio** | `PLAY_MUSIC`, `PAUSE_MUSIC`, `NEXT_TRACK`, `PLAY_YOUTUBE`, `TAKE_PHOTO`, `RECORD_VIDEO` | `MediaSession` controls, YouTube deep links, Camera API |
| **Finance** | `SEND_UPI_PAYMENT`, `CHECK_BALANCE`, `SPLIT_BILL` | Android UPI deep link intents (`upi://pay`), embedded calculator |
| **Smart Home** | `CONTROL_GOOGLE_HOME`, `SMART_DEVICE_COMMAND` | Google Home integration intents |
| **Macros** | `RECORD_MACRO`, `PLAY_MACRO`, `LIST_MACROS` | Procedural memory playback via `MacroEntity` |
| **Advanced Control**| `CLICK_COORDINATES`, `CLICK_TEXT`, `TYPE_TEXT`, `SCROLL`, `PRESS_BACK`, `PRESS_HOME`, `SHELL_COMMAND` | Accessibility gesture dispatch, Shizuku/Root shell execution |
| **Notification** | `READ_NOTIFICATIONS`, `DISMISS_NOTIFICATION`, `SEND_AUTO_REPLY` | `NotificationListenerService`, status bar notification hooks |
| **Routine** | `MORNING_BRIEFING`, `WORK_ROUTINE`, `SLEEP_ROUTINE` | Orchestrated composite action sequences |
| **Social Media** | Multi-platform post scheduling, comment management, analytics | Platform-specific adapters & WorkManager schedulers |

### Strongly Typed Result Hierarchy (`ActionResult`)
```kotlin
sealed class ActionResult {
    data class Success(val dataMap: Map<String, String> = emptyMap()) : ActionResult()
    data class Failure(val errorMsg: String, val fallback: String = "") : ActionResult()
    data class NeedsInput(val question: String, val options: List<String> = emptyList()) : ActionResult()
    data class RequiresApproval(val actionName: String, val description: String) : ActionResult()
    data class UnknownAction(val attemptedAction: String, val availableActions: List<String>) : ActionResult()
}
```

---

## 5. Perception Engine: Accessibility & Multimodal Vision

OpenDroid interacts with third-party applications without requiring dedicated developer APIs through dual perception mechanisms:

### 1. Accessibility Automation (`OpenDroidAccessibilityService.kt`)
- **Node Tree Traversal:** Recursively parses active window `AccessibilityNodeInfo` hierarchies to locate interactive elements by text, resource ID, or content description.
- **Synthetic Gestures:** Dispatches programmatic clicks, coordinate taps, long presses, and directional swipes via `GestureDescription`.
- **IME Automation:** Injects text into editable fields and triggers IME actions (`search`, `go`, `send`) via `performImeEnter()`.
- **Dedicated App Automators:**
  - `WhatsAppAutomator`: Navigates chat view, validates input box presence, inputs text, triggers the send button, and verifies the field cleared.
  - `TelegramAutomator`: Supports official Telegram, Telegram Web, Plus Messenger, and NekoX.
  - `GenericAppAutomator`: Robust polling loops with timeout retries for apps with cold-start layout latency.

### 2. Multimodal Vision Engine (`VisionEngine.kt`)
- Captures screen frames via Android **MediaProjection** (Android 11+).
- Transmits screenshots to multimodal models (e.g., Gemini 2.0 Flash) to resolve semantic bounding boxes into direct screen coordinate taps `(X, Y)`.
- Implements **"Read & Remember"**: extracts unstructured text, tables, meeting agendas, and visual content from the screen and directly inserts them into the Knowledge Graph.

---

## 6. 4-Tier Personal Knowledge Graph & Memory Architecture

Memory is partitioned into four distinct tiers to balance latency, privacy, and permanence:

```
┌────────────────────────────────────────────────────────────────────────┐
│                      Personal Knowledge Graph                          │
├────────────────────┬────────────────────┬───────────────────┬──────────┤
│ ⚡ Level 1         │ 🧠 Level 2         │ 📈 Level 3        │ 🔒 Level 4│
│ TEMPORARY          │ LONG_TERM          │ LEARNED_PATTERN   │ SENSITIVE │
│                    │                    │                   │           │
│ • Volatile active  │ • Explicit facts   │ • Inferred habits │ • AES-256 │
│   plan state       │ • User preferences │ • App sequences   │   GCM     │
│ • Ephemeral context│ • Notes & facts    │ • Confidence score│ • Token/PW│
│ • Step results     │ • Room DB backed   │ • Habit mining    │ • Keystore│
└────────────────────┴────────────────────┴───────────────────┴──────────┘
```

- **Working Memory:** In-memory reactive state holding active execution context.
- **Episodic Memory:** Room tables (`TaskHistoryEntity`, `NotificationEntity`) recording historical actions and outcomes.
- **Semantic Memory:** Room table (`MemoryEntity`) holding extracted personal facts and notes.
- **Procedural Memory:** Room table (`MacroEntity`) storing reusable automation macros.
- **Anti-Poisoning Protection:** Failed, aborted, or hallucinated actions are strictly routed to `UnknownActionEntity` and barred from semantic memory.

---

## 7. Hybrid Inference Pipeline & Model Management

OpenDroid supports **13 cloud LLM providers** alongside multiple **on-device local inference engines** with automated fallback routing:

### Supported Providers
- **Cloud Providers:** Google Gemini, Anthropic Claude, OpenAI, Groq, DeepSeek, Mistral AI, OpenRouter, Together AI, Cohere, GitHub Copilot, Opencode-Go.
- **Local On-Device Backends:**
  1. **Google ML Kit GenAI Prompt API:** Connects directly to Google Play Services **AI Core** on supported devices (Pixel 8+, Samsung S24+) for zero-setup, zero-key **Gemini Nano** execution.
  2. **LiteRT-LM (`litertlm-android:0.14.0`):** Embedded runtime for running quantized `.task` and `.litertlm` models (Gemma 4 E2B-it, Qwen 2.5) offline via GPU/NPU delegates.
  3. **Ollama:** Direct local network connectivity (`http://<ip>:11434`) for self-hosted models without API keys.

### Background Model Downloader (`ModelDownloadWorker`)
- Built atop **Jetpack WorkManager** as a foreground `dataSync` service.
- Implements HTTP `Range` chunked resumption for multi-gigabyte models.
- Authenticates against Hugging Face using Keystore-secured tokens.
- Calculates streaming SHA-256 checksums post-download.
- Runs a JNI engine initialization smoke test (`NativeEngine.init`) before marking a model `READY`.

---

## 8. Embedded MCP Server & Privileged Execution Layer

### 1. Embedded MCP Server (`McpServer.kt`)
OpenDroid runs an embedded Model Context Protocol (MCP) server listening on device loopback (`127.0.0.1:8765`), implementing the JSON-RPC 2.0 specification:
- **Authentication:** Protected by bearer token header `x-opendroid-token`.
- **Exposed Tools:**
  - `device_info`: Returns Android SDK level, package details, and privileged backend status.
  - `list_actions` / `execute_action`: Exposes OpenDroid's 60+ native actions to external tools.
  - `run_privileged_command`: Runs commands through Shizuku, Root, or App shell.
  - `terminal_create`, `terminal_write`, `terminal_read`, `terminal_close`: Interactive persistent shell sessions.
  - `mcp_configure`, `mcp_list_configs`, `mcp_remove_config`: Manages remote MCP server endpoints.

### 2. Privileged Command Executor (`PrivilegedCommandExecutor.kt`)
Three-tiered privilege resolution:
1. **Shizuku / Sui:** Leverages ADB-level system privileges via binder delivery without requiring full device root.
2. **Root Shell (`su`):** Falls back to root shell if a `su` binary is present on device.
3. **App Shell:** Standard sandboxed Linux shell fallback.

---

## 9. Autonomous Habit Mining & Social Media Suites

### 1. Habit & Routine Engine (`HabitRoutineEngine.kt`)
- Passively listens to debounced foreground app transitions from the Accessibility Service.
- Clusters user activities into 30-minute time buckets across days of the week.
- Mines recurring sequences (e.g., Slack $\to$ Gmail $\to$ Calendar on weekday mornings).
- Proactively generates one-click automated routine cards and powers customizable morning briefings.

### 2. Autonomous Social Media Suite (`com.opendroid.ai.social`)
- **Supported Platforms:** Discord, Facebook, Instagram, LinkedIn, Telegram, X (Twitter), YouTube.
- **Capabilities:**
  - `SocialCampaignGenerator`: Multi-step campaign creation from high-level marketing goals.
  - `SocialContentComposer`: Tailors length, tone, and hashtags per platform.
  - `SocialCommentReplyEngine` & `SocialInboxClassifier`: Classifies messages/comments and generates contextual replies based on sentiment.
  - `SocialScheduleWorker`: Jetpack WorkManager background queue for publishing scheduled posts.
  - `SocialAuditLogger`: Immutable audit log of every automated social interaction.

---

## 10. Security Architecture & Cryptographic Boundaries

OpenDroid implements a **Defense-in-Depth** security architecture:

```
                       ┌─────────────────────────────────────────┐
                       │       User Input / Notifications        │
                       └────────────────────┬────────────────────┘
                                            │ Untrusted Data Boundary
                                            ▼
                       ┌─────────────────────────────────────────┐
                       │     Input Sanitizer & Redactor          │
                       └────────────────────┬────────────────────┘
                                            │ Sanitized Data
                                            ▼
                       ┌─────────────────────────────────────────┐
                       │      Hardware Android Keystore          │
                       │          (AES-256-GCM)                  │
                       └────────────────────┬────────────────────┘
                                            │
                     ┌──────────────────────┴──────────────────────┐
                     ▼                                             ▼
        [ProviderCredentialStore]                          [UserProfileStore]
        (API Keys, Tokens, ElevenLabs)                     (Name, Date of Birth)
```

1. **Hardware-Backed Encryption at Rest:**
   - Provider API keys and User Profile information use separate, dedicated Android Keystore master keys (`AES/GCM/NoPadding`, 256-bit).
   - Random 96-bit IV generated per secret envelope.
   - Additional Authenticated Data (AAD) bound to the logical credential ID prevents ciphertext substitution attacks.
2. **Isolated Preferences:**
   - Non-secret application configuration is stored in `AppSettingsStore.kt` (isolated from secrets and excluded from cloud/device backups via `data_extraction_rules.xml`).
3. **Transport Security:**
   - `network_security_config.xml` enforces `<base-config cleartextTrafficPermitted="false"/>`. Unencrypted HTTP is blocked across the app (with an explicit exemption limited to loopback/emulator IPs for local Ollama instances).
4. **PII Redaction in Diagnostics:**
   - `CrashLogRedactor.kt` automatically scrubs API keys (`sk-...`, `hf_...`), authorization headers, phone numbers, and email addresses from stack traces prior to storage or export.

---

## 11. Security Test Suite Verification & Empirical Results

The project's security test suite was executed via Gradle 9.7.0 with JDK 21 toolchain (`./gradlew testDebugUnitTest`):

### Summary
- **Total Security Tests:** **82**
- **Passed:** **82 (100%)**
- **Failures / Errors:** **0**
- **Skipped:** **0**
- **Execution Time:** **11.32s**

### Detailed Suite Results

| Test Class | Tests | Status | Invariants Verified |
|:---|:---:|:---:|:---|
| `ProviderCredentialStoreTest` | 12 | **PASS** | AES-256-GCM envelope serialization, random 96-bit IV, AAD binding, corrupted ciphertext detection, safe key deletion. |
| `LegacySecurePreferencesRetirementTest` | 25 | **PASS** | Zero plaintext leaks during migration, deletion of legacy shared-prefs files, strict separation of secret vs non-secret settings. |
| `CrashLogRedactorTest` | 18 | **PASS** | Redaction of OpenAI, Anthropic, Gemini, Groq, and Hugging Face tokens; sanitization of email addresses, phone numbers, and bearer headers. |
| `AutoApprovalPolicyTest` | 13 | **PASS** | Action risk tiering (`LOW`, `MEDIUM`, `HIGH`, `CRITICAL`), user auto-approval boundaries, permission gate verification. |
| `NeverAutoApproveTest` | 3 | **PASS** | **Hard non-bypassable guard:** Monetary payments, device wiping, and root shell execution can never be auto-approved even in YOLO mode. |
| `ActionRiskPolicyTest` | 3 | **PASS** | Prohibits automated provider fallback for high-risk actions to prevent cross-provider data leakage. |
| `OkHttpNetworkStackTest` | 6 | **PASS** | Enforces TLS encryption, validates mock certificates via OkHttp TLS stack, rejects cleartext HTTP. |
| `McpConfigStoreTest` | 2 | **PASS** | Embedded MCP server token verification (`x-opendroid-token`), header sanitization, endpoint isolation. |

---

## 12. Data Persistence & Room Database Schema (v9)

The local SQLite database (`OpenDroidDatabase.kt`) is at **Schema Version 9** with 20 entities:

```
OpenDroidDatabase (v9)
├── ConversationEntity           (Chat messages, token counts, latency metrics, session links)
├── ChatSessionEntity            (Multi-thread conversation management)
├── PlanEntity                   (Persisted DAG plans, step execution states, error details)
├── MemoryEntity                 (Knowledge graph nodes, memory tiers, confidence ratings)
├── TaskHistoryEntity            (Audit log of executed actions, parameters, outcomes)
├── MacroEntity                  (User-recorded and built-in automation routines)
├── UnknownActionEntity          (Telemetry for unmapped actions requested by LLMs)
├── NotificationEntity           (Intercepted notifications, auto-reply flags)
├── ModelEntity                  (Downloaded LiteRT models, file sizes, SHA-256 checksums)
├── CrashLogEntity               (Redacted on-device crash logs and diagnostic dumps)
├── HabitEventEntity             (Raw foreground app switch telemetry)
├── HabitRoutineEntity           (Discovered candidate recurring app sequences)
└── Social Entities              (8 tables: Accounts, Posts, Comments, Interactions, 
                                  AnalyticsSnapshots, Campaigns, AutomationRules, AuditLogs)
```

---

## 13. User Interface & Presentation Layer

Built with modern **Jetpack Compose** (`compose-bom:2026.06.01`):
- **Cyberpunk / Glassmorphic Theme:**
  - Background: Deep Navy (`#080C10`)
  - Accent / Glow: Neon Green (`#00FF88`)
  - Translucent surfaces with frosted glass borders
- **16+ Native Compose Screens:**
  - `ChatScreen`: Conversation timeline, pulsing audio orb during voice listening, plan step cards.
  - `PlanScreen`: Interactive visual DAG tree of execution plans with live step progress.
  - `MemoryScreen`: Visual explorer for the 4-tier Personal Knowledge Graph.
  - `SocialScreen`: Tabbed dashboard for social media automation (Overview, Accounts, Campaigns, Content, Inbox, Analytics, Reports, Settings).
  - `MacrosScreen`: Macro recording and playback management.
  - `BenchmarkScreen`: Live real-time latency benchmark comparisons across all 13 LLM providers.
  - `SettingsScreen`: Provider configuration, Keystore token inputs, and LiteRT model download manager.
  - `AutoReplySettingsScreen`, `NotificationHistoryScreen`, `PermissionsScreen`, `CrashLogScreen`, `RoutinesScreen`, `OnboardingScreen`, `AboutScreen`, etc.
- **Overlay UI:** Floating accessibility widget (`FloatingWidgetView`) and animated touch target indicators (`TouchTargetView`) showing real-time automation clicks on device.

---

## 14. Build Toolchain, Quality Gates & Compliance

- **Android SDK:** Compile SDK 36, Target SDK 36, Min SDK 26.
- **Toolchain:** Kotlin 2.4.0, Gradle 9.7.0, AGP 9.3.1, Java 21 (`jvmToolchain(21)`).
- **Three-Tier Lint Gate:**
  - `abortOnError = true`, `checkReleaseBuilds = true`, `warningsAsErrors = true`.
  - Frozen, shrink-only `lint-baseline.xml` for legacy entries (never auto-regenerated).
  - Zero tolerance for lint warnings on new code.
- **Release Signing Guard:** Release builds fail loudly if signing credentials or keystores are missing, preventing accidental distribution of debug-signed artifacts.
- **Hardware Flexibility:** Telephony and camera features are flagged as non-mandatory in `AndroidManifest.xml` (`android:required="false"`), allowing OpenDroid to operate seamlessly on ChromeOS, tablets, and foldables.
