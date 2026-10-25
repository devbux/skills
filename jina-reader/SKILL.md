---
name: jina-reader
license: MIT
description: >
  Web reading, document extraction, and web search using the Jina Reader Model Context Protocol
  (MCP) server (read_url, search_web, search_arxiv, search_ssrn, search_images,
  search_jina_blog, capture_screenshot_url). Converts web pages, documentation sites,
  GitHub repositories, and technical articles into clean, LLM-friendly Markdown,
  with question-grounded passage extraction as the token-efficient path and
  server-side tool filtering to keep registration cheap. Completely standalone
  and works with any AI coding assistant or harness.
---

# Web Reading & Search with Jina Reader MCP

The **Jina Reader MCP** server (`https://mcp.jina.ai/v1`, Streamable HTTP)
provides reader-mode document extraction and web search. The full server has 12
tools; the filtered endpoint most users want is
`https://mcp.jina.ai/v1?include_tags=search,read`, which registers exactly 7:
`read_url`, `capture_screenshot_url`, `search_web`, `search_arxiv`,
`search_ssrn`, `search_images`, `search_jina_blog`.

---

## 1. The core workflow: search, then read

Search tools return **titles, URLs, and snippets — not page content**. They
cannot be used alone: always follow with `read_url` on the result URLs. One
`read_url` call takes up to 5 URLs at once.

- Have a URL, need its content → `read_url`.
- Need to discover URLs → `search_web` (general), `search_arxiv` (papers),
  `search_ssrn` (social science / economics / law / finance),
  `search_images` (images, one query at a time), `search_jina_blog` (Jina news
  and blog posts, no API key needed).

---

## 2. Tool reference

### A. `read_url` (Deep Page Reader)

Reads a web page or PDF as Markdown, returning YAML metadata (url, title,
links, images) plus content. `url` takes one URL or an **array of up to 5**,
run concurrently in a single round trip.

| Parameter | Default | Notes |
| --- | --- | --- |
| `url` | required | One URL, or an array of up to 5. |
| `question` | unset | Unset returns the full page. Set returns scored passages instead of `content` (see §3). Gates `chunk_size` and `topk`. |
| `chunk_size` | `100` | Target passage size in words, split at sentence boundaries. Range 1–4096. Use 40–70 for commands, signatures, and numbers; 150–200 for explanation. |
| `topk` | `1` | Passages to keep, best first. Range 1–50. |
| `withAllLinks` / `withAllImages` | — | Include all links / images. Set once for the whole array, not per entry. |
| `ocr` | off | For scanned pages and image-only PDFs: renders the page and returns Markdown with formulas and tables intact. One page per call (page 1 unless `page` says otherwise), and it bills more tokens per page than a plain read costs per document — turn it on only when the HTML path fails or the layout matters. |
| `page` | `1` | Page number for OCR reads. Shared across a URL array, so several pages of one document take one call each. |

A question-grounded read gets a 60-second budget against 30 seconds for a plain
read (chunking and reranking run after the fetch).

### B. `search_web` (Web Search) — API key required

Searches the web; returns titles, URLs, and engine snippets. `query` takes one
query or an **array of up to 5**, run concurrently. Always follow with
`read_url` on the URLs you pick.

### C. The other tools on the filtered endpoint

| Tool | Key required | Notes |
| --- | --- | --- |
| `capture_screenshot_url` | Optional | Captures a screenshot of a page; returns a base64 PNG. |
| `search_arxiv` | Yes | Academic papers and preprints on arXiv. |
| `search_ssrn` | Yes | Papers on SSRN. |
| `search_images` | Yes | Image search; one query at a time (no arrays). |
| `search_jina_blog` | No | Jina AI news and blog posts. The only search tool that works without a key. |

### D. Beyond the filtered endpoint

The unfiltered server also has `primer` (current contextual information, no
key), `guess_datetime_url` (page publish / last-update datetime with a
confidence score, no key), `extract_pdf` (figures, tables, and equations from
PDFs, key required), `sort_by_relevance` (reranking, key required), and
`deduplicate_strings` (semantically unique strings, key required). If you need
only reading and search, the filtered endpoint in §4 keeps these out of your
tool list entirely.

---

## 3. Token rules that matter

