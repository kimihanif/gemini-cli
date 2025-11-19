# Hooks System: Code Flow and Architecture

## Current Implementation Status

### ✅ **Implemented Components**

1. **Type System** (`packages/core/src/hooks/types.ts`)
   - All 11 hook event types defined
   - Input/Output interfaces for each event
   - Hook configuration structures

2. **Hook Registry** (`packages/core/src/hooks/hookRegistry.ts`)
   - Loads hooks from configuration
   - Validates hook definitions
   - Manages hook priorities (Project > User > Extensions)

3. **Hook Planner** (`packages/core/src/hooks/hookPlanner.ts`)
   - Matches hooks to events
   - Filters by matchers (regex)
   - Creates execution plans (sequential vs parallel)

4. **Hook Translator** (`packages/core/src/hooks/hookTranslator.ts`)
   - Converts between SDK and hook formats
   - Handles LLM request/response translation

5. **Configuration Integration** (`packages/core/src/config/config.ts`)
   - `enableHooks` flag
   - Hooks configuration storage
   - `getHooks()` method

### ⚠️ **Missing/Not Visible Components**

1. **Hook Runner/Executor** - The actual execution engine
2. **Integration Points** - Where hooks are triggered in tool/LLM flow
3. **Shell Command Executor** - Process spawning for hook commands
4. **Result Aggregation** - Combining multiple hook outputs

**Note**: The infrastructure is complete, but the execution wiring may be in
progress or in a separate branch.

---

## Architecture Overview

### Component Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│  Configuration Layer (.gemini/settings.json)                    │
├─────────────────────────────────────────────────────────────────┤
│  {                                                              │
│    "tools": { "enableHooks": true },                            │
│    "hooks": {                                                   │
│      "BeforeTool": [...],                                       │
│      "AfterTool": [...]                                         │
│    }                                                            │
│  }                                                              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Config Class (packages/core/src/config/config.ts)              │
├─────────────────────────────────────────────────────────────────┤
│  - Stores hooks configuration                                   │
│  - Provides getHooks() method                                   │
│  - Manages enableHooks flag                                     │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  HookRegistry (packages/core/src/hooks/hookRegistry.ts)         │
├─────────────────────────────────────────────────────────────────┤
│  initialize():                                                  │
│  1. Read hooks from Config.getHooks()                           │
│  2. Process hooks from extensions                               │
│  3. Validate each hook definition                               │
│  4. Create HookRegistryEntry for each                           │
│  5. Store in entries array                                      │
│                                                                 │
│  getHooksForEvent(eventName):                                   │
│  - Filter entries by event name                                 │
│  - Filter by enabled status                                     │
│  - Sort by source priority                                      │
│  - Return matching entries                                      │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  HookPlanner (packages/core/src/hooks/hookPlanner.ts)           │
├─────────────────────────────────────────────────────────────────┤
│  createExecutionPlan(eventName, context):                       │
│  1. Get hooks for event from registry                           │
│  2. Filter by matcher (tool name, trigger)                      │
│  3. Deduplicate identical hooks                                 │
│  4. Determine execution mode (sequential vs parallel)           │
│  5. Return HookExecutionPlan                                    │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Hook Executor (NOT FOUND IN VISIBLE CODE)                      │
├─────────────────────────────────────────────────────────────────┤
│  executeHooks(plan, input):                                     │
│  [INTENDED IMPLEMENTATION]                                      │
│  1. Prepare event-specific JSON input                           │
│  2. For each hook in plan:                                      │
│     a. Spawn shell process (command)                            │
│     b. Write JSON to stdin                                      │
│     c. Read stdout (hook output)                                │
│     d. Parse JSON output                                        │
│     e. Create HookExecutionResult                               │
│  3. Aggregate results                                           │
│  4. Return combined HookOutput                                  │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Decision Application (INTENDED)                                │
├─────────────────────────────────────────────────────────────────┤
│  - Check for blocking decisions (block/deny)                    │
│  - Apply modifications (LLM request/response, tool config)      │
│  - Extract additional context                                   │
│  - Return control to caller                                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Flow: Tool Execution Example

### Example Scenario

User requests: "Edit the file config.json to add a new setting"

