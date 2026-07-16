# Documentation Architect Workflow

## Purpose

This document describes the current implementation state and architectural direction of the Documentation Architect workflow. It serves as the authoritative reference for all agents, nodes, and design decisions in this system.

---

## Current Implementation State

### Repository Tree Scanning

The workflow uses a recursive Git Tree scanner to retrieve the complete repository file tree via the GitHub Git Tree API. This replaced the previous flat document scanner that failed to detect nested documentation files.

- The old `Get Existing Docs List` node and `Format Existing Docs List` node have been removed.
- A `Build Documentation Index` node now creates a normalized `existingDocsList` from the repository tree.
- The Documentation Architect receives a complete documentation inventory regardless of folder depth.

**Why this matters:** Nested documentation files were previously invisible to the workflow. The recursive tree approach ensures the Documentation Architect can locate and extend any existing document rather than creating duplicates.

---

### Safe GitHub Retrieval

Repository Memory and ADR retrieval now return normalized outputs even when files do not exist (HTTP 404). Merge nodes always receive a consistent schema.

**Why this matters:** Before this change, missing files caused unpredictable schema shapes downstream. Deterministic outputs eliminate a class of runtime failures caused by inconsistent repository state.

---

### Cleaned Merge Context

All downstream nodes now receive deterministic data regardless of repository state. The merge context no longer carries optional or undefined fields that required defensive handling in prompts and expressions.

**Why this matters:** Reducing variability in the data received by AI agents directly reduces hallucination risk. Agents that receive clean, predictable input produce more consistent output.

---

### Improved Check Target File

The workflow now checks the generated `file_path` directly. Existing documentation can be located and updated regardless of folder depth.

**Why this matters:** Previously, documents in subdirectories could not be matched correctly during the update check, causing new files to be created instead of extending existing ones.

---

### ADR Numbering

The workflow calculates the next ADR number automatically and passes `nextADRNumber` to the Repository Intelligence agent. Repository Intelligence never calculates ADR numbers itself.

**Why this matters:** Delegating number calculation to the workflow layer removes a source of non-determinism from AI agents. Agents that calculate sequences can produce collisions or gaps.

---

### Responsibility Separation

| Agent | Responsibility |
|---|---|
| Documentation Architect | Documentation generation only |
| Repository Intelligence | Repository-level knowledge and ADR updates only |

This separation ensures each agent operates with a minimal, well-defined context. Agents are not asked to reason about concerns outside their domain.

---

## Architectural Direction: Next Improvements

The following improvements address the current goals of reducing hallucinations, reducing token usage, increasing determinism, and enforcing single-responsibility per agent. None of the items below duplicate completed work.

---

### 1. Output Schema Validation Node

**Problem:** AI agents currently return JSON that is consumed directly by downstream nodes. If an agent returns malformed JSON, an unexpected field, or omits a required field, the failure manifests late in the workflow and is difficult to diagnose.

**Proposed improvement:** Add a dedicated `Validate Agent Output` node after each AI agent response. This node enforces the expected output schema before any downstream node consumes the result. If validation fails, the workflow halts with a structured error rather than propagating bad data.

**Expected benefit:** Reduces silent failures. Provides a single point of enforcement for agent output contracts. Makes schema expectations explicit and version-controllable.

---

### 2. Prompt Versioning and Externalization

**Problem:** Agent prompts are currently embedded inside workflow nodes. When a prompt changes, there is no record of what changed, why it changed, or what the previous behavior was.

**Proposed improvement:** Store prompt templates as versioned documents in the repository under a dedicated path such as `prompts/`. Each prompt document includes its version, purpose, input variables, and output contract. Workflow nodes reference prompt versions rather than containing raw prompt text.

**Expected benefit:** Prompts become reviewable, diffable, and auditable. Prompt regressions can be identified and reverted. Token usage can be audited per prompt version.

---

### 3. Context Trimming Before Agent Invocation

**Problem:** Agents currently receive the full merged context including fields that are irrelevant to their specific task. Passing excess context increases token usage and increases the surface area for hallucination.

**Proposed improvement:** Add a `Prepare Agent Context` node before each AI agent invocation. This node selects only the fields required by that agent and discards the rest. Each agent receives a purpose-built context object.

**Expected benefit:** Reduced token consumption per invocation. Reduced hallucination risk from irrelevant context influencing agent reasoning. Clearer documentation of what each agent actually needs.

---

### 4. Documentation Impact Classification Validation

**Problem:** The Documentation Architect currently performs impact classification as part of its generation task. This combines two distinct reasoning tasks in a single agent call: classifying what changed and generating documentation for it.

**Proposed improvement:** Extract impact classification into a separate lightweight agent or deterministic node that runs before the Documentation Architect. The Documentation Architect receives a pre-computed impact classification and generates documentation only.

**Expected benefit:** Simpler, more focused Documentation Architect prompt. Impact classification logic can be tested and validated independently. Misclassifications are easier to identify when the classification step is isolated.

---

### 5. Affected Documents Execution

**Problem:** The `affected_documents` field returned by the Documentation Architect identifies documents that should also be updated, but the workflow does not currently act on this list. Secondary document updates are left to manual follow-up.

**Proposed improvement:** Add a post-generation step that reads `affected_documents` and enqueues update tasks for each listed file. Each update task passes the relevant context to the appropriate agent and processes the secondary document through the same validation and commit pipeline.

**Expected benefit:** Secondary documentation drift is eliminated automatically. The workflow enforces consistency across all affected documents rather than relying on human follow-up.

---

### 6. Idempotency Check Before Commit

**Problem:** If a workflow run is triggered with input that would produce no meaningful change to an existing document, the workflow currently proceeds to commit anyway. This creates noise in commit history.

**Proposed improvement:** Add a content comparison node after document generation. If the generated content is substantively identical to the existing file content, the workflow exits cleanly without creating a commit.

**Expected benefit:** Cleaner commit history. Reduced API calls to GitHub. Eliminates redundant documentation operations triggered by repeated or duplicate inputs.

---

### 7. Agent Retry and Fallback Policy

**Problem:** There is currently no defined behavior for what happens when an AI agent returns an invalid response or times out. The workflow has no retry logic and no fallback path.

**Proposed improvement:** Define an explicit retry and fallback policy for each agent node. On first failure, retry with the same input. On second failure, route to a structured error handler that logs the failure context and halts the workflow gracefully. Do not silently continue with partial data.

**Expected benefit:** Improved reliability under transient failures. Failure modes are explicit and observable. Partial or corrupted outputs do not propagate to the repository.

---

## Responsibility Map

| Node or Agent | Single Responsibility |
|---|---|
| Git Tree Scanner | Retrieve complete repository file tree |
| Build Documentation Index | Normalize tree into `existingDocsList` |
| Safe GitHub Retrieval | Fetch Repository Memory and ADR with normalized 404 handling |
| Prepare Agent Context | Trim context to only fields required by each agent |
| Documentation Architect | Generate documentation content only |
| Validate Agent Output | Enforce output schema contract |
| Check Target File | Locate existing file by `file_path` |
| Impact Classification | Classify change type before generation |
| Repository Intelligence | Update repository memory and ADR records only |
| Affected Documents Handler | Enqueue secondary document updates |
| Idempotency Check | Compare generated content to existing content |

---

## Related Documents

- `docs/architecture-overview.md`
- `docs/architecture-decisions.md`
- `docs/repository-memory.md`
- `docs/documentation-rules.md`
- `docs/writing-style-guide.md`
