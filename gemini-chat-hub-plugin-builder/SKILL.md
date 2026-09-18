---
name: gemini-chat-hub-plugin-builder
description: Comprehensive operational guide for designing, engineering, testing, and deploying custom plugins for Gemini Chat Hub. Covers OpenAI Function Calling schemas, Node.js server VM sandboxing (zero-CORS), persistent local data storage (storage/localStorage), code execution workspace bridging (workspace/runtimeWorkspace into Python IPython kernel, Bash, and Bun), Secret Vault security, HTTP Actions with workspace auto-save, mid-turn multimodal returns (images, audio, video) via the Gemini Interactions API, and custom output rendering.
license: MIT
compatibility: Node.js VM Server Sandbox, Gemini Interactions API (v1beta), Gemini Chat Hub
metadata:
  version: "2.1.0"
  author: "Gemini Chat Hub Engineering"
---

# Gemini Chat Hub Custom Plugin Development Guide

This operational manual outlines the technical requirements, design patterns, runtime architecture, and deployment procedures for engineering high-performance Custom Plugins within the **Gemini Chat Hub** ecosystem.

---

## 1. Core Architectural Overview

Gemini Chat Hub plugins extend the capabilities of Google Gemini models (including Gemini 3.8 Flash, Gemini 3.5 Flash-Lite, and Gemini 3.1 Pro) using **OpenAI Function Calling API Specifications** through the native **Gemini Interactions API (`v1beta/interactions`)**.

When a plugin is enabled, Gemini Chat Hub registers its tool declarations with the model. The model autonomously invokes the plugin tools during multi-turn interactions, receiving structured execution output and synthesizing responses in real time.

Crucially, plugins in Gemini Chat Hub can return **mid-turn multimodal media** (images, audio, and video) directly to Gemini inside the function-calling execution loop. This allows Gemini to dynamically retrieve, inspect, and analyze visual and auditory content before formulating its final response.

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
│                  └────────────┬───────────┘            │
│                               ▼                        │
│             ┌────────────────────────────────────┐     │
│             │   Mid-Turn Multimodal Pipeline     │     │
│             │   • Images (PNG, JPEG, WebP, SVG)  │     │
│             │   • Audio (MP3, WAV, OGG, WebM)    │     │
│             │   • Video (MP4, WebM, MOV)         │     │
│             └─────────────────┬──────────────────┘     │
│                               ▼                        │
│             Gemini function_result subcontent parts    │
│             (Gemini inspects visuals/audio mid-turn)   │
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
| **Mid-Turn Multimodal** | Text only; cannot feed media back to model | **Native Images, Audio, Video mid-turn returned to Gemini** |
| **Model Integration** | Chat completions API | **Gemini Native Interactions API (`v1beta/interactions`)** |

---

## 2. Interface Configuration: User Settings & Vault Integration

User Settings define configuration inputs required for plugin execution (such as API keys, custom endpoints, or user preferences).

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
| `get` / `getItem` | `await storage.get<T>(key: string): Promise<T | null>` | Retrieves a stored item by key. Returns parsed object/value. |
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
| `saveData` | `await workspace.saveData(name, data, format?): Promise<WorkspaceFileResult>` | Formats array/object as JSON or CSV and writes to `/workspace/in/<name>.[json|csv]`. |
| `listFiles` | `await workspace.listFiles(subDir?): Promise<string[]>` | Lists file names within the designated sub-directory. |
| `getPath` | `workspace.getPath(fileName?, subDir?): string` | Returns the absolute on-disk filesystem path. |

---

## 6. Mid-Turn Multimodal Plugin Outputs (Images, Audio, Video)

Gemini Chat Hub supports **Mid-Turn Multimodal Outputs**, enabling plugins to feed visual and auditory media directly back to Gemini inside the **Gemini Interactions API (`v1beta/interactions`) function-calling turn**.

### The Interaction Cycle
1. **Model Function Call:** The user asks a question (e.g. *"Generate a quarterly revenue chart and tell me which month had the steepest growth"*).
2. **Plugin Execution:** The plugin generates an SVG/PNG chart, records an audio clip, or clips a video segment.
3. **Multimodal Return:** The plugin returns image, audio, or video data (via base64 or workspace file paths).
4. **Mid-Turn Model Perception:** Gemini Chat Hub feeds the media into the `function_result.result` subcontent array as native Gemini parts (`{ type: "image", data, mime_type }`, `{ type: "audio", data, mime_type }`, etc.).
5. **Direct Visual/Audio Inspection:** Gemini "sees" the image or "hears" the audio mid-turn and continues reasoning without requiring a separate turn or manual user upload.
6. **Rich UI Rendering:** The chat interface displays an interactive image gallery, an audio player, or a video player directly inside the tool execution card!

