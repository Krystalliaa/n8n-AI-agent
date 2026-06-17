# Personal AI Assistant with n8n + Claude API
 
A self-hosted, visual AI agent built with n8n that can search the web, create and update Google Docs, and serve as the foundation for a broader home/RV automation system.
 
## Overview
 
This project documents the process of building a personal AI assistant using n8n's visual workflow builder, connected to Anthropic's Claude API and Google's Gemini API. The assistant can research topics in real time using web search, then automatically create and populate Google Docs with the findings — all through natural language requests in a chat interface.
 
The project is part of a larger personal automation ecosystem that will eventually include:
- Home Assistant integration for RV automation (lights, sensors, cameras)
- Voice control via M5 Atom Echo devices
- Self-hosted Nextcloud for private file storage
## Tech Stack
 
- **n8n** (self-hosted via Docker) — visual workflow automation platform
- **Anthropic Claude API** (Claude Haiku) — the LLM powering the AI Agent
- **SerpAPI** — web search tool integration
- **Google Docs API** (OAuth2) — document creation and editing
- **Docker Desktop** — containerization on Windows 11
## Hardware
 
- ASUS TUF Gaming A16 (AMD Ryzen 7 260, RTX 5050, 16GB RAM)
- Windows 11
---
 
## Setup Steps
 
### 1. Install Docker Desktop
 
Docker is required to run n8n locally in a container.
 
