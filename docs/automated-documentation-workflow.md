# Automated Documentation Workflow (n8n)

## Introduction

I built an n8n workflow that automates the generation of project documentation from user-submitted notes. The workflow uses a Documentation Architect AI agent to produce structured, portfolio-ready markdown files and commits them directly to a GitHub repository. This project was motivated by a problem I kept running into: documentation becomes outdated because writing and maintaining it manually is slow, inconsistent, and easy to deprioritise. I wanted to turn documentation into an automated part of my development lifecycle rather than an afterthought.

This document captures the full journey of building that workflow — the goals, the architecture, the technical decisions, and the many challenges I encountered and resolved along the way.

---

## Objectives

- Accept project notes from a user via an n8n Chat Trigger and produce a structured markdown documentation file automatically.
- Classify the project type (n8n, homelab, AI agent, observability) using keyword matching so the correct template is selected.
- Retrieve writing style guides, documentation rules, templates, and examples from a Supabase knowledge base.
- Fetch existing documentation files from the GitHub repository to prevent duplicate or conflicting entries.
- Load repository memory and architecture decision records (ADRs) from the repository to give the AI agent high-level project context.
- Pass all retrieved context to a Documentation Architect AI agent that generates a JSON response containing the file path, file content, and commit message.
- Validate the AI output before any repository operation to prevent malformed or hallucinated content from being published.
- Determine whether the target documentation file already exists and either create it or update it (including SHA retrieval for updates).
- After a successful commit, trigger a Repository Intelligence Agent to evaluate whether the change warrants an update to repository memory or architecture decision records.
- Commit the result back to the GitHub repository and return a response to the user through the Chat Trigger.

---

## Technologies Used

- **n8n** — Workflow automation platform (self-hosted via Docker) used to orchestrate all pipeline steps.
- **OpenAI** — AI provider powering the Documentation Architect agent and the Repository Intelligence Agent.
- **GitHub API** — Used for listing files, checking file existence, creating files, and updating files (with SHA retrieval).
- **raw.githubusercontent.com** — Used for simple retrieval of known files (repository memory, ADRs) without authentication overhead.
- **Supabase** — Stores the knowledge base: documentation rules, writing style guidelines, project classifications, templates, and examples.
- **n8n Chat Trigger** — Entry point for user-submitted project notes.
- **n8n Code Nodes** — Used for classification logic, context aggregation, fallback handling, and output validation.
- **n8n HTTP Request Nodes** — Used for GitHub API calls and raw URL retrieval.
- **n8n Merge Node** — Used to aggregate parallel context retrieval branches before sending context to the Documentation Architect agent.

---

## Architecture Overview

The workflow follows a linear pipeline with a parallel context retrieval stage and a post-commit intelligence stage.

```
User
|
Chat Trigger
|
Input Normalization
|
Project Classification (keyword matching)
|
Knowledge Base Retrieval (Supabase)
|
Parallel Context Retrieval
├── Existing Documentation Scanner (GitHub API)
├── Repository Memory Loader (raw URL)
├── ADR Loader (raw URL)
└── Template Selector (based on classification)
|
Context Aggregation (Merge node)
|
Documentation Architect AI Agent
|
Output Validation (JSON schema, required fields, empty detection)
|
Create vs Update Decision (check if file already exists)
|
GitHub Repository Operation (Create or Update with SHA retrieval if updating)
|
Commit / Publish
|
Repository Intelligence Agent
├── Evaluate repository memory update
└── Evaluate ADR update
|
Response to User (Chat Trigger response node)
```

The knowledge base is intentionally stored in Supabase rather than hardcoded into prompts. This keeps prompts shorter, allows documentation rules and templates to evolve independently of the workflow, and supports multiple documentation styles without modifying any workflow nodes.

Context retrieval is selective by design. The project classification step determines which template and which examples are relevant, so the Documentation Architect agent only receives information it needs. This reduces token usage, controls cost, and prevents attention dilution inside the AI model.

The two AI agents have strictly separated responsibilities. The Documentation Architect generates documentation and returns a file path, content, and commit message. It never touches the repository. The Repository Intelligence Agent evaluates the approved change after it has been committed and decides whether repository memory or architecture decision records require updating. It never generates documentation.

---

## Implementation Process

### Step 1 — Chat Trigger and Input Normalization

I used an n8n Chat Trigger as the entry point. When a user submits project notes, the workflow normalizes the input into a structured JSON object that all downstream nodes can reference consistently.

### Step 2 — Project Classification

