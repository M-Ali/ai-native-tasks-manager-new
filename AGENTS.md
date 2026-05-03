# AGENTS.md — Tasks Management System

This is the canonical reference for all contributors and AI coding assistants working on this codebase. It describes **how we build**, not just what we build. Read this before writing a single line of code.

---

## System Overview

A multi-agent task management platform where users interact in natural language to create, update, and delete tasks. A specialist sub-agent handles appointment-booking flows. The system is designed to run on Kubernetes from day one — every component is a stateless, independently deployable service.

---

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

---

## Agents

### Tasks Manager Agent

Primary user-facing agent. Owns the conversation, parses intent, and routes work to the Tasks MCP for CRUD operations or to the Appointment Booking Agent for bookings.

- Always confirm destructive actions before calling a tool.
- Resolve relative times into absolute UTC timestamps before calling tools.
- Hand off to the Appointment Booking Agent when the user's intent involves booking a slot with another party.

### Appointment Booking Agent

Specialist sub-agent that captures booking intent and persists it to Google Sheets.

- Validate all required booking fields (title, attendees, date, time, duration, location/notes) before writing.
- Ask follow-up questions for any missing fields rather than guessing.
- Return control to the Tasks Manager Agent with a confirmation summary once done.

---

## Components

### Tasks MCP Server

Exposes task mutation tools (`add_tasks`, `update_tasks`, `delete_tasks`) over the MCP protocol. After a successful mutation it calls the Notifications API to schedule or cancel reminders. All calls must carry the user's auth token.

### Notifications API

FastAPI service responsible for scheduling and sending reminders. Decoupled from the agent layer so notification channels can evolve independently.

### UI Layer

Next.js 16 chat interface plus task list and calendar views. Streams agent responses and renders tool-call traces during development.

### Auth Layer

better-auth manages sessions and issues short-lived tokens. Every MCP tool call and Notifications API request must be scoped to the authenticated user — never trust the agent to assert identity on its own.

---

## Data Flow: A Typical Request

User: *"Add 8 pm reminder for friends meetup."*

1. UI sends the message to the Tasks Manager Agent with the user's auth token.
2. Agent parses intent and calls `add_tasks` on the Tasks MCP.
3. Tasks MCP persists the task and calls the Notifications API to schedule an 8 pm reminder.
4. Agent replies in natural language: "Done — I'll remind you at 8 pm tonight."

User: *"Book a 30-min call with Sara on Friday at 3 pm."*

1. Tasks Manager Agent recognises booking intent and hands off to the Appointment Booking Agent.
2. Booking Agent collects any missing fields and appends a row to Google Sheets.
3. Control returns to the Tasks Manager Agent, which confirms with the user.

---

## Programming Principles

These are non-negotiable on this project.

### Always Verify What You Are Doing

Before taking any action — writing code, calling a tool, making a change — confirm that you understand exactly what you are about to do and why. Do not proceed on assumptions.

- Re-read the task or instruction before starting implementation.
- After completing a step, verify the output matches the expected result before moving on.
- If something looks wrong or unexpected, stop and investigate rather than continuing.
- Never assume a previous step succeeded — check its output explicitly.
- When in doubt, ask a clarifying question rather than guessing.

### Verification Infrastructure

Quality requires feedback loops, not assumptions. Give agents and developers the tools to verify their own work — this can 2-3x output quality.

- Every agent action that mutates state must produce a verifiable confirmation (e.g. read back the record after writing it).
- Integration tests must cover the full path (UI → Agent → MCP → Notifications), not just unit-level checks.
- Use the Claude-Reviews-Claude pattern for non-trivial changes: one session writes, a second session reviews with a fresh perspective, the first incorporates feedback.
- Run verification before accepting any output as done.

### Planning Discipline

Use Plan Mode before executing non-trivial tasks. Iterate on the plan until aligned, then switch to execution. Re-enter planning immediately when execution goes sideways.

- Write down the plan before writing code.
- If two correction attempts on the same issue fail, stop, clear context, and rewrite the prompt/plan.
- Give the agent the problem to solve, not the solution to implement — let it investigate freely.

### Self-Evolving Documentation

This AGENTS.md grows with every correction. When a mistake is made, document the rule that prevents it from recurring before moving on.

