# Gemini CLI Hooks System Analysis

## Overview

Hooks are a powerful extension mechanism in Gemini CLI that allows you to
intercept and customize agent behavior at specific lifecycle events. They enable
you to run custom shell commands at various points during execution to:

- Validate or block tool calls before execution
- Add additional context after tool execution
- Modify LLM requests/responses
- Control tool selection
- React to session events

---

## What Are Hooks?

Hooks are **shell commands** that execute at specific **lifecycle events**
during Gemini CLI's operation. They receive context as JSON via stdin and can
output decisions/modifications as JSON to stdout.

**Key Characteristics:**

- **Event-driven**: Triggered at specific points in the execution flow
- **Shell-based**: Execute as external commands/scripts
- **Bidirectional**: Receive input via stdin, return output via stdout
- **Non-blocking**: Can run in parallel or sequential modes
- **Configurable**: Defined in `~/.gemini/settings.json` or project
  `.gemini/settings.json`

---

## Available Hook Events

### Tool Lifecycle Hooks

#### 1. `BeforeTool`

**When**: Before a tool is executed

**Purpose**:

- Validate tool calls
- Block dangerous operations
- Add safety checks

**Input** (`BeforeToolInput`):

```typescript
{
  session_id: string;           // Unique session identifier
  transcript_path: string;      // Path to session transcript
  cwd: string;                  // Current working directory
  hook_event_name: "BeforeTool";
  timestamp: string;            // ISO 8601 timestamp
  tool_name: string;            // Name of tool being called
  tool_input: {                 // Tool parameters
    [key: string]: unknown;
  }
}
```

**Output** (`BeforeToolOutput`):

```typescript
{
  decision?: "ask" | "block" | "deny" | "approve" | "allow";
  reason?: string;                    // Explanation for decision
  continue?: boolean;                 // Stop execution if false
  stopReason?: string;               // Why execution should stop
  suppressOutput?: boolean;          // Hide hook output from user
  systemMessage?: string;            // Message to show user
  hookSpecificOutput?: {
    permissionDecision?: "block" | "deny" | "approve" | "allow";
    permissionDecisionReason?: string;
  }
}
```

**Use Cases:**

- Prevent file writes to protected directories
- Block shell commands matching dangerous patterns
- Require user approval for sensitive operations

---

#### 2. `AfterTool`

**When**: After a tool has executed

**Purpose**:

- Add additional context based on tool results
- Post-process tool outputs
- Log tool executions

**Input** (`AfterToolInput`):

```typescript
{
  session_id: string;
  transcript_path: string;
  cwd: string;
  hook_event_name: "AfterTool";
  timestamp: string;
  tool_name: string;
  tool_input: { [key: string]: unknown };
  tool_response: { [key: string]: unknown };  // Tool execution result
}
```

**Output** (`AfterToolOutput`):

```typescript
{
  continue?: boolean;
  stopReason?: string;
  suppressOutput?: boolean;
  systemMessage?: string;
  hookSpecificOutput?: {
    additionalContext?: string;  // Extra context for the LLM
  }
}
```

**Use Cases:**

- Add coding guidelines after reading a file
- Inject test requirements after code edits
- Append security reminders after shell commands

---

### LLM Lifecycle Hooks

#### 3. `BeforeModel`

**When**: Before sending request to LLM

**Purpose**:

- Modify LLM request parameters
- Inject synthetic responses (bypass LLM call)
- Change model configuration

**Input** (`BeforeModelInput`):

```typescript
{
  session_id: string;
  transcript_path: string;
  cwd: string;
  hook_event_name: "BeforeModel";
  timestamp: string;
  llm_request: {
    model: string;
    messages: Array<{
      role: "user" | "model" | "system";
      content: string | Array<{ type: string; [key: string]: unknown }>;
    }>;
    config?: {
      temperature?: number;
      maxOutputTokens?: number;
      topP?: number;
      topK?: number;
      stopSequences?: string[];
      // ... other config
    };
    toolConfig?: {
      mode?: "AUTO" | "ANY" | "NONE";
      allowedFunctionNames?: string[];
    };
  }
}
```

**Output** (`BeforeModelOutput`):

