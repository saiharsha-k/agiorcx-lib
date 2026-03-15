# Default Agents

Four production-ready agents ship with `@agiorcx/sdk/agents`. Each is built entirely on the SDK's own public interfaces — no special internals. They serve as both usable defaults and living documentation of how to build agents correctly with the SDK.

---

## `SearchAgent`

**Skill**: `SearchSkill` — parallel fan-out, deduplication, relevance scoring

**Default tools** (bundled, overridable):
- `SourceFetchTool` — fetches raw content from URLs returned by search
- `ParserTool` — converts HTML/PDF/DOCX/TXT to `ParsedDocument`

**Injected adapter**: `ToolAdapter` — any search tool (Tavily via MCP, Brave, custom)

**Failure reasons**: `TOOL_UNAVAILABLE`, `RATE_LIMITED`, `EMPTY_RESULTS`, `RESULT_PARSE_FAILED`

**When `EMPTY_RESULTS`**: transitions to `decision_required` — does not silently proceed with zero results

---

## `RetrievalAgent`

**Skill**: `RetrievalSkill` — hybrid sparse+dense retrieval, namespace-scoped

**Default tools** (bundled, overridable):
- `ChunkTool` — splits documents into chunks with deterministic IDs
- `EmbedTool` — embeds chunks and queries; batched

**Injected adapter**: `VectorStoreAdapter` — Qdrant, Pinecone, pgvector

**Key config**: `preserveChunkIds: true` — required if downstream synthesis needs citation injection

**Failure reasons**: `NAMESPACE_EMPTY`, `STORE_UNAVAILABLE`, `EMBEDDING_FAILED`, `THRESHOLD_NOT_MET`

---

## `PlanningAgent`

**Skill**: `PlanSkill` — query decomposition, sub-task generation, agent assignment

**Returns**:
```typescript
type PlanResult = {
  subTasks: Array<{
    taskId: string;
    description: string;
    assignedAgent: string;
    dependsOn: string[];
    sourceRouting: string[];
  }>;
  executionOrder: 'parallel' | 'sequential' | 'mixed';
};
```

**Injected adapter**: `LLMAdapter` — any LLM (Claude 3.7, GPT-o3 via LiteLLM)

**`customInstructions`**: injected into system prompt via `PromptLayer` slot — Space-level constraints propagate here

**When query is ambiguous**: transitions to `decision_required` — never guesses

**Failure reasons**: `LLM_UNAVAILABLE`, `QUERY_TOO_AMBIGUOUS`, `NO_AGENTS_AVAILABLE`, `MAX_SUBTASKS_EXCEEDED`

---

## `HitlAgent`

**Skill**: `ConfirmationSkill` — blocking pause with timeout and notification

The only agent designed to **block execution**. Transitions the job to `decision_required` and waits for explicit human resolution.

**Trigger modes**:
- `'always'` — always pauses at this point in the pipeline
- `'on_condition'` — pauses only when condition function returns true

**Resolution**:
- `APPROVED` → job resumes from next state
- `REJECTED` → job transitions to `failed` with `reason: 'HUMAN_REJECTED'`
- `TIMEOUT_EXPIRED` → auto-rejects if `timeout` seconds elapse with no response

**Failure reasons**: `HUMAN_REJECTED`, `TIMEOUT_EXPIRED`, `NOTIFICATION_FAILED`

**Injected adapter**: `NotificationAdapter` — WebSocket, email, push
