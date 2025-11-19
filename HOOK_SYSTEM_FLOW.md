# Gemini CLI - Hook System Flow & Architecture

## Table of Contents
1. [Overview](#overview)
2. [Hook System Status](#hook-system-status)
3. [Hook Architecture](#hook-architecture)
4. [Hook Registration Flow](#hook-registration-flow)
5. [Hook Execution Flow (Designed)](#hook-execution-flow-designed)
6. [Hook Types & Event Points](#hook-types--event-points)
7. [Complete Example: BeforeTool Hook](#complete-example-beforetool-hook)
8. [Code References](#code-references)

---

## Overview

The **Hook System** in Gemini CLI is designed to intercept and customize agent behavior at specific lifecycle events. Hooks allow users to:

- **Validate** tool execution before it happens (BeforeTool)
- **Process** tool outputs after execution (AfterTool)
- **Modify** prompts before sending to the model (BeforeAgent)
- **Intercept** LLM requests and responses (BeforeModel, AfterModel)
- **Control** tool selection (BeforeToolSelection)
- **React** to session events (SessionStart, SessionEnd)

Hooks are implemented as **shell commands** that receive JSON input via stdin and return JSON output via stdout.

---

## Hook System Status

### ✅ **IMPLEMENTED Components**

| Component | File | Status | Description |
|-----------|------|--------|-------------|
| **Hook Types** | `packages/core/src/hooks/types.ts` | ✅ Complete | All event types and I/O schemas defined |
| **Hook Registry** | `packages/core/src/hooks/hookRegistry.ts` | ✅ Complete | Hook loading, validation, and storage |
| **Hook Planner** | `packages/core/src/hooks/hookPlanner.ts` | ✅ Complete | Execution plan creation and matching |
| **Hook Translator** | `packages/core/src/hooks/hookTranslator.ts` | ✅ Complete | Convert between LLM and hook formats |
| **Configuration** | `packages/cli/src/config/settingsSchema.ts` | ✅ Complete | Settings schema and validation |

### ❌ **NOT YET IMPLEMENTED**

| Component | Reason | Impact |
|-----------|--------|--------|
| **Hook Executor** | Execution engine not built | Hooks don't actually run |
| **Integration Points** | No calls to hook planner | Events don't trigger hooks |
| **Output Processing** | No result handling | Hook decisions ignored |
| **Timeout Handling** | No process management | Could hang indefinitely |

### 🔄 **Current Behavior**

Even though hooks can be configured in `settings.json`, they **DO NOT execute** because:
1. The hook executor (command runner) is not implemented
2. No integration points exist in tool/agent execution flow
3. Feature flag `tools.enableHooks` exists but has no effect

---

## Hook Architecture

### Component Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     Settings.json                            │
│  {                                                           │
│    "hooks": {                                                │
│      "BeforeTool": [{ matcher: "EditTool", hooks: [...] }]  │
│    },                                                        │
│    "tools": { "enableHooks": true }                          │
│  }                                                           │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ├─> Config.initialize()
                         │
┌────────────────────────▼────────────────────────────────────┐
│                   HookRegistry                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ initialize()                                        │    │
│  │  ├─> processHooksFromConfig()                       │    │
│  │  │   ├─> config.getHooks() (project hooks)          │    │
│  │  │   └─> config.getExtensions() (extension hooks)   │    │
│  │  └─> validateHookConfig()                           │    │
│  │                                                      │    │
│  │ entries: HookRegistryEntry[]                        │    │
│  │  [                                                   │    │
│  │    {                                                 │    │
│  │      config: { type: 'command', command: '...' },   │    │
│  │      source: 'project',                             │    │
│  │      eventName: 'BeforeTool',                       │    │
│  │      matcher: 'EditTool',                           │    │
│  │      sequential: true,                              │    │
│  │      enabled: true                                  │    │
│  │    },                                                │    │
│  │    ...                                               │    │
│  │  ]                                                   │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  getHooksForEvent(eventName): HookRegistryEntry[]           │
│  getAllHooks(): HookRegistryEntry[]                         │
│  setHookEnabled(name, enabled): void                        │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ├─> Event Triggered (e.g., BeforeTool)
                         │
┌────────────────────────▼────────────────────────────────────┐
│                   HookPlanner                                │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ createExecutionPlan(eventName, context)             │    │
│  │  ├─> hookRegistry.getHooksForEvent(eventName)       │    │
│  │  ├─> Filter by matcher (tool name, trigger)         │    │
│  │  ├─> Deduplicate identical hooks                    │    │
│  │  └─> Determine sequential vs parallel               │    │
│  │                                                      │    │
│  │ Returns: HookExecutionPlan | null                   │    │
│  │  {                                                   │    │
│  │    eventName: 'BeforeTool',                         │    │
│  │    hookConfigs: [                                   │    │
│  │      { type: 'command', command: './hook.sh' }     │    │
│  │    ],                                                │    │
│  │    sequential: true                                 │    │
│  │  }                                                   │    │
│  └─────────────────────────────────────────────────────┘    │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ├─> [NOT IMPLEMENTED] Hook Executor
                         │
┌────────────────────────▼────────────────────────────────────┐
│             [PLANNED] Hook Executor                          │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ executeHooks(plan, input)                           │    │
│  │  ├─> For each hook in plan.hookConfigs:            │    │
│  │  │   ├─> Build hook input (JSON)                   │    │
│  │  │   ├─> spawn(command)                             │    │
│  │  │   ├─> Write input to stdin                       │    │
│  │  │   ├─> Read output from stdout                    │    │
│  │  │   ├─> Parse JSON output                          │    │
│  │  │   └─> Apply timeout                              │    │
│  │  ├─> Aggregate results                              │    │
│  │  └─> Return: HookExecutionResult                    │    │
│  └─────────────────────────────────────────────────────┘    │
└────────────────────────┬────────────────────────────────────┘
                         │
                         └─> Apply hook decision to flow
```

---

## Hook Registration Flow

### Step-by-Step Registration Process

#### Phase 1: Configuration Loading

**File**: `packages/cli/src/config/config.ts:622-623`

```typescript
// During CLI initialization
const cliConfig = {
  enableHooks: settings.tools?.enableHooks ?? false,
  hooks: settings.hooks || {},
};
```

**Settings.json Example**:
```json
{
  "tools": {
    "enableHooks": true
  },
  "hooks": {
    "BeforeTool": [
      {
        "matcher": "EditTool|WriteFileTool",
        "sequential": true,
        "hooks": [
          {
            "type": "command",
            "command": "./hooks/check_syntax.sh",
            "timeout": 30
          }
        ]
      }
    ],
    "AfterTool": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "./hooks/log_tool.sh"
          }
        ]
      }
    ]
  }
}
```

#### Phase 2: Core Config Initialization

**File**: `packages/core/src/config/config.ts:1382-1384`

```typescript
export class Config {
  private hooks?: { [K in HookEventName]?: HookDefinition[] };

  getHooks(): { [K in HookEventName]?: HookDefinition[] } | undefined {
    return this.hooks;
  }
}
```

#### Phase 3: Hook Registry Initialization

**File**: `packages/core/src/hooks/hookRegistry.ts:59-71`

```typescript
export class HookRegistry {
  async initialize(): Promise<void> {
    if (this.initialized) return;

    this.entries = [];

    // 1. Process hooks from main config (project hooks)
    this.processHooksFromConfig();

    this.initialized = true;

    debugLogger.log(
      `Hook registry initialized with ${this.entries.length} hook entries`
    );
  }
}
```

#### Phase 4: Hook Processing

**File**: `packages/core/src/hooks/hookRegistry.ts:132-149`

```typescript
private processHooksFromConfig(): void {
  // ===== PROJECT HOOKS =====
  const configHooks = this.config.getHooks();
  if (configHooks) {
    this.processHooksConfiguration(
      configHooks,
      ConfigSource.Project  // Priority: 1 (highest)
    );
  }

  // ===== EXTENSION HOOKS =====
  const extensions = this.config.getExtensions() || [];
  for (const extension of extensions) {
    if (extension.isActive && extension.hooks) {
      this.processHooksConfiguration(
        extension.hooks,
        ConfigSource.Extensions  // Priority: 4 (lowest)
      );
    }
  }
}
```

#### Phase 5: Hook Validation

**File**: `packages/core/src/hooks/hookRegistry.ts:182-220`

```typescript
private processHookDefinition(
  definition: HookDefinition,
  eventName: HookEventName,
  source: ConfigSource
): void {
  // Validate structure
  if (!definition || !Array.isArray(definition.hooks)) {
    debugLogger.warn(`Invalid hook definition for ${eventName}`);
    return;
  }

  // Process each hook config
  for (const hookConfig of definition.hooks) {
    if (this.validateHookConfig(hookConfig, eventName, source)) {
      // ===== ADD TO REGISTRY =====
      this.entries.push({
        config: hookConfig,
        source,
        eventName,
        matcher: definition.matcher,
        sequential: definition.sequential,
        enabled: true,
      });
    } else {
      debugLogger.warn(`Invalid hook config`, hookConfig);
    }
  }
}
```

#### Phase 6: Validation Rules

**File**: `packages/core/src/hooks/hookRegistry.ts:226-246`

```typescript
private validateHookConfig(
  config: HookConfig,
  eventName: HookEventName,
  source: ConfigSource
): boolean {
  // Check type is valid
  if (!config.type || !['command', 'plugin'].includes(config.type)) {
    return false;
  }

  // For command type, require command field
  if (config.type === 'command' && !config.command) {
    return false;
  }

  return true;
}
```

### Registry Data Structure

After initialization, the registry contains:

```typescript
HookRegistry.entries = [
  {
    config: {
      type: 'command',
      command: './hooks/check_syntax.sh',
      timeout: 30
    },
    source: 'project',        // ConfigSource.Project
    eventName: 'BeforeTool',  // HookEventName.BeforeTool
    matcher: 'EditTool|WriteFileTool',  // Regex pattern
    sequential: true,
    enabled: true
  },
  {
    config: {
      type: 'command',
      command: './hooks/log_tool.sh',
      timeout: undefined  // Default timeout
    },
    source: 'project',
    eventName: 'AfterTool',
    matcher: '*',  // Match all tools
    sequential: false,
    enabled: true
  }
]
```

---

## Hook Execution Flow (Designed)

### Overview: When Would Hooks Execute?

```
Tool Execution Request
    │
    ├─> [HOOK] BeforeTool
    │   ├─> Hook decides: ALLOW | DENY | ASK_USER
    │   └─> If DENY: abort tool execution
    │
    ├─> Tool Executes
    │   └─> Returns result
    │
    ├─> [HOOK] AfterTool
    │   ├─> Hook processes result
    │   └─> Hook can add context/annotations
    │
    └─> Result sent to Model
```

### Detailed Flow (Designed, Not Implemented)

#### Scenario: User wants to edit a file

**User Message**: "Fix the typo in config.ts"

```
┌─────────────────────────────────────────────────────────────┐
│ TURN 1: Model decides to call EditTool                      │
└─────────────────────────────────────────────────────────────┘

1. Agent Executor receives function call:
   functionCall = {
     name: 'EditTool',
     args: {
       file_path: 'config.ts',
       old_string: 'conifg',
       new_string: 'config'
     }
   }

2. Get tool from registry:
   tool = toolRegistry.getTool('EditTool')

3. Build invocation:
   invocation = tool.build(args)

┌─────────────────────────────────────────────────────────────┐
│ [DESIGNED] HOOK INTERCEPTION POINT - BeforeTool             │
└─────────────────────────────────────────────────────────────┘

4. [NOT IMPLEMENTED] Check for BeforeTool hooks:

   hookPlanner.createExecutionPlan(
     HookEventName.BeforeTool,
     { toolName: 'EditTool' }
   )

   Returns:
   {
     eventName: 'BeforeTool',
     hookConfigs: [
       {
         type: 'command',
         command: './hooks/check_syntax.sh',
         timeout: 30
       }
     ],
     sequential: true
   }

5. [NOT IMPLEMENTED] Execute hook:

   a) Build hook input:
      {
        "session_id": "abc123",
        "transcript_path": "/tmp/session-abc123.json",
        "cwd": "/home/user/project",
        "hook_event_name": "BeforeTool",
        "timestamp": "2025-01-15T10:30:00Z",
        "tool_name": "EditTool",
        "tool_input": {
          "file_path": "config.ts",
          "old_string": "conifg",
          "new_string": "config"
        }
      }

   b) Spawn process:
      const proc = spawn('./hooks/check_syntax.sh')

   c) Write input to stdin:
      proc.stdin.write(JSON.stringify(hookInput))
      proc.stdin.end()

   d) Read output from stdout:
      let output = '';
      proc.stdout.on('data', chunk => output += chunk)

   e) Wait for completion (with timeout):
      await waitForExit(proc, 30000)  // 30 second timeout

   f) Parse output:
      hookOutput = JSON.parse(output)
      {
        "decision": "allow",
        "reason": "Syntax check passed",
        "systemMessage": "✓ Syntax validated"
      }

