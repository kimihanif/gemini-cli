# Gemini CLI System Prompts and Routing Analysis

## Overview

The Gemini CLI uses a sophisticated multi-layered system for managing prompts
and routing requests to appropriate models. This document explains the
architecture, flow, and decision-making process.

---

## System Prompts Hierarchy

### 1. Core System Prompt (Main Agent)

**Location**: `packages/core/src/core/prompts.ts` → `getCoreSystemPrompt()`

**When Used**: For the main interactive CLI agent (the agent users directly
interact with)

**How It's Applied**: Called in `packages/core/src/core/client.ts` at lines 190
and in the `startChat()` method

**Structure**: The core system prompt is built from modular sections that can be
dynamically enabled/disabled:

```typescript
const promptConfig = {
  preamble: 'You are an interactive CLI agent...',
  coreMandates: '# Core Mandates...',
  primaryWorkflows_prefix: '# Primary Workflows...',
  primaryWorkflows_prefix_ci: '...', // When CodebaseInvestigator is enabled
  primaryWorkflows_prefix_ci_todo: '...', // When both CI and TodoWrite are enabled
  primaryWorkflows_todo: '...', // When only TodoWrite is enabled
  primaryWorkflows_suffix: '3. **Implement:** ...',
  operationalGuidelines: '# Operational Guidelines...',
  sandbox: '# Sandbox/Seatbelt information...',
  git: '# Git Repository guidance...',
  finalReminder: '# Final Reminder...',
};
```

**Dynamic Sections**:

- **Primary Workflows**: Changes based on enabled tools:
  - If `CodebaseInvestigatorAgent` + `WriteTodosTool` → uses
    `primaryWorkflows_prefix_ci_todo`
  - If only `CodebaseInvestigatorAgent` → uses `primaryWorkflows_prefix_ci`
  - If only `WriteTodosTool` → uses `primaryWorkflows_todo`
  - Otherwise → uses `primaryWorkflows_prefix`

- **Sandbox Section**: Dynamically includes sandbox warnings based on
  environment:
  - macOS Seatbelt mode (`SANDBOX=sandbox-exec`)
  - Generic sandbox mode (Docker/Podman)
  - No sandbox warning

- **Git Section**: Only included if current directory is a git repository

**Environment Variable Overrides**:

- `GEMINI_SYSTEM_MD` - Path to custom system prompt file (overrides entire
  prompt)
- `GEMINI_WRITE_SYSTEM_MD` - Write computed system prompt to file for inspection
- `GEMINI_PROMPT_<SECTION>` - Set to "0" or "false" to disable specific sections

**User Memory Suffix**: If user has saved memories (via the Memory tool),
they're appended as:

```
---
<user memory content>
```

---

### 2. Classifier System Prompt (Model Router)

**Location**: `packages/core/src/routing/strategies/classifierStrategy.ts` →
`CLASSIFIER_SYSTEM_PROMPT`

**When Used**: During the routing phase to decide between Flash (fast) or Pro
(powerful) models

**Purpose**: Analyzes request complexity to route to appropriate model

**Prompt Structure**:

```
You are a specialized Task Routing AI. Your sole function is to analyze
the user's request and classify its complexity. Choose between `flash`
(SIMPLE) or `pro` (COMPLEX).
```

**Complexity Rubric** (decides Flash vs Pro):

- **COMPLEX (Pro)** if ANY of:
  1. High Operational Complexity (4+ steps/tool calls)
  2. Strategic Planning & Conceptual Design ("how" or "why" questions)
  3. High Ambiguity or Large Scope (extensive investigation)
  4. Deep Debugging & Root Cause Analysis

- **SIMPLE (Flash)** if:
  - Highly specific, bounded tasks
  - Low Operational Complexity (1-3 tool calls)

**Output Format**: JSON with `reasoning` and `model_choice` fields

**Context Provided**:

- Last 4 turns of clean history (tool calls filtered out)
- Current user request

---

### 3. Agent System Prompts (Sub-agents)

**Location**: `packages/core/src/agents/*.ts` → Each agent's
`AgentDefinition.promptConfig.systemPrompt`

**When Used**: When a specialized sub-agent is invoked as a tool

**Example: Codebase Investigator Agent**

**Location**: `packages/core/src/agents/codebase-investigator.ts`

**Prompt Structure**:

```
You are **Codebase Investigator**, a hyper-specialized AI agent and an
expert in reverse-engineering complex software projects...

## Core Directives
1. DEEP ANALYSIS, NOT JUST FILE FINDING
2. SYSTEMATIC & CURIOUS EXPLORATION
3. HOLISTIC & PRECISE
4. Web Search allowed

## Scratchpad Management
[Detailed instructions for maintaining investigation state]

## Termination
Must call `complete_task` tool when done
```

**Query Template**:

