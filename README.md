# Agiorcx Library

> **Deterministic agent coordination for production AI systems.**
> The open-source SDK powering the [Internet of Agents](https://github.com/saiharsha-k/agiorcx-lib).

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![npm version](https://img.shields.io/badge/npm-v0.1.0--alpha-indigo)](https://www.npmjs.com/package/@agiorcx/sdk)
[![TypeScript](https://img.shields.io/badge/TypeScript-strict-blue)](https://www.typescriptlang.org/)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

---

## The Problem

Every major agent framework — LangChain, LlamaIndex, CrewAI — shares the same production failure modes:

- **Silent retries** that burn tokens and obscure where execution actually failed
- **Non-deterministic state** where you can't tell which step a multi-agent pipeline is in
- **No typed failure contracts** — errors are raw exceptions, not attributable results
- **Frameworks as ceilings** — LLMs, vector stores, and tools are locked in, not pluggable
- **No prompt versioning** — a prompt change silently changes agent behaviour with no audit trail

Agiorcx SDK inverts all of this.

---

## What Agiorcx SDK Is

`@agiorcx/sdk` is a **TypeScript-first agent coordination library** built on four non-negotiable principles:

1. **Fail fast, fail loud** — every agent returns a typed `AgentResult<T>` union. Failures are attributed to the exact agent, step, and source. `retryable` defaults to `false`. The SDK never retries autonomously.
2. **Deterministic state** — a forward-only Finite State Machine (`DeterministicStateRunner`) governs every agent job. State transitions are logged before executing. State never rolls back silently.
3. **Adapters, not opinions** — LLMs, vector stores, tools, and memory are injected via clean interfaces. LangChain, LiteLLM, Qdrant, and any MCP server become adapters. Agiorcx SDK is the coordination layer above them.
4. **Production observability by default** — `StepAuditLog`, `ObservabilityHooks`, and `PromptLayer` versioning are first-class primitives, not afterthoughts.

`@agiorcx/mcp` bridges the [Model Context Protocol](https://modelcontextprotocol.io) ecosystem — wrap any MCP server as an Agiorcx tool, or expose any Agiorcx agent as an MCP server.

---

## Packages

| Package | Description | Docs |
|---|---|---|
| [`@agiorcx/sdk`](./packages/sdk) | Core SDK — primitives, adapters, skills, agents, testing | [SDK Docs](./docs/sdk.md) |
| [`@agiorcx/mcp`](./packages/mcp) | MCP bridge — wrap MCP tools, expose agents as MCP servers | [MCP Docs](./docs/mcp.md) |

---

## Quick Start

```bash
npm install @agiorcx/sdk @agiorcx/mcp
```

```typescript
import { defineAgent, AgentResult } from '@agiorcx/sdk/core';
import { SearchAgent } from '@agiorcx/sdk/agents';
import { MCPToolAdapter } from '@agiorcx/mcp';

// 1. Wrap any MCP tool as an Agiorcx ToolAdapter
const tavilyTool = new MCPToolAdapter({
  server: 'tavily-mcp',
  transport: 'streamable-http',
  endpoint: 'https://mcp.tavily.com',
  auth: { type: 'bearer', token: process.env.TAVILY_KEY },
});

// 2. Instantiate a default agent with your tool injected
const searchAgent = new SearchAgent({
  id: 'web-search-agent',
  tool: tavilyTool,
  maxConcurrency: 3,
  onEmptyResult: 'decision_required',
});

// 3. Run the agent — result is always a typed AgentResult
const result = await searchAgent.run({
  query: 'Enterprise AI roles in London 2026',
  namespace: 'space_001',
});

// 4. Handle result — no surprises
if (result.status === 'ok') {
  console.log(result.data);
} else {
  // Full attribution — no generic error
  console.error(`Failed: ${result.reason} | Agent: ${result.agentId} | Step: ${result.stepId}`);
}
```

---

## Architecture

```
@agiorcx/sdk
├── core/              ← Zero-dependency primitives
│   ├── defineAgent
│   ├── AgentResult<T>
│   ├── DeterministicStateRunner
│   ├── PromptLayer
│   ├── StepAuditLog
│   ├── OutputParser
│   ├── AgentMemory
│   ├── Guardrails
│   ├── ObservabilityHooks
│   └── ContextWindowManager
│
├── adapters/          ← Boundary contracts (depend on core)
│   ├── LLMAdapter
│   ├── VectorStoreAdapter
│   ├── ToolAdapter
│   ├── NotificationAdapter
│   ├── PromptStoreAdapter
│   ├── MemoryAdapter
│   └── AuditLogAdapter
│
├── skills/            ← Coordination patterns (depend on core + adapters)
│   ├── SearchSkill
│   ├── RetrievalSkill
│   ├── PlanSkill
│   └── ConfirmationSkill
│
├── agents/            ← Default execution units (depend on all above)
│   ├── SearchAgent     (+ SourceFetchTool, ParserTool)
│   ├── RetrievalAgent  (+ ChunkTool, EmbedTool)
│   ├── PlanningAgent
│   └── HitlAgent
│
└── testing/           ← Test utilities (mock adapters, harness, assertions)

@agiorcx/mcp
├── MCPToolAdapter     ← Any MCP server → Agiorcx ToolAdapter
├── AgentMCPServer     ← Any Agiorcx agent → MCP server
├── MCPClientAdapter
└── MCPAuthProvider
```

Dependency flow is strictly unidirectional: `core ← adapters ← skills ← agents`.

See the full [Architecture Guide](./docs/architecture.md).

---

## Why Not LangChain / LlamaIndex / OpenAI Agents SDK?

| Capability | LangChain | LlamaIndex | OpenAI Agents | **Agiorcx SDK** |
|---|---|---|---|---|
| Typed failure contract | ❌ | ❌ | ❌ | ✅ `AgentResult<T>` |
| Deterministic FSM | ⚠️ LangGraph | ❌ | ❌ | ✅ `DeterministicStateRunner` |
| Prompt versioning | ❌ | ❌ | ❌ | ✅ `PromptLayer` |
| Adapter-first (swap any LLM/store) | ❌ | ❌ | ❌ OpenAI-locked | ✅ |
| MCP-native | ⚠️ Community | ⚠️ Community | ✅ | ✅ `@agiorcx/mcp` |
| Built-in audit trail | ❌ | ❌ | ⚠️ | ✅ `StepAuditLog` |
| No silent retries | ❌ | ❌ | ❌ | ✅ `retryable: false` |
| Testing utilities | ✅ | ✅ | ✅ | ✅ `testing/` |

**MCP vs Agiorcx SDK**: MCP is the USB standard — it defines how tools are described and called. Agiorcx SDK is the operating system — it decides when to call tools, coordinates agents, enforces state, and attributes failures. They are complementary, not competing.

---

## Reference Implementation

**Orion Deep Research Agent** is the production reference implementation of `@agiorcx/sdk`. Every default agent, skill, and adapter in the SDK is consumed by Orion in production.

Orion ships: `PlanningAgent → SearchAgent + RetrievalAgent → SynthesisAgent → HitlAgent`, all coordinated via `DeterministicStateRunner` with `PromptLayer` versioning, `Guardrails` enforcing Space custom instructions, and `AgentMemory` persisting research context across sessions.

See [`examples/orion-research-agent`](./examples/) for the full reference implementation.

---

## Documentation

| Doc | Description |
|---|---|
| [Architecture](./docs/architecture.md) | System design, layer model, dependency rules |
| [SDK Reference](./docs/sdk.md) | Full `@agiorcx/sdk` API reference |
| [MCP Reference](./docs/mcp.md) | Full `@agiorcx/mcp` API reference |
| [AgentResult Contract](./docs/agent-result.md) | Failure contract design and usage |
| [DeterministicStateRunner](./docs/state-runner.md) | FSM design, mandatory states, custom states |
| [PromptLayer](./docs/prompt-layer.md) | Prompt versioning, slot validation, diff |
| [Default Agents](./docs/default-agents.md) | SearchAgent, RetrievalAgent, PlanningAgent, HitlAgent |
| [Adapters](./docs/adapters.md) | Implementing custom adapters |
| [Contributing](./CONTRIBUTING.md) | How to contribute |

---

## Roadmap

**v0.1 (Current)**
- [ ] `core/` — all 10 primitives
- [ ] `adapters/` — all 7 interfaces
- [ ] `skills/` — SearchSkill, RetrievalSkill, PlanSkill, ConfirmationSkill
- [ ] `agents/` — SearchAgent, RetrievalAgent, PlanningAgent, HitlAgent
- [ ] `testing/` — MockAdapters, AgentTestHarness, assertAgentResult
- [ ] `@agiorcx/mcp` — MCPToolAdapter, AgentMCPServer
- [ ] Orion reference implementation in `examples/`

**v0.2**
- [ ] Python bindings
- [ ] `SynthesisAgent` default agent
- [ ] `ContextWindowManager` streaming strategy
- [ ] Integration packages (`@agiorcx/openai`, `@agiorcx/anthropic`, `@agiorcx/qdrant`)

**v1.0**
- [ ] Stable API contract
- [ ] Full documentation site
- [ ] Cordax multi-agent coordination layer

---

## Contributing

Contributions are welcome. Please read [CONTRIBUTING.md](./CONTRIBUTING.md) before opening a PR.

This repository follows the adapter-first principle — if you are adding support for a new LLM, vector store, or tool, implement the relevant adapter interface rather than modifying core primitives.

---

## License

MIT — see [LICENSE](./LICENSE).

---

*Built by [Sai Harsha Kondaveeti](https://github.com/saiharsha-k) | Garvaman Intelligence Labs*  
*Part of the Agiorcx Platform — Internet of Agents infrastructure*