```
┌────────────────────────────────────────────────────────┐
│ User: "Plot revenue trends and tell me the top month"  │
└──────────────────────────┬─────────────────────────────┘
                           ▼
              Gemini emits Tool Call:
          generate_sales_chart({ year: 2026 })
                           ▼
          Plugin Sandbox executes & renders PNG
                           ▼
             Plugin returns Multimodal Image
                           ▼
     Gemini Interactions API function_result subcontent:
       [
         { type: "text", text: "Chart rendered successfully." },
         { type: "image", data: "iVBORw0KGgo...", mime_type: "image/png" }
       ]
                           ▼
          Gemini inspects chart visual features:
  "Looking at the plotted chart, March had the steepest..."
```

### Manifest Declaration for Multimodal Tools
When authoring a plugin function that outputs media, declare `returnsMultimodal` and `multimodalTypes`:
```json
{
  "name": "generate_sales_chart",
  "implementationType": "javascript",
  "outputType": "respond_to_ai",
  "returnsMultimodal": true,
  "multimodalTypes": ["image"],
  "permissions": ["multimodal", "workspace"]
}
```

### The `multimodal` Sandbox Helper API
The Node.js sandbox provides a global `multimodal` helper (also accessible via `resources.multimodal`) with convenient factory methods:

| Method | Signature | Description |
| :--- | :--- | :--- |
| `multimodal.image` | `multimodal.image(dataOrUrl, mimeType?, options?)` | Constructs an image part. Supports base64 data, `data:image/...` URLs, and remote HTTP URLs. |
| `multimodal.audio` | `multimodal.audio(dataOrUrl, mimeType?, options?)` | Constructs an audio part. Supports MP3, WAV, OGG, WebM. |
| `multimodal.video` | `multimodal.video(dataOrUrl, mimeType?, options?)` | Constructs a video part. Supports MP4, WebM, MOV. |
| `multimodal.document` | `multimodal.document(dataOrUrl, mimeType?, options?)` | Constructs a PDF/document part. |
| `multimodal.fromWorkspace` | `await multimodal.fromWorkspace(filePath, options?)` | Reads a file from `/workspace/in/`, `/workspace/out/`, or `/workspace/tmp/`, detects its MIME type, and returns an image, audio, or video part. |
| `multimodal.response` | `multimodal.response(summaryText, ...parts)` | Packages multiple multimodal parts with a concise human-readable summary text. |

### Supported Return Formats
The plugin router normalizes any of the following return signatures:

1. **Composite Return Object:**
```javascript
return {
  text: "Generated 1 chart and 1 audio readout.",
  images: ["data:image/png;base64,iVBORw0KGgo..."],
  audio: ["data:audio/mp3;base64,//uQx..."]
};
```

2. **Multimodal Helper Response:**
```javascript
return multimodal.response(
  "Rendered 3D protein structure view.",
  multimodal.image(pngBase64, "image/png"),
  multimodal.audio(audioBase64, "audio/mp3")
);
```

3. **Workspace File Reference:**
```javascript
return {
  filePath: "out/distribution_plot.png",
  text: "Data distribution plotted and ready for visual inspection."
};
```

4. **Gemini Subcontent Parts Array:**
```javascript
return [
  { type: "text", text: "Analysis results below:" },
  { type: "image", data: base64Data, mime_type: "image/png" }
];
```

---

### Multimodal Code Examples

#### Example 1: Dynamic SVG Chart Generator (Images)
Renders a vector chart in pure JavaScript without external dependencies, converts it to base64, and returns it for Gemini to analyze:

