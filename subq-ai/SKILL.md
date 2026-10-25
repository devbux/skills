---
name: subq-ai
license: MIT
description: >
  SubQ integration craft for high-capacity System One decision models, specifically SubQ Snap
  (subq-snap-1.1). Snap answers typed questions with calibrated probabilities over large context
  windows (up to 384K tokens) and multimodal visual inputs (up to 4 images). Use when verifying
  large code diffs, refereeing complex architecture, comparing visual before/after screenshots,
  or building structured, deterministic decision logic in automated software workflows.
---

# Build with SubQ AI (Snap)

SubQ's **Snap** (`subq-snap-1.1`) is a high-capacity **System One decision model**. Like other decision models, Snap does not generate conversational chat prose. Instead, it accepts an application **state** (large multi-file code diffs, logs, documents, or base64 screenshots) along with **typed questions**, returning structured answers backed by **calibrated probabilities**.

With a **384K token context window** (`393,216` tokens) and native support for up to **4 high-resolution images**, Snap acts as an impartial, high-capacity referee capable of evaluating whole modules, extensive pull request diffs, and visual UI comparisons that exceed smaller model limits.

---

## 1. System One Core Philosophy

### Judgments from Snap, Decisions in Code
Integrations work best when Snap provides calibrated judgments and your application code decides what to do with them.

```
┌─────────────────────────────────┐
│ Application State / Diffs / UI  │
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│     Snap Typed Judgments        │ ──► [Noul: 0.94, Choice: "pass", Score: 4/5]
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│    Code Decision Logic          │ ──► Execute action, fail gate, or escalate
└─────────────────────────────────┘
```

- **Avoid asking Snap to make business policy decisions**: Rather than asking *"Should we block this deploy?"*, ask atomic judgment questions: *"Does this diff introduce any unhandled promise rejections?"* or *"Does this changeset violate the error boundary contract?"*.
- **Represent logic in code**: Code holds the truth table that maps probabilities to actions. This keeps decision logic testable, versioned, and deterministic.

---

## 2. Question Primitives

Snap supports three typed question types, plus a batch tool that evaluates several questions over one shared state:

| Question Type | Output Shape | When to Use |
| --- | --- | --- |
| **`noul`** | Boolean probability `[0.0, 1.0]` | Verifying whether a specific assertion is true (e.g. invariant compliance, bug presence, security leak). |
| **`choice`** | Categorical distribution over named options | Picking the best option among mutually exclusive candidates (e.g. error categorization, intent routing). |
| **`score`** | Graded score along defined discrete levels | Evaluating qualitative levels (e.g. code readability 1–5, test thoroughness, accessibility conformance). |
| **`decide`** (batch tool, not a question type) | Multiple typed answers in one response | Evaluating multiple interrelated questions over the same shared state in a single request. |

### Calibrated Probabilities & Thresholds
- **Noul Thresholds**:
  - `> 0.80`: High-confidence affirmation (**True / Pass**). Code adheres to criteria.
  - `0.50 – 0.80`: **Inconclusive / Uncertain**. Indicates ambiguous evidence, missing context, or borderline criteria. Code should gather hard facts, re-query with explicit criteria, or inspect surrounding code context.
  - `< 0.50`: High-confidence rejection (**False / Defect / Stop**). STOP immediately, investigate the flaw, and fix code before proceeding.
- **Top Answer by Default**: In `choice`, use the option with the highest probability mass, but ensure the runner-up is checked when confidence is diffuse.

---

## 3. High-Capacity Capabilities

### A. Large Context Windows (384K Tokens)
Snap's 384K context window allows passing entire pull request diffs, multiple related source files, and database schemas together:
- Include full surrounding files when checking call-site safety across caller and callee modules.
- Include test files alongside implementation diffs to verify test coverage matches modified code.

### B. Visual Multimodal Comparison (Up to 4 Images)
Images travel as a separate top-level `images` parameter on any tool call, not embedded in the state: an array of up to 4 objects of the form `{ "mediaType": "image/png" | "image/jpeg" | "image/webp", "base64": "..." }`, with plain base64 and no `data:` prefix. Each image must decode to at most 4 MiB, 8 MiB in total, and every question in the request sees every image:
- **Visual Regression**: Compare `baseline_screenshot` against `candidate_screenshot`.
- **Figma to Code Verification**: Compare `design_mockup` against `rendered_component`.
- **State Progression**: Compare initial state, loading skeleton, and populated UI.

---

## 4. MCP Tools & Request Signatures

When using the SubQ Snap MCP server (`@devbux/subq-snap-mcp`), the following tools are available:

### 1. `snap_noul` (Boolean Probability)
```json
{
  "state": "git diff or source code string...",
  "instructions": "Does this diff introduce any memory leaks or uncleaned event listeners?",
  "criteria": {
    "true": "An event listener, timer, or subscription is registered without cleanup on unmount.",
    "false": "All subscriptions and listeners are properly deregistered in cleanup callbacks."
  }
}
```

### 2. `snap_choice` (Categorical Selection)
```json
{
  "state": "diff or error log...",
  "instructions": "Which category best describes the regression in this changeset?",
  "options": {
    "syntax_error": "Static syntax, typing, or compilation breakage.",
    "race_condition": "Asynchronous state mutation without proper locking or ordering.",
    "performance_regression": "Unnecessary re-renders, layout thrashing, or blocking loops.",
    "clean": "No architectural regression is present."
  }
}
```
`options` is an object mapping each option label to its description (2–255 options).

