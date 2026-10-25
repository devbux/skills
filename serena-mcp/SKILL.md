---
name: serena-mcp
license: MIT
description: >
  Language Server Protocol (LSP) backed semantic code intelligence and refactoring using Serena MCP.
  Provides compiler-grade AST symbol exploration, cross-file reference tracking, type diagnostics,
  and atomic, reference-aware refactoring tools (rename_symbol, safe_delete_symbol,
  replace_symbol_body). Supports 40+ languages; the --context flag selects the toolset
  per client, so a missing tool usually means disabled-by-context, not breakage.
---

# Semantic Code Intelligence with Serena MCP

**Serena** is a Language Server Protocol (LSP) backed Model Context Protocol server. It empowers AI coding assistants with compiler-grade understanding of codebases: exploring symbol hierarchies, tracing cross-file caller/callee references, running type diagnostics, and executing atomic, reference-aware refactorings.

---

## 1. When to Use Serena

Use Serena when semantic symbol awareness provides higher safety and efficiency than raw string searches:

- 🔍 **Symbol Exploration**: Finding functions, interfaces, or classes by name path (e.g. `AuthService/validateToken`) without loading whole files into context.
- 🔗 **Cross-File Reference Tracking**: Discovering all callers, implementations, and declarations across the entire repository before modifying a signature.
- ⚡ **Atomic Refactoring**: Renaming symbols project-wide (`rename_symbol`) or deleting unused symbols safely (`safe_delete_symbol`) with automatic import updates.
- 🩺 **Compiler Diagnostics**: Inspecting language-server errors and warnings on touched files (`get_diagnostics_for_file`).
- 📦 **Token Efficiency**: Reading only the exact symbol body you need (`find_symbol` with `include_body=True`) instead of reading 1,000-line files.

Scope: use Serena when a task crosses file boundaries or needs reference-aware
safety; for single-file trivia, native file tools are fine.

---

## 2. Core Concepts & Invariants

1. **0-Based Line Indexing**: Line numbers returned by Serena tools are 0-based.
2. **Name Path Syntax**:
   - Symbols within a file are referenced hierarchically: `Class/method`, `Interface/property`.
   - Simple search matches any symbol by name; a relative path matches that path and paths ending in it; prefix with `/` (e.g. `/Class/method`) for an exact absolute match.
   - `find_symbol` filters: `depth=1` for members only, `substring_matching` for partial names, `include_kinds` / `exclude_kinds` to narrow by symbol kind.
3. **Internal Serialization & Safe Batching**:
   - Serena executes tool calls sequentially internally. You can safely issue independent tool calls in a single turn without risk of race conditions or data corruption.
4. **Project Activation (only when the context requires it)**:
   - Single-project contexts (`ide`, `claude-code`, `grok`) start with the project already set and disable `activate_project` — do not call it there.
   - Global contexts (the default `desktop-app`, Codex, Claude Desktop) require it: call `activate_project` with `{"project": "<name or path>"}` once at session start.
   - `initial_instructions` provides the Serena Instructions Manual only for clients that do not already read it on connect; in clients whose setup carries the manual (e.g. Claude Code with the recommended setup), calling it is redundant.
5. **Contexts and modes are the control plane**:
   - `--context` sets the base toolset (`desktop-app` default = full set; `claude-code`, `ide`, `codex`, `grok`, `agent`, …). Modes (`planning`, `editing`, `interactive`, `one-shot`, `onboarding`, `no-onboarding`, `no-memories`, `query-projects`) refine it.
   - In most configurations only a subset of tools is enabled at once, and tools marked *optional* are disabled by default. A missing tool usually means "disabled by context", not an install failure — check `get_current_config` first; it prints the active projects, tools, contexts, and modes.

---

## 3. Tool Reference