6. [NOT IMPLEMENTED] Apply hook decision:

   if (hookOutput.decision === 'deny' || hookOutput.decision === 'block') {
     // Abort tool execution
     return {
       llmContent: '',
       error: {
         message: `Hook blocked execution: ${hookOutput.reason}`
       }
     };
   }

   if (hookOutput.decision === 'ask') {
     // Prompt user for confirmation
     const approved = await askUser(`Allow EditTool? ${hookOutput.reason}`);
     if (!approved) {
       return { error: { message: 'User denied' } };
     }
   }

   // decision === 'allow' or 'approve' -> continue

┌─────────────────────────────────────────────────────────────┐
│ TOOL EXECUTION                                               │
└─────────────────────────────────────────────────────────────┘

7. Execute tool:
   result = await invocation.execute(signal)

   Result:
   {
     llmContent: "Replaced 1 occurrence in config.ts",
     returnDisplay: "✓ Fixed typo in config.ts"
   }

┌─────────────────────────────────────────────────────────────┐
│ [DESIGNED] HOOK INTERCEPTION POINT - AfterTool              │
└─────────────────────────────────────────────────────────────┘

8. [NOT IMPLEMENTED] AfterTool hook:

   hookPlanner.createExecutionPlan(
     HookEventName.AfterTool,
     { toolName: 'EditTool' }
   )

   Hook input:
   {
     "session_id": "abc123",
     "hook_event_name": "AfterTool",
     "tool_name": "EditTool",
     "tool_input": { "file_path": "config.ts", ... },
     "tool_response": {
       "llmContent": "Replaced 1 occurrence in config.ts",
       "returnDisplay": "✓ Fixed typo in config.ts"
     }
   }

   Hook executes: ./hooks/log_tool.sh

   Hook output:
   {
     "continue": true,
     "systemMessage": "Edit logged to audit trail",
     "hookSpecificOutput": {
       "audit_id": "edit-12345"
     }
   }

