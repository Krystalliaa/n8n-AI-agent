Architecture Decisions

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
