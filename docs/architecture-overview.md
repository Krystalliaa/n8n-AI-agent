# Ecosystem Architecture Overview

Hello! This document describes the full architecture of my personal AI automation ecosystem. It is a living document — updated as new projects are added and the system evolves. Rather than documenting individual workflows in isolation, this file captures how everything connects, depends on each other, and builds toward a unified personal automation platform.

---

## Architecture Philosophy

This ecosystem was not designed top-down. It grew organically from a single problem — I needed a personal AI assistant that fit how I think and work — and expanded outward as each new capability revealed the next missing piece.

The core design principles are:

- **Local-first:** Everything runs on self-hosted infrastructure. No vendor lock-in.
- **Layer-based:** Each project adds a capability layer without breaking existing ones.
- **Database-agnostic:** Storage layers are interchangeable. Workflow logic stays the same.
- **Progressive complexity:** Simple problems first. Infrastructure scales with need.
- **Portfolio-driven:** Every implementation decision is documented for professional visibility.

---

## System Layers

The ecosystem is organized into five distinct layers. Each layer depends on the one below it.

```
┌─────────────────────────────────────────────────────┐
│              LAYER 5 — PHYSICAL ENVIRONMENT          │
│         Home Assistant · RV Voice Assistant          │
│         Smart Devices · Offline Infrastructure       │
├─────────────────────────────────────────────────────┤
│              LAYER 4 — KNOWLEDGE LAYER               │
│      Idea Capture Agent · Structured Database        │
│      Query Engine · Long-term Memory                 │
├─────────────────────────────────────────────────────┤
│           LAYER 3 — DOCUMENTATION LAYER              │
│     README Refiner Agent · GitHub API                │
│     Multi-file Publishing · Prompt Engineering       │
├─────────────────────────────────────────────────────┤
│              LAYER 2 — AI AGENT LAYER                │
│        Claude API · Web Search · Google Docs         │
│        Chat Interface · Memory · Tool Use            │
├─────────────────────────────────────────────────────┤
│           LAYER 1 — INFRASTRUCTURE LAYER             │
│     n8n (Docker) · Windows 11 · Local Network        │
│     API Credentials · Webhook Endpoints              │
└─────────────────────────────────────────────────────┘
```

---

## Project Dependency Map

Each project is a direct dependency of the one that follows it. Nothing is built in isolation.

```
01 — Personal AI Assistant
│   Establishes: n8n instance, Claude API, Docker, Chat interface,
│                Web Search, Google Docs export
│
├── 02 — README Refiner Agent
│       Depends on: 01 (Claude API, n8n, Chat interface)
│       Adds: GitHub API, multi-file documentation publishing,
│             dynamic file path detection, prompt engineering
│
├── 03 — Idea Capture Agent
│       Depends on: 01 (Claude API, n8n, Chat interface)
│       Adds: Intent detection, structured JSON extraction,
│             Google Sheets → SQLite → PostgreSQL migration path,
│             persistent memory layer
│
├── 04 — Job Search Agent
│       Depends on: 01, 02, 03
│       Adds: External job API integration, structured search,
│             automated document generation
│
├── 05 — Home Assistant Integration
│       Depends on: 01, 03
│       Adds: Physical environment awareness, smart device control,
│             local network integration
│
└── 06 — RV Voice Assistant
        Depends on: 01, 03, 05
        Adds: Voice interface, offline-capable LLM,
              mobile infrastructure, off-grid automation
```

---

## Infrastructure Overview

### Core Infrastructure

| Component | Technology | Purpose |
|---|---|---|
| **Automation Engine** | n8n (self-hosted) | All workflow logic |
| **Containerization** | Docker Desktop | Isolated, portable services |
| **AI Engine** | Anthropic Claude API | All LLM-powered agents |
| **Operating System** | Windows 11 | Primary development machine |
| **Version Control** | GitHub | Code, workflows, documentation |

### Storage Layer

