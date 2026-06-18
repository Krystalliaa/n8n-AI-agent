# README Refiner Agent — Multi-File Documentation Publisher

**Document:** `docs/02-readme-refiner-agent.md`
**Status:** ✅ Complete

---

This is the second workflow I built as part of my personal AI automation ecosystem. I created it because I noticed a pattern early on: writing and maintaining technical documentation manually is slow, inconsistent, and easy to deprioritize. I wanted an agent that could take my raw notes, rough drafts, and project descriptions and transform them into professional portfolio-grade documentation — and then publish it directly to GitHub, in the right file, without me leaving the chat.

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
Code Node (Base64 Encoding)
        ↓
HTTP GET (Retrieve file SHA from GitHub)
        ↓
HTTP PUT (Update file via GitHub Contents API)
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

### GitHub Operations Outside the AI Agent

An important architectural lesson I learned during implementation was that **GitHub GET and PUT operations must live in the main workflow execution path — not inside the AI Agent's tool list**. Tool nodes cannot be chained and executed sequentially like standard workflow nodes. The AI Agent's responsibility is limited to generating `file_content`, `commit_message`, and `file_path`. The actual GitHub operations are handled by dedicated workflow nodes downstream.

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

### Step 3 — Code Node (Base64 Encoding)

I added a dedicated Code node between the AI Agent and the GitHub API requests. GitHub's Contents API requires file content to be Base64 encoded before uploading. The node extracts the agent output and encodes it:

```javascript
/** @type {any} */

const input = $input.first().json;

// If file_content is missing, the agent responded conversationally
if (!input.file_content) {
  return [{
    json: {
      chat_response: input.output || "No response",
      file_content: null,
      commit_message: null,
      file_path: null
    }
  }];
}

const content = input.file_content;
const commitMessage = input.commit_message;
const filePath = input.file_path;

const encoded = Buffer.from(content).toString('base64');

return [{
  json: {
    file_content: encoded,
    commit_message: commitMessage,
    file_path: filePath
  }
}];
```

### Step 4 — IF Node (Route Upload vs Chat Response)

I added an IF node after the Code Node to split the workflow into two paths:

```
Condition: {{ $json.file_content }} is not empty
→ TRUE  → HTTP GET → HTTP PUT (GitHub upload)
→ FALSE → Return chat_response to user
```

This prevents the workflow from failing when the agent responds conversationally without triggering a file upload.

### Step 5 — HTTP GET Request (Retrieve SHA)

Before updating any file, I added an HTTP GET request to retrieve the file's current metadata from the GitHub Contents API. GitHub requires the current file SHA when updating an existing file.

- **Method:** GET
- **URL:** `https://api.github.com/repos/Krystalliaa/n8n-AI-agent/contents/{{ $json.file_path }}`
- **Authentication:** GitHub Personal Access Token

### Step 6 — HTTP PUT Request (Update File)

I configured an HTTP PUT request to push the updated file to GitHub. The request body includes the Base64-encoded content, the commit message, and the SHA retrieved in the previous step:

```json
{
  "message": "{{ $json.commit_message }}",
  "content": "{{ $json.file_content }}",
  "sha": "{{ $('HTTP Request GET').item.json.sha }}"
}
```

### Step 7 — Google Docs Tool *(Optional)*

I connected a Google Docs tool for cases where the user wants to export a document draft before publishing to GitHub. This uses the same Google OAuth2 credential from Project 01.

### Step 8 — GitHub Personal Access Token

I created a **Fine-Grained Personal Access Token** with the following configuration:

- **Repository Access:** Only selected repositories → `n8n-AI-agent`
- **Repository Permissions:** Contents → Read and Write

This ensures the workflow can both read repository contents (GET) and update files (PUT) through the GitHub API.

---

## Testing

[SCREENSHOT: Chat interface — raw notes input]

[SCREENSHOT: Agent draft output — formatted documentation]

