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
| **GitHub API** | Multi-file publishing to repository via built-in n8n tools |
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
6. On approval, **publishes directly to GitHub** at the correct path via its built-in tools
7. Confirms completion

---

## Workflow Architecture

```
User Input (Chat)
        ↓
Chat Trigger (n8n)
        ↓
AI Agent (Claude API)
        ├── Determines document type
        ├── Detects target file path from context
        ├── Applies repository writing standards
        ├── Generates documentation draft
        └── Presents draft to user
        ↓
[User Reviews — Requests Changes or Approves]
        ↓
[On Approval — Agent returns structured JSON]
        ↓
Code Node (defensive null check)
        ↓
IF Node
        ├── TRUE (file_content present) → Back to AI Agent → Agent uploads via built-in tools
        └── FALSE (conversational response) → Return chat_response to user
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

### No HTTP Nodes Required

An important discovery during implementation was that **separate HTTP GET and PUT nodes were not needed**. The AI Agent handles all GitHub operations through its built-in tools — including SHA retrieval, Base64 encoding, and file updates. This simplified the workflow significantly and removed several nodes that I initially thought were mandatory.

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

### IF Node Loop Back to AI Agent

After the Code Node validates the output, the IF Node routes the execution:

- **TRUE** — `file_content` is present → the flow loops back to the AI Agent, which uses its built-in tools to upload the file to GitHub
- **FALSE** — the agent responded conversationally → the `chat_response` is returned directly to the user

When the IF Node routes to **TRUE**, the AI Agent receives the structured JSON containing `file_content`, `file_path`, and `commit_message`. The agent recognizes this as an upload instruction and uses its built-in GitHub tools to:

1. Call `get_github_metadata` to retrieve the current file SHA
2. Call `upload_to_github` with the file content, path, commit message, and SHA

This design keeps the upload logic entirely inside the agent's tool layer and avoids the need for any additional HTTP request nodes in the workflow.

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
- **Tools:** Built-in GitHub tools for file retrieval and file upload

### Step 3 — Code Node (Defensive Null Check)

I added a Code node after the AI Agent to handle two possible response types — a structured upload response and a plain conversational response:

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

return [{
  json: {
    file_content: content,
    commit_message: commitMessage,
    file_path: filePath
  }
}];
```

### Step 4 — IF Node (Route Upload vs Chat Response)

I added an IF node after the Code Node to split the workflow into two paths:

```
Condition: {{ $json.file_content }} is not empty
→ TRUE  → Back to AI Agent (agent uploads via built-in tools)
→ FALSE → Return chat_response to user
```

When `file_content` is present, the flow returns to the AI Agent with the structured data. The agent receives `file_content`, `file_path`, and `commit_message` and uses its built-in tools to:

1. Call `get_github_metadata` to retrieve the current SHA of the target file
2. Call `upload_to_github` to push the updated content to GitHub

This eliminates the need for any additional HTTP nodes in the workflow.

### Step 5 — Google Docs Tool *(Optional)*

I connected a Google Docs tool for cases where the user wants to export a document draft before publishing to GitHub. This uses the same Google OAuth2 credential from Project 01.

### Step 6 — GitHub Personal Access Token

I created a **Fine-Grained Personal Access Token** with the following configuration:

- **Repository Access:** Only selected repositories → `n8n-AI-agent`
- **Repository Permissions:** Contents → Read and Write

This ensures the agent's built-in GitHub tools can both read repository contents and update files.

---

## Testing

[SCREENSHOT: Chat interface — raw notes input]

[SCREENSHOT: Agent draft output — formatted documentation]

[SCREENSHOT: Code node — null check output]

[SCREENSHOT: IF Node — TRUE path routing back to AI Agent]

[SCREENSHOT: AI Agent — tool execution for GitHub upload]

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
I discovered that the AI Agent's built-in GitHub tools handle SHA retrieval automatically. Removing the manual HTTP GET / PUT nodes and delegating the upload to the agent's tools resolved the issue entirely.

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

### Issue 4 — Base64 Encoding Not Required When Using Built-in Tools

**Problem:**
Initially I added a Base64 encoding step in the Code Node because the GitHub Contents API requires encoded content. The workflow included `Buffer.from(content).toString('base64')` before sending data to GitHub.

