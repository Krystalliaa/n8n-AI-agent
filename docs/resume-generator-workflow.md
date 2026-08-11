# Resume Generator — n8n Workflow

## Overview

An n8n automation workflow that accepts a job description via chat trigger, reads a structured candidate knowledge base (KB), matches skills to the job description, generates a tailored ATS-friendly resume via a primary LLM agent, reviews it with a second LLM agent, loops for revisions (maximum 2 rounds), and outputs an HTML and PDF file via Gotenberg.

---

## Architecture & Data Flow

```
Chat Trigger
    → Chat Parser (extracts role, company, job title)
    → Base64 Encode Trigger
    → Execute Command (writes .trigger.json + runs read-kb.js)
    → JSON Parse (parses stdout from read-kb.js)
    → JD Analyzer (matches skills, selects profile, fallback to all skills)
    → Subgraph Builder (filters KB: jobs, projects, certs, achievements)
    → LLM Payload Cleaner (strips noise, adds revision_round: 0)
    → Merge Node (entry point: initial pass OR revision loop)
    → Resume Generator (Agent + Claude Sonnet 4.5)
    → Resume Reviewer (Agent + Claude Sonnet 4.6)
    → If1 (contains "REVISED" ?)
        ├─ YES → Merge for Revision (increments round, preserves original data)
            → If Max Rounds (revision_round >= 2 ?)
                ├─ NO  → loop back to Merge Node → Resume Generator
                └─ YES → Prepare HTML + Filename (uses previous_draft)
        └─ NO  → Prepare HTML + Filename (fetches resume from Resume Generator node)
    → Write File (saves HTML + calls Gotenberg for PDF)
```

---

## Node Inventory

| # | Node | Type | Purpose |
|---|------|------|---------|
| 1 | Chat Trigger | chatTrigger | Public webhook input |
| 2 | Chat Parser | Code | Regex-based extraction of role, company, job title |
| 3 | Base64 Encode Trigger | Code | Encodes trigger data for command line |
| 4 | Execute Command | executeCommand | Runs read-kb.js and passes base64 trigger |
| 5 | JSON Parse | Code | Parses stdout from Execute Command into JSON |
| 6 | JD Analyzer | Code | Matches JD to skill taxonomy + ATS clusters; selects profile; guarantees at least all profile skills are matched |
| 7 | Subgraph Builder | Code | Filters KB graph: only jobs/projects/certs linked to matched skills; validates claims against evidence index |
| 8 | LLM Payload Cleaner | Code | Builds minimal payload for LLM; injects revision_round: 0 |
| 9 | Merge Node | Merge | Entry point: accepts initial payload (input 1) OR revision payload (input 2) |
| 10 | Anthropic Chat Model | lmChatAnthropic | Claude Sonnet 4.5 for Resume Generator |
| 11 | Simple Memory | memoryBufferWindow | Shared memory for both agents |
| 12 | Resume Generator | Agent | Writes/revises resume in Markdown |
| 13 | Anthropic Chat Model 1 | lmChatAnthropic | Claude Sonnet 4.6 for Resume Reviewer |
| 14 | Resume Reviewer | Agent | Reviews resume; returns strict JSON with decision/score/issues |
| 15 | If1 | If | Checks if reviewer output contains "REVISED" |
| 16 | Merge for Revision | Code | Parses reviewer JSON; fetches original data from LLM Payload Cleaner; fetches latest draft from Resume Generator; increments revision_round |
| 17 | If Max Rounds | If | Checks revision_round >= 2 |
| 18 | Prepare HTML + Filename | Code | Markdown → HTML + CSS; creates filename from role |
| 19 | Write File | executeCommand | Saves HTML; HTTP POST to Gotenberg for PDF conversion |

---

## Prompts

### Resume Generator — System Message

```
You are an expert technical recruiter and resume writer.

Build a concise, credible resume that presents the candidate as the strongest truthful match for the target role.

=== PRIORITIES ===
1. Accuracy
2. Role relevance
3. Strong evidence
4. Clear writing
5. ATS compatibility
6. Visual readability

=== TRUTH ===
Use only information supported by the supplied source data. Preserve the meaning and scope of claims. Never invent tools, responsibilities, metrics, seniority, ownership, certifications, or experience.

Use metrics only when explicitly supported by the source data.

Treat personal projects as hands-on evidence and professional experience as commercial evidence. Keep that distinction clear.

=== TAILORING ===
Prioritize experience, skills, and projects that directly support the target role.

Mirror relevant terminology from the job description when it accurately describes the candidate's experience.

Do not keyword-stuff. Do not include every skill or project simply because it exists in the knowledge base.

When the candidate has a gap, strengthen adjacent evidence rather than implying the missing experience.

=== WRITING ===
Write concise, specific bullets focused on action, technology, context, and outcome where evidence exists.

Avoid repetition, filler, vague claims, and unnecessary adjectives.

Use the candidate's strongest relevant experience first within each section.

The Professional Summary should explain why this candidate makes sense for this particular role in 3–4 lines.

=== STRUCTURE ===
Use:
- Professional Summary
- Technical Skills
- Professional Experience
- Selected Projects
- Education
- Certifications
- Languages when relevant

Use standard headings, plain text, bullets, and clean Markdown. No tables, graphics, icons, columns, or decorative elements.

Keep the resume to approximately 1–2 pages. Prefer one page when the evidence can be presented without sacrificing important relevance.

Every section must earn its space.
```