```javascript
async function generate_status_chart(params, userSettings, resources) {
  const { labels, values, title = "System Health Metrics" } = params;
  
  const width = 600;
  const height = 300;
  const maxVal = Math.max(...values, 1);
  const barWidth = Math.floor((width - 80) / values.length) - 10;
  
  // Build SVG bars
  let svgBars = "";
  values.forEach((v, i) => {
    const barHeight = Math.floor((v / maxVal) * (height - 100));
    const x = 50 + i * (barWidth + 10);
    const y = height - 40 - barHeight;
    svgBars += `<rect x="${x}" y="${y}" width="${barWidth}" height="${barHeight}" fill="#3b82f6" rx="4"/>`;
    svgBars += `<text x="${x + barWidth / 2}" y="${y - 8}" text-anchor="middle" font-size="12" fill="#333">${v}</text>`;
    svgBars += `<text x="${x + barWidth / 2}" y="${height - 20}" text-anchor="middle" font-size="12" fill="#666">${labels[i] || i}</text>`;
  });

  const svg = `<svg xmlns="http://www.w3.org/2000/svg" width="${width}" height="${height}" style="background:#ffffff;font-family:sans-serif;">
    <text x="${width / 2}" y="30" text-anchor="middle" font-size="16" font-weight="bold" fill="#111">${title}</text>
    <line x1="40" y1="${height - 38}" x2="${width - 20}" y2="${height - 38}" stroke="#ccc" stroke-width="1"/>
    ${svgBars}
  </svg>`;

  // Convert SVG string to base64
  const base64Svg = Buffer.from(svg).toString("base64");
  
  // Also save a copy to workspace/out so the user can download it
  await workspace.writeFile("status_chart.svg", svg, "out");

  // Return mid-turn multimodal image to Gemini
  return multimodal.response(
    `Rendered ${title} with ${values.length} data points.`,
    multimodal.image(base64Svg, "image/svg+xml")
  );
}
```

#### Example 2: Voice Pronunciation & Audio Synthesis (Audio)
Fetches pronunciation audio from a remote API and returns it directly to Gemini:

```javascript
async function synthesize_pronunciation(params, userSettings, resources) {
  const { text, language = "en-US" } = params;
  
  // Direct server-side fetch (zero CORS issues!)
  const url = `https://translate.google.com/translate_tts?ie=UTF-8&q=${encodeURIComponent(text)}&tl=${language}&client=tw-ob`;
  const res = await fetch(url, {
    headers: { "User-Agent": "Mozilla/5.0" }
  });
  
  if (!res.ok) throw new Error(`Audio synthesis failed: ${res.status}`);
  
  const arrayBuffer = await res.arrayBuffer();
  const base64Audio = Buffer.from(arrayBuffer).toString("base64");

  return {
    text: `Synthesized pronunciation for "${text}" in ${language}.`,
    audio: [`data:audio/mp3;base64,${base64Audio}`]
  };
}
```

#### Example 3: Security Camera Video Clipper (Video)
Retrieves a video recording clip from a network camera or video API:

```javascript
async function fetch_camera_clip(params, userSettings, resources) {
  const { cameraName, durationSeconds = 5 } = params;
  
  const cameraUrl = userSettings.camera_feed_url || "https://example.com/api/cam/clip";
  const res = await fetch(`${cameraUrl}?name=${cameraName}&duration=${durationSeconds}`, {
    headers: { "Authorization": `Bearer ${vault.getSecret("CAMERA_API_KEY")}` }
  });

  if (!res.ok) throw new Error(`Camera stream failed: ${res.status}`);
  const buffer = Buffer.from(await res.arrayBuffer());

  // Save to workspace/tmp/
  await workspace.writeFile(`${cameraName}_clip.mp4`, buffer, "tmp");

  // Feed to Gemini mid-turn
  return multimodal.response(
    `Retrieved ${durationSeconds}s clip from camera '${cameraName}'. Inspect for motion or objects.`,
    multimodal.video(buffer.toString("base64"), "video/mp4")
  );
}
```

#### Example 4: Python Workspace Matplotlib Pipeline
Stages a dataset, executes a Python script via the IPython kernel to generate a visualization in `/workspace/out/`, and returns it to Gemini:

```javascript
async function plot_and_inspect_metrics(params, userSettings, resources) {
  const { datasetName, rows } = params;

  // 1. Stage raw data into /workspace/in/
  await workspace.saveData(datasetName, rows, "json");

  // 2. Instruct model to run Python or load the generated file
  // Once Python creates /workspace/out/heatmap.png:
  const part = await multimodal.fromWorkspace("out/heatmap.png");

  return multimodal.response(
    "Data plotted via Matplotlib and staged at out/heatmap.png.",
    part
  );
}
```

---

## 7. Secret Vault Access (`vault` / `secrets`)

Gemini Chat Hub features a centralized Secret Vault (`secretVault`) that discovers credentials from environment variables and encrypted database settings (`vault_secrets`).

### Key Features
1. **Zero Secret Leakage:** All stdout, stderr, logs, and output streams are monitored by an active redaction scrubber that replaces secret values with `[REDACTED_SECRET:<NAME>]`.
2. **Programmatic Access:** Plugins can query secrets by name without requiring users to manually copy/paste keys.

### JavaScript API Methods
Available globally as `vault`, `secrets`, and via `resources.vault`:

| Method | Signature | Description |
| :--- | :--- | :--- |
| `getSecret` | `vault.getSecret(name: string): string | undefined` | Retrieves the raw secret value (case-insensitive). |
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

## 8. Functional Implementation Specs

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

## 9. Implementation Methods

### Method A: JavaScript Server Sandbox Execution
Runs in an isolated Node.js VM context (`node:vm`).

#### Standard Function Signature
```javascript
async function function_name(params, userSettings, resources) {
  // params: Arguments passed by the model
  // userSettings: Configured plugin settings
  // resources: System context { userMessage, chatId, pluginId, storage, workspace, vault, multimodal }
}
```

#### Globals Available in the Sandbox
- **Web & Network**: `fetch`, `Headers`, `Request`, `Response`, `URL`, `URLSearchParams`, `AbortController`, `AbortSignal`, `TextEncoder`, `TextDecoder`, `Blob`, `File`, `FormData`, `Buffer`.
- **Timers**: `setTimeout`, `clearTimeout`, `setInterval`, `clearInterval`, `setImmediate`.
- **Builtins**: `Array`, `Object`, `String`, `Number`, `Boolean`, `RegExp`, `Map`, `Set`, `WeakMap`, `WeakSet`, `Promise`, `Symbol`, `Error`, `Math`, `Date`, `JSON`.
- **Helpers**: `parseInt`, `parseFloat`, `encodeURIComponent`, `decodeURIComponent`, `btoa`, `atob`.
- **Console**: `console.log`, `console.warn`, `console.error` (auto-prefixed with plugin function name in server logs).
- **Gemini Chat Hub Extensions**: `storage` (`localStorage`), `workspace` (`runtimeWorkspace`), `vault` (`secrets`), `multimodal`, `params`, `userSettings`, `resources`.

---

### Method B: HTTP Actions (Declarative Integrations)
Executes HTTP calls without custom JavaScript code.

#### Dynamic Variable & Vault Interpolation
Template placeholders are replaced across URL, Headers, and Body:
- `{param}` or `{{param}}`: Injects function argument or user setting.
- `{{VAULT:SECRET_NAME}}` or `{{vault.SECRET_NAME}}`: Injects secret from Secret Vault.
- `{CHAT_ID}`: Injects current conversation ID.

#### Code Execution Workspace Auto-Save & Binary Media Interception
HTTP actions can automatically pipe API response payloads directly into `/workspace/in/`.
When an endpoint returns binary media (`image/*`, `audio/*`, `video/*`, `application/pdf`), the HTTP runner automatically:
1. Detects the media format.
2. Converts the binary stream into a Base64-encoded multimodal part for mid-turn Gemini inspection.
3. If `save_to_workspace: true` is enabled, saves the raw binary buffer into `/workspace/in/<filename>`.

```json
{
  "http_action": {
    "url": "https://api.chartgenerator.com/v1/render?symbol={symbol}&format=png",
    "method": "GET",
    "save_to_workspace": true,
    "workspace_filename": "{symbol}_chart.png",
    "has_post_processing": false
  }
}
```

---

## 10. Advanced Output Destinations (`outputType`)

| Destination (`outputType`) | Description | Best For |
| :--- | :--- | :--- |
| `"respond_to_ai"` (Default) | Returns output (text and multimodal media) to Gemini in the function-calling loop. Gemini synthesizes a conversational answer. | Data queries, searches, calculation tools, multimodal charts/audio. |
| `"render_markdown"` | Bypasses model text synthesis; renders output directly as GitHub-Flavored Markdown in the chat. | Large tables, pre-formatted documentation, plugin generator JSON. |
| `"display_html_to_user"` | Renders output as an interactive HTML/JS canvas widget. | Custom dashboards, SVG diagrams, calculators, interactive forms. |
| `"card"` | Renders structured card items (e.g. image gallery schema). | Image search results, media cards. |

---

## 11. Canonical Gemini Chat Hub Plugin Manifest

Below is a complete, production-ready manifest exhibiting local storage, workspace bridging, Secret Vault, and **mid-turn multimodal returns**:

```json
{
  "id": "visual_financial_analyst",
  "uuid": "7a32b904-4c8d-4f12-9856-cb8211993214",
  "name": "Visual Financial Analyst",
  "title": "Visual Financial Analyst",
  "emoji": "📊",
  "description": "Pulls financial metrics, generates interactive visual charts, caches ticker data, and returns multimodal charts mid-turn directly to Gemini.",
  "version": "2.1.0",
  "author": "Financial Engineering Group",
  "iconURL": "https://api.iconify.design/lucide:candlestick-chart.svg",
  "contextPrompt": "When analyzing tickers with this plugin, always cite price figures with 2 decimal places and explain chart patterns.",
  "localStorage": {
    "enabled": true,
    "initialData": { "tracked_tickers": ["AAPL", "GOOGL", "NVDA"] }
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
      "required": false,
      "vaultSecret": "FINANCIAL_API_KEY",
      "description": "Enter your market data API key or link FINANCIAL_API_KEY from Vault."
    }
  ],
  "pluginFunctions": [
    {
      "id": "fn_render_stock_trend_chart",
      "name": "Render Stock Trend Chart & Return to Gemini",
      "implementationType": "javascript",
      "outputType": "respond_to_ai",
      "returnsMultimodal": true,
      "multimodalTypes": ["image"],
      "permissions": ["local_storage", "workspace", "vault", "multimodal"],
      "openaiSpec": {
        "name": "render_stock_trend_chart",
        "description": "Generates a high-resolution trend chart for a stock symbol, saves SVG to workspace, and returns a multimodal image part mid-turn directly to Gemini for visual analysis.",
        "parameters": {
          "type": "object",
          "properties": {
            "symbol": {
              "type": "string",
              "description": "Stock ticker symbol (e.g. 'AAPL', 'NVDA')."
            },
            "points": {
              "type": "array",
              "items": { "type": "number" },
              "description": "Array of recent closing prices to plot."
            }
          },
          "required": ["symbol", "points"]
        }
      },
      "code": "async function render_stock_trend_chart(params, userSettings, resources) {\n  const { symbol, points = [] } = params;\n  const max = Math.max(...points, 1);\n  const min = Math.min(...points, 0);\n  const range = max - min || 1;\n\n  const width = 500;\n  const height = 200;\n  const step = points.length > 1 ? (width - 60) / (points.length - 1) : width;\n\n  const polylinePoints = points.map((pt, i) => {\n    const x = 30 + i * step;\n    const y = height - 30 - ((pt - min) / range) * (height - 60);\n    return `${x},${y}`;\n  }).join(' ');\n\n  const svg = `<svg xmlns=\"http://www.w3.org/2000/svg\" width=\"${width}\" height=\"${height}\" style=\"background:#1e1e2e;color:#cdd6f4;font-family:sans-serif;\">\n    <text x=\"20\" y=\"25\" fill=\"#cdd6f4\" font-weight=\"bold\">${symbol} Price Trend</text>\n    <polyline fill=\"none\" stroke=\"#a6e3a1\" stroke-width=\"3\" points=\"${polylinePoints}\"/>\n  </svg>`;\n\n  // 1. Stage in workspace/out/ for user download\n  await workspace.writeFile(`${symbol.toLowerCase()}_trend.svg`, svg, 'out');\n\n  // 2. Return mid-turn multimodal part directly to Gemini\n  const base64Svg = Buffer.from(svg).toString('base64');\n  return multimodal.response(\n    `Rendered ${symbol} price trend with ${points.length} points.`,\n    multimodal.image(base64Svg, 'image/svg+xml')\n  );\n}"
    }
  ],
  "overviewMarkdown": "# Visual Financial Analyst\n\nPulls live equity market data, renders trend charts, and returns multimodal visuals mid-turn directly to Gemini.",
  "authenticationType": "AUTH_TYPE_NONE"
}
```

---

## 12. Verification & Debugging Checklist

Before exporting or importing your plugin into Gemini Chat Hub:
1. **Unique Function Name:** Is the JavaScript function name unique across your tools?
2. **Server-Side Compatibility:** Ensure code does not rely on browser DOM objects (`window`, `document`, `localStorage.getItem` synchronous). Use asynchronous `await storage.get()` instead.
3. **Multimodal Declaration:** If your tool returns visual or auditory media, did you set `returnsMultimodal: true` and include `"multimodal"` in permissions?
4. **Multimodal Output Format:** Are you returning a valid format (e.g. `multimodal.response(...)`, `multimodal.image(...)`, or `{ text, images: [...] }`)?
5. **Workspace File Locations:** Write inputs to `"in"` (`/workspace/in/`) and user-downloadable deliverables to `"out"` (`/workspace/out/`).
6. **Secret Vault Safety:** Never print or log raw API keys. Let the redaction engine and `vault.getSecret()` handle credentials safely.
7. **No CORS Workarounds Needed:** Do not wrap URLs in third-party CORS proxies; server-side `fetch` communicates directly with any public or private API.
8. **Error Messages:** Throw informative descriptive errors with `throw new Error(...)` to allow Gemini to self-heal and inform the user.