```typescript
{
  decision?: "block" | "deny" | "approve" | "allow";
  reason?: string;
  continue?: boolean;
  stopReason?: string;
  suppressOutput?: boolean;
  systemMessage?: string;
  hookSpecificOutput?: {
    llm_request?: Partial<LLMRequest>;  // Modifications to request
    llm_response?: LLMResponse;         // Synthetic response (skips LLM)
  }
}
```

**Use Cases:**

- Lower temperature for code generation
- Inject canned responses for testing
- Block LLM calls during certain conditions
- Add system messages dynamically

---

#### 4. `AfterModel`

**When**: After receiving LLM response

**Purpose**:

- Modify LLM responses
- Filter/validate model outputs
- Stop execution based on response

**Input** (`AfterModelInput`):

```typescript
{
  session_id: string;
  transcript_path: string;
  cwd: string;
  hook_event_name: "AfterModel";
  timestamp: string;
  llm_request: LLMRequest;   // Original request
  llm_response: {
    text?: string;
    candidates: Array<{
      content: {
        role: "model";
        parts: string[];
      };
      finishReason?: "STOP" | "MAX_TOKENS" | "SAFETY" | "RECITATION" | "OTHER";
      safetyRatings?: Array<{
        category: string;
        probability: string;
        blocked?: boolean;
      }>;
    }>;
    usageMetadata?: {
      promptTokenCount?: number;
      candidatesTokenCount?: number;
      totalTokenCount?: number;
    };
  }
}
```

**Output** (`AfterModelOutput`):

```typescript
{
  continue?: boolean;
  stopReason?: string;
  suppressOutput?: boolean;
  systemMessage?: string;
  hookSpecificOutput?: {
    llm_response?: Partial<LLMResponse>;  // Modified response
  }
}
```

**Use Cases:**

- Filter sensitive information from responses
- Add disclaimers to certain responses
- Stop if response violates policy
- Log token usage

---

#### 5. `BeforeToolSelection`

**When**: Before the model selects which tools to use

**Purpose**:

- Dynamically control which tools are available
- Change tool selection mode (AUTO/ANY/NONE)
- Restrict tool access based on context

**Input** (`BeforeToolSelectionInput`):

```typescript
{
  session_id: string;
  transcript_path: string;
  cwd: string;
  hook_event_name: 'BeforeToolSelection';
  timestamp: string;
  llm_request: LLMRequest;
}
```

**Output** (`BeforeToolSelectionOutput`):

```typescript
{
  continue?: boolean;
  stopReason?: string;
  suppressOutput?: boolean;
  systemMessage?: string;
  hookSpecificOutput?: {
    toolConfig?: {
      mode?: "AUTO" | "ANY" | "NONE";
      allowedFunctionNames?: string[];
    }
  }
}
```

**Use Cases:**

- Disable tools during read-only operations
- Allow only specific tools in certain directories
- Force manual tool selection (mode: "NONE")

---

### Agent Lifecycle Hooks

#### 6. `BeforeAgent`

**When**: Before agent processes user prompt

**Purpose**:

- Add context before processing
- Validate prompts
- Pre-process user input

**Input** (`BeforeAgentInput`):

```typescript
{
  session_id: string;
  transcript_path: string;
  cwd: string;
  hook_event_name: 'BeforeAgent';
  timestamp: string;
  prompt: string; // User's input prompt
}
```

**Output** (`BeforeAgentOutput`):

```typescript
{
  continue?: boolean;
  stopReason?: string;
  suppressOutput?: boolean;
  systemMessage?: string;
  hookSpecificOutput?: {
    additionalContext?: string;  // Context to add to prompt
  }
}
```

**Use Cases:**

- Add project-specific context to prompts
- Block certain types of requests
- Inject coding standards

---

#### 7. `AfterAgent`

**When**: After agent completes response

**Purpose**:

- Post-process agent responses
- Log completed interactions
- Cleanup operations

**Input** (`AfterAgentInput`):

```typescript
{
  session_id: string;
  transcript_path: string;
  cwd: string;
  hook_event_name: 'AfterAgent';
  timestamp: string;
  prompt: string; // Original user prompt
  prompt_response: string; // Agent's response
  stop_hook_active: boolean; // Whether stop hook is active
}
```

---

### Session Lifecycle Hooks

#### 8. `SessionStart`

**When**: When a session starts

**Purpose**:

- Initialize session-specific state
- Load project configuration
- Set up environment

**Input** (`SessionStartInput`):