### Resume Generator — User Message

```
=== MODE: {{ $json.reviewer_feedback ? 'REVISE' : 'GENERATE' }} ===

{{ $json.reviewer_feedback
  ? '=== PREVIOUS DRAFT ===\n' + $json.previous_draft + '\n\n=== REVIEWER FEEDBACK ===\n' + $json.reviewer_feedback + '\n\nRevise the draft using all valid feedback. Return the complete resume.'
  : 'Create a new tailored resume.' }}

=== CANDIDATE ===
{{ JSON.stringify({
  name: $json.profile?.full_name || 'Candidate',
  email: $json.profile?.email || '',
  phone: $json.profile?.phone || '',
  location: $json.profile?.location || '',
  linkedin: $json.profile?.linkedin || '',
  github: $json.profile?.github || ''
}, null, 2) }}

=== TARGET ===
Role: {{ $json.target_role || 'General' }}
Company: {{ $json.target_company || '' }}

=== JOB DESCRIPTION ===
{{ $json.job_description || '' }}

=== SOURCE OF TRUTH ===
Skills:
{{ JSON.stringify($json.filtered_kb?.skills || [], null, 2) }}

Experience:
{{ JSON.stringify($json.filtered_kb?.jobs || [], null, 2) }}

Projects:
{{ JSON.stringify($json.filtered_kb?.projects || [], null, 2) }}

Certifications:
{{ JSON.stringify($json.filtered_kb?.certifications || [], null, 2) }}

Education:
{{ JSON.stringify($json.filtered_kb?.education || [], null, 2) }}

Claims:
{{ JSON.stringify($json.filtered_achievements || [], null, 2) }}

Timeline:
{{ JSON.stringify($json.filtered_timeline || [], null, 2) }}

Create the final ATS-friendly resume in valid Markdown.
```

### Resume Reviewer — System Message

```
You are a senior technical recruiter, ATS specialist, and fact-checking resume editor.

Evaluate whether the resume is accurate, compelling, readable, and well aligned with the target role.

Return ONLY this JSON structure:

{
  "decision": "APPROVED" | "REVISED",
  "score": 0,
  "issues": [
    {
      "severity": "CRITICAL" | "MAJOR" | "MINOR",
      "category": "HALLUCINATION" | "ROLE_ALIGNMENT" | "EVIDENCE" | "MISSING_RELEVANCE" | "GENERIC_WRITING" | "REPETITION" | "FORMAT" | "LENGTH",
      "location": "Summary" | "Skills" | "Experience" | "Projects" | "Education" | "Other",
      "problem": "specific issue",
      "fix": "specific correction"
    }
  ],
  "feedback": "concise revision guidance"
}

=== REVIEW ===

1. TRUTH
Every substantive claim must be supported by the supplied source data.
Flag invented tools, duties, metrics, seniority, ownership, certifications, or experience.

2. ROLE ALIGNMENT
Check whether the resume emphasizes the strongest evidence for the target role.
Relevant existing experience should be reframed when appropriate rather than removed.

3. EVIDENCE
Prefer concrete technologies, environments, responsibilities, and documented results.
Do not penalize a resume simply because a JD metric is absent.

4. SELECTIVITY
Relevant skills and projects should be prioritized. Do not reward unnecessary keyword or project lists.

5. WRITING
Bullets should be concise, specific, and non-repetitive.
Flag vague filler such as "responsible for", "strong knowledge", or "worked on" when stronger wording is supported.

6. STRUCTURE
Check hierarchy, consistency, ATS readability, section balance, and excessive length.

7. CANDIDATE POSITIONING
The resume should present one coherent professional story. Tailoring may change emphasis, but must not change the candidate's actual background.

=== SCORING ===

Accuracy: 35%
Role alignment: 30%
Evidence and specificity: 20%
Writing and structure: 15%

APPROVED requires:
- score >= 88
- zero CRITICAL issues
- zero MAJOR issues

Otherwise return REVISED.
```

### Resume Reviewer — User Message

```
=== TARGET ===
Role: {{ $json.target_role || '' }}

=== JOB DESCRIPTION ===
{{ $json.job_description || '' }}

=== RESUME ===
{{ $json.output || $json.message?.content || $json.content || '' }}

=== SOURCE OF TRUTH ===
Skills:
{{ JSON.stringify(($json.filtered_kb?.skills || []).map(s => s.canonical_name)) }}

Projects:
{{ JSON.stringify(($json.filtered_kb?.projects || []).map(p => p.name)) }}

Claims:
{{ JSON.stringify(($json.filtered_achievements || []).map(a => ({
  id: a.claim_id,
  text: a.raw_text
}))) }}

Review the resume for truth, relevance, evidence, quality, and length.
Return STRICT JSON only.
```

