# Automated Documentation Workflow

## Overview

This document describes the n8n-based automated documentation workflow that generates and maintains GitHub portfolio documentation.

## Applied Improvements

The following changes were applied to the workflow based on AI recommendations.

---

## STEP 1 — Replace Existing Documentation Scanner

### Problem

The original node `Get Existing Docs List` used:

```
GET /contents/docs
```

This only scanned the first level of the `docs/` directory. Nested files were invisible to the workflow.

**Example — what was missing:**

```
docs/
├── n8n-agent.md              ← visible
├── repository-memory.md      ← visible
└── projects/
    └── homelab.md            ← NOT visible
```

### Solution

**Deleted node:** `Get Existing Docs List`

**Created node:** `Get Repository Documentation Tree`

| Setting | Value |
|---|---|
| Method | GET |
| URL | `=https://api.github.com/repos/{{$env.GITHUB_OWNER}}/{{$env.GITHUB_REPO}}/git/trees/main?recursive=1` |
| Authentication | Predefined Credential Type — GitHub API |
| Full Response | Yes |
| Never Error | Yes |

**Expected response structure:**

```json
{
  "body": {
    "tree": [
      { "path": "docs/n8n-agent.md", "type": "blob" },
      { "path": "docs/projects/homeassistant.md", "type": "blob" }
    ]
  }
}
```

### New Code Node: `Build Documentation Index`

Added after `Get Repository Documentation Tree`.

```javascript
const res = $json;

let docs = [];

if (
  res.statusCode === 200 &&
  res.body &&
  Array.isArray(res.body.tree)
) {

  docs = res.body.tree
    .filter(item =>
      item.type === "blob" &&
      item.path.startsWith("docs/")
    )
    .map(item => item.path);

}

return [
  {
    json: {
      existingDocsList:
        docs.length
          ? docs.join("\n")
          : "[No existing documentation found]",

      existingDocsCount: docs.length
    }
  }
];
```

**Output improvement:**

Before:
```
README.md
```

After:
```
docs/projects/n8n-agent.md
docs/projects/homeassistant.md
docs/repository-memory.md
docs/architecture-decisions.md
```

### Updated Connection Flow

**Before:**
```
Merge Knowledge Context
        |
Get Existing Docs List
        |
Format Existing Docs List
        |
      Merge
```

**After:**
```
Merge Knowledge Context
        |
Get Repository Documentation Tree
        |
Build Documentation Index
        |
      Merge
```

**Deleted node:** `Format Existing Docs List`

---

## STEP 2 — Make GitHub File Retrieval Safe

### Problem

The nodes `Get Repository Memory` and `Get Architecture Decisions` used direct `GET` requests on specific files. When a file did not exist, GitHub returned:

```json
{
  "statusCode": 404,
  "body": { "message": "Not Found" }
}
```

The workflow continued due to `neverError: true`, but the output structure became inconsistent, causing unstable behavior in the downstream `Merge` node.

### Solution

Both decode nodes were updated to normalize output regardless of file existence.

### Updated Code — `Get + Decode Repository Memory`

```javascript
const res = $json;

let content = "# Repository Memory\n\n[TO BE DOCUMENTED]";
let existed = false;

if (
  res.statusCode === 200 &&
  res.body &&
  res.body.content &&
  res.body.encoding === "base64"
) {

  content = Buffer
    .from(res.body.content, "base64")
    .toString("utf8");

  existed = true;

}

return [
  {
    json: {
      repoMemory: content,
      repoMemoryExisted: existed,
      repoMemoryStatus: res.statusCode
    }
  }
];
```

### Updated Code — `Get + Decode Architecture Decisions`

```javascript
const res = $json;

let content = "# Architecture Decisions\n\nNo ADR entries available.";
let existed = false;

if (
  res.statusCode === 200 &&
  res.body &&
  res.body.content &&
  res.body.encoding === "base64"
) {

  content = Buffer
    .from(res.body.content, "base64")
    .toString("utf8");

  existed = true;

}

return [
  {
    json: {
      adrContent: content,
      adrExisted: existed,
      adrStatus: res.statusCode
    }
  }
];
```

### Behavior After Fix

| Scenario | Before | After |
|---|---|---|
| File exists | Decoded correctly | Decoded correctly |
| File missing (404) | Inconsistent output | Returns safe default content |
| Merge node stability | Unreliable | Consistent |

---

## Current Workflow State

[TO BE DOCUMENTED]
