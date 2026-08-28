---
name: mcp-server-builder
description: Comprehensive development and deployment guide for Model Context Protocol (MCP) servers. Covers local STDIO implementations across 8 programming languages (TypeScript, Python, Go, Rust, C#, Java, Kotlin, Ruby) and production remote MCP servers using Streamable HTTP (2025-03-26 spec), legacy HTTP+SSE fallbacks, OAuth 2.1 with PKCE authentication, Cloudflare Workers (@cloudflare/agents), and containerized FastMCP on Google Cloud Run. Use this skill whenever designing, building, debugging, testing, or deploying custom MCP servers.
license: MIT
compatibility: Node.js 20+, Python 3.10+, Go 1.24+, Rust 1.70+, .NET 8+, Java 17+, Ruby 2.7+
metadata:
  version: "2.0.0"
  spec_version: "2025-03-26"
---

# MCP Server Builder: The Master Implementation Guide

This skill provides complete architectural specifications, code templates, and deployment recipes for building custom Model Context Protocol (MCP) servers.

---

## 1. Architectural Decision Matrix

Select the implementation pattern matching your deployment target and transport requirements:

```
                          ┌──────────────────────────┐
                          │   Deployment Target?     │
                          └─────────────┬────────────┘
                                        │
           ┌────────────────────────────┼────────────────────────────┐
           ▼                            ▼                            ▼
  ┌──────────────────┐        ┌──────────────────┐        ┌──────────────────┐
  │ Local Desktop    │        │ Serverless Edge  │        │ Remote Backend   │
  │ (Claude Desktop, │        │ (Cloudflare      │        │ (Node.js/Express,│
  │ Cursor, IDEs)    │        │  Workers/D1/KV)  │        │  GCP Cloud Run)  │
  └────────┬─────────┘        └────────┬─────────┘        └────────┬─────────┘
           │                           │                           │
           ▼                           ▼                           ▼
     STDIO Transport          createMcpHandler()          Streamable HTTP
  (Zero network config;     (Stateless HTTP endpoint;    (Single /mcp endpoint;
   JSON-RPC over stdin/       OAuthProvider binding;       Per-session McpServer;
   stderr logging only)       Wrangler deployment)         OAuth 2.1 + PKCE)
```

| Deployment Mode | Transport Protocol | Auth Model | Recommended Stack | Primary Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **Local Desktop** | STDIO (stdin/stdout) | Local OS / Env Vars | TS (`@modelcontextprotocol/server`), Python (`mcp`), Go, Rust, C#, Java, Kotlin, Ruby | Personal tools, local filesystem access, private developer workflows |
| **Edge Serverless** | Streamable HTTP | Cloudflare Access / OAuthProvider | Cloudflare Workers + `createMcpHandler` | Global low-latency APIs, database edge tooling (D1/KV/R2) |
| **Container / Cloud** | Streamable HTTP | GCP IAM / OAuth 2.1 Bearer | Python `FastMCP` on Google Cloud Run | Scalable internal team tools, data pipelines, compute-heavy tasks |
| **Enterprise SaaS** | Dual (Streamable HTTP + SSE) | OAuth 2.1 + PKCE / JWT | Node.js Express + `@modelcontextprotocol/sdk` | Public AI connectors, user-scoped SaaS integrations |

---

## 2. Core Protocol & Transport Specifications

### Transport Comparison

| Feature | Streamable HTTP (Modern: `2025-03-26`) | HTTP + SSE (Legacy: `2024-11-05`) | STDIO (Local) |
| :--- | :--- | :--- | :--- |
| **Endpoints** | Single `POST /mcp` (and `DELETE /mcp`) | Dual: `GET /mcp` (SSE) + `POST /messages` | Process I/O streams |
| **Session Tracking** | `Mcp-Session-Id` HTTP header | URL query parameter `?sessionId=...` | Process lifetime |
| **Response Type** | Streamed chunked or JSON | Server-Sent Events stream | Single JSON-RPC lines |
| **Serverless Fit** | Excellent (scales to zero) | Poor (requires persistent SSE open connection) | N/A (Desktop process) |

### Golden Rules of MCP Protocol Compliance
1. **STDIO Logging Isolation:** Never write application logs to `stdout`. Any call to `print()`, `console.log()`, `fmt.Println()`, or `System.out.println()` will corrupt the JSON-RPC message framing and crash the client connection. **Always write logs to `stderr`**.
2. **Session-Server 1:1 Mapping:** An `McpServer` instance in Node.js must be created **per-session**. Do not share a single global `McpServer` instance across multiple HTTP transports, as response routing binds to the most recently connected transport.
3. **Session Header Casing:** Always send `Mcp-Session-Id` in outgoing response headers. When parsing incoming headers in Node.js/Express, access it via lowercase `req.headers['mcp-session-id']`.
4. **Content Negotiation:** Support both `application/json` and `text/event-stream` responses based on the client's `Accept` header.

---

## 3. Remote Server Implementation: Node.js / Express

This production pattern supports **Streamable HTTP**, **Legacy HTTP+SSE fallback**, and full **OAuth 2.1 with PKCE**.

### File Structure
```
remote-mcp-server/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts              # Express router & MCP transport lifecycle
│   ├── auth-routes.ts        # OAuth 2.1 discovery, authorize, token endpoints
│   ├── auth-middleware.ts    # Bearer token validation & AuthInfo injection
│   └── tools.ts              # Tool registry definitions
└── public/
    └── authorize.html        # User login consent screen
```

### Complete Implementation

#### `src/tools.ts`
```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";

export function registerTools(server: McpServer) {
  server.registerTool(
    "query_database",
    {
      description: "Query user-scoped records with optional filtering",
      inputSchema: z.object({
        filter: z.string().optional().describe("Filter parameter for records"),
        limit: z.number().int().min(1).max(100).default(10).describe("Max records to fetch"),
      }),
    },
    async (params, extra) => {
      // Access injected authentication metadata
      const authInfo = (extra as any)?.authInfo;
      const userToken = authInfo?.token;

      if (authInfo?.scopes && !authInfo.scopes.includes("read:data")) {
        return {
          content: [{ type: "text", text: "Error 403: Insufficient scope (requires 'read:data')" }],
          isError: true,
        };
      }

      return {
        content: [
          {
            type: "text",
            text: JSON.stringify({
              status: "success",
              user: authInfo?.clientId ?? "anonymous",
              filterApplied: params.filter ?? "none",
              results: [{ id: 1, title: "Sample Record" }],
            }),
          },
        ],
      };
    }
  );
}
```

#### `src/auth-routes.ts`
```typescript
import { Router } from "express";
import crypto from "node:crypto";
import { v4 as uuidv4 } from "uuid";

export const authRouter = Router();

// In-memory credential store (replace with Redis/Database in clustered environments)
export const authCodes = new Map<string, {
  clientId: string;
  codeChallenge: string;
  codeChallengeMethod: string;
  redirectUri: string;
  scopes: string[];
  userId: string;
  expiresAt: number;
}>();

export const accessTokens = new Map<string, {
  clientId: string;
  userId: string;
  scopes: string[];
  expiresAt: number;
}>();

// 1. Protected Resource Metadata Discovery (RFC 9728)
authRouter.get("/.well-known/oauth-protected-resource", (req, res) => {
  const baseUrl = `${req.protocol}://${req.get("host")}`;
  res.json({
    resource: baseUrl,
    authorization_servers: [{ issuer: baseUrl, authorization_endpoint: `${baseUrl}/authorize` }],
    scopes_supported: ["read:data", "write:data"],
  });
});

