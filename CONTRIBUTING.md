# Contributing to Agiorcx Library

Thank you for your interest in contributing. This document outlines how to contribute effectively.

## Core Principle

**The adapter-first rule is non-negotiable.** If you are adding support for a new LLM, vector store, tool, or external service, implement the relevant adapter interface. Do not modify `core/` primitives to accommodate a specific third-party library.

## Repository Structure

```
agiorcx-lib/
├── packages/
│   ├── sdk/          @agiorcx/sdk
│   └── mcp/          @agiorcx/mcp
├── docs/             Documentation
├── examples/         Reference implementations
└── CONTRIBUTING.md
```

## Development Setup

```bash
git clone https://github.com/saiharsha-k/agiorcx-lib.git
cd agiorcx-lib
npm install
npm run build
npm run test
```

## Contribution Types

### Adding a New Adapter Implementation

If you want to contribute an adapter implementation (e.g. a Pinecone `VectorStoreAdapter`):

1. Implement the interface from `@agiorcx/sdk/adapters`
2. All methods must return `AgentResult<T>` — never throw
3. Write unit tests using `MockAdapters` from `@agiorcx/sdk/testing`
4. Add an example in `examples/` showing the adapter in use
5. Document the adapter in `docs/adapters.md`

### Adding a New Default Agent

New agents must:
1. Be built entirely on public SDK interfaces — no internal access
2. Declare all `failureReasons` upfront in `defineAgent`
3. Return `AgentResult<T>` from every public method
4. Have 100% test coverage via `AgentTestHarness`
5. Be documented in `docs/default-agents.md`

### Adding a New Skill

Skills must:
1. Be pure coordination patterns — no direct third-party imports
2. Accept adapters as constructor injection
3. Be documented in `docs/` with pattern description and usage example

### Modifying `core/`

Changes to `core/` are the highest bar. Open an issue first describing the proposed change and the production use case it solves. Core PRs require:
- Rationale for why the change belongs in `core/` vs `adapters/` or `skills/`
- No new external dependencies
- Full test coverage
- Documentation update

## Pull Request Guidelines

- One concern per PR — do not mix adapter work with core changes
- All tests must pass: `npm run test`
- TypeScript strict mode — no `any`, no type assertions without comment
- `AgentResult` contract must be preserved — no new throws in agent execution paths
- Update relevant docs alongside code changes

## Issue Reporting

When reporting a bug, include:
- The `AgentResult` failure object (if applicable) — `agentId`, `stepId`, `reason`, `source`
- The state the `DeterministicStateRunner` was in when the issue occurred
- Minimal reproduction case

## Code of Conduct

Be direct, be precise, be respectful. This is a production engineering project — clarity and correctness are valued over enthusiasm.

---

*Maintained by [Sai Harsha Kondaveeti](https://github.com/saiharsha-k)*
