# Project Classification Rules

Purpose:
Classify incoming documentation requests and select the correct documentation template.

These rules are used by the workflow before documentation generation.

Classification must be based only on explicit keywords found in the user input.

Do not infer project type from assumptions.

---

## n8n Automation

Use:

n8n-automation-template.md

Keywords:

- n8n
- workflow
- automation
- webhook
- node
- trigger
- execution
- AI workflow

---

## Homelab / Infrastructure

Use:

homelab-template.md

Keywords:

- Raspberry Pi
- Home Assistant
- Docker
- Linux server
- Ubuntu Server
- Debian
- Nextcloud
- Active Directory
- Windows Server
- Domain Controller
- VirtualBox
- VMware
- Proxmox

---

## AI Agent

Use:

ai-agent-template.md

Keywords:

- AI Agent
- LLM
- OpenAI
- Claude
- Anthropic
- Prompt Engineering
- RAG
- Memory
- Embeddings
- Vector Database

---

## Observability

Use:

observability-template.md

Keywords:

- Grafana
- Loki
- Prometheus
- OpenTelemetry
- Monitoring
- Logging
- Metrics
- SIEM
- Telemetry

---

## Cybersecurity

Use:

cybersecurity-template.md

Keywords:

- SOC
- SIEM
- QRadar
- Splunk
- Wazuh
- Malware
- Detection
- Incident Response
- Active Directory Security

---

## Multiple Matches

If multiple categories match:

Priority order:

1. AI Agent
2. n8n Automation
3. Cybersecurity
4. Observability
5. Homelab

Return the highest priority category.

---

## Documentation Improvement

Use only when:

- an existing document is modified
- no new project knowledge is introduced

Examples:

Allowed:
- grammar correction
- restructuring
- clarification

Not allowed:

- creating documentation for a new project
- documenting a new workflow
- adding new architecture knowledge