[SCREENSHOT: Code node — Base64 encoding output]

[SCREENSHOT: HTTP GET — file metadata and SHA retrieved]

[SCREENSHOT: HTTP PUT — file update request and response]

[SCREENSHOT: GitHub repository — file created/updated]

[SCREENSHOT: Approval flow — user confirmation]

---

## Troubleshooting

### Issue 1 — GitHub API Error: "sha" wasn't supplied

**Error:**
```
Your request is invalid or could not be processed by the service.
Invalid request. "sha" wasn't supplied.
```

**Cause:**
When updating an existing file through the GitHub Contents API, GitHub requires the current file SHA. The workflow attempted to update the file without first retrieving the SHA of the existing file.

**Solution:**
I added an HTTP GET request before the HTTP PUT request. The workflow now follows this sequence:

```
AI Agent
   ↓
HTTP GET (Retrieve file metadata and SHA)
   ↓
HTTP PUT (Update file using returned SHA)
```

The PUT request body was updated to include:

```json
{
  "message": "{{ $json.commit_message }}",
  "content": "{{ $json.file_content }}",
  "sha": "{{ $('HTTP Request GET').item.json.sha }}"
}
```

---

### Issue 2 — n8n Code Node Error: `.item` only works in "Run Once for Each Item" mode

**Error:**
```
.item only works correctly in 'Run Once for Each Item' mode.
Property 'item' does not exist on type 'N8nInput'.
```

**Cause:**
The Code node was configured to run in "Run Once for All Items" mode while using the per-item API (`$input.item`).

**Solution:**
I updated the code to use:

```javascript
$input.first().json
```

instead of:

```javascript
$input.item.json
```

This matches the execution mode of the node.

---

### Issue 3 — n8n Code Node Error: `buffer is not defined`

**Error:**
```
ReferenceError: buffer is not defined
```

**Cause:**
The code used `buffer.from(...)` with a lowercase `b`. In Node.js, the correct global object is `Buffer` with an uppercase `B`.

**Solution:**
I changed:

```javascript
const encoded = buffer.from(content).toString('base64');
```

to:

```javascript
const encoded = Buffer.from(content).toString('base64');
```

---

### Issue 4 — Base64 Encoding Requirement for GitHub API

**Problem:**
GitHub's Contents API requires file content to be Base64 encoded before uploading. The initial workflow sent raw markdown content, which the API rejected.

**Solution:**
I added a dedicated Code node between the AI Agent and the GitHub API requests to handle encoding:

```javascript
/** @type {any} */

const content = $input.first().json.file_content;
const commitMessage = $input.first().json.commit_message;
const filePath = $input.first().json.file_path;

const encoded = Buffer.from(content).toString('base64');

return [{
  json: {
    file_content: encoded,
    commit_message: commitMessage,
    file_path: filePath
  }
}];
```

---

### Issue 5 — AI Agent Tool Architecture Confusion

**Problem:**
Initially, the GitHub GET and PUT operations were configured as AI Agent tools. Tool nodes cannot be chained and executed sequentially like standard workflow nodes, which caused the operations to fail or execute out of order.

**Solution:**
I moved the GitHub operations into the main workflow execution path:

```
Chat Trigger
   ↓
AI Agent
   ↓
Code Node (Base64 Encoding)
   ↓
HTTP GET (Retrieve SHA)
   ↓
HTTP PUT (Update File)
```

The AI Agent became responsible only for generating `file_content`, `commit_message`, and `file_path`. All GitHub operations are performed by downstream workflow nodes.

---

### Issue 6 — GitHub Personal Access Token Permissions

**Problem:**
The GitHub token did not have sufficient repository permissions, causing the API to reject write requests.

**Solution:**
I created a **Fine-Grained Personal Access Token** with:

- **Repository Access:** Only selected repositories → `n8n-AI-agent`
- **Repository Permissions:** Contents → Read and Write

This ensures the workflow can read repository contents and update files through the GitHub API.