- **Question mode is the token saver.** Pass `question` and the page is
  chunked, passages are scored against it by Reranker v3.5, and only the best
  ones return — as `snippets` with `snippet_source: content` and no `content`
  field. Measured on the official README: a 69,530-byte page → 1,195 bytes
  with a question; an 83,786-byte docs page → 874 bytes. If extraction cannot
  run, the full body comes back with `snippet_source: full_content` and a note.
- **Filter server-side, not client-side.** Registering a tool costs tokens for
  its name, description, and schema whether or not it is called. Filter on the
  endpoint URL: `include_tools`, `exclude_tools`, `include_tags`,
  `exclude_tags`, `max_tokens`, applied in that precedence order
  (exclude_tools > exclude_tags > include_tools > include_tags). The leanest
  setup is `include_tools=read_url,search_web`. Tags: `search` (the 5 search
  tools), `read` (`read_url`, `capture_screenshot_url`), `utility` (`primer`,
  `guess_datetime_url`, `extract_pdf`), `rerank` (`sort_by_relevance`,
  `deduplicate_strings`).
- **Truncation is marked, not silent.** Claude Code, Claude Desktop, and
  Cursor enforce a 25k-token limit on MCP tool responses, so the server applies
  a guardrail to `read_url`: items are kept whole while they fit, the first
  that does not fit is cut to a prefix, later items are dropped, and a
  `[jina-mcp] ...` note records what was truncated. At least one item always
  survives. Set your own budget with `max_tokens` on the endpoint URL
  (`max_tokens=50000`); `max_tokens=0` disables truncation entirely.
- **Question-mode failure modes (do not get bitten by these).** Positional
  questions fail — ranking matches text, not document order, so read the first
  screen for "latest / first / current". Tables, fenced code, and page
  furniture are removed before ranking, so a number that lives in a table is
  not reachable this way. Inline code loses tokens — never run a command
  copied out of a passage without checking the source. Blocked pages return
  their login wall as content with no error.
- **No sequential-only rule.** Arrays run concurrently by design; the old
  `parallel_*` tools were removed in v1.10.0 precisely because arrays replaced
  them. Batch URLs and queries instead of calling one at a time.
- **Stale tools?** Some clients cache tool definitions and do not refresh
  them. If tools are missing or look outdated, remove and re-add the server
  to force a refresh.

---

## 4. Setup & Client Configuration

- **Base URL**: `https://mcp.jina.ai/v1` (the `/sse` URL is kept for backward
  compatibility; `/v1` is recommended — both use Streamable HTTP).
- **Filtered URL** (reading + search only):
  `https://mcp.jina.ai/v1?include_tags=search,read`
- **Authentication**: Bearer token is **optional**. Reads work without a key
  at rate limits; search requires a key except `search_jina_blog`. A key buys
  higher limits. Free keys at jina.ai. Server name in official examples is
  `jina-mcp-server`.

### Cursor / Claude Desktop / Standard MCP (`mcpServers`)

```json
{
  "mcpServers": {
    "jina-mcp-server": {
      "url": "https://mcp.jina.ai/v1?include_tags=search,read",
      "headers": {
        "Authorization": "Bearer ${JINA_API_KEY}"
      }
    }
  }
}
```

Omit the `headers` block if unauthenticated (reads only). Clients without
native remote-MCP support need the `mcp-remote` local proxy instead:

```json
{
  "mcpServers": {
    "jina-mcp-server": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "https://mcp.jina.ai/v1?include_tags=search,read",
        "--header",
        "Authorization: Bearer ${JINA_API_KEY}"
      ]
    }
  }
}
```

### VS Code (`.vscode/mcp.json`, native schema)

```json
{
  "servers": {
    "jina-mcp-server": {
      "type": "http",
      "url": "https://mcp.jina.ai/v1?include_tags=search,read",
      "headers": {
        "Authorization": "Bearer ${input:jinaApiKey}"
      }
    }
  }
}
```

### Claude Code CLI

```bash
claude mcp add -s user --transport http jina https://mcp.jina.ai/v1 --header "Authorization: Bearer ${JINA_API_KEY}"
```

Use the filtered URL in place of the base URL when you want only the 7
reading/search tools registered.
