# Automated Documentation Workflow

## Overview

This document describes the architecture and operation of the automated documentation workflow used to generate and maintain portfolio documentation for this repository.

The workflow uses n8n for orchestration, GitHub for repository operations, and AI agents exclusively for reasoning and content generation. Repository file operations are performed by n8n nodes, not by AI agents.

---

## Architecture Evolution

### Previous Architecture

The Documentation Architect agent previously received large amounts of repository context directly inside the prompt.

This included:

- Writing rules
- Templates
- Examples
- Repository governance
- Classification information
- Previous documentation patterns

This created unnecessary token consumption because the agent received information that was not always required for the current documentation task.

---

### Current Architecture

The workflow has migrated from a large static context injection model to a selective knowledge retrieval architecture.

A Supabase knowledge base now stores reusable documentation intelligence. The workflow synchronizes selected repository files into Supabase and retrieves only the knowledge required for each documentation task.

**Goals achieved by this migration:**

- Lower AI token consumption
- Cleaner agent prompts
- Reusable documentation intelligence
- Scalable knowledge management
- Separation between knowledge storage and execution
- Easier maintenance of templates and rules

The repository documentation system is now moving toward a retrieval-based architecture instead of a prompt-heavy architecture.

---

## Knowledge Base Architecture

### Supabase Knowledge Base

Selected repository files are synchronized into Supabase as structured knowledge entries.

Each knowledge entry contains the following fields:

| Field | Description |
|---|---|
| key | Unique identifier for the knowledge entry |
| path | Original repository file path |
| content | Decoded file content |
| summary | AI-generated summary for context selection |
| category | Entry category (e.g. rule, template, example) |
| project_type | Applicable project type or global |
| priority | Retrieval priority value |
| always_load | Whether this entry is always injected regardless of project type |
| tags | Descriptive tags for retrieval matching |
| updated_at | Synchronization timestamp |

**Example entry metadata:**

```
key:           style_guide
category:      rule
project_type:  global
priority:      100
always_load:   true
```

---

### Current Synchronized Knowledge Sources

The workflow maintains a controlled list of repository files that are synchronized into Supabase.

Current synchronized sources:

- Writing style guide
- Documentation rules
- Project classification rules
- Repository governance
- Documentation templates
- Project examples

Files are fetched through the GitHub Contents API.

---

## Knowledge Base Synchronization Workflow

### Reference File List Node

The workflow maintains a controlled list of repository files eligible for synchronization. Only listed files are fetched and stored.

---

### Decode Reference Files

GitHub API responses return file content encoded as Base64.

The workflow decodes the content before storing it in Supabase.

Each stored knowledge item retains:

- Original repository path
- Entry metadata
- Decoded content
- Synchronization timestamp

---

### AI Summary Generation

Each knowledge entry is processed by an AI summarization step before storage.

The generated summary is used during context selection to determine whether a knowledge entry is relevant to the current documentation task.

The summary should explain:

- When this knowledge should be used
- What type of documentation problem it solves
- Important constraints the agent must follow

The full content is stored separately and retrieved only when required.

---

## Project Classification

### Classify Project Node

The workflow analyzes the user's documentation request and determines the project type before building the knowledge context.

**Supported project classifications:**

| Classification | Key Indicators |
|---|---|
| n8n | n8n, workflow, automation, webhook |
| homelab | Active Directory, Windows Server, Raspberry Pi, Docker, Nextcloud, home server |
| ai_agent | LLM, AI Agent, OpenAI, prompt engineering, memory |
| observability | Grafana, Prometheus, Loki, monitoring, logging |
| cybersecurity | [TO BE DOCUMENTED] |

Classification is based on keyword matching against the project input.

---

## Dynamic Knowledge Selection

Instead of loading all documentation resources for every task, the workflow builds a targeted knowledge requirement list based on project classification.

### Always Required

The following knowledge entries are always loaded regardless of project type:

- `style_guide`
- `documentation_rules`
- `repository_governance`

### Project-Specific Resources

Additional knowledge entries are selected based on the classified project type.

| Project Type | Additional Knowledge Loaded |
|---|---|
| n8n | `template_n8n` |
| homelab | `template_homelab`, `example_homelab` |
| observability | `template_observability`, `example_observability` |
| ai_agent | [TO BE DOCUMENTED] |
| cybersecurity | [TO BE DOCUMENTED] |

Unused templates and examples are not injected into the agent prompt.

---

## Knowledge Retrieval

The workflow retrieves knowledge dynamically from Supabase based on:

- Project classification result
- Required knowledge key list
- Entry priority value
- `always_load` flag

### Agent Context Composition

The Documentation Architect agent receives only:

1. Required global rules
2. Matching project template
3. Relevant example documentation
4. Repository memory
5. Existing document content when the task is an update

---

## Agent Responsibilities

### Documentation Architect

Responsible for:

- Generating documentation content
- Selecting the correct document structure
- Creating file content
- Creating the commit message

Does not:

- Perform GitHub API calls
- Manage repository memory
- Create architecture decision records
- Handle SHA values or file state

---

### Repository Intelligence Agent

Responsible for:

- Evaluating approved documentation changes
- Deciding whether repository memory requires updating
- Deciding whether architecture decisions require updating

Before suggesting memory updates, the agent must:

- Check existing repository memory content
- Avoid recording duplicate knowledge
- Return only genuinely new information

---

## GitHub Operations

All GitHub repository operations are controlled exclusively by n8n nodes.

AI agents do not perform:

- GitHub API calls
- File creation or updates
- SHA retrieval or handling
- Repository state management
- Permission operations

The workflow determines:

- Whether to create or update a file
- Target file path
- Final file content
- Commit message

GitHub nodes execute the actual repository changes.

---

## Related Documentation

- [Documentation Rules](documentation-rules.md)
- [Writing Style Guide](writing-style-guide.md)
- [Repository Governance](repository-governance.md)
- [Repository Memory](repository-memory.md)
- [Architecture Decisions](architecture-decisions.md)