// 2. Authorization Server Metadata Discovery (RFC 8414)
authRouter.get("/.well-known/oauth-authorization-server", (req, res) => {
  const baseUrl = `${req.protocol}://${req.get("host")}`;
  res.json({
    issuer: baseUrl,
    authorization_endpoint: `${baseUrl}/authorize`,
    token_endpoint: `${baseUrl}/token`,
    registration_endpoint: `${baseUrl}/register`,
    token_endpoint_auth_methods_supported: ["none"],
    response_types_supported: ["code"],
    response_modes_supported: ["query"],
    grant_types_supported: ["authorization_code", "refresh_token"],
    code_challenge_methods_supported: ["S256"],
    scopes_supported: ["read:data", "write:data"],
  });
});

// 3. Dynamic Client Registration (RFC 7591)
authRouter.post("/register", (req, res) => {
  const { client_name, redirect_uris = [] } = req.body;
  const client_id = uuidv4();
  res.status(201).json({
    client_id,
    client_name: client_name || "MCP Client",
    redirect_uris,
    token_endpoint_auth_method: "none",
  });
});

// 4. Token Exchange with PKCE Verification
authRouter.post("/token", (req, res) => {
  const { grant_type, code, code_verifier, client_id, redirect_uri } = req.body;

  if (grant_type !== "authorization_code") {
    return res.status(400).json({ error: "unsupported_grant_type" });
  }

  const storedAuth = authCodes.get(code);
  if (!storedAuth || Date.now() > storedAuth.expiresAt) {
    return res.status(400).json({ error: "invalid_grant", error_description: "Code expired or invalid" });
  }

  // PKCE S256 Challenge Verification
  if (storedAuth.codeChallenge) {
    if (!code_verifier) {
      return res.status(400).json({ error: "invalid_request", error_description: "Missing code_verifier" });
    }
    const hash = crypto.createHash("sha256").update(code_verifier).digest("base64url");
    if (hash !== storedAuth.codeChallenge) {
      return res.status(400).json({ error: "invalid_grant", error_description: "PKCE verification failed" });
    }
  }

  authCodes.delete(code);

  const token = `mcp_at_${uuidv4().replace(/-/g, "")}`;
  accessTokens.set(token, {
    clientId: client_id || storedAuth.clientId,
    userId: storedAuth.userId,
    scopes: storedAuth.scopes,
    expiresAt: Date.now() + 3600 * 1000 * 24 * 30, // 30 days
  });

  res.json({
    access_token: token,
    token_type: "Bearer",
    expires_in: 2592000,
    scope: storedAuth.scopes.join(" "),
  });
});
```

#### `src/index.ts`
```typescript
import express from "express";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import { SSEServerTransport } from "@modelcontextprotocol/sdk/server/sse.js";
import { randomUUID } from "node:crypto";
import { authRouter, accessTokens } from "./auth-routes.js";
import { registerTools } from "./tools.js";