**Solution:**
After switching to the AI Agent's built-in GitHub tools, I discovered that **Base64 encoding is handled automatically** by the tools. I removed the encoding step from the Code Node, which simplified the logic significantly.

---

### Issue 5 — HTTP GET and PUT Nodes Were Not Needed

**Problem:**
The initial workflow design included a dedicated HTTP GET node to retrieve the file SHA and a dedicated HTTP PUT node to update the file on GitHub. This added complexity and required manual handling of authentication headers, request bodies, and SHA management.

**Solution:**
I replaced both HTTP nodes by routing the TRUE path of the IF Node back to the AI Agent. The agent's built-in GitHub tools handle SHA retrieval, encoding, and file updates internally. This removed two nodes from the workflow and eliminated all manual GitHub API configuration.

---

### Issue 6 — GitHub Personal Access Token Permissions

**Problem:**
The GitHub token did not have sufficient repository permissions, causing the API to reject write requests.

**Solution:**
I created a **Fine-Grained Personal Access Token** with:

- **Repository Access:** Only selected repositories → `n8n-AI-agent`
- **Repository Permissions:** Contents → Read and Write

---

### Issue 7 — Code Node Error: `Missing file_content`

**Error:**
```
Error: Missing file_content. Received: {"output":"..."}
```

**Cause:**
The AI Agent was returning a conversational response wrapped in an `output` field instead of the expected JSON structure with `file_content`, `commit_message`, and `file_path`. The Code Node expected `file_content` to always be present and threw an error when it was missing.

**Solution:**
I updated the Code Node to handle both cases — a structured upload response and a plain conversational response:

```javascript
/** @type {any} */

const input = $input.first().json;

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

return [{
  json: {
    file_content: input.file_content,
    commit_message: input.commit_message,
    file_path: input.file_path
  }
}];
```

I also added an **IF Node** after the Code Node to route the two paths:

```
Condition: {{ $json.file_content }} is not empty
→ TRUE  → Back to AI Agent (upload via built-in tools)
→ FALSE → Return chat_response to user
```

---

## What I Learned

- **Dynamic tool parameterization:** I learned how to design an AI agent that determines its own tool parameters (file path, repository, commit message) from conversation context rather than hardcoded values.

- **Prompt engineering as system design:** The system prompt is effectively the architecture of this agent. Writing it required the same thinking as designing a software specification — defining rules, edge cases, output formats, and behavioral constraints.

- **Approval gates in autonomous systems:** I made a deliberate decision to require explicit human approval before any GitHub write operation. This introduced me to the concept of human-in-the-loop design in agentic workflows.

- **Built-in tools eliminate manual API management:** I initially built the GitHub integration using raw HTTP GET and PUT nodes, managing SHA retrieval, Base64 encoding, and authentication manually. Switching to the AI Agent's built-in GitHub tools removed all of that complexity automatically.

- **Looping back to the AI Agent is a valid routing pattern:** Instead of adding more downstream nodes to handle uploads, I routed the TRUE path of the IF Node back to the AI Agent. The agent then used its built-in tools to complete the upload. This kept the workflow simple and the upload logic centralized inside the agent.

- **The Code Node acts as a router, not a transformer:** After removing the Base64 encoding step, the Code Node's only job became detecting whether the agent returned a structured upload response or a conversational one — and routing accordingly.

- **GitHub file updates require the current file SHA:** Any update to an existing file via the GitHub Contents API must include the file's current SHA. The built-in tools handle this automatically, but understanding why it is required helped me debug earlier failures.

- **n8n Code node execution mode affects input access:** When a Code node runs in "Run Once for All Items" mode, the correct way to access input data is `$input.first().json`, not `$input.item.json`.

- **`Buffer` is case-sensitive in Node.js:** `buffer` (lowercase) is not a defined global. A single character difference caused a `ReferenceError` that halted the workflow.

- **Fine-grained GitHub tokens require explicit Contents permissions:** A token scoped only to metadata or code reading is not sufficient for file updates. Contents → Read and Write must be explicitly granted on the target repository.

- **The AI Agent output field wraps conversational responses:** When the agent responds conversationally, the output is wrapped in an `output` field. The Code Node must handle this gracefully with a null check, and an IF Node must split the upload path from the chat response path.