### A. Exploration & Navigation
| Tool | Purpose | Typical Use Case |
| --- | --- | --- |
| **`activate_project`** | Activates a project by name or path. | Global contexts only, once at session start (`{"project": "/path/to/repo"}`); disabled in single-project contexts. |
| **`get_current_config`** | Prints the active projects, tools, contexts, and modes. | First debugging step when a tool is missing. |
| **`get_symbols_overview`** | Returns top-level symbols in a file. | Rapidly understanding a file's public interface and structure. |
| **`find_symbol`** | Locates symbols matching a name pattern. | Finding definitions; use `depth=1` for members, `include_body=True` to read implementation. |
| **`find_referencing_symbols`** | Finds all usages and call-sites of a symbol. | Pre-refactoring blast-radius analysis across the codebase. |
| **`find_declaration`** | Jumps to symbol definition/declaration. | Resolving where an imported symbol originates. Generally does not work for declarations in external dependencies. |
| **`find_implementations`** | Finds concrete implementations of an interface/trait. | Inspecting polymorphic classes or handler implementations. Only available for some languages, limited by the language server. |
| **`get_diagnostics_for_file`** | Fetches compiler/LSP diagnostics (errors/warnings). | Verifying type correctness after modifying code. |

### B. Atomic Refactoring & Editing
| Tool | Purpose | Typical Use Case |
| --- | --- | --- |
| **`rename_symbol`** | Renames a symbol and updates all call-sites atomically. | Safe, project-wide identifier renaming across declarations and imports. |
| **`safe_delete_symbol`** | Deletes a symbol only if zero references exist. | Removing deprecated code safely without introducing broken imports. |
| **`replace_symbol_body`** | Replaces the exact body of a function or class. | Updating a method implementation cleanly without line-number guesswork. |
| **`insert_after_symbol`** | Inserts new code immediately after a specified symbol. | Appending helper functions or new methods to an existing module. |
| **`insert_before_symbol`** | Inserts new code immediately before a specified symbol. | Adding decorators, imports, or preceding comments. |
| **`replace_content`** | Targeted string or regex replacement within a file. | Small multi-line edits inside a function body. |
| **`replace_in_files`** | Multi-file replacement with dry-run support. | Applying consistent updates across multiple files simultaneously. |

### C. Project Memory & Notes
| Tool | Purpose | Typical Use Case |
| --- | --- | --- |
| **`write_memory`** | Persists a structured note or architectural pattern for the project. | Storing project-specific rules, invariants, or onboarding facts. |
| **`read_memory`** | Reads a stored project memory by key. | Retrieving architectural context across sessions. |
| **`list_memories`** | Lists all persisted memory keys for the project. | Reviewing established project conventions. |

Memories are project-specific `.serena/memories/*.md` files. Also available:
`edit_memory`, `delete_memory` (only when explicitly instructed or permitted),
`rename_memory` (updates references marked with the `mem:` prefix).

### D. Also available (subset depends on context/mode)

| Tool | Notes |
| --- | --- |
| **`onboarding`** | First-session project familiarization; creates project memories. |
| **`serena_repl`** *(optional, BETA)* | One tool executing Python against Serena facades (`lsp.find_symbol(...)`, …). The biggest token saver for multi-step retrievals: only the result you actually need enters the context; intermediate results stay in the Python runtime. |
| **`serena_info`** *(optional)* | Information on an advanced topic on demand, for context efficiency. |
| **`execute_shell_command`**, **`create_text_file`**, **`find_file`**, **`list_dir`**, **`read_file`**, **`search_for_pattern`** | File, search, and shell utilities. In agentic harnesses these are typically disabled by default, since the harness already provides overlapping capabilities. |
| **`get_diagnostics_for_symbol`** *(optional)*, **`restart_language_server`** *(optional)* | Symbol diagnostics; language-server restart. |
| **`delete_lines`** / **`insert_at_line`** / **`replace_lines`** *(optional)* | Line-based edits; `delete_lines` and `replace_lines` require the exact range to have been read first via `read_file`. |
| **`list_queryable_projects`** / **`query_project`** *(optional)* | Read-only queries against projects other than the current one. |
| **`jet_brains_*`** *(optional)* | The JetBrains plugin backend (paid plugin, free trial): move, inline, type hierarchy, inspections, interactive debugging — exclusive to that backend, not the LSP one. |