```typescript
query: `Your task is to do a deep investigation of the codebase to find
all relevant files, code locations, architectural mental map and insights
to solve for the following user objective:
<objective>
${objective}
</objective>`;
```

**Available Tools**: Limited to read-only tools:

- `ls`
- `read_file`
- `glob`
- `grep`

**Agent-Specific Configuration**:

- Model: `DEFAULT_GEMINI_MODEL` (Pro)
- Temperature: 0.1
- Top P: 0.95
- Thinking Budget: -1 (unlimited)
- Max Time: 5 minutes
- Max Turns: 15

---

### 4. Compression System Prompt

**Location**: `packages/core/src/core/prompts.ts` → `getCompressionPrompt()`

**When Used**: When conversation history becomes too long and needs compression

**Purpose**: Summarizes chat history into structured XML snapshot

**Prompt Structure**:

```
You are the component that summarizes internal chat history into a given
structure.

[Instructions for creating structured snapshot]

<state_snapshot>
  <overall_goal>...</overall_goal>
  <key_knowledge>...</key_knowledge>
  <file_system_state>...</file_system_state>
  <recent_actions>...</recent_actions>
  <current_plan>...</current_plan>
</state_snapshot>
```

**Output**: XML-formatted state snapshot that becomes agent's memory

---

## Model Routing Flow

### Routing Strategy Chain of Responsibility

**Entry Point**: `packages/core/src/routing/modelRouterService.ts` →
`ModelRouterService`

The system uses a **CompositeStrategy** with prioritized strategies:

```typescript
new CompositeStrategy(
  [
    new FallbackStrategy(), // Priority 1
    new OverrideStrategy(), // Priority 2
    new ClassifierStrategy(), // Priority 3
    new DefaultStrategy(), // Priority 4 (Terminal)
  ],
  'agent-router',
);
```

### Strategy Execution Flow

```
┌─────────────────────────────────────────────────┐
│  User Request Arrives                           │
└─────────────────┬───────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────┐
│  1. FallbackStrategy                            │
│  ────────────────────────────────────────────   │
│  Check: config.isInFallbackMode()               │
│  If YES: Return fallback model                  │
│  If NO:  Return null → Continue to next         │
└─────────────────┬───────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────┐
│  2. OverrideStrategy                            │
│  ────────────────────────────────────────────   │
│  Check: config.getModel() != 'auto'             │
│  If user specified model: Return that model     │
│  If 'auto': Return null → Continue to next      │
└─────────────────┬───────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────┐
│  3. ClassifierStrategy                          │
│  ────────────────────────────────────────────   │
│  Actions:                                       │
│  1. Get last 20 history items                   │
│  2. Filter out function calls/responses         │
│  3. Take last 4 clean turns                     │
│  4. Call LLM with CLASSIFIER_SYSTEM_PROMPT      │
│  5. Parse JSON response                         │
│  6. Return Flash or Pro model based on result   │
│  If error: Return null → Continue to next       │
└─────────────────┬───────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────┐
│  4. DefaultStrategy (Terminal)                  │
│  ────────────────────────────────────────────   │
│  Always returns: DEFAULT_GEMINI_MODEL (Pro)     │
│  Cannot return null - guarantees decision       │
└─────────────────┬───────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────┐
│  Routing Decision Made                          │
│  ────────────────────────────────────────────   │
│  Contains:                                      │
│  - model: string                                │
│  - metadata:                                    │
│    - source: "agent-router/<strategy>"          │
│    - latencyMs: number                          │
│    - reasoning: string                          │
└─────────────────────────────────────────────────┘
```

---

## Complete Request Flow

### Main Agent Request Flow

```
1. USER INPUT
   │
   ▼
2. ROUTING PHASE (ModelRouterService.route())
   │
   ├─► FallbackStrategy checks
   ├─► OverrideStrategy checks
   ├─► ClassifierStrategy (uses CLASSIFIER_SYSTEM_PROMPT)
   │   └─► Calls LLM to classify complexity
   │       └─► Returns Flash or Pro
   └─► DefaultStrategy (fallback to Pro)
   │
   ▼
3. MODEL SELECTED
   │
   ▼
4. SYSTEM PROMPT CONSTRUCTION (getCoreSystemPrompt())
   │
   ├─► Build modular sections based on:
   │   ├─► Enabled tools (CodebaseInvestigator, WriteTodos)
   │   ├─► Sandbox mode (Seatbelt, Docker, None)
   │   ├─► Git repository status
   │   └─► Environment variable overrides
   │
   ├─► Append user memory if exists
   │
   └─► Return final system prompt string
   │
   ▼
5. CHAT INITIALIZATION (GeminiClient.startChat())
   │
   ├─► Create GeminiChat with:
   │   ├─► systemInstruction: getCoreSystemPrompt()
   │   ├─► tools: All registered tool declarations
   │   ├─► history: Initial context + previous history
   │   └─► config: temp=0, topP=1, thinking config
   │
   ▼
6. GENERATE RESPONSE
   │
   ├─► Model uses system prompt as identity/rules
   ├─► Model has access to all tools
   └─► Model generates response
   │
   ▼
7. TOOL EXECUTION (if requested)
   │
   ├─► If tool is sub-agent (e.g., codebase_investigator):
   │   │
   │   ├─► AgentExecutor.create()
   │   ├─► Use agent's systemPrompt (e.g., Codebase Investigator prompt)
   │   ├─► Template with input parameters
   │   ├─► Limited tool access (agent-specific)
   │   └─► Run agent loop until complete_task called
   │
   └─► If regular tool:
       └─► Execute and return result
   │
   ▼
8. RESPONSE TO USER
```