This triggers the `EditTool` which should execute `BeforeTool` and `AfterTool`
hooks.

---

### Step 1: Configuration Loading

**File**: `packages/core/src/config/config.ts`

```typescript
// In Config class constructor
constructor(params: ConfigParameters) {
  // ... other initialization ...

  this.enableHooks = params.enableHooks ?? false;
  this.hooks = params.hooks;  // Loaded from settings.json
}

// Getter method
getHooks(): { [K in HookEventName]?: HookDefinition[] } | undefined {
  return this.hooks;
}
```

**What happens**:

- Configuration from `.gemini/settings.json` is loaded
- Hooks definitions are stored in Config instance
- `enableHooks` flag is set

---

### Step 2: Hook Registry Initialization

**File**: `packages/core/src/hooks/hookRegistry.ts`

```typescript
export class HookRegistry {
  private entries: HookRegistryEntry[] = [];

  async initialize(): Promise<void> {
    if (this.initialized) return;

    this.entries = [];
    this.processHooksFromConfig(); // <-- Loads hooks
    this.initialized = true;
  }

  private processHooksFromConfig(): void {
    // Get hooks from config
    const configHooks = this.config.getHooks();
    if (configHooks) {
      this.processHooksConfiguration(configHooks, ConfigSource.Project);
    }

    // Get hooks from extensions
    const extensions = this.config.getExtensions() || [];
    for (const extension of extensions) {
      if (extension.isActive && extension.hooks) {
        this.processHooksConfiguration(
          extension.hooks,
          ConfigSource.Extensions,
        );
      }
    }
  }

  private processHooksConfiguration(
    hooksConfig: { [K in HookEventName]?: HookDefinition[] },
    source: ConfigSource,
  ): void {
    // For each event type (BeforeTool, AfterTool, etc.)
    for (const [eventName, definitions] of Object.entries(hooksConfig)) {
      for (const definition of definitions) {
        // For each hook in the definition
        for (const hookConfig of definition.hooks) {
          if (this.validateHookConfig(hookConfig, eventName, source)) {
            // Store in registry
            this.entries.push({
              config: hookConfig,
              source,
              eventName: eventName as HookEventName,
              matcher: definition.matcher,
              sequential: definition.sequential,
              enabled: true,
            });
          }
        }
      }
    }
  }
}
```

**What happens**:

1. Reads hooks from `Config.getHooks()`
2. Processes each hook definition
3. Validates hook configurations
4. Creates `HookRegistryEntry` for each valid hook
5. Stores in `entries` array with metadata

**Example entry created**:

```typescript
{
  config: {
    type: 'command',
    command: './hooks/check_edit.sh',
    timeout: 60000
  },
  source: ConfigSource.Project,
  eventName: HookEventName.BeforeTool,
  matcher: 'EditTool',
  sequential: false,
  enabled: true
}
```

---

### Step 3: Tool Execution Begins

**File**: `packages/core/src/core/nonInteractiveToolExecutor.ts`

```typescript
export async function executeToolCall(
  config: Config,
  toolCallRequest: ToolCallRequestInfo,
  abortSignal: AbortSignal,
): Promise<CompletedToolCall> {
  return new Promise<CompletedToolCall>((resolve, reject) => {
    // Creates CoreToolScheduler to execute the tool
    const scheduler = new CoreToolScheduler({
      config,
      getPreferredEditor: () => undefined,
      onEditorClose: () => {},
      onAllToolCallsComplete: async (completedToolCalls) => {
        resolve(completedToolCalls[0]);
      },
    });

    scheduler.schedule(toolCallRequest, abortSignal).catch(reject);
  });
}
```

**File**: `packages/core/src/core/coreToolScheduler.ts`