1. Download Docker Desktop from [docker.com](https://www.docker.com/products/docker-desktop/) — choose the **AMD64** version (correct for AMD Ryzen and Intel CPUs; ARM64 is for devices like Raspberry Pi)
2. Run the installer and restart the machine
3. Launch Docker Desktop and wait for it to start (green icon in the system tray)
### 2. Run n8n via Docker
 
Open PowerShell and run:
 
```powershell
docker run -it --rm --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```
 
This pulls the n8n image and starts the service. Once you see `Editor is now accessible`, open:
 
```
http://localhost:5678
```
 
Create a local account (no email verification needed for self-hosted).
 
### 3. Get a Claude API Key
 
1. Go to [console.anthropic.com](https://console.anthropic.com)
2. Create an API key
3. Add a small amount of credit (a few dollars covers extensive testing — Claude Haiku is very low-cost per message)
### 4. Get a SerpAPI Key (for web search)
 
1. Sign up at [serpapi.com](https://serpapi.com) (free tier available, ~100 searches/month)
2. Copy the API key from the dashboard
---
 
## Building the Workflow
 
### Step 1 — Chat Trigger
 
Add a **Chat Trigger** node. This is the entry point — it provides a chat interface within n8n to talk to the agent.
 
### Step 2 — AI Agent Node
 
Add an **AI Agent** node and connect it to the Chat Trigger.
 
**Chat Model:**
- Add credential → Anthropic
- Paste the Claude API key
- Select model: `claude-haiku-3-5` (fast and inexpensive for testing)
**System Prompt:**
 
```
You are a personal AI assistant helping with research, job searching, and organizing ideas.
 
Always respond in the same language the user writes in — if they write in Greek, respond in Greek.
 
Keep responses concise and clear. Do not use markdown formatting like ## or **bold** — use plain text only.
 
IMPORTANT BEHAVIOR RULES:
 
1. If the user asks you to research a topic and save/create a document about it, you must automatically:
   a. Use the search tool to research the topic
   b. Call create_google_doc with an appropriate title
   c. Call update_google_doc_content with the documentId from step b, inserting your research findings as the content
   You do not need to be told explicitly to "update" or "use the content" — if the user mentions creating a document about a topic, always complete all three steps automatically.
 
2. If the user just asks a question without mentioning documents, simply answer in chat — do not create documents unless asked.
 
3. Always confirm at the end what document was created with its title.
```
 
**Memory:**
Add a **Simple Memory** (or Window Buffer Memory) node so the agent retains conversation context.
 
### Step 3 — Web Search Tool (SerpAPI)
 
1. In the AI Agent node, click the **+** next to **Tools**
2. Search for and add **SerpAPI**
3. Create a new credential with the SerpAPI key
4. In the **Search Query (q)** field, switch to **Expression** mode and enter:
```
{{ $fromAI('query', 'the search query to look up') }}
```
 
This lets the AI Agent dynamically decide the search query at runtime, rather than using a fixed value.
 
### Step 4 — Google Docs: Create Tool
 
1. Add another tool: **Google Docs**
2. Authenticate via **OAuth2** (this opens a Google login window — grant the requested permissions)
3. **Resource:** Document | **Operation:** Create
4. **Title** field (set to Expression mode):
```
{{ $fromAI('title', 'the title for the document') }}
```
 
5. Rename the tool itself (not just the operation) to something explicit, e.g. `create_google_doc` — this matters for the AI Agent to reliably distinguish between multiple similar tools.
> **Note:** The n8n Google Docs "Create" operation only creates the document with a title — it does not have a content field. A second node is required to insert content.
 
### Step 5 — Google Docs: Update Tool
 
1. Add a second **Google Docs** tool node
2. **Resource:** Document | **Operation:** Update
3. **Document ID** field (Expression mode):
```
{{ $fromAI('documentId', 'the ID of the document just created') }}
```
 
4. Under **Actions**, add an action:
   - **Action:** Insert
   - **Insert Segment:** Body
   - **Insert Location:** At the end / specific position
   - **Text** field (Expression mode):
```
{{ $fromAI('content', 'the content to insert') }}
```
 
5. Rename this tool to something explicit, e.g. `update_google_doc_content`
> **Key lesson learned:** Giving both Google Docs tools generic/similar names caused the AI Agent to describe the tool call as text in its chat response instead of actually executing it (the node showed as grey/not-executed in the Executions log). Renaming the tools to distinct, descriptive names (`create_google_doc`, `update_google_doc_content`) — combined with explicit instructions in the system prompt — fixed this reliably.
 
---
 
## Testing
 
With the workflow saved, open the **Chat** panel (bottom-left of the canvas) and try:
 
```
Κάνε research για το Raspberry Pi 5 και φτιάξε μου ένα doc
```
 
Expected behavior:
1. The agent calls the search tool to find current information
2. It calls `create_google_doc` to create a new document with a relevant title
3. It calls `update_google_doc_content` using the returned document ID to insert the researched content
4. It confirms in chat what document was created
You can verify execution success by checking **Executions** in the n8n left sidebar — every tool call in the chain should show green (success), not grey (never executed) or red (error).
 
---
 
## Architecture Diagram
 
```
                    ┌─────────────────┐
                    │   Chat Trigger    │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │     AI Agent      │
                    │  (Claude Haiku)   │
                    └────────┬─────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
┌───────▼───────┐  ┌─────────▼──────────┐  ┌──────▼───────────────┐
│ Simple Memory  │  │  SerpAPI Search    │  │  Google Docs Tools    │
│                │  │  (web search)      │  │  - create_google_doc  │
│                │  │                    │  │  - update_google_doc_ │
│                │  │                    │  │    content            │
└────────────────┘  └────────────────────┘  └────────────────────────┘
```
 
---
 
## Lessons Learned
 
1. **n8n does not charge for AI nodes** — only the underlying LLM provider (Anthropic, OpenAI, Google) bills for token usage. n8n Cloud plans add a separate, metered "AI Workflow Builder" credit system that is unrelated to the AI Agent node itself.
2. **`$fromAI()` expressions only resolve during live agent execution** — testing a tool node standalone (outside of an actual chat-triggered run) will show "undefined", which is expected behavior, not a bug.
3. **Google's "Create Document" and "Update Document" operations are separate steps** — there's no single call that creates a doc with content already inside it; this must be explicitly chained.
4. **Tool naming matters a lot for agent reliability.** When two tools have similar/generic names, the LLM can become confused about which to call, and may "hallucinate" a tool call in its text response without actually triggering it. Clear, distinct tool names combined with explicit step-by-step instructions in the system prompt resolved this.
5. **System prompts should encode multi-step behaviors explicitly.** Initially, simple prompts ("create a doc about X") failed to trigger both the create and update steps. Adding explicit "always do steps A, B, C in this scenario" instructions to the system prompt made the behavior consistent without needing increasingly detailed prompts each time.
---
 
## Roadmap (Next Steps)
 
- [ ] Add job search tool (LinkedIn/Indeed search integration)
- [ ] Add scheduled/cron trigger for periodic research digests
- [ ] Set up Raspberry Pi 5 (8GB) with Home Assistant OS for RV automation
- [ ] Integrate Hiseeu PoE camera system with Home Assistant (via RTSP/ONVIF)
- [ ] Flash M5 Atom Echo with ESPHome for voice wake-word detection
- [ ] Run Whisper (speech-to-text) and Piper (text-to-speech) locally on the laptop, connected to Home Assistant via the Wyoming protocol
- [ ] Connect the n8n AI Agent to Home Assistant entities for full voice-controlled automation
---
 
## Notes on Cost
 
- **Claude API (Haiku):** Extremely low cost per message — a few dollars of credit covers extensive personal use.
- **SerpAPI:** Free tier covers ~100 searches/month, sufficient for personal/light use.
- **Cloud GPU rental** (e.g. RunPod, for running local LLMs) was evaluated and ruled out for this use case — running a model 24/7 in the cloud costs roughly $190–$1000+/month depending on the GPU tier, which is far more expensive than API usage for a personal assistant workload. The RTX 5050 in the local laptop is sufficient for running smaller local models (e.g. Llama 3.1 8B) in the future, without any cloud cost.
 
