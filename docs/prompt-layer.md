# PromptLayer

`PromptLayer` is a first-class SDK primitive for prompt versioning, slot validation, and change tracking. It is part of `@agiorcx/sdk/core` — not an external service.

## Why Prompt Versioning Matters in Production

Without versioning:
- A prompt change silently changes agent behaviour with no audit trail
- You cannot roll back a bad prompt without a code deployment
- You cannot compare outputs between prompt versions
- Injected instructions (e.g. Space custom instructions) have no history

With `PromptLayer`:
- Every prompt has a semver version and a mandatory `changelog` entry
- Slot validation ensures no prompt reaches the LLM with unfilled variables
- The full resolution record (which slots were filled with what values) is logged
- Two versions can be diffed to understand exact behavioural changes

## Versioning Semantics

| Bump | Meaning | `changelog` required? |
|---|---|---|
| `patch` (1.0.0 → 1.0.1) | Wording fix, no behavioural change | Optional |
| `minor` (1.0.0 → 1.1.0) | Structural change, same intent | Required |
| `major` (1.0.0 → 2.0.0) | Intent change — different agent behaviour | Required, non-empty |

## Slot Validation

Slots are declared upfront in the prompt definition. At resolve time, every declared slot must be filled. A missing slot returns `AgentResult` failed with `reason: 'SLOT_NOT_FILLED'` — the prompt never reaches the LLM with a literal `{{slot_name}}` in it.

```typescript
// This fails with SLOT_NOT_FILLED for 'max_subtasks'
const result = promptLayer.resolve('planner-system-prompt', {
  custom_instructions: 'Focus on UK market',
  available_agents: 'SearchAgent',
  source_context: sources,
  // max_subtasks missing
});
// result.status === 'failed'
// result.reason === 'SLOT_NOT_FILLED'
// result.source === 'max_subtasks'
```

## Storage

`PromptLayer` stores prompts via `PromptStoreAdapter`. Three implementations ship:

| Adapter | Use Case |
|---|---|
| `FilePromptStoreAdapter` | Local development — prompts stored as `.json` files |
| `PostgresPromptStoreAdapter` | Production — versioned rows in `prompts` table |
| `InMemoryPromptStoreAdapter` | Testing — no persistence, cleared between runs |
