# @agiorcx/mcp — API Reference

## Installation

```bash
npm install @agiorcx/mcp
```

Requires `@agiorcx/sdk` as a peer dependency.

---

## Overview

`@agiorcx/mcp` provides two capabilities:

1. **`MCPToolAdapter`** — wrap any MCP server as an Agiorcx `ToolAdapter`, making it usable in any Agiorcx agent
2. **`AgentMCPServer`** — expose any Agiorcx agent as an MCP server, making it callable from any MCP host (Claude Desktop, Cursor, other agents)

---

## `MCPToolAdapter`

Wraps a remote MCP server as an Agiorcx `ToolAdapter`. The result plugs directly into any Agiorcx agent that accepts a `ToolAdapter`.

```typescript
import { MCPToolAdapter } from '@agiorcx/mcp';

const tavilyTool = new MCPToolAdapter({
  server: 'tavily-mcp',
  transport: 'streamable-http',     // MCP 2026 production transport
  endpoint: 'https://mcp.tavily.com',
  auth: { type: 'bearer', token: process.env.TAVILY_KEY },
  timeout: 10000,                   // ms — failure if exceeded, not silent hang
});

// Plug into SearchAgent
const searchAgent = new SearchAgent({
  id: 'web-search-agent',
  tool: tavilyTool,
});
```

When the MCP call fails, `execute()` returns a typed `AgentResult` failure — never throws.

**Supported transports:**
- `streamable-http` (recommended for production)
- `stdio` (local MCP servers, development)

---

## `AgentMCPServer`

Exposes any Agiorcx agent as an MCP server. External MCP hosts can then call your agent as an MCP tool.

```typescript
import { AgentMCPServer } from '@agiorcx/mcp';
import { z } from 'zod';

const server = new AgentMCPServer({
  agent: synthesisAgent,
  transport: 'streamable-http',
  port: 8080,
  schema: {
    name: 'orion_research',
    description: 'Run deep research synthesis across indexed sources in a Space',
    inputSchema: z.object({
      query: z.string(),
      spaceId: z.string(),
    }),
  },
});

await server.start();
// Orion's synthesis agent is now an MCP tool callable by any MCP host
```

---

## `MCPClientAdapter`

Discovers all tools exposed by an MCP server and returns them as `ToolAdapter[]` ready for agent registration.

```typescript
import { MCPClientAdapter } from '@agiorcx/mcp';

const client = new MCPClientAdapter({
  endpoint: 'https://mcp.myserver.com',
  transport: 'streamable-http',
  auth: { type: 'bearer', token: process.env.MCP_KEY },
});

const tools = await client.discoverTools();
// tools: ToolAdapter[] — ready to inject into any Agiorcx agent
```

---

## `MCPAuthProvider`

Pluggable auth interface for MCP connections.

```typescript
interface MCPAuthProvider {
  getHeaders(): Promise<Record<string, string>>;
  refresh?(): Promise<void>;
}

// Implementations shipped in @agiorcx/mcp:
// BearerTokenAuth — static bearer token
// OAuth2Auth       — OAuth2 client credentials flow
// ApiKeyAuth       — X-API-Key header
```