```typescript
export class CoreToolScheduler {
  async schedule(
    request: ToolCallRequestInfo,
    abortSignal: AbortSignal,
  ): Promise<void> {
    // [INTENDED HOOK INTEGRATION POINT #1]
    // BEFORE tool validation/execution:
    // if (config.enableHooks) {
    //   const hookOutput = await executeBeforeToolHooks(request);
    //   if (hookOutput.isBlockingDecision()) {
    //     return handleBlockedTool(hookOutput);
    //   }
    // }

    // Current code validates and executes tool
    const toolCall = await this.validateAndPrepare(request);

    // Execute the tool
    const result = await this.executeTool(toolCall);

    // [INTENDED HOOK INTEGRATION POINT #2]
    // AFTER tool execution:
    // if (config.enableHooks) {
    //   const hookOutput = await executeAfterToolHooks(request, result);
    //   if (hookOutput.getAdditionalContext()) {
    //     result.context = hookOutput.getAdditionalContext();
    //   }
    // }

    return result;
  }
}
```

---

### Step 4: BeforeTool Hook Execution (INTENDED FLOW)

**Intended File**: `packages/core/src/hooks/hookExecutor.ts` (NOT FOUND)

```typescript
// [THIS IS THE INTENDED IMPLEMENTATION - NOT VISIBLE IN CODE]

async function executeBeforeToolHooks(
  config: Config,
  hookRegistry: HookRegistry,
  hookPlanner: HookPlanner,
  request: ToolCallRequestInfo,
): Promise<DefaultHookOutput> {
  // Step 4.1: Create execution plan
  const plan = hookPlanner.createExecutionPlan(HookEventName.BeforeTool, {
    toolName: request.tool_name,
  });

  if (!plan || plan.hookConfigs.length === 0) {
    return new DefaultHookOutput({}); // No hooks to run
  }

  // Step 4.2: Prepare input JSON
  const hookInput: BeforeToolInput = {
    session_id: config.getSessionId(),
    transcript_path: config.getTranscriptPath(),
    cwd: process.cwd(),
    hook_event_name: 'BeforeTool',
    timestamp: new Date().toISOString(),
    tool_name: request.tool_name,
    tool_input: request.arguments,
  };

  // Step 4.3: Execute hooks (parallel or sequential)
  const results = plan.sequential
    ? await executeSequential(plan.hookConfigs, hookInput)
    : await executeParallel(plan.hookConfigs, hookInput);

  // Step 4.4: Aggregate results
  return aggregateHookResults(results);
}

async function executeSingleHook(
  hookConfig: HookConfig,
  input: HookInput,
): Promise<HookExecutionResult> {
  const startTime = Date.now();

  try {
    // Step 4.5: Spawn shell process
    const process = spawn(hookConfig.command, [], {
      shell: true,
      stdio: ['pipe', 'pipe', 'pipe'],
    });

    // Step 4.6: Write input to stdin
    process.stdin.write(JSON.stringify(input));
    process.stdin.end();

    // Step 4.7: Read stdout/stderr
    const stdout = await readStream(process.stdout);
    const stderr = await readStream(process.stderr);

    // Step 4.8: Wait for exit
    const exitCode = await waitForExit(process, hookConfig.timeout);

    // Step 4.9: Parse output
    let output: HookOutput | undefined;
    if (stdout.trim()) {
      try {
        output = JSON.parse(stdout);
      } catch (parseError) {
        // Invalid JSON output
        output = undefined;
      }
    }

    return {
      hookConfig,
      eventName: HookEventName.BeforeTool,
      success: exitCode === 0,
      output,
      stdout,
      stderr,
      exitCode,
      duration: Date.now() - startTime,
    };
  } catch (error) {
    return {
      hookConfig,
      eventName: HookEventName.BeforeTool,
      success: false,
      duration: Date.now() - startTime,
      error: error as Error,
    };
  }
}
```

---

### Step 5: Hook Execution Plan Creation

**File**: `packages/core/src/hooks/hookPlanner.ts`

