Architecture Decisions

ADR-

Date: 2025-07-14
Status: Accepted
Decision: Enforce all agent output contracts, repository memory scope, and ADR sequence integrity at the workflow layer through dedicated validation nodes rather than relying on agent prompt instructions alone.
Reason: Agent prompts are insufficient as sole enforcement mechanisms because model updates or prompt drift can cause agents to ignore instructions, produce incorrect sequences, or store unverified content as fact. Workflow-layer validation nodes act as contract enforcement points independent of prompt behavior.
Impact: Three validation nodes are added to the workflow: Validate Agent Output, Validate Memory Payload, and ADR Injection Validator. Failures at any validation node halt the workflow with a structured error before any commit is attempted. Downstream nodes operate only on data that has passed the relevant contract check.
Alternatives Considered: Relying solely on prompt instructions to constrain agent behavior; adding post-commit correction logic to repair bad ADR entries or memory pollution after the fact. Both alternatives were rejected because they allow invalid data to propagate before detection.


ADR-

Date: 2025-07-10
Status: Accepted
Decision: Establish formal responsibility separation between Documentation Architect and Repository Intelligence agents, with Documentation Architect responsible for generation only and Repository Intelligence responsible for repository memory and ADR updates only. ADR numbering is calculated at the workflow layer and passed to Repository Intelligence; agents never calculate sequence numbers themselves.
Reason: Combining generation and classification or sequence calculation in a single agent increases hallucination risk, reduces testability, and makes failures harder to isolate. Single-responsibility agents receive minimal, well-defined context and produce more consistent outputs.
Impact: All future agent nodes in the Documentation Architect workflow must follow single-responsibility design. Any new agent added to the workflow must have an explicit, documented responsibility boundary. Workflow nodes handle sequencing, numbering, and routing logic rather than delegating those concerns to AI agents.
Alternatives Considered: Keeping a single omnibus agent to handle generation, classification, and memory updates — rejected because it increases prompt complexity, token usage, and the surface area for non-deterministic behavior.


ADR-001
Date: 2025-07-10
Status: Accepted
Decision: Replace flat docs/ directory scanner with recursive GitHub Git Trees API call and normalize file retrieval nodes to handle 404 responses gracefully.
Reason: Original GET /contents/docs only scanned top-level files, missing nested documentation. File retrieval nodes produced inconsistent output on missing files, destabilizing the downstream Merge node.
Impact: All docs/ files at any nesting depth are now visible to the workflow. Merge node receives consistent output regardless of whether managed files exist in the repository.
Alternatives Considered: Keeping flat scanner and manually listing all doc paths (rejected: brittle and requires manual maintenance); adding error branches per file node (rejected: adds workflow complexity without addressing root cause).


ADR-001
Date: 2025-07-09
Status: Accepted
Decision: Migrate the automated documentation workflow from a large static context injection model to a selective retrieval-based knowledge architecture using Supabase as the knowledge store.
Reason: The previous architecture injected all documentation resources (writing rules, templates, examples, governance, classification information) directly into the Documentation Architect agent prompt for every task. This caused unnecessary token consumption and included information irrelevant to the current task. The new architecture stores reusable documentation intelligence in Supabase and retrieves only the knowledge required for each documentation task based on project classification.
Impact: Lower AI token consumption per documentation task. Cleaner and smaller agent prompts. Reusable and centrally managed documentation intelligence. Scalable knowledge management as new project types are added. Clear separation between knowledge storage and workflow execution. Easier maintenance of templates and rules without modifying workflow logic.
Alternatives Considered: Retaining the full static context injection model with minor prompt trimming. Storing knowledge in a flat file retrieved per run rather than a structured database.


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
