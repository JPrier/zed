# Zed Agentic Coding - Documentation Summary

This directory contains comprehensive documentation on how Zed implements agentic coding (AI-powered code generation and editing).

## Overview

Zed's agentic coding system is a production-ready implementation that connects large language models (LLMs) with a codebase through a tool-based architecture. The agent can read files, edit code, run commands, search for patterns, and more - all while maintaining safety, transparency, and user control.

## Documentation Files

### 1. [AGENTIC_CODING_ARCHITECTURE.md](./AGENTIC_CODING_ARCHITECTURE.md)

**The main architectural document** - Start here for a complete understanding.

**Contents:**
- **Overview**: Design principles and high-level architecture
- **Core Architecture**: Directory structure and component hierarchy
- **Component Deep Dive**: 
  - NativeAgent (session management)
  - Thread (conversation state)
  - Tool System (15+ built-in tools)
  - EditAgent (specialized file editing)
- **Tool System**: How tools are implemented, registered, and secured
- **LLM Integration**: Provider abstraction, streaming, prompt construction
- **File Editing System**: Two-layer edit architecture with fuzzy matching
- **UI Integration**: ACP thread layer, event streaming, diff visualization
- **Data Flow**: Complete request/response cycle
- **Implementation Guide**: Phase-by-phase guide to building your own

**Best for:** Understanding the complete system architecture and design decisions.

### 2. [AGENTIC_CODING_DIAGRAMS.md](./AGENTIC_CODING_DIAGRAMS.md)

**Visual representations** - Use this to understand data flow and interactions.

**Contents:**
- System Architecture (component hierarchy)
- Request/Response Flow (complete lifecycle)
- Edit Agent Flow (how file editing works)
- Tool Execution with Authorization (permission system)
- Message Context Formatting (how @mentions work)
- Persistence & Recovery (database and session management)
- Model Selection & Registry (LLM provider management)
- Token Management & Context Windows (handling token limits)

**Best for:** Visual learners and understanding how components interact.

### 3. [AGENTIC_CODING_EXAMPLES.md](./AGENTIC_CODING_EXAMPLES.md)

**Practical code examples** - Use this as a reference implementation.

**Contents:**
- Minimal Agent Implementation (basic conversation loop)
- Tool Implementation Examples:
  - ReadFile (with line ranges and security)
  - Terminal (command execution with timeout)
  - Grep (regex search across files)
- LLM Integration Examples:
  - Abstract model interface
  - Anthropic implementation
  - Streaming response handling
- Edit Agent Examples:
  - Simple full-file replacement
  - XML-based incremental edits
  - Fuzzy matching for edit application
- Authorization System (permission requests)
- Testing Examples (unit and integration tests)

**Best for:** Developers building their own implementation.

## Quick Start

**Want to understand Zed's agentic coding?**
1. Read [AGENTIC_CODING_ARCHITECTURE.md](./AGENTIC_CODING_ARCHITECTURE.md) sections 1-2 (Overview & Core Architecture)
2. Look at [AGENTIC_CODING_DIAGRAMS.md](./AGENTIC_CODING_DIAGRAMS.md) (System Architecture and Request/Response Flow)
3. Explore the actual code in `crates/agent/src/`

**Want to build your own agentic coding system?**
1. Read [AGENTIC_CODING_ARCHITECTURE.md](./AGENTIC_CODING_ARCHITECTURE.md) section "Implementation Guide"
2. Use [AGENTIC_CODING_EXAMPLES.md](./AGENTIC_CODING_EXAMPLES.md) as reference implementations
3. Study the diagrams in [AGENTIC_CODING_DIAGRAMS.md](./AGENTIC_CODING_DIAGRAMS.md) to understand data flow

**Want to extend Zed's agent?**
1. Read the "Tool System" section in [AGENTIC_CODING_ARCHITECTURE.md](./AGENTIC_CODING_ARCHITECTURE.md)
2. Study existing tool implementations in [AGENTIC_CODING_EXAMPLES.md](./AGENTIC_CODING_EXAMPLES.md)
3. Look at `crates/agent/src/tools/` for real examples

## Key Concepts

### 1. Tool-First Architecture

The agent doesn't have direct access to files or commands. Instead, it uses **tools** - structured, validated, and sandboxed capabilities. Each tool:
- Has a JSON schema for inputs
- Returns structured outputs
- Can request authorization
- Is fully testable in isolation

### 2. Two-Layer Edit System

File editing uses a two-agent approach:
1. **Primary Agent** (conversational) decides *what* to edit and *why*
2. **Edit Agent** (specialized) figures out *how* to edit it precisely

This separation allows:
- Better prompts for each task
- Different models optimized for each purpose
- Streaming edits with fuzzy matching
- Recovery from LLM mistakes

### 3. Streaming Everything

All operations stream results:
- LLM responses stream text and tool calls
- Edit operations stream edits as they're parsed
- Tool outputs stream to UI
- File diffs stream as changes occur

This provides:
- Immediate feedback
- Ability to cancel long operations
- Better UX with real-time progress

### 4. Safety by Design

Multiple layers of protection:
- **Path validation**: All file paths must be within project
- **Private files**: Respect `private_files` settings
- **Authorization**: Dangerous operations require permission
- **Sandboxing**: Tools can't escape their constraints
- **Audit trail**: All actions logged in ActionLog

### 5. Asynchronous Architecture

Built on GPUI's async runtime:
- No blocking the UI thread
- Parallel tool execution where safe
- Graceful cancellation
- Error recovery

## Architecture Highlights

