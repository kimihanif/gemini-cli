# Gemini CLI - Flow Explanation

## Table of Contents
1. [Overview](#overview)
2. [Application Startup Flow](#application-startup-flow)
3. [User Interaction Flow](#user-interaction-flow)
4. [Tool Calling Flow](#tool-calling-flow)
5. [Agent Execution Loop Flow](#agent-execution-loop-flow)
6. [Tool Scheduler Flow](#tool-scheduler-flow)
7. [Complete Request-Response Cycle](#complete-request-response-cycle)
8. [Error Handling Flow](#error-handling-flow)
9. [MCP Integration Flow](#mcp-integration-flow)
10. [Real-World Examples](#real-world-examples)

---

## Overview

This document explains the execution flows in Gemini CLI, from application startup through complex multi-tool agent interactions. Each section includes sequence diagrams, code references, and real-world examples.

---

## Application Startup Flow

### High-Level Startup Sequence

```
Terminal Launch
    │
    ├─> 1. Parse CLI Arguments
    │      └─ File: packages/cli/src/gemini.tsx:50-100
    │
    ├─> 2. Load Configuration
    │      ├─ Environment variables
    │      ├─ Settings file (~/.gemini/settings.json)
    │      └─ CLI argument overrides
    │      └─ File: packages/cli/src/config/config.ts
    │
    ├─> 3. Initialize Core Config
    │      ├─ Create Gemini API client
    │      ├─ Initialize services (File, Git, Shell)
    │      ├─ Register built-in tools
    │      ├─ Discover project tools
    │      ├─ Initialize policy engine
    │      └─ Setup telemetry
    │      └─ File: packages/core/src/config/config.ts:initialize()
    │
    ├─> 4. Initialize UI
    │      ├─ Setup React contexts
    │      ├─ Load chat history
    │      ├─ Apply theme settings
    │      └─ Initialize vim mode (if enabled)
    │      └─ File: packages/cli/src/ui/AppContainer.tsx
    │
    └─> 5. Render Application
           └─ React/Ink renders terminal UI
           └─ File: packages/cli/src/gemini.tsx:render()
```

### Detailed Startup Flow

#### Phase 1: Argument Parsing (gemini.tsx:50-100)

```typescript
// Entry point
async function main() {
  // 1. Parse arguments
  const argv = yargs(process.argv.slice(2))
    .option('model', { type: 'string' })
    .option('api-key', { type: 'string' })
    .option('non-interactive', { type: 'boolean' })
    .parse();

  // 2. Load CLI config
  const cliConfig = await loadCliConfig(argv);

  // 3. Load settings
  const settings = await loadSettings();

  // 4. Merge configuration
  const finalConfig = mergeConfig(cliConfig, settings, argv);

  // 5. Initialize app
  await initializeApp(finalConfig);

  // 6. Render UI
  if (argv.nonInteractive) {
    await runNonInteractive(finalConfig);
  } else {
    render(<AppContainer config={finalConfig} />);
  }
}
```

#### Phase 2: Core Initialization (config.ts:initialize())

```typescript
export class Config {
  async initialize(): Promise<Config> {
    // Step 1: Validate API key
    if (!this.apiKey) {
      throw new Error('GEMINI_API_KEY not set');
    }

    // Step 2: Initialize services
    this.fileDiscoveryService = new FileDiscoveryService(this.workingDir);
    this.gitService = new GitService(this.workingDir);
    this.shellExecutionService = new ShellExecutionService();

    // Step 3: Create Gemini client
    this.client = new GeminiClient({
      apiKey: this.apiKey,
      model: this.model,
    });

    // Step 4: Initialize tool registry
    this.toolRegistry = new ToolRegistry(this);

    // Step 5: Register built-in tools
    this.toolRegistry.registerTool(new ReadFileTool(this));
    this.toolRegistry.registerTool(new WriteFileTool(this));
    this.toolRegistry.registerTool(new EditTool(this));
    this.toolRegistry.registerTool(new ShellTool(this));
    this.toolRegistry.registerTool(new GlobTool(this));
    this.toolRegistry.registerTool(new GrepTool(this));
    // ... more tools

    // Step 6: Discover project-specific tools
    await this.toolRegistry.discoverAllTools();

    // Step 7: Initialize policy engine
    this.policyEngine = new PolicyEngine(this);
    await this.policyEngine.loadPolicies();

    // Step 8: Initialize telemetry
    this.telemetry = new TelemetryService(this);
    await this.telemetry.initialize();

    // Step 9: Log startup event
    this.telemetry.logEvent({
      name: 'app_started',
      properties: {
        model: this.model,
        toolCount: this.toolRegistry.getAllToolNames().length,
      },
    });

    return this;
  }
}
```

#### Phase 3: UI Initialization (AppContainer.tsx)

```typescript
export function AppContainer({ config }: Props) {
  // 1. Initialize state
  const [messages, setMessages] = useState<Message[]>([]);
  const [isProcessing, setIsProcessing] = useState(false);
  const [settings, setSettings] = useState(config.settings);

  // 2. Load chat history
  useEffect(() => {
    const loadHistory = async () => {
      const history = await config.chatRecordingService.loadHistory();
      setMessages(history);
    };
    loadHistory();
  }, []);

  // 3. Setup contexts
  return (
    <AppContext.Provider value={{ config, messages, setMessages }}>
      <SettingsContext.Provider value={{ settings, setSettings }}>
        <SessionContext.Provider value={sessionInfo}>
          <UIStateContext.Provider value={uiState}>
            <KeypressContext.Provider>
              <VimModeContext.Provider>
                <App />
              </VimModeContext.Provider>
            </KeypressContext.Provider>
          </UIStateContext.Provider>
        </SessionContext.Provider>
      </SettingsContext.Provider>
    </AppContext.Provider>
  );
}
```

---

## User Interaction Flow

### User Input Processing

```
User Types Message
    │
    ├─> 1. Keypress Event
    │      └─ KeypressContext captures input
    │      └─ File: packages/cli/src/ui/contexts/KeypressContext.tsx
    │
    ├─> 2. Input Validation
    │      ├─ Check for slash commands (/model, /help, etc.)
    │      └─ Validate input length
    │      └─ Hook: useSlashCommandProcessor
    │
    ├─> 3. Message Submission
    │      ├─ Add to message history
    │      ├─ Set isProcessing = true
    │      └─ Trigger API call
    │      └─ File: AppContainer.tsx:handleSubmit()
    │
    └─> 4. Stream Response
           ├─ useGeminiStream hook
           └─ Render streaming response
           └─ Hook: packages/cli/src/ui/hooks/useGeminiStream.ts
```

### Detailed Input Processing

#### Step 1: Keypress Capture

```typescript
// KeypressContext.tsx
export function KeypressProvider({ children }) {
  const [input, setInput] = useState('');

  useInput((inputChar, key) => {
    if (key.return) {
      handleSubmit(input);
      setInput('');
    } else if (key.backspace) {
      setInput(prev => prev.slice(0, -1));
    } else {
      setInput(prev => prev + inputChar);
    }
  });

  return (
    <KeypressContext.Provider value={{ input, setInput }}>
      {children}
    </KeypressContext.Provider>
  );
}
```

#### Step 2: Slash Command Processing

```typescript
// useSlashCommandProcessor.ts
export function useSlashCommandProcessor() {
  const processInput = async (input: string) => {
    // Check if slash command
    if (input.startsWith('/')) {
      const [command, ...args] = input.slice(1).split(' ');

      switch (command) {
        case 'model':
          await handleModelCommand(args);
          return { handled: true };

        case 'help':
          await handleHelpCommand();
          return { handled: true };

        case 'clear':
          await handleClearCommand();
          return { handled: true };

        default:
          // Check for custom commands
          const customCommand = await loadCustomCommand(command);
          if (customCommand) {
            await executeCustomCommand(customCommand, args);
            return { handled: true };
          }
      }
    }

    return { handled: false, message: input };
  };

  return { processInput };
}
```

#### Step 3: Message Submission

```typescript
// AppContainer.tsx
const handleSubmit = async (input: string) => {
  // 1. Process slash commands
  const { handled, message } = await processInput(input);
  if (handled) return;

  // 2. Add user message to history
  const userMessage: Message = {
    id: generateId(),
    role: 'user',
    content: [{ role: 'user', parts: [{ text: message }] }],
    timestamp: Date.now(),
  };
  setMessages(prev => [...prev, userMessage]);

  // 3. Set processing state
  setIsProcessing(true);

  // 4. Stream response
  try {
    await streamGeminiResponse(message);
  } catch (error) {
    handleError(error);
  } finally {
    setIsProcessing(false);
  }
};
```

---

## Tool Calling Flow

### Overview: Model Decides Which Tools to Call

```
User Message → Gemini API
    │
    ├─> Gemini analyzes message + tool schemas
    │   └─ Model has access to:
    │      ├─ Tool names
    │      ├─ Tool descriptions
    │      └─ Parameter schemas
    │
    ├─> Gemini decides:
    │   ├─ Which tools to call (0, 1, or multiple)
    │   ├─ What parameters to pass
    │   └─ In what order (parallel in single turn)
    │
    └─> Returns:
        ├─ Text response (optional)
        └─ Function calls array
```

### Detailed Tool Calling Flow

#### Step 1: Prepare Tools for Model

**File**: `packages/core/src/agents/executor.ts:930-982`

```typescript
private prepareToolsList(): FunctionDeclaration[] {
  // 1. Get all registered tools
  const allTools = this.toolRegistry.getAllToolNames();

  // 2. Filter by agent's allowed tools
  const allowedTools = allTools.filter(name =>
    this.agentDefinition.tools.includes(name)
  );

  // 3. Get function declarations
  const declarations = allowedTools.map(name => {
    const tool = this.toolRegistry.getTool(name);
    return tool.schema;  // { name, description, parameters }
  });

  // 4. Add mandatory complete_task tool
  declarations.push({
    name: 'complete_task',
    description: 'Call this when task is complete',
    parameters: {
      type: 'object',
      properties: {
        result: { type: 'string' },
      },
    },
  });

  return declarations;
}
```

#### Step 2: Send to Gemini API

**File**: `packages/core/src/agents/executor.ts:591-649`

```typescript
private async callModel(
  chat: Chat,
  message: Content,
  signal: AbortSignal
): Promise<ModelResponse> {
  // 1. Prepare tools
  const tools = this.prepareToolsList();

  // 2. Build request
  const request = {
    contents: [message],
    tools: [{ functionDeclarations: tools }],
    generationConfig: this.modelConfig.generationConfig,
  };

  // 3. Call API with streaming
  const stream = await this.client.generateContentStream(request);

  // 4. Collect response
  let text = '';
  const functionCalls: FunctionCall[] = [];

  for await (const chunk of stream) {
    // Text response
    if (chunk.text) {
      text += chunk.text;
    }

    // Function calls
    if (chunk.candidates?.[0]?.content?.parts) {
      for (const part of chunk.candidates[0].content.parts) {
        if (part.functionCall) {
          functionCalls.push({
            name: part.functionCall.name,
            args: part.functionCall.args,
          });
        }
      }
    }
  }

  return { text, functionCalls };
}
```

#### Step 3: Validate Tool Calls

**File**: `packages/core/src/agents/executor.ts:840-861`

```typescript
private validateFunctionCalls(
  functionCalls: FunctionCall[]
): ValidationResult {
  const allowedToolNames = new Set(
    this.toolRegistry.getAllToolNames()
  );
  allowedToolNames.add('complete_task');

  for (const fc of functionCalls) {
    // Check if tool exists
    if (!allowedToolNames.has(fc.name)) {
      return {
        valid: false,
        error: `Unauthorized tool: ${fc.name}`,
      };
    }

    // Validate parameters (schema validation)
    const tool = this.toolRegistry.getTool(fc.name);
    const validation = tool.validateParams(fc.args);

    if (!validation.valid) {
      return {
        valid: false,
        error: `Invalid params for ${fc.name}: ${validation.error}`,
      };
    }
  }

  return { valid: true };
}
```

#### Step 4: Execute Tools (Parallel)

**File**: `packages/core/src/agents/executor.ts:872-925`

```typescript
private async processFunctionCalls(
  functionCalls: FunctionCall[],
  signal: AbortSignal
): Promise<Content> {
  // Create execution promises for all function calls
  const toolExecutionPromises = functionCalls.map(async (fc, index) => {
    try {
      // 1. Get tool from registry
      const tool = this.toolRegistry.getTool(fc.name);

      // 2. Build invocation
      const invocation = tool.build(fc.args);

      // 3. Execute tool
      const result = await invocation.execute(signal);

      // 4. Return response part
      return {
        functionResponse: {
          name: fc.name,
          response: result.llmContent,
        },
      };
    } catch (error) {
      // Return error response
      return {
        functionResponse: {
          name: fc.name,
          response: {
            error: error.message,
          },
        },
      };
    }
  });

  // Execute ALL tools in parallel
  const responseParts = await Promise.all(toolExecutionPromises);

  // Return as function response content
  return {
    role: 'function',
    parts: responseParts,
  };
}
```

---

## Agent Execution Loop Flow

### High-Level Agent Loop

```
Agent.run(inputs)
    │
    ├─> Initialize Chat
    │   └─ Create chat with system prompt
    │
    ├─> Build Initial Message
    │   └─ Inject user inputs into prompt
    │
    └─> WHILE turnCount < maxTurns:
        │
        ├─> Execute Turn
        │   ├─ Call Model (with tools)
        │   ├─ Check for complete_task
        │   └─ Process Function Calls (parallel)
        │
        ├─> Check Termination
        │   ├─ complete_task called?
        │   ├─ Max turns reached?
        │   └─ Timeout?
        │
        └─> Continue or Return Result
```

### Detailed Agent Loop

**File**: `packages/core/src/agents/executor.ts:366-556`

```typescript
export class AgentExecutor {
  async run(
    inputs: Record<string, unknown>,
    signal: AbortSignal
  ): Promise<AgentResult> {
    // ===== INITIALIZATION =====

    // 1. Initialize chat with system prompt
    const chat = this.client.startChat({
      systemInstruction: this.agentDefinition.systemPrompt,
      generationConfig: this.modelConfig.generationConfig,
    });

    // 2. Build initial message with user inputs
    const initialMessage = this.buildInitialMessage(inputs);

    // 3. Initialize loop variables
    let currentMessage: Content = initialMessage;
    let turnCount = 0;
    const maxTurns = this.agentDefinition.maxTurns || 25;

    // ===== MAIN EXECUTION LOOP =====

    while (turnCount < maxTurns) {
      // Increment turn counter
      turnCount++;

      // Log turn start
      this.logger.debug(`Turn ${turnCount}/${maxTurns} starting`);

      try {
        // ===== EXECUTE TURN =====
        const turnResult = await this.executeTurn(
          chat,
          currentMessage,
          signal
        );

        // ===== CHECK TERMINATION =====
        if (turnResult.shouldTerminate) {
          this.logger.info('Agent completed task', {
            turnCount,
            result: turnResult.result,
          });

          return {
            result: turnResult.result,
            terminate_reason: 'task_complete',
            turns: turnCount,
          };
        }

        // ===== PREPARE NEXT TURN =====
        currentMessage = turnResult.nextMessage;

      } catch (error) {
        // Handle errors
        if (signal.aborted) {
          return {
            result: null,
            terminate_reason: 'cancelled',
            turns: turnCount,
          };
        }

        throw error;
      }
    }

    // ===== MAX TURNS EXCEEDED =====
    this.logger.warn('Max turns exceeded', { maxTurns });

    // Attempt recovery
    const recoveryResult = await this.executeFinalWarningTurn(chat, signal);

    return {
      result: recoveryResult,
      terminate_reason: 'max_turns',
      turns: turnCount,
    };
  }
}
```

### Single Turn Execution

**File**: `packages/core/src/agents/executor.ts:182-239`

```typescript
private async executeTurn(
  chat: Chat,
  message: Content,
  signal: AbortSignal
): Promise<TurnResult> {
  // ===== STEP 1: CALL MODEL =====

  const modelResponse = await this.callModel(chat, message, signal);

  this.logger.debug('Model response received', {
    textLength: modelResponse.text?.length,
    functionCallCount: modelResponse.functionCalls?.length,
  });

  // ===== STEP 2: CHECK FOR TASK COMPLETION =====

  const completeTaskCall = modelResponse.functionCalls?.find(
    fc => fc.name === 'complete_task'
  );

  if (completeTaskCall) {
    // Validate output if schema provided
    if (this.agentDefinition.outputConfig) {
      const validation = this.validateOutput(
        completeTaskCall.args.result,
        this.agentDefinition.outputConfig.schema
      );

      if (!validation.valid) {
        // Return error to model
        return {
          shouldTerminate: false,
          nextMessage: {
            role: 'function',
            parts: [{
              functionResponse: {
                name: 'complete_task',
                response: {
                  error: `Invalid output: ${validation.error}`,
                },
              },
            }],
          },
        };
      }
    }

    // Task complete - terminate
    return {
      shouldTerminate: true,
      result: completeTaskCall.args.result,
    };
  }

  // ===== STEP 3: PROCESS FUNCTION CALLS =====

  if (modelResponse.functionCalls && modelResponse.functionCalls.length > 0) {
    // Validate function calls
    const validation = this.validateFunctionCalls(
      modelResponse.functionCalls
    );

    if (!validation.valid) {
      // Return error to model
      return {
        shouldTerminate: false,
        nextMessage: {
          role: 'model',
          parts: [{ text: `Error: ${validation.error}` }],
        },
      };
    }

    // Execute function calls (PARALLEL)
    const nextMessage = await this.processFunctionCalls(
      modelResponse.functionCalls,
      signal
    );

    return {
      shouldTerminate: false,
      nextMessage,
    };
  }

  // ===== NO FUNCTION CALLS - MODEL JUST RESPONDED =====

  // This shouldn't happen in agent mode
  this.logger.warn('Model responded with text only, no function calls');

  return {
    shouldTerminate: false,
    nextMessage: {
      role: 'user',
      parts: [{ text: 'Please call a tool or complete_task' }],
    },
  };
}
```

---

## Tool Scheduler Flow

### Sequential Tool Execution

The tool scheduler ensures tools execute **one at a time** with user approval where needed.

```
schedule(toolRequest)
    │
    ├─> Check if scheduler is running
    │   ├─ YES: Add to requestQueue, return
    │   └─ NO: Continue
    │
    ├─> _schedule(request)
    │   ├─ Get tool from registry
    │   ├─ Build invocation
    │   ├─ Add to toolCallQueue
    │   └─ _processNextInQueue()
    │
    └─> _processNextInQueue()
        │
        ├─> Pop from toolCallQueue
        ├─> Add to active toolCalls
        │
        ├─> VALIDATION Phase
        │   └─ invocation.shouldConfirmExecute()
        │
        ├─> CONFIRMATION Phase (if needed)
        │   ├─ Query policy engine
        │   ├─ Ask user (if policy = ASK_USER)
        │   └─ Wait for response
        │
        ├─> SCHEDULED Phase
        │   └─ Mark as ready to execute
        │
        ├─> EXECUTION Phase
        │   ├─ invocation.execute(signal, onOutput)
        │   └─ Stream output to UI
        │
        ├─> COMPLETION
        │   ├─ Mark as successful/errored
        │   ├─ Move to completedBatch
        │   ├─ Call onAllToolCallsComplete
        │   └─ Process next in queue
        │
        └─> Repeat until queue empty
```

### Detailed Scheduler Flow

**File**: `packages/core/src/core/coreToolScheduler.ts:668-945`

#### Main Schedule Method

```typescript
export class CoreToolScheduler {
  async schedule(
    request: ToolCallRequestInfo | ToolCallRequestInfo[],
    signal: AbortSignal
  ): Promise<void> {
    // Handle array of requests
    const requests = Array.isArray(request) ? request : [request];

    // ===== CHECK IF RUNNING =====
    if (this.isRunning()) {
      // Add to queue and wait
      return new Promise((resolve, reject) => {
        this.requestQueue.push({
          request: requests,
          signal,
          resolve,
          reject,
        });
      });
    }

    // ===== PROCESS EACH REQUEST =====
    for (const req of requests) {
      await this._schedule(req, signal);
    }
  }

  private isRunning(): boolean {
    return (
      this.isFinalizingToolCalls ||
      this.toolCalls.some(
        tc => tc.status === 'executing' || tc.status === 'awaiting_approval'
      )
    );
  }
}
```

#### Schedule Individual Tool

```typescript
private async _schedule(
  request: ToolCallRequestInfo,
  signal: AbortSignal
): Promise<void> {
  // ===== STEP 1: GET TOOL =====
  const tool = this.toolRegistry.getTool(request.toolName);

  if (!tool) {
    throw new Error(`Tool not found: ${request.toolName}`);
  }

  // ===== STEP 2: BUILD INVOCATION =====
  let invocation: ToolInvocation;
  try {
    invocation = tool.build(request.params);
  } catch (error) {
    throw new Error(`Failed to build invocation: ${error.message}`);
  }

  // ===== STEP 3: CREATE TOOL CALL =====
  const toolCall: ToolCall = {
    id: generateId(),
    status: 'validating',
    toolName: request.toolName,
    params: request.params,
    invocation,
    requestInfo: request,
  };

  // ===== STEP 4: ADD TO QUEUE =====
  this.toolCallQueue.push(toolCall);

  // ===== STEP 5: PROCESS QUEUE =====
  await this._processNextInQueue();
}
```

#### Process Queue

```typescript
private async _processNextInQueue(): Promise<void> {
  // ===== GET NEXT TOOL CALL =====
  const toolCall = this.toolCallQueue.shift();
  if (!toolCall) return;

  // ===== ADD TO ACTIVE LIST =====
  this.toolCalls.push(toolCall);

  try {
    // ===== VALIDATION PHASE =====
    toolCall.status = 'validating';

    const needsConfirmation = await toolCall.invocation
      .shouldConfirmExecute(this.signal);

    // ===== CONFIRMATION PHASE =====
    if (needsConfirmation) {
      toolCall.status = 'awaiting_approval';

      // Emit event for UI
      this.emit('tool_awaiting_approval', toolCall);

      // Get policy decision
      const decision = await this.policyEngine.evaluateToolCall(
        toolCall.toolName,
        toolCall.params
      );

      if (decision.decision === 'DENY') {
        throw new Error('Policy denied tool execution');
      }

      if (decision.decision === 'ASK_USER') {
        // Wait for user approval
        const approved = await this.waitForUserApproval(toolCall);

        if (!approved) {
          toolCall.status = 'cancelled';
          this.emit('tool_cancelled', toolCall);
          return;
        }
      }
    }

    // ===== SCHEDULED PHASE =====
    toolCall.status = 'scheduled';
    this.emit('tool_scheduled', toolCall);

    // ===== EXECUTE =====
    await this.attemptExecutionOfScheduledCalls();

  } catch (error) {
    toolCall.status = 'errored';
    toolCall.error = error;
    this.emit('tool_errored', toolCall);

    await this.handleError(toolCall);
  }
}
```

#### Execute Scheduled Calls

```typescript
private async attemptExecutionOfScheduledCalls(): Promise<void> {
  // Get all scheduled tool calls
  const scheduledCalls = this.toolCalls.filter(
    tc => tc.status === 'scheduled'
  );

  // Execute each one sequentially
  for (const toolCall of scheduledCalls) {
    // ===== MARK AS EXECUTING =====
    toolCall.status = 'executing';
    this.emit('tool_executing', toolCall);

    try {
      // ===== EXECUTE WITH OUTPUT STREAMING =====
      const result = await toolCall.invocation.execute(
        this.signal,
        (output: OutputChunk) => {
          // Stream output to UI in real-time
          this.emit('tool_output', {
            toolCallId: toolCall.id,
            output,
          });
        }
      );

      // ===== MARK AS SUCCESSFUL =====
      toolCall.status = 'successful';
      toolCall.result = result;
      this.emit('tool_completed', toolCall);

    } catch (error) {
      // ===== MARK AS ERRORED =====
      toolCall.status = 'errored';
      toolCall.error = error;
      this.emit('tool_errored', toolCall);
    }

    // ===== MOVE TO COMPLETED BATCH =====
    this.completedToolCallsForBatch.push(toolCall);
    this.toolCalls = this.toolCalls.filter(tc => tc.id !== toolCall.id);

    // ===== NOTIFY COMPLETION =====
    await this.checkAndNotifyCompletion();
  }
}
```

#### Completion Notification

```typescript
private async checkAndNotifyCompletion(): Promise<void> {
  // Check if all tool calls in batch are complete
  if (this.toolCalls.length === 0 && this.toolCallQueue.length === 0) {
    // ===== ALL COMPLETE =====

    // Call completion callback
    if (this.onAllToolCallsComplete) {
      await this.onAllToolCallsComplete(
        this.completedToolCallsForBatch
      );
    }

    // Clear batch
    this.completedToolCallsForBatch = [];

    // ===== PROCESS NEXT REQUEST FROM QUEUE =====
    if (this.requestQueue.length > 0) {
      const nextRequest = this.requestQueue.shift();

      try {
        await this.schedule(
          nextRequest.request,
          nextRequest.signal
        );
        nextRequest.resolve();
      } catch (error) {
        nextRequest.reject(error);
      }
    }
  }
}
```

---

## Complete Request-Response Cycle

### End-to-End Flow Example

**User Message**: "Read the package.json file and tell me the version"

```
┌─────────────────────────────────────────────────────────┐
│ USER INPUT PHASE                                        │
└─────────────────────────────────────────────────────────┘

1. User types: "Read the package.json file and tell me the version"
   └─ File: AppContainer.tsx:handleSubmit()

2. Add user message to history
   └─ State: messages = [...messages, userMessage]

3. Call useGeminiStream hook
   └─ File: useGeminiStream.ts:streamMessage()

┌─────────────────────────────────────────────────────────┐
│ API REQUEST PHASE                                       │
└─────────────────────────────────────────────────────────┘

4. Build API request
   ├─ contents: [{ role: 'user', parts: [{ text: message }] }]
   ├─ tools: [{ functionDeclarations: [...all tools...] }]
   └─ generationConfig: { temperature: 0.7, ... }

5. Send to Gemini API
   └─ client.generateContentStream(request)

┌─────────────────────────────────────────────────────────┐
│ MODEL DECISION PHASE                                    │
└─────────────────────────────────────────────────────────┘

6. Gemini analyzes:
   ├─ User wants to read a file
   ├─ Filename: package.json
   └─ Available tool: read_file

7. Gemini decides to call:
   └─ read_file({ file_path: "package.json" })

8. API streams response:
   └─ functionCall: {
       name: "read_file",
       args: { file_path: "package.json" }
     }

┌─────────────────────────────────────────────────────────┐
│ TOOL EXECUTION PHASE                                    │
└─────────────────────────────────────────────────────────┘

9. Agent executor receives function call
   └─ File: executor.ts:processFunctionCalls()

10. Get tool from registry
    └─ tool = toolRegistry.getTool('read_file')

11. Build invocation
    └─ invocation = tool.build({ file_path: "package.json" })

12. Execute tool
    ├─ invocation.execute(signal)
    ├─ Read file from filesystem
    └─ Format with line numbers

13. Return result:
    {
      llmContent: "1\t{\n2\t  \"name\": \"gemini-cli\",\n3\t  \"version\": \"0.15.0-nightly\"\n...",
      returnDisplay: "Read package.json (100 lines)"
    }

┌─────────────────────────────────────────────────────────┐
│ MODEL RESPONSE PHASE                                    │
└─────────────────────────────────────────────────────────┘

14. Send tool result back to Gemini
    └─ contents: [{
        role: 'function',
        parts: [{
          functionResponse: {
            name: 'read_file',
            response: result.llmContent
          }
        }]
      }]

15. Gemini analyzes file content
    └─ Finds "version": "0.15.0-nightly" on line 3

16. Gemini generates final response:
    └─ "The version in package.json is 0.15.0-nightly"

┌─────────────────────────────────────────────────────────┐
│ UI RENDERING PHASE                                      │
└─────────────────────────────────────────────────────────┘

17. Stream text response to UI
    └─ For each chunk: setResponse(prev => prev + chunk)

18. Render complete message
    ├─ User: "Read the package.json file..."
    ├─ Assistant used: read_file
    └─ Assistant: "The version in package.json is 0.15.0-nightly"

19. Add to chat history
    └─ Save to recording service for persistence

20. Set isProcessing = false
    └─ Ready for next user input
```

### Multi-Turn Example

**User Message**: "Find all TypeScript files in src/ and count them"

```
┌─────────────────────────────────────────────────────────┐
│ TURN 1: Initial Request                                │
└─────────────────────────────────────────────────────────┘

User: "Find all TypeScript files in src/ and count them"

Model Decision:
  └─ Call: glob({ pattern: "src/**/*.ts" })

Tool Execution:
  └─ Returns: ["src/index.ts", "src/utils.ts", "src/config.ts"]

Model receives: List of 3 files

┌─────────────────────────────────────────────────────────┐
│ TURN 2: Model Analyzes Results                         │
└─────────────────────────────────────────────────────────┘

Model Decision:
  └─ Call: complete_task({ result: "Found 3 TypeScript files..." })

Agent Executor:
  └─ Detects complete_task → terminate loop

Final Response:
  └─ "Found 3 TypeScript files in src/:
      - src/index.ts
      - src/utils.ts
      - src/config.ts"

Total Turns: 2
```

### Complex Multi-Tool Example

**User Message**: "Fix the bug in auth.ts"

```
┌─────────────────────────────────────────────────────────┐
│ TURN 1: Locate File                                    │
└─────────────────────────────────────────────────────────┘

Model Decision:
  └─ Call: glob({ pattern: "**/auth.ts" })

Result:
  └─ ["src/auth.ts"]

┌─────────────────────────────────────────────────────────┐
│ TURN 2: Read File                                      │
└─────────────────────────────────────────────────────────┘

Model Decision:
  └─ Call: read_file({ file_path: "src/auth.ts" })

Result:
  └─ File content with line numbers

┌─────────────────────────────────────────────────────────┐
│ TURN 3: Search for Bug Patterns                        │
└─────────────────────────────────────────────────────────┘

Model Decision (PARALLEL):
  ├─ Call: search_file_content({
      pattern: "null|undefined",
      path: "src/auth.ts"
    })
  └─ Call: search_file_content({
      pattern: "TODO|FIXME|BUG",
      path: "src/auth.ts"
    })

Results:
  ├─ Found "null" on lines 45, 67
  └─ Found "TODO: Fix null check" on line 45

┌─────────────────────────────────────────────────────────┐
│ TURN 4: Fix the Bug                                    │
└─────────────────────────────────────────────────────────┘

Model Decision:
  └─ Call: replace({
      file_path: "src/auth.ts",
      old_string: "if (user == null)",
      new_string: "if (!user)"
    })

Tool Scheduler:
  ├─ Status: awaiting_approval
  ├─ Ask user: "Replace code in src/auth.ts?"
  └─ User approves

Result:
  └─ File edited successfully

┌─────────────────────────────────────────────────────────┐
│ TURN 5: Verify Fix                                     │
└─────────────────────────────────────────────────────────┘

Model Decision:
  └─ Call: run_shell_command({
      command: "npm test src/auth.test.ts"
    })

Tool Scheduler:
  ├─ Status: awaiting_approval
  ├─ Ask user: "Run 'npm test src/auth.test.ts'?"
  └─ User approves

Result:
  └─ All tests passed

┌─────────────────────────────────────────────────────────┐
│ TURN 6: Complete                                       │
└─────────────────────────────────────────────────────────┘

Model Decision:
  └─ Call: complete_task({
      result: "Fixed null check bug in src/auth.ts:45. Tests passing."
    })

Final Response:
  └─ "I've fixed the bug in src/auth.ts. The issue was an incorrect
      null check on line 45. I replaced `if (user == null)` with
      `if (!user)` for better JavaScript practice. All tests are
      now passing."

Total Turns: 6
Tools Called: glob, read_file, search_file_content (x2), replace, run_shell_command
```

---

## Error Handling Flow

### Tool Execution Error

```
Tool.execute()
    │
    ├─> Try executing
    │
    ├─> Error occurs
    │   └─ Catch error
    │
    ├─> Wrap error in ToolResult
    │   └─ { error: { message, code, details } }
    │
    ├─> Return to Agent Executor
    │
    ├─> Send error to Model
    │   └─ functionResponse: { error: "..." }
    │
    └─> Model decides:
        ├─ Retry with different params
        ├─ Try alternative approach
        └─ Or report error to user
```

### Example Error Handling

```typescript
// In tool invocation
async execute(signal: AbortSignal): Promise<ToolResult> {
  try {
    const result = await fs.readFile(this.params.file_path);
    return {
      llmContent: result,
      returnDisplay: `Read ${this.params.file_path}`,
    };
  } catch (error) {
    // Return error as tool result
    return {
      llmContent: '',
      error: {
        message: `Failed to read file: ${error.message}`,
        code: error.code,
      },
    };
  }
}
```

### Agent-Level Error Handling

```typescript
// In agent executor
try {
  const turnResult = await this.executeTurn(chat, currentMessage, signal);
  // ...
} catch (error) {
  // Check for abort
  if (signal.aborted) {
    return {
      result: null,
      terminate_reason: 'cancelled',
      turns: turnCount,
    };
  }

  // Check for quota errors
  if (this.isQuotaError(error)) {
    return {
      result: null,
      terminate_reason: 'quota_exceeded',
      error: error.message,
    };
  }

  // Retry with backoff
  if (this.shouldRetry(error, retryCount)) {
    await this.sleep(this.getBackoffDelay(retryCount));
    retryCount++;
    continue;
  }

  // Unrecoverable error
  throw error;
}
```

---

## MCP Integration Flow

### MCP Server Discovery

```
App Startup
    │
    ├─> Load Settings
    │   └─ Read mcpServers from settings.json
    │
    ├─> For each MCP server:
    │   │
    │   ├─> Start MCP server process
    │   │   └─ spawn(command, args, { env })
    │   │
    │   ├─> Create stdio transport
    │   │   └─ StdioClientTransport({ stdin, stdout })
    │   │
    │   ├─> Initialize MCP client
    │   │   └─ client.connect(transport)
    │   │
    │   ├─> List available tools
    │   │   └─ client.request({ method: 'tools/list' })
    │   │
    │   └─> Register each tool
    │       └─ toolRegistry.registerTool(new DiscoveredMCPTool(...))
    │
    └─> MCP tools now available
```

### MCP Tool Invocation

```
Model calls MCP tool
    │
    ├─> Agent executor receives function call
    │
    ├─> Get DiscoveredMCPTool from registry
    │
    ├─> Build invocation
    │   └─ invocation.mcpClient = client for that server
    │
    ├─> Execute invocation
    │   │
    │   ├─> Send to MCP server
    │   │   └─ client.request({
    │   │       method: 'tools/call',
    │   │       params: { name, arguments }
    │   │     })
    │   │
    │   ├─> MCP server executes
    │   │
    │   └─> Return result
    │
    └─> Send result back to model
```

---

## Real-World Examples

### Example 1: Simple File Read

**Request**: "Show me the README file"

**Flow**:
```
1. User input → "Show me the README file"

2. Gemini API call
   └─ Tools: [read_file, glob, ...]

3. Model decision
   └─ Call: read_file({ file_path: "README.md" })

4. Tool execution
   ├─ Read README.md
   └─ Return: File content (100 lines)

5. Model response
   └─ "Here's the README:
       [formatted content]"

Timeline: ~2-3 seconds
Turns: 1
Tools: 1 (read_file)
```

### Example 2: Search and Replace

**Request**: "Replace all instances of 'var' with 'const' in utils.js"

**Flow**:
```
1. User input → "Replace all instances of 'var' with 'const' in utils.js"

2. Turn 1: Read file to verify
   ├─ Model calls: read_file({ file_path: "utils.js" })
   └─ Result: File content with 5 instances of 'var'

3. Turn 2: Perform replacement
   ├─ Model calls: replace({
   │    file_path: "utils.js",
   │    old_string: "var ",
   │    new_string: "const ",
   │    replace_all: true
   │  })
   ├─ Policy check: ALLOW (trusted folder)
   └─ Result: 5 replacements made

4. Turn 3: Complete
   └─ Model calls: complete_task({
       result: "Replaced 5 instances of 'var' with 'const'"
     })

Timeline: ~5-7 seconds
Turns: 3
Tools: 2 (read_file, replace)
```

### Example 3: Complex Investigation

**Request**: "Analyze the test coverage and find untested functions"

**Flow**:
```
1. Turn 1: Discover test files
   ├─ Call: glob({ pattern: "**/*.test.ts" })
   └─ Found: 15 test files

2. Turn 2: Read test files (PARALLEL)
   ├─ Call: read_file({ file_path: "test1.test.ts" })
   ├─ Call: read_file({ file_path: "test2.test.ts" })
   ├─ ... (15 parallel calls)
   └─ Results: All test file contents

3. Turn 3: Find source files
   ├─ Call: glob({ pattern: "src/**/*.ts" })
   └─ Found: 45 source files

4. Turn 4: Search for function definitions
   ├─ Call: search_file_content({
   │    pattern: "function|export const.*=.*=>",
   │    glob: "src/**/*.ts"
   │  })
   └─ Found: 120 function definitions

5. Turn 5: Run coverage command
   ├─ Call: run_shell_command({ command: "npm run coverage" })
   ├─ User approval: YES
   └─ Result: Coverage report

6. Turn 6: Analyze and complete
   └─ Call: complete_task({
       result: {
         totalFunctions: 120,
         testedFunctions: 95,
         untestedFunctions: [...],
         coverage: "79%"
       }
     })

Timeline: ~20-30 seconds
Turns: 6
Tools: 4 (glob x2, read_file x15, search_file_content, run_shell_command)
```

### Example 4: Debugging Session

**Request**: "The app crashes when I click login. Debug it."

**Flow**:
```
1. Turn 1: Find login code
   ├─ Call: search_file_content({
   │    pattern: "login|onClick.*login",
   │    glob: "src/**/*.{ts,tsx}"
   │  })
   └─ Found: src/components/LoginButton.tsx:45

2. Turn 2: Read the file
   ├─ Call: read_file({ file_path: "src/components/LoginButton.tsx" })
   └─ Result: Component code

3. Turn 3: Check for error handling
   ├─ Call: search_file_content({
   │    pattern: "try|catch|error",
   │    path: "src/components/LoginButton.tsx"
   │  })
   └─ Found: No error handling!

4. Turn 4: Check logs
   ├─ Call: search_file_content({
   │    pattern: "console\\.error|logger\\.error",
   │    glob: "**/*.log"
   │  })
   └─ Found: TypeError: Cannot read property 'token' of undefined

5. Turn 5: Identify the issue
   ├─ Call: read_file({
   │    file_path: "src/components/LoginButton.tsx",
   │    offset: 40,
   │    limit: 20
   │  })
   └─ Analysis: Line 45 accesses response.data.token without null check

6. Turn 6: Fix the bug
   ├─ Call: replace({
   │    file_path: "src/components/LoginButton.tsx",
   │    old_string: "const token = response.data.token;",
   │    new_string: "const token = response?.data?.token ?? null;"
   │  })
   ├─ User approval: YES
   └─ Result: Fixed

7. Turn 7: Verify fix
   ├─ Call: run_shell_command({ command: "npm run build" })
   ├─ User approval: YES
   └─ Result: Build successful

8. Turn 8: Complete
   └─ Call: complete_task({
       result: "Fixed crash in LoginButton.tsx:45. Added null safety."
     })

Timeline: ~30-40 seconds
Turns: 8
Tools: 5 (search_file_content x3, read_file x2, replace, run_shell_command)
User Approvals: 2
```

---

## Summary

This flow documentation demonstrates:

1. **Startup Flow**: Configuration → Initialization → UI Rendering
2. **User Input Flow**: Keypress → Validation → API Call
3. **Tool Calling Flow**: Model Decision → Validation → Execution
4. **Agent Loop Flow**: Turn-based execution with termination conditions
5. **Scheduler Flow**: Sequential execution with approval gates
6. **Error Handling**: Graceful degradation and retry logic
7. **MCP Integration**: External tool server integration
8. **Real Examples**: Simple to complex multi-tool workflows

The system's power comes from:
- **Model-driven tool selection** (AI decides which tools to use)
- **Parallel execution within turns** (efficiency)
- **Sequential turns** (controlled, observable behavior)
- **User approval gates** (safety for dangerous operations)
- **Comprehensive error handling** (robustness)

This architecture enables Gemini CLI to autonomously solve complex, multi-step tasks while maintaining safety and user control.
