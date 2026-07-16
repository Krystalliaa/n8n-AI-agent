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

### Agent Responsibility Separation

| Agent | Responsibility |
|---|---|
| Documentation Architect | Documentation generation only. Does not manage repository state, memory, or ADRs. |
| Repository Intelligence | Maintains repository memory. Creates ADR entries only for permanent architectural decisions. |

This separation ensures each agent operates with a minimal, well-defined context. Agents are not asked to reason about concerns outside their domain.

---

### Prompt and Schema Improvements

Documentation Architect and Repository Intelligence prompts have been updated. JSON output schemas are now stricter. Agent inputs are delivered through explicit, controlled fields rather than freeform context.

**Why this matters:** Strict schemas reduce the space in which agents can produce unexpected output. Explicit input fields prevent agents from reasoning over data they were not intended to see.

---

### Current Validation Status

| Area | Status |
|---|---|
| Documentation generation | Working |
| Repository Intelligence | Working |
| ADR number injection validation | Refinement in progress |
| Repository Memory scope enforcement | Refinement in progress |
| Output validation | Refinement in progress |

---

## Active Reliability Improvements

The following improvements target the three remaining refinement areas identified in the current validation result. None of the items below duplicate completed work.

---

### 1. ADR Number Injection Validation

**Problem:** The workflow passes `nextADRNumber` to Repository Intelligence, but there is currently no enforcement that the agent uses the injected value in its output. An agent that ignores the field and generates its own number would produce an ADR with an incorrect sequence position.

**Proposed improvement:** Add a post-agent validation step that reads the ADR entry produced by Repository Intelligence and asserts that the ADR number present in the output matches the `nextADRNumber` value passed by the workflow. If the values do not match, the workflow halts with a structured error before any commit is attempted.

**Expected benefit:** The ADR sequence remains consistent even if the agent prompt drifts or a model update changes the agent's behavior. The validation node acts as a contract enforcement point that does not rely on prompt instructions alone.

---

### 2. Repository Memory Scope Enforcement

**Problem:** Repository Intelligence is currently responsible for deciding what qualifies as a permanent architectural decision worth storing in Repository Memory. Without an explicit boundary, the agent may store future roadmap items, aspirational descriptions, or implementation plans that have not yet been completed. This pollutes the memory with unverified information that future agents will treat as fact.

**Proposed improvement:** Enforce the boundary in two layers.

- **Prompt layer:** The Repository Intelligence prompt must explicitly instruct the agent that Repository Memory records only completed, verified implementation facts. Roadmap items, planned improvements, and candidate changes must never be written to memory.
- **Workflow layer:** Add a `Validate Memory Payload` node that inspects the memory update produced by Repository Intelligence before it is committed. The node checks for markers of future intent such as modal language ("will", "planned", "intended", "future") and rejects payloads that contain them.

**Expected benefit:** Repository Memory remains a reliable source of truth for completed work. Downstream agents that consume memory can trust that everything in it reflects actual system state rather than aspirational state.

---

### 3. Structured Output Validation

**Problem:** Agent outputs are currently consumed by downstream nodes without a systematic check that all required fields are present, all values are of the correct type, and no unexpected fields have been injected. A missing `file_path`, a null `file_content`, or an unexpected extra field can cause downstream failures that are difficult to trace back to the agent response.

**Proposed improvement:** Add a dedicated `Validate Agent Output` node after each AI agent response. The node applies a strict schema check against the expected output contract for that agent. Required fields are asserted as present and non-empty. Field types are verified. Unknown fields are flagged. Failures route to a structured error handler rather than continuing the workflow.

The output contracts for each agent are:

**Documentation Architect output contract:**

| Field | Type | Required |
|---|---|---|
| `file_path` | string, non-empty | Yes |
| `file_content` | string, non-empty | Yes |
| `commit_message` | string, non-empty | Yes |
| `affected_documents` | array of strings | Yes (may be empty) |

**Repository Intelligence output contract:**

| Field | Type | Required |
|---|---|---|
| `repository_memory_update` | object | Yes |
| `adr_entry` | object or null | Yes |
| `adr_number` | integer, matches `nextADRNumber` | Conditional on ADR creation |

**Expected benefit:** Agent output failures are caught immediately at a known validation point. Downstream nodes operate only on data that has passed the contract check. The output contracts are explicit and can be updated independently of prompt text.

---

## Responsibility Map

| Node or Agent | Single Responsibility |
|---|---|
| Git Tree Scanner | Retrieve complete repository file tree |
| Build Documentation Index | Normalize tree into `existingDocsList` |
| Safe GitHub Retrieval | Fetch Repository Memory and ADR with normalized 404 handling |
| Documentation Architect | Generate documentation content only |
| Validate Agent Output | Enforce output schema contract per agent |
| Check Target File | Locate existing file by `file_path` |
| Repository Intelligence | Update repository memory and create ADR entries for completed decisions only |
| Validate Memory Payload | Reject memory updates containing future-tense or unverified content |
| ADR Injection Validator | Assert that ADR output number matches workflow-supplied `nextADRNumber` |

---

## Design Principles

The following principles govern all future changes to this workflow.

- **Agents receive only what they need.** Context trimming is applied before every agent invocation.
- **Agents are never trusted to enforce their own contracts.** All agent output is validated by a workflow node before downstream consumption.
- **Repository Memory records only completed facts.** Planned or aspirational content is never written to memory.
- **Sequence values are never delegated to agents.** ADR numbers, version numbers, and any other sequences are calculated by the workflow and injected as explicit inputs.
- **Failures halt explicitly.** No failure mode silently continues with partial or invalid data.

---

## Related Documents

- `docs/architecture-overview.md`
- `docs/architecture-decisions.md`
- `docs/repository-memory.md`
- `docs/documentation-rules.md`
- `docs/writing-style-guide.md`
