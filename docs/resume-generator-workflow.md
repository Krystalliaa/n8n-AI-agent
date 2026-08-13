# Resume Generator Workflow

## Overview

This document describes the architecture, components, and lessons learned for the resume generation pipeline built on n8n Self-Hosted.

The pipeline consists of two primary agents:

- **KB Agent — Master**: Knowledge Base Auditor & Editor Agent
- **Documentation Architect**: Transforms project notes into portfolio documentation

---

## KB Agent — Master

### Purpose

The KB Agent is an n8n workflow acting as a Knowledge Base Auditor and Editor for the resume generation pipeline. It manages structured career data files, performs audits, discovers skill gaps, and enforces safe write operations through a two-step approval flow.

**Platform:** n8n Self-Hosted (v2.33.7) with filesystem access at `/data/resume-kb/`

---

## Knowledge Base File Structure

| File | Type | Contents |
|---|---|---|
| `master_kb.yaml` | YAML | Profile, `skill_inventory` (array), `work_experience`, `projects`, `certifications`, `education` |
| `taxonomy_dictionary.json` | JSON | `canonical_name → [synonyms]` (key = canonical name, not `skill_id`) |
| `experience_graph.json` | JSON | `skills_to_jobs`, `skills_to_projects` mappings |
| `achievement_index.json` | JSON | Claims with `associated_skills` |
| `evidence_index.json` | JSON | `claim → source` mapping |
| `ats_keywords.json` | JSON | Job families → keywords mappings |
| `career_timeline.json` | JSON | Chronological entries |
| `generation_rules.yaml` | YAML | Prompt templates for resume generation |
| `gold_standard.md` | Markdown | Reference resume for quality comparison |
| `resume_profiles/*.yaml` | YAML | Role-specific skill selections |

---

## Implemented Features

### Feature 1: Synonym Discovery

- Reads `skill_inventory` in array format: `[{skill_id, canonical_name, category, ...}]`
- Checks whether each `canonical_name` exists as a key in `taxonomy_dictionary.json`
- When missing, calls Anthropic Claude (claude-sonnet-4-6) to generate 5 ATS-friendly synonyms
- **Smart Cache:** `/data/resume-kb/.cache/synonym_proposals.json` — prevents redundant LLM calls for already-processed skills
- **Approval Flow:** Two-step process: proposals → pre-flight diff → apply

### Feature 2: Skill Gap Analysis

- Reads Job Descriptions from `/data/resume-kb/jds/` (`.txt`, `.md`, `.json`)
- Performs frequency analysis: how many JDs reference each term
- Compares JD terms against existing skills in `master_kb.yaml`
- LLM ranks and categorizes gaps: `core_must_have` / `nice_to_have` / `emerging`
- Approval flow triggers addition to `master_kb.yaml`

### Feature 3: Broken Link Detector

- Cross-references `linked_job_ids`, `linked_project_ids`, `associated_skills`
- Validates against `experience_graph.json` and `achievement_index.json`
- Read-only report — no write operations

### Feature 4: Resume Quality Audit

- Compares `gold_standard.md` against generated resumes from `/data/resume-kb/generated/`
- Metrics: density, keyword overlap, section comparison
- Read-only report — no write operations

### Feature 5: Test Suite (7/7 PASS)

Test coverage:

1. YAML Validity
2. JSON Validity
3. Broken Links
4. Taxonomy Coverage
5. Orphaned Taxonomy
6. Claim Metrics (checks both job and project claims)
7. JD Storage

Tests run via Execute Command node using the **heredoc pattern** to avoid `js-yaml` sandbox issues.

### Feature 6: Supabase Integration (In Progress)

- Table: `project_docs` — book-like chunking for cost-efficient LLM context
- Table: `kb_snapshots` — versioning
- Intended flow: user requests architect context → reads from Supabase → proposes additions to KB

> **Status:** Schema and chunking design are defined. Tables were empty at time of last implementation; data population is pending.