A Code node performs keyword matching against the user input to assign a project type (n8n, homelab, AI agent, observability). This classification drives template selection and example retrieval in subsequent steps.

### Step 3 — Knowledge Base Retrieval from Supabase

The workflow queries Supabase to retrieve the documentation rules, writing style guide, classification rules, the relevant template, and any matching examples. I designed this retrieval to always return a value — even if a Supabase entry is missing, the workflow falls back to a default so downstream nodes never receive undefined.

### Step 4 — Parallel Context Retrieval

Four retrieval branches run in parallel:

- **Existing Documentation Scanner** — Calls the GitHub Contents API to list files in the `/docs` directory, allowing the agent to avoid generating content that duplicates an existing file.
- **Repository Memory Loader** — Fetches `repository-memory.md` from the repository using a raw.githubusercontent.com URL.
- **ADR Loader** — Fetches `architecture-decisions.md` the same way.
- **Template Selector** — Pulls the correct template from Supabase based on the classification result.

For the repository memory and ADR files, I switched from the GitHub Contents API to raw URLs because they return plain text with no base64 decoding required. For dynamic operations — listing files, checking existence, and committing — the GitHub API remained necessary.

### Step 5 — Context Aggregation

A Merge node and a subsequent Code node combine all retrieved context into a single JSON object. The Code node also applies fallback defaults for any branch that produced no output, ensuring the Documentation Architect agent always receives a complete context object.

### Step 6 — Documentation Architect AI Agent

The merged context object is passed to the Documentation Architect AI Agent. The system prompt defines the agent's role, responsibilities, restrictions, and output schema. The user message contains the project notes and all retrieved context injected via n8n expressions (`$json.styleGuide`, `$json.docRules`, `$json.classification`, `$json.templateContent`, `$json.exampleContent`, `$json.existingDocsList`). The agent returns a JSON object with three fields: `file_path`, `file_content`, and `commit_message`.

The agent is explicitly restricted from touching the repository, calling APIs, retrieving SHA values, or deciding whether to create or update files. Those responsibilities belong to the workflow.

### Step 7 — Output Validation

Before any repository operation, a Code node validates the AI output:
- Confirms the response is valid JSON.
- Checks that `file_path`, `file_content`, and `commit_message` are all present and non-empty.
- Detects and rejects empty or whitespace-only content.

This prevents malformed output, missing file paths, and low-quality documentation from being committed to the repository.

### Step 8 — Create vs Update Decision

The workflow calls the GitHub Contents API to check whether the target file already exists. If it does not exist, the workflow creates it. If it does exist, the workflow first retrieves the current file's SHA, then performs an update using that SHA.

### Step 9 — GitHub Commit

The appropriate GitHub operation is executed and the commit is published to the repository.

### Step 10 — Repository Intelligence Agent

After a successful commit, the Repository Intelligence Agent evaluates the approved change. It receives the current repository memory, the current ADR history, and the committed file path, commit message, and content. It decides independently whether the change warrants appending to repository memory or creating a new ADR entry. It returns a structured JSON response and does not modify the repository directly — the workflow handles any resulting write operations.

The agent only updates memory for new projects, major features, important troubleshooting knowledge, significant lessons, or workflow architecture changes. It only creates ADR entries for architecture changes, storage changes, integration changes, repository structure changes, or workflow design changes. Normal documentation edits do not trigger updates.

### Step 11 — Response to User

The workflow returns a confirmation response to the user through the Chat Trigger response node.

---

## Challenges Encountered

### 1. Environment Variables Not Accessible in Code Nodes

I used `process.env.GITHUB_OWNER` inside a Code node and received an error. n8n does not support `process.env` — it exposes environment variables through `$env`.

### 2. `$http` Not Recognised in Code Nodes

I wrote `$http.get(...)` in a Code node and received `Cannot find name '$http'`. The `$http` object is only available when the **Allow HTTP requests** toggle is explicitly enabled in the Code node settings.

### 3. GitHub API Returns 404 for Documentation Files

API calls to the GitHub Contents endpoint returned 404 even though the files existed. The likely causes were incorrect branch parameter, path casing issues, and missing handling for nonexistent files. I switched known-file retrieval to raw.githubusercontent.com URLs, which return plain text and require no authentication for public repositories. Dynamic operations continued to use the GitHub API.

### 4. Merge Node Failing Because Upstream Decode Node Never Ran