const app = express();
const PORT = process.env.PORT || 3000;

app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// CORS headers mandatory for browser-based MCP clients
app.use((req, res, next) => {
  res.header("Access-Control-Allow-Origin", "*");
  res.header("Access-Control-Allow-Methods", "GET, POST, OPTIONS, DELETE");
  res.header("Access-Control-Allow-Headers", "Content-Type, Authorization, Mcp-Session-Id");
  res.header("Access-Control-Expose-Headers", "Mcp-Session-Id, WWW-Authenticate");
  if (req.method === "OPTIONS") return res.status(200).end();
  next();
});

app.use(authRouter);

// Session storage mapping
interface SessionEntry {
  server: McpServer;
  transport: StreamableHTTPServerTransport | SSEServerTransport;
  authInfo?: any;
}
const transports = new Map<string, SessionEntry>();

function createServerInstance(authInfo?: any): McpServer {
  const server = new McpServer({
    name: "Production-Remote-MCP",
    version: "2.0.0",
  });
  registerTools(server);
  return server;
}

// Token Verification Helper
function verifyAuth(req: express.Request, res: express.Response) {
  const authHeader = req.headers["authorization"] || "";
  const token = authHeader.replace(/^Bearer\s+/i, "").trim();
  const baseUrl = `${req.protocol}://${req.get("host")}`;

  if (!token) {
    res.setHeader("WWW-Authenticate", `Bearer realm="MCP Server", resource_metadata_uri="${baseUrl}/.well-known/oauth-protected-resource"`);
    res.status(401).json({
      jsonrpc: "2.0",
      error: { code: -32000, message: "Missing Bearer access token" },
      id: req.body?.id ?? null,
    });
    return null;
  }

  const tokenData = accessTokens.get(token);
  if (!tokenData || Date.now() > tokenData.expiresAt) {
    res.status(403).json({
      jsonrpc: "2.0",
      error: { code: -32001, message: "Invalid or expired access token" },
      id: req.body?.id ?? null,
    });
    return null;
  }

  return { token, clientId: tokenData.clientId, userId: tokenData.userId, scopes: tokenData.scopes };
}

