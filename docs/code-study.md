# Code Agent Workflow Technical Design

> Transform the `/code` page into a true Code Agent, implementing an AI Chat-style code agent client.
> Reference implementations: Codex Desktop Client, Claude Code Desktop Client.

---

## Table of Contents

- [1. Open-Source npm Module Selection](#1-open-source-npm-module-selection)
- [2. Overall Architecture Design](#2-overall-architecture-design)
- [3. Agent Loop Core Design](#3-agent-loop-core-design)
- [4. Tool System](#4-tool-system)
- [5. MCP Manager (Dynamic Extension)](#5-mcp-manager-dynamic-extension)
- [6. Skill Manager (Custom Skills)](#6-skill-manager-custom-skills)
- [7. Sandbox Validator](#7-sandbox-validator)
- [8. Git Diff Tracker](#8-git-diff-tracker)
- [9. IPC Communication Design](#9-ipc-communication-design)
- [10. Frontend UI Design](#10-frontend-ui-design)
- [11. File Structure Planning](#11-file-structure-planning)
- [12. Implementation Steps](#12-implementation-steps)
- [13. Cost Estimation](#13-cost-estimation)
- [Appendix: Complete Copyable Implementation Prompt](#appendix-complete-copyable-implementation-prompt)

---

## 1. Open-Source npm Module Selection

### 1.1 Core Solution Comparison

| Solution | Type | License | Advantages | Disadvantages | Recommendation |
|----------|------|---------|------------|--------------|----------------|
| **Custom Agent Loop (OpenAI SDK + Tool Calling)** | SDK | MIT | Fully controllable, flexible customization, deep Electron integration | Requires self-implementation of Agent loop | ⭐⭐⭐⭐⭐ |
| `@openai/codex` | CLI | Apache 2.0 | Official OpenAI, sandbox-safe, fully open source | CLI tool, difficult to embed in Electron | ⭐⭐ |
| `@posthog/code-agent` | SDK | MIT | Unified interface, MCP Bridge support, streaming events | Depends on external CLI (Claude Code / Codex), adds complexity | ⭐⭐⭐ |
| `@quantum-ai/openclaude` | CLI Fork | MIT | Full Claude Code capability, supports any LLM | Based on leaked source code, stability concerns, legal risks | ⭐⭐ |
| `aider` | CLI | Apache 2.0 | Mature and stable, model-agnostic, Git-native | CLI tool, difficult to embed | ⭐⭐ |
| `@modelcontextprotocol/sdk` | SDK | MIT | Official MCP protocol implementation, supports stdio/SSE transports | Protocol layer only; Agent logic must be self-implemented | — (Auxiliary Library) |

### 1.2 Recommended Selection: Custom Agent Loop Approach

**Core Dependencies:**

```json
{
  "openai": "^7.3.0",
  "@modelcontextprotocol/sdk": "^1.21.0",
  "simple-git": "^3.27.0",
  "diff": "^7.0.0",
  "glob": "^11.0.0",
  "zod": "^3.24.0"
}
```

**Rationale:**

1. **Full Control** — Agent loop, tool definitions, and permission control are all in code, enabling precise adaptation to the Electron IPC architecture
2. **Flexible Customization** — Customizable Thinking Chain display, Workflow approval process, and Diff review
3. **Low Cost** — Only requires the OpenAI SDK (already installed in the project), no dependency on large CLI tools
4. **Consistent with Mainstream Architecture** — Codex CLI and Claude Code are both internally based on Tool Calling Agent Loops; our Node.js implementation follows the same pattern
5. **Zero Additional License Fees** — All libraries use MIT/BSD/Apache 2.0 licenses

---

## 2. Overall Architecture Design

### 2.1 Layered Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                     Electron Main Process                         │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                   Code Agent Engine                         │  │
│  │                                                              │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐  │  │
│  │  │  Agent Loop  │  │  Tool         │  │  MCP Manager    │  │  │
│  │  │  (Core Loop) │  │  Registry     │  │  (Dynamic MCP)  │  │  │
│  │  │              │  │  (Tool Reg.)  │  │                 │  │  │
│  │  └──────┬───────┘  └──────┬───────┘  └────────┬────────┘  │  │
│  │         │                 │                     │           │  │
│  │  ┌──────┴───────┐  ┌──────┴───────┐  ┌─────────┴────────┐ │  │
│  │  │  Skill       │  │  Sandbox     │  │  Diff Tracker    │ │  │
│  │  │  Manager     │  │  Executor    │  │  (Git-based)     │ │  │
│  │  │  (Skill Mgmt)│  │  (Sandbox Val)│  │  (Change Track)  │ │  │
│  │  └──────────────┘  └──────────────┘  └──────────────────┘ │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                │                                  │
│                       IPC Bridge Layer                            │
│                 (ipcMain.handle / event.sender.send)              │
│                                │                                  │
├────────────────────────────────┼──────────────────────────────────┤
│                      Renderer Process                             │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                     /code Page UI                            │  │
│  │                                                              │  │
│  │  ┌──────────┐  ┌──────────────┐  ┌──────────────────────┐  │  │
│  │  │  Sender  │  │  Thinking     │  │  Diff Viewer         │  │  │
│  │  │  (Input) │  │  Chain        │  │  (Code Diff Viewer)  │  │  │
│  │  │          │  │  (Thought Disp)│  │                      │  │  │
│  │  └──────────┘  └──────────────┘  └──────────────────────┘  │  │
│  │                                                              │  │
│  │  ┌──────────┐  ┌──────────────┐  ┌──────────────────────┐  │  │
│  │  │ Workflow │  │  Approval     │  │  Accept / Reject     │  │  │
│  │  │ Panel    │  │  Panel        │  │  All Buttons         │  │  │
│  │  │ (Workflow)│  │  (Approval)  │  │  (Accept/Reject All) │  │  │
│  │  └──────────┘  └──────────────┘  └──────────────────────┘  │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 Agent Workflow Four Phases

```
User Prompt: "Add a Redis cache layer to this project"
    │
    ▼
┌──────────────────────────────────────────────────────────────┐
│  Phase 1: Context Analysis (Context Analysis + Thought Display)│
│                                                               │
│  1. Read projectDir's package.json, directory structure       │
│  2. Analyze project tech stack (Node.js / Express / TypeScript)│
│  3. Understand user intent: add Redis cache                    │
│  4. [Streaming Display] Thinking Content to user              │
│     "Analyzing project structure... found this is an Express  │
│      + TypeScript project, uses dotenv for env vars. Need to  │
│      add ioredis dependency, create cache middleware, modify   │
│      existing routes..."                                      │
└──────────────────────────┬───────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  Phase 2: Workflow Planning (Planning + User Approval)        │
│                                                               │
│  AI generates a code modification plan:                       │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ Step 1: Install dependencies                            │ │
│  │   npm install ioredis                                    │ │
│  │                                                          │ │
│  │ Step 2: Create file src/config/redis.ts                  │ │
│  │   Redis connection config and singleton                  │ │
│  │                                                          │ │
│  │ Step 3: Create file src/middleware/cache.ts              │ │
│  │   Redis cache middleware                                 │ │
│  │                                                          │ │
│  │ Step 4: Modify file src/routes/user.ts                   │ │
│  │   Integrate cache middleware in getUser route            │ │
│  │                                                          │ │
│  │ Step 5: Modify file .env.example                         │ │
│  │   Add REDIS_URL environment variable                     │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
│  User clicks [Allow] or [Deny]                               │
└──────────────────────────┬───────────────────────────────────┘
                           ▼ (User Allow)
┌──────────────────────────────────────────────────────────────┐
│  Phase 3: Code Execution (Code Execution + Real-time Progress)│
│                                                               │
│  Agent executes step-by-step via Tool Calling:                │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ [1/5] 🔧 run_command: npm install ioredis        ✓ Done │ │
│  │ [2/5] ✏️  write_file: src/config/redis.ts         ✓ Done │ │
│  │ [3/5] ✏️  write_file: src/middleware/cache.ts     ✓ Done │ │
│  │ [4/5] ✏️  edit_file: src/routes/user.ts           ✓ Done │ │
│  │ [5/5] ✏️  edit_file: .env.example                  ✓ Done │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
│  Sandbox verification: npx tsc --noEmit → ✓ TypeScript passes │
└──────────────────────────┬───────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  Phase 4: Diff Review (Diff Review + Accept/Reject)          │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ 📁 Modified Files (5)                                    │ │
│  │                                                          │ │
│  │  📄 package.json          +2 lines  (dependencies)       │ │
│  │  📄 src/config/redis.ts   +25 lines (new file)           │ │
│  │  📄 src/middleware/cache.ts +45 lines (new file)         │ │
│  │  📄 src/routes/user.ts    +8/-3 lines                   │ │
│  │  📄 .env.example          +1 line                       │ │
│  │                                                          │ │
│  │  ┌──────────────────────────────────────────────────┐   │ │
│  │  │  [View Diff] Each file can be expanded to view   │   │ │
│  │  │  specific differences                            │   │ │
│  │  └──────────────────────────────────────────────────┘   │ │
│  │                                                          │ │
│  │  [✅ Accept All]  [❌ Reject All]                        │ │
│  └─────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. Agent Loop Core Design

### 3.1 Core Class: `CodeAgentLoop`

File path: `src/main/code-agent/agent-loop.ts`

```typescript
import OpenAI from 'openai';
import type { ChatCompletionMessageParam } from 'openai/resources/chat/completions';
import { ToolRegistry } from './tool-registry';
import { GitDiffTracker } from './diff-tracker';
import { SandboxExecutor } from './sandbox';
import type {
  AgentConfig,
  AgentCallbacks,
  AgentResult,
  WorkflowStep,
  FileDiff,
  ToolCallRecord,
} from './types';

/**
 * Code Agent core loop.
 * Implements multi-turn conversation + tool execution based on OpenAI Tool Calling.
 */
export class CodeAgentLoop {
  private openai: OpenAI;
  private toolRegistry: ToolRegistry;
  private diffTracker: GitDiffTracker;
  private sandbox: SandboxExecutor;
  private config: AgentConfig;
  private toolCallHistory: ToolCallRecord[] = [];

  constructor(config: AgentConfig) {
    this.config = config;
    this.openai = new OpenAI({
      apiKey: config.apiKey,
      baseURL: config.baseURL,
      timeout: config.timeout || 120000,
      maxRetries: config.maxRetries || 0,
      dangerouslyAllowBrowser: false,
    });
    this.toolRegistry = new ToolRegistry();
    this.diffTracker = new GitDiffTracker(config.projectDir);
    this.sandbox = new SandboxExecutor();
  }

  /**
   * Execute the complete Agent workflow.
   *
   * @param prompt - The user input prompt
   * @param callbacks - Streaming callbacks (thinking / content / tool-use / progress / workflow / diff)
   * @returns AgentResult - Contains diff list and execution summary
   */
  async run(prompt: string, callbacks: AgentCallbacks): Promise<AgentResult> {
    // 1. Initialize Git tracking
    await this.diffTracker.initialize();

    // 2. Build system prompt
    const systemPrompt = this.buildSystemPrompt();

    // 3. Build message list
    const messages: ChatCompletionMessageParam[] = [
      { role: 'system', content: systemPrompt },
      { role: 'user', content: prompt },
    ];

    // 4. Agent Loop: multi-turn Tool Calling
    let fullResponse = '';
    let fullThinking = '';

    while (true) {
      const stream = await this.openai.chat.completions.create({
        model: this.config.model,
        messages,
        tools: this.toolRegistry.getOpenAIToolDefinitions(),
        tool_choice: 'auto',
        stream: true,
      });

      // Collect streaming responses
      let content = '';
      let thinking = '';
      let toolCalls: Map<number, { id: string; name: string; args: string }> = new Map();

      for await (const chunk of stream) {
        const delta = chunk.choices?.[0]?.delta;
        if (!delta) continue;

        // Thinking chain (DeepSeek R1, Claude extended thinking, etc.)
        // eslint-disable-next-line @typescript-eslint/no-explicit-any
        const reasoning = (delta as any).reasoning_content as string | undefined;
        if (reasoning) {
          thinking += reasoning;
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

      // If no tool calls, end the Agent loop
      if (toolCalls.size === 0) {
        break;
      }

      // Add assistant message (with tool_calls) to message list
      const assistantMsg: ChatCompletionMessageParam = {
        role: 'assistant',
        content: content || null,
        tool_calls: Array.from(toolCalls.entries()).map(([idx, tc]) => ({
          id: tc.id,
          type: 'function' as const,
          function: { name: tc.name, arguments: tc.args },
        })),
      };
      messages.push(assistantMsg);

      // Execute all tool calls
      for (const [, tc] of toolCalls) {
        callbacks.onToolUse({ toolName: tc.name, args: tc.args, status: 'start' });

        try {
          const result = await this.toolRegistry.execute(tc.name, JSON.parse(tc.args));
          this.toolCallHistory.push({
            toolName: tc.name,
            args: tc.args,
            result: typeof result === 'string' ? result : JSON.stringify(result),
            timestamp: Date.now(),
            success: true,
          });

          // Track file changes
          if (['write_file', 'edit_file'].includes(tc.name)) {
            const args = JSON.parse(tc.args);
            this.diffTracker.trackFileChange(args.filePath);
          }

          callbacks.onToolUse({ toolName: tc.name, args: tc.args, status: 'done', result });

          messages.push({
            role: 'tool',
            tool_call_id: tc.id,
            content: typeof result === 'string' ? result : JSON.stringify(result),
          });
        } catch (error) {
          const errMsg = error instanceof Error ? error.message : String(error);
          this.toolCallHistory.push({
            toolName: tc.name,
            args: tc.args,
            result: errMsg,
            timestamp: Date.now(),
            success: false,
          });

          callbacks.onToolUse({ toolName: tc.name, args: tc.args, status: 'error', error: errMsg });

          messages.push({
            role: 'tool',
            tool_call_id: tc.id,
            content: `Error: ${errMsg}`,
          });
        }
      }
    }

    // 5. Sandbox verification
    const validationResult = await this.sandbox.validateChanges(
      this.config.projectDir,
      this.diffTracker.getChangedFiles(),
    );

    // 6. Collect Diff results
    const diffs = await this.diffTracker.collectDiffs();

    return {
      thinking: fullThinking,
      response: fullResponse,
      diffs,
      validation: validationResult,
      toolCalls: this.toolCallHistory,
    };
  }

  /**
   * Apply all modifications (accept).
   */
  async applyAll(): Promise<void> {
    // Modifications are already written to disk; just confirm Git status
    await this.diffTracker.finalize();
  }

  /**
   * Revert all modifications.
   */
  async revertAll(): Promise<void> {
    await this.diffTracker.revertAll();
  }

  /**
   * Build system prompt, including project context and behavioral constraints.
   */
  private buildSystemPrompt(): string {
    return `You are an expert coding agent working in the project at: ${this.config.projectDir}

## Your Capabilities
You have access to the following tools to read, write, and modify code:
- read_file: Read file contents
- write_file: Create or overwrite a file
- edit_file: Perform exact string replacements in a file
- glob_files: Find files matching glob patterns
- grep_search: Search file contents with regex
- run_command: Execute shell commands in the project directory
- list_directory: List directory contents

## Workflow Requirements
When the user asks you to make code changes, you MUST follow this process:

### Phase 1: Analysis
1. First, understand the project structure by reading key files (package.json, tsconfig.json, main entry points)
2. Analyze the user's intent and what needs to be changed
3. Output your analysis as a clear thinking process

### Phase 2: Planning
1. Create a detailed step-by-step plan for the code changes
2. Each step should specify: action (create/edit/delete), file path, and what changes
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
- NEVER make changes before the user has approved the plan
- Always read a file before editing it (to ensure you have the latest content)
- Use edit_file for modifications (it's safer than write_file for existing files)
- Run validation commands after making changes (e.g., npm run lint, npx tsc --noEmit)
- If you encounter errors, analyze and fix them before proceeding
- Respect the project's existing code style and conventions
- Use relative paths from the project root`;
  }
}
```

### 3.2 Type Definitions

File path: `src/main/code-agent/types.ts`

```typescript
/**
 * Agent configuration
 */
export interface AgentConfig {
  /** API Key */
  apiKey: string;
  /** API Base URL (optional, for custom endpoints) */
  baseURL?: string;
  /** Model name */
  model: string;
  /** Project directory */
  projectDir: string;
  /** Request timeout (ms) */
  timeout?: number;
  /** Maximum retry count */
  maxRetries?: number;
}

/**
 * Streaming callback interface
 */
export interface AgentCallbacks {
  /** Thinking chain content streaming callback */
  onThinking: (content: string) => void;
  /** AI response content streaming callback */
  onContent: (content: string) => void;
  /** Tool call status callback */
  onToolUse: (event: ToolUseEvent) => void;
}

/**
 * Tool call event
 */
export interface ToolUseEvent {
  toolName: string;
  args: string;
  status: 'start' | 'done' | 'error';
  result?: string;
  error?: string;
}

/**
 * Tool call record
 */
export interface ToolCallRecord {
  toolName: string;
  args: string;
  result: string;
  timestamp: number;
  success: boolean;
}

/**
 * Workflow step
 */
export interface WorkflowStep {
  /** Step number */
  index: number;
  /** Operation type */
  action: 'create' | 'edit' | 'delete' | 'run_command';
  /** File path */
  filePath: string;
  /** Operation description */
  description: string;
}

/**
 * File diff
 */
export interface FileDiff {
  /** File path (relative to project root) */
  filePath: string;
  /** Operation type */
  action: 'added' | 'modified' | 'deleted';
  /** Unified diff text */
  diff: string;
  /** Addition line count */
  additions: number;
  /** Deletion line count */
  deletions: number;
}

/**
 * Sandbox validation result
 */
export interface ValidationResult {
  /** Whether validation passed */
  passed: boolean;
  /** List of validation commands */
  commands: string[];
  /** Output for each command */
  outputs: { command: string; stdout: string; stderr: string; exitCode: number }[];
  /** Error messages */
  errors: string[];
}

/**
 * Agent execution result
 */
export interface AgentResult {
  /** Thinking chain content */
  thinking: string;
  /** AI response content */
  response: string;
  /** File diff list */
  diffs: FileDiff[];
  /** Sandbox validation result */
  validation: ValidationResult;
  /** Tool call history */
  toolCalls: ToolCallRecord[];
}

/**
 * MCP server configuration
 */
export interface MCPServerConfig {
  /** Unique name */
  name: string;
  /** Launch command */
  command: string;
  /** Command arguments */
  args?: string[];
  /** Environment variables */
  env?: Record<string, string>;
  /** Whether to auto-approve tools */
  autoApprove?: boolean;
}

/**
 * Custom Skill definition
 */
export interface SkillDefinition {
  /** Skill name */
  name: string;
  /** Description */
  description: string;
  /** Instructions injected into System Prompt */
  systemPromptExtension: string;
  /** Optional: Skill-specific tool definitions */
  tools?: ToolDefinition[];
}

/**
 * Tool definition (OpenAI Function Calling format)
 */
export interface ToolDefinition {
  type: 'function';
  function: {
    name: string;
    description: string;
    parameters: Record<string, unknown>;
  };
}
```

---

## 4. Tool System

### 4.1 Tool Registry

File path: `src/main/code-agent/tool-registry.ts`

```typescript
import type { ToolDefinition } from './types';
import { builtinTools } from './tools';

/**
 * Tool registry.
 * Manages all tools available to the Agent, including built-in tools and MCP dynamically registered tools.
 */
export class ToolRegistry {
  private tools: Map<string, {
    definition: ToolDefinition;
    handler: (args: Record<string, unknown>) => Promise<string>;
  }> = new Map();

  constructor() {
    // Register built-in tools
    for (const tool of builtinTools) {
      this.register(tool.definition, tool.handler);
    }
  }

  /**
   * Register a tool.
   */
  register(definition: ToolDefinition, handler: (args: Record<string, unknown>) => Promise<string>): void {
    this.tools.set(definition.function.name, { definition, handler });
  }

  /**
   * Register tools from MCP.
   */
  registerMCPTools(mcpTools: Array<{ definition: ToolDefinition; handler: (args: Record<string, unknown>) => Promise<string> }>): void {
    for (const tool of mcpTools) {
      // Prefix MCP tool name to avoid conflicts
      const mcpDef = {
        ...tool.definition,
        function: {
          ...tool.definition.function,
          name: `mcp__${tool.definition.function.name}`,
        },
      };
      this.register(mcpDef, tool.handler);
    }
  }

  /**
   * Get the tool definition list in OpenAI Tool Calling format.
   */
  getOpenAIToolDefinitions(): ToolDefinition[] {
    return Array.from(this.tools.values()).map(t => t.definition);
  }

  /**
   * Execute a tool.
   */
  async execute(name: string, args: Record<string, unknown>): Promise<string> {
    const tool = this.tools.get(name);
    if (!tool) {
      throw new Error(`Tool not found: ${name}`);
    }
    return tool.handler(args);
  }

  /**
   * Get the tool count.
   */
  get size(): number {
    return this.tools.size;
  }
}
```

### 4.2 Built-in Tool Implementations

File path: `src/main/code-agent/tools/index.ts`

```typescript
import * as fs from 'fs/promises';
import * as path from 'path';
import { exec } from 'child_process';
import { promisify } from 'util';
import { glob } from 'glob';
import type { ToolDefinition } from '../types';

const execAsync = promisify(exec);

/**
 * Built-in tool list.
 * Each tool includes an OpenAI-format definition and Node.js implementation.
 */
export const builtinTools: Array<{
  definition: ToolDefinition;
  handler: (args: Record<string, unknown>) => Promise<string>;
}> = [
  // ==================== read_file ====================
  {
    definition: {
      type: 'function',
      function: {
        name: 'read_file',
        description: 'Read the contents of a file. Returns the file content with line numbers.',
        parameters: {
          type: 'object',
          properties: {
            filePath: {
              type: 'string',
              description: 'Path to the file, relative to the project root',
            },
            startLine: {
              type: 'number',
              description: 'Optional: start line number (1-based)',
            },
            endLine: {
              type: 'number',
              description: 'Optional: end line number (1-based, inclusive)',
            },
          },
          required: ['filePath'],
        },
      },
    },
    handler: async (args) => {
      const { filePath, startLine, endLine } = args as {
        filePath: string;
        startLine?: number;
        endLine?: number;
      };
      const content = await fs.readFile(filePath, 'utf-8');
      const lines = content.split('\n');

      if (startLine && endLine) {
        const sliced = lines.slice(startLine - 1, endLine);
        return sliced.map((l, i) => `${startLine + i}: ${l}`).join('\n');
      }

      return lines.map((l, i) => `${i + 1}: ${l}`).join('\n');
    },
  },

  // ==================== write_file ====================
  {
    definition: {
      type: 'function',
      function: {
        name: 'write_file',
        description: 'Create a new file or overwrite an existing file with the given content.',
        parameters: {
          type: 'object',
          properties: {
            filePath: {
              type: 'string',
              description: 'Path to the file, relative to the project root',
            },
            content: {
              type: 'string',
              description: 'The complete content to write to the file',
            },
          },
          required: ['filePath', 'content'],
        },
      },
    },
    handler: async (args) => {
      const { filePath, content } = args as { filePath: string; content: string };
      const dir = path.dirname(filePath);
      await fs.mkdir(dir, { recursive: true });
      await fs.writeFile(filePath, content, 'utf-8');
      return `File written successfully: ${filePath} (${content.split('\n').length} lines)`;
    },
  },

  // ==================== edit_file ====================
  {
    definition: {
      type: 'function',
      function: {
        name: 'edit_file',
        description:
          'Perform exact string replacement in a file. The old_string must match exactly, including whitespace and indentation. If old_string is not unique in the file, the edit will fail.',
        parameters: {
          type: 'object',
          properties: {
            filePath: {
              type: 'string',
              description: 'Path to the file, relative to the project root',
            },
            oldString: {
              type: 'string',
              description: 'The exact text to replace (must match exactly)',
            },
            newString: {
              type: 'string',
              description: 'The text to replace it with',
            },
          },
          required: ['filePath', 'oldString', 'newString'],
        },
      },
    },
    handler: async (args) => {
      const { filePath, oldString, newString } = args as {
        filePath: string;
        oldString: string;
        newString: string;
      };
      const content = await fs.readFile(filePath, 'utf-8');

      if (!content.includes(oldString)) {
        throw new Error(`oldString not found in ${filePath}. Make sure it matches exactly, including indentation.`);
      }

      const occurrences = content.split(oldString).length - 1;
      if (occurrences > 1) {
        throw new Error(
          `oldString appears ${occurrences} times in ${filePath}. It must be unique. Provide more context to make it unique.`,
        );
      }

      const newContent = content.replace(oldString, newString);
      await fs.writeFile(filePath, newContent, 'utf-8');
      return `File edited successfully: ${filePath}`;
    },
  },

  // ==================== glob_files ====================
  {
    definition: {
      type: 'function',
      function: {
        name: 'glob_files',
        description: 'Find files matching a glob pattern. Useful for discovering project structure.',
        parameters: {
          type: 'object',
          properties: {
            pattern: {
              type: 'string',
              description: 'Glob pattern, e.g. "src/**/*.ts" or "*.json"',
            },
            cwd: {
              type: 'string',
              description: 'Optional: working directory for the glob (defaults to project root)',
            },
          },
          required: ['pattern'],
        },
      },
    },
    handler: async (args) => {
      const { pattern, cwd } = args as { pattern: string; cwd?: string };
      const files = await glob(pattern, {
        cwd: cwd || process.cwd(),
        ignore: ['**/node_modules/**', '**/dist/**', '**/.git/**'],
        nodir: true,
      });
      return files.length > 0 ? files.join('\n') : 'No files found matching the pattern.';
    },
  },

  // ==================== grep_search ====================
  {
    definition: {
      type: 'function',
      function: {
        name: 'grep_search',
        description: 'Search file contents using a regular expression. Returns matching lines with file paths.',
        parameters: {
          type: 'object',
          properties: {
            pattern: {
              type: 'string',
              description: 'Regular expression pattern to search for',
            },
            include: {
              type: 'string',
              description: 'Optional: glob pattern to filter files (e.g. "*.ts")',
            },
            path: {
              type: 'string',
              description: 'Optional: subdirectory to search in (defaults to project root)',
            },
          },
          required: ['pattern'],
        },
      },
    },
    handler: async (args) => {
      const { pattern, include, path: searchPath } = args as {
        pattern: string;
        include?: string;
        path?: string;
      };
      const { stdout } = await execAsync(
        `grep -rn --include="${include || '*'}" "${pattern.replace(/"/g, '\\"')}" ${searchPath || '.'}`,
        { cwd: process.cwd(), maxBuffer: 10 * 1024 * 1024 },
      );
      return stdout || 'No matches found.';
    },
  },

  // ==================== run_command ====================
  {
    definition: {
      type: 'function',
      function: {
        name: 'run_command',
        description:
          'Execute a shell command in the project directory. Use for installing dependencies, running tests, linting, etc.',
        parameters: {
          type: 'object',
          properties: {
            command: {
              type: 'string',
              description: 'The shell command to execute',
            },
            cwd: {
              type: 'string',
              description: 'Optional: working directory for the command (defaults to project root)',
            },
          },
          required: ['command'],
        },
      },
    },
    handler: async (args) => {
      const { command, cwd } = args as { command: string; cwd?: string };
      const timeout = 60000; // 60 seconds max
      const { stdout, stderr } = await execAsync(command, {
        cwd: cwd || process.cwd(),
        timeout,
        maxBuffer: 10 * 1024 * 1024,
      });
      return JSON.stringify({ stdout: stdout.slice(0, 5000), stderr: stderr.slice(0, 5000) });
    },
  },

  // ==================== list_directory ====================
  {
    definition: {
      type: 'function',
      function: {
        name: 'list_directory',
        description: 'List the contents of a directory. Shows files and subdirectories.',
        parameters: {
          type: 'object',
          properties: {
            dirPath: {
              type: 'string',
              description: 'Path to the directory, relative to the project root. Use "." for root.',
            },
            recursive: {
              type: 'boolean',
              description: 'Whether to list recursively (default: false). Max depth: 3.',
            },
          },
          required: ['dirPath'],
        },
      },
    },
    handler: async (args) => {
      const { dirPath, recursive } = args as { dirPath: string; recursive?: boolean };

      if (recursive) {
        const files = await glob('**/*', {
          cwd: dirPath,
          ignore: ['**/node_modules/**', '**/dist/**', '**/.git/**'],
          nodir: true,
          maxDepth: 3,
        });
        return files.length > 0 ? files.join('\n') : 'Directory is empty.';
      }

      const entries = await fs.readdir(dirPath, { withFileTypes: true });
      return entries
        .map((e) => `${e.isDirectory() ? '📁' : '📄'} ${e.name}`)
        .join('\n');
    },
  },
];
```

---

## 5. MCP Manager (Dynamic Extension)

### 5.1 Core Class: `MCPManager`

File path: `src/main/code-agent/mcp-manager.ts`

```typescript
import { Client } from '@modelcontextprotocol/sdk/client/index.js';
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js';
import type { MCPServerConfig, ToolDefinition } from './types';
import type { ToolRegistry } from './tool-registry';

/**
 * MCP (Model Context Protocol) Manager.
 * Supports dynamic loading of MCP npm packages and custom MCP Server processes.
 *
 * The MCP protocol allows the Agent to use external tools, for example:
 * - Database operations (Postgres MCP Server)
 * - Browser automation (Playwright MCP Server)
 * - File system operations (Filesystem MCP Server)
 * - API integration (GitHub MCP Server)
 */
export class MCPManager {
  private servers: Map<string, {
    config: MCPServerConfig;
    client: Client;
    transport: StdioClientTransport;
  }> = new Map();

  private toolRegistry: ToolRegistry;

  constructor(toolRegistry: ToolRegistry) {
    this.toolRegistry = toolRegistry;
  }

  /**
   * Load an MCP Server.
   * Starts an external process via stdio, establishes MCP connection, and registers its tools.
   *
   * @param config - MCP Server configuration
   *
   * @example
   * ```typescript
   * await mcpManager.loadServer({
   *   name: 'filesystem',
   *   command: 'npx',
   *   args: ['-y', '@modelcontextprotocol/server-filesystem', '/path/to/project'],
   * });
   * ```
   */
  async loadServer(config: MCPServerConfig): Promise<void> {
    const transport = new StdioClientTransport({
      command: config.command,
      args: config.args || [],
      env: config.env,
    });

    const client = new Client(
      { name: 'code-agent', version: '1.0.0' },
      { capabilities: { tools: {} } },
    );

    await client.connect(transport);

    // Get the list of tools provided by the MCP Server
    const { tools } = await client.listTools();

    // Register tools to ToolRegistry (with mcp__ prefix)
    const mcpTools = tools.map((tool) => ({
      definition: {
        type: 'function' as const,
        function: {
          name: `mcp__${tool.name}`,
          description: tool.description || `MCP tool: ${tool.name}`,
          parameters: tool.inputSchema as Record<string, unknown>,
        },
      },
      handler: async (args: Record<string, unknown>) => {
        const result = await client.callTool({
          name: tool.name,
          arguments: args,
        });
        // Extract text content
        const textContents = result.content
          .filter((c): c is { type: 'text'; text: string } => c.type === 'text')
          .map((c) => c.text);
        return textContents.join('\n');
      },
    }));

    this.toolRegistry.registerMCPTools(mcpTools);

    this.servers.set(config.name, { config, client, transport });
    console.log(`[MCP] Loaded server: ${config.name} (${tools.length} tools)`);
  }

  /**
   * Unload an MCP Server.
   */
  async unloadServer(name: string): Promise<void> {
    const server = this.servers.get(name);
    if (!server) return;
    await server.client.close();
    this.servers.delete(name);
    console.log(`[MCP] Unloaded server: ${name}`);
  }

  /**
   * Unload all MCP Servers.
   */
  async unloadAll(): Promise<void> {
    for (const [name] of this.servers) {
      await this.unloadServer(name);
    }
  }

  /**
   * Get the list of loaded MCP Servers.
   */
  getLoadedServers(): string[] {
    return Array.from(this.servers.keys());
  }
}
```

### 5.2 MCP Server Configuration Example

Users can configure custom MCP Servers through the UI:

```json
{
  "mcpServers": [
    {
      "name": "github",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxx"
      }
    },
    {
      "name": "postgres",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost/mydb"]
    },
    {
      "name": "playwright",
      "command": "npx",
      "args": ["-y", "@playwright/mcp"]
    }
  ]
}
```

---

## 6. Skill Manager (Custom Skills)

### 6.1 Core Class: `SkillManager`

File path: `src/main/code-agent/skill-manager.ts`

```typescript
import type { SkillDefinition } from './types';
import type { ToolRegistry } from './tool-registry';

/**
 * Skill Manager.
 * Skills are predefined code modification patterns that guide Agent behavior through extending the System Prompt.
 *
 * Difference from MCP:
 * - MCP provides external tools (databases, browsers, APIs, etc.)
 * - Skills provide code modification patterns (React component generation, API endpoint creation, test writing, etc.)
 *
 * @example
 * ```typescript
 * skillManager.registerSkill({
 *   name: 'react-component',
 *   description: 'Generate a React component with TypeScript and CSS modules',
 *   systemPromptExtension: `
 *     When creating a React component:
 *     1. Use functional components with TypeScript interfaces
 *     2. Export as default, co-locate CSS modules
 *     3. Use React.memo() for performance
 *     4. Add JSDoc comments for props
 *   `,
 * });
 * ```
 */
export class SkillManager {
  private skills: Map<string, SkillDefinition> = new Map();
  private toolRegistry: ToolRegistry;

  constructor(toolRegistry: ToolRegistry) {
    this.toolRegistry = toolRegistry;
  }

  /**
   * Register a Skill.
   */
  registerSkill(skill: SkillDefinition): void {
    this.skills.set(skill.name, skill);

    // If the Skill has custom tools, register them
    if (skill.tools && skill.tools.length > 0) {
      for (const toolDef of skill.tools) {
        // Skill tools do not register handlers for now (user needs to provide)
        // Actual usage can be handled through Skill hooks
        this.toolRegistry.register(toolDef, async (args) => {
          return `Skill "${skill.name}" tool "${toolDef.function.name}" called with: ${JSON.stringify(args)}`;
        });
      }
    }

    console.log(`[Skill] Registered: ${skill.name}`);
  }

  /**
   * Unregister a Skill.
   */
  unregisterSkill(name: string): void {
    this.skills.delete(name);
  }

  /**
   * Get the System Prompt extension for all registered Skills (merged).
   */
  getSystemPromptExtension(): string {
    if (this.skills.size === 0) return '';

    const sections: string[] = ['\n## Active Skills\n'];
    for (const [name, skill] of this.skills) {
      sections.push(`### ${name}: ${skill.description}`);
      sections.push(skill.systemPromptExtension);
      sections.push('');
    }
    return sections.join('\n');
  }

  /**
   * Get the list of all registered Skills.
   */
  getSkills(): Array<{ name: string; description: string }> {
    return Array.from(this.skills.values()).map((s) => ({
      name: s.name,
      description: s.description,
    }));
  }
}
```

### 6.2 Built-in Skill Examples

```typescript
/**
 * Built-in Skill definitions.
 * Users can enable/disable these Skills through the UI.
 */
export const builtinSkills: SkillDefinition[] = [
  {
    name: 'typescript-strict',
    description: 'Enforce strict TypeScript patterns',
    systemPromptExtension: `
When writing TypeScript code:
1. Always define explicit types for function parameters and return values
2. Use interfaces over type aliases for object shapes
3. Avoid using 'any' - use 'unknown' instead
4. Use readonly modifiers for immutable data
5. Prefer const assertions over type casting
6. Use discriminated unions for state management`,
  },
  {
    name: 'react-patterns',
    description: 'Follow React best practices',
    systemPromptExtension: `
When writing React code:
1. Use functional components with hooks
2. Co-locate styles with components (CSS modules or CSS-in-JS)
3. Use React.memo() for pure components
4. Extract custom hooks for reusable logic
5. Use useCallback/useMemo for expensive computations
6. Add proper accessibility attributes (aria-labels, roles)`,
  },
  {
    name: 'testing',
    description: 'Generate tests alongside code changes',
    systemPromptExtension: `
When making code changes:
1. Always consider test coverage
2. If modifying a function, update its corresponding tests
3. If creating a new function, suggest test cases
4. Use describe/it blocks for organization
5. Test edge cases: null, undefined, empty arrays, boundary values`,
  },
];
```

---

## 7. Sandbox Validator

### 7.1 Core Class: `SandboxExecutor`

File path: `src/main/code-agent/sandbox.ts`

```typescript
import * as fs from 'fs/promises';
import * as path from 'path';
import * as os from 'os';
import { exec } from 'child_process';
import { promisify } from 'util';
import { glob } from 'glob';
import type { ValidationResult } from './types';

const execAsync = promisify(exec);

/**
 * Sandbox Validator.
 * Validates the correctness of code modifications in an isolated temporary directory.
 *
 * Validation strategy:
 * 1. Copy project files to a temporary sandbox directory
 * 2. Apply code modifications
 * 3. Run validation commands (TypeScript compilation, ESLint, tests, etc.)
 * 4. Report results
 * 5. Clean up sandbox
 */
export class SandboxExecutor {
  private sandboxDir: string | null = null;

  /**
   * Validate code modifications in a sandbox.
   *
   * @param projectDir - Project directory
   * @param changedFiles - List of changed file paths
   * @returns Validation result
   */
  async validateChanges(
    projectDir: string,
    changedFiles: string[],
  ): Promise<ValidationResult> {
    const commands: string[] = [];
    const outputs: ValidationResult['outputs'] = [];
    const errors: string[] = [];

    // Detect project type and determine validation commands
    const packageJsonPath = path.join(projectDir, 'package.json');
    let hasPackageJson = false;

    try {
      await fs.access(packageJsonPath);
      hasPackageJson = true;
    } catch {
      // No package.json, skip npm-based validation
    }

    if (hasPackageJson) {
      const packageJson = JSON.parse(await fs.readFile(packageJsonPath, 'utf-8'));
      const scripts = packageJson.scripts || {};

      // Detect TypeScript
      const tsFiles = changedFiles.filter((f) => f.endsWith('.ts') || f.endsWith('.tsx'));
      if (tsFiles.length > 0) {
        // Check for TypeScript
        const hasTsConfig = await this.fileExists(path.join(projectDir, 'tsconfig.json'));
        if (hasTsConfig) {
          if (scripts['type-check'] || scripts['typecheck']) {
            commands.push(`npm run ${scripts['type-check'] ? 'type-check' : 'typecheck'}`);
          } else if (scripts['build']) {
            // Use build's pre-check
            commands.push('npx tsc --noEmit');
          }
        }
      }

      // Lint check
      if (scripts['lint']) {
        commands.push('npm run lint');
      }

      // Tests
      if (scripts['test']) {
        commands.push('npm run test -- --passWithNoTests');
      }
    }

    // If no validation commands detected, at minimum verify file syntax
    if (commands.length === 0) {
      commands.push('echo "No validation commands configured. Skipping sandbox verification."');
    }

    // Execute validation commands
    for (const command of commands) {
      try {
        const { stdout, stderr } = await execAsync(command, {
          cwd: projectDir,
          timeout: 120000,
          maxBuffer: 10 * 1024 * 1024,
        });
        outputs.push({ command, stdout, stderr, exitCode: 0 });
      } catch (error: unknown) {
        const execError = error as { stdout?: string; stderr?: string; code?: number };
        outputs.push({
          command,
          stdout: execError.stdout || '',
          stderr: execError.stderr || '',
          exitCode: execError.code || 1,
        });
        errors.push(`Command "${command}" failed with exit code ${execError.code}`);
      }
    }

    return {
      passed: errors.length === 0,
      commands,
      outputs,
      errors,
    };
  }

  /**
   * Create sandbox directory (copy project files).
   */
  private async createSandbox(projectDir: string): Promise<string> {
    const sandboxDir = path.join(os.tmpdir(), `code-agent-sandbox-${Date.now()}`);
    await fs.mkdir(sandboxDir, { recursive: true });

    // Copy project files (excluding node_modules, .git, dist)
    const files = await glob('**/*', {
      cwd: projectDir,
      ignore: ['**/node_modules/**', '**/.git/**', '**/dist/**', '**/build/**'],
      nodir: true,
    });

    for (const file of files) {
      const src = path.join(projectDir, file);
      const dest = path.join(sandboxDir, file);
      await fs.mkdir(path.dirname(dest), { recursive: true });
      await fs.copyFile(src, dest);
    }

    this.sandboxDir = sandboxDir;
    return sandboxDir;
  }

  /**
   * Clean up sandbox directory.
   */
  async cleanup(): Promise<void> {
    if (this.sandboxDir) {
      await fs.rm(this.sandboxDir, { recursive: true, force: true });
      this.sandboxDir = null;
    }
  }

  private async fileExists(filePath: string): Promise<boolean> {
    try {
      await fs.access(filePath);
      return true;
    } catch {
      return false;
    }
  }
}
```

---

## 8. Git Diff Tracker

### 8.1 Core Class: `GitDiffTracker`

File path: `src/main/code-agent/diff-tracker.ts`

```typescript
import simpleGit, { SimpleGit } from 'simple-git';
import * as path from 'path';
import type { FileDiff } from './types';

/**
 * Git Diff Tracker.
 * Tracks file changes based on Git, supporting:
 * - Recording the Git state before modification
 * - Tracking the list of changed files
 * - Generating unified diff
 * - Reverting all modifications
 */
export class GitDiffTracker {
  private git: SimpleGit;
  private projectDir: string;
  private changedFiles: Set<string> = new Set();
  private beforeHash: string = '';

  constructor(projectDir: string) {
    this.projectDir = projectDir;
    this.git = simpleGit(projectDir);
  }

  /**
   * Initialize the tracker, record current Git HEAD.
   */
  async initialize(): Promise<void> {
    try {
      this.beforeHash = await this.git.revparse(['HEAD']);
    } catch {
      // May not have a Git repository, use empty state
      this.beforeHash = '';
    }
  }

  /**
   * Track a file change.
   */
  trackFileChange(filePath: string): void {
    // Convert to relative path
    const relativePath = path.isAbsolute(filePath)
      ? path.relative(this.projectDir, filePath)
      : filePath;
    this.changedFiles.add(relativePath);
  }

  /**
   * Get the list of changed files.
   */
  getChangedFiles(): string[] {
    return Array.from(this.changedFiles);
  }

  /**
   * Collect diffs for all changed files.
   */
  async collectDiffs(): Promise<FileDiff[]> {
    const diffs: FileDiff[] = [];

    for (const file of this.changedFiles) {
      try {
        // Check file status
        const status = await this.git.status([file]);

        let action: FileDiff['action'] = 'modified';
        if (status.not_added?.includes(file) || status.deleted?.includes(file)) {
          action = 'deleted';
        } else if (status.created?.includes(file)) {
          action = 'added';
        }

        // Get diff
        let diffText = '';
        try {
          diffText = await this.git.diff([file]);
          if (!diffText) {
            // For new files, use --cached
            diffText = await this.git.diff(['--cached', file]);
          }
        } catch {
          diffText = `(Unable to generate diff for ${file})`;
        }

        // Count additions/deletions
        const additions = (diffText.match(/^\+(?!\+\+)/gm) || []).length;
        const deletions = (diffText.match(/^-(?!--)/gm) || []).length;

        diffs.push({
          filePath: file,
          action,
          diff: diffText,
          additions,
          deletions,
        });
      } catch {
        // File may have been deleted or not found
        diffs.push({
          filePath: file,
          action: 'deleted',
          diff: `(File not found: ${file})`,
          additions: 0,
          deletions: 0,
        });
      }
    }

    return diffs;
  }

  /**
   * Revert all modifications (restore to pre-modification state).
   */
  async revertAll(): Promise<void> {
    for (const file of this.changedFiles) {
      try {
        // Check if file is newly created
        const status = await this.git.status([file]);
        if (status.created.includes(file)) {
          // New file: delete directly
          const { promises: fs } = await import('fs');
          await fs.unlink(path.join(this.projectDir, file));
        } else {
          // Existing file: checkout to restore
          await this.git.checkout(['--', file]);
        }
      } catch {
        console.warn(`[DiffTracker] Failed to revert: ${file}`);
      }
    }
    this.changedFiles.clear();
  }

  /**
   * Confirm modifications (stage all changes).
   */
  async finalize(): Promise<void> {
    for (const file of this.changedFiles) {
      try {
        await this.git.add(file);
      } catch {
        console.warn(`[DiffTracker] Failed to stage: ${file}`);
      }
    }
  }
}
```

---

## 9. IPC Communication Design

### 9.1 IPC Channels

#### Renderer → Main (send / invoke)

| Channel | Type | Payload | Description |
|---------|------|---------|-------------|
| `code:agent-start` | send | `{ modelConfigId, prompt, projectDir, sessionId }` | Start Agent |
| `code:agent-approve` | send | `{ sessionId, approved: boolean }` | User approval workflow |
| `code:agent-apply` | send | `{ sessionId, action: 'accept' \| 'reject' }` | Apply/revert modifications |

#### Main → Renderer (event.sender.send)

| Channel | Payload | Description |
|---------|---------|-------------|
| `code:agent-thinking` | `{ content: string }` | Thinking chain content streaming push |
| `code:agent-response` | `{ content: string }` | AI response content streaming push |
| `code:agent-tool-use` | `{ toolName, args, status, result?, error? }` | Tool call status |
| `code:agent-workflow` | `{ steps: WorkflowStep[], summary: string }` | Workflow plan |
| `code:agent-progress` | `{ currentStep, totalSteps, filePath }` | Execution progress |
| `code:agent-diff` | `{ files: FileDiff[] }` | Diff results |
| `code:agent-validation` | `{ result: ValidationResult }` | Sandbox validation results |
| `code:agent-done` | `{ summary: string }` | Agent completed |
| `code:agent-error` | `{ error: string, phase: string }` | Error message |

### 9.2 preload.ts Update

```typescript
// New send channels
const validSendChannels = [
  'to-main', 'app:action',
  'chat:send-stream', 'code:send-stream',
  'code:agent-start', 'code:agent-approve', 'code:agent-apply',  // New
];

// New on channels
const validOnChannels = [
  'from-main', 'app:event',
  'chat:stream-chunk', 'chat:stream-done', 'chat:stream-error',
  'code:stream-chunk', 'code:stream-done', 'code:stream-error',
  // New Agent channels
  'code:agent-thinking', 'code:agent-response',
  'code:agent-tool-use', 'code:agent-workflow',
  'code:agent-progress', 'code:agent-diff',
  'code:agent-validation', 'code:agent-done', 'code:agent-error',
];
```

### 9.3 code-handlers.ts Update

```typescript
import { CodeAgentLoop } from './code-agent/agent-loop';
import { MCPManager } from './code-agent/mcp-manager';
import { SkillManager } from './code-agent/skill-manager';
import { builtinSkills } from './code-agent/skills';

// Store active Agent instances
const activeAgents = new Map<string, CodeAgentLoop>();

ipcMain.on('code:agent-start', async (event, payload) => {
  const { modelConfigId, prompt, projectDir, sessionId } = payload;

  // Get model configuration
  const configItem = await storage.getById('models', modelConfigId);
  const config = configItem.data;

  // Create Agent instance
  const agent = new CodeAgentLoop({
    apiKey: config.apiKey,
    baseURL: config.baseURL,
    model: config.model,
    projectDir,
    timeout: config.timeout || 120000,
    maxRetries: config.maxRetries || 0,
  });

  // Load MCP Servers
  const mcpManager = new MCPManager(agent.toolRegistry);
  // ... Load MCP Servers from config

  // Load Skills
  const skillManager = new SkillManager(agent.toolRegistry);
  for (const skill of builtinSkills) {
    skillManager.registerSkill(skill);
  }

  activeAgents.set(sessionId, agent);

  // Execute Agent
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
  });

  // Send Diff results
  event.sender.send('code:agent-diff', { files: result.diffs });
  event.sender.send('code:agent-validation', { result: result.validation });
  event.sender.send('code:agent-done', { summary: result.response });
});

ipcMain.on('code:agent-apply', async (event, payload) => {
  const { sessionId, action } = payload;
  const agent = activeAgents.get(sessionId);
  if (!agent) return;

  if (action === 'accept') {
    await agent.applyAll();
  } else {
    await agent.revertAll();
  }

  activeAgents.delete(sessionId);
});
```

### 9.4 Message Flow Sequence Diagram

```
Renderer                          Main Process
   │                                   │
   │── code:agent-start ──────────────▶│  1. User sends Prompt
   │                                   │     Agent starts analysis
   │◀─ code:agent-thinking ───────────│  2. Streaming thought process
   │◀─ code:agent-thinking ───────────│     (multiple pushes)
   │◀─ code:agent-response ───────────│  3. Streaming response content
   │◀─ code:agent-response ───────────│     (multiple pushes)
   │                                   │
   │◀─ code:agent-workflow ───────────│  4. Return workflow plan
   │                                   │
   │── code:agent-approve ───────────▶│  5. User approval
   │                                   │
   │◀─ code:agent-tool-use ───────────│  6. Execute tools (multiple)
   │◀─ code:agent-progress ───────────│  7. Progress updates
   │◀─ code:agent-tool-use ───────────│
   │◀─ code:agent-progress ───────────│
   │                                   │
   │◀─ code:agent-validation ─────────│  8. Sandbox validation result
   │◀─ code:agent-diff ───────────────│  9. Diff results
   │                                   │
   │── code:agent-apply ─────────────▶│ 10. User selects accept/reject
   │                                   │
   │◀─ code:agent-done ───────────────│ 11. Complete
```

---

## 10. Frontend UI Design

### 10.1 New Components

| Component | File Path | Function | Priority |
|-----------|-----------|----------|----------|
| `ThinkingChain` | `src/renderer/components/ThinkingChain.tsx` | Display thought process, supports collapse/expand | P0 |
| `WorkflowPanel` | `src/renderer/components/WorkflowPanel.tsx` | Display workflow steps and approval buttons | P0 |
| `DiffViewer` | `src/renderer/components/DiffViewer.tsx` | Code diff comparison display (unified diff) | P0 |
| `FileChangeList` | `src/renderer/components/FileChangeList.tsx` | Modified file list, click to expand diff | P0 |
| `ProgressIndicator` | `src/renderer/components/ProgressIndicator.tsx` | Tool execution progress indicator | P1 |

### 10.2 ThinkingChain Component Design

```tsx
/**
 * ThinkingChain component
 *
 * Displays the AI's thought process, similar to ChatGPT's "Thinking" expandable area.
 * - Default collapsed, shows "Thinking..." title
 * - Click to expand and show complete thinking content
 * - Supports streaming updates (content displayed progressively)
 * - Uses Ant Design Collapse component
 */
interface ThinkingChainProps {
  /** Thinking content */
  content: string;
  /** Whether currently streaming output */
  isStreaming: boolean;
  /** Whether expanded by default */
  defaultExpanded?: boolean;
}
```

### 10.3 WorkflowPanel Component Design

```tsx
/**
 * WorkflowPanel component
 *
 * Displays the AI-generated code modification plan, awaiting user approval.
 * - Uses Ant Design Card + Steps components
 * - Each step shows: number, operation type icon, file path, description
 * - Bottom shows [Allow] [Deny] buttons
 */
interface WorkflowPanelProps {
  /** Workflow step list */
  steps: WorkflowStep[];
  /** Workflow summary */
  summary: string;
  /** Approval callback */
  onApprove: () => void;
  /** Rejection callback */
  onDeny: () => void;
  /** Whether awaiting approval */
  isPending: boolean;
}
```

### 10.4 DiffViewer Component Design

```tsx
/**
 * DiffViewer component
 *
 * Displays code diff comparison, similar to GitHub PR diff view.
 * - Uses unified diff format
 * - Added lines with green background, deleted lines with red background
 * - File name as title, shows +/- line count
 * - Uses Ant Design Card + Typography.Text code style
 */
interface DiffViewerProps {
  /** File diff list */
  files: FileDiff[];
  /** Whether loading */
  loading?: boolean;
}
```

### 10.5 FileChangeList Component Design

```tsx
/**
 * FileChangeList component
 *
 * Displays a list of all modified files. Click to expand and view specific diffs.
 * - Uses Ant Design List + Collapse components
 * - Each file shows: file path, operation type tag, +N/-N lines
 * - Bottom shows [✅ Accept All] [❌ Reject All] buttons
 */
interface FileChangeListProps {
  /** File diff list */
  files: FileDiff[];
  /** Accept all callback */
  onAcceptAll: () => Promise<void>;
  /** Reject all callback */
  onRejectAll: () => Promise<void>;
  /** Whether performing operation */
  loading?: boolean;
}
```

### 10.6 Code.tsx Page Update

Building on the existing `Code.tsx`, add the following states and logic:

```typescript
// New states
const [agentPhase, setAgentPhase] = useState<'idle' | 'thinking' | 'workflow' | 'executing' | 'diff' | 'done'>('idle');
const [thinkingContent, setThinkingContent] = useState('');
const [responseContent, setResponseContent] = useState('');
const [workflowSteps, setWorkflowSteps] = useState<WorkflowStep[]>([]);
const [fileDiffs, setFileDiffs] = useState<FileDiff[]>([]);
const [toolCalls, setToolCalls] = useState<ToolUseEvent[]>([]);
const [validationResult, setValidationResult] = useState<ValidationResult | null>(null);

// New handleSend logic
const handleSend = useCallback(async (text: string) => {
  // ... existing logic ...

  // Use new Agent IPC channels
  const unsubs: (() => void)[] = [];

  unsubs.push(window.electronAPI!.on('code:agent-thinking', (data) => {
    setThinkingContent(prev => prev + data.content);
    setAgentPhase('thinking');
  }));

  unsubs.push(window.electronAPI!.on('code:agent-response', (data) => {
    setResponseContent(prev => prev + data.content);
  }));

  unsubs.push(window.electronAPI!.on('code:agent-tool-use', (data) => {
    setToolCalls(prev => [...prev, data]);
    setAgentPhase('executing');
  }));

  unsubs.push(window.electronAPI!.on('code:agent-diff', (data) => {
    setFileDiffs(data.files);
    setAgentPhase('diff');
  }));

  unsubs.push(window.electronAPI!.on('code:agent-validation', (data) => {
    setValidationResult(data.result);
  }));

  unsubs.push(window.electronAPI!.on('code:agent-done', () => {
    setAgentPhase('done');
    unsubs.forEach(fn => fn());
  }));

  window.electronAPI!.send('code:agent-start', {
    modelConfigId: selectedModelId,
    prompt: text.trim(),
    projectDir: activeSession.data.projectDir,
    sessionId: activeSession.id,
  });
}, [activeSession, selectedModelId]);
```

### 10.7 Message Bubble Extension

In the `MessageBubbles` component, render different message cards based on `agentPhase`:

- **thinking phase**: Display `ThinkingChain` component (collapsed thought process)
- **workflow phase**: Display `WorkflowPanel` component (approval card)
- **executing phase**: Display `ProgressIndicator` component (tool execution progress)
- **diff phase**: Display `FileChangeList` + `DiffViewer` components (code diff)
- **done phase**: Display completion summary

---

## 11. File Structure Planning

```
packages/your-ai-app-agent/src/
├── main/
│   ├── code-agent/
│   │   ├── index.ts                    # CodeAgent unified entry point
│   │   ├── agent-loop.ts               # Agent loop core
│   │   ├── tool-registry.ts            # Tool registry
│   │   ├── mcp-manager.ts              # MCP manager
│   │   ├── skill-manager.ts            # Skill manager
│   │   ├── sandbox.ts                  # Sandbox executor
│   │   ├── diff-tracker.ts             # Git Diff tracker
│   │   ├── types.ts                    # Type definitions
│   │   ├── skills/
│   │   │   └── index.ts                # Built-in Skill definitions
│   │   └── tools/
│   │       ├── index.ts                # Built-in tool collection
│   │       ├── file-read.ts            # File read tool
│   │       ├── file-write.ts           # File write tool
│   │       ├── file-edit.ts            # File edit tool
│   │       ├── glob-search.ts          # File search tool
│   │       ├── grep-search.ts          # Content search tool
│   │       ├── run-command.ts          # Command execution tool
│   │       └── list-directory.ts       # Directory listing tool
│   ├── code-handlers.ts                # Update: Agent IPC handlers
│   ├── ipc-handlers.ts                 # Update: register new handlers
│   └── preload.ts                      # Update: add new channels
│
├── renderer/
│   ├── components/
│   │   ├── ThinkingChain.tsx           # New: Thinking chain component
│   │   ├── WorkflowPanel.tsx           # New: Workflow panel
│   │   ├── DiffViewer.tsx              # New: Diff viewer
│   │   ├── FileChangeList.tsx          # New: File change list
│   │   ├── ProgressIndicator.tsx       # New: Progress indicator
│   │   ├── types.ts                    # Update: add Agent types
│   │   └── index.ts                    # Update: export new components
│   └── pages/
│       └── Code.tsx                    # Update: integrate Agent flow
│
└── docs/
    └── code-agent-workflow-design.md   # This document
```

---

## 12. Implementation Steps

### Phase 1: Basic Agent Loop (3-4 days)

**Goal:** Implement the Agent core loop that can read and modify code through Tool Calling.

| Step | Task | Output |
|------|------|--------|
| 1.1 | Create `src/main/code-agent/` directory structure | Directory skeleton |
| 1.2 | Implement `types.ts`: all type definitions | Type file |
| 1.3 | Implement `tools/index.ts`: 7 built-in tools | Tool system |
| 1.4 | Implement `tool-registry.ts`: tool registry | Tool registration |
| 1.5 | Implement `agent-loop.ts`: Agent loop core | Agent core |
| 1.6 | Implement `diff-tracker.ts`: Git Diff tracking | Change tracking |
| 1.7 | Update `code-handlers.ts`: add `code:agent-start` handler | IPC integration |
| 1.8 | Update `preload.ts`: expose new IPC channels | Channel exposure |

**Validation criteria:** Can call Agent from Main Process, see tool calls and Diff output in console.

### Phase 2: Frontend Interaction (2-3 days)

**Goal:** Implement the complete UI interaction flow.

| Step | Task | Output |
|------|------|--------|
| 2.1 | Implement `ThinkingChain.tsx` component | Thought display |
| 2.2 | Implement `WorkflowPanel.tsx` component | Workflow approval |
| 2.3 | Implement `DiffViewer.tsx` component | Code diff |
| 2.4 | Implement `FileChangeList.tsx` component | File list |
| 2.5 | Implement `ProgressIndicator.tsx` component | Progress indicator |
| 2.6 | Update `types.ts`: frontend Agent types | Type update |
| 2.7 | Update `Code.tsx`: integrate complete Agent flow | Page integration |
| 2.8 | Implement Accept All / Reject All functionality | Modification confirmation |

**Validation criteria:** User inputs Prompt → sees thought chain → approves workflow → sees execution progress → sees Diff → chooses accept/reject.

### Phase 3: MCP + Skill Extension (2-3 days)

**Goal:** Support dynamic loading of MCP Servers and custom Skills.

| Step | Task | Output |
|------|------|--------|
| 3.1 | Install `@modelcontextprotocol/sdk` dependency | Dependency installation |
| 3.2 | Implement `mcp-manager.ts` | MCP management |
| 3.3 | Implement `skill-manager.ts` | Skill management |
| 3.4 | Implement `skills/index.ts`: built-in Skills | Built-in skills |
| 3.5 | Implement `sandbox.ts`: sandbox validation | Sandbox validation |
| 3.6 | UI supports MCP configuration and Skill management | Configuration UI |

**Validation criteria:** Can load external MCP Server (e.g., filesystem), Agent can use its tools.

### Phase 4: Testing and Optimization (1-2 days)

**Goal:** Ensure stability and user experience.

| Step | Task | Output |
|------|------|--------|
| 4.1 | End-to-end testing of Agent workflow | Test report |
| 4.2 | Error handling and edge cases | Robustness |
| 4.3 | Large file Diff performance optimization | Performance |
| 4.4 | User experience optimization (loading, empty states, error prompts) | UX optimization |

---

## 13. Cost Estimation

| Item | Solution | Cost |
|------|----------|------|
| LLM API | Use OpenAI-compatible API (already in project) | Pay-per-token, approx. $0.01-0.05/call |
| Agent Framework | Custom-built (based on openai SDK) | Zero additional npm license cost |
| MCP Support | `@modelcontextprotocol/sdk` (MIT free) | Zero additional cost |
| Git Diff | `simple-git` (MIT free) | Zero additional cost |
| Code Diff | `diff` (BSD free) | Zero additional cost |
| Sandbox | Process isolation + temporary directory | Zero additional cost |
| **Total** | | **Zero additional npm license cost, only LLM API call fees** |

---

## Appendix: Complete Copyable Implementation Prompt

> **Usage:** The following Prompt can be directly copied to AI programming assistants (Claude Code, Codex CLI, Cursor, etc.) to implement this solution. It is recommended to execute Phase by Phase.

---

````markdown
# Code Agent Workflow Implementation Task

## Project Background

You are implementing a Code Agent workflow for the `packages/your-ai-app-agent/` project (Electron + React + TypeScript).
The project already has a basic `/code` page UI (`Code.tsx`) and streaming chat IPC handling in `code-handlers.ts`.
You need to transform the `/code` page into a true AI Code Agent, supporting:
1. Complete Thinking Chain display
2. Workflow Planning and user approval
3. Code modification execution based on Tool Calling
4. Code Diff display and Accept All / Reject All

## Tech Stack

- Runtime: Electron 23 + Node.js
- Frontend: React 19 + Ant Design 6 + TypeScript
- LLM SDK: openai ^7.3.0 (already installed)
- New dependencies: simple-git, diff, @modelcontextprotocol/sdk, zod

## Implementation Steps

### Phase 1: Basic Agent Loop

#### Step 1.1: Install New Dependencies

Run in the `packages/your-ai-app-agent/` directory:

```bash
pnpm add simple-git diff zod
pnpm add @modelcontextprotocol/sdk
```

#### Step 1.2: Create Directory Structure

Create the `src/main/code-agent/` directory with the following subdirectories:
- `tools/` - built-in tool implementations

#### Step 1.3: Implement Type Definitions

Create `src/main/code-agent/types.ts`, including the following interfaces:
- AgentConfig: API configuration
- AgentCallbacks: streaming callbacks (onThinking, onContent, onToolUse)
- ToolUseEvent: tool call event
- ToolCallRecord: tool call record
- WorkflowStep: workflow step
- FileDiff: file diff
- ValidationResult: sandbox validation result
- AgentResult: Agent execution result
- MCPServerConfig: MCP server configuration
- SkillDefinition: custom Skill definition
- ToolDefinition: tool definition

#### Step 1.4: Implement Built-in Tools

Create `src/main/code-agent/tools/index.ts`, implementing 7 built-in tools:

1. **read_file** - Read file contents (supports line range)
2. **write_file** - Create or overwrite file
3. **edit_file** - Precise string replacement edit (must be unique match)
4. **glob_files** - File glob search
5. **grep_search** - Content regex search
6. **run_command** - Execute shell command (60s timeout limit)
7. **list_directory** - List directory contents

Each tool includes:
- OpenAI Function Calling format definition (name, description, parameters)
- Node.js implementation function (handler)

#### Step 1.5: Implement Tool Registry

Create `src/main/code-agent/tool-registry.ts`:
- `register(definition, handler)` - register tool
- `registerMCPTools(mcpTools)` - register MCP tools (with mcp__ prefix)
- `getOpenAIToolDefinitions()` - get OpenAI format tool definitions
- `execute(name, args)` - execute tool

#### Step 1.6: Implement Agent Loop

Create `src/main/code-agent/agent-loop.ts`:
- Agent loop based on OpenAI Tool Calling
- Supports multi-turn tool calls
- Streaming output of thinking + content
- Builds System Prompt with project context and behavioral constraints
- Integrates ToolRegistry and GitDiffTracker

#### Step 1.7: Implement Git Diff Tracker

Create `src/main/code-agent/diff-tracker.ts`:
- Change tracking based on simple-git
- `trackFileChange(filePath)` - track file change
- `collectDiffs()` - collect diffs for all changed files
- `revertAll()` - revert all modifications
- `finalize()` - confirm modifications

#### Step 1.8: Update IPC Handlers

Update `src/main/code-handlers.ts`:
- Add `code:agent-start` handler: create Agent instance and execute
- Add `code:agent-apply` handler: apply/revert modifications
- Streaming events: code:agent-thinking, code:agent-response, code:agent-tool-use
- Result events: code:agent-diff, code:agent-validation, code:agent-done, code:agent-error

#### Step 1.9: Update preload.ts

Add IPC channels to validSendChannels and validOnChannels

### Phase 2: Frontend UI

#### Step 2.1: Implement ThinkingChain Component

Create `src/renderer/components/ThinkingChain.tsx`:
- Use Ant Design Collapse component
- Streaming update support
- Collapse/expand toggle

#### Step 2.2: Implement WorkflowPanel Component

Create `src/renderer/components/WorkflowPanel.tsx`:
- Use Ant Design Card + Steps components
- Display step list (number, operation type, file path, description)
- Allow / Deny approval buttons

#### Step 2.3: Implement DiffViewer Component

Create `src/renderer/components/DiffViewer.tsx`:
- Unified diff format display
- Added lines green background, deleted lines red background
- File name title + line count

#### Step 2.4: Implement FileChangeList Component

Create `src/renderer/components/FileChangeList.tsx`:
- File list + click to expand diff
- Operation type tags (Added/Modified/Deleted)
- Accept All / Reject All buttons

#### Step 2.5: Implement ProgressIndicator Component

Create `src/renderer/components/ProgressIndicator.tsx`:
- Tool call real-time progress display
- Use Ant Design Steps or Timeline component

#### Step 2.6: Update Type Definitions

Update `src/renderer/components/types.ts`:
- Add AgentPhase type
- Add WorkflowStep, FileDiff, ValidationResult and other frontend types

#### Step 2.7: Update Code.tsx Page

Update `src/renderer/pages/Code.tsx`:
- Add agentPhase state management
- Integrate ThinkingChain, WorkflowPanel, DiffViewer, FileChangeList, ProgressIndicator
- Use new Agent IPC channels
- Implement Accept All / Reject All functionality

#### Step 2.8: Update Component Exports

Update `src/renderer/components/index.ts`: export new components

### Phase 3: MCP + Skill Extension

#### Step 3.1: Implement MCP Manager

Create `src/main/code-agent/mcp-manager.ts`:
- Based on @modelcontextprotocol/sdk
- `loadServer(config)` - start MCP Server via stdio
- `unloadServer(name)` - unload MCP Server
- Auto-register MCP tools to ToolRegistry (with mcp__ prefix)

#### Step 3.2: Implement Skill Manager

Create `src/main/code-agent/skill-manager.ts`:
- `registerSkill(skill)` - register Skill
- `getSystemPromptExtension()` - get merged System Prompt extension
- Built-in Skills: typescript-strict, react-patterns, testing

#### Step 3.3: Implement Sandbox Validator

Create `src/main/code-agent/sandbox.ts`:
- Validate code modifications in isolated environment
- Auto-detect project type and select validation commands
- TypeScript projects: tsc --noEmit
- ESLint: npm run lint
- Tests: npm run test

### Phase 4: Testing and Validation

#### Step 4.1: End-to-End Testing

1. Start Electron application
2. Navigate to /code page
3. Select a project directory
4. Send Prompt: "Add a README.md file to this project"
5. Verify: thought chain display → workflow approval → execution progress → Diff display → accept/reject

#### Step 4.2: Error Handling Validation

1. Test invalid prompts
2. Test tool call failures
3. Test network interruption
4. Test revert operation

## Notes

1. **Do not modify** existing chat functionality in `Chat.tsx` and `chat-handlers.ts`
2. **Maintain compatibility** with existing `code:send-stream` and other legacy IPC channels (can keep as fallback)
3. **Follow existing code style**: use Ant Design components, follow TypeScript strict mode
4. **All tool calls** must be executed in the Main Process (cannot go through IPC back to Renderer for execution)
5. **File paths** should uniformly use relative paths (relative to projectDir)
6. **run_command tool** must set a 60-second timeout and output buffer limit
````

---

> **Document version:** v1.0
> **Created:** 2026-08-02
> **Applicable project:** `packages/your-ai-app-agent/`