---

### Issue 7 — Code Node Error: `Missing file_content`

**Error:**
```
Error: Missing file_content. Received: {"output":"..."}
```

**Cause:**
The AI Agent was returning a conversational response wrapped in an `output` field instead of the expected JSON structure with `file_content`, `commit_message`, and `file_path`. This happened when the agent responded to a message without triggering an upload — for example, answering a question or explaining something. The Code Node expected `file_content` to always be present and threw an error when it was missing.

**Solution:**
I updated the Code Node to handle both cases — a structured upload response and a plain conversational response:

```javascript
/** @type {any} */

const input = $input.first().json;

// If file_content is missing, the agent responded conversationally
if (!input.file_content) {
  return [{
    json: {
      chat_response: input.output || "No response",
      file_content: null,
      commit_message: null,
      file_path: null
    }
  }];
}

const content = input.file_content;
const commitMessage = input.commit_message;
const filePath = input.file_path;

const encoded = Buffer.from(content).toString('base64');

return [{
  json: {
    file_content: encoded,
    commit_message: commitMessage,
    file_path: filePath
  }
}];
```

I also added an **IF Node** after the Code Node to route the two paths:

```
Condition: {{ $json.file_content }} is not empty
→ TRUE  → HTTP GET → HTTP PUT (GitHub upload)
→ FALSE → Return chat_response to user
```

This made the workflow resilient to conversational messages and eliminated the runtime crash entirely.

---

## What I Learned

- **Dynamic tool parameterization:** I learned how to design an AI agent that determines its own tool parameters (file path, repository, commit message) from conversation context rather than hardcoded values. This is a pattern directly applicable to autonomous agent design.

- **Prompt engineering as system design:** The system prompt is effectively the architecture of this agent. Writing it required the same thinking as designing a software specification — defining rules, edge cases, output formats, and behavioral constraints.

- **Approval gates in autonomous systems:** I made a deliberate decision to require explicit human approval before any GitHub write operation. This introduced me to the concept of human-in-the-loop design in agentic workflows.

- **Repository-aware documentation:** Designing an agent that understands the full structure of a repository — and can route content to the correct file — required me to think about documentation as a system, not just a collection of individual files.

- **Multi-file publishing architecture:** Extending the agent from single-file to multi-file capability required no changes to the n8n workflow itself — only the system prompt. This reinforced the value of prompt-driven architecture for flexible AI agents.

- **GitHub file updates require the current file SHA:** Any PUT request to the GitHub Contents API that targets an existing file must include the file's current SHA. This is a non-negotiable requirement of the API and must be retrieved with a prior GET request.

- **GitHub Contents API expects Base64 encoded content:** Raw text content is not accepted. Every file update must be encoded with `Buffer.from(content).toString('base64')` before being sent.

- **n8n Code node execution mode affects input access:** When a Code node runs in "Run Once for All Items" mode, the correct way to access input data is `$input.first().json`, not `$input.item.json`. Mixing execution modes and input APIs causes runtime errors.

- **`Buffer` is case-sensitive in Node.js:** `buffer` (lowercase) is not a defined global. `Buffer` (uppercase) is the correct Node.js global object for binary data operations. A single character difference caused a `ReferenceError` that halted the workflow.

- **AI Agent tools should not be used for sequential workflow logic:** Tool nodes in n8n are designed for agent-invoked actions, not deterministic sequential pipelines. GitHub read/write operations belong in the main workflow execution path, not in the agent's tool list.

- **Fine-grained GitHub tokens require explicit Contents permissions:** A token scoped only to metadata or code reading is not sufficient for file updates. Contents → Read and Write must be explicitly granted on the target repository.

- **The AI Agent output field wraps conversational responses:** When the agent responds conversationally (not with a structured JSON upload), the output is wrapped in an `output` field. The Code Node must handle this gracefully with a null check, and an IF Node must split the upload path from the chat response path.