// ==========================================
// 1. Modern Streamable HTTP Endpoint (/mcp)
// ==========================================
app.post("/mcp", async (req, res) => {
  const authInfo = verifyAuth(req, res);
  if (!authInfo) return;

  const sessionId = req.headers["mcp-session-id"] as string | undefined;
  const isInit = req.body?.method === "initialize";

  try {
    if (isInit) {
      const server = createServerInstance(authInfo);
      const transport = new StreamableHTTPServerTransport({
        sessionIdGenerator: () => randomUUID(),
        onsessioninitialized: (id) => {
          transports.set(id, { server, transport, authInfo });
        },
      });

      transport.onclose = () => {
        if (transport.sessionId) transports.delete(transport.sessionId);
      };

      await server.connect(transport);
      await transport.handleRequest(req, res, req.body);
      return;
    }

    if (!sessionId || !transports.has(sessionId)) {
      return res.status(sessionId ? 404 : 400).json({
        jsonrpc: "2.0",
        error: { code: sessionId ? -32004 : -32003, message: sessionId ? "Session not found or expired" : "Missing Mcp-Session-Id header" },
        id: req.body?.id ?? null,
      });
    }

    const session = transports.get(sessionId)!;
    await (session.transport as StreamableHTTPServerTransport).handleRequest(req, res, req.body);
  } catch (error: any) {
    if (!res.headersSent) {
      res.status(500).json({ jsonrpc: "2.0", error: { code: -32603, message: error.message }, id: req.body?.id ?? null });
    }
  }
});

// Session Termination
app.delete("/mcp", (req, res) => {
  const sessionId = req.headers["mcp-session-id"] as string;
  if (sessionId && transports.has(sessionId)) {
    transports.delete(sessionId);
    res.status(204).end();
  } else {
    res.status(404).json({ error: "Session not found" });
  }
});

// ==========================================
// 2. Legacy HTTP + SSE Fallback Endpoints
// ==========================================
app.get("/mcp", async (req, res) => {
  const authInfo = verifyAuth(req, res);
  if (!authInfo) return;

  const server = createServerInstance(authInfo);
  const transport = new SSEServerTransport("/messages", res);
  transports.set(transport.sessionId, { server, transport, authInfo });

  res.setHeader("Content-Type", "text/event-stream");
  res.setHeader("Cache-Control", "no-cache");
  res.setHeader("Connection", "keep-alive");
  res.setHeader("X-Accel-Buffering", "no");
  res.setHeader("Mcp-Session-Id", transport.sessionId);

  try {
    await server.connect(transport);
  } catch (err) {
    transports.delete(transport.sessionId);
    if (!res.headersSent) res.status(500).end();
  }
});

app.post("/messages", async (req, res) => {
  const sessionId = req.query.sessionId as string;
  const authInfo = verifyAuth(req, res);
  if (!authInfo) return;

  if (!sessionId || !transports.has(sessionId)) {
    return res.status(404).json({ error: "Session not found" });
  }

  const session = transports.get(sessionId)!;
  if (session.transport instanceof SSEServerTransport) {
    await session.transport.handlePostMessage(req, res, req.body);
  } else {
    res.status(400).json({ error: "Invalid transport for /messages" });
  }
});

app.listen(PORT, () => console.error(`MCP Server running on port ${PORT}`));
```

---

## 4. Serverless Remote MCP on Cloudflare Workers

Cloudflare Workers provides edge execution using the `@cloudflare/agents` SDK.

### Project Setup
```bash
npm create cloudflare@latest -- my-cf-mcp-server --template=cloudflare/ai/demos/remote-mcp-github-oauth
cd my-cf-mcp-server
```

### Worker Configuration (`wrangler.jsonc`)
```jsonc
{
  "name": "mcp-edge-server",
  "main": "src/index.ts",
  "compatibility_date": "2026-03-01",
  "compatibility_flags": ["nodejs_compat"],
  "kv_namespaces": [
    {
      "binding": "OAUTH_KV",
      "id": "<YOUR_KV_NAMESPACE_ID>"
    }
  ]
}
```

### Server Code (`src/index.ts`)
```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { createMcpHandler } from "@cloudflare/agents/mcp";
import { OAuthProvider } from "@cloudflare/agents/oauth";
import { z } from "zod";

