# max-mcp: Autonomous Software Builder

**Project:** max-mcp
**Location:** `/home/maxwell/max-mcp`
**Description:** An MCP Server that provides an autonomous "resolve_issue" tool. It iteratively writes code and runs tests to fix GitHub issues or implement features.

## 1. Architecture

The system runs locally and uses temporary directories to safely build and test code.

### Core Components
*   **MCP Server:** Exposes the tool capability to clients (Claude Desktop, Cursor, etc.).
*   **Agent Engine:** A loop that cycles through `Think` -> `Edit` -> `Run Tests` -> `Analyze Output`.
*   **Sandbox:** Manages temporary directories and `git` operations (clone/checkout).

## 2. Technology Stack

*   **Runtime:** Python 3.14
*   **MCP SDK:** `mcp` (Model Context Protocol Python SDK)
*   **LLM Clients:**
    *   `anthropic` (for Claude models)
    *   `openai` (standard interface for GitHub Models/Copilot)
*   **Git:** `gitpython`
*   **Env Management:** `python-dotenv`

## 3. Supported Models

The agent supports the following models via the `LLM_PROVIDER` and `LLM_MODEL` env vars.

### Anthropic (Provider: `anthropic`)
*   `claude-3-5-sonnet-20241022` (Recommended for Coding)
*   `claude-3-opus-20240229`

### GitHub Copilot / GitHub Models (Provider: `openai` compatible)
*   `gpt-4o`
*   `o1-preview`
*   `o1-mini`

## 4. Workflow: `resolve_issue`

1.  **Initialize:**
    *   Clone target repo to `/tmp/max-mcp-{uuid}`.
    *   Checkout branch `fix/issue-{n}`.
2.  **Reproduce (TDD):**
    *   Agent writes a reproduction test case that FAILS.
3.  **Iterate:**
    *   **Modify:** Agent edits source code.
    *   **Verify:** Run tests.
    *   **Loop:** If tests fail, feed error back to LLM and repeat (max 10x).
4.  **Finalize:**
    *   Commit changes.
    *   Push to remote.

## 5. Configuration (.env)

```ini
GITHUB_TOKEN=ghp_...
ANTHROPIC_API_KEY=sk-ant-...
# Optional: For GitHub Models/Copilot
OPENAI_API_KEY=... 
OPENAI_BASE_URL=https://models.inference.ai.azure.com
LLM_PROVIDER=anthropic
LLM_MODEL=claude-3-5-sonnet-20241022
```