### Component Hierarchy

```
NativeAgent                    # Top-level coordinator
  ├─ Sessions                  # Active agent sessions
  │   ├─ Thread                # Core conversation logic
  │   └─ AcpThread            # UI protocol layer
  ├─ LanguageModels           # Model registry
  ├─ ProjectContext           # Cached project info
  └─ Templates                # Prompt templates
```

### Request Lifecycle

```
User Input → Thread.run_next_turn()
  → Build LLM request (system prompt + history + tools)
  → Stream completion from LLM
  → Parse tool calls
  → Execute tools (with authorization)
  → Append tool results
  → Continue turn if needed
  → Save to database
```

### Tools Available

**File Operations**: ReadFile, EditFile, SaveFile, DeletePath, CopyPath, MovePath, CreateDirectory, RestoreFileFromDisk

**Navigation**: FindPath, Grep, ListDirectory, Diagnostics

**Execution**: Terminal

**Context**: OpenTool (opens files in editor), NowTool (current time)

**Advanced**: ThinkingTool (extended reasoning), FetchTool (HTTP), WebSearchTool

## Key Files in Codebase

```
crates/agent/src/
├── agent.rs                 # NativeAgent - main coordinator
├── thread.rs                # Thread - conversation state & execution
├── tools.rs                 # Tool registry and trait definitions
├── edit_agent.rs           # EditAgent - specialized file editing
├── templates.rs            # Handlebars template system
├── templates/
│   └── system_prompt.hbs   # Main system prompt template
└── tools/
    ├── read_file_tool.rs   # Example tool: read files
    ├── edit_file_tool.rs   # Example tool: edit files
    ├── terminal_tool.rs    # Example tool: run commands
    └── ... (12+ more tools)
```

## Implementation Phases

If you're building your own system, follow these phases:

**Phase 1**: Core infrastructure (Project, Buffer, async runtime)
**Phase 2**: LLM integration (one provider, basic streaming)
**Phase 3**: Tool system (ReadFile, WriteFile, RunCommand)
**Phase 4**: Edit agent (full-file replacement first)
**Phase 5**: Conversation management (Thread state, history)
**Phase 6**: UI integration (message view, diffs, terminals)
**Phase 7**: Persistence (database, serialization)
**Phase 8**: Advanced features (parallel tools, streaming edits, caching)

See [AGENTIC_CODING_ARCHITECTURE.md](./AGENTIC_CODING_ARCHITECTURE.md) "Implementation Guide" for detailed steps.

## Common Patterns

### Tool Implementation

1. Define input struct with `JsonSchema` derive
2. Implement `AgentTool` trait
3. Add authorization checks in `run()`
4. Send updates via `ToolCallEventStream`
5. Return formatted result

### LLM Integration

1. Implement `LanguageModel` trait
2. Convert between your format and provider format
3. Stream `CompletionEvent`s back
4. Handle errors with exponential backoff
5. Support tool calling

### Testing

1. Use temporary directories for file operations
2. Mock LLM responses for deterministic tests
3. Test security boundaries (path traversal, etc.)
4. Test authorization flow
5. Integration tests with real LLM (expensive, mark as such)

## Performance Considerations

### Prompt Caching

Claude supports prompt caching - cache the system prompt and project context to reduce:
- Latency (faster responses)
- Cost (90% reduction for cached content)

Mark messages with `cache: true` to enable.

### Token Management

1. Count tokens before each request
2. Truncate middle of conversation if needed
3. Always keep: system prompt, recent messages, tool schemas
4. Insert "[Earlier messages truncated]" marker

### Streaming

1. Stream all LLM responses
2. Parse tool calls as they arrive
3. Apply edits incrementally
4. Update UI in real-time

## Security Best Practices

1. **Validate all paths**: Ensure within project roots
2. **Respect settings**: Honor `private_files` and `file_scan_exclusions`
3. **Request authorization**: For destructive operations (overwrite, delete)
4. **Sanitize inputs**: Prevent command injection in Terminal tool
5. **Audit trail**: Log all tool executions
6. **Rate limiting**: Prevent API abuse
7. **Timeout commands**: Bound execution time

## Debugging Tips

1. **Enable verbose logging**: See all LLM requests/responses
2. **Check ActionLog**: See what agent has done
3. **Inspect tool results**: Verify tools returned expected data
4. **Test tools in isolation**: Rule out LLM vs tool issues
5. **Mock LLM**: Test agent logic without API calls
6. **Check token counts**: Ensure context isn't truncated unexpectedly

## Further Resources

- **Zed Source Code**: https://github.com/zed-industries/zed
- **GPUI Framework**: https://www.gpui.rs
- **Model Context Protocol**: https://modelcontextprotocol.io
- **Anthropic Tool Use**: https://docs.anthropic.com/claude/docs/tool-use
- **OpenAI Function Calling**: https://platform.openai.com/docs/guides/function-calling

## Contributing

If you find issues or want to improve these docs:
1. Check existing documentation is accurate
2. Add examples where helpful
3. Keep diagrams up to date with code changes
4. Test code examples actually work

---

## Summary

Zed's agentic coding system is a sophisticated, production-ready implementation that demonstrates best practices for:
- LLM-powered code generation
- Tool-based agent architecture
- Safe file system operations
- Real-time streaming UX
- Multi-turn conversations
- Authorization and security

These documents provide everything you need to understand, use, extend, or replicate this system. Start with the architecture document, refer to diagrams for visual understanding, and use examples as reference implementations.

Good luck building your own agentic coding system!