A decode node was skipped when its upstream HTTP node failed (even with Continue on Fail enabled). Because the decode node was set to Run Once for Each Item, it never executed when the HTTP node returned zero items. The Merge node then threw `Cannot assign to read only property 'name'` when trying to reference the skipped node. I replaced the HTTP and decode pair with a single Code node that always returns an object, using try/catch and fallback defaults.

### 5. AI Agent Node — "No prompt specified" and "non-whitespace text" Errors

The user message field used `{{ $('Merge Context For Agent').item.json.userNotes }}`. The `.item` accessor is not valid in n8n expressions. I changed the expression to `{{ $json.userNotes }}`. A second error occurred because the userNotes field was empty, causing the model to reject the message. I ensured the field is always populated before the agent node executes.

### 6. Merge Node Referencing Deleted Nodes

After disabling old nodes, the Merge node still referenced them by name and threw errors. I deleted the old nodes entirely and updated all references to point to the new node names.

### 7. Missing Knowledge Base Entries in Supabase

When Supabase entries were absent, the agent received empty strings and produced poor documentation. I ensured default entries exist for all required keys and added fallback logic in the workflow so empty results never reach the agent without a placeholder value.

### 8. GitHub Update API Requires SHA

Updating an existing file through the GitHub Contents API requires the SHA of the current file version. Initial attempts to update files without this SHA returned `sha wasn't supplied` errors. The solution was to always perform a GET request before any update, extract the SHA from the response, and include it in the PUT request body.

### 9. n8n Item Execution Model Confusion

Several errors came from misunderstanding how n8n passes data between nodes. `$input.item`, `$('Node Name').item.json`, `.first()`, and Run Once for Each Item behave differently depending on context. I standardised the workflow to use `$input.first()` for single-context aggregation nodes, `$json` inside AI nodes, and Run Once for All Items for aggregation logic.

### 10. Chat Trigger Response Configuration

The workflow executed without error but the user interface received no response. The Chat Trigger was not configured correctly. I changed the Response Mode to Using Response Nodes.

### 11. Context Window and Token Optimisation

Early designs passed too much information to the agent — entire documentation directories, large templates, and previous documents — increasing token cost, latency, and the risk of the model losing focus. I redesigned retrieval to be selective: classification determines which template and examples are loaded, repository memory provides only high-level context, and existing docs are used specifically for duplication prevention.

### 12. AI Hallucination Prevention

The Documentation Architect agent was designed with explicit guardrails: a strict role definition, a required JSON output schema, and a clear rule that unknown information must become `[TO BE DOCUMENTED]` rather than being invented. The agent is explicitly restricted from inventing file paths, technologies, architecture components, or solutions not provided in the input.

### 13. Output Validation Before Publishing

Without validation, malformed JSON, missing required fields, or empty content could be committed to the repository. I added a validation Code node that checks JSON structure, required fields, and content length before any GitHub operation proceeds.

### 14. Create vs Update Decision Logic

The workflow needed to distinguish between creating a new file and updating an existing one, since these are different GitHub API operations with different requirements. I added an existence check step that determines which path to follow.

### 15. Task Runner Timeout

Large AI executions occasionally exceeded the default execution timeout, producing `Task request timed out` errors. The combined cost of multiple external API calls, AI generation, and validation exceeded the default runner limit. I adjusted the runner timeout configuration via `N8N_RUNNERS_TASK_REQUEST_TIMEOUT` and optimised workflow execution to reduce unnecessary steps.

### 16. Security Considerations

GitHub tokens are stored in n8n credentials and environment variables, not passed to the AI model. The AI agents receive only the context necessary to perform their task — no secrets, no credentials, no internal system details.

### 17. Separating Agent Responsibilities

An early design considered having a single agent handle both documentation generation and repository memory updates. I separated these into two distinct agents because combining them created prompt complexity and increased the risk of one concern polluting the other. The Documentation Architect focuses entirely on producing documentation. The Repository Intelligence Agent focuses entirely on evaluating whether the committed change affects repository-level knowledge.

### 18. Repository Intelligence Agent Prompt Design

The Repository Intelligence Agent receives the full content of the committed file alongside the current repository memory and ADR history. The challenge was defining clear criteria for when an update is warranted. Without precise rules, the agent would update memory for every documentation change. I defined explicit inclusion criteria for both memory updates and ADR entries, and added an explicit instruction that normal documentation edits do not qualify.

---

## Troubleshooting

### `process.env` undefined in Code nodes
**Cause:** n8n does not support `process.env`.
**Fix:** Replace all `process.env.VARIABLE` references with `$env.VARIABLE`.

