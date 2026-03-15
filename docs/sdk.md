# @agiorcx/sdk — API Reference

## Installation

```bash
npm install @agiorcx/sdk
```

---

## `core/`

### `defineAgent`

Declares a typed agent manifest. This is the entry point for every agent in the system.

```typescript
import { defineAgent } from '@agiorcx/sdk/core';

const MyAgent = defineAgent({
  id: 'my-agent',
  role: 'research-planner',
  capabilities: ['query-decomposition', 'source-routing'],
  requires: {
    llm: LLMAdapter,
    memory: MemoryAdapter,
  },
  failureReasons: [
    'LLM_UNAVAILABLE',
    'QUERY_TOO_AMBIGUOUS',
    'CONTEXT_LOAD_FAILED',
  ],
});
```

---

### `AgentResult<T>`

Every agent returns this type. Never raw data. Never throws.

```typescript
type AgentResult<T> =
  | { status: 'ok'; data: T; agentId: string; stepId: string }
  | {
      status: 'failed';
      agentId: string;
      stepId: string;
      reason: string;
      source?: string;
      retryable: false;
    };
```

**Usage pattern:**
```typescript
const result = await agent.run(input);

if (result.status === 'ok') {
  // use result.data
} else {
  // result.reason — declared failure reason
  // result.agentId — which agent failed
  // result.stepId — which step failed
  // result.source — which source/tool triggered the failure (if applicable)
  // result.retryable — always false; you decide if retry is warranted
}
```

---

### `DeterministicStateRunner`

Forward-only FSM. Mandatory states (`queued`, `decision_required`, `failed`) are always present. Custom states are configurable.

```typescript
import { createStateRunner } from '@agiorcx/sdk/core';

const runner = createStateRunner({
  // mandatory states are injected automatically — do not declare them
  customStates: ['planning', 'executing', 'synthesizing', 'review', 'complete'],
  initial: 'queued',
  transitions: [
    { from: 'queued',       to: 'planning',          on: 'START' },
    { from: 'planning',     to: 'executing',         on: 'PLAN_READY' },
    { from: 'planning',     to: 'decision_required', on: 'PLAN_AMBIGUOUS' },
    { from: 'planning',     to: 'failed',            on: 'PLAN_FAILED' },
    { from: 'executing',    to: 'synthesizing',      on: 'EXECUTION_COMPLETE' },
    { from: 'executing',    to: 'failed',            on: 'EXECUTION_FAILED' },
    { from: 'synthesizing', to: 'review',            on: 'SYNTHESIS_READY' },
    { from: 'synthesizing', to: 'failed',            on: 'SYNTHESIS_FAILED' },
    { from: 'review',       to: 'complete',          on: 'USER_APPROVED' },
    { from: 'review',       to: 'decision_required', on: 'NEEDS_CONFIRMATION' },
    { from: 'decision_required', to: 'executing',   on: 'CONFIRMED' },
    { from: 'decision_required', to: 'failed',      on: 'REJECTED' },
  ],
  onTransition: async (from, to, event, context) => {
    // called and awaited BEFORE next state executes
    await auditLog.write({ from, to, event, context, timestamp: Date.now() });
  },
});

await runner.dispatch('START', context);
console.log(runner.currentState); // 'planning'
```

**Invariant**: `onTransition` is awaited before the next state handler runs. State is never assumed — always confirmed written.

---

### `PromptLayer`

Prompt registry with semver versioning, slot validation, and diff.

```typescript
import { PromptLayer } from '@agiorcx/sdk/core';

const promptLayer = new PromptLayer({ store: myPromptStoreAdapter });

// Register a versioned prompt
promptLayer.register({
  id: 'planner-system-prompt',
  version: '1.0.0',
  role: 'system',
  template: `
    You are a research planning agent.
    {{custom_instructions}}
    Available agents: {{available_agents}}
    Sources: {{source_context}}
    Max sub-tasks: {{max_subtasks}}
  `,
  slots: ['custom_instructions', 'available_agents', 'source_context', 'max_subtasks'],
  changelog: 'Initial planner prompt',
});

// Resolve — slot validation returns AgentResult
const result = promptLayer.resolve('planner-system-prompt', {
  custom_instructions: 'Focus on UK market only',
  available_agents: 'SearchAgent, RetrievalAgent',
  source_context: sourceList,
  max_subtasks: '8',
});

if (result.status === 'ok') {
  // result.data.content — fully resolved prompt string
  // result.data.slotValues — audit trail of what was injected
}
// If a slot is missing → AgentResult failed with reason: 'SLOT_NOT_FILLED'
```

**Versioning semantics:**
- `patch` (1.0.0 → 1.0.1) — wording fix, no behavioural change
- `minor` (1.0.0 → 1.1.0) — structural change, same intent
- `major` (1.0.0 → 2.0.0) — intent change; non-empty `changelog` required

---

### `OutputParser<T>`

Zod-schema-based structured output parsing.

```typescript
import { OutputParser } from '@agiorcx/sdk/core';
import { z } from 'zod';

const ReportSchema = z.object({
  title: z.string(),
  sections: z.array(z.object({
    heading: z.string(),
    content: z.string(),
    citations: z.array(z.object({
      chunkId: z.string(),
      sourceId: z.string(),
    })),
  })),
  uncitedClaims: z.array(z.string()),
});

const parser = new OutputParser({
  schema: ReportSchema,
  // repair is explicit opt-in — not automatic
  repair: async (raw) => myLLM.fixJson(raw),
});

const result = parser.parse(llmOutput);
// result is AgentResult<z.infer<typeof ReportSchema>>
```