```typescript
{
  session_id: string;
  transcript_path: string;
  cwd: string;
  hook_event_name: 'SessionStart';
  timestamp: string;
  source: 'startup' | 'resume' | 'clear' | 'compress';
}
```

**Output** (`SessionStartOutput`):

```typescript
{
  continue?: boolean;
  stopReason?: string;
  suppressOutput?: boolean;
  systemMessage?: string;
  hookSpecificOutput?: {
    additionalContext?: string;  // Context for initial session
  }
}
```

**Use Cases:**

- Display project guidelines
- Check environment prerequisites
- Load project-specific memory

---

#### 9. `SessionEnd`

**When**: When a session ends

**Purpose**:

- Cleanup operations
- Save session state
- Analytics

**Input** (`SessionEndInput`):

```typescript
{
  session_id: string;
  transcript_path: string;
  cwd: string;
  hook_event_name: 'SessionEnd';
  timestamp: string;
  reason: 'exit' | 'clear' | 'logout' | 'prompt_input_exit' | 'other';
}
```

**Use Cases:**

- Archive transcripts
- Clean up temporary files
- Submit usage analytics

---

#### 10. `PreCompress`

**When**: Before conversation history compression

**Purpose**:

- Prepare for compression
- Save important context externally
- Notify user

**Input** (`PreCompressInput`):

```typescript
{
  session_id: string;
  transcript_path: string;
  cwd: string;
  hook_event_name: 'PreCompress';
  timestamp: string;
  trigger: 'manual' | 'auto';
}
```

---

#### 11. `Notification`

**When**: System notifications occur

**Purpose**:

- React to notifications
- Suppress certain notifications
- Custom notification handling

**Input** (`NotificationInput`):

```typescript
{
  session_id: string;
  transcript_path: string;
  cwd: string;
  hook_event_name: "Notification";
  timestamp: string;
  notification_type: "ToolPermission" | ...;
  message: string;
  details: { [key: string]: unknown };
}
```

---

## Hook Configuration

### Configuration Location

Hooks are configured in `settings.json`:

- **Project-level**: `.gemini/settings.json` (highest priority)
- **User-level**: `~/.gemini/settings.json`
- **Extension-level**: Via extensions configuration

### Configuration Structure

```json
{
  "tools": {
    "enableHooks": true // REQUIRED: Enable hooks system
  },
  "hooks": {
    "BeforeTool": [
      {
        "matcher": "EditTool", // Optional: Match specific tools (regex)
        "sequential": false, // Optional: Run sequentially vs parallel
        "hooks": [
          {
            "type": "command",
            "command": "./hooks/check_style.sh",
            "timeout": 60000 // Optional: Timeout in ms
          }
        ]
      }
    ],
    "AfterTool": [
      {
        "matcher": ".*", // Match all tools
        "hooks": [
          {
            "type": "command",
            "command": "node ./hooks/log_tool.js"
          }
        ]
      }
    ],
    "BeforeModel": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python ./hooks/modify_request.py"
          }
        ]
      }
    ]
  }
}
```

### Configuration Fields

| Field             | Type    | Required | Description                                |
| ----------------- | ------- | -------- | ------------------------------------------ |
| `matcher`         | string  | No       | Regex pattern to match tools/triggers      |
| `sequential`      | boolean | No       | Run hooks sequentially (default: parallel) |
| `hooks`           | array   | Yes      | Array of hook configurations               |
| `hooks[].type`    | string  | Yes      | Hook type (currently only "command")       |
| `hooks[].command` | string  | Yes      | Shell command to execute                   |
| `hooks[].timeout` | number  | No       | Timeout in milliseconds                    |

---

## Hook Execution Flow

### 1. Event Trigger

```
User Action / System Event
    ↓
Event Name Determined (e.g., BeforeTool)
    ↓
HookRegistry.getHooksForEvent(eventName)
```

### 2. Hook Planning

```
HookPlanner.createExecutionPlan()
    ↓
Filter by matcher (if specified)
    ↓
Deduplicate identical hooks
    ↓
Determine execution strategy (sequential vs parallel)
    ↓
Create HookExecutionPlan
```

### 3. Hook Execution

```
For each hook in plan:
    ↓
1. Prepare input JSON (event-specific)
    ↓
2. Execute shell command with stdin
    ↓
3. Read stdout/stderr
    ↓
4. Parse JSON output
    ↓
5. Create HookOutput object
```

### 4. Decision Application