---

## Safety Architecture

| Layer | Implementation |
|---|---|
| Atomic Writes | `cp backup → write temp → validate → mv` |
| Backups | `filename.backup.{timestamp}` |
| Audit Logs | `/data/resume-kb/.audit/change_log.jsonl` (append-only) |
| State Machine | `/data/resume-kb/.state/session.json` (approval persistence) |
| Two-Step Approval | Proposals → Pre-flight (backup path + exact diff) → Confirm → Apply |
| Post-Write Validation | YAML/JSON validated after every write before success is reported |

---

## Architecture Decisions

### State Machine over Wait Node

n8n Chat Trigger executions are stateless — each message is a new execution with no memory of prior state. A `Wait` node was not implemented. The workaround is a persistent state file at `/data/resume-kb/.state/session.json` that stores the current approval step and which feature is awaiting confirmation.

**Routing key format:** `step|workflow` (e.g., `awaiting_proposal_review|synonyms`) — used to distinguish which feature is pending approval.

**Router priority rule:** Commands are evaluated **before** state. If a user sends a new command while an approval is pending, state resets to `idle`. This prevents stale state from intercepting fresh commands.

### Anthropic Claude over Agent Node

The basic Chat Model node is used instead of the Agent node. The Agent node added unnecessary complexity and made output control harder. The basic LLM node with structured prompts is sufficient for synonym generation and skill ranking.

### Heredoc Pattern for Execute Command

Inline `node -e "..."` in Execute Command nodes fails with nested quotes, newlines, or `$` variables. The solution is to write the script to a temp file first:

```bash
cat > /tmp/script.js << 'EOF'
// code here
EOF
node /tmp/script.js
```

### Taxonomy Matching by canonical_name

`taxonomy_dictionary.json` uses `canonical_name` as the key (e.g., `"Python": [...]`), not `skill_id`. All matching must use `canonical_name`:

```javascript
const hasTaxonomy = taxonomy[skill.canonical_name] !== undefined;
```

---

## Lessons Learned

This section documents all significant problems encountered during construction to prevent recurrence.

### 1. n8n Code Node Sandbox & js-yaml

**Problem:** n8n (v2.33.7) runs Code Nodes in a sandboxed VM. `require('js-yaml')` fails with:
```
Cannot assign to read only property 'constructor' of object 'Error'
```

**What worked:**
- Setting `NODE_FUNCTION_ALLOW_EXTERNAL=*` (or `=js-yaml`) in the environment
- Using absolute path: `require('/data/resume-kb/node_modules/js-yaml')`
- Fallback: Execute Command node with `node -e "..."` for YAML parsing outside the sandbox

**What did not work:**
- Global install (`npm install -g js-yaml`) — n8n does not see globally installed packages
- Built-in `require('js-yaml')` without `ALLOW_EXTERNAL`

---

### 2. Chat Trigger Input Format

**Problem:** `$input.first().json.chatInput` can be either a plain string or an object with a `.message` property. When the router received an object, `chatMsg` resolved to `undefined`, causing all intents to fall through to `unknown`.

**Solution:**

```javascript
const rawChatInput = $input.first().json.chatInput;
const chatMsg = (typeof rawChatInput === 'string'
  ? rawChatInput
  : (rawChatInput?.message || '')).trim();
```

---

### 3. State Machine and Stale State

**Problem:** Stale state in `session.json` caused fresh commands to be intercepted by approval logic. For example, sending `test` while state was `awaiting_proposal_review` routed to the approval handler instead of the test suite.

**Fix:** Router checks commands first, then state. Any new recognized command resets state to `idle` before processing.

**Additional risk:** `/tmp/n8n_pending_synonyms.json` persists between runs but is lost on restart. Persistent state must live in `/data/resume-kb/.state/`, not `/tmp/`.

---

### 4. LLM Output Parsing

**Problem:** Claude wraps JSON responses in markdown code fences:

````
```json
{ "proposals": [...] }
```
````

`JSON.parse()` fails without stripping the fences.

**Solution:**

```javascript
const clean = llmRaw.replace(/```json\s*/gi, '').replace(/\s*```/g, '');
const match = clean.match(/\{.*\}/s);
```

**Additional issue:** Large outputs (>4K tokens) are truncated. Use compact format or batch requests.

---

### 5. YAML Structure Mismatch

**Problem:** `master_kb.yaml` stores skills as a `skill_inventory` array, not as a `skills` object. Code Nodes that assumed object structure returned empty arrays (`missing_skills: []`).

**Fix:** Add a diagnostic node that inspects `Object.keys(masterKb)` and adapts parsing to the actual structure before any processing.

---

### 6. Approval Switch Node Left Empty

**Problem:** The Approval Step Switch node was created but left without any rules defined, breaking all approval routing silently.

**Fix:** Always verify Switch node rules are populated after any import or workflow restructure.

---

### 7. Router Priority: State Before Commands

**Problem:** The router originally checked `session.json` state before evaluating the incoming command. A stale `awaiting_proposal_review` state caused all subsequent messages — including `test` and `gaps` — to be routed to approval logic.

**Fix:** Commands are always evaluated first. State is only consulted when no recognized command is detected.

---

### 8. Execute Command Output Truncation

**Problem:** `stdout` from Execute Command nodes is truncated for large outputs (e.g., the full `master_kb.json`). The downstream parse node then fails on incomplete JSON.

**Mitigation:** Write large outputs to a file and read the file separately, rather than relying on stdout.

---

### 9. Duplicate YAML Keys

**Problem:** `master_kb.yaml` contained `claims:` twice. Most YAML parsers either fail silently (last value wins) or throw a parse error.

**Fix:** Validate YAML structure with a linter before loading. The test suite's YAML Validity check catches this.

---

### 10. Supabase Return Format

**Problem:** The Supabase "Get Many" node returns `{data: [...]}`, not an array of n8n items. Using `$input.all()` did not return the expected records.

**Fix:** Access `$input.first().json.data` to retrieve the array.

---

### 11. Metrics Calculation Scope

**Problem:** Claim metrics initially only checked job-linked claims, missing project-linked claims. This produced incomplete coverage reports.

**Fix:** Claim metrics now check both `achievement_index.json` entries linked to jobs and those linked to projects.

---

### 12. Two-Step Approval UX Complexity

**Problem:** The pre-flight diff step (show backup path + exact diff before final apply) adds a second wait cycle. This doubles the state machine complexity and the number of Switch node branches required.

**Why it was kept:** The pre-flight step is a safety requirement. Users must see the exact diff and backup location before any write is confirmed. The complexity is accepted in exchange for auditability.

---

### 13. Webhooks vs Chat Trigger Response Mode

**Problem:** Using "Using Response Nodes" mode requires a `Respond to Webhook` node connected at the end of **every** branch. Missing connections cause branches to hang without returning a response.

**Fix:** Audit all Switch node branches after any restructure to confirm each terminal node is a `Respond to Webhook`.

---

### 14. Cache Invalidation

**Problem:** `/tmp/n8n_pending_synonyms.json` is lost on container restart, causing the workflow to re-propose synonyms that were already reviewed.

**Fix:** All cache files are stored at `/data/resume-kb/.cache/synonym_proposals.json` for persistence across restarts.

---

## Path Reference

| Path | Purpose |
|---|---|
| `/data/resume-kb/` | Root knowledge base directory |
| `/data/resume-kb/.state/session.json` | Approval state persistence |
| `/data/resume-kb/.audit/change_log.jsonl` | Append-only audit log |
| `/data/resume-kb/.cache/synonym_proposals.json` | LLM synonym cache |
| `/data/resume-kb/jds/` | Job description input files |
| `/data/resume-kb/generated/` | Generated resume output files |