```typescript
export class HookPlanner {
  createExecutionPlan(
    eventName: HookEventName,
    context?: HookEventContext,
  ): HookExecutionPlan | null {
    // Step 5.1: Get hooks for this event from registry
    const hookEntries = this.hookRegistry.getHooksForEvent(eventName);
    // Returns: All hooks registered for BeforeTool

    if (hookEntries.length === 0) {
      return null;
    }

    // Step 5.2: Filter by matcher
    const matchingEntries = hookEntries.filter((entry) =>
      this.matchesContext(entry, context),
    );
    // If context.toolName = "EditTool", only hooks with matcher
    // matching "EditTool" are included

    if (matchingEntries.length === 0) {
      return null;
    }

    // Step 5.3: Deduplicate
    const deduplicatedEntries = this.deduplicateHooks(matchingEntries);

    // Step 5.4: Extract configs
    const hookConfigs = deduplicatedEntries.map((entry) => entry.config);

    // Step 5.5: Determine execution strategy
    const sequential = deduplicatedEntries.some(
      (entry) => entry.sequential === true,
    );

    // Step 5.6: Return plan
    return {
      eventName,
      hookConfigs, // Array of { type: 'command', command: '...' }
      sequential, // true or false
    };
  }

  private matchesContext(
    entry: HookRegistryEntry,
    context?: HookEventContext,
  ): boolean {
    if (!entry.matcher || !context) {
      return true; // No matcher = match all
    }

    // For tool events, match against tool name
    if (context.toolName) {
      return this.matchesToolName(entry.matcher, context.toolName);
    }

    return true;
  }

  private matchesToolName(matcher: string, toolName: string): boolean {
    try {
      // Try as regex
      const regex = new RegExp(matcher);
      return regex.test(toolName);
    } catch {
      // Fallback to exact match
      return matcher === toolName;
    }
  }
}
```

**Example Execution Plan Created**:

```typescript
{
  eventName: HookEventName.BeforeTool,
  hookConfigs: [
    {
      type: 'command',
      command: './hooks/check_edit.sh',
      timeout: 60000
    }
  ],
  sequential: false
}
```

---

### Step 6: Decision Application (INTENDED)

```typescript
// After hook execution, apply the decisions

function applyHookDecision(
  hookOutput: DefaultHookOutput,
  request: ToolCallRequestInfo,
): { proceed: boolean; modifiedRequest?: ToolCallRequestInfo } {
  // Check for blocking decision
  if (hookOutput.isBlockingDecision()) {
    console.error(
      'Hook blocked tool execution:',
      hookOutput.getEffectiveReason(),
    );
    return { proceed: false };
  }

  // Check if execution should stop
  if (hookOutput.shouldStopExecution()) {
    console.error('Hook stopped execution:', hookOutput.getEffectiveReason());
    return { proceed: false };
  }

  // No blocking, proceed with execution
  return {
    proceed: true,
    modifiedRequest: request, // Could be modified by hook
  };
}
```

---

### Step 7: AfterTool Hook Execution (INTENDED)

Similar to BeforeTool, but with tool response included:

```typescript
async function executeAfterToolHooks(
  config: Config,
  request: ToolCallRequestInfo,
  response: ToolCallResponseInfo,
): Promise<DefaultHookOutput> {
  const plan = hookPlanner.createExecutionPlan(HookEventName.AfterTool, {
    toolName: request.tool_name,
  });

  if (!plan) return new DefaultHookOutput({});

  // Prepare input with response
  const hookInput: AfterToolInput = {
    session_id: config.getSessionId(),
    transcript_path: config.getTranscriptPath(),
    cwd: process.cwd(),
    hook_event_name: 'AfterTool',
    timestamp: new Date().toISOString(),
    tool_name: request.tool_name,
    tool_input: request.arguments,
    tool_response: response.result, // <-- Tool result included
  };

  const results = await executeHooks(plan, hookInput);

  return aggregateHookResults(results);
}
```

---

## Complete Flow Diagram with Code References

