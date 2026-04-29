# Copilot Instructions

This file contains custom instructions for GitHub Copilot Coding Agent when working in this repository.
The sections below are adapted from the project's Claude slash commands.

---

## PR Creation (`/pr`)

Create a pull request from the current branch to the `main` branch.

### Arguments

The user may provide issue numbers that this PR fixes.

- If provided (e.g., `123` or `123,456`), use these issue numbers
- If not provided, check the branch name for issue numbers (e.g., `fix/123-bug` or `issue-456-feature` → extract `123` or `456`)
- If no issues are found, omit the "Closes" section

### Steps

First, analyze the current branch to understand what changes have been made:
1. Run `git log --oneline -10` to see recent commit history and understand commit style
2. Run `git log main..HEAD --oneline` to see all commits on this branch that will be included in the PR
3. Run `git diff main...HEAD --stat` to see a summary of file changes
4. Run `git branch --show-current` to get the branch name for issue detection (if no explicit issues provided)

Then generate a PR title that:
- Follows conventional commit format (e.g., `fix:`, `feat:`, `chore:`, `refactor:`)
- Is concise and accurately describes the changes
- Matches the style of recent commits in the repository

Generate a PR body with:
- A `## Summary` section using rich bullets with bold action leads
- A `Closes #X` line for each issue number (if any were provided or detected from branch name)

Each Summary bullet must follow this format:
- **Bold action phrase** (imperative verb: "Add X", "Replace Y", "Fix Z") — followed by em dash and a 1-2 sentence conceptual description of what changed and why
- Keep descriptions conceptual — no inline code references (no backticks for function/file names). The diff shows the code
- Use 2-5 bullets, scaling with PR size. Group related changes into single bullets rather than listing every file touched

Example:
```
## Summary

- **Add preflight validation** — validates repo path, config, and credentials before agent execution. Fails fast with actionable errors
- **Replace error strings** — pipe-delimited segments rendered as multi-line blocks with phase context, type, message, and remediation hint
- **Add error classification** — new error codes for repo, auth, and billing failures with proper retry classification
```

Finally, create the PR using the gh CLI:
```
gh pr create --base main --title "<generated title>" --body "$(cat <<'EOF'
## Summary
<rich bullets>

Closes #<issue1>
Closes #<issue2>
EOF
)"
```

Note: Omit the "Closes" lines entirely if no issues are associated with this PR.

IMPORTANT:
- Do NOT include any Claude Code attribution in the PR
- Use the conventional commit prefix that best matches the changes (fix, feat, chore, refactor, docs, etc.)
- The `Closes #X` syntax will automatically close the referenced issues when the PR is merged

---

## Code Review (`/review`)

Review the current changes (staged or working directory) with focus on Shannon-specific patterns and common mistakes.

### Step 1: Gather Changes
Run these commands to understand the scope:
```bash
git diff --stat HEAD
git diff HEAD
```

### Step 2: Check Shannon-Specific Patterns

#### Error Handling (CRITICAL)
- [ ] **All errors use PentestError** - Never use raw `Error`. Use `new PentestError(message, type, retryable, context)`
- [ ] **Error type is appropriate** - Use correct type: 'config', 'network', 'tool', 'prompt', 'filesystem', 'validation', 'billing', 'unknown'
- [ ] **Retryable flag matches behavior** - If error will be retried, set `retryable: true`
- [ ] **Context includes debugging info** - Add relevant paths, tool names, error codes to context object
- [ ] **Never swallow errors silently** - Always log or propagate errors
- [ ] **Use ErrorCode enum** - Prefer `ErrorCode.CONFIG_INVALID` over string matching for classification
- [ ] **Result<T,E> for service returns** - Services return `Result`, not throw

#### Audit System & Concurrency (CRITICAL)
- [ ] **Mutex protection for parallel operations** - Use `sessionMutex.lock()` when updating `session.json` during parallel agent execution
- [ ] **Reload before modify** - Always call `this.metricsTracker.reload()` before updating metrics in mutex block
- [ ] **Atomic writes for session.json** - Use `atomicWrite()` for session metadata, never `fs.writeFile()` directly
- [ ] **Stream drain handling** - Log writes must wait for buffer drain before resolving
- [ ] **Semaphore release in finally** - Git semaphore must be released in `finally` block

