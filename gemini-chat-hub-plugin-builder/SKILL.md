---
name: gemini-chat-hub-plugin-builder
description: Comprehensive operational guide for designing, engineering, testing, and deploying custom plugins for Gemini Chat Hub. Covers OpenAI Function Calling schemas, Node.js server VM sandboxing (zero-CORS), persistent local data storage (storage/localStorage), code execution workspace bridging (workspace/runtimeWorkspace into Python IPython kernel, Bash, and Bun), Secret Vault security, HTTP Actions with workspace auto-save, and custom output rendering.
license: MIT
compatibility: Node.js VM Server Sandbox, Gemini Interactions API (v1beta), Gemini Chat Hub
metadata:
  version: "2.0.0"
  author: "Gemini Chat Hub Engineering"
---

# Gemini Chat Hub Custom Plugin Development Guide

This operational manual outlines the technical requirements, design patterns, runtime architecture, and deployment procedures for engineering high-performance Custom Plugins within the **Gemini Chat Hub** ecosystem.

---

## 1. Core Architectural Overview

Gemini Chat Hub plugins extend the capabilities of Google Gemini models (including Gemini 3.8 Flash, Gemini 3.5 Flash-Lite, and Gemini 3.1 Pro) using **OpenAI Function Calling API Specifications** through the native **Gemini Interactions API (`v1beta/interactions`)**.

When a plugin is enabled, Gemini Chat Hub registers its tool declarations with the model. The model autonomously invokes the plugin tools during multi-turn interactions, receiving structured execution output and synthesizing responses in real time.

```
┌────────────────────────────────────────────────────────┐
│                   Gemini Chat Hub                      │
│             (Next.js Server Runtime)                   │
├────────────────────────────────────────────────────────┤
│  Interactions API Multi-Turn Function Calling Loop     │
│                         │                              │
│         ┌───────────────┴───────────────┐              │
│         ▼                               ▼              │
│ ┌───────────────────────┐   ┌────────────────────────┐ │
│ │  JavaScript Sandbox   │   │      HTTP Action       │ │
│ │   (Node.js VM Core)   │   │  (Templated Fetcher)   │ │
│ └───────────┬───────────┘   └───────────┬────────────┘ │
│             │                           │              │
│    ┌────────┼─────────────────┐         │              │
│    ▼        ▼                 ▼         ▼              │
│ [Storage] [Vault]        [Workspace Bridge]            │
│ (Postgres (Secrets       (/workspace/in, out, tmp)     │
│  + Disk)   Redaction)         │                        │
│                               ▼                        │
│                  ┌────────────────────────┐            │
│                  │ Stateful Code Runtimes │            │
│                  │ • IPython Kernel       │            │
│                  │ • Persistent Bash      │            │
│                  │ • Polyglot Bun (TS/JS) │            │
│                  └────────────────────────┘            │
└────────────────────────────────────────────────────────┘
```

### Key Differences from Browser-Based Plugins (e.g. TypingMind)
| Capability | Legacy Browser Plugins (TypingMind) | Gemini Chat Hub Plugins |
| :--- | :--- | :--- |
| **Execution Environment** | Client-side browser `<iframe>` (`allow-scripts`) | **Server-side Node.js VM (`node:vm`)** |
| **Network & CORS** | Restricted by browser CORS; requires proxy workers | **Direct server fetch (Zero CORS restrictions)** |
| **Execution Timeout** | Typically 10–15 seconds | **90 seconds** (multi-step pipelines supported) |
| **Data Persistence** | Volatile or browser localStorage | **Dual-persistence: PostgreSQL DB + disk mirror** |
| **Code Execution Bridge** | None | **Direct bridge to Python, Bash, and Bun runtimes** |
| **Output Artifacts** | Manual copy/paste | **Files in `/workspace/out/` auto-index as UI artifacts** |
| **Secret Management** | Exposed plain-text settings | **Integrated Secret Vault with active output scrubbing** |
| **Model Integration** | Chat completions API | **Gemini Native Interactions API (`v1beta/interactions`)** |

---

## 2. Interface Configuration: User Settings & Vault Integration