| Phase | Technology | Used In | Status |
|---|---|---|---|
| Phase 1 | Google Sheets | Idea Capture Agent | Planned |
| Phase 2 | SQLite | Idea Capture Agent | Planned |
| Phase 3 | PostgreSQL | All agents (shared) | Planned |

### External APIs & Services

| API / Service | Used In | Purpose |
|---|---|---|
| **Anthropic Claude API** | All AI agents | LLM inference |
| **GitHub API** | README Refiner Agent | File publishing |
| **Google Docs API** | Personal AI Assistant | Document export |
| **Google Sheets API** | Idea Capture Agent | Phase 1 storage |
| **Tavily / Web Search** | Personal AI Assistant | Live web search |

---

## Data Flow Architecture

### Flow 1 — Standard AI Conversation

```
User (Chat)
    ↓
n8n Chat Trigger
    ↓
AI Agent (Claude API)
    ├── Web Search Tool (if needed)
    └── Google Docs Tool (if needed)
    ↓
Response → User
```

### Flow 2 — Documentation Publishing

```
User (Chat)
    ↓
n8n Chat Trigger
    ↓
README Refiner Agent (Claude API)
    ├── Detects target file from context
    ├── Generates/refines documentation
    ├── Presents draft to user
    └── Awaits approval
    ↓
[User Approves]
    ↓
GitHub API → Upload to target file path
    ↓
Confirmation → User
```

### Flow 3 — Idea Capture

```
User (Chat)
    ↓
n8n Chat Trigger
    ↓
AI Agent (Claude API) — Intent Detection
    ├── NORMAL_CHAT → Standard Response
    └── IDEA_CAPTURE
            ↓
        Structured JSON Extraction
            ↓
        Function Node (UUID + Timestamps)
            ↓
        Storage Router
            ├── Google Sheets (Phase 1)
            ├── SQLite (Phase 2)
            └── PostgreSQL (Phase 3)
```

---

## Storage Migration Strategy

A core architectural decision in this ecosystem is the **phased storage migration** used in the Idea Capture Agent. The workflow logic is written once and never modified — only the active storage backend is swapped.

```
Phase 1 — Google Sheets
│   Fast prototype. Zero infrastructure.
│   Validates schema and workflow logic.
│
Phase 2 — SQLite
│   Local-first. Fully offline.
│   Ideal for Raspberry Pi and Home Assistant ecosystem.
│   Easy backup via Nextcloud or rsync.
│
Phase 3 — PostgreSQL
    Scalable production backend.
    JSONB tags, full-text search, composite indexing.
    Multi-device access and analytics dashboards.
```

---

## Credential Architecture

All credentials are managed centrally inside n8n's built-in credential store. No API keys are hardcoded in any workflow.

| Credential | Type | Used By |
|---|---|---|
| `Anthropic Claude API` | API Key | All AI agents |
| `GitHub Personal Access Token` | OAuth / PAT | README Refiner Agent |
| `Google OAuth2` | OAuth2 | Personal AI Assistant, Idea Capture Agent |
| `PostgreSQL Connection` | DB Credentials | Idea Capture Agent (Phase 3) |

---

## Planned Architecture Expansions

| Expansion | Target Project | Description |
|---|---|---|
| **Shared Knowledge Base** | Project 04+ | PostgreSQL layer shared across all agents |
| **Voice Interface** | Project 06 | Whisper / local STT for RV Assistant |
| **Local LLM Fallback** | Project 06 | Offline-capable inference for off-grid use |
| **Home Assistant API** | Project 05 | Bidirectional smart home control |
| **Analytics Dashboard** | Future | Query and visualize the ideas database |
| **GitHub Issues Integration** | Future | Auto-generate issues from captured ideas |

---

## Architecture Diagrams

[SCREENSHOT: Full ecosystem architecture diagram]
[SCREENSHOT: n8n Docker container overview]
[SCREENSHOT: Credential store — n8n]
[SCREENSHOT: Workflow dependency map]

---

## Document History

| Version | Update |
|---|---|
| v1.0 | Initial architecture overview — Projects 01-02 documented |
| v1.1 | [To be updated as new projects are added] |
