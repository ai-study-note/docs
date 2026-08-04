# Code Agent Implementation Analysis & External Project Implementation Prompt

> This document provides a complete analysis of the features, workflow, and implementation of
> `packages/<package-name>/src/main/code-agent/`, and outputs an **English implementation prompt**
> that can be copied and used by other projects to quickly implement the same Code Agent.

---

## Table of Contents

- [1. Module Architecture & File List](#1-module-architecture--file-list)
- [2. Core Feature Analysis](#2-core-feature-analysis)
- [3. Agent Loop Workflow Details](#3-agent-loop-workflow-details)
- [4. System Prompt Construction Logic](#4-system-prompt-construction-logic)
- [5. Tool System Analysis](#5-tool-system-analysis)
- [6. MCP Manager Analysis](#6-mcp-manager-analysis)
- [7. Skill Manager Analysis](#7-skill-manager-analysis)
- [8. Git Diff Tracker Analysis](#8-git-diff-tracker-analysis)
- [9. Sandbox Executor Analysis](#9-sandbox-executor-analysis)
- [10. IPC Communication & Frontend-Backend Integration](#10-ipc-communication--frontend-backend-integration)
- [11. External Project Implementation Prompt (Copy-Ready)](#11-external-project-implementation-prompt-copy-ready)

---

## 1. Module Architecture & File List

### 1.1 Directory Structure

```
packages/<package-name>/src/main/code-agent/
├── index.ts              # Unified entry point, exports all public classes and types
├── agent-loop.ts         # Agent core loop (most important)
├── tool-registry.ts      # Tool registry
├── types.ts              # Complete type definitions
├── diff-tracker.ts       # Git Diff tracker
├── sandbox.ts            # Sandbox executor
├── mcp-manager.ts        # MCP manager (dynamic extension)
├── skill-manager.ts      # Skill manager
├── skills/
│   └── index.ts          # Built-in Skill definitions (3)
└── tools/
    └── index.ts          # Built-in tool implementations (7)
```

### 1.2 Module Responsibilities

| Module | File | Responsibility |
|--------|------|----------------|
| Entry | [index.ts](./packages/<package-name>/src/main/code-agent/index.ts) | Unified export of all public APIs |
| Core Loop | [agent-loop.ts](./packages/<package-name>/src/main/code-agent/agent-loop.ts) | Multi-turn Tool Calling loop, System Prompt construction, streaming callbacks |
| Type System | [types.ts](./packages/<package-name>/src/main/code-agent/types.ts) | AgentConfig, AgentCallbacks, AgentResult, WorkflowStep, FileDiff, ToolDefinition, etc. |
| Tool Registry | [tool-registry.ts](./packages/<package-name>/src/main/code-agent/tool-registry.ts) | Tool registration, MCP tool registration, OpenAI format export, tool execution |
| Tool Implementation | [tools/index.ts](./packages/<package-name>/src/main/code-agent/tools/index.ts) | Business logic for 7 built-in tools |
| Diff Tracking | [diff-tracker.ts](./packages/<package-name>/src/main/code-agent/diff-tracker.ts) | Git-based file change tracking, diff generation, revert/confirm |
| Sandbox Validation | [sandbox.ts](./packages/<package-name>/src/main/code-agent/sandbox.ts) | Auto-detect project type, run tsc/lint/test validation |
| MCP Manager | [mcp-manager.ts](./packages/<package-name>/src/main/code-agent/mcp-manager.ts) | Load MCP servers via stdio, discover and register tools |
| Skill Manager | [skill-manager.ts](./packages/<package-name>/src/main/code-agent/skill-manager.ts) | Skill registration, System Prompt extension merging |
| Built-in Skills | [skills/index.ts](./packages/<package-name>/src/main/code-agent/skills/index.ts) | typescript-strict, react-patterns, testing — three built-in Skills |

---

## 2. Core Feature Analysis

### 2.1 Feature Overview

The Code Agent is an **autonomous programming agent based on OpenAI Tool Calling** with the following core capabilities:

1. **Code Understanding**: Explore project structure via `read_file`, `glob_files`, `grep_search`, `list_directory`
2. **Code Modification**: Modify code via `write_file` (create/overwrite) and `edit_file` (exact string replacement)
3. **Command Execution**: Execute shell commands via `run_command` (install dependencies, run tests, build, etc.)
4. **Multi-turn Conversation**: Agent Loop automatically performs multiple rounds of tool calling until the task is complete
5. **Streaming Display**: Thinking chain and response content are streamed to the frontend in real time
6. **Change Tracking**: Automatically track all file changes via Git, generating unified diffs
7. **Sandbox Validation**: Automatically run TypeScript compilation, ESLint, tests, and other validation commands
8. **Cancellable**: Support AbortController to cancel execution at any time, preserving partial results
9. **Review & Confirm**: Users can view diffs and then choose Accept All or Reject All
10. **MCP Extension**: Support dynamic loading of external MCP servers to extend tool capabilities
11. **Skill Extension**: Support injecting coding standards and patterns via System Prompt extensions

### 2.2 Dependencies

```json
{
  "openai": "For LLM API calls and Tool Calling",
  "simple-git": "For Git Diff tracking",
  "glob": "For file searching",
  "@modelcontextprotocol/sdk": "For MCP Server connections"
}
```

---

## 3. Agent Loop Workflow Details

### 3.1 Overall Flow

```
User Input Prompt
    │
    ▼
[1. Initialize Git Tracking]
    │  GitDiffTracker.initialize()
    │  Records current Git HEAD for subsequent diff calculation
    │
    ▼
[2. Build System Prompt]
    │  Base Prompt (project context + tool list + 4-phase workflow + rules)
    │  + Skill extensions (SkillManager.getSystemPromptExtension())
    │
    ▼
[3. Build Message List]
    │  messages = [
    │    { role: 'system', content: systemPrompt },
    │    { role: 'user', content: prompt },
    │  ]
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  [4. Agent Loop: Multi-turn Tool Calling]                │
│                                                          │
│  while (true) {                                          │
│    ① Call OpenAI chat.completions.create({               │
│         model, messages, tools, tool_choice: 'auto',     │
│         stream: true                                     │
│       })                                                 │
│                                                          │
│    ② Stream collection:                                  │
│       - reasoning_content → callbacks.onThinking()       │
│       - delta.content → callbacks.onContent()            │
│       - delta.tool_calls → accumulate tool_calls Map     │
│                                                          │
│    ③ If toolCalls.size === 0 → break (complete)         │
│                                                          │
│    ④ Append assistant message (with tool_calls)          │
│                                                          │
│    ⑤ Iterate and execute each tool call:                 │
│       - callbacks.onToolUse({ status: 'start' })         │
│       - Execute toolRegistry.execute(name, args)         │
│       - If tool is write_file/edit_file,                 │
│         call diffTracker.trackFileChange()               │
│       - callbacks.onToolUse({ status: 'done/error' })    │
│       - Append tool result message to messages           │
│  }                                                       │
└─────────────────────────────────────────────────────────┘
    │
    ▼
[5. Sandbox Validation]
    │  SandboxExecutor.validateChanges()
    │  Auto-detect project type, run:
    │  - npx tsc --noEmit (when tsconfig.json exists)
    │  - npm run lint (when lint script exists)
    │  - npm run test -- --passWithNoTests (when test script exists)
    │
    ▼
[6. Collect Diffs]
    │  GitDiffTracker.collectDiffs()
    │  For each changed file:
    │  - Determine action type via git status (added/modified/deleted)
    │  - Get unified diff text via git diff
    │  - Count +/- lines
    │
    ▼
[7. Return AgentResult]
    │  {
    │    thinking: "complete thinking content",
    │    response: "complete response content",
    │    diffs: FileDiff[],
    │    validation: ValidationResult,
    │    toolCalls: ToolCallRecord[],
    │    aborted: false
    │  }
    │
    ▼
[8. User Approval]
    │  Accept All → agent.applyAll() → save all changed files
    │  Reject All → agent.revertAll() → git checkout restore files / delete new files
```

### 3.2 Cancellation Handling

The Agent Loop supports cancellation via `AbortSignal`:

- Check `signal.aborted` before each LLM call
- Catch `AbortError` in the stream's for-await loop
- Check `signal.aborted` before each tool execution
- After cancellation, return `AgentResult` with `aborted: true`, empty `diffs` and `validation`

### 3.3 Error Handling

- **Tool execution failure**: Catch exception, notify frontend with `ToolUseEvent.status: 'error'`, and return the error message as tool result to the LLM so it can self-correct
- **Diff collection failure**: Wrapped in try-catch, returns empty array on failure without interrupting the flow
- **Entire Agent run failure**: Notify frontend via `code:agent-error` IPC event, clean up Agent instance

---

## 4. System Prompt Construction Logic

### 4.1 Construction Flow

The System Prompt is assembled in the `buildSystemPrompt()` method of [agent-loop.ts](./packages/<package-name>/src/main/code-agent/agent-loop.ts#L330-L383):

```typescript
private buildSystemPrompt(): string {
  let prompt = `[Base Prompt: project context, tool list, 4-phase workflow, rules]`

  // Append Skill extensions
  const skillExtension = this.skillManager.getSystemPromptExtension()
  if (skillExtension) {
    prompt += '\n' + skillExtension
  }

  return prompt
}
```

### 4.2 Base Prompt Structure

The base prompt contains the following sections:

**a) Identity Declaration**
```
You are an expert coding agent working in the project at: ${projectDir}
```

**b) Capability Declaration (Tool List)**
```
## Your Capabilities
You have access to the following tools to read, write, and modify code:
- read_file: Read file contents with optional line range
- write_file: Create or overwrite a file
- edit_file: Perform exact string replacements in a file (old_string must be unique)
- glob_files: Find files matching glob patterns
- grep_search: Search file contents with regex
- run_command: Execute shell commands (60s timeout, output truncated to 5000 chars)
- list_directory: List directory contents, optionally recursive (max depth 3)
```

**c) Workflow Requirements (4 Phases)**
```
## Workflow Requirements
When the user asks you to make code changes, you MUST follow this process:

### Phase 1: Analysis
1. First, understand the project structure by reading key files
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
```

**d) Important Rules**
```
## Important Rules
- NEVER make changes before presenting a plan
- Always read a file before editing it
- Use edit_file for modifications (it's safer than write_file for existing files)
- Run validation commands after making changes
- If you encounter errors, analyze and fix them before proceeding
- Respect the project's existing code style and conventions
- Use relative paths from the project root
```

### 4.3 Skill Extensions

Skill extensions are merged via `SkillManager.getSystemPromptExtension()`, combining the `systemPromptExtension` fields of all registered Skills:

```typescript
getSystemPromptExtension(): string {
  if (this.skills.size === 0) return ''

  const sections: string[] = ['\n## Active Skills\n']
  for (const [name, skill] of this.skills) {
    sections.push(`### ${name}: ${skill.description}`)
    sections.push(skill.systemPromptExtension)
    sections.push('')
  }
  return sections.join('\n')
}
```

Three built-in Skills:
- **typescript-strict**: Enforce strict TypeScript patterns (explicit types, interfaces, no any, readonly, discriminated unions)
- **react-patterns**: React best practices (functional components, CSS Modules, React.memo, custom hooks, useCallback/useMemo, accessibility)
- **testing**: Generate tests alongside code changes (test coverage, update existing tests, describe/it organization, boundary value testing)

---

## 5. Tool System Analysis

### 5.1 Tool Definition Format

Each tool follows the OpenAI Function Calling format:

```typescript
interface ToolDefinition {
  type: 'function';
  function: {
    name: string;           // Tool name
    description: string;    // Tool description
    parameters: Record<string, unknown>;  // JSON Schema parameter definition
  };
}
```

### 5.2 7 Built-in Tools

| Tool | Function | Key Parameters | Safety Limits |
|------|----------|----------------|---------------|
| `read_file` | Read file contents with line numbers | `filePath`, `startLine?`, `endLine?` | Path relative to project root |
| `write_file` | Create new file or overwrite existing | `filePath`, `content` | Auto-create parent directories |
| `edit_file` | Exact string replacement (old_string must be unique) | `filePath`, `oldString`, `newString` | Refuse on multiple matches |
| `glob_files` | File glob search | `pattern`, `cwd?` | Auto-exclude node_modules/.git/dist/build |
| `grep_search` | Regex content search | `pattern`, `include?`, `searchPath?` | 30s timeout, 10MB buffer |
| `run_command` | Execute shell commands | `command`, `cwd?` | 60s timeout, output truncated to 5000 chars |
| `list_directory` | List directory contents | `dirPath`, `recursive?` | Max recursion depth 3, excludes node_modules |

### 5.3 Tool Registry

[ToolRegistry](./packages/<package-name>/src/main/code-agent/tool-registry.ts) is a Map-based structure:

- `register(definition, handler)` — Register a single tool
- `registerMCPTools(mcpTools)` — Register MCP tools (auto-prefixed with `mcp__`)
- `getOpenAIToolDefinitions()` — Export tool list in OpenAI format
- `execute(name, args)` — Execute a tool by name

The constructor accepts a `projectDir` parameter; all built-in tool file operations are resolved relative to this directory.

### 5.4 edit_file Safety Design

`edit_file` is the most complex tool with two safety checks:

1. **oldString must exist**: If oldString is not found in the file, throw an error
2. **oldString must be unique**: If oldString appears multiple times in the file, throw an error requiring more context to make it unique

This ensures precise and safe editing operations, preventing accidental modifications.

---

## 6. MCP Manager Analysis

### 6.1 Design Goal

[MCPManager](./packages/<package-name>/src/main/code-agent/mcp-manager.ts) allows the Agent to dynamically load external MCP servers to extend tool capabilities.

### 6.2 Workflow

```
1. User configures MCP Server (name, command, args, env)
       │
       ▼
2. mcpManager.loadServer(config)
       │
       ├── Create StdioClientTransport (launch external process)
       ├── Create MCP Client and connect
       ├── Call client.listTools() to discover tools
       ├── Create adapters for each tool (with mcp__ prefix)
       │   - definition: Wrap in OpenAI Function Calling format
       │   - handler: Call client.callTool() and extract text content
       └── Register into ToolRegistry
```

### 6.3 Tool Prefix

MCP tools are automatically prefixed with `mcp__` when registered to avoid conflicts with built-in tools. For example, a `read_file` tool from an MCP server would be registered as `mcp__read_file`.

### 6.4 Configuration Example

```typescript
{
  name: 'filesystem',
  command: 'npx',
  args: ['-y', '@modelcontextprotocol/server-filesystem', '/path/to/project'],
}
```

---

## 7. Skill Manager Analysis

### 7.1 Design Goal

[SkillManager](./packages/<package-name>/src/main/code-agent/skill-manager.ts) guides the Agent's coding behavior through System Prompt extensions.

### 7.2 Comparison with MCP

| Dimension | MCP | Skill |
|-----------|-----|-------|
| Extension Method | Provides external tools (database, browser, API) | Provides coding patterns (React components, API endpoints, tests) |
| Implementation | Launch external process, communicate via MCP protocol | Plain text injection into System Prompt |
| Tool Registration | Dynamic discovery and registration | Optional, Skills can include custom tools |
| Scope | Grants Agent new operational capabilities | Guides Agent on how to write code |

### 7.3 Skill Data Structure

```typescript
interface SkillDefinition {
  name: string;                    // Unique name
  description: string;             // Description
  systemPromptExtension: string;   // Instructions injected into System Prompt
  tools?: ToolDefinition[];        // Optional custom tools
}
```

### 7.4 Built-in Skill Details

**typescript-strict**:
- Enforce explicit type annotations
- Prefer interface over type alias
- Forbid any, use unknown
- Use readonly modifiers
- Use const assertions and discriminated unions

**react-patterns**:
- Use functional components and hooks
- Co-locate styles with components (CSS Modules)
- Use React.memo() for pure components
- Extract custom hooks for reusable logic
- Use useCallback/useMemo for optimization
- Add accessibility attributes

**testing**:
- Always consider test coverage
- Update tests when modifying functions
- Suggest test cases for new functions
- Use describe/it organization
- Test edge cases: null, undefined, empty arrays, etc.

---

## 8. Git Diff Tracker Analysis

### 8.1 Design Goal

[GitDiffTracker](./packages/<package-name>/src/main/code-agent/diff-tracker.ts) is based on the `simple-git` library and can work even without an existing Git repository.

### 8.2 Core Methods

| Method | Function |
|--------|----------|
| `initialize()` | Record current Git HEAD (for subsequent comparison) |
| `trackFileChange(filePath)` | Record a file change (auto-convert to relative path) |
| `getChangedFiles()` | Get list of all changed file paths |
| `collectDiffs()` | Collect unified diffs for all changed files |
| `revertAll()` | Revert all changes (delete new files, git checkout existing files) |
| `finalize()` | Confirm changes (save all changed files) |

### 8.3 Diff Collection Logic

`collectDiffs()` for each changed file:
1. Determine action type via `git status` (added/modified/deleted)
2. Get diff text via `git diff` (use `--cached` for new files)
3. Count `+` and `-` lines
4. Return array of `FileDiff` objects

### 8.4 Revert Logic

`revertAll()` for each changed file:
- If it's a newly created file (`status.created` includes the file): delete via `fs.unlink`
- If it's an existing file: restore via `git checkout -- file`

---

## 9. Sandbox Executor Analysis

### 9.1 Design Goal

[SandboxExecutor](./packages/<package-name>/src/main/code-agent/sandbox.ts) automatically runs validation commands after code changes to ensure correctness.

### 9.2 Validation Command Auto-Detection

```
1. Check if package.json exists
       │
       ▼
2. Detect changed file types
       │
       ├── .ts/.tsx files changed + tsconfig.json exists
       │   ├── Has type-check/typecheck script → npm run type-check
       │   └── No type-check script → npx tsc --noEmit
       │
       ├── Has lint script → npm run lint
       │
       └── Has test script → npm run test -- --passWithNoTests
```

### 9.3 Validation Result

```typescript
interface ValidationResult {
  passed: boolean;       // Whether all passed
  commands: string[];    // List of executed commands
  outputs: Array<{
    command: string;
    stdout: string;
    stderr: string;
    exitCode: number;
  }>;
  errors: string[];      // Error messages for failed commands
}
```

---

## 10. IPC Communication & Frontend-Backend Integration

### 10.1 Communication Architecture

This project is an Electron application. The Agent runs in the Main Process, and the frontend runs in the Renderer Process.

```
Renderer (React)              Main Process (Node.js)
      │                              │
      │── code:agent-start ─────────▶│  Start Agent
      │                              │  Create CodeAgentLoop instance
      │                              │  Call agent.run()
      │                              │
      │◀─ code:agent-thinking ──────│  Stream thinking content
      │◀─ code:agent-response ──────│  Stream response content
      │◀─ code:agent-tool-use ──────│  Tool call status (start/done/error)
      │◀─ code:agent-diff ──────────│  Changed file diff list
      │◀─ code:agent-validation ────│  Sandbox validation result
      │◀─ code:agent-done ──────────│  Complete (with summary, thinking, stats)
      │◀─ code:agent-error ─────────│  Error (with phase marker)
      │◀─ code:agent-aborted ───────│  Cancelled by user (with partial results)
      │                              │
      │── code:agent-cancel ───────▶│  Cancel execution (AbortController)
      │── code:agent-apply ────────▶│  Accept/revert changes
```

### 10.2 Key IPC Payloads

**Start Agent**:
```typescript
// Renderer → Main
{ modelConfigId: string; prompt: string; projectDir: string; sessionId: string }
```

**Agent Complete**:
```typescript
// Main → Renderer
{
  summary: string;           // Response content
  thinking: string;          // Thinking content
  toolCallCount: number;     // Number of tool calls
  diffCount: number;         // Number of changed files
  validationPassed: boolean; // Whether validation passed
}
```

**Agent Aborted**:
```typescript
// Main → Renderer
{
  summary: string;           // Partial response content
  thinking: string;          // Partial thinking content
  toolCallCount: number;     // Number of executed tool calls
}
```

### 10.3 Agent Instance Management

- `activeAgents: Map<string, CodeAgentLoop>` — Store active Agents by sessionId
- `activeAbortControllers: Map<string, AbortController>` — Store abort controllers by sessionId
- Agent is created on `code:agent-start`, cleaned up after `code:agent-apply`
- Abort controller is cleaned up after Agent completes; Agent instance is kept for user approval

---

## 11. External Project Implementation Prompt (Copy-Ready)

> **Usage**: The following prompt can be copied directly to an AI coding assistant (Claude Code, Codex CLI, Cursor, etc.)
> to implement the same Code Agent system in **any project**. Execute in Phase order.

---

```markdown
# Code Agent Implementation Task

## Project Background

You are implementing an AI Code Agent based on OpenAI Tool Calling for a project.
The Agent can autonomously read code, modify files, execute commands, and display its thinking process through streaming callbacks.

## Tech Stack

- Runtime: Node.js (not limited to Electron; any Node.js backend works)
- LLM SDK: openai (npm package)
- Required dependencies: simple-git, glob
- Optional dependency: @modelcontextprotocol/sdk (for MCP extension)

## Implementation Steps

### Phase 1: Type Definitions

Create `types.ts` with the following interfaces:

```typescript
// Agent configuration
export interface AgentConfig {
  apiKey: string;
  baseURL?: string;
  model: string;
  projectDir: string;
  timeout?: number;
  maxRetries?: number;
}

// Streaming callbacks
export interface AgentCallbacks {
  onThinking: (content: string) => void;
  onContent: (content: string) => void;
  onToolUse: (event: ToolUseEvent) => void;
}

// Tool call event
export interface ToolUseEvent {
  toolName: string;
  args: string;
  status: 'start' | 'done' | 'error';
  result?: string;
  error?: string;
}

// Tool call record
export interface ToolCallRecord {
  toolName: string;
  args: string;
  result: string;
  timestamp: number;
  success: boolean;
}

// Workflow step
export interface WorkflowStep {
  index: number;
  action: 'create' | 'edit' | 'delete' | 'run_command';
  filePath: string;
  description: string;
}

// File diff
export interface FileDiff {
  filePath: string;
  action: 'added' | 'modified' | 'deleted';
  diff: string;
  additions: number;
  deletions: number;
}

// Validation result
export interface ValidationResult {
  passed: boolean;
  commands: string[];
  outputs: Array<{
    command: string;
    stdout: string;
    stderr: string;
    exitCode: number;
  }>;
  errors: string[];
}

// Agent execution result
export interface AgentResult {
  thinking: string;
  response: string;
  diffs: FileDiff[];
  validation: ValidationResult;
  toolCalls: ToolCallRecord[];
  aborted?: boolean;
}

// OpenAI tool definition format
export interface ToolDefinition {
  type: 'function';
  function: {
    name: string;
    description: string;
    parameters: Record<string, unknown>;
  };
}

// Skill definition
export interface SkillDefinition {
  name: string;
  description: string;
  systemPromptExtension: string;
  tools?: ToolDefinition[];
}

// MCP Server configuration
export interface MCPServerConfig {
  name: string;
  command: string;
  args?: string[];
  env?: Record<string, string>;
  autoApprove?: boolean;
}
```

### Phase 2: Tool System

#### Step 2.1: Create Tool Registry

Create `tool-registry.ts`:

```typescript
import type { ToolDefinition } from './types';

export class ToolRegistry {
  private tools: Map<string, {
    definition: ToolDefinition;
    handler: (args: Record<string, unknown>) => Promise<string>;
  }> = new Map();

  constructor(projectDir: string) {
    // Register built-in tools (see Step 2.2)
    const builtinTools = createBuiltinTools(projectDir);
    for (const tool of builtinTools) {
      this.register(tool.definition, tool.handler);
    }
  }

  register(definition: ToolDefinition, handler: (args: Record<string, unknown>) => Promise<string>): void {
    this.tools.set(definition.function.name, { definition, handler });
  }

  registerMCPTools(mcpTools: Array<{
    definition: ToolDefinition;
    handler: (args: Record<string, unknown>) => Promise<string>;
  }>): void {
    for (const tool of mcpTools) {
      const mcpDef: ToolDefinition = {
        ...tool.definition,
        function: {
          ...tool.definition.function,
          name: `mcp__${tool.definition.function.name}`,
        },
      };
      this.register(mcpDef, tool.handler);
    }
  }

  getOpenAIToolDefinitions(): ToolDefinition[] {
    return Array.from(this.tools.values()).map((t) => t.definition);
  }

  async execute(name: string, args: Record<string, unknown>): Promise<string> {
    const tool = this.tools.get(name);
    if (!tool) throw new Error(`Tool not found: ${name}`);
    return tool.handler(args);
  }
}
```

#### Step 2.2: Implement 7 Built-in Tools

Create `tools.ts`, implementing the `createBuiltinTools(projectDir)` function that returns 7 tools:

1. **read_file**
   - Parameters: `filePath` (required), `startLine?`, `endLine?`
   - Function: Read file, return content with line numbers (supports line range)
   - Path: Resolved relative to projectDir

2. **write_file**
   - Parameters: `filePath` (required), `content` (required)
   - Function: Create or overwrite file, auto-create parent directories
   - Returns: Success message (with line count)

3. **edit_file**
   - Parameters: `filePath` (required), `oldString` (required), `newString` (required)
   - Function: Exact string replacement, oldString must be unique
   - Safety checks: Error if not found, error if multiple occurrences

4. **glob_files**
   - Parameters: `pattern` (required), `cwd?`
   - Function: File glob search
   - Auto-exclude: node_modules, dist, .git, build

5. **grep_search**
   - Parameters: `pattern` (required), `include?`, `searchPath?`
   - Function: Regex content search, returns matching lines
   - Safety limits: 30s timeout, 10MB buffer

6. **run_command**
   - Parameters: `command` (required), `cwd?`
   - Function: Execute shell commands
   - Safety limits: 60s timeout, output truncated to 5000 chars

7. **list_directory**
   - Parameters: `dirPath` (required), `recursive?`
   - Function: List directory contents, max recursion depth 3
   - Auto-exclude: node_modules, dist, .git, build

Each tool includes an OpenAI Function Calling format definition and a Node.js implementation function.

### Phase 3: Git Diff Tracker

Create `diff-tracker.ts`, implementing the `GitDiffTracker` class:
- Uses the `simple-git` library
- `initialize()` — Record current Git HEAD
- `trackFileChange(filePath)` — Track file change (auto-convert to relative path)
- `getChangedFiles()` — Get list of changed files
- `collectDiffs()` — Collect unified diffs for all changed files:
  - Determine action type via `git status` (added/modified/deleted)
  - Get diff text via `git diff`
  - Count +/- lines
- `revertAll()` — Revert all changes:
  - New files → delete via `fs.unlink`
  - Existing files → restore via `git checkout -- file`
- `finalize()` — Confirm changes: `save` all changed files

### Phase 4: Sandbox Executor

Create `sandbox.ts`, implementing the `SandboxExecutor` class:
- `validateChanges(projectDir, changedFiles)` — Validate code changes:
  1. Check if `package.json` exists
  2. Detect changed file types (.ts/.tsx)
  3. If TypeScript files changed + tsconfig.json exists → run `npx tsc --noEmit`
  4. If lint script exists → run `npm run lint`
  5. If test script exists → run `npm run test -- --passWithNoTests`
  6. Return `ValidationResult` (passed, commands, outputs, errors)

### Phase 5: Skill Manager

Create `skill-manager.ts`, implementing the `SkillManager` class:
- `registerSkill(skill)` — Register a Skill (optional custom tools)
- `getSystemPromptExtension()` — Merge all Skill System Prompt extensions
- Three built-in Skills (can be used directly):

**typescript-strict**:
```
When writing TypeScript code:
1. Always define explicit types for function parameters and return values
2. Use interfaces over type aliases for object shapes
3. Avoid using 'any' - use 'unknown' instead
4. Use readonly modifiers for immutable data
5. Prefer const assertions over type casting
6. Use discriminated unions for state management
```

**react-patterns**:
```
When writing React code:
1. Use functional components with hooks
2. Co-locate styles with components (CSS modules or CSS-in-JS)
3. Use React.memo() for pure components
4. Extract custom hooks for reusable logic
5. Use useCallback/useMemo for expensive computations
6. Add proper accessibility attributes (aria-labels, roles)
```

**testing**:
```
When making code changes:
1. Always consider test coverage
2. If modifying a function, update its corresponding tests
3. If creating a new function, suggest test cases
4. Use describe/it blocks for organization
5. Test edge cases: null, undefined, empty arrays, boundary values
```

### Phase 6: Agent Loop Core

Create `agent-loop.ts`, implementing the `CodeAgentLoop` class. This is the most critical module.

#### Constructor

```typescript
constructor(config: AgentConfig) {
  // 1. Create OpenAI client
  this.openai = new OpenAI({
    apiKey: config.apiKey,
    baseURL: config.baseURL,
    timeout: config.timeout || 120000,
    maxRetries: config.maxRetries || 0,
  });

  // 2. Create Tool Registry (pass in projectDir)
  this.toolRegistry = new ToolRegistry(config.projectDir);

  // 3. Create Git Diff Tracker
  this.diffTracker = new GitDiffTracker(config.projectDir);

  // 4. Create Sandbox Executor
  this.sandbox = new SandboxExecutor();

  // 5. Create Skill Manager and register built-in Skills
  this.skillManager = new SkillManager(this.toolRegistry);
  for (const skill of builtinSkills) {
    this.skillManager.registerSkill(skill);
  }
}
```

#### Core Method: run(prompt, callbacks, options?)

Complete flow:

```typescript
async run(prompt: string, callbacks: AgentCallbacks, options?: { signal?: AbortSignal }): Promise<AgentResult> {
  const signal = options?.signal;

  // 1. Initialize Git tracking
  await this.diffTracker.initialize();

  // 2. Build System Prompt
  const systemPrompt = this.buildSystemPrompt();

  // 3. Build initial messages
  const messages = [
    { role: 'system', content: systemPrompt },
    { role: 'user', content: prompt },
  ];

  let fullResponse = '';
  let fullThinking = '';
  let aborted = false;

  // 4. Agent Loop
  while (true) {
    // 4a. Check cancellation signal
    if (signal?.aborted) { aborted = true; break; }

    // 4b. Call OpenAI (streaming + Tool Calling)
    const stream = await this.openai.chat.completions.create({
      model: this.config.model,
      messages,
      tools: this.toolRegistry.getOpenAIToolDefinitions(),
      tool_choice: 'auto',
      stream: true,
    }, signal ? { signal } : undefined);

    // 4c. Collect streaming response
    let content = '';
    const toolCalls = new Map<number, { id: string; name: string; args: string }>();

    for await (const chunk of stream) {
      const delta = chunk.choices?.[0]?.delta;
      if (!delta) continue;

      // Thinking chain (DeepSeek R1, Claude extended thinking)
      const reasoning = (delta as any).reasoning_content;
      if (reasoning) {
        fullThinking += reasoning;
        callbacks.onThinking(reasoning);
      }

      // Regular content
      if (delta.content) {
        content += delta.content;
        fullResponse += delta.content;
        callbacks.onContent(delta.content);
      }

      // Tool calls
      if (delta.tool_calls) {
        for (const tc of delta.tool_calls) {
          const idx = tc.index;
          if (!toolCalls.has(idx)) {
            toolCalls.set(idx, { id: tc.id || '', name: tc.function?.name || '', args: '' });
          }
          const entry = toolCalls.get(idx)!;
          if (tc.id) entry.id = tc.id;
          if (tc.function?.name) entry.name = tc.function.name;
          if (tc.function?.arguments) entry.args += tc.function.arguments;
        }
      }
    }

    // 4d. No tool calls → complete
    if (toolCalls.size === 0) break;

    // 4e. Append assistant message (with tool_calls) to conversation
    messages.push({
      role: 'assistant',
      content: content || null,
      tool_calls: Array.from(toolCalls.entries()).map(([, tc]) => ({
        id: tc.id,
        type: 'function',
        function: { name: tc.name, arguments: tc.args },
      })),
    });

    // 4f. Execute all tool calls
    for (const [, tc] of toolCalls) {
      if (signal?.aborted) { aborted = true; break; }

      callbacks.onToolUse({ toolName: tc.name, args: tc.args, status: 'start' });

      try {
        const result = await this.toolRegistry.execute(tc.name, JSON.parse(tc.args));
        // Record tool call
        this.toolCallHistory.push({ toolName: tc.name, args: tc.args, result, timestamp: Date.now(), success: true });
        // Track file changes
        if (['write_file', 'edit_file'].includes(tc.name)) {
          const args = JSON.parse(tc.args);
          if (args.filePath) this.diffTracker.trackFileChange(args.filePath);
        }
        callbacks.onToolUse({ toolName: tc.name, args: tc.args, status: 'done', result });
        messages.push({ role: 'tool', tool_call_id: tc.id, content: result });
      } catch (error) {
        const errMsg = error instanceof Error ? error.message : String(error);
        this.toolCallHistory.push({ toolName: tc.name, args: tc.args, result: errMsg, timestamp: Date.now(), success: false });
        callbacks.onToolUse({ toolName: tc.name, args: tc.args, status: 'error', error: errMsg });
        messages.push({ role: 'tool', tool_call_id: tc.id, content: `Error: ${errMsg}` });
      }
    }

    if (aborted) break;
  }

  // 5. Handle cancellation
  if (aborted) {
    return {
      thinking: fullThinking,
      response: fullResponse,
      diffs: [],
      validation: { passed: false, commands: [], outputs: [], errors: ['Aborted by user'] },
      toolCalls: this.toolCallHistory,
      aborted: true,
    };
  }

  // 6. Sandbox validation
  const validation = await this.sandbox.validateChanges(
    this.config.projectDir,
    this.diffTracker.getChangedFiles(),
  );

  // 7. Collect diffs
  const diffs = await this.diffTracker.collectDiffs();

  return {
    thinking: fullThinking,
    response: fullResponse,
    diffs,
    validation,
    toolCalls: this.toolCallHistory,
    aborted: false,
  };
}
```

#### System Prompt Construction Method

```typescript
private buildSystemPrompt(): string {
  let prompt = `You are an expert coding agent working in the project at: ${this.config.projectDir}

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
- Use relative paths from the project root`;

  // Append Skill extensions
  const skillExtension = this.skillManager.getSystemPromptExtension();
  if (skillExtension) {
    prompt += '\n' + skillExtension;
  }

  return prompt;
}
```

#### Other Methods

```typescript
// Accept all changes
async applyAll(): Promise<void> {
  await this.diffTracker.finalize();
}

// Revert all changes
async revertAll(): Promise<void> {
  await this.diffTracker.revertAll();
}
```

### Phase 7: MCP Manager (Optional)

If the project needs MCP extension, create `mcp-manager.ts`:

```typescript
import { Client } from '@modelcontextprotocol/sdk/client/index.js';
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js';

export class MCPManager {
  private servers: Map<string, { config: MCPServerConfig; client: Client; transport: StdioClientTransport }> = new Map();
  private toolRegistry: ToolRegistry;

  constructor(toolRegistry: ToolRegistry) {
    this.toolRegistry = toolRegistry;
  }

  async loadServer(config: MCPServerConfig): Promise<void> {
    // 1. Create stdio transport layer
    const transport = new StdioClientTransport({
      command: config.command,
      args: config.args || [],
      env: config.env,
    });

    // 2. Create MCP Client and connect
    const client = new Client(
      { name: 'code-agent', version: '1.0.0' },
      { capabilities: {} },
    );
    await client.connect(transport);

    // 3. Discover tools
    const { tools } = await client.listTools();

    // 4. Register tools (with mcp__ prefix)
    const mcpTools = tools.map((tool) => ({
      definition: {
        type: 'function',
        function: {
          name: `mcp__${tool.name}`,
          description: tool.description || `MCP tool: ${tool.name}`,
          parameters: tool.inputSchema || { type: 'object', properties: {} },
        },
      },
      handler: async (args) => {
        const result = await client.callTool({ name: tool.name, arguments: args });
        const textContents = (result.content as Array<{ type: string; text?: string }>)
          .filter((c) => c.type === 'text')
          .map((c) => c.text);
        return textContents.join('\n');
      },
    }));

    this.toolRegistry.registerMCPTools(mcpTools);
    this.servers.set(config.name, { config, client, transport });
  }

  async unloadServer(name: string): Promise<void> {
    const server = this.servers.get(name);
    if (!server) return;
    await server.client.close();
    this.servers.delete(name);
  }

  async unloadAll(): Promise<void> {
    for (const [name] of this.servers) {
      await this.unloadServer(name);
    }
  }
}
```

### Phase 8: Integration into Application

#### Usage

```typescript
import { CodeAgentLoop } from './agent-loop';

// 1. Create Agent instance
const agent = new CodeAgentLoop({
  apiKey: 'sk-xxx',
  model: 'gpt-4o',
  projectDir: '/path/to/project',
});

// 2. Create abort controller
const abortController = new AbortController();

// 3. Execute Agent
const result = await agent.run(
  'Add a Redis caching layer to this project',
  {
    onThinking: (content) => {
      // Stream thinking content → push to frontend
      console.log('[Thinking]', content);
    },
    onContent: (content) => {
      // Stream response content → push to frontend
      console.log('[Content]', content);
    },
    onToolUse: (event) => {
      // Tool call status → push to frontend
      console.log('[Tool]', event.toolName, event.status);
    },
  },
  { signal: abortController.signal }
);

// 4. Process results
console.log('Diffs:', result.diffs.length, 'files');
console.log('Validation:', result.validation.passed ? 'PASS' : 'FAIL');
console.log('Tool calls:', result.toolCalls.length);

// 5. User selection
if (userApproved) {
  await agent.applyAll();  // Keep changes
} else {
  await agent.revertAll(); // Revert changes
}
```

#### Cancelling Execution

```typescript
// User can cancel at any time
abortController.abort();
// Agent will return AgentResult with aborted: true, containing partial results
```

### Key Design Decisions

1. **edit_file's old_string must be unique**: A safety design to prevent accidental modification of multiple code locations
2. **Tool execution failure does not interrupt the loop**: Error messages are returned as tool results to the LLM, allowing it to self-correct
3. **run_command limited to 60-second timeout**: Prevents long-running commands from blocking the Agent
4. **Diff collection failure does not interrupt the flow**: Wrapped in try-catch, returns empty array on failure
5. **MCP tool prefix `mcp__`**: Avoids naming conflicts between MCP tools and built-in tools
6. **Skills injected via System Prompt**: Plain text extension, does not modify Agent loop logic
7. **Cancellation signal checked at multiple points**: Before LLM calls, during stream loops, and before tool execution

### Important Notes

1. All file operation paths are resolved relative to `projectDir`
2. `read_file` returns content with line numbers, making it easier for `edit_file` to match precisely
3. `edit_file`'s `oldString` must include complete indentation and whitespace characters
4. `run_command` output is truncated to 5000 characters to avoid context overflow
5. If MCP support is needed, install `@modelcontextprotocol/sdk`
6. If the project has no Git repository, DiffTracker will gracefully degrade


 