### `Cannot find name '$http'` in Code nodes
**Cause:** The Allow HTTP requests toggle was not enabled.
**Fix:** Open the Code node settings and enable Allow HTTP requests.

### GitHub Contents API returning 404
**Cause:** Incorrect branch parameter, path casing mismatch, or missing handling for nonexistent files.
**Fix:** Switch to raw.githubusercontent.com URLs for known static files. Continue using the GitHub API for dynamic operations.

### Merge node throwing `Cannot assign to read only property 'name'`
**Cause:** An upstream decode node was skipped because its HTTP node returned zero items, and the Merge node tried to reference it.
**Fix:** Replace the HTTP and decode pair with a single Code node that uses try/catch and always returns an object with fallback defaults.

### `$('Node Name').item.json` throwing errors in AI node expressions
**Cause:** `.item` is not a valid accessor in n8n expression context.
**Fix:** Use `$json.fieldName` inside AI nodes to reference the current input item.

### `sha wasn't supplied` on GitHub file update
**Cause:** The GitHub Contents API requires the current file SHA for any update operation.
**Fix:** Perform a GET request to the file endpoint before updating, extract the `sha` field, and include it in the PUT request.

### Chat Trigger producing no response to user
**Cause:** Response Mode was not configured correctly.
**Fix:** Set Response Mode to Using Response Nodes in the Chat Trigger settings.

### Task request timed out
**Cause:** The combined execution time of API calls, AI generation, and validation exceeded the default runner timeout.
**Fix:** Increase the runner timeout via `N8N_RUNNERS_TASK_REQUEST_TIMEOUT` and reduce unnecessary workflow steps.

---

## Lessons Learned

- I learned that n8n Code nodes require `$env.VARIABLE` rather than `process.env.VARIABLE` — the execution environment is not a standard Node.js process.
- I learned that `$http` inside a Code node requires the Allow HTTP requests toggle to be explicitly enabled; it is not available by default.
- I learned that when a node may receive zero items, it must be set to Run Once for All Items or handle emptiness explicitly in code — otherwise downstream nodes that reference it will fail.
- I learned that downstream nodes must always receive defined values. Providing fallback defaults at every context retrieval step is not optional — it is what makes the pipeline resilient.
- I learned that raw.githubusercontent.com is the right tool for simple retrieval of known public files, but the GitHub Contents API is necessary for listing, existence checking, and write operations.
- I learned that GitHub file updates require optimistic concurrency control through SHA validation. This is fundamentally different from overwriting a file — GitHub requires proof that you are updating the version you think you are updating.
- I learned that AI agents perform better when given structured, relevant context rather than the maximum available information. Selective retrieval improves output quality and reduces cost.
- I learned that separating the system prompt (role, rules, restrictions) from the user message (project-specific context) reduces prompt duplication and produces more reliable AI output.
- I learned that validating AI output before any side effect — in this case, a repository write — is essential. The validation step is what separates an experimental prototype from a production-ready workflow.
- I learned that n8n's item-based execution model requires deliberate design. AI pipelines should normalize all context into a single JSON object before it reaches the agent node.
- I learned that self-hosted n8n requires explicit timeout configuration for workflows that combine multiple external calls with AI generation. Default timeouts are designed for traditional automation, not AI pipelines.
- I learned that giving each AI agent a single, clearly scoped responsibility produces better results than combining multiple concerns into one agent. The Documentation Architect and the Repository Intelligence Agent each perform one job well.
- I learned that treating security as a first-class concern from the start — not retrofitted — means credentials never need to be removed from prompt history or logs, because they were never there.

---

## Future Improvements

- Implement webhook-based triggering so documentation generation can be initiated directly from GitHub events (push, pull request, release) in addition to manual Chat Trigger submission.
- Add a review step that surfaces the generated documentation to the user for approval before committing, rather than committing automatically.
- Expand the Supabase knowledge base to support multiple documentation styles and output formats, enabling the same pipeline to generate READMEs, changelogs, and runbooks in addition to portfolio documents.
- Implement automatic detection of stale documentation by comparing repository memory with existing doc files and flagging entries that have not been updated recently.
- Add observability to the workflow itself — execution logs, token usage tracking, and failure alerting — so the pipeline can be monitored and improved over time.
- Explore prompt caching strategies to reduce latency and token cost as the knowledge base and context size grow.
- Extend the Repository Intelligence Agent to proactively suggest documentation gaps based on repository memory, not only react to committed changes.