function buildServer() {
  const server = new McpServer({
    name: "cloudflare-edge-mcp",
    version: "1.0.0",
  });

  server.registerTool(
    "calculate_math",
    {
      description: "Perform arithmetic calculations",
      inputSchema: z.object({
        operation: z.enum(["add", "subtract", "multiply", "divide"]),
        a: z.number(),
        b: z.number(),
      }),
    },
    async ({ operation, a, b }) => {
      let result = 0;
      switch (operation) {
        case "add": result = a + b; break;
        case "subtract": result = a - b; break;
        case "multiply": result = a * b; break;
        case "divide": result = b !== 0 ? a / b : NaN; break;
      }
      return {
        content: [{ type: "text", text: `Result: ${result}` }],
      };
    }
  );

  return server;
}

// Stateless handler for high-performance scale
const mcpHandler = createMcpHandler(buildServer);

export default new OAuthProvider({
  apiRoute: "/mcp",
  apiHandler: mcpHandler,
  authorizeEndpoint: "/authorize",
  tokenEndpoint: "/token",
  clientRegistrationEndpoint: "/register",
});
```

### Deployment Commands
```bash
# Set secrets
npx wrangler secret put GITHUB_CLIENT_ID
npx wrangler secret put GITHUB_CLIENT_SECRET
npx wrangler secret put COOKIE_ENCRYPTION_KEY # Generate with: openssl rand -hex 32

# Deploy
npx wrangler deploy
```

---

## 5. Python FastMCP on Google Cloud Run

For Python developers, `FastMCP` combined with Google Cloud Run provides auto-scaling and optional Google IAM service authorization.

### Project Setup
```bash
mkdir mcp-on-cloudrun && cd mcp-on-cloudrun
uv init --bare --python 3.11
uv add "fastmcp>=2.6.0" "httpx>=0.27.0"
```

### Application Code (`server.py`)
```python
import asyncio
import logging
import os
import sys
from fastmcp import FastMCP

# Ensure logging writes to STDERR
logging.basicConfig(
    format="[%(asctime)s] [%(levelname)s]: %(message)s",
    level=logging.INFO,
    stream=sys.stderr
)
logger = logging.getLogger("mcp_server")

