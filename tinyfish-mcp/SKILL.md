---
name: tinyfish-mcp
license: MIT
description: >
  Live web search and webpage content fetching using the direct TinyFish Model Context Protocol
  (MCP) server (search_web, fetch_url). Provides real-time organic search results, authoritative
  documentation discovery, and clean webpage extraction. Completely standalone and works with
  any AI coding assistant or harness.
---

# Web Search & Content Fetching with TinyFish MCP

The **TinyFish MCP** connects AI coding agents directly to real-time web search and webpage fetching via the direct TinyFish Model Context Protocol endpoint (`https://agent.tinyfish.ai/mcp`). It provides live organic web search and clean webpage extraction without requiring local proxy gateways.

---

## 1. When to Use TinyFish MCP

TinyFish MCP provides two complementary tools for real-time web research:

### A. Live Organic Web Search (`search_web`)
Use when you need to discover current external information or canonical links:
- 🔍 **Documentation & Release Discovery**: Finding official library docs, release announcements, migration guides, and RFCs.
- ⚡ **Real-Time Fact Verification**: Checking current package versions, deprecation notices, breaking changes, or security advisories (CVEs).
- 🛠️ **Troubleshooting & Error Diagnosis**: Searching for compiler error messages, GitHub issues, and community solutions.
- 🌐 **URL Discovery**: Discovering the exact canonical URLs for a technical topic before deep inspection.

### B. Lightweight Webpage Fetching (`fetch_url`)
Use when you need clean, readable content from a specific web URL:
- 📄 **Webpage Extraction**: Reading articles, blog posts, documentation pages, or changelogs.
- 🧹 **Noise Stripping**: Converting HTML pages into clean, LLM-friendly Markdown while removing scripts, styles, and boilerplate.
- 📑 **Windowed Reading**: Paginating through large documents using character offsets.

---

## 2. Tool Reference

### A. `search_web` (Live Web Search)
Performs real-time organic web search and returns ranked results with titles, URLs, and contextual snippets.

#### Signature & Parameters
```json
{
  "query": "TypeScript 5.8 satisfies operator type narrowing",
  "max_results": 5,
  "country": "us",
  "language": "en",
  "search_type": "web"
}
```

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| **`query`** | `string` | **Yes** | The search query string. Keep queries concise and keyword-focused. |
| **`max_results`** | `number` | No | Number of results to return (default: `5`, max: `20`). |
| **`country`** | `string` | No | Optional two-letter country code filter (e.g. `"us"`, `"de"`). |
| **`language`** | `string` | No | Optional two-letter language code filter (e.g. `"en"`). |
| **`search_type`** | `string` | No | Search type (default: `"web"`). |

---

### B. `fetch_url` (Webpage Fetching)
Fetches clean, markdown-converted content from a specific public web URL.

#### Signature & Parameters
```json
{
  "url": "https://example.com/changelog",
  "format": "markdown",
  "max_characters": 0,
  "offset": 0
}
```

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| **`url`** | `string` | **Yes** | Absolute `http://` or `https://` target URL to fetch. |
| **`format`** | `string` | No | Output format: `"markdown"` (default), `"text"`, or `"html"`. |
| **`max_characters`** | `number` | No | Maximum number of characters to return (`0` returns entire document). Use a smaller value for chunked reading. |
| **`offset`** | `number` | No | Character offset to begin reading from for windowed pagination. |

---

## 3. Best Practices & Operational Rules

### 1. Constructing Effective Queries
- **Use technical keywords**: Instead of conversational questions like *"How do I fix this error?"*, use targeted technical identifiers: `"React 19" "useActionState" type signature examples`.
- **Exclude fluff**: Avoid polite phrases, filler words, or unnecessary punctuation.
- **Scope by version**: Always append the library version when researching modern frameworks (e.g. `Tailwind CSS v4 @theme inline`).

### 2. Sequential Execution (Strict Invariant)
Execute web search queries and page fetches sequentially. Do not fire parallel requests in rapid succession to prevent rate-limiting (HTTP 429) from search providers.

### 3. Data Privacy & Security
Search queries and fetched URLs are processed by external web infrastructure:
- **Never include secrets**: Do not pass passwords, API keys, private tokens, or sensitive internal credentials in search queries.
- **Public endpoints only**: Only fetch public URLs; never pass internal network hostnames, private staging IP addresses, or localhost endpoints.

---

## 4. Setup & Client Configuration

### Direct MCP Endpoint
- **Endpoint URL**: `https://agent.tinyfish.ai/mcp`
- **Authentication**: Optional Bearer token from [TinyFish](https://agent.tinyfish.ai/). Works unauthenticated with basic limits; setting an API key unlocks higher throughput.

### Configuration Examples

#### Cursor / Claude Desktop / Standard MCP (`mcpServers`)
```json
{
  "mcpServers": {
    "tinyfish": {
      "url": "https://agent.tinyfish.ai/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_TINYFISH_API_KEY"
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
    "tinyfish": {
      "url": "https://agent.tinyfish.ai/mcp",
      "headers": {
        "Authorization": "Bearer ${input:tinyfishApiKey}"
      }
    }
  }
}
```

#### Claude Code CLI
```bash
claude mcp add --transport http tinyfish "https://agent.tinyfish.ai/mcp" --header "Authorization: Bearer YOUR_TINYFISH_API_KEY"
```
