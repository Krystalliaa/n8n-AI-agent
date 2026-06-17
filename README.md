# 🤖 n8n AI Automation Ecosystem

> A self-hosted, Docker-based personal automation ecosystem built with n8n, Claude AI, and open-source tools. This repository documents my ongoing journey building AI agents, automation workflows, and homelab infrastructure — serving as both a technical portfolio and a living knowledge base.

---

## About This Repository

This repository is not a finished product. It is a **continuous learning journey** — each project builds on the previous one, adding new capabilities, integrations, and architectural layers.

Every workflow documented here was built to solve a real problem. Every design decision is explained. Every lesson learned is recorded.

The ecosystem is built on three principles:

- **Local-first:** Everything runs on self-hosted infrastructure. No vendor lock-in.
- **Layer-based:** Each new project extends the system without breaking existing workflows.
- **Portfolio-driven:** Every implementation is documented to demonstrate real-world technical experience.

---

## Technology Stack

| Category | Technologies |
|---|---|
| **Automation Engine** | n8n (self-hosted via Docker) |
| **AI / LLM** | Anthropic Claude API |
| **Infrastructure** | Docker Desktop · Windows 11 |
| **Version Control** | GitHub API |
| **Productivity** | Google Docs API · Google Sheets API |
| **Search** | Tavily Web Search |
| **Databases** | Google Sheets → SQLite → PostgreSQL |
| **Future** | Home Assistant · Raspberry Pi · Local LLMs |

---

## Repository Structure

```
/
├── README.md                          ← You are here
├── docs/
│   ├── 01-personal-ai-assistant.md    ← Workflow documentation
│   ├── 02-readme-refiner-agent.md
│   ├── 03-idea-capture-agent.md
│   └── architecture-overview.md      ← Full ecosystem architecture
├── workflows/                         ← n8n workflow JSON exports
├── images/                            ← Screenshots per workflow
├── prompts/                           ← System prompt files
├── schemas/                           ← Database schemas
└── roadmap/                           ← Future planning documents
```

---

## Workflow Catalog

| # | Project | Status | Description |
|---|---|---|---|
| 01 | [Personal AI Assistant](docs/01-personal-ai-assistant.md) | ✅ Complete | Self-hosted Claude-powered assistant with web search and Google Docs export |
| 02 | [README Refiner Agent](docs/02-readme-refiner-agent.md) | ✅ Complete | Multi-file documentation publisher with dynamic GitHub publishing |
| 03 | [Idea Capture Agent](docs/03-idea-capture-agent.md) | 🔄 In Progress | Intent-detecting agent that extracts and stores structured ideas to a database |
| 04 | Job Search Agent | 📋 Planned | Automated job search with structured results and document generation |
| 05 | Home Assistant Integration | 📋 Planned | Bidirectional smart home control via n8n |
| 06 | RV Voice Assistant | 📋 Planned | Offline-capable voice assistant for mobile/off-grid use |

---

## Architecture Overview

The ecosystem is organized into five layers. Each layer depends on the one below it.

```
┌─────────────────────────────────────────────────────┐
│              LAYER 5 — PHYSICAL ENVIRONMENT          │
│         Home Assistant · RV Voice Assistant          │
├─────────────────────────────────────────────────────┤
│              LAYER 4 — KNOWLEDGE LAYER               │
│      Idea Capture Agent · Structured Database        │
├─────────────────────────────────────────────────────┤
│           LAYER 3 — DOCUMENTATION LAYER              │
│     README Refiner Agent · GitHub API                │
├─────────────────────────────────────────────────────┤
│              LAYER 2 — AI AGENT LAYER                │
│        Claude API · Web Search · Google Docs         │
├─────────────────────────────────────────────────────┤
│           LAYER 1 — INFRASTRUCTURE LAYER             │
│     n8n (Docker) · Windows 11 · Local Network        │
└─────────────────────────────────────────────────────┘
```

➡️ Full architecture details: [docs/architecture-overview.md](docs/architecture-overview.md)

---

## Project Progression

Each project introduced new skills and capabilities that enabled the next one:

```
01 — Personal AI Assistant
│   → Established: n8n, Docker, Claude API, Chat interface
│
├── 02 — README Refiner Agent
│       → Added: GitHub API, prompt engineering, multi-file publishing
│
├── 03 — Idea Capture Agent
│       → Added: Intent detection, JSON extraction, database storage
│
├── 04 — Job Search Agent
│       → Adds: External API integration, structured search
│
├── 05 — Home Assistant Integration
│       → Adds: Physical environment, smart device control
│
└── 06 — RV Voice Assistant
        → Adds: Voice interface, offline LLM, off-grid infrastructure
```

---

## Roadmap

| Phase | Goal | Status |
|---|---|---|
| **Phase 1** | Core AI assistant + documentation pipeline | ✅ Complete |
| **Phase 2** | Persistent knowledge base (Idea Capture Agent) | 🔄 In Progress |
| **Phase 3** | PostgreSQL shared memory layer across all agents | 📋 Planned |
| **Phase 4** | Home Assistant integration + smart home automation | 📋 Planned |
| **Phase 5** | Offline-capable voice assistant for RV / off-grid use | 📋 Planned |

---

## Documentation

| Document | Description |
|---|---|
| [Architecture Overview](docs/architecture-overview.md) | Full system architecture, data flows, dependency map |
| [Workflow 01 — Personal AI Assistant](docs/01-personal-ai-assistant.md) | First workflow deep-dive |
| [Workflow 02 — README Refiner Agent](docs/02-readme-refiner-agent.md) | Documentation publisher deep-dive |

---

*This repository is actively maintained and updated as new projects are completed.*
