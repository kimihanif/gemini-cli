# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with
code in this repository.

## Commands

### Development

- `npm start` - Start CLI from source (runs from `packages/cli/dist`)
- `npm run debug` - Start CLI in debug mode with inspector
- `npm run build` - Build all packages (TypeScript → JavaScript)
- `npm run build:all` - Build CLI + sandbox container + VS Code extension
- `npm run build:sandbox` - Build sandbox Docker/Podman container only

### Testing

- `npm run test` - Run unit tests across all packages (Vitest)
- `npm run test:ci` - Run tests in CI mode
- `npm run test:e2e` - Run integration tests without sandboxing
- `npm run test:integration:sandbox:none` - Integration tests without sandbox
- `npm run test:integration:sandbox:docker` - Integration tests with Docker
  sandbox
- `npm run test:integration:sandbox:podman` - Integration tests with Podman
  sandbox
- `npm run test:scripts` - Run tests for build scripts

### Code Quality

- `npm run preflight` - **Run before submitting PRs** (clean, install, format,
  lint, build, typecheck, test)
- `npm run lint` - Lint TypeScript files with ESLint
- `npm run lint:fix` - Auto-fix linting issues and format
- `npm run format` - Format code with Prettier
- `npm run typecheck` - Run TypeScript type checking across packages

## Architecture

### Package Structure

This is a monorepo with workspaces in `packages/`:

1. **`packages/cli`** - Frontend (user-facing terminal interface)
   - Uses React + Ink for terminal UI rendering
   - Handles input processing, history, display, themes
   - Entry point: `dist/index.js` (built from `src/`)

2. **`packages/core`** - Backend (business logic)
   - Gemini API client and prompt management
   - Tool registration and execution
   - State management for conversations
   - MCP (Model Context Protocol) server integration

3. **`packages/test-utils`** - Shared test utilities
   - Temporary file system helpers for tests

4. **`packages/vscode-ide-companion`** - VS Code extension
   - Companion extension that pairs with Gemini CLI

5. **`packages/a2a-server`** - A2A server (Experimental)
   - Agent-to-agent communication server

### Tool System

Tools are located in `packages/core/src/tools/` and extend Gemini's
capabilities:

- File system operations (read, write, edit files)
- Shell command execution
- Web fetching and search (with Google Search grounding)
- MCP server integrations for custom tools

### Build System

- Built with `esbuild` for fast bundling
- TypeScript compiled to ESM (ES modules)
- Bundle output: `bundle/gemini.js`
- Build scripts in `scripts/` directory handle package compilation

### Configuration Files

- `~/.gemini/settings.json` - User settings (MCP servers, etc.)
- `GEMINI.md` - Project-specific context for the CLI
- `.gemini/.env` - Project-specific environment variables
- `~/.env` - Global environment variables

## Development Workflow

### Running from Source

After making changes:

```bash
npm run build        # Rebuild packages
npm start           # Run CLI from source
```

Or use the combined command:

```bash
npm run build-and-start
```

### Testing Strategy

- **Unit tests**: Co-located with source files (`*.test.ts`, `*.test.tsx`)
- **Integration tests**: In `integration-tests/` directory
- Test framework: Vitest with `describe`, `it`, `expect`, `vi`
- React component tests use `ink-testing-library`
- Always run `npm run preflight` before submitting changes

### Sandboxing

Multiple sandboxing modes available via `GEMINI_SANDBOX` env var:

- `false` - No sandboxing
- `true` - Auto-detect Docker or Podman
- `docker` - Use Docker container
- `podman` - Use Podman container

On macOS, Seatbelt profiles available:

- `permissive-open` (default) - Restricts writes to project folder
- `restrictive-closed` - Denies all operations by default

Build sandbox: `npm run build:sandbox`

### Branch and PR Guidelines

- Main branch: `main`
- PRs must link to existing issues
- Keep PRs small and focused on single issues
- Run `/review-frontend <PR_NUMBER>` for automated React review (experimental)
- All PRs must pass `npm run preflight` checks

## Code Architecture Patterns

### Communication Flow

```
User Input → CLI Package → Core Package → Gemini API
                ↑              ↓
                └─── Tools ←───┘
```

1. CLI receives user input
2. Core constructs prompt and calls Gemini API
3. Gemini may request tool execution
4. Tools execute (with user approval for writes)
5. Results flow back through Core to CLI
6. CLI displays formatted output to user

### Tool Execution

- Read-only tools (file reads, searches) often auto-approve
- Write tools (file edits, shell commands) require user confirmation
- Tools send results back to Gemini API for final response generation

### React Component Structure

- CLI UI built with React and Ink (terminal rendering)
- Components in `packages/cli/src/ui/`
- Custom hooks in `packages/cli/src/hooks/`
- Follow React best practices from GEMINI.md (functional components, hooks,
  immutability)

### Module System

- TypeScript with ES modules (`"type": "module"`)
- Use `import`/`export` for encapsulation (not class private members)
- Target: ES2022, Module: NodeNext
- Strict TypeScript config enabled

## Important Notes

### TypeScript Guidelines

- Avoid `any` type - use `unknown` with type narrowing instead
- Avoid type assertions unless absolutely necessary
- Use `checkExhaustive` helper in switch default clauses (see
  `packages/cli/src/utils/checks.ts`)
- Prefer plain objects + interfaces over classes
- Never use placeholders in tool parameters

### Testing Conventions

- Mock ES modules with `vi.mock('module-name', async (importOriginal) => ...)`
- Place critical dependency mocks at top of test files
- Use `vi.hoisted(() => vi.fn())` for hoisted mocks
- Common mocked modules: `fs`, `os`, `@google/genai`,
  `@modelcontextprotocol/sdk`
- Always include cleanup in `afterEach` with `vi.restoreAllMocks()`

### React Best Practices

- Use functional components with hooks only (no classes)
- Avoid `useEffect` for side effects - prefer event handlers
- Never call `setState` inside `useEffect` (performance issue)
- Keep components pure during rendering
- Use functional state updates: `setCount(c => c + 1)`
- React Compiler enabled - skip manual `useMemo`/`useCallback`/`React.memo`

### Code Style

- Use hyphens in flag names, not underscores (`--my-flag`)
- Minimize comments - let code be self-documenting
- Use array operators (`.map()`, `.filter()`, `.reduce()`) for functional style
- Follow Google Developer Documentation Style Guide for docs

### Integration with External Systems

- Supports MCP servers for extensibility
- Configure in `~/.gemini/settings.json`
- OAuth login with Google Account (free tier: 60 req/min, 1000 req/day)
- Alternative auth: Gemini API key or Vertex AI
- GitHub Actions integration available via `run-gemini-cli` action

## File Locations

### Key Source Files

- CLI entry: `packages/cli/src/gemini.tsx`
- Core entry: `packages/core/src/index.ts`
- Commands: `packages/cli/src/commands/`
- Tools: `packages/core/src/tools/`
- Config: `packages/cli/src/config/`

### Build Artifacts

- Built packages: `packages/*/dist/`
- Bundle output: `bundle/gemini.js`
- Sandbox Dockerfile: `Dockerfile`

### Documentation

- Main docs: `docs/` directory
- Architecture: `docs/architecture.md`
- Integration tests: `docs/integration-tests.md`
- Contributing: `CONTRIBUTING.md`
- Project-specific AI context: `GEMINI.md`
