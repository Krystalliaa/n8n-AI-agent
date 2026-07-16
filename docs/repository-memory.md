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