```
Process HookOutput
    ↓
Check blocking decision (block/deny)
    ↓
Apply modifications (if any)
    ↓
Add additional context (if any)
    ↓
Continue or stop execution
```

---

## Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│  USER ACTION / SYSTEM EVENT                                 │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  EVENT DETERMINATION                                        │
│  ──────────────────────────────────────────────────────────│
│  Examples:                                                  │
│  - User prompt → BeforeAgent                                │
│  - Tool call → BeforeTool                                   │
│  - LLM request → BeforeModel                                │
│  - Session start → SessionStart                             │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  HOOK REGISTRY LOOKUP                                       │
│  ──────────────────────────────────────────────────────────│
│  HookRegistry.getHooksForEvent(eventName)                   │
│  - Filters by event name                                    │
│  - Returns enabled hooks                                    │
│  - Sorted by source priority (Project > User > Extensions)  │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  HOOK PLANNING                                              │
│  ──────────────────────────────────────────────────────────│
│  HookPlanner.createExecutionPlan()                          │
│  1. Filter by matcher (tool name, trigger, etc.)            │
│  2. Deduplicate identical hooks                             │
│  3. Determine execution mode (sequential vs parallel)       │
│  4. Create execution plan                                   │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
    [No Hooks]            [Has Hooks]
          │                     │
          │                     ▼
          │     ┌──────────────────────────────────────┐
          │     │  PREPARE HOOK INPUT                  │
          │     │  ────────────────────────────────────│
          │     │  Create event-specific JSON:         │
          │     │  - session_id, timestamp, cwd        │
          │     │  - Event-specific fields             │
          │     │    (tool_name, llm_request, etc.)    │
          │     └────────────┬─────────────────────────┘
          │                  │
          │                  ▼
          │     ┌──────────────────────────────────────┐
          │     │  EXECUTE HOOKS                       │
          │     │  ────────────────────────────────────│
          │     │  If sequential=true:                 │
          │     │    Execute hooks one by one          │
          │     │  Else:                               │
          │     │    Execute hooks in parallel         │
          │     │                                      │
          │     │  For each hook:                      │
          │     │  1. Spawn shell command              │
          │     │  2. Write input JSON to stdin        │
          │     │  3. Read stdout/stderr               │
          │     │  4. Parse JSON output                │
          │     │  5. Create HookExecutionResult       │
          │     └────────────┬─────────────────────────┘
          │                  │
          │                  ▼
          │     ┌──────────────────────────────────────┐
          │     │  PROCESS HOOK OUTPUTS                │
          │     │  ────────────────────────────────────│
          │     │  For each successful result:         │
          │     │  1. Parse output JSON                │
          │     │  2. Create HookOutput object         │
          │     │  3. Check decisions                  │
          │     └────────────┬─────────────────────────┘
          │                  │
          │                  ▼
          │     ┌──────────────────────────────────────┐
          │     │  DECISION EVALUATION                 │
          │     │  ────────────────────────────────────│
          │     │  Check for blocking decisions:       │
          │     │  - decision: "block" or "deny"       │
          │     │  - continue: false                   │
          │     │                                      │
          │     │  If blocked:                         │
          │     │    → Stop execution                  │
          │     │    → Show reason to user             │
          │     │  Else:                               │
          │     │    → Continue to next step           │
          │     └────────────┬─────────────────────────┘
          │                  │
          │                  ▼
          │     ┌──────────────────────────────────────┐
          │     │  APPLY MODIFICATIONS                 │
          │     │  ────────────────────────────────────│
          │     │  Event-specific modifications:       │
          │     │                                      │
          │     │  BeforeModel:                        │
          │     │    - Modify LLM request              │
          │     │    - Inject synthetic response       │
          │     │                                      │
          │     │  AfterModel:                         │
          │     │    - Modify LLM response             │
          │     │                                      │
          │     │  BeforeToolSelection:                │
          │     │    - Modify tool config              │
          │     │    - Change allowed tools            │
          │     │                                      │
          │     │  BeforeAgent/AfterTool:              │
          │     │    - Add additional context          │
          │     └────────────┬─────────────────────────┘
          │                  │
          └──────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  CONTINUE NORMAL EXECUTION                                  │
│  ──────────────────────────────────────────────────────────│
│  With modifications/context applied                         │
└─────────────────────────────────────────────────────────────┘
```

---

## Example Hook Scripts

### 1. Block Dangerous Shell Commands

**File**: `.gemini/hooks/check_shell.sh`

```bash
#!/bin/bash