---

## Revision Loop Design

The revision loop was a deliberate architectural choice to improve resume quality without manual iteration. The design uses a standard n8n Merge node as the single entry point before the Resume Generator, accepting two input streams:

- **Input 1**: initial payload from LLM Payload Cleaner (first pass)
- **Input 2**: revision payload from Merge for Revision (subsequent passes)

This pattern avoids routing ambiguity and cleanly separates the generate-from-scratch path from the revise-from-feedback path. A maximum of 2 revision rounds is enforced by carrying `revision_round` through the payload rather than querying node state, which avoids n8n self-reference errors.

When max rounds are reached, the workflow exits using `previous_draft` carried in the payload rather than re-querying the Resume Generator node output.

---

## Troubleshooting Log

### Challenge 1: JSON Syntax Error — Duplicate `index` Key

**Problem:** The connection from Resume Generator → Resume Reviewer had `"index": 0` written twice in the workflow JSON, causing n8n to fail parsing the workflow.

**Fix:** Removed the duplicate key.

---

### Challenge 2: Swapped System Messages

**Problem:** The Resume Generator agent had the reviewer's system prompt (requesting strict JSON review output), and the Resume Reviewer had a loose text prompt. As a result, the generator was outputting JSON reviews instead of resumes.

**Fix:** Swapped the system messages back to their correct roles. The generator now holds the resume writer persona; the reviewer holds the strict JSON fact-checker persona.

---

### Challenge 3: Broken Revision Loop

**Problem:** There was no return path from Merge for Revision back to Resume Generator. The workflow could only execute a single pass.

**Fix:** Added a Merge node as the entry point before Resume Generator. Input 1 receives the initial payload from LLM Payload Cleaner. Input 2 receives the revision payload from Merge for Revision. The merge feeds into Resume Generator, enabling the loop.

---

### Challenge 4: Missing Max Rounds Protection

**Problem:** Without a round counter, the revision loop could run indefinitely if the reviewer kept returning `REVISED`.

**Fix:**
- Added `revision_round: 0` in LLM Payload Cleaner.
- In Merge for Revision, the round is read from `$json.revision_round` (carried through the loop) and incremented.
- Added If Max Rounds node after Merge for Revision. Condition: `revision_round >= 2`. If true, exits to Prepare HTML. If false, loops back to Merge Node.

---

### Challenge 5: Prepare HTML Receiving Reviewer JSON Instead of Resume

**Problem:** When If1 took the approved (false) branch, `$json` contained the reviewer's JSON output (`{decision, score, issues...}`), not the resume Markdown. The HTML node attempted to render the review JSON as a resume.

**Fix:** In Prepare HTML + Filename, the code now explicitly fetches the actual resume from the Resume Generator node using `$('Resume Generator').last()?.json`. It falls back to `$json.previous_draft` only when arriving from the max-rounds exit path.

---

### Challenge 6: Merge for Revision Self-Reference Error

**Problem:** The node code used `$('Merge for Revision').last()?.json?.revision_round` to read the previous round count. n8n threw: `"There is no connection back to the node 'Merge for Revision', but it's used in code here."`

**Fix:** Replaced the self-reference with `$json.revision_round || 0`. The round counter travels through the loop via the payload (injected by LLM Payload Cleaner and carried by the Merge node), so it is always available in `$json` without referencing the node itself.

---

### Challenge 7: Route Input vs Merge Node

**Problem:** Initially, a Code node (Route Input) using `return items;` was used as the loop entry point. This caused data routing issues because it did not properly merge two different input streams (initial vs revision).

**Fix:** Replaced Route Input with a standard n8n Merge node in append mode. This cleanly combines the initial path (Input 1) and the revision loop path (Input 2) into a single stream feeding Resume Generator.

---

## External Dependencies

| Service | Purpose | Endpoint / Path |
|---------|---------|----------------|
| read-kb.js | Reads candidate KB from disk | Executed via Execute Command node |
| Gotenberg | HTML → PDF conversion | `http://gotenberg:3000/forms/chromium/convert/html` |
| Anthropic API | LLM inference | Via n8n credentials |

---

## Output Files

| File | Path |
|------|------|
| HTML resume | `/data/resume-kb/output/{role}_{timestamp}.html` |
| PDF resume | `/data/resume-kb/output/{role}_{timestamp}.pdf` |
| Gotenberg error log (if PDF fails) | `/data/resume-kb/output/{role}_{timestamp}.pdf.error.txt` |

---

## Current Status

- Workflow runs end-to-end from chat trigger to PDF output.
- Revision loop works correctly: maximum 2 rounds, then exits.
- Resume Generator produces proper Markdown resumes.
- Resume Reviewer returns strict JSON with `APPROVED` / `REVISED` decisions.
- HTML and PDF generation uses actual resume content, not reviewer metadata.
- No infinite loops.
- Execution completes in a single pass or two passes depending on reviewer decision.
