---
name: serena-mcp
license: MIT
description: >
  Language Server Protocol (LSP) backed semantic code intelligence and refactoring using Serena MCP.
  Provides compiler-grade AST symbol exploration, cross-file reference tracking, type diagnostics,
  and atomic, reference-aware refactoring tools (rename_symbol, safe_delete_symbol,
  replace_symbol_body). Universal across TypeScript, JavaScript, Python, Go, Rust, and C#.
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

> [!TIP]
> **Optional Tooling Guideline**:
> If your AI assistant or development environment already provides fast native AST symbol search and file viewing tools, Serena is strictly optional. Use Serena when native tools lack cross-file symbol graph intelligence or when performing automated cross-file refactorings.

---

## 2. Core Concepts & Invariants

1. **0-Based Line Indexing**: Line numbers returned by Serena tools are 0-based.
2. **Name Path Syntax**:
   - Symbols within a file are referenced hierarchically: `Class/method`, `Interface/property`.
   - Overloaded symbols append an index: `Class/method[0]`, `Class/method[1]`.
   - Simple search matches any symbol by name; prefix with `/` (e.g. `/Class/method`) for an exact absolute match.
3. **Internal Serialization & Safe Batching**:
   - Serena executes tool calls sequentially internally. You can safely issue independent tool calls in a single turn without risk of race conditions or data corruption.
4. **Project Activation**:
   - Always initialize the project root with `activate_project` and review initial setup via `initial_instructions`.

---

## 3. Tool Reference

### A. Exploration & Navigation
| Tool | Purpose | Typical Use Case |
| --- | --- | --- |
| **`activate_project`** | Activates the target project directory. | Called once at session start (`{"project_path": "/path/to/repo"}`). |
| **`get_symbols_overview`** | Returns top-level symbols in a file. | Rapidly understanding a file's public interface and structure. |
| **`find_symbol`** | Locates symbols matching a name pattern. | Finding definitions; use `depth=1` for members, `include_body=True` to read implementation. |
| **`find_referencing_symbols`** | Finds all usages and call-sites of a symbol. | Pre-refactoring blast-radius analysis across the codebase. |
| **`find_declaration`** | Jumps to symbol definition/declaration. | Resolving where an imported symbol originates. |
| **`find_implementations`** | Finds concrete implementations of an interface/trait. | Inspecting polymorphic classes or handler implementations. |
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

---

## 4. Recommended Workflows

### 1. Token-Efficient Code Inspection
```
1. `get_symbols_overview(relative_path="src/auth/service.ts")`
   ──► Review list of classes, functions, and interfaces.
2. `find_symbol(name_path_pattern="AuthService/login", relative_path="src/auth/service.ts", include_body=True)`
   ──► Reads only the exact lines of the `login` method, saving context tokens.
```

### 2. Safe Cross-File Refactoring
```
1. `find_referencing_symbols(name_path_pattern="OldService/fetchUser")`
   ──► Inspect all call sites across the project.
2. `rename_symbol(name_path_pattern="OldService/fetchUser", new_name="getUserProfile")`
   ──► Serena updates declaration, imports, and all call sites in one atomic operation.
3. `get_diagnostics_for_file(relative_path="src/controllers/user.ts")`
   ──► Confirm zero compiler diagnostics or type errors remain.
```

---

## 5. Installation & Client Setup

Serena is written in Python and installed locally via **`uv tool install`** for zero-latency local execution without git checks on startup.

### Prerequisites & Local Installation
```bash
# 1. Install uv (if not already installed)
curl -LsSf https://astral.sh/uv/install.sh | sh

# 2. Install Serena locally (places binary in ~/.local/bin/serena)
uv tool install -p 3.13 serena-agent

# Or from GitHub source:
# uv tool install git+https://github.com/oraios/serena
```

### Starting Serena Standalone
```bash
serena start-mcp-server --open-web-dashboard=False --project /path/to/project
```

### MCP Client Configurations

#### Cursor / Claude Desktop / VS Code (`mcpServers`)
```json
{
  "mcpServers": {
    "serena": {
      "command": "serena",
      "args": [
        "start-mcp-server",
        "--open-web-dashboard=False",
        "--project-from-cwd"
      ]
    }
  }
}
```

#### VS Code (`.vscode/mcp.json` - Native Schema)
```json
{
  "servers": {
    "serena": {
      "command": "serena",
      "args": [
        "start-mcp-server",
        "--open-web-dashboard=False",
        "--project-from-cwd"
      ]
    }
  }
}
```

#### Claude Code CLI
```bash
claude mcp add serena -- serena start-mcp-server --open-web-dashboard=False --project-from-cwd
```

*(Note: If running ephemerally without installing locally, substitute `uvx --from git+https://github.com/oraios/serena serena` for `serena`).*