- Update AGENTS.md immediately when a new constraint or lesson is discovered.
- Prune rules that are no longer relevant — a long ignored document is worse than a short focused one.
- The whole team contributes to this file. Every correction is a candidate entry.

### Session and Context Management

Context degrades as it fills. Manage it actively.

- Use `/clear` between unrelated tasks to reset context.
- Use subagents for investigation tasks to keep the main context clean — subagents explore in isolation and report summaries back.
- Maintain focused sessions per workstream rather than mixing unrelated concerns in one conversation.
- Narrow scope or delegate to a subagent before context fills with unscoped exploration.

### Autonomous Problem Solving

- Give the agent the problem, not the solution. Let it investigate without micromanagement.
- Use subagents to parallelise complex work (append "use subagents" to a prompt to trigger this).
- Pre-allow safe, repeated operations (builds, tests, formatters) so the agent is not blocked by permission prompts on routine work.

### Failure Pattern Avoidance

| Pattern | Symptom | Fix |
|---|---|---|
| Mixed-concern session | Unrelated tasks in one conversation, cluttered context | `/clear` between unrelated work |
| Correction spiral | Same issue fails to resolve after 2+ attempts | Clear context, rewrite the prompt from scratch |
| Trust-without-verify | Output looks plausible but edge cases are untested | Always provide and run a verification step |
| Infinite exploration | Context fills with unscoped investigation | Narrow scope or delegate to a subagent |
| Stale rules | AGENTS.md rules ignored because the file is too long | Prune irrelevant rules ruthlessly |

### Test-Driven Development

Write the test before the implementation. Every feature, tool, and agent behaviour starts with a failing test that defines the expected outcome. No code ships without a corresponding test that was written first.

- For MCP tools: write an integration test covering the full path (UI → Agent → MCP → Notifications) before implementing the tool.
- For agents: write an eval — a fixed set of natural-language inputs paired with expected tool calls — before changing any system prompt.
- For API endpoints: write a contract test before writing the handler.

### Docs-First Implementation

Before implementing anything that uses an external SDK or framework, read the official documentation. Use the framework the way its authors intended — not based on prior knowledge or assumptions that may be outdated.

- Use agent skills or MCP tools to fetch and consult official docs during implementation when needed.
- If the docs and your intuition conflict, trust the docs.
- Pin dependency versions after consulting the changelog. Do not blindly upgrade.

### Stateless Services

Every component must be stateless. Any state lives in the database or external store, never in process memory. This is a hard requirement for Kubernetes — pods are ephemeral and can be restarted or rescheduled at any time.

### Secrets Management

Secrets (API keys, service-account credentials, database passwords) live in environment variables only, injected at runtime via Kubernetes Secrets. Never commit secrets, never log them in agent traces, never hard-code them.

### Language and Layer Boundaries

Agents and MCP server are Python. UI is TypeScript (Next.js 16). These layers communicate over MCP and HTTP — never share in-process state or import across boundaries.

### Adding a New Agent

Add it as a handoff target from the Tasks Manager Agent. The user always enters through the Tasks Manager Agent — never expose a sub-agent directly to the UI.

---

## Kubernetes Deployment

The system is built to run on Kubernetes from the start. Design decisions must account for this target from the first line of code.

- **Every service is a Deployment** with at least two replicas. No single points of failure.
- **Health checks first.** Every service exposes `/healthz` (liveness) and `/readyz` (readiness) before any feature work is merged.
- **Horizontal scaling.** Services must scale out by adding replicas. If a design requires shared mutable state across instances, reconsider the design.
- **Config via environment.** All configuration (endpoints, feature flags, timeouts) is injected through environment variables or ConfigMaps. No config files bundled into images.
- **Container images.** Each service has its own Dockerfile. Images are minimal (distroless or slim base). No root processes.
- **Resource limits.** Every container spec defines CPU and memory requests and limits. Unbounded containers will not be merged.
- **Service discovery.** Services communicate via Kubernetes Services by name. No hardcoded IPs or hostnames.
- **Graceful shutdown.** Every service handles SIGTERM and drains in-flight requests before exiting.

---

## Open Questions

- Primary task store: does the MCP server own its own database, or delegate to a separate persistence service? Decide before implementing `update_tasks` and `delete_tasks` so the id scheme is stable.
- Notification channels at launch: email only, or push as well?
- Multi-user / shared tasks: out of scope for v1, but the auth model must not preclude it.