User Settings define the configuration inputs required for plugin execution (such as API keys, custom endpoints, or user preferences).

### JSON Field Descriptor Schema
Each setting object supports the following configuration properties:

| Property | Type | Description | Required |
| :--- | :--- | :--- | :--- |
| `name` | String | Key identifier used to retrieve the variable programmatically (camelCase or snake_case). | Yes |
| `label` | String | User-facing input label displayed in the configuration UI. | Yes |
| `type` | String | Input widget type: `text`, `password`, `email`, `number`, or `enum`. (Defaults to `text`). | No |
| `required` | Boolean | Declares if the field must be populated to save the plugin. | No |
| `description` | String | Explanatory subtext or format constraints. | No |
| `placeholder` | String | Placeholder helper text inside empty input fields. | No |
| `values` / `enum` | Array[Str] | Valid options array. **Required** when type is `enum`. | Conditional |
| `defaultValue` | String | Fallback value if the field is left empty. | No |
| `vaultSecret` | String | Optional default Secret Vault key to suggest for autofill (e.g. `"GITHUB_PAT"`). | No |

### Automatic API Key Fallbacks & Vault Linking
1. **Server Fallbacks:** If `google_api_key` or `gemini_api_key` is declared, Gemini Chat Hub automatically falls back to `process.env.GEMINI_API_KEY` if the user leaves the field empty.
2. **Vault Secret Dynamic Linking:** Users can click **Autofill from Vault** in the UI to link a secret as `{{VAULT:SECRET_NAME}}` or `{{vault.SECRET_NAME}}`. The server resolves this securely at execution time.

### Config Blueprint: User Settings Array
```json
[
  {
    "name": "exa_api_key",
    "label": "Exa Search API Key",
    "type": "password",
    "required": true,
    "vaultSecret": "EXA_API_KEY",
    "description": "Enter your Exa API key or link it from the Secret Vault."
  },
  {
    "name": "result_limit",
    "label": "Result Limit",
    "type": "number",
    "defaultValue": "10",
    "description": "Maximum number of search results to retrieve."
  },
  {
    "name": "search_category",
    "label": "Search Category",
    "type": "enum",
    "values": ["general", "research paper", "news", "github"],
    "defaultValue": "general"
  }
]
```

---

## 3. Persistent Prompting: Plugin Context (`contextPrompt`)

Plugin Context allows injection of instructions, schema definitions, or domain indices directly into Gemini's **system instruction**. This context is injected whenever the plugin is active, even before any tool is triggered.

### Template Interpolation
Plugin context supports dynamic variable interpolation using `{settingName}`:
```json
{
  "contextPrompt": "You have access to the internal company knowledge base at endpoint {api_base_url}. When searching for financial records, always query using ISO 8601 UTC dates."
}
```

In Gemini Chat Hub, all active plugin context prompts are automatically compiled into the system instructions under `### [Plugin Context: <Plugin Name>]`.

---

## 4. Persistent Local Data Storage (`storage` / `localStorage`)

Gemini Chat Hub provides a built-in persistent key-value storage engine for custom plugins.

### Storage Architecture
- **Database Persistence:** Every plugin has an isolated record in PostgreSQL (`Setting` table under key `plugin_storage:<pluginId>`).
- **Disk Mirroring:** Mirrored to disk at `/workspace/plugin_data/<pluginId>.json` (or `<workspaceRoot>/plugin_data/<pluginId>.json`) for inspection and cross-process access.
- **In-Memory Cache:** Fast synchronous reads during execution turns with background database and disk sync.

### Manifest Declaration
Enable storage in the plugin manifest:
```json
{
  "localStorage": {
    "enabled": true,
    "initialData": {
      "query_history": [],
      "favorites": []
    }
  }
}
```

### JavaScript API Methods
Storage is available globally in your script as `storage` and `localStorage`, and via `resources.storage`:

| Method | Signature | Description |
| :--- | :--- | :--- |
| `get` / `getItem` | `await storage.get<T>(key: string): Promise<T \| null>` | Retrieves a stored item by key. Returns parsed object/value. |
| `set` / `setItem` | `await storage.set(key: string, value: any): Promise<void>` | Saves an item. Value can be string, number, object, or array. |
| `delete` / `removeItem` | `await storage.delete(key: string): Promise<void>` | Deletes a stored key. |
| `clear` | `await storage.clear(): Promise<void>` | Clears all data for this plugin. |
| `list` / `getAll` | `await storage.list(): Promise<Record<string, any>>` | Returns all key-value entries as an object. |

### Code Example: Caching API Results in Local Storage
```javascript
async function fetch_crypto_prices(params, userSettings, resources) {
  const { coin } = params;
  const cacheKey = `price_${coin.toLowerCase()}`;
  
  // 1. Check persistent local storage
  const cached = await storage.get(cacheKey);
  if (cached && (Date.now() - cached.timestamp < 60000)) {
    return JSON.stringify({ ...cached.data, source: "local_cache" });
  }

  // 2. Fetch fresh data (no CORS restrictions on server)
  const res = await fetch(`https://api.coingecko.com/api/v3/simple/price?ids=${coin}&vs_currencies=usd`);
  const data = await res.json();

  // 3. Persist to local storage
  await storage.set(cacheKey, { data, timestamp: Date.now() });

  return JSON.stringify({ ...data, source: "live_network" });
}
```

---

## 5. Code Execution Workspace Bridge (`workspace` / `runtimeWorkspace`)

Gemini Chat Hub contains an integrated, stateful code execution runtime:
- **Resident IPython Kernel (`execute_python`)**: Persists variables, DataFrames, functions, and imports in RAM across sequential conversational turns.
- **Persistent Bash Shell (`execute_bash`)**: Stateful shell for system commands, FFmpeg, Git, and package management.
- **Polyglot Bun Runner (`execute_bun`)**: Sub-millisecond JavaScript and TypeScript execution.

Plugins can write directly to the workspace environment, allowing seamless multi-step pipelines where a plugin ingests or generates data, and the model subsequently runs Python or Bash to analyze or transform it!

### Workspace Directory Structure
- `/workspace/in/`: Inbound data storage. Files saved here are immediately accessible in Python via `open('/workspace/in/...')` or `pd.read_csv('/workspace/in/...')`, and in Bash.
- `/workspace/out/`: Designated output directory. **ANY file written here is automatically indexed as an artifact and triggers a download card directly in the user interface!**
- `/workspace/tmp/`: Transient scratchpad for intermediate processing.

### Manifest Declaration
Enable workspace access in the plugin manifest:
```json
{
  "workspaceAccess": {
    "enabled": true,
    "allowedDirectories": ["in", "out", "tmp"]
  }
}
```

### JavaScript API Methods
Available globally as `workspace`, `runtimeWorkspace`, and via `resources.workspace`:

| Method | Signature | Description |
| :--- | :--- | :--- |
| `writeFile` | `await workspace.writeFile(fileName, content, subDir?): Promise<WorkspaceFileResult>` | Writes file (string, Buffer, or Uint8Array) into `"in"`, `"out"`, or `"tmp"`. |
| `readFile` | `await workspace.readFile(fileName, subDir?): Promise<string>` | Reads a workspace file as a UTF-8 string. |
| `saveData` | `await workspace.saveData(name, data, format?): Promise<WorkspaceFileResult>` | Automatically formats array/object as JSON or CSV and writes to `/workspace/in/<name>.[json\|csv]`. |
| `listFiles` | `await workspace.listFiles(subDir?): Promise<string[]>` | Lists file names within the designated sub-directory. |
| `getPath` | `workspace.getPath(fileName?, subDir?): string` | Returns the absolute on-disk filesystem path. |

### `WorkspaceFileResult` Object
```typescript
interface WorkspaceFileResult {
  fileName: string;        // e.g. "sales_report.csv"
  relativePath: string;    // e.g. "in/sales_report.csv"
  absolutePath: string;    // Absolute disk path
  sizeBytes: number;       // File size in bytes
  mimeType: string;        // Detected MIME type
  subDir: "in"|"out"|"tmp";
  agentNote: string;       // Context note for the LLM explaining file path and usage
}
```

### Workflow 1: Saving Ingested Data for Python Analysis
```javascript
async function fetch_and_stage_dataset(params, userSettings, resources) {
  const { endpoint, datasetName } = params;
  
  const res = await fetch(endpoint);
  const rows = await res.json();

  // Automatically convert array of objects to CSV and stage in /workspace/in/
  const fileResult = await workspace.saveData(datasetName, rows, "csv");

  return JSON.stringify({
    success: true,
    rowsCount: rows.length,
    workspaceFile: fileResult.relativePath,
    agentInstruction: `Dataset saved to ${fileResult.relativePath}. You can now run 'execute_python' using 'import pandas as pd; df = pd.read_csv("/workspace/${fileResult.relativePath}")' to perform statistical analysis and visualizations.`
  });
}
```

### Workflow 2: Generating Downloadable User Artifacts (`/workspace/out/`)
```javascript
async function generate_report_artifact(params, userSettings, resources) {
  const { reportTitle, markdownContent } = params;
  const fileName = `${reportTitle.toLowerCase().replace(/[^a-z0-9]/g, '_')}.md`;

  // Writing to "out" automatically triggers a download card in the UI!
  const artifact = await workspace.writeFile(fileName, markdownContent, "out");

  return JSON.stringify({
    message: "Report successfully generated and staged for download.",
    artifactName: artifact.fileName,
    sizeBytes: artifact.sizeBytes,
    downloadAvailable: true
  });
}
```

---

## 6. Secret Vault Access (`vault` / `secrets`)

Gemini Chat Hub features a centralized Secret Vault (`secretVault`) that discovers credentials from environment variables and encrypted database settings (`vault_secrets`).

### Key Features
1. **Zero Secret Leakage:** All stdout, stderr, logs, and output streams are monitored by an active redaction scrubber that replaces secret values with `[REDACTED_SECRET:<NAME>]`.
2. **Programmatic Access:** Plugins can query secrets by name without requiring users to manually copy/paste keys.

### JavaScript API Methods
Available globally as `vault`, `secrets`, and via `resources.vault`:

| Method | Signature | Description |
| :--- | :--- | :--- |
| `getSecret` | `vault.getSecret(name: string): string \| undefined` | Retrieves the raw secret value (case-insensitive). |
| `listSecrets` | `vault.listSecrets(): string[]` | Returns a list of all registered secret names (names only, no values). |
| `hasSecret` | `vault.hasSecret(name: string): boolean` | Returns true if the secret exists. |

### Code Example: Secure Vault Usage
```javascript
async function query_github_repo(params, userSettings, resources) {
  const { repoPath } = params;
  
  // 1. Retrieve secret from Vault or user settings fallback
  const token = vault.getSecret("GITHUB_PAT") || userSettings.github_token;
  if (!token) {
    throw new Error("Missing GitHub Token. Please register GITHUB_PAT in Workspace Settings or plugin settings.");
  }

  // 2. Perform authenticated request
  const res = await fetch(`https://api.github.com/repos/${repoPath}`, {
    headers: {
      "Authorization": `Bearer ${token}`,
      "User-Agent": "GeminiChatHubPlugin/1.0"
    }
  });

  if (!res.ok) {
    throw new Error(`GitHub API error: ${res.status} ${res.statusText}`);
  }

  const data = await res.json();
  return JSON.stringify({
    stars: data.stargazers_count,
    forks: data.forks_count,
    open_issues: data.open_issues_count
  });
}
```

---

## 7. Functional Implementation Specs

Every plugin function starts with an OpenAI Function Spec declaring its input parameters.

```json
{
  "name": "query_database_records",
  "description": "Searches internal database records using SQL filters and pagination.",
  "parameters": {
    "type": "object",
    "properties": {
      "table_name": {
        "type": "string",
        "description": "The table to query, e.g. 'users', 'invoices', 'metrics'."
      },
      "limit": {
        "type": "number",
        "description": "Maximum number of records to return (1-100)."
      }
    },
    "required": ["table_name"]
  }
}
```

---

## 8. Implementation Methods

### Method A: JavaScript Server Sandbox Execution
Runs in an isolated Node.js VM context (`node:vm`).

#### Standard Function Signature
```javascript
async function function_name(params, userSettings, resources) {
  // params: Arguments passed by the model
  // userSettings: Configured plugin settings
  // resources: System context { userMessage, chatId, pluginId, storage, workspace, vault }
}
```
*Note:* The runner also supports `main(params, userSettings, resources)` or `run(params, userSettings, resources)`.

#### Globals Available in the Sandbox
- **Web & Network**: `fetch`, `Headers`, `Request`, `Response`, `URL`, `URLSearchParams`, `AbortController`, `AbortSignal`, `TextEncoder`, `TextDecoder`, `Blob`, `File`, `FormData`, `Buffer`.
- **Timers**: `setTimeout`, `clearTimeout`, `setInterval`, `clearInterval`, `setImmediate`.
- **Builtins**: `Array`, `Object`, `String`, `Number`, `Boolean`, `RegExp`, `Map`, `Set`, `WeakMap`, `WeakSet`, `Promise`, `Symbol`, `Error`, `Math`, `Date`, `JSON`.
- **Helpers**: `parseInt`, `parseFloat`, `encodeURIComponent`, `decodeURIComponent`, `btoa`, `atob`.
- **Console**: `console.log`, `console.warn`, `console.error` (auto-prefixed with plugin function name in server logs).
- **Gemini Chat Hub Extensions**: `storage` (`localStorage`), `workspace` (`runtimeWorkspace`), `vault` (`secrets`), `params`, `userSettings`, `resources`.

---

### Method B: HTTP Actions (Declarative Integrations)
Executes HTTP calls without custom JavaScript code.

#### Dynamic Variable & Vault Interpolation
Template placeholders are replaced across URL, Headers, and Body:
- `{param}` or `{{param}}`: Injects function argument or user setting.
- `{{VAULT:SECRET_NAME}}` or `{{vault.SECRET_NAME}}`: Injects secret from Secret Vault.
- `{CHAT_ID}`: Injects current conversation ID.

#### Code Execution Workspace Auto-Save
HTTP actions can automatically pipe API response payloads directly into `/workspace/in/`:
```json
{
  "http_action": {
    "url": "https://api.coingecko.com/api/v3/coins/markets?vs_currency=usd&order=market_cap_desc&per_page={count}",
    "method": "GET",
    "save_to_workspace": true,
    "workspace_filename": "crypto_market_data_{Date.now()}.json",
    "has_post_processing": true,
    "post_processing_type": "handlebars",
    "post_processing_code": "Staged {{length}} cryptocurrency market records to workspace."
  }
}
```

#### Post-Processing Engines
- **JMESPath**: Filter large JSON responses down to specific keys: `data.items[*].{id: id, name: title}`.
- **Handlebars**: Reformat into concise Markdown or summaries using `{{__VARIABLES.param}}`.

---

## 9. Advanced Output Destinations (`outputType`)

| Destination (`outputType`) | Description | Best For |
| :--- | :--- | :--- |
| `"respond_to_ai"` (Default) | Returns raw output to Gemini in the function-calling loop. Gemini synthesizes a conversational answer. | Data queries, searches, calculation tools. |
| `"render_markdown"` | Bypasses model text synthesis; renders output directly as GitHub-Flavored Markdown in the chat. | Large tables, pre-formatted documentation, instant responses. |
| `"display_html_to_user"` | Renders output as an interactive HTML/JS canvas widget. | Custom dashboards, SVG diagrams, calculators, interactive forms. |
| `"card"` | Renders structured card items (e.g. image gallery schema). | Image search results, media cards. |

---

## 10. Canonical Gemini Chat Hub Plugin Manifest

```json
{
  "id": "financial_analytics_suite",
  "uuid": "7a32b904-4c8d-4f12-9856-cb8211993214",
  "name": "Financial Analytics Suite",
  "title": "Financial Analytics Suite",
  "emoji": "📈",
  "description": "Pulls real-time equity metrics, caches tick history in local storage, and stages CSV datasets for Python analysis.",
  "version": "2.0.0",
  "author": "Financial Engineering Group",
  "iconURL": "https://api.iconify.design/lucide:trending-up.svg",
  "contextPrompt": "When analyzing tickers with this plugin, always format currency outputs to 2 decimal places and cite timestamps in UTC.",
  "localStorage": {
    "enabled": true,
    "initialData": { "tracked_tickers": ["AAPL", "GOOGL", "MSFT"] }
  },
  "workspaceAccess": {
    "enabled": true,
    "allowedDirectories": ["in", "out", "tmp"]
  },
  "vaultSecrets": ["FINANCIAL_API_KEY"],
  "userSettings": [
    {
      "name": "apiKey",
      "label": "Market API Key",
      "type": "password",
      "required": true,
      "vaultSecret": "FINANCIAL_API_KEY",
      "description": "Enter your market data API key or link FINANCIAL_API_KEY from Vault."
    }
  ],
  "pluginFunctions": [
    {
      "id": "fn_fetch_ticker_history",
      "name": "Fetch Ticker History & Stage for Python",
      "implementationType": "javascript",
      "outputType": "respond_to_ai",
      "permissions": ["local_storage", "workspace", "vault"],
      "openaiSpec": {
        "name": "fetch_ticker_history",
        "description": "Fetches historical equity prices for a ticker symbol, caches them locally, and stages a CSV in /workspace/in/ for Python analysis.",
        "parameters": {
          "type": "object",
          "properties": {
            "symbol": {
              "type": "string",
              "description": "Stock ticker symbol (e.g. 'AAPL', 'NVDA')."
            },
            "days": {
              "type": "number",
              "description": "Number of historical days to pull (1-90)."
            }
          },
          "required": ["symbol"]
        }
      },
      "code": "async function fetch_ticker_history(params, userSettings, resources) {\n  const { symbol, days = 30 } = params;\n  const key = userSettings.apiKey || vault.getSecret('FINANCIAL_API_KEY');\n  \n  // 1. Check local storage cache\n  const cacheKey = `history_${symbol.toUpperCase()}`;\n  let records = await storage.get(cacheKey);\n  \n  if (!records) {\n    const url = `https://api.example.com/v1/history?symbol=${symbol}&days=${days}&key=${key}`;\n    const res = await fetch(url);\n    if (!res.ok) throw new Error(`Market API error: ${res.status}`);\n    records = await res.json();\n    await storage.set(cacheKey, records);\n  }\n  \n  // 2. Stage directly into code execution workspace as CSV\n  const fileName = `${symbol.toLowerCase()}_history.csv`;\n  const wsResult = await workspace.saveData(fileName, records, 'csv');\n  \n  return JSON.stringify({\n    symbol: symbol.toUpperCase(),\n    recordCount: records.length,\n    workspaceFile: wsResult.relativePath,\n    agentNote: `Historical data staged at ${wsResult.relativePath}. Call 'execute_python' using pandas to compute rolling volatilities, moving averages, or generate plots.`\n  });\n}"
    }
  ],
  "overviewMarkdown": "# Financial Analytics Suite\n\nPulls live equity market data, caches lookups, and automatically stages datasets into the code execution environment.",
  "authenticationType": "AUTH_TYPE_NONE"
}
```

---

## 11. Verification & Debugging Checklist

Before exporting or importing your plugin into Gemini Chat Hub:
1. **Unique Function Name:** Is the JavaScript function name unique across your tools?
2. **Server-Side Compatibility:** Ensure code does not rely on browser DOM objects (`window`, `document`, `localStorage.getItem` synchronous). Use asynchronous `await storage.get()` instead.
3. **Workspace File Locations:** Write inputs to `"in"` (`/workspace/in/`) and user-downloadable deliverables to `"out"` (`/workspace/out/`).
4. **Secret Vault Safety:** Never print or log raw API keys. Let the redaction engine and `vault.getSecret()` handle credentials safely.
5. **No CORS Workarounds Needed:** Do not wrap URLs in third-party CORS proxies; server-side `fetch` communicates directly with any public or private API.
6. **Error Messages:** Throw informative descriptive errors with `throw new Error(...)` to allow Gemini to self-heal and inform the user.
