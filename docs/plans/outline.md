# Project Mastermind - Secure Personal Agent Outline

## 1. Goal

To build a secure, high-capability personal AI agent in **Rust** that rivals "OpenClaw/OpenClaw" in functionality but eliminates its critical security flaws. The agent will be hosted on **Google Cloud Platform (GCP)** and run as a personal service designated for the user.

## 2. Core Philosophy: "Paranoid Autonomy"

Unlike OpenClaw's "maximalist default permissions," this agent operates on a "least privilege" model with progressive trust.

- **Default Deny**: The agent cannot execute high-risk actions (spending money, deleting files, sending broad communications) without explicit confirmation.
- **Secure by Design**: All components are memory-safe (Rust) and credentials are never stored in plaintext.

## 3. Technology Stack

### Infrastructure

- **Language**: Rust (for memory safety and performance).
- **Hosting**: Google Cloud Run (serverless container) or GCE (hardened VM).
- **AI Backend**: Google Vertex AI (Gemini Pro/Flash models).
- **Database**:
  - **State**: Cloud SQL (PostgreSQL) or Firestore.
  - **Vector Store**: Vertex AI Vector Search or pgvector.

### Security layer

- **Infrastructure Secrets**: GCP Secret Manager (API keys, service account tokens).
- **User Service Credentials**: **Nango** (Self-Hosted/OSS version) or a custom Rust-based encrypted vault.
  - *Why?* Replaces the insecure "file-based tokens" of OpenClaw. Nango handles OAuth token refresh loops securely.

### Architecture

1. **Core Gateway (Rust)**: The main brain. Receives inputs, decides actions, and delegates.
2. **Plugin Sandbox**:
   - OpenClaw Flaw: Plugins ran as the user with full OS access.
   - **Our Fix**: Plugins run in **WebAssembly (Wasm)** environments (e.g., using `wasmtime`) or strictly isolated containers. They have *no* direct IO access unless explicitly granted via capability handles.
3. **Communication Interface**:
   - Secure Webhook / Bot API (Telegram/Signal/Matrix).
   - *No exposed unauthenticated web panels.* All admin actions require strict Mutual TLS (mTLS) or OAuth2/OIDC authentication.

## 4. Addressing OpenClaw Security Flaws

| OpenClaw Flaw | Our Solution |
| :--- | :--- |
| **RCE via Malicious Links/Plugins** | **Sandbox Isolation**: Plugins run in Wasm sandboxes with zero network/file access by default. 1-Click RCE impossible due to strict input sanitization and deep verification. |
| **Exposed Control Panels** | **Identity-Aware Proxy (IAP)**: The agent's dashboard is hidden behind GCP IAP. No public internet surface area for the control panel. |
| **Plaintext Credentials** | **Double-Encryption**: GCP Secret Manager for system keys; Encrypted Database (AES-256-GCM) + Nango for third-party user tokens. |
| **Prompt Injection** | **Dual-LLM Supervisor Pattern**: A secondary "Supervisor" LLM (or rigorous logic layer) checks outgoing tool calls for unusual patterns before execution. |
| **Broad Permissions** | **Scoped Capabilities**: "Read-Only" mode by default. "Active Mode" requires time-limited user approval (e.g., "Grant write access for 1 hour"). |

## 5. Development Plan

### Phase 1: The Secure Foundation

- Set up GCP Project & Rust boilerplate.
- Implement the "Core Gateway" with Vertex AI integration.
- Implement the "Secrets Manager" interface (GCP + encrypted local store).

### Phase 2: The Action Layer

- Build the **Wasm Plugin System**.
- Port key capabilities (Calendar, Email, Docs) as sandboxed plugins.
- Integrate **Nango** (or similar) for OAuth.

### Phase 3: The Interaction Layer

- Connect to Messaging App (Telegram/Signal).
- Implement the "Supervisor" security layer (Prompt Injection defense).

### Phase 4: Migration (Optional)

- Analyze legacy submodules (`mastermind-desktop`, `mastermind-obsidian`) for feature parity but *do not copy code directly* to maintain Rust/Security strictness.
