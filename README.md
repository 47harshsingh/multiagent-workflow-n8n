# Multi-Agent AI Assistant (n8n)

 multi-agent workflow built in [n8n](https://n8n.io) that handles email, calendar, and meeting-notes tasks from a single chat interface. A Master Orchestrator Agent reads each request and routes it to the right specialist agent, which then calls the appropriate tools to complete the task — including chaining multiple agents together for combined requests (e.g. "summarize my last meeting and email the team").

## Architecture

```
When chat message received
        |
Master Orchestrator Agent  (routes requests by intent)
        |
   ┌────┼─────────────────┐
   |    |                 |
Email  Calendar        Meetings
Agent  Agent            Agent
   |    |                 |
5 Gmail 5 Google Calendar 2 Fireflies
tools   tools             HTTP tools
```

**Master Orchestrator Agent** — classifies each incoming message and delegates to one or more specialist agents in priority order, based on intent (email / calendar / meetings / combination).

**Email Agent** — Gmail tools: Get many messages, Send, Create draft, Reply, Delete. Replies and deletes first fetch the inbox to resolve a Message ID.

**Calendar Agent** — Google Calendar tools: Get many events, Create event, Update event, Check availability, Delete event. Updates and deletes first fetch events to resolve an Event ID.

**Meetings Agent** — Fireflies.ai tools via raw HTTP Request (GraphQL): Get list of transcripts, Get full transcript (with summary + action items). Extracts a transcript ID from the list before pulling the full transcript.

Each agent runs its own LLM (Claude Sonnet, via n8n's AI Gateway) and its own short-term conversational memory, scoped to the chat session.

## Setup

1. Import `multiagent-workflow.json` into your n8n instance (**Workflows → Import from File**).
2. Connect credentials for each service used by the tool nodes:
   - **Gmail** (OAuth2) — used by all 5 Email Agent tools
   - **Google Calendar** (OAuth2) — used by all 5 Calendar Agent tools
   - **Anthropic** — used by the Chat Model nodes (or swap for your own LLM provider)
3. Set your **Fireflies.ai API key** in both HTTP Request tool nodes (`Get list of transcripts`, `Get the full transcript`):
   - Open the node → **Header Parameters** → replace `{{YOUR_FIREFLIES_API_KEY}}` in the `Authorization` header value with your actual key.
   - Get your key from [app.fireflies.ai](https://app.fireflies.ai) → **Integrations → Fireflies API**.
   - Store it in an n8n credential or environment variable rather than pasting it directly into the node if you plan to export/share the workflow again.
4. Test each agent individually before publishing (see **Test Cases** below), then activate the workflow.

## Test Cases

| # | Prompt | Expected route |
|---|--------|-----------------|
| 1 | "Show me my latest emails." | Email Agent → Get many messages |
| 2 | "Create a draft email to [recipient] with subject [subject]." | Email Agent → Create draft |
| 3 | "Show my events between [start] and [end]." | Calendar Agent → Get many events |
| 4 | "Check my calendar availability between [start] and [end]." | Calendar Agent → Check availability |
| 5 | "List my latest Fireflies transcripts." | Meetings Agent → Get list of transcripts |
| 6 | "Summarize the transcript titled [meeting title]." | Meetings Agent → Get full transcript |
| 7 | "Find my meeting transcript, summarize the action items, and draft a follow-up email." | Meetings Agent → Email Agent (chained) |

## Notes

- API keys and credentials are intentionally excluded from this repo. Placeholder text (`{{YOUR_FIREFLIES_API_KEY}}`) marks where a real key is required.
- Memory is stored locally per n8n instance via Simple Memory nodes — not suitable for multi-worker/production setups without an external memory store (Redis, Postgres, etc.).
- Built as a hands-on exercise in multi-agent orchestration: agent-to-agent routing, shared tool design, and combining agents for compound requests.
