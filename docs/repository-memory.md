# Repository Memory

## Project: Automated Documentation Workflow (n8n)

An n8n workflow that automates generation of portfolio-ready markdown documentation from user-submitted project notes. The workflow uses a Documentation Architect AI agent (OpenAI) to produce structured markdown files and commits them directly to a GitHub repository via the GitHub Contents API.

**Core pipeline:** Chat Trigger -> Input Normalization -> Project Classification (keyword matching) -> Knowledge Base Retrieval (Supabase) -> Parallel Context Retrieval (GitHub API, raw.githubusercontent.com, Supabase templates) -> Context Aggregation (Merge node) -> Documentation Architect AI Agent -> Output Validation -> Create vs Update Decision -> GitHub Commit -> Chat Trigger Response.

**Key technologies:** n8n (self-hosted via Docker), OpenAI, GitHub API, raw.githubusercontent.com, Supabase, n8n Chat Trigger, Code nodes, HTTP Request nodes, Merge node.

**Important troubleshooting knowledge:**
- n8n Code nodes use `$env.VARIABLE` not `process.env.VARIABLE`
- `$http` in Code nodes requires Allow HTTP requests toggle to be enabled
- GitHub Contents API file updates require current file SHA (retrieve via GET before PUT)
- Nodes that may receive zero items must use Run Once for All Items or handle emptiness explicitly
- Raw.githubusercontent.com is preferred for static known-file retrieval; GitHub API required for listing, existence checks, and writes
- AI node expressions use `$json.fieldName` not `$('Node Name').item.json`
- Chat Trigger response requires Response Mode set to Using Response Nodes
- Self-hosted n8n AI pipelines may require increased `N8N_RUNNERS_TASK_REQUEST_TIMEOUT`

**Key design decisions:** Knowledge base stored in Supabase (not hardcoded in prompts); selective context retrieval driven by project classification; output validation before any repository write; system prompt separated from user message for AI agent; GitHub credentials stored in n8n credentials only, never passed to AI model.

## Automated Documentation Workflow (n8n)

An n8n workflow that automates generation of project documentation from user-submitted notes. Uses a dual-agent architecture: a Documentation Architect AI agent that produces structured markdown files and commits them to GitHub, and a Repository Intelligence Agent that evaluates committed changes and maintains repository memory and ADRs. Context is retrieved selectively from a Supabase knowledge base (rules, templates, style guides, examples) and from GitHub/raw.githubusercontent.com (existing docs, repository memory, ADRs). Output is validated before any repository write. The workflow is self-hosted via Docker and uses OpenAI as the AI provider.

Key technical decisions: selective context retrieval driven by keyword-based project classification; raw.githubusercontent.com for static file retrieval; GitHub Contents API for dynamic operations; SHA-based optimistic concurrency for file updates; strict agent responsibility separation; output validation before commit; n8n runner timeout configured via N8N_RUNNERS_TASK_REQUEST_TIMEOUT.

The automated documentation workflow has migrated from a large static context injection model to a selective knowledge retrieval architecture. A Supabase knowledge base stores reusable documentation intelligence. Selected repository files are synchronized into Supabase as structured knowledge entries with fields: key, path, content, summary, category, project_type, priority, always_load, tags, updated_at. The workflow classifies the project type before building knowledge context and retrieves only the knowledge entries required for each task. Global entries (style_guide, documentation_rules, repository_governance) are always loaded. Project-specific templates and examples are loaded based on classification. The Documentation Architect agent receives only the relevant subset of knowledge rather than the full repository context. All GitHub repository operations are performed by n8n nodes, not AI agents.

The automated documentation workflow uses a retrieval-based knowledge architecture backed by Supabase. Selected repository files (style guide, documentation rules, project classification rules, repository governance, templates, examples) are synchronized into Supabase as structured knowledge entries. Each entry has a key, path, content, AI-generated summary, category, project type, priority, always_load flag, tags, and updated_at timestamp. The workflow classifies projects by type (n8n, homelab, ai_agent, observability, cybersecurity) using keyword matching, then retrieves only the relevant knowledge entries for each documentation task. Global entries (style_guide, documentation_rules, repository_governance) are always loaded. Project-specific templates and examples are loaded based on classification. All GitHub file operations are performed by n8n nodes exclusively. AI agents handle only reasoning and content generation.

## Automated Documentation Workflow (n8n)

The portfolio uses an n8n-based workflow to generate and maintain GitHub documentation.

### Repository Scanner
Uses GitHub Git Trees API with recursive=1 to scan all docs/ files at any nesting depth. Node: `Get Repository Documentation Tree`. Output normalized by `Build Documentation Index` code node.