---

## 4. Recommended Workflows

### 1. Token-Efficient Code Inspection
```
1. `get_symbols_overview(relative_path="src/auth/service.ts")` — file outline first.
2. `find_symbol(name_path_pattern="AuthService/login", relative_path="src/auth/service.ts", include_body=True)` — only the lines of the method you will touch.
```
Standing rule: overview first, bodies only for symbols you will touch.

### 2. Safe Cross-File Refactoring
```
1. `find_referencing_symbols(name_path_pattern="OldService/fetchUser")` — blast radius first.
2. `rename_symbol(name_path_pattern="OldService/fetchUser", new_name="getUserProfile")` — declaration, imports, and call sites update atomically.
3. `get_diagnostics_for_file(relative_path="src/controllers/user.ts")` — confirm zero diagnostics remain.
```

---

## 5. Installation & Client Setup

Serena is written in Python and installed locally via **`uv tool install`**
for zero-latency local execution without git checks on startup. Serena itself
is GPL-3.0-or-later (its SolidLSP component is MIT); this skill file is MIT
like the rest of this repo. **Do not install Serena via an MCP or plugin
marketplace** — those carry outdated and suboptimal installation commands.

### Prerequisites, install & initialise
```bash
# 1. Install uv (if not already installed) — via the installer script,
#    or your package manager: brew install uv on macOS, pip install uv otherwise
curl -LsSf https://astral.sh/uv/install.sh | sh

# 2. Install Serena locally (places binary in ~/.local/bin/serena)
uv tool install -p 3.13 serena-agent

# 3. Initialise (default: language-server backend; add -b JetBrains for the plugin backend)
serena init
```

Some languages need extra dependencies for their language server (e.g. gopls
for Go, rust-analyzer via rustup for Rust). For larger projects, index the
project first (`serena project index` from the project directory) — otherwise
the first tool call can be very slow.

### Client configurations — the `--context` flag is mandatory knowledge

The context selects the toolset; omitting it silently loads the default
`desktop-app` full set everywhere. Recommended path for Claude Code is simply
`serena setup claude-code`.

Claude Code, global (all projects):

```bash
claude mcp add --scope user serena -- serena start-mcp-server --context claude-code --project-from-cwd
```

Claude Code, per project:

```bash
claude mcp add serena -- serena start-mcp-server --context claude-code --project "$(pwd)"
```

VS Code, global (then prompt "Activate the current dir as project using
serena" at session start — VS Code cannot auto-activate in this setting):

```bash
serena start-mcp-server --context=vscode
```

VS Code, workspace (auto-activates the open project):

```bash
serena start-mcp-server --context=vscode --project ${workspaceFolder}
```

Claude Desktop / other global-config clients (activation required, see §2):

```json
{
  "mcpServers": {
    "serena": {
      "command": "serena",
      "args": ["start-mcp-server", "--context=desktop-app"]
    }
  }
}
```

IDEs generally (Cursor, Cline, Windsurf): use `--context ide` to reduce tool
duplication with the IDE's own tools.

Starting Serena standalone:

```bash
serena start-mcp-server --open-web-dashboard False --project /path/to/project
```

If running ephemerally without installing locally, substitute
`uvx --from git+https://github.com/oraios/serena serena` for `serena`.

### Keep Serena working in long sessions

Claude Code and VS Code users should set up the reminder hooks
(`serena-hooks remind`, `activate`, `cleanup`, plus `auto-approve` for Claude
Code) — agents otherwise stop using Serena's tools in long sessions or after
dynamic tool loading (agent drift). If Serena is slow to start under Claude
Code, set `MCP_TIMEOUT` higher (e.g. `export MCP_TIMEOUT=60000`).