```
┌─────────────────────────────────────────────────────────────────┐
│  1. CONFIGURATION LOADING                                       │
│  ───────────────────────────────────────────────────────────────│
│  File: packages/core/src/config/config.ts                       │
│                                                                 │
│  constructor(params)                                            │
│    this.enableHooks = params.enableHooks                        │
│    this.hooks = params.hooks  // from settings.json             │
│                                                                 │
│  getHooks() { return this.hooks; }                              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. HOOK REGISTRY INITIALIZATION                                │
│  ───────────────────────────────────────────────────────────────│
│  File: packages/core/src/hooks/hookRegistry.ts                  │
│                                                                 │
│  new HookRegistry(config)                                       │
│  await hookRegistry.initialize()                                │
│    → processHooksFromConfig()                                   │
│      → config.getHooks()                                        │
│      → processHooksConfiguration()                              │
│        → validateHookConfig()                                   │
│        → entries.push(HookRegistryEntry)                        │
│                                                                 │
│  Result: entries[] populated with validated hooks               │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. USER ACTION: Edit config.json                               │
│  ───────────────────────────────────────────────────────────────│
│  LLM decides to call EditTool                                   │
│  Creates FunctionCall for EditTool                              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. TOOL EXECUTION INITIATED                                    │
│  ───────────────────────────────────────────────────────────────│
│  File: packages/core/src/core/nonInteractiveToolExecutor.ts     │
│                                                                 │
│  executeToolCall(config, toolCallRequest, signal)               │
│    → new CoreToolScheduler(config)                              │
│    → scheduler.schedule(toolCallRequest, signal)                │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. [INTENDED] BEFORE TOOL HOOK TRIGGER                         │
│  ───────────────────────────────────────────────────────────────│
│  File: [NOT IMPLEMENTED] packages/core/src/hooks/hookExecutor.ts│
│                                                                 │
│  if (config.enableHooks) {                                      │
│    hookPlanner.createExecutionPlan(                             │
│      HookEventName.BeforeTool,                                  │
│      { toolName: 'EditTool' }                                   │
│    )                                                            │
│  }                                                              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. HOOK PLANNING                                               │
│  ───────────────────────────────────────────────────────────────│
│  File: packages/core/src/hooks/hookPlanner.ts                   │
│                                                                 │
│  createExecutionPlan(BeforeTool, {toolName: 'EditTool'})        │
│    → hookRegistry.getHooksForEvent(BeforeTool)                  │
│      → Returns hooks with eventName=BeforeTool, enabled=true    │
│    → matchesContext(entry, {toolName: 'EditTool'})              │
│      → matchesToolName('EditTool', 'EditTool')  // regex match  │
│      → Returns true if matcher matches                          │
│    → deduplicateHooks(matchingEntries)                          │
│    → Return HookExecutionPlan {                                 │
│        eventName: BeforeTool,                                   │
│        hookConfigs: [{type:'command', command:'...'}],          │
│        sequential: false                                        │
│      }                                                          │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  7. [INTENDED] HOOK EXECUTION                                   │
│  ───────────────────────────────────────────────────────────────│
│  File: [NOT IMPLEMENTED]                                        │
│                                                                 │
│  For each hookConfig in plan.hookConfigs:                       │
│    1. Prepare JSON input:                                       │
│       {                                                         │
│         session_id, timestamp, cwd,                             │
│         hook_event_name: "BeforeTool",                          │
│         tool_name: "EditTool",                                  │
│         tool_input: {...}                                       │
│       }                                                         │
│                                                                 │
│    2. Spawn process:                                            │
│       spawn('./hooks/check_edit.sh', {stdio: ['pipe'...]})     │
│                                                                 │
│    3. Write to stdin:                                           │
│       process.stdin.write(JSON.stringify(input))                │
│                                                                 │
│    4. Read stdout:                                              │
│       const output = await readStream(process.stdout)           │
│                                                                 │
│    5. Parse JSON:                                               │
│       const hookOutput = JSON.parse(output)                     │
│       // { decision: "block", reason: "..." } OR {}             │
│                                                                 │
│    6. Create HookExecutionResult                                │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  8. [INTENDED] DECISION EVALUATION                              │
│  ───────────────────────────────────────────────────────────────│
│  File: packages/core/src/hooks/types.ts (HookOutput classes)    │
│                                                                 │
│  const aggregated = new DefaultHookOutput(results)              │
│                                                                 │
│  if (aggregated.isBlockingDecision()) {                         │
│    // decision === 'block' or 'deny'                            │
│    throw new Error(aggregated.getEffectiveReason())             │
│    → Tool execution STOPS                                       │
│  }                                                              │
│                                                                 │
│  if (aggregated.shouldStopExecution()) {                        │
│    // continue === false                                        │
│    throw new Error(aggregated.getEffectiveReason())             │
│    → Tool execution STOPS                                       │
│  }                                                              │
│                                                                 │
│  // No blocking → Continue to tool execution                    │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  9. TOOL EXECUTION                                              │
│  ───────────────────────────────────────────────────────────────│
│  File: packages/core/src/core/coreToolScheduler.ts              │
│                                                                 │
│  // Hook did not block, proceed with tool                       │
│  const tool = toolRegistry.getTool('EditTool')                  │
│  const result = await tool.run(arguments)                       │
│  // EditTool executes and modifies config.json                  │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  10. [INTENDED] AFTER TOOL HOOK TRIGGER                         │
│  ────────────────────────────────────────────────────────────── │
│  if (config.enableHooks) {                                      │
│    const afterHookOutput = await executeAfterToolHooks(         │
│      request,                                                   │
│      result  // Tool execution result                           │
│    )                                                            │
│                                                                 │
│    // Check for additional context                              │
│    const context = afterHookOutput.getAdditionalContext()       │
│    if (context) {                                               │
│      // Append context to tool result                           │
│      result.additionalContext = context                         │
│    }                                                            │
│  }                                                              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  11. RETURN RESULT                                              │
│  ────────────────────────────────────────────────────────────── │
│  Return tool result (with optional hook context) to LLM         │
│  LLM continues conversation with tool result                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Example: Concrete Code Walkthrough

### Configuration

**File**: `.gemini/settings.json`

```json
{
  "tools": {
    "enableHooks": true
  },
  "hooks": {
    "BeforeTool": [
      {
        "matcher": "EditTool",
        "sequential": false,
        "hooks": [
          {
            "type": "command",
            "command": "./hooks/check_edit.sh",
            "timeout": 5000
          }
        ]
      }
    ]
  }
}
```

### Hook Script

**File**: `.gemini/hooks/check_edit.sh`

```bash
#!/bin/bash
input=$(cat)  # Read JSON from stdin

