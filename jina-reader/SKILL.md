---
name: jina-reader
license: MIT
description: >
  Web reading, document extraction, and web search using the Jina Reader Model Context Protocol
  (MCP) server (read_url, search). Converts web pages, documentation sites, GitHub repositories,
  and technical articles into clean, LLM-friendly Markdown while executing client-side JavaScript
  for SPAs. Also provides integrated web search for real-time information retrieval. Completely
  standalone and works with any AI coding assistant or harness.
---

# Web Reading & Search with Jina Reader MCP

The **Jina Reader MCP** provides intelligent, reader-mode document extraction and web search via the Jina Reader Model Context Protocol endpoint (`https://mcp.jina.ai/v1?include_tags=search,read`). It transforms noisy, JavaScript-rendered web pages, documentation portals, and online resources into clean, LLM-friendly Markdown with high fidelity.

---

## 1. When to Use Jina Reader

Jina Reader provides two complementary capabilities:

### A. Deep Document Reading (`read_url`)
Use when you have a specific target URL and need its complete, readable content:
- 📚 **Official Library Documentation**: Reading framework guides, API references, configuration schemas, and migration paths.
- 🐙 **GitHub & Repositories**: Reading raw issues, pull request discussions, release changelogs, or RFC documents.
- 📰 **Technical Articles & Benchmarks**: Reading engineering deep-dives, blog posts, and whitepapers.
- ⚡ **JavaScript SPAs & Dynamic Pages**: Automatically executing client-side JavaScript to extract DOM content that standard HTTP fetch commands miss.

### B. Integrated Web Search (`search`)
Use when you need to query the web for real-time answers, discover documentation links, or look up error solutions:
- 🔍 **Documentation & API Lookup**: Finding the canonical guide or reference page for a library or API.
- 🛠️ **Error Diagnostics**: Searching for specific compiler error strings, stack traces, and GitHub issues.
- 🌐 **General Web Search**: Gathering real-time external information directly in clean markdown format.

---

## 2. Tool Reference

### A. `read_url` (Deep Page Reader)
Fetches clean, reader-mode markdown from any public web URL. Strips banners, navigation headers, cookie notices, and advertisements.

#### Signature & Parameters
```json
{
  "url": "https://example.com/docs/api-guide",
  "format": "markdown",
  "max_characters": 0,
  "offset": 0
}
```

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| **`url`** | `string` | **Yes** | The absolute `http://` or `https://` web URL to fetch and read. |
| **`format`** | `string` | No | Output format: `"markdown"` (default, preserves headings/code/tables/links), `"text"` (plain text), or `"html"`. |
| **`max_characters`** | `number` | No | Maximum number of characters to return in this call. `0` (default) returns the full document. Use a smaller value (e.g. `15000`) for windowed reading of very large pages. |
| **`offset`** | `number` | No | Character offset to begin reading from. Used to resume or paginate through long documents. |

---

### B. `search` (Web Search)
Performs web search via Jina and returns ranked results directly formatted in clean, reader-friendly markdown.

#### Signature & Parameters
```json
{
  "query": "React 19 Server Actions cache private"
}
```

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| **`query`** | `string` | **Yes** | The search phrase or technical query string to search on the web. |

---

## 3. Best Practices & Operational Rules

### 1. Sequential Execution (Strict Invariant)
Always execute `read_url` and `search` requests sequentially. Do not fire multiple web requests in parallel to prevent rate-limit errors (HTTP 429) or connection drops. Wait for one document or query to complete before initiating the next.

### 2. Large Document Pagination
For long documents (specifications, monolithic documentation pages):
1. Issue an initial call with `max_characters: 20000`.
2. Inspect the returned section and continuation indicator.
3. If additional content is needed, make the subsequent call with `offset: 20000`.
4. Stop as soon as the target section or concept is retrieved to conserve context tokens.

### 3. Data Privacy & Cloud Boundaries
Jina Reader operates through an external cloud endpoint:
- **Never submit private credentials**: Do not query private internal URLs containing authentication tokens, session cookies, passwords, or personal data.
- **Public resources only**: Query public documentation, repositories, and articles.

### 4. Error Handling & Recovery
- **HTTP 404 (Not Found)**: Use `search` to locate the updated or relocated documentation URL.
- **HTTP 403 / Bot Challenge**: Jina Reader bypasses standard bot challenges; if a page remains inaccessible, search for an official GitHub repository mirror or alternative documentation source.
- **HTTP 429 (Rate Limit)**: Pause briefly before retrying, or configure an optional `JINA_API_KEY` for higher limits.

---

## 4. Setup & Client Configuration

### Direct MCP Endpoint
- **Endpoint URL**: `https://mcp.jina.ai/v1?include_tags=search,read`
- **Authentication**: Optional Bearer token from [jina.ai](https://jina.ai). Works unauthenticated with basic limits; an API key unlocks unbounded document lengths and higher throughput.

### Configuration Examples

#### Cursor / Claude Desktop / Standard MCP (`mcpServers`)
```json
{
  "mcpServers": {
    "jina": {
      "url": "https://mcp.jina.ai/v1?include_tags=search,read",
      "headers": {
        "Authorization": "Bearer YOUR_JINA_API_KEY"
      }
    }
  }
}
```
*(If unauthenticated, omit the `"headers"` block).*

#### VS Code Native Schema (`.vscode/mcp.json`)
```json
{
  "servers": {
    "jina": {
      "url": "https://mcp.jina.ai/v1?include_tags=search,read",
      "headers": {
        "Authorization": "Bearer ${input:jinaApiKey}"
      }
    }
  }
}
```

#### Claude Code CLI
```bash
claude mcp add --transport http jina "https://mcp.jina.ai/v1?include_tags=search,read" --header "Authorization: Bearer YOUR_JINA_API_KEY"
```