---

## Key Decision Points

### When is the Core System Prompt Used?

**Used for**: Main interactive agent (the CLI user talks to)

**Not used for**:

- Model routing decisions (uses Classifier prompt)
- Sub-agents (use their own system prompts)
- Compression (uses Compression prompt)

### When is Model Routing Performed?

**Performed**:

- At the start of each user turn
- Before calling the main LLM
- Uses last 4 clean conversation turns as context

**Bypassed when**:

- User explicitly specified a model (via `-m` flag or config)
- System is in fallback mode
- Classifier fails (falls back to Default strategy → Pro)

### When are Sub-Agent Prompts Used?

**Triggered when**:

- Main agent calls a sub-agent tool (e.g., `codebase_investigator`)
- Sub-agent runs in isolated loop with own prompt
- Sub-agent has limited tool access

**Example**: When main agent calls `codebase_investigator`:

1. Main agent uses Core System Prompt
2. Decides to invoke `codebase_investigator` tool
3. AgentExecutor creates new chat session
4. Uses Codebase Investigator System Prompt
5. Runs in loop with only read tools
6. Returns structured report
7. Main agent receives report and continues

---

## Prompt Selection Decision Tree

```
Is this a routing decision?
├─► YES: Use CLASSIFIER_SYSTEM_PROMPT
│   └─► Classify complexity → Flash or Pro
│
└─► NO: Is this the main agent?
    ├─► YES: Use getCoreSystemPrompt()
    │   ├─► Check for GEMINI_SYSTEM_MD override
    │   ├─► Build from modular sections
    │   ├─► Add user memory if exists
    │   └─► Return core prompt
    │
    └─► NO: Is this a sub-agent?
        ├─► YES: Use agent's systemPrompt
        │   ├─► Template with input parameters
        │   └─► Add directory context if needed
        │
        └─► NO: Is this compression?
            └─► YES: Use getCompressionPrompt()
```

---

## Environment Variables That Affect Prompts

| Variable                  | Effect                                                  |
| ------------------------- | ------------------------------------------------------- |
| `GEMINI_SYSTEM_MD`        | Override entire core system prompt with custom file     |
| `GEMINI_WRITE_SYSTEM_MD`  | Write computed system prompt to file for inspection     |
| `GEMINI_PROMPT_<SECTION>` | Disable specific prompt section (set to "0" or "false") |
| `SANDBOX`                 | Affects sandbox section of core prompt                  |
| `GEMINI_SANDBOX`          | Affects sandbox configuration and warnings              |

**Example**:

```bash
# Disable git-specific instructions
export GEMINI_PROMPT_GIT=false

# Use custom system prompt
export GEMINI_SYSTEM_MD=~/.gemini/custom-system.md

# Write computed prompt to file
export GEMINI_WRITE_SYSTEM_MD=true
# → Writes to ~/.gemini/system.md
```

---

## Summary

### System Prompt Types

1. **Core System Prompt** - Main agent identity and rules (modular, dynamic)
2. **Classifier Prompt** - Routes requests to Flash or Pro (fixed, small)
3. **Agent Prompts** - Specialized sub-agents (task-specific)
4. **Compression Prompt** - Summarizes history (structured XML output)

### Routing Strategy Priority

1. **Fallback** (emergency mode)
2. **Override** (user-specified model)
3. **Classifier** (AI-based complexity analysis)
4. **Default** (fallback to Pro)

### Key Files

- `packages/core/src/core/prompts.ts` - Core and compression prompts
- `packages/core/src/routing/strategies/classifierStrategy.ts` - Classifier
  prompt
- `packages/core/src/agents/codebase-investigator.ts` - Example agent prompt
- `packages/core/src/routing/modelRouterService.ts` - Routing orchestration
- `packages/core/src/core/client.ts` - Main agent initialization
- `packages/core/src/agents/executor.ts` - Sub-agent execution

### Flow Summary

```
User Input
  → Routing (Classifier Prompt)
  → Model Selection
  → Main Agent (Core System Prompt)
  → [Optional] Sub-Agent Invocation (Agent-Specific Prompt)
  → Response
  → [Optional] Compression (Compression Prompt)
```
