<p>
  <img src="../nibchat_logo_full.svg" alt="" width="288" align="middle" />
</p>

**Turn any MCP Server into a ready-to-go AI Agent — in minutes.**

Nibchat is a lightweight, self-hosted chat interface that connects directly to any [Model Context Protocol (MCP)](https://modelcontextprotocol.io) server and exposes it as a fully functional AI agent. Drop in your `agent.yaml`, point it at your MCP server, and your team has a branded, conversational interface to interact with your tools — no extra infrastructure required.

---

## Why Nibchat?

Most MCP servers expose powerful tools — databases, APIs, file systems, internal services — but have no user-facing interface. Nibchat bridges that gap:

- **Any MCP Server → Conversational Agent.** Connect Nibchat to your MCP server and it automatically discovers all available tools, making them accessible through natural language.
- **Zero friction for end users.** No login, no accounts. Each user gets an anonymous session automatically — just open the browser and start chatting.
- **Brand it as your own.** Configure the agent name, description, avatar, system prompt, and starter messages through a single YAML file at deploy time.
- **Self-hosted and private.** Runs entirely in your infrastructure. Your conversations, your data.

---

## See it in Action

Here are some examples of Nibchat in action:
- [Math Tutor](https://mathtutor.nibchat.ai/) [(agent config file)](./examples/math-tutor/agent.yaml)
- [GitHub Roaster](https://githubroaster.nibchat.ai/) [(agent config file)](./examples/github-roaster/agent.yaml)
- [Croupier - The Blackjack Dealer](https://croupier.nibchat.ai) [(agent config file)](./examples/croupier/agent.yaml) [(mcp source)](https://github.com/nibchat-ai/blackjack)



## Features

- **PostgreSQL support** — swap SQLite for PostgreSQL by setting a single env var (`POSTGRESQL_DSN`); includes native vector search via `pgvector` for the knowledge base, with an automatic in-memory cosine-similarity fallback if the extension is not installed
- **Flexible file storage** — uploaded files can be stored in the database (default, zero config), on the local filesystem (`FILE_STORAGE_PATH`), or in any S3-compatible bucket (`S3_BUCKET`) — AWS S3, Cloudflare R2, MinIO, Tigris
- **Optional user authentication** — enable email/password login with `AUTH_ENABLED=true`; supports open registration or invite-only mode (`INVITE_ONLY=true`); conversations are scoped to the logged-in user and accessible from any device
- **Knowledge base (RAG)** — drop `.txt`, `.md`, or `.pdf` files into a `knowledge/` folder; Nibchat indexes them at startup using OpenAI embeddings and gives the agent a built-in `search_knowledge` tool for semantic retrieval — no extra infrastructure, everything stored in the same SQLite database
- **MCP-UI / MCP Apps rendering** — tool results that return interactive HTML UIs are rendered inline in the chat as sandboxed iframes; iframes can call tools back interactively via a built-in postMessage relay; UIs persist across conversation reloads
- **MCP tool calling** — full agentic loop with real-time tool call visibility in the chat
- **Multiple MCP servers** — connect to several MCP servers simultaneously; tools are auto-namespaced to avoid collisions
- **Streaming responses** — tokens streamed live via Server-Sent Events
- **File & image input** — attach images (vision) and PDFs; files persist across the conversation and can be re-sent in follow-up messages
- **Agent configuration** — customize name, avatar, description, system prompt, and starter messages via `agent.yaml`
- **Any OpenAI-compatible provider** — point Nibchat at OpenRouter, Ollama, LM Studio, or any other OpenAI-compatible endpoint via `OPENAI_BASE_URL`; fully transparent to users
- **Flexible API key setup** — each user can configure their own key via the UI, or an admin can lock the provider, model, and key via environment variables
- **Dark / light mode** — toggle in the sidebar, preference persisted in the browser
- **Conversation history** — persistent per-session history stored in SQLite
- **Collapsible sidebar** — clean ChatGPT-style UI; on mobile the sidebar becomes a full-screen drawer opened via a hamburger button
- **Built-in agent memory** — the agent proactively remembers facts about each user across conversations; memories are stored per session and auto-injected into the system prompt
- **Built-in web search & page fetch** — optional `web_search` tool powered by Tavily and always-on `fetch_page` tool powered by Jina Reader; both visible inline in the chat
- **Admin dashboard** — protected admin area at `/admin` with session stats, conversation browser, and admin account management
- **Markdown & syntax highlighting** — full Markdown rendering in assistant responses
- **Docker-ready** — single container, single `docker compose up`

---

## Quick Start

The fastest way to run Nibchat is with the pre-built Docker image.

**1. Create a `docker-compose.yml`:**

```yaml
services:
  nibchat:
    container_name: nibchat
    image: ghcr.io/nibchat-ai/nibchat:latest
    ports:
      - "3000:3000"
    environment:
      - PORT=3000
      - DB_PATH=/app/data/nibchat.db
      - MCP_SERVER_URL=${MCP_SERVER_URL:-http://host.docker.internal:8080}
      - SESSION_SECRET=change-me-to-a-random-string
      # Uncomment and set if using an agent config file:
      #- AGENT_CONFIG_PATH=/app/agent.yaml
    volumes:
      - nibchat-data:/app/data
      # Mount your agent.yaml to enable agent configuration:
      #- ./agent.yaml:/app/agent.yaml
    restart: unless-stopped
    extra_hosts:
      - "host.docker.internal:host-gateway"

volumes:
  nibchat-data:
```

**2. Start it:**

```bash
docker compose up -d
```

**3. Open your browser at [http://localhost:3000](http://localhost:3000)**

On first use, open **Settings** in the sidebar to enter your OpenAI API key and choose a model — unless a shared key is configured via environment variables (see below).

---

## Connecting Your MCP Server

### Single server

Set the `MCP_SERVER_URL` environment variable to point to your MCP server's HTTP/Streamable endpoint:

```bash
MCP_SERVER_URL=http://my-mcp-server:8080 docker compose up -d
```

### Multiple servers

Set `MCP_SERVER_URLS` to a comma-separated list of endpoints:

```bash
MCP_SERVER_URLS=http://mcp-server-1:8080,http://mcp-server-2:8081 docker compose up -d
```

Or declare them in `agent.yaml` (recommended — lets you name each server):

```yaml
agent:
  mcpServers:
    - url: "http://mcp-server-1:8080"
      name: "warehouse"
    - url: "http://mcp-server-2:8081"
      name: "crm"
```

The `name` field is used as a prefix when two servers expose a tool with the same name (e.g. `warehouse__search` vs `crm__search`). If omitted, the server hostname is used as the prefix.

Nibchat connects to all configured servers on demand (per user session), discovers their tools, and makes the combined tool set available to the language model. Tool calls and their results are shown inline in the chat as they happen.

> **Running MCP locally?** The `extra_hosts: host.docker.internal:host-gateway` entry in the compose file lets the container reach services on your host machine, so `http://host.docker.internal:8080` works out of the box.

### MCP server authentication

**OAuth 2.0** — Servers that require OAuth (e.g. Atlassian, Asana) are handled automatically. When a server requires authorization, an **Authorize** button appears in the sidebar. Clicking it opens a popup where the user completes the OAuth flow. Tokens are stored per-user session in the database and reused on subsequent visits.

Set `PUBLIC_URL` to your deployment's public-facing base URL so the OAuth redirect URI resolves correctly:

```yaml
environment:
  - PUBLIC_URL=https://nibchat.example.com
```

Register `https://nibchat.example.com/oauth/callback` as the allowed redirect URI in your OAuth provider's developer console.

**Static headers** — Servers that use a fixed API key or Bearer token:

```yaml
agent:
  mcpServers:
    - url: "http://my-server:8080"
      name: "myserver"
      headers:
        Authorization: "Bearer my-api-key"
```

Any number of headers can be specified. Header-based auth and OAuth are mutually exclusive — setting `headers` skips the OAuth flow for that server.

---

## MCP-UI / MCP Apps Support

Nibchat supports the [MCP-UI / MCP Apps](https://mcpui.dev) standard. When an MCP tool returns an interactive HTML UI, Nibchat renders it inline in the chat as a sandboxed iframe — directly below the tool call block, as a natural part of the conversation flow.

Two patterns are supported:

**Pattern A — HTML embedded in the tool result**

The tool result content array contains a resource block with `mimeType: text/html;profile=mcp-app`:

```json
{
  "type": "resource",
  "resource": {
    "uri": "ui://my-server/widget",
    "mimeType": "text/html;profile=mcp-app",
    "text": "<html>...</html>"
  }
}
```

**Pattern B — MCP Apps (`_meta.ui.resourceUri`)**

The tool definition declares a UI resource URI in its metadata. After the tool runs, Nibchat fetches the HTML from the MCP server via `resources/read` and renders it:

```json
{
  "name": "show_dashboard",
  "_meta": {
    "ui": { "resourceUri": "ui://my-server/dashboard" }
  }
}
```

Rendered UIs are persisted alongside the conversation so they reappear correctly when revisiting a past chat.

### Interactive tool calling from within the UI

Embedded UIs can call MCP tools back interactively via a built-in `postMessage` relay. Both wire formats are supported:

**Legacy MCP-UI** (`{ type: "tool", ... }`):
```javascript
// Inside the iframe
window.parent.postMessage({
  type: 'tool',
  messageId: 'abc-123',
  payload: { toolName: 'search', params: { query: 'hello' } }
}, '*')

// Host acknowledges, executes the tool, then replies:
// { type: 'ui-message-response', messageId: 'abc-123', payload: { response: { result: '...' } } }
```

**MCP Apps JSON-RPC** (`{ jsonrpc: "2.0", method: "tools/call" }`):
```javascript
// Inside the iframe
window.parent.postMessage({
  jsonrpc: '2.0',
  id: 1,
  method: 'tools/call',
  params: { name: 'search', arguments: { query: 'hello' } }
}, '*')

// Host replies:
// { jsonrpc: '2.0', id: 1, result: { content: [{ type: 'text', text: '...' }] } }
```

Link opening (`ui/open-link` / `type: "link"`) and iframe resize notifications (`ui/notifications/size-changed` / `type: "ui-size-change"`) are also handled.

---

## Agent Configuration

Transform Nibchat into a branded agent by providing an `agent.yaml` file at deploy time. This is how you go from a generic chat UI to a purpose-built assistant for your team.

**`agent.yaml` example:**

```yaml
agent:
  name: "Data Assistant"
  description: "Ask me anything about our data warehouse. I can run queries, generate reports, and explain results."

  # System prompt — shapes the agent's personality and behavior
  instructions: |
    You are a helpful data analyst assistant. You have access to tools that can query
    the company data warehouse. Always explain your queries before running them,
    and present results in a clear, readable format.

  # Shown on the welcome screen as quick-start prompts
  starterMessages:
    - "Show me sales for last month"
    - "Which products have the lowest inventory?"
    - "Summarize today's support tickets"
    - "Generate a weekly revenue report"

  # Single MCP server (takes precedence over the env var):
  mcpServer:
    url: "http://my-mcp-server:8080"

  # — OR — multiple MCP servers with optional auth:
  # mcpServers:
  #   - url: "http://my-mcp-server-1:8080"
  #     name: "warehouse"             # used as tool prefix on collision
  #   - url: "http://my-mcp-server-2:8081"
  #     name: "crm"
  #     headers:
  #       Authorization: "Bearer my-api-key"   # static header auth
  #   - url: "https://mcp.atlassian.com/v1/mcp"
  #     name: "jira"                  # OAuth — users get an Authorize button

  # Optional: knowledge base directory for RAG (default: ./knowledge)
  # knowledgeDir: "./knowledge"

  # Optional: agent avatar (base64-encoded PNG)
  # icon: "data:image/png;base64,..."
  # iconDark: "data:image/png;base64,..."
```

**Mount it in your compose file:**

```yaml
volumes:
  - nibchat-data:/app/data
  - ./agent.yaml:/app/agent.yaml
environment:
  - AGENT_CONFIG_PATH=/app/agent.yaml
```

---

## Shared OpenAI Credentials

By default each user enters their own OpenAI API key via the Settings UI. If you want to provide a shared key for all users (e.g. a team deployment), set these environment variables:

```yaml
environment:
  - OPENAI_API_KEY=sk-...
  - OPENAI_MODEL=gpt-4o        # optional, defaults to gpt-4o
```

When `OPENAI_API_KEY` is set, the settings modal is disabled for all users — they can open it to see which model is active, but cannot change anything.

---

## Custom Provider / Local Models

Nibchat works with any OpenAI-compatible API endpoint — not just OpenAI. Set `OPENAI_BASE_URL` to point it at a different provider:

```yaml
environment:
  # OpenRouter — access hundreds of models with one API key
  - OPENAI_BASE_URL=https://openrouter.ai/api/v1
  - OPENAI_MODEL=openai/gpt-4o
  - OPENAI_API_KEY=sk-or-...

  # Ollama — local models, no API key needed
  - OPENAI_BASE_URL=http://host.docker.internal:11434/v1
  - OPENAI_MODEL=llama3.2
  - OPENAI_API_KEY=ollama          # dummy value required by the SDK

  # LM Studio — local models
  - OPENAI_BASE_URL=http://host.docker.internal:1234/v1
  - OPENAI_MODEL=lmstudio-community/Meta-Llama-3-8B-Instruct-GGUF
  - OPENAI_API_KEY=lm-studio       # dummy value required by the SDK
```

### Locking behaviour

| `OPENAI_BASE_URL` | `OPENAI_MODEL` | Result |
|---|---|---|
| set | set | Settings UI fully locked — provider, model, and key all from env |
| set | **not set** | Provider endpoint fixed, but users configure their own key + model |
| not set | — | Standard behaviour (see [Shared OpenAI Credentials](#shared-openai-credentials)) |

When locked, the Settings modal shows the provider endpoint and model as read-only fields. When the provider is set but the model is not, users see a read-only "Provider endpoint" field alongside the editable key and model inputs.

---

## Knowledge Base (RAG)

Give your agent long-term knowledge by dropping documents into a `knowledge/` folder. Nibchat indexes them at startup using OpenAI embeddings, stores the vectors in SQLite, and exposes a hidden `search_knowledge` tool the model uses automatically when answering domain-specific questions.

### How it works

1. On startup Nibchat scans the `knowledge/` directory for `.txt`, `.md`, and `.pdf` files
2. Each file is parsed, split into overlapping chunks (~600 tokens each), and embedded using `text-embedding-3-small`
3. Embeddings are stored in SQLite alongside the text chunks
4. File hashes are tracked — unchanged files are skipped on subsequent restarts
5. When the knowledge base is non-empty, a `search_knowledge` tool is silently injected into every conversation and the system prompt is automatically extended with a RAG directive

The `search_knowledge` tool never appears in the sidebar — it is a built-in capability invisible to end users.

### Setup

**1. Set `OPENAI_API_KEY`** (required — indexing is skipped entirely if absent):

```yaml
environment:
  - OPENAI_API_KEY=sk-...
```

**2. Create a `knowledge/` folder** next to your `docker-compose.yml` and drop your files in:

```
knowledge/
  product-manual.pdf
  faq.md
  internal-policies.txt
```

**3. Mount it into the container:**

```yaml
volumes:
  - nibchat-data:/app/data
  - ./knowledge:/app/knowledge
```

**4. Start (or restart) the container.** Watch the logs — you'll see indexing progress:

```
Knowledge indexing: found 3 file(s) in ./knowledge
  Indexing product-manual.pdf: 42 chunk(s)
  Indexing faq.md: 8 chunk(s)
  Indexing internal-policies.txt: 5 chunk(s)
Knowledge indexing complete: 55 chunk(s) in memory.
```

On the next restart, only files whose content has changed will be re-indexed.

### Custom knowledge directory

The knowledge directory defaults to `./knowledge` relative to the server working directory. Override it in `agent.yaml`:

```yaml
agent:
  name: "My Agent"
  knowledgeDir: "./my-docs"
```

### Supported file types

| Extension | Notes |
|---|---|
| `.txt` | Plain text |
| `.md` | Markdown (content is indexed as-is) |
| `.pdf` | Text extracted via `pdf-parse`; scanned/image PDFs are not supported |

Subdirectories are scanned recursively. Files of other types are silently ignored.

---

## Built-in Agent Memory

Nibchat gives every agent a persistent, per-session memory. The agent proactively saves facts about the user (preferences, habits, context) and they are automatically injected into the system prompt at the start of every conversation — so the agent always has context without being asked.

Three tools are silently available in every conversation:

| Tool | Description |
|---|---|
| `save_memory` | Saves a free-form note about the user. Called proactively by the agent. |
| `delete_memory` | Removes a memory by ID. Called when the user asks to forget something. |
| `list_memories` | Lists all saved memories with their IDs. |

Memory tool calls are invisible in the chat UI — the agent communicates naturally in text. Users can ask _"What do you remember about me?"_ to trigger `list_memories`, and _"Forget that I prefer X"_ to trigger `delete_memory`.

Memories are stored per session in SQLite and persist indefinitely until deleted.

---

## Web Search & Page Fetch

Two optional built-in tools give the agent access to the web.

### `web_search`

Searches the web using the [Tavily](https://tavily.com) API. Returns an AI-generated answer summary plus the top results. Enabled when `TAVILY_API_KEY` is set.

```yaml
environment:
  - TAVILY_API_KEY=tvly-...
```

### `fetch_page`

Fetches and reads a web page as clean Markdown using [Jina Reader](https://jina.ai). Always available — no API key required. Set `JINA_API_KEY` to increase your usage quota.

```yaml
environment:
  - JINA_API_KEY=jina_...            # optional — increases quota
  - WEB_FETCH_MAX_TOKENS=2000        # max tokens returned (default 2000, ~8 000 chars)
```

Both tools are visible in the chat as standard tool call blocks.

---

## Admin Dashboard

Nibchat includes a built-in admin dashboard at `/admin`. It is protected by email + password authentication.

### First-time setup

On first startup, if no admin account exists, a one-time **bootstrap token** is printed to the server logs:

```
========================================
  No admin account found.
  Visit /admin/setup to create one.
  Bootstrap token: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
========================================
```

Visit `/admin/setup`, enter the token, and choose an email and password to create the first admin account. The setup page is permanently disabled once an admin exists.

### Features

- **Overview** — total sessions, active sessions (7d / 30d), total conversations, total messages
- **Sessions** — list of all user sessions with conversation count and last-active timestamp; delete a session and all its data
- **Conversation browser** — drill into any session to see its conversations; open any conversation to read the full message history; delete individual conversations
- **Admin accounts** — list all admins, create new accounts, delete accounts, change your own password
- **User management** (when `AUTH_ENABLED=true`) — list registered users with session and conversation counts; create user accounts (bypasses `INVITE_ONLY`); view a user's sessions; delete a user and all their data

### Password requirements

Admin passwords must be at least 8 characters and contain uppercase, lowercase, a number, and a special character.

---

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `PORT` | `3000` | HTTP port the server listens on |
| `DB_PATH` | `/app/data/nibchat.db` | Path to the SQLite database file |
| `SESSION_SECRET` | — | Secret used to encrypt stored API keys — **change this** |
| `MCP_SERVER_URL` | — | Single MCP server URL (HTTP/Streamable transport) |
| `MCP_SERVER_URLS` | — | Comma-separated list of MCP server URLs (takes precedence over `MCP_SERVER_URL`) |
| `OPENAI_API_KEY` | — | Shared OpenAI API key for all users — disables the settings UI when set; also required to enable knowledge base indexing |
| `OPENAI_MODEL` | `gpt-4o` | Model to use when `OPENAI_API_KEY` is set |
| `OPENAI_BASE_URL` | — | Custom OpenAI-compatible endpoint (OpenRouter, Ollama, LM Studio, etc.) — locks settings UI when combined with `OPENAI_MODEL` |
| `AGENT_CONFIG_PATH` | `./agent.yaml` | Path to the agent configuration YAML file |
| `PUBLIC_URL` | — | Public base URL used to build the OAuth redirect URI (e.g. `https://nibchat.example.com`) |
| `OPENAI_EMBEDDING_MODEL` | `text-embedding-3-small` | Embedding model used for knowledge base indexing — set to your provider's embedding model when using a custom endpoint |
| `KNOWLEDGE_CHUNK_SIZE` | `600` | Knowledge base chunk size in tokens (1 token ≈ 4 chars) |
| `KNOWLEDGE_CHUNK_OVERLAP` | `100` | Overlap between consecutive chunks in tokens |
| `TAVILY_API_KEY` | — | Enables the built-in `web_search` tool (get a free key at [tavily.com](https://tavily.com)) |
| `JINA_API_KEY` | — | Optional — increases quota for the built-in `fetch_page` tool |
| `WEB_FETCH_MAX_TOKENS` | `2000` | Max tokens returned by `fetch_page` (1 token ≈ 4 chars) |
| `POSTGRESQL_DSN` | — | PostgreSQL connection string — when set, PostgreSQL is used instead of SQLite (e.g. `postgres://user:pass@host:5432/db`) |
| `AUTH_ENABLED` | `false` | Set to `true` to require email/password login before accessing the chat |
| `INVITE_ONLY` | `false` | Set to `true` to disable open registration — admin must create user accounts |
| `FILE_STORAGE_PATH` | — | Absolute path to a local directory for file storage; if unset, files are stored in the database |
| `S3_BUCKET` | — | Bucket name — enables S3-compatible file storage (takes precedence over `FILE_STORAGE_PATH`) |
| `S3_ENDPOINT` | — | Custom S3 endpoint for R2/MinIO/Tigris; omit for AWS |
| `S3_REGION` | `auto` | S3 region |
| `S3_ACCESS_KEY_ID` | — | S3 access key |
| `S3_SECRET_ACCESS_KEY` | — | S3 secret key |
| `S3_PRESIGN_TTL` | `3600` | Presigned URL TTL in seconds |

---

## PostgreSQL

By default Nibchat uses SQLite. To use PostgreSQL instead, set `POSTGRESQL_DSN`:

```yaml
environment:
  - POSTGRESQL_DSN=postgres://user:password@db:5432/nibchat
```

Nibchat auto-creates all tables on startup. The knowledge base uses a `vector(1536)` column with an HNSW index when the `pgvector` extension is available; if it is not installed, Nibchat falls back to in-memory cosine similarity transparently.

A ready-made PostgreSQL service is included (commented out) in the project's `docker-compose.yml` as a starting point.

---

## File Storage

By default uploaded files are stored as base64 BLOBs inside the database. Two external storage backends are available.

### Local filesystem

```yaml
environment:
  - FILE_STORAGE_PATH=/app/uploads
volumes:
  - nibchat-uploads:/app/uploads
```

Files are written to that directory as raw binary; data URLs are reconstructed on the fly when they are read back.

### S3-compatible object storage

Works with AWS S3, Cloudflare R2, MinIO, Tigris, and any other S3-compatible service.

```yaml
environment:
  - S3_BUCKET=my-nibchat-files
  - S3_REGION=auto                          # or us-east-1, etc.
  - S3_ENDPOINT=https://...r2.cloudflarestorage.com   # omit for AWS
  - S3_ACCESS_KEY_ID=...
  - S3_SECRET_ACCESS_KEY=...
  - S3_PRESIGN_TTL=3600                     # presigned URL TTL in seconds (default 3600)
```

Files are served via short-lived presigned URLs — no public bucket access required.

**Priority**: if both `S3_BUCKET` and `FILE_STORAGE_PATH` are set, S3 wins.

**Cleanup**: when a conversation, session, or user is deleted via the UI or admin panel, their associated files are automatically removed from whichever storage backend is in use.

---

## User Authentication

By default Nibchat is anonymous — anyone who visits the app gets a session automatically. To require login:

```yaml
environment:
  - AUTH_ENABLED=true
```

### Open registration (default when AUTH_ENABLED=true)

Users register at `/register` with an email and password. Conversations are tied to the user account and accessible from any browser or device after login.

### Invite-only mode

```yaml
environment:
  - AUTH_ENABLED=true
  - INVITE_ONLY=true
```

The `/register` page is disabled. Only an admin can create user accounts from the **Users** page in the admin dashboard.

### Account management

- Users can change their password from the **Settings** modal in the sidebar.
- Admins can manage all user accounts from `/admin` → **Users**.

### Password requirements

Passwords must be at least 8 characters and contain uppercase, lowercase, a number, and a special character (applies to both admin and user accounts).

---

## Building from Source

```bash
# Clone the repository
git clone https://github.com/nibchat-ai/nibchat.git
cd nibchat

# Install dependencies
npm install

# Start in development mode (hot reload on both client and server)
npm run dev
```

The client runs on `http://localhost:5173` and the server on `http://localhost:3000`.

To build the production image locally:

```bash
docker build -t nibchat .
```

---

## Architecture

```
┌─────────────────────────────────────┐
│           Browser (React)           │
│  Sidebar · Chat · File Upload · SSE │
└────────────────┬────────────────────┘
                 │ HTTP / SSE
┌────────────────▼────────────────────┐
│         Nibchat Server             │
│  Express · Sessions · SQLite        │
│  OpenAI SDK · Agentic Tool Loop     │
│  Knowledge Indexer (RAG)            │
└──────────┬─────────────┬────────────┘
           │ MCP         │ OpenAI API
┌──────────▼──────┐  ┌───▼───────────┐
│  Your MCP Server│  │  knowledge/   │
│  (tools & UIs)  │  │  (txt/md/pdf) │
└─────────────────┘  └───────────────┘
```

- **Frontend** — React 18, Vite, Tailwind CSS (dark / light mode)
- **Backend** — Node.js, Express, TypeScript (ESM)
- **Database** — SQLite (default) or PostgreSQL via Drizzle ORM (auto-migrated at startup); embeddings stored as BLOBs (SQLite) or `vector(1536)` with HNSW index (pgvector)
- **LLM** — OpenAI API (user-supplied key, any model)
- **Protocol** — Model Context Protocol SDK (HTTP/Streamable transport)
- **Embeddings** — `text-embedding-3-small` via OpenAI API; cosine similarity search in-memory at query time
