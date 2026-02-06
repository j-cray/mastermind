# Project Mastermind - Secure Personal Agent Outline

## 1. Goal

To build a secure, high-capability personal AI agent in **Rust** that rivals "OpenClaw/OpenClaw" in functionality but eliminates its critical security flaws. The agent will be hosted on **Google Cloud Platform (GCP)** and run as a personal service designated for the user.

## 2. Core Philosophy: "Paranoid Autonomy"

Unlike OpenClaw's "maximalist default permissions," this agent operates on a "least privilege" model with progressive trust.

- **Default Deny**: The agent cannot execute high-risk actions (spending money, deleting files, sending broad communications) without explicit confirmation.
- **Secure by Design**: All components are memory-safe (Rust) and credentials are never stored in plaintext.
- **Brokered Credentials**: We use **Arcade** to ensure the LLM never sees the actual OAuth tokens or API keys.
- **Context-Aware Security**: Security decisions are made based on the *context* of the request (e.g., a roadmap update is low risk; a bank transfer is critical).

## 3. Technology Stack

### Infrastructure

- **Language**: Rust (for memory safety, performance, and strong type system).
  - *Crates*: `tokio` (async runtime), `axum` (web server), `reqwest` (http client), `sqlx` (database).
- **Hosting**: Google Cloud Run (serverless container) for the core agent.
- **AI Backend**: Google Vertex AI (Gemini 1.5 Pro for complex reasoning, Flash for quick tasks).
- **Database**:
  - **Relational**: Cloud SQL (PostgreSQL) for transactional state (permissions, audit logs, active sessions).
- **Observability**: **OpenTelemetry** exporting to **Google Cloud Trace** for visualizing agent thought processes and latency.

### Phase 3: The Interaction Layer

- Connect to Messaging Apps (**Signal & Matrix**).
- Implement **Voice Note Transcription**.
- Implement the "Supervisor" security layer (Prompt Injection defense).
  - **Hierarchical Memory System**:
    - **Short-term**: Sliding window (Tokens).
    - **Episodic**: `pgvector` store of past interactions/actions.
    - **Semantic**: Knowledge base (RAG) from Obsidian/Docs.

### Security layer

- **Infrastructure Secrets**: GCP Secret Manager (API keys, service account tokens).
- **User Service Credentials**: **Arcade** (AI Agent Authentication).
  - *Why?* Replaces the insecure "file-based tokens" of OpenClaw and the general-purpose nature of Nango.
  - **Key Features**:
    - **Brokered Credentials**: The agent asks Arcade to execute a tool; Arcade handles the auth and returns the result. The token never leaks to the generic agent logic.
    - **Granular Scopes**: Permissions are minimized per tool.
    - **Human-in-the-Loop**: If a token is expired or missing, Arcade pauses and requests user re-auth via a secure flow.
    - **Audit Trails**: Full log of *who* (which agent session) accessed *what* service.
    - **Crypto**: Local encryption for any cached data using `ring` (AEAD via AES-256-GCM).

### Architecture

1. Core Gateway (Rust - "Cerebrum"):
   - **Role**: **Orchestrator Agent** (ADK Pattern).
   - **Event Bus**: `tokio::broadcast` channel for internal component communication.
   - **Context Manager**: Manages the sliding window of conversation history and retrieves relevant memories.
   - **Iterative Planner**: Implements a "Thought-Plan-Act-Observe" loop for complex multi-step reasoning.
   - **Proactivity Engine**: "Level 3" intrusive. Runs background loops checking Calendar/Email/Obsidian to proactively push suggestions (e.g., "Meeting in 10m, here is the doc").
   - **Voice Processor**: Integrates Vertex AI (or local Whisper) to transcribe incoming voice notes from Signal/Matrix.

2. Arcade Tool Runtime:
   - **Google Calendar**: Write access (Scheduling).
   - **GitHub**: Non-destructive actions (Read repos, comment on Issues/PRs).
   - **Gmail** (Optional): If Arcade supports it better than generic IMAP.

3. Specialist Agents (ADK Pattern):
   - **Researcher Agent**: A dedicated agent with its own system prompt and tools (Arcade Browsing, Serper.dev) to perform multi-step deep research and compile reports.
   - **Coder Agent** (Future): Specialized in writing and executing code via the Interpreter.

4. The Workshop (MCP Tool Layer):
   - **Architecture**: Implements the **Model Context Protocol (MCP)** to expose deterministic functions to Agents.
   - **Generic IMAP MCP**: Secure Rust IMAP tool for Triage.
   - **Obsidian MCP**: File-system watcher and editor exposed as an MCP server.
   - **Home Assistant Bridge**: Websocket client exposed as an MCP tool for smart home control.
   - **RSS/News Monitor**: Background service fetching feeds.
   - **Code Interpreter**: A secure sandbox (Wasm/Firecracker) exposed as a tool for the Coder/Orchestrator Agent.
   - **Plugin Sandbox**: Usage for specific data logic (e.g., parsing `openproject-sync` CSVs).

5. Supervisor System ("The Cortex"):
   - **Logic**: Uses a lightweight Gemini Flash model with a "Security Auditor" system prompt. Checks:
     - `Safe`: Read-only GitHub, Obsidian Read, Web Search, RSS Fetch.
     - `Sensitive`: Calendar Write, Email Triage (Archive), Obsidian Write, Home Automation (Lights/Climate).
     - `Critical`: Email Delete, Unsubscribe, Financial, Home Automation (Locks/Security), Shell Commands.

6. Communication Interface:
   - **Primary**: **Signal** and **Matrix** (via bridges or native libraries).
   - **Dashboard**: Minimal admin UI hidden behind **GCP Identity-Aware Proxy (IAP)**.

## 4. Addressing OpenClaw Security Flaws

| OpenClaw Flaw | Our Solution |
| :--- | :--- |
| **RCE via Malicious Links/Plugins** | **Sandbox Isolation**: Plugins run in Wasm sandboxes. **Arcade Execution**: Risky third-party tools run in Arcade's managed environment, not our server. |
| **Exposed Control Panels** | **Identity-Aware Proxy (IAP)**: The agent's dashboard is hidden behind GCP IAP. No public internet surface area. |
| **Plaintext Credentials** | **Brokered Auth**: GCP Secret Manager for system keys; **Arcade** for user tokens (tokens are never stored locally). |
| **Prompt Injection** | **Dual-LLM Supervisor** + **Arcade Scopes**: A Supervisor LLM checks intent. Arcade enforces strict API scopes, limiting blast radius even if injection succeeds. |
| **Broad Permissions** | **Human-in-the-Loop**: Arcade handles authorization friction. "Active Mode" is temporary and audit-logged. |
| **Unchecked Loops** | **Rate Limiting & Cost Budgets**: Strict limits on API calls and autonomous loops to prevent runaway costs or actions. |

### Phase 2: The Action Layer

- Build the **Wasm Plugin System** for local custom logic.
- Integrate **Arcade Tool Execution** for major SaaS (GitHub, Calendar).
- Implement **Generic IMAP** module for heavy email triage.
- Implement **Obsidian Bridge** for local note integration.
- Implement **Web Research** and **Home Assistant** modules.

## 5. Migration (Optional)

- Analyze legacy submodules (`mastermind-desktop`, `mastermind-obsidian`) for feature parity but *do not copy code directly* to maintain Rust/Security strictness.
