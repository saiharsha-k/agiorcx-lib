# AgentResult Contract

## The Core Invariant

Every agent in the Agiorcx SDK returns `AgentResult<T>`. No agent returns raw data. No agent throws an exception into the calling code. This is the SDK's most important design decision.

```typescript
type AgentResult<T> =
  | {
      status: 'ok';
      data: T;
      agentId: string;
      stepId: string;
    }
  | {
      status: 'failed';
      agentId: string;
      stepId: string;
      reason: string;       // declared in the agent's failureReasons list
      source?: string;      // which source/tool triggered the failure
      retryable: false;     // structurally false — not configurable
    };
```

## Why `retryable: false` is Structural

`retryable` is not a field you can set to `true`. It does not exist as a configuration option. The SDK has no built-in retry loop.

This is intentional. Automatic retries in agent systems cause:
- **Token cost explosion** — retrying a failed LLM call N times multiplies cost by N with no guaranteed improvement
- **State obscurity** — after N retries, the system is in an unknown state that is hard to reason about
- **Failure masking** — intermittent failures that should surface as signals are hidden by retries

If a consuming system decides to retry, it must do so **explicitly** in its own code. That decision is then visible, auditable, and intentional.

## Declared Failure Reasons

Failure reasons are declared upfront in `defineAgent`. At runtime, only declared reasons can appear. This makes failure handling exhaustive:

```typescript
const SearchAgent = defineAgent({
  failureReasons: [
    'TOOL_UNAVAILABLE',
    'RATE_LIMITED',
    'EMPTY_RESULTS',
    'RESULT_PARSE_FAILED',
  ],
});

// At the call site — handle every declared reason
if (result.status === 'failed') {
  switch (result.reason) {
    case 'TOOL_UNAVAILABLE': // MCP server unreachable
    case 'RATE_LIMITED':     // quota exceeded
    case 'EMPTY_RESULTS':    // → trigger decision_required state
    case 'RESULT_PARSE_FAILED': // malformed response
  }
}
```

## Attribution Fields

Every failure includes:
- `agentId` — which agent failed (e.g. `'web-search-agent'`)
- `stepId` — which step within the agent failed (e.g. `'source-fetch'`)
- `source` — which external source or tool triggered the failure (e.g. `'https://example.com'`)

This makes the failure directly actionable — you know exactly where to look.