mcp = FastMCP("Google Cloud Run Math Engine")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers together."""
    logger.info(f"Executing add({a}, {b})")
    return a + b

@mcp.tool()
def subtract(a: int, b: int) -> int:
    """Subtract b from a."""
    logger.info(f"Executing subtract({a}, {b})")
    return a - b

if __name__ == "__main__":
    port = int(os.getenv("PORT", 8080))
    logger.info(f"Starting Streamable HTTP MCP server on 0.0.0.0:{port}")
    
    asyncio.run(
        mcp.run_async(
            transport="streamable-http",
            host="0.0.0.0",
            port=port,
        )
    )
```

### Production Dockerfile (`Dockerfile`)
```dockerfile
FROM python:3.11-slim
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

WORKDIR /app
COPY pyproject.toml .
RUN uv sync --frozen --no-cache

COPY . .
ENV PYTHONUNBUFFERED=1
EXPOSE 8080

CMD ["uv", "run", "server.py"]
```

### Google Cloud Run Deployment Script
```bash
export PROJECT_ID=$(gcloud config get-value project)

# Deploy with IAM Authentication Required (--no-allow-unauthenticated)
gcloud run deploy mcp-server \
  --source . \
  --region us-central1 \
  --no-allow-unauthenticated \
  --port 8080

# Authenticate local client to remote Cloud Run instance
gcloud run services proxy mcp-server --region us-central1
# Proxied local endpoint: http://127.0.0.1:8080/mcp
```

---

## 6. Local STDIO Implementations (All Major Languages)

For local desktop tool integrations (Claude Desktop, Cursor, Zed, Goose), implement STDIO transport. Ensure **zero output reaches stdout** other than the official JSON-RPC payloads emitted by the SDK.

### TypeScript / Node.js
```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({ name: "ts-stdio-server", version: "1.0.0" });

server.registerTool(
  "fetch_data",
  {
    description: "Sample tool",
    inputSchema: z.object({ query: z.string() }),
  },
  async ({ query }) => ({
    content: [{ type: "text", text: `Queried: ${query}` }],
  })
);

async function run() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("TypeScript STDIO MCP Server active");
}
run().catch((e) => { console.error("Fatal:", e); process.exit(1); });
```

### Python (`mcp`)
```python
import logging
import sys
from mcp.server import MCPServer

logging.basicConfig(stream=sys.stderr, level=logging.INFO)
mcp = MCPServer("python-stdio-server")

@mcp.tool()
async def get_system_status() -> str:
    """Check status of system processes."""
    return "All systems operational"

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

### Go (`go-sdk/mcp`)
```go
package main

import (
	"context"
	"log"
	"os"

	"github.com/modelcontextprotocol/go-sdk/mcp"
)

type StatusInput struct{}

func getStatus(ctx context.Context, req *mcp.CallToolRequest, input StatusInput) (*mcp.CallToolResult, any, error) {
	return &mcp.CallToolResult{
		Content: []mcp.Content{&mcp.TextContent{Text: "System OK"}},
	}, nil, nil
}

func main() {
	logger := log.New(os.Stderr, "[MCP-GO] ", log.LstdFlags)
	server := mcp.NewServer(&mcp.Implementation{Name: "go-mcp", Version: "1.0.0"}, nil)

	mcp.AddTool(server, &mcp.Tool{
		Name:        "get_status",
		Description: "Check system status",
	}, getStatus)

	logger.Println("Starting Go STDIO server...")
	if err := server.Run(context.Background(), &mcp.StdioTransport{}); err != nil {
		logger.Fatal(err)
	}
}
```

### Rust (`rmcp`)
```rust
use anyhow::Result;
use rmcp::{
    handler::server::{router::tool::ToolRouter, tool::Parameters},
    model::*,
    schemars, tool, tool_handler, tool_router, ServerHandler, ServiceExt,
};

#[derive(serde::Deserialize, schemars::JsonSchema)]
pub struct QueryRequest {
    pub key: String,
}

pub struct CoreServer {
    tool_router: ToolRouter<CoreServer>,
}

#[tool_router]
impl CoreServer {
    fn new() -> Self {
        Self { tool_router: Self::tool_router() }
    }

    #[tool(description = "Lookup value by key")]
    async fn lookup(&self, Parameters(QueryRequest { key }): Parameters<QueryRequest>) -> String {
        format!("Value for key '{}': Valid", key)
    }
}

#[tool_handler]
impl ServerHandler for CoreServer {
    fn get_info(&self) -> ServerInfo {
        ServerInfo {
            capabilities: ServerCapabilities::builder().enable_tools().build(),
            ..Default::default()
        }
    }
}

#[tokio::main]
async fn main() -> Result<()> {
    eprintln!("Rust STDIO server starting");
    let transport = (tokio::io::stdin(), tokio::io::stdout());
    let service = CoreServer::new().serve(transport).await?;
    service.waiting().await?;
    Ok(())
}
```

### C# (.NET 8+)
```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using ModelContextProtocol.Server;
using System.ComponentModel;

var builder = Host.CreateEmptyApplicationBuilder(settings: null);
builder.Services.AddMcpServer()
    .WithStdioServerTransport()
    .WithToolsFromAssembly();

var app = builder.Build();
await app.RunAsync();

[McpServerToolType]
public static class Tools
{
    [McpServerTool, Description("Echo back input parameter.")]
    public static Task<string> Echo([Description("Text to echo")] string input)
    {
        return Task.FromResult($"Echo: {input}");
    }
}
```

### Java (Spring Boot / Spring AI)
```java
@Service
public class EchoService {
    @Tool(description = "Echoes the incoming message")
    public String echoMessage(@ToolParam(description = "Message string") String msg) {
        return "Received: " + msg;
    }
}

@SpringBootApplication
public class McpApplication {
    public static void main(String[] args) {
        SpringApplication.run(McpApplication.class, args);
    }

    @Bean
    public ToolCallbackProvider tools(EchoService service) {
        return MethodToolCallbackProvider.builder().toolObjects(service).build();
    }
}
```

### Kotlin
```kotlin
fun runMcpServer() {
    val server = Server(
        Implementation(name = "kotlin-mcp", version = "1.0.0"),
        ServerOptions(capabilities = ServerCapabilities(tools = ServerCapabilities.Tools(listChanged = true)))
    )

    server.addTool(
        name = "ping",
        description = "Returns pong",
        inputSchema = ToolSchema(properties = buildJsonObject {}, required = emptyList())
    ) { _ ->
        CallToolResult(content = listOf(TextContent("pong")))
    }

    val transport = StdioServerTransport(System.`in`.asInput(), System.out.asSink().buffered())
    runBlocking {
        val session = server.createSession(transport)
        val job = Job()
        session.onClose { job.complete() }
        job.join()
    }
}
```

### Ruby
```ruby
require "mcp"

class PingTool < MCP::Tool
  tool_name "ping"
  description "Health check tool"
  input_schema(properties: {}, required: [])

  def self.call
    MCP::Tool::Response.new([{ type: "text", text: "pong" }])
  end
end

server = MCP::Server.new(name: "ruby-mcp", version: "1.0.0", tools: [PingTool])
transport = MCP::Server::Transports::StdioTransport.new(server)
transport.open
```

---

## 7. Client Configuration & Verification

### Claude Desktop Configuration (`claude_desktop_config.json`)

#### Path Locations:
- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Linux:** `~/.config/Claude/claude_desktop_config.json`
- **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

#### Local & Remote Bridged Entries:
```json
{
  "mcpServers": {
    "local-python": {
      "command": "uv",
      "args": ["--directory", "/ABSOLUTE/PATH/TO/project", "run", "server.py"]
    },
    "local-node": {
      "command": "node",
      "args": ["/ABSOLUTE/PATH/TO/project/build/index.js"]
    },
    "remote-bridge-authenticated": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://my-mcp-server.example.com/mcp",
        "--header",
        "Authorization: Bearer ${AUTH_TOKEN}",
        "--transport",
        "http-only"
      ],
      "env": {
        "AUTH_TOKEN": "my-secret-token"
      }
    }
  }
}
```

### Interactive Verification via MCP Inspector
```bash
# Launch interactive web client tester
npx @modelcontextprotocol/inspector@latest
```
1. Set Transport to **Streamable HTTP** (or SSE).
2. Enter URL: `http://localhost:3000/mcp`.
3. Test OAuth flow under **OAuth Settings** -> **Quick OAuth Flow**.
4. Select **Connect** -> **List Tools** -> Verify tool invocations.

---

## 8. Critical Pitfalls & Diagnostic Guide

| Failure Mode | Root Cause | Immediate Fix |
| :--- | :--- | :--- |
| **Silent JSON Parsing Failure on Desktop** | `print()` or `console.log()` polluted `stdout`. | Switch all logging across dependencies to write strictly to `stderr`. |
| **`Session not found (404)` on 2nd Request** | Server didn't echo `Mcp-Session-Id` header or regenerated ID. | Set `res.setHeader('Mcp-Session-Id', id)` on every response. |
| **OAuth Registration Fails Silently** | Client expects RFC 7591 dynamic client registration endpoint. | Return 201 from `/register` with `client_id` even if storage is dummy. |
| **Serverless Memory Leaks** | Transport map accumulating unbounded dead session instances. | Attach `.onclose` callbacks to remove session keys and set a TTL cleaner. |
| **Cloud Run Idle Scaling Lock** | Using legacy SSE transport keeps connections alive indefinitely. | Switch transport to `streamable-http` to allow container scale-to-zero. |
| **CORS Failure in Web Clients** | Missing `Mcp-Session-Id` in `Access-Control-Expose-Headers`. | Add `res.header('Access-Control-Expose-Headers', 'Mcp-Session-Id, WWW-Authenticate')`. |