# Extract file path from tool input
file_path=$(echo "$input" | jq -r '.tool_input.file_path')

# Check if editing protected file
if [ "$file_path" = "/etc/hosts" ]; then
    # Block the edit
    echo '{
        "decision": "block",
        "reason": "Editing /etc/hosts is not allowed",
        "systemMessage": "⚠️ Hook blocked editing of protected file"
    }'
else
    # Allow the edit
    echo '{}'
fi
```

### Execution Trace

```
1. User: "Edit /etc/hosts to add google.com"

2. LLM decides: Call EditTool with file_path="/etc/hosts"

3. CoreToolScheduler.schedule() called

4. [INTENDED] HookPlanner.createExecutionPlan(BeforeTool, {toolName: 'EditTool'})
   → Returns plan with ./hooks/check_edit.sh

5. [INTENDED] Hook execution:
   stdin  → {
              "session_id": "abc123",
              "tool_name": "EditTool",
              "tool_input": {
                "file_path": "/etc/hosts",
                "content": "127.0.0.1 google.com"
              },
              ...
            }

   stdout ← {
              "decision": "block",
              "reason": "Editing /etc/hosts is not allowed"
            }

6. [INTENDED] Decision evaluation:
   hookOutput.isBlockingDecision() === true
   → Throw error: "Editing /etc/hosts is not allowed"

7. Tool execution CANCELLED

8. Error returned to LLM:
   "Tool call blocked by hook: Editing /etc/hosts is not allowed"

9. LLM responds to user:
   "I cannot edit /etc/hosts as it's a protected system file"
```

---

## Summary

### What's Implemented

✅ Type system and interfaces ✅ Configuration storage ✅ Hook registry
(loading, validation, filtering) ✅ Hook planner (matching, deduplication,
execution planning) ✅ Hook translator (format conversion) ✅ Hook output
classes (decision evaluation)

### What's Missing in Visible Code

❌ Hook executor (shell process spawning, I/O handling) ❌ Integration points in
tool execution flow ❌ Integration points in LLM request/response flow ❌ Result
aggregation logic ❌ Error handling and retry logic

### How It Should Work

1. Configuration defines hooks for events
2. HookRegistry loads and validates at startup
3. Before each tool/LLM call, HookPlanner creates execution plan
4. Hook executor spawns processes, handles I/O
5. Results are aggregated and decisions applied
6. Tool/LLM proceeds or is blocked based on hook output

The infrastructure is solid and well-designed. The missing pieces are the
execution engine and integration points in the actual tool/LLM flow.
