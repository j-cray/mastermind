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

### Phase 3: The Interaction Layer

- Connect to Messaging Apps (**Signal & Matrix**).
- Implement **Voice Note Transcription**.
- Implement the "Supervisor" security layer (Prompt Injection defense).
  - **Vector**: pgvector (integrated in Cloud SQL) for long-term semantic memory.

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
   - **Event Bus**: `tokio::broadcast` channel for internal component communication.
   - **Context Manager**: Manages the sliding window of conversation history and retrieves relevant memories.
   - **Proactivity Engine**: "Level 3" intrusive. Runs background loops checking Calendar/Email/Obsidian to proactively push suggestions (e.g., "Meeting in 10m, here is the doc").
   - **Voice Processor**: Integrates Vertex AI (or local Whisper) to transcribe incoming voice notes from Signal/Matrix.

2. Arcade Tool Runtime:
   - **Google Calendar**: Write access (Scheduling).
   - **GitHub**: Non-destructive actions (Read repos, comment on Issues/PRs).
   - **Gmail** (Optional): If Arcade supports it better than generic IMAP.

3. Custom Rust/Wasm Modules ("The Workshop"):
   - **Generic IMAP Client**: Secure Rust IMAP client for Triage (Archive, Spam, Unsubscribe). Creds stored in GCP Secret Manager.
   - **Obsidian Bridge**: File-system watcher and editor to read daily notes for context and append tasks/journals.
   - **Researcher Module**: A "Deep Search" agent that uses Arcade's browsing tools or Serper.dev to perform multi-step research and compile reports.
   - **Home Assistant Bridge**: Websocket client to connect to a local Home Assistant instance (via VPN/Tailscale) for smart home control.
   - **RSS/News Monitor**: Background service fetching feeds to feed the Proactivity Engine ("New article on Rust 1.85, here is a summary").
   - **Plugin Sandbox**: Usage for specific data logic (e.g., parsing `openproject-sync` CSVs).

4. Supervisor System ("The Cortex"):
   - **Logic**: Uses a lightweight Gemini Flash model with a "Security Auditor" system prompt. Checks:
     - `Safe`: Read-only GitHub, Obsidian Read, Web Search, RSS Fetch.
     - `Sensitive`: Calendar Write, Email Triage (Archive), Obsidian Write, Home Automation (Lights/Climate).
     - `Critical`: Email Delete, Unsubscribe, Financial, Home Automation (Locks/Security), Shell Commands.

5. Communication Interface:
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

### Phase 2: The Action Layer

- Build the **Wasm Plugin System** for local custom logic.
- Integrate **Arcade Tool Execution** for major SaaS (GitHub, Calendar).
- Implement **Generic IMAP** module for heavy email triage.
- Implement **Obsidian Bridge** for local note integration.
- Implement **Web Research** and **Home Assistant** modules.

## 5. Migration (Optional)

- Analyze legacy submodules (`mastermind-desktop`, `mastermind-obsidian`) for feature parity but *do not copy code directly* to maintain Rust/Security strictness.
