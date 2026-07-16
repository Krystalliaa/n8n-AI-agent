Architecture Decisions

ADR-001
Date: 2025-07-10
Status: Accepted
Decision: Migrate the automated documentation workflow from static context injection to a selective knowledge retrieval architecture backed by a Supabase knowledge base.
Reason: The previous architecture injected all documentation resources (writing rules, templates, examples, governance, classification information) directly into the Documentation Architect agent prompt for every task, causing unnecessary token consumption and bloated prompts regardless of relevance to the current task.
Impact: Lower AI token consumption per documentation task. Cleaner and more focused agent prompts. Reusable documentation intelligence stored in Supabase. Dynamic knowledge selection based on project classification. Easier maintenance of templates and rules. The system now scales without increasing per-task token cost.
Alternatives Considered: Retaining the static context injection model with trimmed content. Using a vector similarity search for knowledge retrieval instead of keyword-based project classification and key-list selection.


ADR-001
Date: 2025-07-10
Status: Accepted
Decision: Adopt a dual-agent architecture for the automated documentation workflow, separating documentation generation (Documentation Architect) from repository knowledge management (Repository Intelligence Agent).
Reason: Combining both responsibilities into a single agent created prompt complexity and increased the risk of concerns polluting each other. Separating agents gives each a single, clearly scoped responsibility and produces more reliable output.
Impact: All documentation generation is handled exclusively by the Documentation Architect agent. All repository memory and ADR evaluation is handled exclusively by the Repository Intelligence Agent. Neither agent performs repository write operations directly — the workflow orchestrates all side effects.
Alternatives Considered: Single agent handling both documentation generation and memory updates (rejected due to prompt complexity and cross-concern interference).

ADR-002
Date: 2025-07-10
Status: Accepted
Decision: Store the documentation knowledge base (rules, templates, style guides, examples) in Supabase rather than hardcoding content into workflow prompts.
Reason: Keeps prompts shorter, allows documentation rules and templates to evolve independently of the workflow, and supports multiple documentation styles without modifying workflow nodes.
Impact: All knowledge base content is retrieved dynamically at runtime. Supabase must contain valid entries for all required keys; fallback defaults are applied in the workflow for missing entries.
Alternatives Considered: Hardcoding templates and rules directly into node prompts (rejected due to maintainability and inflexibility).

ADR-003
Date: 2025-07-10
Status: Accepted
Decision: Use raw.githubusercontent.com for retrieval of known static files (repository memory, ADRs) and the GitHub Contents API for dynamic operations (listing, existence checking, create, update).
Reason: Raw URLs return plain text with no base64 decoding required and no authentication overhead for public repositories. The GitHub Contents API is necessary for write operations and dynamic listing.
Impact: Repository memory and ADR files must remain at known, stable paths. Dynamic file operations continue to use the authenticated GitHub API.
Alternatives Considered: Using the GitHub Contents API for all file retrieval (rejected due to base64 decoding overhead and unnecessary complexity for static known files).


ADR-001
Date: 2025-07-10
Status: Accepted
Decision: Store documentation knowledge base (rules, style guides, templates, examples) in Supabase rather than hardcoding into n8n workflow prompts or nodes.
Reason: Keeps prompts shorter, allows documentation rules and templates to evolve independently of the workflow, supports multiple documentation styles without modifying workflow nodes, and enables selective retrieval based on project classification to reduce token usage.
Impact: All documentation generation depends on Supabase availability. Missing Supabase entries require fallback defaults in workflow Code nodes to prevent empty context reaching the AI agent.
Alternatives Considered: Hardcoding templates and rules directly into the AI agent system prompt (rejected: inflexible, increases prompt size, requires workflow edits to update rules); storing templates as files in the GitHub repository (rejected: adds API call complexity and couples template management to repository structure).

ADR-002
Date: 2025-07-10
Status: Accepted
Decision: Use raw.githubusercontent.com URLs for retrieval of known static repository files (repository memory, ADRs) and the GitHub Contents API only for dynamic operations (listing files, existence checks, create, update).
Reason: Raw URLs return plain text with no base64 decoding required and no authentication overhead for public repositories. GitHub Contents API is necessary for write operations and dynamic listing where file paths are not known in advance.
Impact: Repository memory and ADR files must remain at known, stable paths. Any path change requires updating the raw URLs in the workflow HTTP Request nodes.
Alternatives Considered: Using GitHub Contents API for all retrieval (rejected: requires base64 decoding, authentication, and adds unnecessary complexity for static known files).

ADR-003
Date: 2025-07-10
Status: Accepted
Decision: Validate AI agent output (JSON structure, required fields, non-empty content) in a Code node before any GitHub repository write operation.
Reason: Prevents malformed JSON, missing file paths, empty content, and hallucinated output from being committed to the repository. Separates experimental prototype behaviour from production-ready pipeline behaviour.
Impact: Any AI output that fails validation stops the workflow before the GitHub commit step. The user receives no documentation commit for that run and must resubmit.
Alternatives Considered: Trusting AI output directly without validation (rejected: risk of committing malformed or empty files to the repository); validating inside the AI agent prompt only (rejected: prompt-level instructions do not guarantee structured output format compliance).


Decisions
