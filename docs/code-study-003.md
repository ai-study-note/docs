# Code Agent Implementation Analysis & Cross-Project Implementation Prompt

> This document provides a thorough analysis of `packages/<package-name>/src/main/code-agent` covering features,
> workflow, and implementation, followed by a reusable prompt for implementing the same Code Agent system
> in other projects. The analysis incorporates the design logic from
> `packages/<package-name>/docs/code-agent-workflow-design.md`.

---

## Table of Contents

- [1. Features Analysis](#1-features-analysis)
  - [1.1 Module Overview](#11-module-overview)
  - [1.2 Core Feature List](#12-core-feature-list)
  - [1.3 Detailed Feature Analysis](#13-detailed-feature-analysis)
- [2. Workflow Analysis](#2-workflow-analysis)
  - [2.1 Complete Agent Workflow](#21-complete-agent-workflow)
  - [2.2 IPC Message Flow](#22-ipc-message-flow)
  - [2.3 Cancellation Flow](#23-cancellation-flow)
- [3. Implementation Analysis](#3-implementation-analysis)
  - [3.1 File Structure](#31-file-structure)
  - [3.2 Class Dependency Diagram](#32-class-dependency-diagram)
  - [3.3 Key Design Patterns](#33-key-design-patterns)
  - [3.4 Dependencies](#34-dependencies)
  - [3.5 IPC Integration](#35-ipc-integration)
- [4. Cross-Project Implementation Prompt](#4-cross-project-implementation-prompt)

---

## 1. Features Analysis

### 1.1 Module Overview

`src/main/code-agent` is an **Autonomous Coding Agent** built on **OpenAI Tool Calling**, running in the
Electron main process and communicating with the renderer process via IPC. The module consists of
**10 source files**, organized into 6 core classes and 2 data modules.

### 1.2 Core Feature List

| # | Feature | File | Description |
|---|---------|------|-------------|
| F1 | **Agent Loop** | `agent-loop.ts` | Multi-turn Tool Calling conversation loop with streaming output, parallel tool execution, and cancellation |
| F2 | **Tool Registry** | `tool-registry.ts` | Manages all Agent-available tools (built-in + MCP), supports dynamic registration |
| F3 | **Built-in Tools** | `tools/index.ts` | 7 built-in tools: read_file, write_file, edit_file, glob_files, grep_search, run_command, list_directory |
| F4 | **Git Diff Tracker** | `diff-tracker.ts` | Tracks file changes via simple-git, generates unified diffs, supports full revert |
| F5 | **MCP Manager** | `mcp-manager.ts` | Dynamically loads MCP servers (stdio transport), auto-registers their tools |
| F6 | **Skill Manager** | `skill-manager.ts` | Manages System Prompt extensions (Skills) that inject coding conventions |
| F7 | **Built-in Skills** | `skills/index.ts` | 3 built-in skills: typescript-strict, react-patterns, testing |
| F8 | **Sandbox Validator** | `sandbox.ts` | Auto-detects project type, runs validation commands (tsc --noEmit, lint, test) |
| F9 | **Cancellation** | `agent-loop.ts` | Supports user-initiated abort via AbortSignal during agent execution |
| F10 | **Async Validation** | `agent-loop.ts` | Supports skipping synchronous validation, with async validation via `validate()` |

### 1.3 Detailed Feature Analysis

#### F1: Agent Loop

**File:** `agent-loop.ts`, Class: `CodeAgentLoop`

This is the core of the entire system, implementing the complete agent workflow.

**Constructor initialization:**

```
constructor(config: AgentConfig)
  1. Create OpenAI client
     - apiKey: config.apiKey
     - baseURL: config.baseURL || undefined
     - timeout: config.timeout || 120000
     - maxRetries: config.maxRetries || 0
     - dangerouslyAllowBrowser: false
  2. Create ToolRegistry(config.projectDir) — all tool operations scoped to project directory
  3. Create GitDiffTracker(config.projectDir) — track file changes
  4. Create SandboxExecutor() — code validation
  5. Create SkillManager(this.toolRegistry) — skill management
  6. Register built-in skills (iterate over builtinSkills array)
```

**`run()` method execution flow (6 phases):**

```
run(prompt, callbacks, options?)
  │
  ├─ Phase 1: Initialize Git tracking
  │   await this.diffTracker.initialize()
  │
  ├─ Phase 2: Build System Prompt
  │   const systemPrompt = this.buildSystemPrompt()
  │   (includes project context + Skill extensions)
  │
  ├─ Phase 3: Build message list
  │   messages = [
  │     { role: 'system', content: systemPrompt },
  │     { role: 'user', content: prompt }
  │   ]
  │
  ├─ Phase 4: Agent Loop (multi-turn)
  │   while (true) {
  │     ├─ Checkpoint 1: signal?.aborted → abort
  │     ├─ Call LLM (stream: true, tools: tool list)
  │     ├─ Iterate streaming response chunks:
  │     │   ├─ reasoning_content → callbacks.onThinking()
  │     │   ├─ delta.content → callbacks.onContent()
  │     │   └─ delta.tool_calls → collect into Map<index, {id, name, args}>
  │     ├─ Checkpoint 2: AbortError → abort
  │     ├─ Checkpoint 3: signal?.aborted → abort
  │     ├─ If toolCalls.size === 0 → break (loop ends)
  │     ├─ Append assistant message (with tool_calls) to messages
  │     ├─ Send onToolUse({ status: 'start' }) for all tools
  │     ├─ Execute all tools in parallel (Promise.all)
  │     │   Check signal?.aborted before each tool execution
  │     ├─ Sort results by original index
  │     ├─ Process each tool result:
  │     │   ├─ Success → record ToolCallRecord, track file changes, send done event
  │     │   └─ Failure → record error, send error event
  │     ├─ Append tool role messages to messages
  │     └─ Check signal?.aborted → abort
  │   }
  │
  ├─ Phase 5: Sandbox validation (if skipValidation is false)
  │   validation = await this.sandbox.validateChanges(...)
  │
  ├─ Phase 6: Collect diffs
  │   diffs = await this.diffTracker.collectDiffs()
  │
  └─ Return AgentResult { thinking, response, diffs, validation, toolCalls, aborted }
```

**Cancellation mechanism:**

- Supports `AbortSignal` passed into `run()` method
- Three checkpoints: before LLM call, during streaming (AbortError), before tool execution
- Returns partial results with `aborted: true` flag on cancellation
- `AbortController` is created and managed externally (`code-handlers.ts`)

**Key design decisions:**
- Tool calls execute **in parallel** (`Promise.all`), significantly improving execution speed
- Tool results sorted by original index before processing, ensuring consistent message order
- `reasoning_content` field supports DeepSeek R1, Claude extended thinking, etc.

#### F2: Tool Registry

**File:** `tool-registry.ts`, Class: `ToolRegistry`

```
constructor(projectDir: string)
  - Calls createBuiltinTools(projectDir) to create built-in tools
  - Iterates and registers each built-in tool

register(definition, handler)
  - Stores tool in Map<name, { definition, handler }>

registerMCPTools(mcpTools)
  - Adds mcp__ prefix to each MCP tool name
  - Avoids naming conflicts with built-in tools

getOpenAIToolDefinitions(): ToolDefinition[]
  - Returns all tool definitions for OpenAI API calls

execute(name, args): Promise<string>
  - Looks up tool by name and executes handler
  - Throws Error if tool not found
```

**Key design:** Constructor receives `projectDir` parameter, passed to `createBuiltinTools()`, ensuring all file operations are scoped to the project directory.

#### F3: Built-in Tools

**File:** `tools/index.ts`, Function: `createBuiltinTools(projectDir)`

Factory function pattern, receives `projectDir` parameter, returns 7 tool definitions and handlers.

**Path resolution helper:**

```typescript
function resolvePath(projectDir: string, filePath: string): string {
  if (path.isAbsolute(filePath)) return filePath;
  return path.resolve(projectDir, filePath);
}
```

**7 built-in tools detail table:**

| Tool | Function | Required Params | Optional Params | Safety Limits |
|------|----------|----------------|-----------------|---------------|
| `read_file` | Read file contents (with line numbers) | filePath | startLine, endLine | Scoped to projectDir |
| `write_file` | Create or overwrite file | filePath, content | — | Scoped to projectDir, auto-creates parent dirs |
| `edit_file` | Exact string replacement editing | filePath, oldString, newString | — | oldString must be unique match, errors otherwise |
| `glob_files` | File glob search | pattern | cwd | Ignores node_modules, dist, .git, build |
| `grep_search` | Content regex search | pattern | include, searchPath | 30s timeout, 10MB buffer, grep exit code 1 = no matches |
| `run_command` | Execute shell command | command | cwd | 60s timeout, 10MB buffer, output truncated to 5000 chars |
| `list_directory` | List directory contents | dirPath | recursive | Max recursive depth 3, ignores node_modules etc. |

**Tool design highlights:**
- `edit_file` has two-layer validation: oldString must exist AND must be unique
- `run_command` returns JSON-formatted `{ stdout, stderr }`, each truncated to 5000 chars
- `grep_search` correctly distinguishes "no matches" (grep exit code 1) from "execution error" (other exit codes)
- All tools resolve paths automatically via the `rp()` closure function

#### F4: Git Diff Tracker

**File:** `diff-tracker.ts`, Class: `GitDiffTracker`

Based on `simple-git`, core methods:

```
initialize()
  - Records current Git HEAD hash
  - Sets to empty string if no Git repository

trackFileChange(filePath)
  - Converts absolute paths to relative (relative to projectDir)
  - Stores in changedFiles Set

getChangedFiles(): string[]
  - Returns list of changed files (relative paths)

collectDiffs(): Promise<FileDiff[]>
  - For each changed file:
    1. Determine action via git status (added/modified/deleted)
    2. Get unified diff text via git diff
    3. Count additions (lines starting with + but not +++) and deletions (lines starting with - but not ---)
  - Error handling: individual file failures don't affect others

revertAll()
  - New files → fs.unlink delete
  - Existing files → git checkout restore
  - Clears changedFiles after completion

finalize()
  - Only clears changedFiles set
  - Does NOT perform git add or git commit
  - Files are already persisted to disk by tool handlers
```

**Key design decision (per project conventions):**
- `finalize()` does NOT perform `git add`/`git commit`, only clears the tracking set
- File changes are already persisted to disk by tool handlers, no additional git operations needed

#### F5: MCP Manager

**File:** `mcp-manager.ts`, Class: `MCPManager`

Based on `@modelcontextprotocol/sdk`, loads external MCP servers via stdio transport protocol.

```
loadServer(config: MCPServerConfig)
  1. Create StdioClientTransport(command, args, env)
  2. Create Client({ name: 'code-agent', version: '1.0.0' })
  3. client.connect(transport)
  4. client.listTools() to get tool list
  5. Convert each tool to OpenAI Function Calling format:
     - Name with mcp__ prefix
     - Description preserved from original
     - parameters from inputSchema
  6. Register to ToolRegistry (via registerMCPTools)
  7. Store in servers Map

unloadServer(name)
  - client.close() to close connection
  - Remove from servers Map

unloadAll()
  - Iterate and unload all servers

getLoadedServers(): string[]
  - Return list of loaded server names
```

**MCP vs Skill distinction:**
- MCP provides **external tools** (database, browser, API, etc.)
- Skills provide **code modification patterns** (coding conventions, component generation patterns, etc.)

#### F6: Skill Manager

**File:** `skill-manager.ts`, Class: `SkillManager`

```
registerSkill(skill: SkillDefinition)
  - Store in skills Map
  - If skill has custom tools, register to ToolRegistry (placeholder handler)

unregisterSkill(name)
  - Remove from skills Map

getSystemPromptExtension(): string
  - Iterate all Skills, merge systemPromptExtension
  - Format:
    ## Active Skills
    ### {name}: {description}
    {systemPromptExtension}
  - Returns empty string if no Skills

getSkills(): Array<{ name, description }>
  - Return list of registered Skills
```

#### F7: Built-in Skills

**File:** `skills/index.ts`, Export: `builtinSkills`

3 built-in skills, all extending the System Prompt to guide Agent behavior:

| Skill Name | Description | Injected Rules |
|-----------|-------------|---------------|
| `typescript-strict` | Enforce strict TypeScript patterns | Explicit types, use interface, forbid any, use readonly, const assertions, discriminated unions |
| `react-patterns` | Follow React best practices | Functional components + hooks, co-located styles, React.memo(), custom hooks, useCallback/useMemo, accessibility |
| `testing` | Generate tests alongside code changes | Test coverage, update existing tests, edge cases, describe/it blocks |

#### F8: Sandbox Validator

**File:** `sandbox.ts`, Class: `SandboxExecutor`

Runs validation commands directly in the project directory (not an isolated sandbox), auto-detects project type:

```
validateChanges(projectDir, changedFiles)
  1. Check if package.json exists
  2. Read scripts field
  3. Determine validation commands based on changed file types:
     - .ts/.tsx file changes + tsconfig.json exists → npm run type-check or npx tsc --noEmit
     - lint script exists → npm run lint
     - test script exists → npm run test -- --passWithNoTests
  4. No validation commands → return passed: true (skip validation)
  5. Execute validation commands sequentially (120s timeout, 10MB buffer)
  6. Collect stdout, stderr, exitCode for each command
  7. Return ValidationResult { passed, commands, outputs, errors }
```

#### F9: Cancellation

Implemented via the standard Web API `AbortSignal`, integrated in `code-handlers.ts`:

```
Flow in code-handlers.ts:
  1. Create AbortController when code:agent-start triggers
  2. Pass AbortController.signal into agent.run() options
  3. Call abortController.abort() when code:agent-cancel triggers
  4. Three checkpoints verify abort:
     - Before LLM call (signal?.aborted)
     - During streaming (AbortError exception catch)
     - Before tool execution (signal?.aborted)
  5. Return AgentResult { aborted: true } on cancellation
  6. Send code:agent-aborted event to frontend
```

#### F10: Async Validation

`run()` method supports `skipValidation: true` option:

- Skips synchronous validation, returns results immediately (validation is `{ passed: true, commands: [], outputs: [], errors: [] }`)
- Frontend displays Diff results first, validation runs in background
- `validate()` method triggers async validation manually
- Validation results pushed to frontend via `code:agent-validation` IPC event
- In `code-handlers.ts`, agent sends `code:agent-done` immediately after completion, then asynchronously executes `agent.validate()` and pushes results

---

## 2. Workflow Analysis

### 2.1 Complete Agent Workflow

Per the design in `code-agent-workflow-design.md`, the actual implementation follows this workflow:

```
User inputs Prompt
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  Phase 1: Initialization                                         │
│  - GitDiffTracker.initialize() records current Git state         │
│  - buildSystemPrompt() builds system prompt (with Skill ext.)    │
│  - Build initial message list [system, user]                     │
└──────────────────────────┬──────────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  Phase 2: Agent Loop (Multi-turn Tool Calling)                   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 2a. Call LLM (stream: true, tools: registered tool list)  │  │
│  │     - Stream thinking → code:agent-thinking              │  │
│  │     - Stream content → code:agent-response               │  │
│  │     - Collect tool_calls → Map<index, {id, name, args}>   │  │
│  └──────────────────────┬───────────────────────────────────┘  │
│                         ▼                                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 2b. Check: tool_calls.size === 0 ?                        │  │
│  │     - Yes → loop ends, proceed to Phase 3                 │  │
│  │     - No → continue execution                             │  │
│  └──────────────────────┬───────────────────────────────────┘  │
│                         ▼                                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 2c. Send onToolUse({ status: 'start' }) for all tools     │  │
│  │     Execute all tools in parallel (Promise.all)            │  │
│  │     - Success → record ToolCallRecord, track file changes │  │
│  │     - Failure → record error message                      │  │
│  │     Send onToolUse({ status: 'done'/'error' })            │  │
│  └──────────────────────┬───────────────────────────────────┘  │
│                         ▼                                       │
│  │ 2d. Append assistant message (with tool_calls) and tool      │  │
│  │     results to messages list                                 │  │
│  │ 2e. Return to 2a for next turn                               │  │
└─────────────────────────────────────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  Phase 3: Sandbox Validation (if skipValidation is false)        │
│  - Auto-detect project type (TypeScript / ESLint / Jest)         │
│  - Run validation commands                                       │
│  - Send code:agent-validation event                             │
└──────────────────────────┬──────────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  Phase 4: Diff Collection & Result Return                        │
│  - GitDiffTracker.collectDiffs() collects all file diffs         │
│  - Send code:agent-diff event                                   │
│  - Send code:agent-done event (with full results)               │
│  - Async execute agent.validate() and push results (if skipValidation) │
└──────────────────────────┬──────────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  Phase 5: User Decision                                          │
│  - Accept All → code:agent-apply { action: 'accept' }           │
│    → agent.applyAll() → diffTracker.finalize() (clear tracking)  │
│  - Reject All → code:agent-apply { action: 'reject' }           │
│    → agent.revertAll() → diffTracker.revertAll() (git checkout)  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 IPC Message Flow

**IPC channel definitions (from `preload.ts` and `code-handlers.ts`):**

**Renderer → Main (send):**

| Channel | Payload | Trigger |
|---------|---------|---------|
| `code:agent-start` | `{ modelConfigId, prompt, projectDir, sessionId }` | User sends message |
| `code:agent-apply` | `{ sessionId, action: 'accept' \| 'reject' }` | User chooses accept/reject |
| `code:agent-cancel` | `{ sessionId }` | User clicks cancel |

**Main → Renderer (event.sender.send):**

| Channel | Payload | Trigger |
|---------|---------|---------|
| `code:agent-thinking` | `{ content: string }` | Stream thinking chain (multiple) |
| `code:agent-response` | `{ content: string }` | Stream response content (multiple) |
| `code:agent-tool-use` | `{ toolName, args, status, result?, error? }` | Tool call status changes |
| `code:agent-diff` | `{ files: FileDiff[] }` | After agent completes |
| `code:agent-validation` | `{ result: ValidationResult }` | After validation completes (may be async) |
| `code:agent-done` | `{ summary, thinking, response, diffs, validation, toolCalls, aborted? }` | After agent completes |
| `code:agent-error` | `{ error: string, phase: string }` | On execution error |
| `code:agent-aborted` | `{ thinking, response, toolCalls }` | When cancelled by user |

**Message flow sequence:**

```
Renderer                              Main Process
   │                                       │
   │── code:agent-start ──────────────────▶│  1. Start Agent
   │                                       │     - Create CodeAgentLoop
   │                                       │     - Create AbortController
   │                                       │     - Call agent.run()
   │                                       │
   │◀─ code:agent-thinking ───────────────│  2. Stream thinking (multiple)
   │◀─ code:agent-response ───────────────│  3. Stream response (multiple)
   │◀─ code:agent-tool-use ───────────────│  4. Tool call status (multiple per tool)
   │                                       │
   │── code:agent-cancel ─────────────────▶│  5. User cancels (optional)
   │◀─ code:agent-aborted ────────────────│  6. Abort confirmation
   │                                       │
   │◀─ code:agent-diff ───────────────────│  7. Diff results
   │◀─ code:agent-done ───────────────────│  8. Agent complete
   │                                       │     - Async execute validate()
   │◀─ code:agent-validation ─────────────│  9. Async validation result
   │                                       │
   │── code:agent-apply ──────────────────▶│ 10. User decision
   │                                       │     - accept → finalize()
   │                                       │     - reject → revertAll()
```

### 2.3 Cancellation Flow

```
User clicks cancel button
    │
    ▼
Renderer: window.electronAPI.send('code:agent-cancel', { sessionId })
    │
    ▼
Main Process: code-handlers.ts
    │
    ├─ activeAbortControllers.get(sessionId) → abortController
    ├─ abortController.abort()
    │
    ▼
Agent Loop (agent-loop.ts) three checkpoints:
    │
    ├─ Checkpoint 1: Before next LLM call
    │   if (signal?.aborted) { aborted = true; break; }
    │
    ├─ Checkpoint 2: During streaming
    │   catch (AbortError) { aborted = true; break; }
    │
    └─ Checkpoint 3: Before each tool execution
        if (signal?.aborted) { return { ...error: 'Aborted by user' }; }
    │
    ▼
Return AgentResult { aborted: true, thinking, response, toolCalls }
    │
    ▼
Main Process: Send code:agent-aborted event to Renderer
```

---

## 3. Implementation Analysis

### 3.1 File Structure

```
src/main/code-agent/
├── index.ts              # Module entry, re-exports all public classes and types
├── agent-loop.ts         # Core agent loop (CodeAgentLoop class)
├── tool-registry.ts      # Tool registry (ToolRegistry class)
├── tools/
│   └── index.ts          # Built-in tools factory (createBuiltinTools)
├── diff-tracker.ts       # Git diff tracker (GitDiffTracker class)
├── mcp-manager.ts        # MCP manager (MCPManager class)
├── skill-manager.ts      # Skill manager (SkillManager class)
├── skills/
│   └── index.ts          # Built-in skill definitions (builtinSkills)
├── sandbox.ts            # Sandbox validator (SandboxExecutor class)
└── types.ts              # All type/interface definitions (15 types)
```

### 3.2 Class Dependency Diagram

```
CodeAgentLoop (agent-loop.ts)
├── depends on → OpenAI (openai npm SDK)
├── depends on → ToolRegistry (tool-registry.ts)
│   └── depends on → createBuiltinTools(projectDir) (tools/index.ts)
│       ├── depends on → fs/promises, path, child_process, util, glob
│       └── produces → 7 built-in tools
├── depends on → GitDiffTracker (diff-tracker.ts)
│   └── depends on → simple-git, fs/promises, path
├── depends on → SandboxExecutor (sandbox.ts)
│   └── depends on → fs/promises, path, child_process
├── depends on → SkillManager (skill-manager.ts)
│   ├── depends on → ToolRegistry (same instance)
│   └── depends on → builtinSkills (skills/index.ts)
│       └── depends on → types.ts (SkillDefinition)
└── externally injected → MCPManager (mcp-manager.ts) [via getToolRegistry()]
    └── depends on → @modelcontextprotocol/sdk
```

### 3.3 Key Design Patterns

#### Pattern 1: Factory Function Pattern

Tools are created using the `createBuiltinTools(projectDir)` factory function:

```typescript
// tools/index.ts
export function createBuiltinTools(projectDir: string) {
  const rp = (filePath: string) => resolvePath(projectDir, filePath);
  return [
    { definition: {...}, handler: async (args) => { const fullPath = rp(args.filePath); ... } },
    // ... 7 tools
  ];
}
```

**Benefit:** All file operations are automatically scoped to the project directory via the `rp()` closure,
preventing the Agent from accessing files outside the project.

#### Pattern 2: Callback Pattern

The Agent communicates externally via the `AgentCallbacks` interface, decoupling core logic from the communication layer:

```typescript
interface AgentCallbacks {
  onThinking: (content: string) => void;
  onContent: (content: string) => void;
  onToolUse: (event: ToolUseEvent) => void;
}
```

Mapped to IPC events in `code-handlers.ts`:

```typescript
agent.run(prompt, {
  onThinking: (content) => event.sender.send('code:agent-thinking', { content }),
  onContent: (content) => event.sender.send('code:agent-response', { content }),
  onToolUse: (event) => event.sender.send('code:agent-tool-use', event),
}, { signal: abortController.signal, skipValidation: true });
```

#### Pattern 3: Parallel Tool Execution

Tools are executed in parallel using `Promise.all`:

```typescript
const toolResults = await Promise.all(
  Array.from(toolCalls.entries()).map(async ([idx, tc]) => {
    if (signal?.aborted) return { ...error: 'Aborted by user' };
    try {
      const result = await this.toolRegistry.execute(tc.name, parsedArgs);
      return { idx, tc, result, success: true };
    } catch (error) {
      return { idx, tc, error: error.message, success: false };
    }
  })
);
// Sort by original index to ensure consistent message order
toolResults.sort((a, b) => a.idx - b.idx);
```

#### Pattern 4: AbortSignal Cancellation Pattern

Cancellation uses the standard Web API `AbortSignal`, natively compatible with the OpenAI SDK.

### 3.4 Dependencies

| Dependency | Version | Purpose | License |
|-----------|---------|---------|---------|
| `openai` | ^7.3.0 | LLM API calls (Tool Calling + Streaming) | Apache 2.0 |
| `@modelcontextprotocol/sdk` | ^1.21.0 | MCP protocol implementation (stdio transport) | MIT |
| `simple-git` | ^3.27.0 | Git operations (diff, status, checkout, revparse) | MIT |
| `glob` | ^11.0.0 | File glob search | ISC |

### 3.5 IPC Integration

**Integration in `code-handlers.ts`:**

```
1. Import CodeAgentLoop
2. Create two Maps for lifecycle management:
   - activeAgents: Map<string, CodeAgentLoop>     // sessionId → Agent instance
   - activeAbortControllers: Map<string, AbortController>  // sessionId → abort controller
3. Register three IPC handlers:
   a. code:agent-start
      - Read model config from database
      - Create CodeAgentLoop instance
      - Create AbortController
      - Call agent.run(prompt, callbacks, { signal, skipValidation: true })
      - Send code:agent-diff, code:agent-done
      - Async execute agent.validate() and push code:agent-validation
      - Error handling → code:agent-error
   b. code:agent-cancel
      - Get abortController and call abort()
   c. code:agent-apply
      - accept → agent.applyAll()
      - reject → agent.revertAll()
      - Clean up activeAgents
```

**Integration in `preload.ts`:**

```
validSendChannels additions:
  'code:agent-start', 'code:agent-apply', 'code:agent-cancel'

validOnChannels additions:
  'code:agent-thinking', 'code:agent-response', 'code:agent-tool-use',
  'code:agent-diff', 'code:agent-validation', 'code:agent-done',
  'code:agent-error', 'code:agent-aborted'
```

---

## 4. Cross-Project Implementation Prompt

> **Usage:** The following prompt can be copied directly to an AI coding assistant for implementing the
> same Code Agent system in other projects. Adjust according to the target project's tech stack and architecture.

---

### Task Objective

Implement a Code Agent system based on **OpenAI Tool Calling** in the target project.
This system allows users to interact with AI through natural language to read, modify, and manage project code,
supporting full thinking chain display, tool calling, code diff tracking, sandbox validation,
MCP extension, and Skill system.

---

### Tech Stack Requirements

- **Runtime:** Node.js (Electron main process or other Node.js environment)
- **LLM SDK:** `openai` ^7.3.0 (supports Tool Calling + Streaming)
- **Git Operations:** `simple-git` ^3.27.0
- **File Search:** `glob` ^11.0.0
- **MCP Protocol (optional):** `@modelcontextprotocol/sdk` ^1.21.0

---

### Step 1: Install Dependencies

```bash
npm install openai simple-git glob
# Optional: MCP support
npm install @modelcontextprotocol/sdk
```

---

### Step 2: Create Type Definitions

Create `types.ts` with the following 11 interfaces/types.

**Interface descriptions:**

1. **AgentConfig** — Agent configuration
   - `apiKey: string` — API key
   - `baseURL?: string` — Optional API Base URL
   - `model: string` — Model name
   - `projectDir: string` — Project directory (absolute path)
   - `timeout?: number` — Request timeout (default 120000ms)
   - `maxRetries?: number` — Maximum retry attempts (default 0)

2. **AgentCallbacks** — Streaming callback interface
   - `onThinking: (content: string) => void` — Thinking chain callback
   - `onContent: (content: string) => void` — Response content callback
   - `onToolUse: (event: ToolUseEvent) => void` — Tool call status callback

3. **ToolUseEvent** — Tool call event
   - `toolName: string` — Tool name
   - `args: string` — JSON-serialized arguments
   - `status: 'start' | 'done' | 'error'` — Execution status
   - `result?: string` — Execution result (when status is 'done')
   - `error?: string` — Error message (when status is 'error')

4. **ToolCallRecord** — Tool call record
   - `toolName: string`, `args: string`, `result: string`, `timestamp: number`, `success: boolean`

5. **WorkflowStep** — Workflow step
   - `index: number` — Step number
   - `action: 'create' | 'edit' | 'delete' | 'run_command'` — Operation type
   - `filePath: string` — File path (relative)
   - `description: string` — Description

6. **FileDiff** — File diff
   - `filePath: string` — File path (relative)
   - `action: 'added' | 'modified' | 'deleted'` — Change type
   - `diff: string` — Unified diff text
   - `additions: number` — Lines added
   - `deletions: number` — Lines deleted

7. **ValidationResult** — Sandbox validation result
   - `passed: boolean` — Whether validation passed
   - `commands: string[]` — Commands executed
   - `outputs: Array<{ command, stdout, stderr, exitCode }>` — Per-command output
   - `errors: string[]` — Error messages

8. **AgentResult** — Agent execution result
   - `thinking: string` — Thinking chain content
   - `response: string` — Response content
   - `diffs: FileDiff[]` — File diff list
   - `validation: ValidationResult` — Validation result
   - `toolCalls: ToolCallRecord[]` — Tool call history
   - `aborted?: boolean` — Whether aborted

9. **MCPServerConfig** — MCP server configuration
   - `name: string` — Unique name
   - `command: string` — Launch command
   - `args?: string[]` — Command arguments
   - `env?: Record<string, string>` — Environment variables
   - `autoApprove?: boolean` — Whether to auto-approve

10. **SkillDefinition** — Skill definition
    - `name: string` — Name
    - `description: string` — Description
    - `systemPromptExtension: string` — System Prompt extension content
    - `tools?: ToolDefinition[]` — Optional custom tools

11. **ToolDefinition** — OpenAI Function Calling tool definition
    - `type: 'function'`
    - `function: { name: string, description: string, parameters: Record<string, unknown> }`

---

### Step 3: Implement Built-in Tools

Create `tools/index.ts`, implementing the `createBuiltinTools(projectDir: string)` factory function.

First, implement the path resolution helper:

```typescript
function resolvePath(projectDir: string, filePath: string): string {
  if (path.isAbsolute(filePath)) return filePath;
  return path.resolve(projectDir, filePath);
}
```

Then implement 7 built-in tools, each with an OpenAI Function Calling format `definition` and a Node.js `handler`.

**Tool 1: read_file**

- Function: Read file contents, returns content with line numbers
- Parameters: `filePath: string` (required), `startLine?: number`, `endLine?: number`
- Implementation: Use `fs.readFile`, support line range filtering
- Path handling: Use `rp(filePath)` to convert relative to absolute

**Tool 2: write_file**

- Function: Create new file or overwrite existing file
- Parameters: `filePath: string` (required), `content: string` (required)
- Implementation: Auto-create parent directory (`fs.mkdir(dir, { recursive: true })`), then write file
- Returns: Success message with file path and line count

**Tool 3: edit_file**

- Function: Exact string replacement editing in file
- Parameters: `filePath: string` (required), `oldString: string` (required), `newString: string` (required)
- Implementation:
  1. Read file content
  2. Check oldString exists (error if not found)
  3. Check oldString is unique (error if appears multiple times, request more context)
  4. Perform replacement and write file
- Returns: Success message

**Tool 4: glob_files**

- Function: Search files using glob patterns
- Parameters: `pattern: string` (required), `cwd?: string`
- Implementation: Use `glob` library, auto-ignore `node_modules`, `dist`, `.git`, `build`
- Returns: Matching file paths list (newline-separated), "No files found matching the pattern." if none

**Tool 5: grep_search**

- Function: Search file contents using regex
- Parameters: `pattern: string` (required), `include?: string` (file filter), `searchPath?: string`
- Implementation: Use system `grep -rn` command, 30s timeout, 10MB buffer
- Note: grep exit code 1 means no matches (not an error), other exit codes are actual errors
- Returns: Matching lines (with file path and line number), "No matches found." if none

**Tool 6: run_command**

- Function: Execute shell command in project directory
- Parameters: `command: string` (required), `cwd?: string`
- Implementation: Use `child_process.exec`, 60s timeout, 10MB output buffer
- Returns: JSON-serialized `{ stdout, stderr }`, each truncated to 5000 chars

**Tool 7: list_directory**

- Function: List directory contents
- Parameters: `dirPath: string` (required), `recursive?: boolean` (max depth 3)
- Implementation: Non-recursive uses `fs.readdir`, recursive uses `glob`
- Returns: File/directory list (with 📁/📄 icons), "Directory is empty." if empty

---

### Step 4: Implement Tool Registry

Create `tool-registry.ts`, implementing the `ToolRegistry` class:

```
constructor(projectDir: string)
  - Call createBuiltinTools(projectDir) to get built-in tools
  - Iterate and register each built-in tool

register(definition: ToolDefinition, handler: (args) => Promise<string>): void
  - Store tool in Map<name, { definition, handler }>

registerMCPTools(mcpTools: Array<{ definition, handler }>): void
  - Add mcp__ prefix to each MCP tool name before registering
  - Avoid naming conflicts with built-in tools

getOpenAIToolDefinitions(): ToolDefinition[]
  - Return all registered tool definitions

execute(name: string, args: Record<string, unknown>): Promise<string>
  - Look up tool by name and execute handler
  - Throw Error if tool not found

get size(): number
  - Return count of registered tools
```

---

### Step 5: Implement Git Diff Tracker

Create `diff-tracker.ts`, implementing the `GitDiffTracker` class:

```
constructor(projectDir: string)
  - Initialize simple-git instance

async initialize(): Promise<void>
  - Record current Git HEAD hash
  - Set to empty string if no Git repository

trackFileChange(filePath: string): void
  - Convert absolute path to relative (relative to projectDir)
  - Store in changedFiles Set

getChangedFiles(): string[]
  - Return list of changed files (relative paths)

async collectDiffs(): Promise<FileDiff[]>
  - For each changed file:
    1. Determine action via git status (added/modified/deleted)
    2. Get diff text via git diff
    3. Count additions (lines starting with + but not +++) and deletions (lines starting with - but not ---)
  - Individual file exceptions don't affect other files

async revertAll(): Promise<void>
  - New files → fs.unlink delete
  - Existing files → git checkout restore
  - Clear changedFiles after completion

async finalize(): Promise<void>
  - Only clear changedFiles set
  - Do NOT perform git add or git commit
  - File changes already persisted to disk by tool handlers
```

---

### Step 6: Implement Agent Core Loop

Create `agent-loop.ts`, implementing the `CodeAgentLoop` class:

```
constructor(config: AgentConfig)
  1. Create OpenAI client (apiKey, baseURL, timeout, maxRetries, dangerouslyAllowBrowser: false)
  2. Create ToolRegistry(config.projectDir)
  3. Create GitDiffTracker(config.projectDir)
  4. Create SandboxExecutor()
  5. Create SkillManager(this.toolRegistry)
  6. Register built-in skills (iterate over builtinSkills)

async run(prompt, callbacks, options?): Promise<AgentResult>
  Full flow described after the System Prompt template below

async applyAll(): Promise<void>
  - Call diffTracker.finalize()

async revertAll(): Promise<void>
  - Call diffTracker.revertAll()

async validate(): Promise<ValidationResult>
  - Call sandbox.validateChanges(...)
  - Used for async validation scenarios

getToolRegistry(): ToolRegistry
  - Return toolRegistry instance (for external MCP tool registration)

getSkillManager(): SkillManager
  - Return skillManager instance (for external Skill registration)

private buildSystemPrompt(): string
  - Build System Prompt containing:
    1. Agent role definition
    2. Project directory info
    3. Available tools list with descriptions
    4. Workflow requirements (Phase 1-4: Analysis, Planning, Execution, Verification)
    5. Important rules
    6. Append Skill extensions (skillManager.getSystemPromptExtension())
```

**`run()` method complete execution flow:**

```
1. Initialize Git tracking: await this.diffTracker.initialize()
2. Build System Prompt: const systemPrompt = this.buildSystemPrompt()
3. Build message list: messages = [system, user]
4. Enter Agent Loop:
   while (true) {
     a. Check signal?.aborted → abort
     b. Call LLM (stream: true, tools: tool list, signal)
     c. Iterate streaming response chunks:
        - reasoning_content → callbacks.onThinking()
        - delta.content → callbacks.onContent()
        - delta.tool_calls → collect into Map<index, {id, name, args}>
     d. Catch AbortError → abort
     e. Check signal?.aborted → abort
     f. If toolCalls.size === 0 → break
     g. Append assistant message (with tool_calls) to messages
     h. Send onToolUse({ status: 'start' }) for all tools
     i. Execute all tools in parallel (Promise.all)
        - Check signal?.aborted before each tool execution
        - Success → record ToolCallRecord, track file changes (write_file/edit_file)
        - Failure → record error
     j. Sort results by original index
     k. Process results: send done/error events, append tool role messages
     l. Check signal?.aborted → abort
   }
5. If aborted → return AgentResult { aborted: true }
6. Sandbox validation (if skipValidation is false)
7. Collect diffs: await this.diffTracker.collectDiffs()
8. Return AgentResult
```

**System Prompt Template (required content):**

````
You are an expert coding agent working in the project at: {projectDir}

## Your Capabilities
You have access to the following tools to read, write, and modify code:
- read_file: Read file contents with optional line range
- write_file: Create or overwrite a file
- edit_file: Perform exact string replacements in a file (old_string must be unique)
- glob_files: Find files matching glob patterns
- grep_search: Search file contents with regex
- run_command: Execute shell commands (60s timeout, output truncated to 5000 chars)
- list_directory: List directory contents, optionally recursive (max depth 3)

## Workflow Requirements
When the user asks you to make code changes, you MUST follow this process:

### Phase 1: Analysis
1. First, understand the project structure by reading key files (package.json, tsconfig.json, main entry points)
2. Analyze the user's intent and what needs to be changed
3. Output your analysis as a clear thinking process

### Phase 2: Planning
1. Create a detailed step-by-step plan for the code changes
2. Each step should specify: action (create/edit/delete/run_command), file path, and what changes
3. Present the plan before making any changes

### Phase 3: Execution
1. Execute changes one step at a time
2. Use read_file to verify current file contents before editing
3. Use edit_file for targeted changes, write_file for new files
4. Run validation commands (lint, type-check, test) after changes

### Phase 4: Verification
1. Verify all changes are consistent
2. Run any relevant tests
3. Report what was changed and why

## Important Rules
- NEVER make changes before presenting a plan
- Always read a file before editing it (to ensure you have the latest content)
- Use edit_file for modifications (it's safer than write_file for existing files)
- Run validation commands after making changes (e.g., npm run lint, npx tsc --noEmit)
- If you encounter errors, analyze and fix them before proceeding
- Respect the project's existing code style and conventions
- Use relative paths from the project root
````

---

### Step 7: Implement Sandbox Validator

Create `sandbox.ts`, implementing the `SandboxExecutor` class:

```
async validateChanges(projectDir, changedFiles): Promise<ValidationResult>
  1. Check if package.json exists
  2. Read scripts field
  3. Determine validation commands based on changed file types:
     - .ts/.tsx file changes and tsconfig.json exists:
       → npm run type-check or npx tsc --noEmit
     - lint script exists → npm run lint
     - test script exists → npm run test -- --passWithNoTests
  4. No validation commands → return passed: true (skip validation)
  5. Execute validation commands sequentially (120s timeout, 10MB buffer)
  6. Collect stdout, stderr, exitCode for each command
  7. Return ValidationResult { passed, commands, outputs, errors }
```

---

### Step 8 (Optional): Implement MCP Manager

Create `mcp-manager.ts`, implementing the `MCPManager` class:

```
constructor(toolRegistry: ToolRegistry)

async loadServer(config: MCPServerConfig): Promise<void>
  1. Create StdioClientTransport(command, args, env)
  2. Create Client({ name: 'code-agent', version: '1.0.0' }, { capabilities: {} })
  3. client.connect(transport)
  4. client.listTools() to get tool list
  5. Convert each tool to OpenAI format:
     - Name with mcp__ prefix
     - Description preserved from original
     - parameters from inputSchema or default empty object
  6. toolRegistry.registerMCPTools(mcpTools) to register all tools
  7. Store in servers Map

async unloadServer(name: string): Promise<void>
  - client.close() to close connection

async unloadAll(): Promise<void>
  - Iterate and unload all servers

getLoadedServers(): string[]
  - Return list of loaded server names
```

---

### Step 9 (Optional): Implement Skill Manager

Create `skill-manager.ts`, implementing the `SkillManager` class:

```
constructor(toolRegistry: ToolRegistry)

registerSkill(skill: SkillDefinition): void
  - Store in skills Map
  - If skill has custom tools, register to ToolRegistry (placeholder handler)

unregisterSkill(name: string): void
  - Remove from skills Map

getSystemPromptExtension(): string
  - Iterate all Skills, merge systemPromptExtension
  - Format: \n## Active Skills\n### {name}: {description}\n{extension}\n
  - Returns empty string if no Skills

getSkills(): Array<{ name, description }>
  - Return list of registered Skills
```

**Built-in Skill definitions (create `skills/index.ts`):**

Provide 3 built-in skills as examples:

1. `typescript-strict` — Enforce strict TypeScript patterns (explicit types, interface over type, forbid any, readonly, const assertions, discriminated unions)
2. `react-patterns` — Follow React best practices (functional components + hooks, co-located styles, React.memo, custom hooks, useCallback/useMemo, accessibility)
3. `testing` — Generate tests alongside code changes (test coverage, update existing tests, edge cases, describe/it blocks)

---

### Step 10: Integrate into Application

**IPC channel definitions:**

Renderer → Main (send):
- `code:agent-start` — Start Agent
- `code:agent-apply` — Apply/revert changes
- `code:agent-cancel` — Cancel running Agent

Main → Renderer (event.sender.send):
- `code:agent-thinking` — Stream thinking chain
- `code:agent-response` — Stream response content
- `code:agent-tool-use` — Tool call status
- `code:agent-diff` — Diff results
- `code:agent-validation` — Validation results
- `code:agent-done` — Agent complete
- `code:agent-error` — Error info
- `code:agent-aborted` — Abort confirmation

**IPC Handler integration code template:**

```typescript
// Store active Agent instances and abort controllers
const activeAgents = new Map<string, CodeAgentLoop>();
const activeAbortControllers = new Map<string, AbortController>();

// Start Agent
ipcMain.on('code:agent-start', async (event, payload) => {
  const { modelConfigId, prompt, projectDir, sessionId } = payload;
  
  // 1. Read model config from database
  const config = await getModelConfig(modelConfigId);
  
  // 2. Create Agent instance
  const agent = new CodeAgentLoop({
    apiKey: config.apiKey,
    baseURL: config.baseURL,
    model: config.model,
    projectDir,
    timeout: config.timeout || 120000,
    maxRetries: config.maxRetries || 0,
  });
  
  activeAgents.set(sessionId, agent);
  
  // 3. Create AbortController
  const abortController = new AbortController();
  activeAbortControllers.set(sessionId, abortController);
  
  try {
    // 4. Execute Agent (skipValidation: true for async validation)
    const result = await agent.run(prompt, {
      onThinking: (content) => {
        event.sender.send('code:agent-thinking', { content });
      },
      onContent: (content) => {
        event.sender.send('code:agent-response', { content });
      },
      onToolUse: (toolEvent) => {
        event.sender.send('code:agent-tool-use', toolEvent);
      },
    }, { signal: abortController.signal, skipValidation: true });
    
    // 5. Send results
    if (result.aborted) {
      event.sender.send('code:agent-aborted', {
        thinking: result.thinking,
        response: result.response,
        toolCalls: result.toolCalls,
      });
    } else {
      event.sender.send('code:agent-diff', { files: result.diffs });
      event.sender.send('code:agent-done', {
        summary: result.response,
        thinking: result.thinking,
        response: result.response,
        diffs: result.diffs,
        validation: result.validation,
        toolCalls: result.toolCalls,
      });
      
      // 6. Async validation
      try {
        const validationResult = await agent.validate();
        event.sender.send('code:agent-validation', { result: validationResult });
      } catch (err) {
        event.sender.send('code:agent-validation', {
          result: { passed: false, commands: [], outputs: [], errors: [err.message] }
        });
      }
    }
  } catch (err) {
    event.sender.send('code:agent-error', { error: err.message, phase: 'execution' });
  } finally {
    activeAbortControllers.delete(sessionId);
  }
});

// Cancel Agent
ipcMain.on('code:agent-cancel', (_event, payload) => {
  const controller = activeAbortControllers.get(payload.sessionId);
  if (controller) {
    controller.abort();
  }
});

// Apply/Revert changes
ipcMain.on('code:agent-apply', async (_event, payload) => {
  const { sessionId, action } = payload;
  const agent = activeAgents.get(sessionId);
  if (!agent) return;
  
  try {
    if (action === 'accept') {
      await agent.applyAll();
    } else {
      await agent.revertAll();
    }
  } finally {
    activeAgents.delete(sessionId);
  }
});
```

---

### Implementation Notes

1. **All file paths must use relative paths** (relative to projectDir), tools internally convert via `resolvePath()` to absolute paths
2. **Tool calls must execute in parallel** (`Promise.all`), improving response speed for multi-tool scenarios
3. **run_command must set timeout and output limits** (60s timeout, 5000 char output truncation) to prevent malicious commands or infinite output
4. **edit_file must verify oldString existence and uniqueness** to prevent accidental multi-location modifications
5. **finalize() does NOT perform git add/commit**, only clears tracking list, files are already persisted to disk by tools
6. **Support AbortSignal cancellation**, with three checkpoints: before LLM call, during streaming, before tool execution
7. **MCP tools get name prefix** (`mcp__`) to avoid conflicts with built-in tools
8. **Skills are injected via System Prompt extension**, not modifying Agent core logic
9. **Sandbox validation auto-detects project type**, selecting validation commands based on package.json and tsconfig.json
10. **Error handling must be robust**, individual tool failures should not interrupt the entire Agent loop, tool results are returned as error messages to the LLM

---

### Verification Criteria

After implementation, verify through the following scenarios:

1. **Basic code modification:** Send "Create a README.md file", verify Agent can create file and return Diff
2. **Multi-file modification:** Send "Add JSDoc comments to all .ts files", verify Agent can search and edit multiple files
3. **Tool invocation:** Send "Install lodash and create a utility function using it", verify Agent can execute npm install and write files
4. **Cancellation:** Trigger cancellation during Agent execution, verify correct partial results and aborted flag are returned
5. **Revert changes:** Execute modifications then select Reject All, verify files are restored to pre-modification state
6. **Sandbox validation:** After modifying TypeScript files, verify Agent automatically runs tsc --noEmit
7. **MCP extension (optional):** Load Filesystem MCP Server, verify Agent can use MCP tools
8. **Skill extension (optional):** Register typescript-strict Skill, verify Agent follows strict TypeScript conventions

 
