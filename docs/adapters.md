# Adapters

Adapters are the boundary contracts between `@agiorcx/sdk` and external systems. Every third-party library — LiteLLM, Qdrant, Tavily, Redis — enters the SDK by implementing an adapter interface.

## The Rule

The SDK never imports a third-party library directly. If you want to use Qdrant, you implement `VectorStoreAdapter` and inject it. This means:

- You can swap any external dependency without changing agent code
- You can mock any adapter in tests without a real external service
- The SDK has zero third-party runtime dependencies in `core/`

## Adapter Interfaces

### `LLMAdapter`

```typescript
interface LLMAdapter {
  complete(prompt: string, options?: LLMOptions): Promise<AgentResult<LLMResponse>>;
  embed(text: string | string[]): Promise<AgentResult<number[][]>>;
  stream(prompt: string, options?: LLMOptions): AsyncGenerator<string>;
}
```

Implemented by: LiteLLM wrapper, direct OpenAI client, Anthropic client.

### `VectorStoreAdapter`

```typescript
interface VectorStoreAdapter {
  upsert(chunks: EmbeddedChunk[], namespace: string): Promise<AgentResult<void>>;
  query(embedding: number[], namespace: string, topK: number, options?: QueryOptions): Promise<AgentResult<Chunk[]>>;
  delete(ids: string[], namespace: string): Promise<AgentResult<void>>;
  namespaceExists(namespace: string): Promise<AgentResult<boolean>>;
}
```

Implemented by: Qdrant, Pinecone, pgvector.

### `ToolAdapter`

```typescript
interface ToolAdapter {
  name: string;
  description: string;
  inputSchema: ZodSchema;
  execute(input: unknown): Promise<AgentResult<unknown>>;
}
```

Implemented by: `MCPToolAdapter` (from `@agiorcx/mcp`), custom tools.

### `NotificationAdapter`

```typescript
interface NotificationAdapter {
  send(channel: NotificationChannel, message: string, userId: string): Promise<AgentResult<void>>;
}

type NotificationChannel = 'websocket' | 'email' | 'push';
```

### `MemoryAdapter`

```typescript
interface MemoryAdapter {
  // Working tier
  workingSet(key: string, value: unknown, ttl?: number): Promise<AgentResult<void>>;
  workingGet(key: string): Promise<AgentResult<unknown>>;
  workingClear(jobId: string): Promise<AgentResult<void>>;

  // Episodic tier
  episodicWrite(entry: MemoryEntry, namespace: string): Promise<AgentResult<void>>;
  episodicRecall(namespace: string, limit?: number): Promise<AgentResult<MemoryEntry[]>>;

  // Semantic tier
  semanticStore(text: string, metadata: unknown, namespace: string): Promise<AgentResult<void>>;
  semanticSearch(query: string, namespace: string, topK: number): Promise<AgentResult<MemoryEntry[]>>;
}
```

### `PromptStoreAdapter`

```typescript
interface PromptStoreAdapter {
  save(prompt: PromptDefinition): Promise<AgentResult<void>>;
  load(id: string, version?: string): Promise<AgentResult<PromptDefinition>>;
  list(id: string): Promise<AgentResult<PromptDefinition[]>>;
}
```

### `AuditLogAdapter`

```typescript
interface AuditLogAdapter {
  write(entry: AuditEntry): Promise<void>;
  read(jobId: string): Promise<AuditEntry[]>;
}
```

Note: `write` does not return `AgentResult` — audit log writes must never fail silently but also must never block the agent pipeline. Failures in `write` are logged to `ObservabilityHooks.onError` only.

## Implementing a Custom Adapter

```typescript
import { VectorStoreAdapter, AgentResult, EmbeddedChunk, Chunk } from '@agiorcx/sdk/adapters';

export class MyVectorStoreAdapter implements VectorStoreAdapter {
  async upsert(chunks: EmbeddedChunk[], namespace: string): Promise<AgentResult<void>> {
    try {
      await myVectorDB.insert(chunks, namespace);
      return { status: 'ok', data: undefined, agentId: 'vector-store', stepId: 'upsert' };
    } catch (err) {
      return {
        status: 'failed',
        agentId: 'vector-store',
        stepId: 'upsert',
        reason: 'STORE_UNAVAILABLE',
        source: namespace,
        retryable: false,
      };
    }
  }
  // ... implement other methods
}
```