### 3. `snap_score` (Multi-Level Graded Evaluation)
```json
{
  "state": "unit test file contents...",
  "instructions": "Rate the thoroughness of edge-case coverage in this test suite.",
  "levels": [
    "Only happy path tested with trivial mocks.",
    "Tests standard success and one basic error branch.",
    "Tests multiple error branches and boundary inputs.",
    "Thoroughly tests concurrent calls, null/undefined states, and failure recovery."
  ]
}
```
`levels` is an ordered array of level descriptions, lowest to highest (2–10 levels). The returned score is probability-weighted, so it can fall between levels.

### 4. `snap_decide` (Batch Decision Matrix)
Evaluates multiple questions against a single shared state efficiently in one parallel pass:
```json
{
  "state": {
    "diff": "...",
    "changed_files": ["src/components/user-card.tsx", "src/hooks/use-user.ts"]
  },
  "questions": {
    "q1_clean_architecture": {
      "type": "noul",
      "instructions": "Does this implementation conform to project component boundaries without anti-patterns?"
    },
    "q2_error_handling": {
      "type": "noul",
      "instructions": "Are all API error responses caught and presented gracefully to the user?"
    }
  }
}
```

### 5. `snap_list_models` (Available Models)
Takes no parameters. Lists the model IDs available to the configured SubQ API key (GET `/v1/models`, OpenAI-compatible shape). Use a returned id with the `model` parameter on any tool above when you need something other than the default `subq-snap-1.1`; note that the Decisions tools need a decision model, not the separate `subq-preview` chat model.

### 6. Multimodal Visual Comparison Request Example
A `snap_noul` call with images as a top-level parameter (plain base64, no `data:` prefix; up to 4 images, 4 MiB each, 8 MiB total):
```json
{
  "state": {
    "description": "Visual regression comparison for modal dialog on dark mode"
  },
  "instructions": "Does Image 2 match the design layout and contrast specifications of Image 1 without visual clipping?",
  "criteria": {
    "true": "Layout, typography, padding, and contrast in Image 2 match Image 1 cleanly without clipping.",
    "false": "There is noticeable layout shift, incorrect contrast, or clipped content in Image 2."
  },
  "images": [
    { "mediaType": "image/png", "base64": "iVBORw0KGgoAAAANSUhEUgAA..." },
    { "mediaType": "image/png", "base64": "iVBORw0KGgoAAAANSUhEUgAA..." }
  ]
}
```

---

## 5. Designing the State (The Human Test)

A reliable check for any Snap question is the **Human Test**:
> *Could a senior engineer, given only this state and this question, answer quickly and conclusively?*

- **Provide complete context**: When asking about a function, include its definition and the definitions of imported custom types.
- **Structure with semantic keys**: Use JSON objects with clear paths (`state.changed_files`, `state.diff`, `state.error_trace`) rather than unformatted blobs.
- **Strip irrelevancies**: Remove large binary assets or minified third-party vendor code that unnecessarily dilutes context.

---

## 6. Data Privacy & Handling

Snap is a hosted decision model service:
- **Never submit production secrets**: Do not pass raw database credentials, live customer PII, secret keys, or authentication tokens in the `state` payload.
- **Seeded data for UI screenshots**: Any screenshot sent to Snap must be generated from synthetic, seeded, or mock test data.
- **Redact before submission**: If diffs or logs contain real sensitive identifiers, redact them before calling Snap.

---

## 7. Anti-Patterns & Yellow Flags

- 🚩 **Prompt Gymnastics**: Never write conversational preambles ("Please act as a senior developer and carefully examine..."). Snap is a decision model; write concise, factual criteria.
- 🚩 **Double-Barreled Questions**: Avoid questions joined with "and/or" (e.g. *"Is this code clean and accessible?"*). Split into two independent questions: one for architecture, one for accessibility.
- 🚩 **Assuming Confidence Equals Correctness**: Calibrated probability indicates distribution sharpness. Always check the actual probability value against your code threshold.
- 🚩 **Ignoring Uncertain Bands**: Never treat `0.55` as a solid "Yes". Values in the `0.50–0.80` range require verification or escalation.

---

## 8. Setup & Client Configuration

### Package Execution
Run directly via `bunx` or `npx`:
```bash
bunx @devbux/subq-snap-mcp@latest
# Or using npx:
# npx -y @devbux/subq-snap-mcp@latest
```

### Environment Variable
- `SUBQ_API_KEY`: API key from [subq.ai](https://docs.subq.ai/).

### MCP Client Configurations

#### Cursor / Claude Desktop / VS Code (`mcpServers`)
```json
{
  "mcpServers": {
    "subq": {
      "command": "bunx",
      "args": ["@devbux/subq-snap-mcp@latest"],
      "env": {
        "SUBQ_API_KEY": "YOUR_SUBQ_API_KEY"
      }
    }
  }
}
```

#### VS Code (`.vscode/mcp.json` - Native Schema)
```json
{
  "servers": {
    "subq": {
      "command": "bunx",
      "args": ["@devbux/subq-snap-mcp@latest"],
      "env": {
        "SUBQ_API_KEY": "${input:subqApiKey}"
      }
    }
  }
}
```

#### Claude Code CLI
```bash
claude mcp add subq -e SUBQ_API_KEY=YOUR_KEY -- bunx @devbux/subq-snap-mcp@latest
```