#### Claude SDK Integration (CRITICAL)
- [ ] **MCP server configuration** - Verify Playwright MCP uses `--isolated` and unique `--user-data-dir`
- [ ] **Prompt variable interpolation** - Check all `{{VARIABLE}}` placeholders are replaced
- [ ] **Turn counting** - Increment `turnCount` on assistant messages, not tool calls
- [ ] **Cost tracking** - Extract cost from final `result` message, track even on failure
- [ ] **API error detection** - Check for "session limit reached" (fatal) vs other errors

#### Configuration & Validation (CRITICAL)
- [ ] **FAILSAFE_SCHEMA for YAML** - Never use default schema (prevents code execution)
- [ ] **Security pattern detection** - Check for path traversal (`../`), HTML injection (`<>`), JavaScript URLs
- [ ] **Rule conflict detection** - Rules cannot appear in both `avoid` AND `focus`
- [ ] **Duplicate rule detection** - Same `type:url_path` cannot appear twice
- [ ] **JSON Schema validation before use** - Config must pass AJV validation

#### Services Layer & DI Container (CRITICAL)
- [ ] **Business logic in services, not activities** — Activities: heartbeat loop, error classification, container calls only. Domain logic → `src/services/`
- [ ] **Services accept ActivityLogger** — Never import `@temporalio/*` in services. Use `ActivityLogger` interface from `src/types/`
- [ ] **Result type for fallible operations** — Service methods return `Result<T, PentestError>`, unwrap with `isOk()`/`isErr()`. Activities call `executeOrThrow()` at the boundary
- [ ] **Container lifecycle** — `getOrCreateContainer()` at activity start, `removeContainer()` only in workflow cleanup
- [ ] **AuditSession not in container** — Must be passed per-agent call (parallel safety)

#### Session & Agent Management (CRITICAL)
- [ ] **Deliverable dependencies respected** - Exploitation agents only run if vulnerability queue exists AND has items
- [ ] **Queue validation before exploitation** - Use `safeValidateQueueAndDeliverable()` to check eligibility
- [ ] **Git checkpoint before agent run** - Create checkpoint for rollback on failure
- [ ] **Git rollback on retry** - Call `rollbackGitWorkspace()` before each retry attempt
- [ ] **Agent prerequisites checked** - Verify prerequisite agents completed before running dependent agent

#### Parallel Execution
- [ ] **Promise.allSettled for parallel agents** - Never use `Promise.all` (partial failures should not crash batch)
- [ ] **Staggered startup** - 2-second delay between parallel agent starts to prevent API throttle
- [ ] **Individual retry loops** - Each agent retries independently (3 attempts max)
- [ ] **Results aggregated correctly** - Handle both 'fulfilled' and 'rejected' results from `Promise.allSettled`

### Step 3: TypeScript Safety

#### Type Assertions (WARNING)
- [ ] **No double casting** - Never use `as unknown as SomeType` (bypasses type safety)
- [ ] **Validate before casting** - JSON parsed data should be validated (JSON Schema) before `as Type`
- [ ] **Prefer type guards** - Use `instanceof` or property checks instead of assertions where possible

#### Null/Undefined Handling
- [ ] **Explicit null checks** - Use `if (x === null || x === undefined)` not truthy checks for critical paths
- [ ] **Nullish coalescing** - Use `??` for null/undefined, not `||` which also catches empty string/0
- [ ] **Optional chaining** - Use `?.` for nested property access on potentially undefined objects

#### Imports & Types
- [ ] **Type imports** - Use `import type { ... }` for type-only imports
- [ ] **No implicit any** - All function parameters and returns must have explicit types
- [ ] **Readonly for constants** - Use `Object.freeze()` and `Readonly<>` for immutable data

### Step 4: Security Review