# Read input JSON from stdin
input=$(cat)

# Extract tool name and command
tool_name=$(echo "$input" | jq -r '.tool_name')
command=$(echo "$input" | jq -r '.tool_input.command // ""')

# Check if this is a shell command
if [ "$tool_name" = "shell" ]; then
    # Block dangerous commands
    if echo "$command" | grep -qE "rm -rf /|dd if=|mkfs|:(){ :|:& };:"; then
        # Return blocking decision
        jq -n \
            --arg reason "Dangerous command blocked: $command" \
            '{
                decision: "block",
                reason: $reason,
                systemMessage: "⚠️  Hook blocked dangerous shell command"
            }'
        exit 0
    fi
fi

# Allow by default
echo '{}'
```

**Configuration**:

```json
{
  "hooks": {
    "BeforeTool": [
      {
        "matcher": "shell",
        "hooks": [
          {
            "type": "command",
            "command": ".gemini/hooks/check_shell.sh"
          }
        ]
      }
    ]
  }
}
```

---

### 2. Add Coding Standards After File Reads

**File**: `.gemini/hooks/add_standards.py`

```python
#!/usr/bin/env python3
import json
import sys
import os

# Read input
input_data = json.load(sys.stdin)
tool_name = input_data.get('tool_name', '')
tool_input = input_data.get('tool_input', {})

# Check if we read a code file
if tool_name == 'read_file':
    file_path = tool_input.get('file_path', '')

    # Check if it's a Python file
    if file_path.endswith('.py'):
        output = {
            'hookSpecificOutput': {
                'additionalContext': (
                    'CODING STANDARDS:\n'
                    '- Use type hints for all function parameters\n'
                    '- Maximum line length: 100 characters\n'
                    '- Follow PEP 8 style guide\n'
                    '- Add docstrings to all public functions'
                )
            }
        }
        json.dump(output, sys.stdout)
        sys.exit(0)

# No action
json.dump({}, sys.stdout)
```

**Configuration**:

```json
{
  "hooks": {
    "AfterTool": [
      {
        "matcher": "read_file",
        "hooks": [
          {
            "type": "command",
            "command": "python .gemini/hooks/add_standards.py"
          }
        ]
      }
    ]
  }
}
```

---

### 3. Lower Temperature for Code Generation

**File**: `.gemini/hooks/adjust_temperature.js`

```javascript
#!/usr/bin/env node
const input = JSON.parse(require('fs').readFileSync(0, 'utf-8'));

// Check if recent messages mention code generation
const messages = input.llm_request?.messages || [];
const recentText = messages
  .slice(-3)
  .map((m) => (typeof m.content === 'string' ? m.content : ''))
  .join(' ')
  .toLowerCase();

if (
  recentText.includes('write') ||
  recentText.includes('code') ||
  recentText.includes('function')
) {
  // Lower temperature for more deterministic code
  const output = {
    hookSpecificOutput: {
      llm_request: {
        config: {
          temperature: 0.2,
          topP: 0.8,
        },
      },
    },
    systemMessage: '🎯 Adjusted temperature for code generation',
  };

  console.log(JSON.stringify(output));
} else {
  // No modification
  console.log('{}');
}
```

**Configuration**:

```json
{
  "hooks": {
    "BeforeModel": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node .gemini/hooks/adjust_temperature.js"
          }
        ]
      }
    ]
  }
}
```

---

### 4. Log All Tool Executions

**File**: `.gemini/hooks/log_tools.sh`

```bash
#!/bin/bash

input=$(cat)

tool_name=$(echo "$input" | jq -r '.tool_name')
timestamp=$(echo "$input" | jq -r '.timestamp')
session_id=$(echo "$input" | jq -r '.session_id')

# Append to log file
echo "[$timestamp] Session: $session_id | Tool: $tool_name" >> ~/.gemini/tool_audit.log

