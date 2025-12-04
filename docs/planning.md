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

## 6. Agent Pipeline System

### Overview
The system uses a multi-agent pipeline architecture where each agent specializes in a specific task. All agents log their activities to a database for tracking, monitoring, and analysis.

### Agent Roles & Responsibilities

#### 1. **GitHub Issues Agent** (Pipeline Entry Point)
- **Purpose:** Pulls unresolved issues from configured GitHub repositories
- **Responsibilities:**
  - Connects to specified GitHub repos via API
  - Retrieves open issues based on criteria (labels, milestone, etc.)
  - Passes issue details to downstream agents
  - Logs all fetched issues to database

#### 2. **TDD Agent** (Test-Driven Development Specialist)
- **Purpose:** Writes comprehensive tests for the issue
- **Responsibilities:**
  - Analyzes issue requirements and acceptance criteria
  - Writes test cases that validate the solution
  - Can modify tests during the TDD/Expert loop
  - Cannot modify implementation code
  - Logs all generated tests to database

#### 3. **Expert AI Engineer Agent** (Implementation Specialist)
- **Purpose:** Implements clean, readable code to satisfy tests
- **Responsibilities:**
  - Writes production code based on TDD requirements
  - Focuses on code quality and readability
  - Cannot modify tests (only TDD agent can)
  - Works in iterative loop with TDD agent if tests fail
  - Logs all code changes and attempts to database

#### 4. **Supervision Agent** (Quality Gate)
- **Purpose:** Validates issue resolution completeness and correctness
- **Responsibilities:**
  - Verifies all tests pass
  - Reviews code changes for correctness
  - Validates issue requirements are met
  - Provides feedback for Expert agent refinement
  - Logs validation results and recommendations

#### 5. **UI Test Agent** (End-to-End Testing)
- **Purpose:** Tests UI functionality in headless browser mode
- **Responsibilities:**
  - Runs headless browser tests (Selenium, Playwright, etc.)
  - Validates UI interactions and rendering
  - Executes end-to-end test scenarios
  - Logs UI test results and screenshots/videos

#### 6. **Linux Guy Agent** (Infrastructure & CI/CD)
- **Purpose:** Troubleshoots CI/CD failures and maintains bash scripts
- **Responsibilities:**
  - Diagnoses CI/CD pipeline failures
  - Writes and maintains bash scripts for deployment
  - Handles environment setup and configuration
  - Logs scripts and troubleshooting steps to database
  - Provides fallback solutions if other agents encounter system-level issues

### Pipeline Flow

```
GitHub Issues Agent 
    ↓
TDD Agent (writes tests)
    ↓
Expert AI Engineer Agent
    ↓
Test Execution
    ├─ Tests Pass → Continue
    └─ Tests Fail → Loop back to Expert/TDD
    ↓
Supervision Agent (validates all aspects)
    ├─ Validation Passes → Continue
    └─ Validation Fails → Feedback to Expert/TDD
    ↓
UI Test Agent (if applicable)
    ├─ UI Tests Pass → Continue
    └─ UI Tests Fail → Feedback to Expert
    ↓
Linux Guy Agent (CI/CD verification)
    ├─ Pipeline OK → Ready to Deploy
    └─ Pipeline Error → Troubleshoot & Fix
    ↓
Deploy / Merge to Main
```

### TDD/Expert Loop

When tests fail or code doesn't meet requirements:
1. Expert AI Engineer implements/modifies code
2. Tests are run
3. If tests still fail:
   - TDD Agent analyzes failures and can improve tests
   - Expert Agent refines code based on test feedback
   - Loop repeats until all tests pass

## 7. Database Schema

### Core Tables

#### `agents`
- `id` (UUID, PK)
- `name` (VARCHAR) - e.g., "GitHub Issues Agent", "TDD Agent"
- `type` (VARCHAR) - Agent type/role
- `created_at` (TIMESTAMP)

#### `tasks`
- `id` (UUID, PK)
- `github_issue_id` (VARCHAR) - Reference to GitHub issue
- `github_repo` (VARCHAR) - Repository URL/name
- `status` (ENUM: pending, in_progress, completed, failed)
- `created_at` (TIMESTAMP)
- `started_at` (TIMESTAMP, nullable)
- `completed_at` (TIMESTAMP, nullable)
- `total_duration_ms` (INT, nullable)
- `timeout_duration_ms` (INT) - Configured timeout for task

#### `agent_logs`
- `id` (UUID, PK)
- `task_id` (UUID, FK) - References tasks
- `agent_id` (UUID, FK) - References agents
- `action` (VARCHAR) - What the agent did (e.g., "write_test", "write_code", "run_tests")
- `status` (ENUM: pending, in_progress, completed, failed)
- `input` (JSONB/TEXT) - Input to the agent
- `output` (JSONB/TEXT) - Agent output/result
- `error_message` (TEXT, nullable)
- `timestamp` (TIMESTAMP)
- `duration_ms` (INT, nullable)

