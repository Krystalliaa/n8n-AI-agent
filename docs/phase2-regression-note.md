# Phase 2 Regression Note — n8n Runtime Stabilization

## Overview

This document captures the stabilization changes applied to the n8n runtime environment as part of the Phase 2 regression effort in the cybersecurity test project. These changes address surface area reduction, supply chain integrity, and secrets hygiene across the automation stack.

---

## Classification

- **Change Type:** Infrastructure change, Security hardening
- **Project Context:** Cybersecurity test project — Phase 2 regression
- **Scope:** n8n runtime, container image management, secrets handling, internal service exposure

---

## Stabilization Changes

### 1. n8n Bound to 127.0.0.1

**What changed:** n8n is now bound exclusively to the loopback interface (`127.0.0.1`) rather than listening on all network interfaces.

**Why:** Exposing n8n on all interfaces increases the attack surface, allowing unintended access from external or adjacent network segments. Binding to `127.0.0.1` ensures that n8n is only reachable locally, enforcing the principle of least exposure. Any external access must be intentionally proxied through a controlled gateway.

---

### 2. Images Pinned by Digest

**What changed:** Container images used in the stack are now referenced by their cryptographic digest rather than by mutable tags.

**Why:** Mutable image tags (e.g., `latest` or version strings) can silently resolve to different image layers over time, introducing unverified code into the runtime. Pinning by digest guarantees that the exact image content is reproducible and auditable across deployments. This directly addresses supply chain integrity concerns relevant to the cybersecurity test scope.

---

### 3. Secrets Moved to `.env`

**What changed:** Secrets and sensitive configuration values have been migrated out of inline configuration and into a `.env` file.

**Why:** Hardcoding secrets in workflow definitions, compose files, or source-tracked configuration exposes credentials to anyone with repository access or container inspection capability. Centralising secrets in a `.env` file — excluded from version control — reduces the risk of accidental secret leakage. This also simplifies secret rotation without touching application logic.

> **Note:** The `.env` file must be listed in `.gitignore` and must never be committed to version control.

---

### 4. Gotenberg Internal Only

**What changed:** The Gotenberg service is now configured for internal-only access and is not exposed on any public or host-facing network port.

**Why:** Gotenberg provides document conversion capabilities. Exposing this service externally introduces an unnecessary attack vector — particularly relevant in a security-sensitive environment where arbitrary document rendering could be exploited. Restricting Gotenberg to internal service-to-service communication ensures it is only reachable by trusted components within the same network namespace.

---

## Verification Results

- All 11 Phase 2 checks passed.
- Credentials decrypt 17/17.

---

## Security Posture Summary

| Control | Before | After |
|---|---|---|
| n8n network binding | All interfaces | `127.0.0.1` only |
| Container image references | Mutable tags | Digest-pinned |
| Secrets management | Inline / mixed | Centralised `.env` |
| Gotenberg exposure | [TO BE DOCUMENTED] | Internal only |

---

## Affected Components

- n8n automation runtime
- Container orchestration configuration (Docker Compose or equivalent)
- Gotenberg document service
- Environment secrets configuration

---

## Notes

- Verification procedures for digest pinning across environments: [TO BE DOCUMENTED]
- Secret rotation process and ownership: [TO BE DOCUMENTED]
- Proxy configuration for controlled n8n external access (if applicable): [TO BE DOCUMENTED]