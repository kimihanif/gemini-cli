# Gemini CLI - Detailed Design Documentation

## Table of Contents
1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [Component Design](#component-design)
4. [Tool System Design](#tool-system-design)
5. [Agent Execution Design](#agent-execution-design)
6. [Data Models](#data-models)
7. [Configuration System](#configuration-system)
8. [Security & Policy Design](#security--policy-design)
9. [Extension System](#extension-system)
10. [Integration Points](#integration-points)

---

## Overview

### What is Gemini CLI?

Gemini CLI is an enterprise-grade, terminal-based AI agent that provides command-line access to Google's Gemini API. It's architected as a monorepo application with multiple interdependent packages, designed for extensibility, observability, and production use.

### Key Statistics
- **Version**: 0.15.0-nightly
- **Total Files**: 928 TypeScript files
- **License**: Apache 2.0
- **Lines of Code**: ~500,000+ (estimated)
- **Primary Language**: TypeScript 5.3.3
- **Runtime**: Node.js 20+

### Design Principles
1. **Separation of Concerns**: CLI (presentation) and Core (business logic) are distinct packages
2. **Extensibility**: Plugin architecture via MCP (Model Context Protocol)
3. **Observability**: Comprehensive telemetry and logging
4. **Type Safety**: Full TypeScript with strict mode
5. **User Safety**: Policy engine for dangerous operations
6. **Performance**: Parallel tool execution where possible

---

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Terminal Interface                       │
│                    (Ink/React Rendering)                     │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                      CLI Package                             │
│  ┌────────────┐  ┌─────────────┐  ┌──────────────────┐     │
│  │ AppContainer│  │  UI         │  │  Config &        │     │
│  │  (State)   │  │ Components  │  │  Settings        │     │
│  └────────────┘  └─────────────┘  └──────────────────┘     │
│  ┌────────────┐  ┌─────────────┐  ┌──────────────────┐     │
│  │  Hooks     │  │  Commands   │  │  Services        │     │
│  └────────────┘  └─────────────┘  └──────────────────┘     │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                     Core Package                             │
│  ┌────────────────────────────────────────────────┐         │
│  │           Gemini Chat Orchestration            │         │
│  └────────────┬──────────────────┬────────────────┘         │
│               │                  │                           │
│  ┌────────────▼────────┐  ┌─────▼──────────────┐           │
│  │  Agent Executor     │  │  Tool Scheduler    │           │
│  │  (Turn Management)  │  │  (Queue-based)     │           │
│  └────────────┬────────┘  └─────┬──────────────┘           │
│               │                  │                           │
│  ┌────────────▼──────────────────▼────────────────┐         │
│  │            Tool Registry & Execution            │         │
│  └────────────┬────────────────────────────────────┘        │
│               │                                              │
│  ┌────────────▼────────────────────────────────────┐        │
│  │  Services (Shell, File, Git, Model Config)      │        │
│  └──────────────────────────────────────────────────┘       │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                  External Systems                            │
│  ┌─────────────┐  ┌──────────┐  ┌───────────────────┐      │
│  │  Gemini API │  │   MCP    │  │  IDE Integrations │      │
│  │  (@google/  │  │  Servers │  │  (VS Code, Zed)   │      │
│  │   genai)    │  └──────────┘  └───────────────────┘      │
│  └─────────────┘                                             │
└──────────────────────────────────────────────────────────────┘
```

### Package Structure

```
gemini-cli/
├── packages/
│   ├── cli/                    # User-facing CLI application
│   │   ├── src/
│   │   │   ├── gemini.tsx      # Main entry point
│   │   │   ├── ui/             # React/Ink UI components (190 files)
│   │   │   ├── config/         # Configuration & settings
│   │   │   ├── commands/       # Built-in commands
│   │   │   ├── services/       # CLI-specific services
│   │   │   └── utils/          # CLI utilities
│   │   └── package.json
│   │
│   ├── core/                   # Backend logic & AI orchestration
│   │   ├── src/
│   │   │   ├── core/           # Chat orchestration (27 files, 498KB)
│   │   │   ├── tools/          # Tool system (47 files, 775KB)
│   │   │   ├── agents/         # Agent execution (14 files, 154KB)
│   │   │   ├── services/       # Core services (19 files, 230KB)
│   │   │   ├── config/         # Configuration (18 files, 120KB)
│   │   │   ├── mcp/            # MCP integration (154KB)
│   │   │   ├── policy/         # Policy engine (10 files, 105KB)
│   │   │   ├── telemetry/      # Observability (38 files, 405KB)
│   │   │   └── routing/        # Model routing (23KB)
│   │   └── package.json
│   │
│   ├── a2a-server/             # Agent-to-Agent server
│   │   ├── src/
│   │   │   ├── server.ts       # Express server
│   │   │   └── routes/         # API endpoints
│   │   └── package.json
│   │
│   ├── vscode-ide-companion/   # VS Code extension
│   │   ├── src/
│   │   │   ├── extension.ts    # Extension entry
│   │   │   └── mcp/            # MCP integration
│   │   └── package.json
│   │
│   └── test-utils/             # Shared testing utilities
│       └── src/
│
├── docs/                       # Documentation
├── integration-tests/          # E2E tests
├── schemas/                    # JSON schemas
└── scripts/                    # Build & deployment
```

---

## Component Design

### 1. CLI Package Components

#### 1.1 AppContainer (Main Application Component)

**File**: `packages/cli/src/ui/AppContainer.tsx` (44KB)

**Responsibilities**:
- Root application state management
- User input handling and dispatch
- Chat history management
- Session lifecycle management
- Context provider orchestration

**Key State**:
```typescript
interface AppState {
  messages: Message[];              // Conversation history
  currentInput: string;             // User input buffer
  isProcessing: boolean;            // API call in progress
  settings: Settings;               // User settings
  session: SessionInfo;             // Session metadata
  uiState: UIState;                 // UI-specific state
}
```

**React Context Hierarchy**:
```
<AppContext.Provider>
  <SettingsContext.Provider>
    <SessionContext.Provider>
      <UIStateContext.Provider>
        <UIActionsContext.Provider>
          <KeypressContext.Provider>
            <VimModeContext.Provider>
              <ScrollProvider>
                <MouseContext.Provider>
                  <App />
```

#### 1.2 Hook System

**Location**: `packages/cli/src/ui/hooks/` (~50 hooks, 838KB)

**Core Hooks**:

| Hook | File | Purpose | Key Dependencies |
|------|------|---------|------------------|
| `useGeminiStream` | useGeminiStream.ts | Gemini API streaming integration | Core package client |
| `useHistory` | useHistory.ts | Chat history management | SessionContext |
| `useKeypress` | useKeypress.ts | Keyboard input handling | Ink useInput |
| `useSlashCommandProcessor` | useSlashCommandProcessor.ts | Command parsing & execution | CommandService |
| `useAuthCommand` | useAuthCommand.ts | Authentication flow | Core auth |
| `useModelCommand` | useModelCommand.ts | Model switching | Config |
| `useMemoryMonitor` | useMemoryMonitor.ts | Memory tool integration | MemoryTool |
| `useVim` | useVim.ts | Vim keybindings | VimModeContext |

**Hook Pattern Example**:
```typescript
export function useGeminiStream(config: Config) {
  const [streaming, setStreaming] = useState(false);
  const [response, setResponse] = useState<string>('');

  const streamMessage = useCallback(async (message: string) => {
    setStreaming(true);
    const stream = await config.client.sendMessage(message);

    for await (const chunk of stream) {
      setResponse(prev => prev + chunk);
    }

    setStreaming(false);
  }, [config]);

  return { streaming, response, streamMessage };
}
```

#### 1.3 Configuration System

**File**: `packages/cli/src/config/config.ts`

**Configuration Sources** (priority order):
1. CLI arguments (highest priority)
2. Environment variables
3. Settings file (~/.gemini/settings.json)
4. Default values (lowest priority)

**Settings Schema**:
```typescript
interface Settings {
  model?: string;                    // Default model
  apiKey?: string;                   // Gemini API key
  autoApproveTools?: boolean;        // Auto-approve safe tools
  theme?: 'light' | 'dark' | 'auto'; // UI theme
  vimMode?: boolean;                 // Vim keybindings
  telemetry?: boolean;               // Telemetry opt-in
  extensions?: ExtensionConfig[];    // Installed extensions
  mcpServers?: MCPServerConfig[];    // MCP server configs
}
```

**Settings Persistence**:
- Location: `~/.gemini/settings.json`
- Format: JSON with comments (via comment-json)
- Schema validation: Zod schemas
- Auto-migration: Version-based migration logic

#### 1.4 Command System

**Location**: `packages/cli/src/commands/`

**Built-in Commands**:
- `/help` - Show help
- `/model` - Switch model
- `/auth` - Authenticate
- `/clear` - Clear history
- `/settings` - Manage settings
- `/extensions` - Extension management
- `/mcp` - MCP server management

**Command Registration**:
```typescript
// packages/cli/src/services/BuiltinCommandLoader.ts
export class BuiltinCommandLoader {
  loadCommands(): Command[] {
    return [
      {
        name: 'model',
        description: 'Switch the active model',
        execute: async (args, config) => { /* ... */ }
      },
      // ...
    ];
  }
}
```

### 2. Core Package Components

#### 2.1 Gemini Chat Orchestrator

**File**: `packages/core/src/core/geminiChat.ts` (20.5KB)

**Responsibilities**:
- Manage conversation state
- Coordinate API calls
- Handle streaming responses
- Implement retry logic
- Validate responses
- Error recovery

**Key Methods**:

```typescript
class GeminiChat {
  // Send message and stream response
  async sendMessage(
    message: string,
    options: ChatOptions
  ): AsyncGenerator<ChatChunk>

  // Add message to history
  addMessage(role: 'user' | 'model', content: Content[]): void

  // Get chat history
  getHistory(): Content[]

  // Clear history
  clearHistory(): void

  // Validate response
  validateResponse(response: GenerateContentResponse): ValidationResult
}
```

**Streaming Architecture**:
```typescript
async *sendMessage(message: string) {
  const stream = await this.client.generateContentStream({
    contents: this.history,
    tools: this.toolRegistry.getFunctionDeclarations(),
  });

  for await (const chunk of stream) {
    // Validate chunk
    if (chunk.candidates?.[0]?.content) {
      yield {
        text: chunk.text,
        functionCalls: chunk.functionCalls,
      };
    }
  }
}
```

#### 2.2 Configuration Class

**File**: `packages/core/src/config/config.ts` (44KB)

**Responsibilities**:
- Initialize all subsystems
- Provide service access
- Manage API clients
- Configure tools
- Set model parameters

**Initialization Sequence**:
```typescript
export class Config {
  async initialize() {
    // 1. Initialize base services
    this.fileDiscoveryService = new FileDiscoveryService();
    this.gitService = new GitService();
    this.shellExecutionService = new ShellExecutionService();

    // 2. Initialize Gemini client
    this.client = new GeminiClient({
      apiKey: this.apiKey,
      model: this.model,
    });

    // 3. Initialize tool registry
    this.toolRegistry = new ToolRegistry();
    this.registerBuiltinTools();

    // 4. Discover project tools
    await this.toolRegistry.discoverAllTools();

    // 5. Initialize policy engine
    this.policyEngine = new PolicyEngine();

    // 6. Initialize telemetry
    this.telemetry = new TelemetryService();

    return this;
  }
}
```

#### 2.3 Service Architecture

**Location**: `packages/core/src/services/`

**Core Services**:

##### FileDiscoveryService
**File**: `fileDiscoveryService.ts`

```typescript
export class FileDiscoveryService {
  // Discover files matching patterns
  async discoverFiles(
    patterns: string[],
    options?: DiscoveryOptions
  ): Promise<FileInfo[]>

  // Get workspace context
  async getWorkspaceContext(): Promise<WorkspaceContext>

  // Respect .gitignore
  private shouldIgnore(path: string): boolean
}
```

##### GitService
**File**: `gitService.ts`

```typescript
export class GitService {
  // Get git status
  async getStatus(): Promise<GitStatus>

  // Get diff
  async getDiff(options?: DiffOptions): Promise<string>

  // Commit changes
  async commit(message: string): Promise<void>

  // Push to remote
  async push(branch?: string): Promise<void>
}
```

##### ShellExecutionService
**File**: `shellExecutionService.ts` (25.5KB)

```typescript
export class ShellExecutionService {
  // Execute command
  async execute(
    command: string,
    options?: ExecutionOptions
  ): Promise<ExecutionResult>

  // Execute with PTY (pseudo-terminal)
  async executeWithPty(
    command: string,
    options?: PtyOptions
  ): AsyncGenerator<OutputChunk>

  // Kill running process
  kill(pid: number, signal?: string): void
}
```

**Execution Options**:
```typescript
interface ExecutionOptions {
  cwd?: string;           // Working directory
  env?: Record<string, string>;  // Environment variables
  timeout?: number;       // Execution timeout (ms)
  signal?: AbortSignal;   // Cancellation signal
  input?: string;         // Stdin input
  shell?: string;         // Shell to use
}
```

---

## Tool System Design

### Tool Architecture

#### Class Hierarchy

```
ToolBuilder (interface)
    │
    ├─ DeclarativeTool<TParams, TResult> (abstract)
    │   │
    │   ├─ BaseDeclarativeTool<TParams, TResult> (abstract)
    │   │   │
    │   │   ├─ ReadFileTool
    │   │   ├─ WriteFileTool
    │   │   ├─ EditTool
    │   │   ├─ GlobTool
    │   │   ├─ GrepTool
    │   │   ├─ LsTool
    │   │   ├─ ShellTool
    │   │   ├─ WebFetchTool
    │   │   ├─ WebSearchTool
    │   │   ├─ MemoryTool
    │   │   └─ WriteTodosTool
    │   │
    │   └─ DiscoveredTool (for project-specific tools)
    │
    └─ DiscoveredMCPTool (for MCP server tools)
```

#### Tool Definition Pattern

**File**: `packages/core/src/tools/tools.ts`

```typescript
// Base class that all tools extend
export abstract class DeclarativeTool<TParams, TResult>
  implements ToolBuilder {

  // Metadata
  readonly name: string;
  readonly displayName: string;
  readonly description: string;
  readonly kind: Kind;  // read, edit, execute, etc.

  // Parameter schema for validation
  abstract readonly parameterSchema: unknown;

  // Create an executable invocation
  abstract build(params: TParams): ToolInvocation<TParams, TResult>;

  // Schema for Gemini API
  get schema(): FunctionDeclaration {
    return {
      name: this.name,
      description: this.description,
      parameters: this.parameterSchema,
    };
  }

  // Validate parameters
  protected validateToolParams(params: TParams): string | undefined {
    // Zod validation logic
  }
}
```

#### Tool Invocation Pattern

```typescript
// Represents a single tool call execution
export interface ToolInvocation<TParams, TResult> {
  // Unique invocation ID
  readonly invocationId: string;

  // Tool reference
  readonly tool: ToolBuilder;

  // Validated parameters
  readonly params: TParams;

  // Check if user confirmation needed
  shouldConfirmExecute(signal: AbortSignal): Promise<boolean>;

  // Execute the tool
  execute(
    signal: AbortSignal,
    onOutput?: OutputCallback
  ): Promise<TResult>;

  // Get display name for UI
  getDisplayName(): string;
}
```

### Tool Registry

**File**: `packages/core/src/tools/tool-registry.ts` (534 lines)

**Data Structure**:
```typescript
export class ToolRegistry {
  private tools: Map<string, ToolBuilder> = new Map();
  private discoveredTools: DiscoveredTool[] = [];
  private mcpTools: DiscoveredMCPTool[] = [];

  // Register built-in tool
  registerTool(tool: ToolBuilder): void {
    this.tools.set(tool.name, tool);
  }

  // Get tool by name
  getTool(name: string): ToolBuilder | undefined {
    return this.tools.get(name)
      || this.discoveredTools.find(t => t.name === name)
      || this.mcpTools.find(t => t.name === name);
  }

  // Get all tool names
  getAllToolNames(): string[] {
    return [
      ...this.tools.keys(),
      ...this.discoveredTools.map(t => t.name),
      ...this.mcpTools.map(t => t.name),
    ];
  }

  // Get schemas for Gemini API
  getFunctionDeclarations(): FunctionDeclaration[] {
    return this.getAllTools().map(tool => tool.schema);
  }
}
```

**Tool Discovery Process**:
```typescript
async discoverAllTools(): Promise<void> {
  // 1. Run discovery command
  const discoveryCmd = this.config.toolDiscoveryCommand;
  const result = await exec(discoveryCmd);

  // 2. Parse JSON array
  const declarations = JSON.parse(result.stdout);

  // 3. Create DiscoveredTool instances
  this.discoveredTools = declarations.map(decl =>
    new DiscoveredTool(decl, this.config)
  );

  // 4. Discover MCP tools
  await this.discoverMCPTools();

  // 5. Sort tools by priority
  this.sortTools();
}
```

### Built-in Tool Implementations

#### ReadFileTool

**File**: `packages/core/src/tools/read-file.ts`

```typescript
interface ReadFileParams {
  file_path: string;
  offset?: number;  // Line offset
  limit?: number;   // Max lines to read
}

class ReadFileToolInvocation extends BaseToolInvocation<
  ReadFileParams,
  ToolResult
> {
  async execute(signal: AbortSignal): Promise<ToolResult> {
    const { file_path, offset = 0, limit = 2000 } = this.params;

    // 1. Read file
    const content = await fs.readFile(file_path, 'utf-8');

    // 2. Split into lines
    const lines = content.split('\n');

    // 3. Apply offset and limit
    const selectedLines = lines.slice(offset, offset + limit);

    // 4. Format with line numbers
    const formatted = selectedLines
      .map((line, i) => `${offset + i + 1}\t${line}`)
      .join('\n');

    return {
      llmContent: formatted,      // For model
      returnDisplay: formatted,   // For user display
    };
  }
}

export class ReadFileTool extends BaseDeclarativeTool<
  ReadFileParams,
  ToolResult
> {
  name = 'read_file';
  kind = Kind.Read;
  parameterSchema = {
    type: 'object',
    properties: {
      file_path: { type: 'string' },
      offset: { type: 'number' },
      limit: { type: 'number' },
    },
    required: ['file_path'],
  };

  build(params: ReadFileParams): ToolInvocation {
    return new ReadFileToolInvocation(params, this, ...);
  }
}
```

#### ShellTool

**File**: `packages/core/src/tools/shell.ts`

```typescript
interface ShellParams {
  command: string;
  description?: string;
  timeout?: number;
  run_in_background?: boolean;
}

class ShellToolInvocation extends BaseToolInvocation<
  ShellParams,
  ToolResult
> {
  async shouldConfirmExecute(): Promise<boolean> {
    // Shell commands always require confirmation
    return true;
  }

  async execute(
    signal: AbortSignal,
    onOutput?: OutputCallback
  ): Promise<ToolResult> {
    const { command, timeout = 120000, run_in_background } = this.params;

    if (run_in_background) {
      // Start background process
      const shellId = await this.shellService.executeBackground(
        command,
        { timeout, signal }
      );

      return {
        llmContent: `Started background shell ${shellId}`,
        returnDisplay: `Background shell started: ${shellId}`,
      };
    } else {
      // Execute and stream output
      const stream = this.shellService.executeWithPty(
        command,
        { timeout, signal }
      );

      let output = '';
      for await (const chunk of stream) {
        output += chunk.data;
        onOutput?.(chunk);  // Stream to UI
      }

      return {
        llmContent: output,
        returnDisplay: output,
      };
    }
  }
}
```

#### EditTool (Find/Replace)

**File**: `packages/core/src/tools/edit.ts`

```typescript
interface EditParams {
  file_path: string;
  old_string: string;
  new_string: string;
  replace_all?: boolean;
}

class EditToolInvocation extends BaseToolInvocation<
  EditParams,
  ToolResult
> {
  async execute(signal: AbortSignal): Promise<ToolResult> {
    const { file_path, old_string, new_string, replace_all } = this.params;

    // 1. Read file
    const content = await fs.readFile(file_path, 'utf-8');

    // 2. Find occurrences
    const occurrences = this.findOccurrences(content, old_string);

    if (occurrences === 0) {
      throw new Error(`String not found: ${old_string}`);
    }

    if (occurrences > 1 && !replace_all) {
      throw new Error(
        `Found ${occurrences} occurrences. Use replace_all=true`
      );
    }

    // 3. Replace
    const newContent = replace_all
      ? content.replaceAll(old_string, new_string)
      : content.replace(old_string, new_string);

    // 4. Write file
    await fs.writeFile(file_path, newContent, 'utf-8');

    return {
      llmContent: `Replaced ${occurrences} occurrence(s) in ${file_path}`,
      returnDisplay: `✓ File edited: ${file_path}`,
    };
  }
}
```

### Tool Categories (Kind Enum)

```typescript
export enum Kind {
  Read = 'read',        // Read-only operations
  Edit = 'edit',        // File modifications
  Delete = 'delete',    // Deletion operations
  Move = 'move',        // Move/rename operations
  Search = 'search',    // Search operations
  Execute = 'execute',  // Command execution
  Think = 'think',      // Memory/reasoning
  Fetch = 'fetch',      // Network operations
  Other = 'other',      // Unclassified
}

// Tools that modify system state
const MUTATOR_KINDS = [
  Kind.Edit,
  Kind.Delete,
  Kind.Move,
  Kind.Execute,
];
```

---

## Agent Execution Design

### Agent Executor Architecture

**File**: `packages/core/src/agents/executor.ts` (1099 lines)

#### Agent Definition

```typescript
interface AgentDefinition {
  name: string;
  systemPrompt: string;
  modelConfig: ModelConfig;
  tools: string[];          // Tool names to enable
  maxTurns?: number;        // Max conversation turns
  timeout?: number;         // Execution timeout (ms)
  outputConfig?: {
    schema: ZodSchema;      // Output validation
    schemaName: string;
  };
}
```

#### Agent Executor Class

```typescript
export class AgentExecutor {
  constructor(
    private agentDefinition: AgentDefinition,
    private toolRegistry: ToolRegistry,
    private runtimeContext: RuntimeContext
  ) {}

  async run(
    inputs: Record<string, unknown>,
    signal: AbortSignal
  ): Promise<AgentResult> {
    // Initialize chat
    const chat = this.initializeChat();

    // Main execution loop
    let currentMessage = this.buildInitialMessage(inputs);
    let turnCount = 0;

    while (turnCount < this.agentDefinition.maxTurns) {
      // Execute single turn
      const turnResult = await this.executeTurn(
        chat,
        currentMessage,
        signal
      );

      if (turnResult.shouldTerminate) {
        return turnResult.result;
      }

      currentMessage = turnResult.nextMessage;
      turnCount++;
    }

    throw new Error('Max turns exceeded');
  }
}
```

#### Turn Execution

```typescript
private async executeTurn(
  chat: Chat,
  message: Content,
  signal: AbortSignal
): Promise<TurnResult> {
  // 1. Call model
  const response = await this.callModel(chat, message, signal);

  // 2. Check for task completion
  if (response.functionCalls?.some(fc => fc.name === 'complete_task')) {
    return {
      shouldTerminate: true,
      result: this.extractTaskResult(response),
    };
  }

  // 3. Process function calls
  const nextMessage = await this.processFunctionCalls(
    response.functionCalls,
    signal
  );

  return {
    shouldTerminate: false,
    nextMessage,
  };
}
```

#### Model Calling

```typescript
private async callModel(
  chat: Chat,
  message: Content,
  signal: AbortSignal
): Promise<ModelResponse> {
  // 1. Prepare tools list
  const tools = this.prepareToolsList();

  // 2. Send message with streaming
  const stream = await chat.sendMessageStream({
    contents: [message],
    tools,
    generationConfig: this.modelConfig.generationConfig,
  });

  // 3. Collect response
  let text = '';
  const functionCalls: FunctionCall[] = [];

  for await (const chunk of stream) {
    if (chunk.text) text += chunk.text;
    if (chunk.functionCalls) functionCalls.push(...chunk.functionCalls);
  }

  return { text, functionCalls };
}
```

#### Parallel Tool Execution

```typescript
private async processFunctionCalls(
  functionCalls: FunctionCall[],
  signal: AbortSignal
): Promise<Content> {
  // Execute all function calls in parallel
  const toolExecutionPromises = functionCalls.map(async (fc) => {
    // 1. Get tool from registry
    const tool = this.toolRegistry.getTool(fc.name);

    // 2. Build invocation
    const invocation = tool.build(fc.args);

    // 3. Execute
    const result = await invocation.execute(signal);

    return {
      functionResponse: {
        name: fc.name,
        response: result.llmContent,
      },
    };
  });

  // Wait for all to complete
  const responses = await Promise.all(toolExecutionPromises);

  return {
    role: 'function',
    parts: responses,
  };
}
```

### Tool Scheduler Design

**File**: `packages/core/src/core/coreToolScheduler.ts` (1395 lines)

#### Tool Call State Machine

```typescript
type ToolCallStatus =
  | 'validating'          // Validating parameters
  | 'awaiting_approval'   // Waiting for user confirmation
  | 'scheduled'           // Approved, ready to execute
  | 'executing'           // Currently running
  | 'successful'          // Completed successfully
  | 'errored'             // Failed during execution
  | 'cancelled';          // Cancelled by user

interface ToolCall {
  id: string;
  status: ToolCallStatus;
  toolName: string;
  params: unknown;
  invocation?: ToolInvocation;
  result?: ToolResult;
  error?: Error;
}
```

#### Scheduler Class

```typescript
export class CoreToolScheduler {
  private toolCalls: ToolCall[] = [];
  private toolCallQueue: ToolCall[] = [];
  private requestQueue: QueuedRequest[] = [];
  private completedToolCallsForBatch: ToolCall[] = [];

  async schedule(
    request: ToolCallRequestInfo | ToolCallRequestInfo[],
    signal: AbortSignal
  ): Promise<void> {
    // Queue request if already running
    if (this.isRunning()) {
      return this.queueRequest(request, signal);
    }

    await this._schedule(request, signal);
  }

  private async _schedule(
    request: ToolCallRequestInfo,
    signal: AbortSignal
  ): Promise<void> {
    // 1. Get tool from registry
    const tool = this.toolRegistry.getTool(request.toolName);

    // 2. Build invocation
    const invocation = tool.build(request.params);

    // 3. Create tool call
    const toolCall: ToolCall = {
      id: generateId(),
      status: 'validating',
      toolName: request.toolName,
      params: request.params,
      invocation,
    };

    // 4. Add to queue
    this.toolCallQueue.push(toolCall);

    // 5. Process queue
    await this._processNextInQueue();
  }
}
```

#### Sequential Processing

```typescript
private async _processNextInQueue(): Promise<void> {
  // Get next tool call
  const toolCall = this.toolCallQueue.shift();
  if (!toolCall) return;

  // Add to active list
  this.toolCalls.push(toolCall);

  try {
    // 1. Validation phase
    const needsConfirmation = await toolCall.invocation
      .shouldConfirmExecute(this.signal);

    if (needsConfirmation) {
      toolCall.status = 'awaiting_approval';
      await this.requestUserApproval(toolCall);
    }

    // 2. Scheduled phase
    toolCall.status = 'scheduled';

    // 3. Execution phase
    await this.attemptExecutionOfScheduledCalls();

  } catch (error) {
    toolCall.status = 'errored';
    toolCall.error = error;
    await this.handleError(toolCall);
  }
}
```

#### Execution with Output Streaming

```typescript
private async attemptExecutionOfScheduledCalls(): Promise<void> {
  const scheduledCalls = this.toolCalls.filter(
    tc => tc.status === 'scheduled'
  );

  for (const toolCall of scheduledCalls) {
    // Mark as executing
    toolCall.status = 'executing';

    // Execute with output callback
    const result = await toolCall.invocation.execute(
      this.signal,
      (output: OutputChunk) => {
        // Stream output to UI
        this.onOutput?.(toolCall.id, output);
      }
    );

    // Mark as successful
    toolCall.status = 'successful';
    toolCall.result = result;

    // Move to completed batch
    this.completedToolCallsForBatch.push(toolCall);
    this.toolCalls = this.toolCalls.filter(tc => tc.id !== toolCall.id);

    // Notify completion
    await this.checkAndNotifyCompletion();
  }
}
```

---

## Data Models

### Message Model

```typescript
interface Message {
  id: string;
  role: 'user' | 'model' | 'function';
  content: Content[];
  timestamp: number;
  metadata?: MessageMetadata;
}

interface Content {
  role: string;
  parts: Part[];
}

type Part =
  | TextPart
  | FunctionCallPart
  | FunctionResponsePart
  | InlineDataPart;

interface TextPart {
  text: string;
}

interface FunctionCallPart {
  functionCall: {
    name: string;
    args: Record<string, unknown>;
  };
}

interface FunctionResponsePart {
  functionResponse: {
    name: string;
    response: unknown;
  };
}
```

### Tool Result Model

```typescript
interface ToolResult {
  // Content to send back to model
  llmContent: string | object;

  // Content to display to user
  returnDisplay?: string;

  // Error if execution failed
  error?: {
    message: string;
    code?: string;
    details?: unknown;
  };

  // Additional metadata
  metadata?: {
    executionTime?: number;
    resourcesUsed?: ResourceUsage;
  };
}
```

### Model Configuration

```typescript
interface ModelConfig {
  model: string;  // Model identifier
  generationConfig?: {
    temperature?: number;
    topP?: number;
    topK?: number;
    maxOutputTokens?: number;
    stopSequences?: string[];
  };
  safetySettings?: SafetySetting[];
  systemInstruction?: string;
}

interface SafetySetting {
  category: HarmCategory;
  threshold: HarmBlockThreshold;
}
```

### Runtime Context

```typescript
interface RuntimeContext {
  workingDirectory: string;
  projectRoot: string;
  gitRepository?: GitRepository;
  environmentVariables: Record<string, string>;
  user: UserInfo;
  session: SessionInfo;
}

interface SessionInfo {
  id: string;
  startTime: number;
  messageCount: number;
  toolCallCount: number;
  totalTokens: number;
}
```

---

## Configuration System

### Configuration Hierarchy

```
Default Config
    ↓
Environment Variables (GEMINI_*)
    ↓
Settings File (~/.gemini/settings.json)
    ↓
CLI Arguments (--model, --api-key, etc.)
    ↓
Final Configuration
```

### Settings File Schema

**Location**: `packages/cli/src/config/settingsSchema.ts`

```typescript
const settingsSchema = z.object({
  // API Configuration
  model: z.string().optional(),
  apiKey: z.string().optional(),

  // UI Configuration
  theme: z.enum(['light', 'dark', 'auto']).optional(),
  vimMode: z.boolean().optional(),

  // Tool Configuration
  autoApproveTools: z.boolean().optional(),
  toolDiscoveryCommand: z.string().optional(),

  // Extensions
  extensions: z.array(z.object({
    name: z.string(),
    enabled: z.boolean(),
    config: z.record(z.unknown()).optional(),
  })).optional(),

  // MCP Servers
  mcpServers: z.array(z.object({
    name: z.string(),
    command: z.string(),
    args: z.array(z.string()).optional(),
    env: z.record(z.string()).optional(),
  })).optional(),

  // Telemetry
  telemetry: z.boolean().optional(),
});
```

### Model Configuration

**File**: `packages/core/src/config/defaultModelConfigs.ts`

```typescript
export const DEFAULT_MODEL_CONFIGS: Record<string, ModelConfig> = {
  'gemini-2.0-flash-exp': {
    model: 'gemini-2.0-flash-exp',
    generationConfig: {
      temperature: 0.7,
      topP: 0.95,
      maxOutputTokens: 8192,
    },
  },
  'gemini-1.5-pro': {
    model: 'gemini-1.5-pro',
    generationConfig: {
      temperature: 0.7,
      topP: 0.95,
      maxOutputTokens: 8192,
    },
  },
  // ...
};
```

---

## Security & Policy Design

### Policy Engine

**File**: `packages/core/src/policy/policy-engine.ts`

#### Policy Configuration (TOML)

```toml
# Example: .gemini/policy.toml

[tools.run_shell_command]
mode = "ask_user"  # always_allow, always_deny, ask_user

[tools.write_file]
mode = "always_allow"
excluded_paths = [
  "~/.ssh/*",
  "/etc/*",
]

[tools.replace]
mode = "always_allow"
```

#### Policy Engine Class

```typescript
export class PolicyEngine {
  private policies: Map<string, ToolPolicy> = new Map();

  async evaluateToolCall(
    toolName: string,
    params: unknown
  ): Promise<PolicyDecision> {
    const policy = this.policies.get(toolName);

    if (!policy) {
      return { decision: 'ASK_USER' };
    }

    // Check mode
    if (policy.mode === 'always_allow') {
      // Check exclusions
      if (this.matchesExclusion(params, policy.excluded_paths)) {
        return { decision: 'ASK_USER' };
      }
      return { decision: 'ALLOW' };
    }

    if (policy.mode === 'always_deny') {
      return { decision: 'DENY', reason: 'Policy denies this tool' };
    }

    return { decision: 'ASK_USER' };
  }
}
```

### Trusted Folders

**File**: `packages/cli/src/config/trustedFolders.ts`

```typescript
export class TrustedFoldersManager {
  private trustedFolders: Set<string> = new Set();

  // Check if folder is trusted
  isTrusted(folderPath: string): boolean {
    return this.trustedFolders.has(normalize(folderPath));
  }

  // Add trusted folder
  addTrusted(folderPath: string): void {
    this.trustedFolders.add(normalize(folderPath));
    this.persist();
  }

  // Prompt user to trust folder
  async promptTrust(folderPath: string): Promise<boolean> {
    const response = await prompts({
      type: 'confirm',
      name: 'trust',
      message: `Trust folder: ${folderPath}?`,
    });

    if (response.trust) {
      this.addTrusted(folderPath);
      return true;
    }

    return false;
  }
}
```

---

## Extension System

### MCP (Model Context Protocol)

**File**: `packages/core/src/mcp/oauth-provider.ts`

#### MCP Server Configuration

```typescript
interface MCPServerConfig {
  name: string;
  command: string;
  args?: string[];
  env?: Record<string, string>;
  oauth?: {
    clientId: string;
    clientSecret: string;
    scopes: string[];
  };
}
```

#### MCP Client

**File**: `packages/core/src/tools/mcp-client.ts`

```typescript
export class MCPClient {
  async connect(config: MCPServerConfig): Promise<void> {
    // 1. Start MCP server process
    this.process = spawn(config.command, config.args, {
      env: { ...process.env, ...config.env },
    });

    // 2. Initialize stdio transport
    this.transport = new StdioClientTransport({
      stdin: this.process.stdin,
      stdout: this.process.stdout,
    });

    // 3. Create MCP client
    this.client = new Client({
      name: 'gemini-cli',
      version: '1.0.0',
    }, {
      capabilities: {
        tools: {},
        prompts: {},
        resources: {},
      },
    });

    await this.client.connect(this.transport);
  }

  async listTools(): Promise<Tool[]> {
    const response = await this.client.request(
      { method: 'tools/list' },
      ListToolsResultSchema
    );

    return response.tools;
  }

  async callTool(
    name: string,
    args: unknown
  ): Promise<ToolResult> {
    const response = await this.client.request(
      {
        method: 'tools/call',
        params: { name, arguments: args },
      },
      CallToolResultSchema
    );

    return response;
  }
}
```

---

## Integration Points

### IDE Integration

#### VS Code Extension

**File**: `packages/vscode-ide-companion/src/extension.ts`

```typescript
export function activate(context: vscode.ExtensionContext) {
  // 1. Start MCP server
  const mcpServer = new MCPServer();
  await mcpServer.start();

  // 2. Register commands
  context.subscriptions.push(
    vscode.commands.registerCommand(
      'gemini.applyDiff',
      async (diff: Diff) => {
        await applyDiffToEditor(diff);
      }
    )
  );

  // 3. Register diff provider
  context.subscriptions.push(
    vscode.workspace.registerTextDocumentContentProvider(
      'gemini-diff',
      new DiffContentProvider()
    )
  );
}
```

### Telemetry Integration

**File**: `packages/core/src/telemetry/loggers.ts`

```typescript
export class TelemetryLogger {
  private logger: winston.Logger;
  private otelTracer: Tracer;

  logEvent(event: TelemetryEvent): void {
    // 1. Log to Winston
    this.logger.info(event.name, event.properties);

    // 2. Create OpenTelemetry span
    const span = this.otelTracer.startSpan(event.name);
    span.setAttributes(event.properties);
    span.end();

    // 3. Export to Google Cloud
    if (this.config.telemetry) {
      this.gcpExporter.export([span]);
    }
  }
}
```

---

## Summary

This design document provides a comprehensive overview of the Gemini CLI architecture, including:

1. **Modular Architecture**: Clear separation between CLI (UI) and Core (logic)
2. **Extensible Tool System**: Plugin architecture with MCP support
3. **Robust Execution**: Two-level execution (Agent + Scheduler)
4. **Type Safety**: Full TypeScript with Zod validation
5. **User Safety**: Policy engine and trusted folders
6. **Observability**: Comprehensive telemetry and logging
7. **IDE Integration**: VS Code, Zed, and other editors

The system is designed for production use with enterprise-grade features while maintaining flexibility for individual developers.