#### Defensive Tool Security
- [ ] **No credentials in logs** - Check that passwords, tokens, TOTP secrets are not logged to audit files
- [ ] **Config file size limit** - Ensure 1MB max for config files (DoS prevention)
- [ ] **Safe shell execution** - Command arguments must be escaped/sanitized

#### Code Injection Prevention
- [ ] **YAML safe parsing** - FAILSAFE_SCHEMA only
- [ ] **No eval/Function** - Never use dynamic code evaluation
- [ ] **Input validation at boundaries** - URLs, paths validated before use

### Step 5: Common Mistakes to Avoid

#### Anti-Patterns Found in Codebase
- [ ] **Catch + re-throw without context** - Don't just `throw error`, wrap with additional context
- [ ] **Silent failures in session loading** - Corrupted session files should warn user, not silently reset
- [ ] **Duplicate retry logic** - Don't implement retry at both caller and callee level
- [ ] **Hardcoded error message matching** - Prefer error codes over regex on error.message
- [ ] **Missing timeout on long operations** - Git operations and API calls should have timeouts
- [ ] **Console.log in services** — Use `ActivityLogger`. Only CLI display code (`client.ts`, `worker.ts`, `output-formatters.ts`) uses console.log
- [ ] **Temporal imports in services** — Services must stay Temporal-agnostic. If you need Temporal APIs, it belongs in activities

#### Code Quality
- [ ] **No dead code added** - Remove unused imports, functions, variables
- [ ] **No over-engineering** - Don't add abstractions for single-use operations
- [ ] **Comments only where needed** - Self-documenting code preferred over excessive comments
- [ ] **Consistent file naming** - kebab-case for files (e.g., `queue-validation.ts`)

### Step 6: Provide Feedback

For each issue found:
1. **Location**: File and line number
2. **Issue**: What's wrong and why it matters
3. **Fix**: How to correct it (with code example if helpful)
4. **Severity**: Critical / Warning / Suggestion

#### Severity Definitions
- **Critical**: Will cause bugs, crashes, data loss, or security issues
- **Warning**: Code smell, inconsistent pattern, or potential future issue
- **Suggestion**: Style improvement or minor enhancement

Summarize with:
- Total issues by severity
- Overall assessment (Ready to commit / Needs fixes / Needs discussion)

---

## Debugging (`/debug`)

Follow this structured approach to avoid spinning in circles when debugging an issue.

### Step 1: Capture Error Context
- Read the full error message and stack trace
- Identify the layer where the error originated:
  - **CLI/Args** - Input validation, path resolution
  - **Config Parsing** - YAML parsing, JSON Schema validation (`src/config-parser.ts`)
  - **Session Management** - Agent definitions (`src/session-manager.ts`), mutex (`src/utils/concurrency.ts`)
  - **DI Container** - Container initialization/lookup (`src/services/container.ts`)
  - **Services** - AgentExecutionService, ConfigLoaderService, ExploitationCheckerService, error-handling (`src/services/`)
  - **Audit System** - Logging, metrics tracking, atomic writes (`src/audit/`)
  - **Claude SDK** - Agent execution, MCP servers, turn handling (`src/ai/claude-executor.ts`)
  - **Git Operations** - Checkpoints, rollback, commit (`src/services/git-manager.ts`)
  - **Validation** - Deliverable checks, queue validation (`src/services/queue-validation.ts`)

### Step 2: Check Relevant Logs

**Session audit logs:**
```bash
# Find most recent session
ls -lt workspaces/ | head -5

# Check session metrics and errors
cat workspaces/<session>/session.json | jq '.errors, .agentMetrics'

# Check agent execution logs
ls -lt workspaces/<session>/agents/
cat workspaces/<session>/agents/<latest>.log
```

### Step 3: Trace the Call Path

For Shannon, trace through these layers:

1. **Worker + Client** → `src/temporal/worker.ts` - Combined worker + workflow submission
2. **Workflow** → `src/temporal/workflows.ts` - Pipeline orchestration
3. **Activities** → `src/temporal/activities.ts` - Thin wrappers: heartbeat, error classification
4. **Container** → `src/services/container.ts` - Per-workflow DI
5. **Services** → `src/services/agent-execution.ts` - Agent lifecycle
6. **Config** → `src/config-parser.ts` via `src/services/config-loader.ts`
7. **Prompts** → `src/services/prompt-manager.ts`
8. **Audit** → `src/audit/audit-session.ts` - Logging facade, metrics tracking
9. **Executor** → `src/ai/claude-executor.ts` - SDK calls, MCP setup, retry logic
10. **Validation** → `src/services/queue-validation.ts` - Deliverable checks

### Step 4: Identify Root Cause

**Common Shannon-specific issues:**

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Agent hangs indefinitely | MCP server crashed, Playwright timeout | Check Playwright logs in `/tmp/playwright-*` |
| "Validation failed: Missing deliverable" | Agent didn't create expected file | Check `deliverables/` dir, review prompt |
| Git checkpoint fails | Uncommitted changes, git lock | Run `git status`, remove `.git/index.lock` |
| "Session limit reached" | Claude API billing limit | Not retryable - check API usage |
| Parallel agents all fail | Shared resource contention | Check mutex usage, stagger startup timing |
| Cost/timing not tracked | Metrics not reloaded before update | Add `metricsTracker.reload()` before updates |
| session.json corrupted | Partial write during crash | Delete and restart, or restore from backup |
| YAML config rejected | Invalid schema or unsafe content | Run through AJV validator manually |
| Prompt variable not replaced | Missing `{{VARIABLE}}` in context | Check `src/services/prompt-manager.ts` interpolation |
| Service returns Err result | Check `ErrorCode` in Result | Trace through `classifyErrorForTemporal()` in `src/services/error-handling.ts` |
| Container not found | `getOrCreateContainer()` not called | Check activity setup code in `src/temporal/activities.ts` |
| ActivityLogger undefined | `createActivityLogger()` not called | Must be called at top of each activity function |

**MCP Server Issues:**
```bash
# Check if Playwright browsers are installed
npx playwright install chromium

# Check MCP server startup (look for connection errors)
grep -i "mcp\|playwright" workspaces/<session>/agents/*.log
```

**Git State Issues:**
```bash
# Check for uncommitted changes
git status

# Check for git locks
ls -la .git/*.lock

# View recent git operations from Shannon
git reflog | head -10
```

### Step 5: Apply Fix with Retry Limit

- **CRITICAL**: Track consecutive failed attempts
- After **3 consecutive failures** on the same issue, STOP and:
  - Summarize what was tried
  - Explain what's blocking progress
  - Ask the user for guidance or additional context
- After a successful fix, reset the failure counter

### Step 6: Validate the Fix

**For code changes:**
```bash
# Compile TypeScript
npx tsc --noEmit

# Quick validation run
shannon <URL> <REPO> --pipeline-testing
```

**For audit/session issues:**
- Verify `session.json` is valid JSON after fix
- Check that atomic writes complete without errors
- Confirm mutex release in `finally` blocks

**For agent issues:**
- Verify deliverable files are created in correct location
- Check that validation functions return expected results
- Confirm retry logic triggers on appropriate errors

### Anti-Patterns to Avoid

- Don't delete `session.json` without checking if session is active
- Don't modify git state while an agent is running
- Don't retry billing/quota errors (they're not retryable)
- Don't ignore PentestError type - it indicates the error category
- Don't make random changes hoping something works
- Don't fix symptoms without understanding root cause
- Don't bypass mutex protection for "quick fixes"

### Quick Reference: Error Types

`ErrorCode` enum in `src/types/errors.ts` provides finer-grained classification used by `classifyErrorForTemporal()` in `src/services/error-handling.ts`.

| PentestError Type | Meaning | Retryable? |
|-------------------|---------|------------|
| `config` | Configuration file issues | No |
| `network` | Connection/timeout issues | Yes |
| `tool` | External tool (nmap, etc.) failed | Yes |
| `prompt` | Claude SDK/API issues | Sometimes |
| `filesystem` | File read/write errors | Sometimes |
| `validation` | Deliverable validation failed | Yes (via retry) |
| `billing` | API quota/billing limit | No |
| `unknown` | Unexpected error | Depends |