9. [NOT IMPLEMENTED] Augment result with hook output:

   finalResult = {
     ...result,
     metadata: {
       hookMessages: ["Edit logged to audit trail"],
       hookData: { audit_id: "edit-12345" }
     }
   }

┌─────────────────────────────────────────────────────────────┐
│ CONTINUE AGENT FLOW                                          │
└─────────────────────────────────────────────────────────────┘

10. Send result to model:

    functionResponse = {
      name: 'EditTool',
      response: finalResult.llmContent
    }

11. Model generates final response:
    "I've fixed the typo in config.ts, changing 'conifg' to 'config'."
```

### Hook Execution Pseudocode

```typescript
// [NOT IMPLEMENTED] - Designed flow

async function executeToolWithHooks(
  toolName: string,
  toolArgs: unknown,
  signal: AbortSignal
): Promise<ToolResult> {

  // ===== BEFORE TOOL HOOK =====
  const beforePlan = hookPlanner.createExecutionPlan(
    HookEventName.BeforeTool,
    { toolName }
  );

  if (beforePlan) {
    const beforeResult = await hookExecutor.execute(beforePlan, {
      tool_name: toolName,
      tool_input: toolArgs,
    }, signal);

    // Check decision
    if (beforeResult.isBlocking()) {
      return {
        llmContent: '',
        error: { message: beforeResult.reason }
      };
    }

    // Check for user confirmation request
    if (beforeResult.decision === 'ask') {
      const approved = await promptUser(beforeResult.reason);
      if (!approved) {
        return { error: { message: 'User denied' } };
      }
    }

    // Log system message
    if (beforeResult.systemMessage) {
      console.log(beforeResult.systemMessage);
    }
  }

  // ===== EXECUTE TOOL =====
  const tool = toolRegistry.getTool(toolName);
  const invocation = tool.build(toolArgs);
  const result = await invocation.execute(signal);

  // ===== AFTER TOOL HOOK =====
  const afterPlan = hookPlanner.createExecutionPlan(
    HookEventName.AfterTool,
    { toolName }
  );

  if (afterPlan) {
    const afterResult = await hookExecutor.execute(afterPlan, {
      tool_name: toolName,
      tool_input: toolArgs,
      tool_response: result,
    }, signal);

    // Augment result with hook data
    if (afterResult.systemMessage) {
      result.metadata = result.metadata || {};
      result.metadata.hookMessages = [afterResult.systemMessage];
    }

    if (afterResult.hookSpecificOutput) {
      result.metadata = result.metadata || {};
      result.metadata.hookData = afterResult.hookSpecificOutput;
    }

    // Check if hook wants to stop execution
    if (afterResult.shouldStop()) {
      throw new Error(`Hook stopped execution: ${afterResult.stopReason}`);
    }
  }

  return result;
}
```

---

## Hook Types & Event Points

### Complete Hook Event Catalog

**File**: `packages/core/src/hooks/types.ts:23-35`

```typescript
export enum HookEventName {
  BeforeTool = 'BeforeTool',
  AfterTool = 'AfterTool',
  BeforeAgent = 'BeforeAgent',
  Notification = 'Notification',
  AfterAgent = 'AfterAgent',
  SessionStart = 'SessionStart',
  SessionEnd = 'SessionEnd',
  PreCompress = 'PreCompress',
  BeforeModel = 'BeforeModel',
  AfterModel = 'AfterModel',
  BeforeToolSelection = 'BeforeToolSelection',
}
```

### Hook Event Details

#### 1. BeforeTool

**When**: Before any tool executes
**Purpose**: Validate, approve, or deny tool execution
**Use Cases**:
- Syntax checking before file edits
- Security validation before shell commands
- Policy enforcement
- Logging tool invocations

**Input** (packages/core/src/hooks/types.ts:374-388):
```typescript
{
  session_id: string;
  transcript_path: string;
  cwd: string;
  hook_event_name: "BeforeTool";
  timestamp: string;
  tool_name: string;        // e.g., "EditTool"
  tool_input: unknown;      // Tool parameters
}
```

**Output**:
```typescript
{
  decision?: 'allow' | 'deny' | 'block' | 'ask';
  reason?: string;
  systemMessage?: string;
}
```

**Example Hook**:
```bash
#!/bin/bash
# hooks/validate_edit.sh