### File Retrieval Safety
Nodes `Get + Decode Repository Memory` and `Get + Decode Architecture Decisions` normalize output on 404 responses, returning safe default content to ensure stable Merge node behavior.

### Key files managed
- docs/repository-memory.md
- docs/architecture-decisions.md
- docs/projects/* (nested docs supported)

The Documentation Architect workflow completed a structural refactor establishing the following confirmed baseline: recursive Git Tree scanning replaces flat document scanning, safe GitHub retrieval with normalized 404 handling, cleaned merge context for deterministic downstream data, improved Check Target File supporting subdirectory matching, and automatic ADR numbering calculated at the workflow layer. Responsibility is formally separated between Documentation Architect (generation only) and Repository Intelligence (repository memory and ADR updates only). The architectural direction targets: output schema validation nodes, prompt versioning and externalization under prompts/, context trimming before agent invocation, impact classification as a separate pre-generation step, affected documents execution pipeline, idempotency check before commit, and agent retry and fallback policy.

The Documentation Architect workflow enforces agent output validation, repository memory scope enforcement, and ADR number injection validation as workflow-layer responsibilities rather than agent-layer responsibilities. Three dedicated validation nodes are specified: Validate Agent Output (enforces output schema contracts per agent), Validate Memory Payload (rejects memory updates containing future-tense or unverified content), and ADR Injection Validator (asserts ADR output number matches workflow-supplied nextADRNumber). ADR numbers are calculated by the workflow and injected as explicit inputs; agents never calculate sequence values. Repository Memory records only completed, verified implementation facts. Agent responsibility is separated: Documentation Architect handles documentation generation only; Repository Intelligence handles repository memory updates and ADR entries for completed decisions only.

## Motorhome Pi5 — Home Assistant (Planned)

A second Raspberry Pi project exists in the portfolio targeting mobile smart home automation. Hardware is a Raspberry Pi 5 with 8 GB RAM, intended for vehicle-mounted use in a motorhome. Target software is Home Assistant. As of initial documentation, no software has been installed and no configuration applied. This project is a sibling to the Nextcloud Home Server — both follow the same self-hosted, low-power, no-cloud-dependency philosophy. The two systems are independent and not currently planned to integrate with each other. Documented at `docs/motorhome-pi5-homeassistant.md`.

A second Raspberry Pi project exists: Motorhome Pi5 running Home Assistant. It is a sibling project to the Nextcloud home server, following the same self-hosted, low-power, no third-party cloud philosophy. The two projects are independent systems with no planned integration. As of initial documentation, this project is not yet implemented — hardware has not been acquired and no software has been installed.

Project: Motorhome Pi5 Home Assistant. A Raspberry Pi 5 8GB with NVMe HAT used as a central Home Assistant hub for a motorhome. Hardware acquired: Raspberry Pi 5 8GB and NVMe HAT. Confirmed technology choices: Home Assistant as the home automation platform, NVMe storage over SD card for reliability in mobile/vibration environments. Planned capabilities: water level monitoring, battery monitoring. Software installation and sensor integration not yet started.

## Resume Generator Workflow

**Status:** Implemented

**Purpose:** Accepts a job description via chat trigger, reads a structured candidate knowledge base, matches skills to the job description, generates a tailored ATS-friendly resume via a primary LLM agent, reviews it with a second LLM agent, loops for revisions (maximum 2 rounds), and outputs an HTML and PDF file.

**Technology Stack:**
- Orchestration: n8n (19 nodes)
- LLM: Anthropic Claude Sonnet 4.5 (Resume Generator), Claude Sonnet 4.6 (Resume Reviewer)
- PDF conversion: Gotenberg (HTTP POST)
- KB reader: read-kb.js (executed via shell command)

**Key Architecture Decisions:**
- Dual-agent design: Resume Generator writes Markdown; Resume Reviewer returns strict JSON with decision/score/issues
- Revision loop uses a Merge node as single entry point (Input 1: initial payload, Input 2: revision payload)
- Maximum 2 revision rounds enforced via revision_round counter carried through the payload (not via node self-reference)
- When max rounds reached, workflow exits using previous_draft from payload rather than re-querying Resume Generator node output
- Reviewer approval threshold: score >= 88, zero CRITICAL issues, zero MAJOR issues

**Confirmed Constraints:**
- n8n does not support self-referencing node output within a loop; round counter must travel via payload
- A standard Merge node in append mode is required to combine two input streams into a loop entry point; a Code node returning items does not work for this pattern
- Prepare HTML must explicitly fetch resume content from Resume Generator node output, not from $json when arriving via the approved branch (which contains reviewer JSON)

**Output:** HTML and PDF resume files written to disk via shell command and Gotenberg.
