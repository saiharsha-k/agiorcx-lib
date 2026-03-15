# Architecture Guide

## Overview

Agiorcx SDK is structured as a strict layered system. Each layer depends only on layers below it. No layer imports from a layer above it. This is enforced as a lint rule and documented here as the primary architectural invariant.

```
core  ←  adapters  ←  skills  ←  agents  ←  testing
```

---

## Layer Definitions

### `core/` — Zero-Dependency Primitives

Contains only types, interfaces, and logic that have **zero external dependencies**. Nothing in `core/` knows about LiteLLM, Qdrant, Tavily, or any third-party library. It is independently publishable and testable.

| Module | Role |
|---|---|
| `defineAgent` | Typed agent declaration — id, role, capabilities, required adapters, declared failure reasons |
| `AgentResult<T>` | Typed union result — every agent returns this, never raw data or throws |
| `DeterministicStateRunner` | Forward-only FSM — mandatory states baked in, custom states configurable |
| `PromptLayer` | Prompt registry — register, resolve (slot validation), version (semver), diff |
| `StepAuditLog` | Audit entry interface — every state transition and agent result is logged |
| `OutputParser<T>` | Zod-schema structured output parsing — parse failure is a typed AgentResult |
| `AgentMemory` | Three-tier memory — working (ephemeral), episodic (session), semantic (vector) |
| `Guardrails` | Input / output / source guardrails — check() returns AgentResult, never throws |
| `ObservabilityHooks` | Lifecycle hooks — fire-and-forget, never block the agent pipeline |
| `ContextWindowManager` | Token budget management — fit() surfaces excluded chunks, never drops silently |

### `adapters/` — Boundary Contracts

Interfaces that define how the SDK communicates with external systems. Third-party libraries implement these interfaces. The SDK never imports a third-party library directly.

| Adapter | Implemented By (examples) |
|---|---|
| `LLMAdapter` | LiteLLM, direct OpenAI/Anthropic clients |
| `VectorStoreAdapter` | Qdrant, Pinecone, pgvector |
| `ToolAdapter` | Any MCP tool via `MCPToolAdapter`, custom tools |
| `NotificationAdapter` | WebSocket, email (Resend), push (web-push) |
| `PromptStoreAdapter` | PostgreSQL, filesystem, in-memory |
| `MemoryAdapter` | Redis (working), PostgreSQL (episodic), Qdrant (semantic) |
| `AuditLogAdapter` | PostgreSQL (production), in-memory (testing) |

### `skills/` — Coordination Patterns

Reusable behavioural patterns that agents apply. A skill defines *how* an agent coordinates its tools and handles results — not *what* tools it uses (that is injected via adapters).

| Skill | Pattern |
|---|---|
| `SearchSkill` | Parallel fan-out, result deduplication, relevance scoring, empty-result gate |
| `RetrievalSkill` | Hybrid sparse+dense retrieval, namespace-scoped, chunk ID preservation |
| `PlanSkill` | Query decomposition, sub-task generation, agent assignment, dependency ordering |
| `ConfirmationSkill` | Blocking pause → `decision_required` → resume or terminate |

### `agents/` — Default Execution Units

Four production-ready agents built entirely on the SDK's own public interfaces. No special internals.

| Agent | Skill | Default Tools | Injected Adapter |
|---|---|---|---|
| `SearchAgent` | `SearchSkill` | `SourceFetchTool`, `ParserTool` | `ToolAdapter` |
| `RetrievalAgent` | `RetrievalSkill` | `ChunkTool`, `EmbedTool` | `VectorStoreAdapter` |
| `PlanningAgent` | `PlanSkill` | — | `LLMAdapter` |
| `HitlAgent` | `ConfirmationSkill` | — | `NotificationAdapter` |

### `testing/` — Test Utilities

Mock implementations of all adapters plus `AgentTestHarness` and `assertAgentResult`. Tests run the full `DeterministicStateRunner` with mocked dependencies — no shortcuts.

---

## Mandatory State Invariants

The `DeterministicStateRunner` enforces three states that **cannot be removed** regardless of custom state configuration:

| Mandatory State | Meaning |
|---|---|
| `queued` | Job entered the system and is awaiting execution |
| `decision_required` | Agent is paused — explicit human or system decision needed to proceed |
| `failed` | Typed `AgentResult` failure — always surfaces with full attribution |

Custom states (e.g. `planning`, `executing`, `synthesizing`, `review`, `complete`) are defined by the consuming application. They sit between the mandatory states.

---

## The `AgentResult<T>` Contract

This is the SDK's core invariant. No agent returns raw data. No agent throws.

```typescript
type AgentResult<T> =
  | { status: 'ok'; data: T; agentId: string; stepId: string }
  | {
      status: 'failed';
      agentId: string;
      stepId: string;
      reason: string;
      source?: string;
      retryable: false;   // structurally false — caller decides retry, SDK never does
    };
```

`retryable` is not configurable. It is structurally `false`. If a consuming system decides to retry, it must do so explicitly and the decision is logged. The SDK never retries autonomously.

---

## Cordax — Multi-Agent Coordination (Separate Library)

Agents in `@agiorcx/sdk` are intentionally **dumb execution units**. They do not know about other agents. They do not decide routing.

**Cordax** is a separate coordination library that sits above `@agiorcx/sdk`. It decides which agent runs, in what order, and how typed context is passed between agents. This separation keeps individual agents testable in isolation and keeps the SDK free of orchestration opinions.

`@agiorcx/sdk` + `@agiorcx/mcp` → agent primitives  
`cordax` → inter-agent coordination and routing  
`agiorcx-platform` → enterprise governance, contract ledger, cross-org trust

---

## MCP Integration Model

MCP handles tool **discovery and transport**. Agiorcx SDK handles agent **coordination and failure**.

```
MCP Server (Tavily, Firecrawl, etc.)
        ↓
MCPToolAdapter (@agiorcx/mcp)
        ↓  implements
ToolAdapter (@agiorcx/sdk/adapters)
        ↓  injected into
SearchAgent (@agiorcx/sdk/agents)
        ↓  governed by
DeterministicStateRunner + AgentResult contract
```

MCP is the USB standard. Agiorcx SDK is the OS that decides when and how to use USB devices.