INPUT=$(cat)
TOOL_NAME=$(echo "$INPUT" | jq -r '.tool_name')
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path')

# Only validate EditTool
if [ "$TOOL_NAME" != "EditTool" ]; then
  echo '{"decision":"allow"}'
  exit 0
fi

# Check if file is in protected directory
if [[ "$FILE_PATH" == /etc/* ]]; then
  echo '{
    "decision": "deny",
    "reason": "Cannot edit system files in /etc"
  }'
  exit 0
fi

# Run syntax check
if ! check_syntax "$FILE_PATH"; then
  echo '{
    "decision": "ask",
    "reason": "Syntax check failed. Allow anyway?"
  }'
  exit 0
fi

echo '{
  "decision": "allow",
  "systemMessage": "✓ Validation passed"
}'
```

#### 2. AfterTool

**When**: After tool completes execution
**Purpose**: Process results, add annotations, log events
**Use Cases**:
- Audit logging
- Metrics collection
- Result validation
- Notifications

**Input** (packages/core/src/hooks/types.ts:391-407):
```typescript
{
  session_id: string;
  hook_event_name: "AfterTool";
  tool_name: string;
  tool_input: unknown;
  tool_response: {
    llmContent: string | object;
    returnDisplay?: string;
    error?: unknown;
  };
}
```

**Output**:
```typescript
{
  continue?: boolean;
  systemMessage?: string;
  hookSpecificOutput?: Record<string, unknown>;
}
```

#### 3. BeforeModel

**When**: Before sending request to Gemini API
**Purpose**: Intercept, modify, or replace LLM requests
**Use Cases**:
- Request caching
- Mock responses for testing
- Request filtering
- Cost tracking

**Input** (packages/core/src/hooks/types.ts:529-543):
```typescript
{
  session_id: string;
  hook_event_name: "BeforeModel";
  llm_request: {
    contents: Content[];
    tools?: ToolConfig[];
    generationConfig?: GenerationConfig;
  };
}
```

**Output**:
```typescript
{
  continue?: boolean;
  modifiedRequest?: LLMRequest;  // Replace request
  syntheticResponse?: LLMResponse;  // Skip API call entirely
}
```

**Example: Mock Response**:
```bash
#!/bin/bash
# hooks/mock_response.sh

INPUT=$(cat)

# Check if request is asking for help
if echo "$INPUT" | jq -e '.llm_request.contents[-1].parts[0].text | contains("help")'; then
  echo '{
    "continue": false,
    "syntheticResponse": {
      "text": "I can help you with that! What do you need?",
      "functionCalls": []
    }
  }'
  exit 0
fi

# Normal flow
echo '{"continue":true}'
```

#### 4. AfterModel

**When**: After receiving response from Gemini API
**Purpose**: Modify or validate model responses
**Use Cases**:
- Response filtering
- Content moderation
- Response augmentation
- Logging

**Input** (packages/core/src/hooks/types.ts:547-561):
```typescript
{
  session_id: string;
  hook_event_name: "AfterModel";
  llm_request: LLMRequest;
  llm_response: LLMResponse;
}
```

**Output**:
```typescript
{
  modifiedResponse?: LLMResponse;
}
```

#### 5. BeforeToolSelection

**When**: Before model selects which tools to call
**Purpose**: Control tool availability for specific requests
**Use Cases**:
- Tool filtering by context
- Security restrictions
- Tool whitelisting/blacklisting

**Input** (packages/core/src/hooks/types.ts:563-578):
```typescript
{
  session_id: string;
  hook_event_name: "BeforeToolSelection";
  llm_request: LLMRequest;
}
```

**Output**:
```typescript
{
  toolConfig?: HookToolConfig;  // Modified tool list
}
```

#### 6. SessionStart

**When**: At session initialization
**Purpose**: Setup, initialization, logging
**Use Cases**:
- Environment setup
- Session logging
- Resource initialization
- Loading context

**Input** (packages/core/src/hooks/types.ts:470-484):
```typescript
{
  session_id: string;
  hook_event_name: "SessionStart";
  trigger: 'Startup' | 'Resume' | 'Clear' | 'Compress';
}
```

#### 7. SessionEnd

**When**: At session termination
**Purpose**: Cleanup, finalization, logging
**Use Cases**:
- Resource cleanup
- Session summary
- Analytics upload
- Backup creation

**Input** (packages/core/src/hooks/types.ts:499-502):
```typescript
{
  session_id: string;
  hook_event_name: "SessionEnd";
  reason: 'Exit' | 'Clear' | 'Logout' | 'Error' | 'Timeout';
}
```

---

## Complete Example: BeforeTool Hook

### Scenario: Validate Edits Before Execution

#### 1. Configure Hook in settings.json

```json
{
  "tools": {
    "enableHooks": true
  },
  "hooks": {
    "BeforeTool": [
      {
        "matcher": "EditTool|WriteFileTool",
        "sequential": true,
        "hooks": [
          {
            "type": "command",
            "command": ".gemini/hooks/validate_edit.sh",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

#### 2. Create Hook Script

**File**: `.gemini/hooks/validate_edit.sh`

```bash
#!/bin/bash
set -euo pipefail

# Read hook input from stdin
INPUT=$(cat)

# Parse input
TOOL_NAME=$(echo "$INPUT" | jq -r '.tool_name')
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')
OLD_STRING=$(echo "$INPUT" | jq -r '.tool_input.old_string // empty')
NEW_STRING=$(echo "$INPUT" | jq -r '.tool_input.new_string // empty')

# ==================================================
# VALIDATION LOGIC
# ==================================================

# 1. Check if file is in protected directory
if [[ "$FILE_PATH" == /etc/* ]] || [[ "$FILE_PATH" == /usr/* ]]; then
  cat <<EOF
{
  "decision": "deny",
  "reason": "Cannot edit system files in protected directories"
}
EOF
  exit 0
fi

# 2. Check if file contains secrets
if grep -q "password\|secret\|api_key" "$FILE_PATH" 2>/dev/null; then
  cat <<EOF
{
  "decision": "ask",
  "reason": "File may contain secrets. Proceed with edit?"
}
EOF
  exit 0
fi

# 3. Validate replacement doesn't introduce common bugs
if [[ "$NEW_STRING" == *"var "* ]]; then
  cat <<EOF
{
  "decision": "ask",
  "reason": "Using 'var' instead of 'const/let'. Continue anyway?",
  "systemMessage": "⚠️  Consider using const/let instead of var"
}
EOF
  exit 0
fi

# 4. All checks passed
cat <<EOF
{
  "decision": "allow",
  "systemMessage": "✓ Edit validation passed"
}
EOF
```

Make executable:
```bash
chmod +x .gemini/hooks/validate_edit.sh
```

#### 3. Registration Flow

```
App Startup
  │
  ├─> Load settings.json
  │   └─> hooks.BeforeTool = [{ matcher: "EditTool|WriteFileTool", ... }]
  │
  ├─> Config.initialize()
  │   └─> config.hooks = parsed hook definitions
  │
  ├─> HookRegistry.initialize()
  │   ├─> processHooksFromConfig()
  │   ├─> Validate each hook config
  │   └─> Store in registry.entries[]
  │
  └─> Registry contains:
      [
        {
          config: {
            type: 'command',
            command: '.gemini/hooks/validate_edit.sh',
            timeout: 30
          },
          source: 'project',
          eventName: 'BeforeTool',
          matcher: 'EditTool|WriteFileTool',
          sequential: true,
          enabled: true
        }
      ]
```

#### 4. Execution Flow (Designed)

```
User: "Change var to const in utils.js"
  │
  ├─> Model calls: EditTool({
      file_path: "utils.js",
      old_string: "var x = 1;",
      new_string: "const x = 1;"
    })
  │
  ├─> [HOOK] BeforeTool triggered
  │   │
  │   ├─> HookPlanner.createExecutionPlan(
  │   │     'BeforeTool',
  │   │     { toolName: 'EditTool' }
  │   │   )
  │   │   └─> Matches: 'EditTool' =~ /EditTool|WriteFileTool/
  │   │   └─> Returns plan with validate_edit.sh
  │   │
  │   ├─> [NOT IMPL] HookExecutor.execute(plan, input)
  │   │   │
  │   │   ├─> Spawn: .gemini/hooks/validate_edit.sh
  │   │   │
  │   │   ├─> Write to stdin:
  │   │   │   {
  │   │   │     "session_id": "abc123",
  │   │   │     "hook_event_name": "BeforeTool",
  │   │   │     "tool_name": "EditTool",
  │   │   │     "tool_input": {
  │   │   │       "file_path": "utils.js",
  │   │   │       "old_string": "var x = 1;",
  │   │   │       "new_string": "const x = 1;"
  │   │   │     }
  │   │   │   }
  │   │   │
  │   │   ├─> Hook executes validation logic
  │   │   │   ├─> Not in /etc or /usr: ✓
  │   │   │   ├─> No secrets detected: ✓
  │   │   │   ├─> NEW_STRING check: changing TO const (good): ✓
  │   │   │   └─> All checks pass
  │   │   │
  │   │   ├─> Read from stdout:
  │   │   │   {
  │   │   │     "decision": "allow",
  │   │   │     "systemMessage": "✓ Edit validation passed"
  │   │   │   }
  │   │   │
  │   │   └─> Parse and return HookOutput
  │   │
  │   ├─> Check decision: "allow" → Continue
  │   │
  │   └─> Display: "✓ Edit validation passed"
  │
  ├─> Execute EditTool
  │   └─> Result: File edited successfully
  │
  └─> Model responds: "Changed var to const in utils.js"
```

#### 5. Example: Hook Denies Edit

```
User: "Edit /etc/passwd to add a user"
  │
  ├─> Model calls: EditTool({
      file_path: "/etc/passwd",
      old_string: "...",
      new_string: "..."
    })
  │
  ├─> [HOOK] BeforeTool triggered
  │   │
  │   ├─> Hook executes
  │   │   └─> Detects: file_path starts with /etc/
  │   │   └─> Returns:
  │   │       {
  │   │         "decision": "deny",
  │   │         "reason": "Cannot edit system files in protected directories"
  │   │       }
  │   │
  │   ├─> Check decision: "deny" → ABORT
  │   │
  │   └─> Return error to model:
  │       {
  │         error: {
  │           message: "Hook blocked execution: Cannot edit system files in protected directories"
  │         }
  │       }
  │
  ├─> Model receives error
  │
  └─> Model responds: "I cannot edit /etc/passwd as it's a protected system file."
```

---

## Code References

### Core Hook System Files

| File | Lines | Description |
|------|-------|-------------|
| **packages/core/src/hooks/types.ts** | 1-603 | All hook type definitions, event names, I/O schemas |
| **packages/core/src/hooks/hookRegistry.ts** | 1-273 | Hook registration, loading, validation, storage |
| **packages/core/src/hooks/hookPlanner.ts** | 1-154 | Execution plan creation, hook matching, deduplication |
| **packages/core/src/hooks/hookTranslator.ts** | 1-359 | Convert between LLM and hook data formats |

### Configuration Files

| File | Lines | Description |
|------|-------|-------------|
| **packages/cli/src/config/config.ts** | 622-623 | CLI hook configuration loading |
| **packages/core/src/config/config.ts** | 1382-1384 | Core hook access methods |
| **packages/cli/src/config/settingsSchema.ts** | 17-19 | Hook settings schema |
| **schemas/settings.schema.json** | - | JSON schema for validation |
| **docs/get-started/configuration.md** | 449-455 | Hook configuration documentation |

### Key Methods

#### HookRegistry

```typescript
// packages/core/src/hooks/hookRegistry.ts

class HookRegistry {
  // Line 59: Initialize registry
  async initialize(): Promise<void>

  // Line 76: Get hooks for specific event
  getHooksForEvent(eventName: HookEventName): HookRegistryEntry[]

  // Line 92: Get all registered hooks
  getAllHooks(): HookRegistryEntry[]

  // Line 103: Enable/disable hook
  setHookEnabled(hookName: string, enabled: boolean): void

  // Line 132: Process config hooks
  private processHooksFromConfig(): void

  // Line 154: Process hook configuration
  private processHooksConfiguration(
    hooksConfig: { [K in HookEventName]?: HookDefinition[] },
    source: ConfigSource
  ): void

  // Line 182: Process single hook definition
  private processHookDefinition(
    definition: HookDefinition,
    eventName: HookEventName,
    source: ConfigSource
  ): void

  // Line 226: Validate hook config
  private validateHookConfig(
    config: HookConfig,
    eventName: HookEventName,
    source: ConfigSource
  ): boolean
}
```

#### HookPlanner

```typescript
// packages/core/src/hooks/hookPlanner.ts

class HookPlanner {
  // Line 25: Create execution plan
  createExecutionPlan(
    eventName: HookEventName,
    context?: HookEventContext
  ): HookExecutionPlan | null

  // Line 71: Match hook to context
  private matchesContext(
    entry: HookRegistryEntry,
    context?: HookEventContext
  ): boolean

  // Line 100: Match tool name with regex
  private matchesToolName(matcher: string, toolName: string): boolean

  // Line 122: Deduplicate identical hooks
  private deduplicateHooks(entries: HookRegistryEntry[]): HookRegistryEntry[]
}
```

### Hook Event Context

```typescript
// packages/core/src/hooks/hookPlanner.ts:151-154

export interface HookEventContext {
  toolName?: string;   // For tool-related events
  trigger?: string;    // For session events (Startup, Resume, etc.)
}
```

### Example Integration Point (Where it WOULD be called)

```typescript
// [NOT IMPLEMENTED] - Example of where hooks would be called
// packages/core/src/core/coreToolScheduler.ts

async function executeToolInvocation(
  invocation: ToolInvocation,
  signal: AbortSignal
): Promise<ToolResult> {

  // ===== BEFORE TOOL HOOK =====
  const beforePlan = this.hookPlanner.createExecutionPlan(
    HookEventName.BeforeTool,
    { toolName: invocation.tool.name }
  );

  if (beforePlan) {
    const hookResult = await this.hookExecutor.execute(
      beforePlan,
      {
        tool_name: invocation.tool.name,
        tool_input: invocation.params,
      },
      signal
    );

    if (hookResult.isBlockingDecision()) {
      throw new Error(`Hook denied: ${hookResult.reason}`);
    }
  }

  // ===== EXECUTE TOOL =====
  const result = await invocation.execute(signal);

  // ===== AFTER TOOL HOOK =====
  const afterPlan = this.hookPlanner.createExecutionPlan(
    HookEventName.AfterTool,
    { toolName: invocation.tool.name }
  );

  if (afterPlan) {
    const hookResult = await this.hookExecutor.execute(
      afterPlan,
      {
        tool_name: invocation.tool.name,
        tool_input: invocation.params,
        tool_response: result,
      },
      signal
    );

    // Augment result with hook data
    if (hookResult.systemMessage) {
      console.log(hookResult.systemMessage);
    }
  }

  return result;
}
```

---

## Summary

### What EXISTS

✅ **Complete Infrastructure**:
- Type definitions for all 11 hook events
- Hook registry for loading and storing hooks
- Hook planner for creating execution plans
- Hook translator for data format conversion
- Configuration schema and validation
- Feature flag (`tools.enableHooks`)

### What's MISSING

❌ **Execution Components**:
- Hook executor (shell command runner)
- Process management and timeouts
- stdin/stdout I/O handling
- Output parsing and validation
- Integration points in agent/tool flow
- Error handling and recovery

### Current State

**Hooks are fully designed but NOT operational**. You can:
- ✅ Configure hooks in `settings.json`
- ✅ Hooks load and validate during startup
- ✅ Execution plans generate correctly
- ❌ **Hooks DO NOT execute**
- ❌ **Hooks DO NOT affect tool execution**
- ❌ **Feature is non-functional**

### When Will It Work?

The hook system will be functional when:
1. `HookExecutor` class is implemented
2. Integration points added to tool/agent execution
3. Process management (spawn, I/O, timeout) is built
4. Hook output processing is implemented
5. Error handling is added

This documentation describes the **designed architecture** and provides examples of how hooks **will work** once the execution engine is implemented.