---

### `AgentMemory`

Three-tier memory backed by pluggable `MemoryAdapter`.

```typescript
import { AgentMemory } from '@agiorcx/sdk/core';

const memory = new AgentMemory({
  working: redisMemoryAdapter,      // ephemeral, current job only
  episodic: postgresMemoryAdapter,  // persists across sessions, Space-scoped
  semantic: qdrantMemoryAdapter,    // vector-searchable, Space-scoped
});

// Working memory
await memory.working.set('plan_result', planData);
const plan = await memory.working.get('plan_result');

// Episodic memory
await memory.episodic.write({ key: 'last_query', value: query }, spaceId);
const history = await memory.episodic.recall(spaceId, 10);

// Semantic memory
await memory.semantic.store('Report summary text', metadata, spaceId);
const relevant = await memory.semantic.search('UK AI jobs', spaceId, 5);
```

---

### `Guardrails`

```typescript
import { Guardrail } from '@agiorcx/sdk/core';

const noWikipedia: Guardrail = {
  id: 'block-wikipedia',
  type: 'source',
  check: (source) =>
    source.url?.includes('wikipedia.org')
      ? { status: 'failed', agentId: 'guardrail', stepId: 'source-check',
          reason: 'SOURCE_BLOCKED_BY_GUARDRAIL', source: source.url, retryable: false }
      : { status: 'ok', data: 'pass', agentId: 'guardrail', stepId: 'source-check' },
  reason: 'Wikipedia blocked by Space custom instructions',
};
```

**Three guardrail types:**
- `input` — validates agent input before execution
- `output` — validates agent output before returning
- `source` — validates sources before they enter the retrieval or synthesis pipeline

---

### `ContextWindowManager`

```typescript
import { ContextWindowManager } from '@agiorcx/sdk/core';

const manager = new ContextWindowManager({
  strategy: 'priority-rank', // 'truncate' | 'summarise' | 'priority-rank'
});

const result = manager.fit(chunks, 8000); // 8000 token budget

if (result.status === 'ok') {
  // result.data.included — chunks that fit
  // result.data.excluded — chunks dropped (surfaced, never silent)
  // result.data.totalTokens — actual token count used
}
```

---

## `agents/`

### `SearchAgent`

```typescript
import { SearchAgent } from '@agiorcx/sdk/agents';

const agent = new SearchAgent({
  id: 'web-search-agent',
  tool: tavilyMCPAdapter,           // ToolAdapter
  maxConcurrency: 3,
  topK: 10,
  onEmptyResult: 'decision_required',
  deduplication: true,
  defaultTools: {
    sourceFetch: true,              // SourceFetchTool — ON by default
    parser: true,                   // ParserTool — ON by default
    parserOptions: {
      formats: ['html', 'pdf', 'docx', 'txt'],
      cleanHtml: true,
      extractMetadata: true,
    },
  },
});
```

### `RetrievalAgent`

```typescript
import { RetrievalAgent } from '@agiorcx/sdk/agents';

const agent = new RetrievalAgent({
  id: 'document-retrieval-agent',
  store: qdrantAdapter,             // VectorStoreAdapter
  namespace: spaceId,
  topK: 15,
  hybridSearch: true,
  preserveChunkIds: true,           // required for citation injection
  defaultTools: {
    chunker: true,
    chunkerOptions: { strategy: 'recursive', chunkSize: 512, overlap: 64 },
    embedder: true,
    embedderOptions: { model: 'text-embedding-3-small', batchSize: 100 },
  },
});
```

### `PlanningAgent`

```typescript
import { PlanningAgent } from '@agiorcx/sdk/agents';

const agent = new PlanningAgent({
  id: 'planner-agent',
  llm: liteLLMAdapter,              // LLMAdapter
  availableAgents: [searchAgent, retrievalAgent],
  customInstructions: space.instructions,
  onAmbiguousQuery: 'decision_required',
  maxSubTasks: 8,
  promptLayer: promptLayer,         // resolves 'planner-system-prompt'
});
```

### `HitlAgent`

```typescript
import { HitlAgent } from '@agiorcx/sdk/agents';

const agent = new HitlAgent({
  id: 'hitl-agent',
  trigger: 'on_condition',
  condition: (ctx) => ctx.action === 'drive_write' || ctx.action === 'report_finalize',
  timeout: 3600,
  onTimeout: 'rejected',
  onApproved: 'resume',
  onRejected: 'failed',
  notification: {
    channels: ['websocket', 'email'],
    adapter: notificationAdapter,
  },
});
```

---

## `testing/`

```typescript
import {
  MockLLMAdapter,
  MockVectorStoreAdapter,
  MockToolAdapter,
  MockMemoryAdapter,
  AgentTestHarness,
  assertAgentResult,
} from '@agiorcx/sdk/testing';

const harness = new AgentTestHarness({
  agent: PlanningAgent,
  adapters: {
    llm: new MockLLMAdapter({ response: mockPlanResponse }),
    memory: new MockMemoryAdapter(),
  },
  guardrails: [noWikipedia],
  hooks: mockObservabilityHooks,
});

const result = await harness.run({ query: 'Research UK AI job market' });

assertAgentResult(result)
  .isOk()
  .hasState('complete')
  .stateTransitions(['queued', 'planning', 'executing', 'synthesizing', 'review', 'complete'])
  .guardrailNotTriggered('block-wikipedia');
```
