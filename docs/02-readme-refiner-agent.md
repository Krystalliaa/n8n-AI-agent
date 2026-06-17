# README Refiner Agent — Multi-File Documentation Publisher

**Document:** `docs/02-readme-refiner-agent.md`
**Status:** ✅ Complete

---

Hello! This is the second workflow I built as part of my personal AI automation ecosystem. I created it because I noticed a pattern early on: writing and maintaining technical documentation manually is slow, inconsistent, and easy to deprioritize. I wanted an agent that could take my raw notes, rough drafts, and project descriptions and transform them into professional portfolio-grade documentation — and then publish it directly to GitHub, in the right file, without me leaving the chat.

What started as a README generator evolved into something more powerful: a **multi-file documentation publisher** that understands the structure of the entire repository, detects which file needs to be created or updated from context, and pushes the result to GitHub after explicit user approval.

---

## Tech Stack

| Technology | Role |
|---|---|
| **n8n** (self-hosted via Docker) | Workflow automation platform |
| **Anthropic Claude API** | Documentation generation and file path detection |
| **GitHub API** | Multi-file publishing to repository |
| **Google Docs API** | Optional document export before publishing |

---

## Hardware

| Component | Specification |
|---|---|
| **Machine** | ASUS TUF Gaming A16 |
| **CPU** | AMD Ryzen 7 260 |
| **GPU** | RTX 5050 |
| **RAM** | 16GB |
| **OS** | Windows 11 |

---

## What This Agent Does

The README Refiner Agent is a Documentation Architect assistant that:

1. Accepts raw notes, rough drafts, or project descriptions via chat
2. Determines the **document type** (Root README, Workflow Doc, Architecture Doc, Roadmap, etc.)
3. Detects the correct **target file path** automatically from context
4. Rewrites the content into professional portfolio documentation
5. Presents the draft to the user for review
6. On approval, **publishes directly to GitHub** at the correct path
7. Confirms completion

---

## Workflow Architecture

```
User Input (Chat)
        ↓
Chat Trigger (n8n)
        ↓
README Refiner Agent (Claude API)
        ├── Determines document type
        ├── Detects target file path from context
        ├── Applies repository writing standards
        ├── Generates documentation draft
        └── Presents draft to user
        ↓
[User Reviews — Requests Changes or Approves]
        ↓
[On Approval]
        ↓
GitHub Tool
        ├── Repository Owner: Krystalliaa
        ├── Repository Name: n8n-AI-agent
        ├── File Path: detected automatically by model
        └── Commit Message: generated automatically
        ↓
GitHub API → File Created / Updated
        ↓
Confirmation → User
```

---

## Key Design Decisions

### Dynamic File Path Detection

The most important architectural decision in this workflow is that the **file path is never hardcoded**. The agent determines the correct target file from the conversation context:

| User Intent | Detected File Path |
|---|---|
| "Update the main README" | `README.md` |
| "Document the Idea Capture Agent" | `docs/03-idea-capture-agent.md` |
| "Create the architecture overview" | `docs/architecture-overview.md` |
| "Update the roadmap" | `roadmap/future-roadmap.md` |
| "Add the job search agent doc" | `docs/04-job-search-agent.md` |

This means the agent can maintain the **entire repository** from a single chat interface.

### Approval Gate

The agent never publishes automatically. Every upload requires an explicit approval signal from the user:

```
Approval triggers: "upload it", "push it", "looks good",
                   "publish it", "ανέβασέ το"
```

This ensures no accidental overwrites and keeps the user in full control of what enters the repository.

### Document Type System

The agent recognizes and applies different formatting rules based on the document type:

| Document Type | Rules Applied |
|---|---|
| Root README | Ecosystem overview, no implementation details |
| Workflow Document | Full technical deep-dive, first-person, step-by-step |
| Architecture Document | System-level view, dependency maps, data flows |
| Roadmap Document | Future goals, phased planning |
| Troubleshooting Document | Errors encountered, solutions applied |
| Project Summary | High-level overview for non-technical readers |

---

## System Prompt Design

The agent's behavior is entirely defined by its system prompt. Key rules encoded in the prompt:

- **Voice:** First-person, past tense ("I installed...", "I configured...")
- **Structure:** Major sections (`##`), subsections (`###`), numbered steps
- **Code:** Every command in fenced code blocks — never inline
- **Screenshots:** Placeholders only — never invented
- **Troubleshooting:** Never invent errors
- **What I Learned:** Always present, always professionally framed
- **Upload gate:** Never upload without explicit user approval

---

## Step-by-Step Setup

### Step 1 — Chat Trigger

I used n8n's built-in **Chat Trigger** node as the entry point. This is the same trigger node used in Project 01 — no additional configuration was needed.

### Step 2 — AI Agent Node

I configured an AI Agent node with:

- **Model:** Claude (via Anthropic API credential)
- **System Prompt:** Full Documentation Architect prompt defining all writing rules, document types, repository structure, and upload behavior

### Step 3 — GitHub Tool

I added the GitHub tool to the AI Agent's tool list and configured it with:

- **Authentication:** GitHub Personal Access Token (stored in n8n credentials)
- **Repository Owner:** `Krystalliaa`
- **Repository Name:** `n8n-AI-agent`
- **File Path:** Defined dynamically by the model from context
- **Commit Message:** Generated automatically by the model

### Step 4 — Google Docs Tool *(Optional)*

I connected a Google Docs tool for cases where the user wants to export a document draft before publishing to GitHub. This uses the same Google OAuth2 credential from Project 01.

---

## Testing

[SCREENSHOT: Chat interface — raw notes input]
[SCREENSHOT: Agent draft output — formatted documentation]
[SCREENSHOT: GitHub tool call — file path and commit message]
[SCREENSHOT: GitHub repository — file created/updated]
[SCREENSHOT: Approval flow — user confirmation]

---

## Troubleshooting

[Add any troubleshooting notes encountered during implementation.]

---

## What I Learned

- **Dynamic tool parameterization:** I learned how to design an AI agent that determines its own tool parameters (file path, repository, commit message) from conversation context rather than hardcoded values. This is a pattern directly applicable to autonomous agent design.
- **Prompt engineering as system design:** The system prompt is effectively the architecture of this agent. Writing it required the same thinking as designing a software specification — defining rules, edge cases, output formats, and behavioral constraints.
- **Approval gates in autonomous systems:** I made a deliberate decision to require explicit human approval before any GitHub write operation. This introduced me to the concept of human-in-the-loop design in agentic workflows.
- **Repository-aware documentation:** Designing an agent that understands the full structure of a repository — and can route content to the correct file — required me to think about documentation as a system, not just a collection of individual files.
- **Multi-file publishing architecture:** Extending the agent from single-file to multi-file capability required no changes to the n8n workflow itself — only the system prompt. This reinforced the value of prompt-driven architecture for flexible AI agents.