# Don't modify execution
echo '{}'
```

**Configuration**:

```json
{
  "hooks": {
    "BeforeTool": [
      {
        "hooks": [
          {
            "type": "command",
            "command": ".gemini/hooks/log_tools.sh"
          }
        ]
      }
    ]
  }
}
```

---

## Hook Execution Modes

### Parallel Execution (Default)

```json
{
  "hooks": {
    "BeforeTool": [
      {
        "sequential": false, // or omit (default)
        "hooks": [
          { "type": "command", "command": "./hook1.sh" },
          { "type": "command", "command": "./hook2.sh" },
          { "type": "command", "command": "./hook3.sh" }
        ]
      }
    ]
  }
}
```

**Behavior**: All three hooks execute simultaneously

- **Faster**: Hooks run concurrently
- **Order**: Results processed as they complete
- **Use when**: Hooks are independent

### Sequential Execution

```json
{
  "hooks": {
    "BeforeTool": [
      {
        "sequential": true,
        "hooks": [
          { "type": "command", "command": "./hook1.sh" },
          { "type": "command", "command": "./hook2.sh" },
          { "type": "command", "command": "./hook3.sh" }
        ]
      }
    ]
  }
}
```

**Behavior**: Hooks execute one after another

- **Ordered**: hook1 → hook2 → hook3
- **Slower**: Total time = sum of all hooks
- **Use when**: Hooks depend on each other or order matters

---

## Hook Priority and Source

Hooks from different sources have different priorities:

1. **Project** (`.gemini/settings.json`) - Highest priority
2. **User** (`~/.gemini/settings.json`)
3. **System** (Built-in defaults)
4. **Extensions** (From installed extensions) - Lowest priority

When hooks from multiple sources match the same event, they are sorted by
priority and all executed (unless deduplicated).

---

## Matchers

Matchers allow you to target specific tools or triggers:

### Tool Name Matching

```json
{
  "BeforeTool": [
    {
      "matcher": "EditTool",  // Exact match
      "hooks": [...]
    },
    {
      "matcher": ".*Tool",    // Regex: any tool ending with "Tool"
      "hooks": [...]
    },
    {
      "matcher": "shell|bash",  // Regex: shell OR bash
      "hooks": [...]
    }
  ]
}
```

### Match All

```json
{
  "matcher": "*",    // Wildcard: matches all
  "hooks": [...]
}
```

Or omit matcher:

```json
{
  // No matcher = matches all
  "hooks": [...]
}
```

---

## Enabling Hooks

Hooks system requires explicit enablement:

```json
{
  "tools": {
    "enableHooks": true // REQUIRED
  },
  "hooks": {
    // Your hook configurations
  }
}
```

**Important**: Setting `enableHooks: true` is REQUIRED for hooks to execute.

---

## Key Implementation Files

| File                                        | Purpose                                       |
| ------------------------------------------- | --------------------------------------------- |
| `packages/core/src/hooks/types.ts`          | Hook type definitions, events, I/O interfaces |
| `packages/core/src/hooks/hookRegistry.ts`   | Loads and validates hooks from config         |
| `packages/core/src/hooks/hookPlanner.ts`    | Matches hooks and creates execution plans     |
| `packages/core/src/hooks/hookTranslator.ts` | Converts between SDK and hook formats         |
| `packages/core/src/config/config.ts`        | Hook configuration storage                    |

---

## Summary

### Hook Events by Category

**Tool Lifecycle**: `BeforeTool`, `AfterTool` **LLM Lifecycle**: `BeforeModel`,
`AfterModel`, `BeforeToolSelection` **Agent Lifecycle**: `BeforeAgent`,
`AfterAgent` **Session Lifecycle**: `SessionStart`, `SessionEnd`, `PreCompress`,
`Notification`

### Capabilities

✅ **Block/Allow**: Control tool execution ✅ **Modify**: Change LLM
requests/responses ✅ **Inject**: Add context to prompts ✅ **React**: Respond
to events ✅ **Log**: Audit all operations ✅ **Validate**: Enforce policies

### Best Practices

1. **Keep hooks fast**: Slow hooks block execution
2. **Use matchers**: Only run hooks when needed
3. **Handle errors**: Hooks should never crash
4. **Return JSON**: Always output valid JSON (or `{}`)
5. **Test thoroughly**: Hook errors can break workflows
6. **Document**: Comment your hook scripts
7. **Version control**: Keep hooks with project

---

## Limitations

- **Command-only**: Currently only shell commands supported (no plugins)
- **Requires enablement**: Must set `tools.enableHooks: true`
- **No async hooks**: Hooks must complete and return
- **JSON-based**: Input/output must be JSON
- **Platform-dependent**: Shell scripts may not be cross-platform
