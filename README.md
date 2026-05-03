# AI-Native Tasks Manager

A multi-agent task management platform where users interact in natural language to create, update, and delete tasks. A specialist sub-agent handles appointment-booking flows. Designed to run on Kubernetes from day one.

## Architecture

```
┌──────────┐
│   User   │
└─────┬────┘
      │  natural language
      ▼
┌────────────────────────┐         handoff          ┌─────────────────────────────┐
│   Tasks Manager Agent  │ ───────────────────────► │  Appointment Booking Agent  │
│   (OpenAI Agents SDK)  │ ◄─────────────────────── │     (OpenAI Agents SDK)     │
└──────────┬─────────────┘                          └──────────────┬──────────────┘
           │ MCP tool calls                                        │
           ▼                                                       ▼
┌────────────────────────┐                              ┌─────────────────────┐
│       Tasks MCP        │                              │   Google Sheets     │
│  (Python, MCP SDK)     │                              │  (booking records)  │
└──────────┬─────────────┘
           │ HTTP
           ▼
┌────────────────────────┐
│   Notifications API    │
│       (FastAPI)        │
└────────────────────────┘

Cross-cutting:  [ Auth Layer: better-auth ]   [ UI Layer: Next.js 16 ]
```

## Components

| Component | Tech | Responsibility |
|---|---|---|
| Tasks Manager Agent | Python, OpenAI Agents SDK | Primary user-facing agent; routes to MCP or sub-agent |
| Appointment Booking Agent | Python, OpenAI Agents SDK | Captures booking intent, writes to Google Sheets |
| Tasks MCP Server | Python, MCP SDK | Exposes `add_tasks`, `update_tasks`, `delete_tasks` tools |
| Notifications API | FastAPI | Schedules and sends reminders |
| UI Layer | Next.js 16 | Chat interface, task list, calendar views |
| Auth Layer | better-auth | Session management, short-lived token issuance |

## Stack

- **Agents:** Python · OpenAI Agents SDK
- **API:** FastAPI
- **UI:** Next.js 16 (TypeScript)
- **Auth:** better-auth
- **Deployment:** Kubernetes

## Key Principles

- **Test-Driven Development** — tests are written before implementation
- **Docs-First** — official SDK/framework docs are consulted before coding
- **Stateless Services** — all state lives in the database, never in process memory
- **Secrets via Environment** — no secrets committed or logged
- **Kubernetes-Native** — every service is a Deployment with health checks, resource limits, and graceful shutdown

## Getting Started

> Implementation is in progress. See [AGENTS.md](./AGENTS.md) for the full system spec and programming principles.