#### `test_results`
- `id` (UUID, PK)
- `task_id` (UUID, FK)
- `agent_id` (UUID, FK) - Which agent ran the tests
- `total_tests` (INT)
- `passed` (INT)
- `failed` (INT)
- `skipped` (INT)
- `duration_ms` (INT)
- `output` (TEXT)
- `timestamp` (TIMESTAMP)

#### `code_changes`
- `id` (UUID, PK)
- `task_id` (UUID, FK)
- `agent_id` (UUID, FK) - Which agent made the change
- `file_path` (VARCHAR)
- `change_type` (ENUM: create, modify, delete)
- `diff` (TEXT)
- `commit_hash` (VARCHAR, nullable)
- `timestamp` (TIMESTAMP)

#### `metrics`
- `id` (UUID, PK)
- `task_id` (UUID, FK)
- `metric_name` (VARCHAR) - e.g., "test_pass_rate", "total_duration", "loop_iterations"
- `metric_value` (DECIMAL)
- `unit` (VARCHAR) - e.g., "percent", "ms", "count"
- `timestamp` (TIMESTAMP)

#### `bash_scripts`
- `id` (UUID, PK)
- `task_id` (UUID, FK, nullable) - Related task if any
- `agent_id` (UUID, FK) - Usually Linux Guy Agent
- `name` (VARCHAR)
- `script_content` (TEXT)
- `purpose` (TEXT)
- `version` (INT) - For tracking script evolution
- `created_at` (TIMESTAMP)
- `modified_at` (TIMESTAMP)

#### `notifications`
- `id` (UUID, PK)
- `task_id` (UUID, FK)
- `notification_type` (ENUM: timeout, failure, completion, warning)
- `recipient` (VARCHAR) - Email or user ID
- `message` (TEXT)
- `read` (BOOLEAN, default false)
- `created_at` (TIMESTAMP)

### Indexes
- `tasks(status, created_at)` - For querying active/pending tasks
- `agent_logs(task_id, agent_id, timestamp)` - For task-specific agent activity
- `test_results(task_id, timestamp)` - For test history
- `metrics(task_id, metric_name)` - For metrics queries

## 8. Dashboard Features

### Real-time Monitoring
- Current agent status and active tasks
- Pipeline progress visualization
- Live logs of each agent's actions
- Test results in real-time

### Historical Analytics
- Test results over time (pass rate trend)
- Issue-to-completion duration trends
- Agent performance metrics
- Failure rate analysis

### Task Management
- View all GitHub issues and their status
- Filter by repository, status, or date range
- Detailed task history and logs
- Agent activity timeline

### Metrics & Reports
- Test pass rates over time
- Average time per issue
- Failure reasons and patterns
- Agent performance leaderboard
- CI/CD success rates

## 9. Timeout & Auto-Fail Mechanism

### Configuration
- Configurable timeout per task type (default: 1 hour)
- Warning notification at 80% timeout
- Auto-fail at timeout with detailed error report

### Failure Handling
1. Task reaches timeout threshold
2. System logs failure reason and context
3. Notification sent to configured human (email/Slack)
4. Task marked as failed in database
5. Pipeline halts gracefully
6. Human can review logs and restart or modify

### Human Notification
- Email with task summary, logs, and error details
- Links to dashboard for detailed inspection
- Option to restart or escalate task
- Slack webhook support for real-time alerts

## 10. Technology Stack Updates

### Database
- **PostgreSQL 15+** (Primary - structured data, good for analytics)
- **Redis** (Optional - for caching, task queues, session management)

### Backend
- **Python FastAPI** - REST API for dashboard and agent communication
- **SQLAlchemy ORM** - Database abstraction
- **APScheduler** - Task scheduling and timeout management
- **Pydantic** - Data validation

### Dashboard Frontend
- **React/Vue.js** - Real-time UI updates
- **WebSocket** - Live agent status streaming
- **Chart.js/Recharts** - Analytics visualization
- **Tailwind CSS** - Styling

### Deployment
- **Docker** - Containerization for agents and API
- **Kubernetes** (Optional) - Orchestration for scaling
- **GitHub Actions** - CI/CD for max-mcp itself

## 11. Implementation Phases

### Phase 1: Core Infrastructure
- Database schema setup
- Agent logging system
- Basic task management
- MCP server integration

### Phase 2: Agents
- GitHub Issues Agent
- TDD Agent
- Expert AI Engineer Agent (basic loop)

### Phase 3: Quality & Monitoring
- Supervision Agent
- Test result tracking
- Dashboard MVP

### Phase 4: Advanced Agents & Features
- UI Test Agent
- Linux Guy Agent
- Timeout/auto-fail mechanism
- Advanced analytics

### Phase 5: Optimization & Scaling
- Performance tuning
- Distributed agent execution
- Advanced reporting
