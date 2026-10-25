# Agent Skills Catalog

A curated collection of standalone, universal Agent Skills for AI coding assistants and harnesses (Claude Code, Cursor, Antigravity, Windsurf, Codex, etc.).

## Included Skills

| Skill | Description | Primary Capabilities |
| --- | --- | --- |
| **[`jina-reader`](./jina-reader/SKILL.md)** | Deep web reading, clean Markdown extraction, question-grounded passage retrieval, and integrated web search via Jina Reader MCP. | `read_url`, `search_web`, `search_arxiv`, `search_ssrn`, `search_images`, `search_jina_blog`, `capture_screenshot_url` |
| **[`tinyfish-mcp`](./tinyfish-mcp/SKILL.md)** | Live web search, webpage extraction, browser automation, monitors, and deep research via TinyFish MCP (search and fetch free; automation and research metered). | `search`, `fetch_content`, `run_web_automation`, `start_deep_research`, monitors, `get_wallet`, + more |
| **[`subq-ai`](./subq-ai/SKILL.md)** | High-capacity System One decision model craft for SubQ Snap (384K context, multimodal vision with up to 4 images, calibrated probabilities). | `snap_noul`, `snap_choice`, `snap_score`, `snap_decide`, `snap_list_models` |
| **[`serena-mcp`](./serena-mcp/SKILL.md)** | Language Server Protocol (LSP) semantic code intelligence, AST symbol exploration, type diagnostics, and atomic refactoring (40+ languages; toolset selected per client via `--context`). | `find_symbol`, `rename_symbol`, `safe_delete_symbol`, `get_current_config`, `serena_repl`, `onboarding`, etc. |

---

## Installation (`bunx skills`)

Install any skill directly into your AI coding agent using `bunx skills`:

```bash
# Global installation across all supported AI coding agents
bunx skills add devbux/skills --skill jina-reader -g
bunx skills add devbux/skills --skill tinyfish-mcp -g
bunx skills add devbux/skills --skill subq-ai -g
bunx skills add devbux/skills --skill serena-mcp -g

# Or install all skills from the repository at once
bunx skills add devbux/skills -g
```

### Agent-Scoped Installation
```bash
# Scope to Claude Code only
bunx skills add devbux/skills --skill jina-reader -a claude-code

# Scope to Cursor only
bunx skills add devbux/skills --skill jina-reader -a cursor
```

---

## License

[MIT](./LICENSE)
