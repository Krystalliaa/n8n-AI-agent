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
