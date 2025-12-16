# Zed Agentic Coding Architecture

This document provides an in-depth explanation of how Zed implements agentic coding (AI-powered code generation and editing). If you're looking to replicate this system, this guide will walk you through every component.

## Table of Contents

1. [Overview](#overview)
2. [Core Architecture](#core-architecture)
3. [Component Deep Dive](#component-deep-dive)
4. [Tool System](#tool-system)
5. [LLM Integration](#llm-integration)
6. [File Editing System](#file-editing-system)
7. [UI Integration](#ui-integration)
8. [Data Flow](#data-flow)
9. [Implementation Guide](#implementation-guide)

---

## Overview

Zed's agentic coding system is built around a multi-layered architecture that connects:
- **Language Models (LLMs)** - The AI that powers code generation
- **Tools** - Structured capabilities the agent can use (read files, edit code, run commands, etc.)
- **Project Context** - Real-time access to the codebase and environment
- **Thread Management** - Conversation and execution state
- **UI Layer** - Visual representation and user interaction

### Key Design Principles

1. **Tool-First Architecture**: The agent interacts with the codebase through structured tools, not direct file access
2. **Safety**: Multiple layers of validation, authorization, and sandboxing
3. **Transparency**: All agent actions are visible and trackable
4. **Asynchronous**: Everything is non-blocking with GPUI's async architecture
5. **Stateful**: Conversation threads maintain context across multiple turns

---

## Core Architecture

### Directory Structure

```
crates/
├── agent/                    # Core agent logic
│   ├── src/
│   │   ├── agent.rs         # NativeAgent - main coordinator
│   │   ├── thread.rs        # Thread - conversation & execution state
│   │   ├── tools.rs         # Tool registry and definitions
│   │   ├── edit_agent.rs    # Specialized file editing logic
│   │   ├── templates.rs     # Handlebars prompt templates
│   │   ├── db.rs            # Persistent storage
│   │   └── tools/           # Individual tool implementations
│   │       ├── read_file_tool.rs
│   │       ├── edit_file_tool.rs
│   │       ├── terminal_tool.rs
│   │       └── ... (15+ tools)
│   └── templates/           # .hbs template files
├── acp_thread/              # Agent Client Protocol thread layer
├── agent_ui/                # User interface components
├── agent_settings/          # Configuration system
├── language_model/          # LLM abstraction layer
└── prompt_store/            # Project context management
```

### Primary Components

```
┌─────────────────────────────────────────────────────────────┐
│                         User Interface                       │
│                      (agent_ui crate)                        │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│                      AcpThread Layer                         │
│              (Protocol & UI Communication)                   │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│                      NativeAgent                             │
│         (Session Management, Model Selection)                │
└───────────────┬───────────────────────────┬─────────────────┘
                │                           │
┌───────────────▼─────────────┐  ┌─────────▼─────────────────┐
│         Thread               │  │    LanguageModels         │
│  (Conversation & Tools)      │  │  (Model Registry)         │
└───────────────┬─────────────┘  └───────────────────────────┘
                │
┌───────────────▼─────────────────────────────────────────────┐
│                      Tool System                             │
│    (ReadFile, EditFile, Terminal, Grep, etc.)               │
└───────────────────────────────────────────────────────────────┘
```

---

## Component Deep Dive

### 1. NativeAgent (`agent.rs`)

**Purpose**: Top-level coordinator that manages agent sessions, models, and project context.

**Key Responsibilities**:
- Session lifecycle management (create, track, cleanup)
- Language model registry and authentication
- Project context maintenance
- Thread creation and registration

**Core Structure**:
```rust
pub struct NativeAgent {
    sessions: HashMap<SessionId, Session>,      // Active agent sessions
    history: Entity<HistoryStore>,              // Persistent conversation history
    project_context: Entity<ProjectContext>,    // Cached project information
    context_server_registry: Entity<...>,       // External context servers
    templates: Arc<Templates>,                  // Prompt templates
    models: LanguageModels,                     // Available LLM models
    project: Entity<Project>,                   // The active project
    prompt_store: Option<Entity<PromptStore>>,  // Custom prompts/rules
    fs: Arc<dyn Fs>,                           // Filesystem access
}
```

**Session Management**:
```rust
struct Session {
    thread: Entity<Thread>,              // The conversation thread
    acp_thread: WeakEntity<AcpThread>,  // UI protocol thread
    pending_save: Task<()>,              // Async save task
    _subscriptions: Vec<Subscription>,   // Event listeners
}
```

**Model Management**:
The `LanguageModels` struct maintains a registry of available models:
- Discovers models from all registered providers (OpenAI, Anthropic, etc.)
- Handles authentication for each provider
- Groups models (e.g., "Recommended", provider-specific groups)
- Watches for provider updates

### 2. Thread (`thread.rs`)

**Purpose**: Represents a single agent conversation/session with execution state.

**Key Responsibilities**:
- Message history management
- Tool execution coordination
- LLM request/response handling
- Token usage tracking
- Title/summary generation

**Core Structure**:
```rust
pub struct Thread {
    id: SessionId,
    prompt_id: PromptId,
    messages: Vec<Message>,              // Conversation history
    user_store: Entity<UserStore>,       
    completion_mode: CompletionMode,     // How aggressive the agent should be
    running_turn: Option<RunningTurn>,   // Current LLM interaction
    pending_message: Option<AgentMessage>,
    tools: BTreeMap<SharedString, Arc<dyn AnyAgentTool>>,  // Available tools
    model: Option<Arc<dyn LanguageModel>>,  // The LLM being used
    project: Entity<Project>,
    action_log: Entity<ActionLog>,       // What the agent has done
    // ... many more fields for state management
}
```

**Message Types**:
```rust
pub enum Message {
    User(UserMessage),     // User input with context
    Agent(AgentMessage),   // Agent response with tool uses
    Resume,                // Special: continue from truncation
}

pub struct UserMessage {
    id: UserMessageId,
    content: Vec<UserMessageContent>,  // Text, mentions, images
}

pub enum UserMessageContent {
    Text(String),
    Mention { uri: MentionUri, content: String },  // @file, @directory, etc.
    Image(LanguageModelImage),
}

pub struct AgentMessage {
    content: Vec<AgentMessageContent>,  // Text, thinking, tool uses
    tool_results: IndexMap<ToolUseId, ToolResult>,
    reasoning_details: Option<serde_json::Value>,
}
```

**Turn Execution**:

The `running_turn` field tracks the current agent interaction cycle:

1. User sends a message
2. Thread builds LLM request with system prompt + history + tools
3. Stream response from LLM
4. Parse tool calls from response
5. Execute tools (with authorization)
6. Send tool results back to LLM
7. Repeat steps 3-6 until agent provides final response

```rust
struct RunningTurn {
    task: Task<Result<()>>,           // Async execution
    reason: TurnReason,                // Why this turn started
    _subscription: Subscription,       // Stop request listener
}
```

### 3. Tool System (`tools.rs` and `tools/`)

**Purpose**: Structured capabilities that the agent can invoke.

**Tool Trait**:
```rust
pub trait AgentTool: Send + Sync + 'static {
    type Input: JsonSchema + DeserializeOwned;
    type Output: Into<LanguageModelToolResultContent>;
    
    fn name() -> &'static str;
    fn description() -> &'static str;
    fn kind() -> ToolKind;  // Read, Write, Execute
    
    fn input_schema(format: ToolSchemaFormat) -> Schema;
    
    fn initial_title(&self, input: Result<Self::Input>, cx: &mut App) 
        -> SharedString;
    
    fn run(
        self: Arc<Self>,
        input: Self::Input,
        event_stream: ToolCallEventStream,
        cx: &mut App,
    ) -> Task<Result<LanguageModelToolResultContent>>;
}
```

**Available Built-in Tools**:

1. **File Operations**:
   - `ReadFileTool`: Read file contents (with line ranges)
   - `EditFileTool`: Create/edit/overwrite files using the EditAgent
   - `SaveFileTool`: Explicitly save unsaved changes
   - `CopyPathTool`: Copy files/directories
   - `MovePathTool`: Move/rename files/directories
   - `DeletePathTool`: Delete files/directories
   - `RestoreFileFromDiskTool`: Revert unsaved changes

2. **Navigation**:
   - `FindPathTool`: Search for files by name pattern
   - `GrepTool`: Search file contents using regex
   - `ListDirectoryTool`: List directory contents
   - `DiagnosticsTool`: Get compiler/linter errors

3. **Execution**:
   - `TerminalTool`: Run shell commands with output streaming

4. **Information**:
   - `NowTool`: Get current date/time
   - `ThinkingTool`: Extended reasoning (for compatible models)
   - `FetchTool`: Fetch external URLs
   - `WebSearchTool`: Web search integration

5. **Context**:
   - `OpenTool`: Open files in the editor (for user visibility)

**Tool Authorization**:

Tools implement authorization through the `initial_title` and event stream:
```rust
// Tools can request authorization with options
event_stream.request_authorization(vec![
    PermissionOption { id: "allow", label: "Allow" },
    PermissionOption { id: "deny", label: "Deny" },
]);
```

### 4. EditAgent (`edit_agent.rs`)

**Purpose**: Specialized subsystem for AI-powered file editing using streaming diffs.

**Why Separate?**: File editing is complex enough to warrant its own agent:
- Needs a different LLM model (optimized for edits)
- Uses specialized prompts (XML or diff format)
- Handles streaming character-by-character edits
- Manages fuzzy matching of old text to new text

**Core Structure**:
```rust
pub struct EditAgent {
    model: Arc<dyn LanguageModel>,     // Specialized edit model
    action_log: Entity<ActionLog>,
    project: Entity<Project>,
    templates: Arc<Templates>,
    edit_format: EditFormat,            // XML or DiffFenced
}

pub enum EditFormat {
    Xml,          // <old_text>...</old_text><new_text>...</new_text>
    DiffFenced,   // ```diff format
}
```

**Edit Flow**:

1. Primary agent calls `EditFileTool` with high-level description
2. `EditFileTool` constructs detailed prompt for EditAgent
3. EditAgent streams back edit instructions (XML tags or diff hunks)
4. `EditParser` parses streaming output in real-time
5. `StreamingFuzzyMatcher` finds where edits apply in the file
6. Buffer is updated incrementally as edits stream in

**Edit Parsers**:

Two formats supported:

1. **XML Format** (`edit_parser.rs`):
```xml
<edits>
<old_text line=10>
existing code here
</old_text>
<new_text>
new code here
</new_text>
</edits>
```

2. **Diff Fenced Format**:
````
```diff
- old line
+ new line
```
````

**Streaming Fuzzy Matching**:

The `StreamingFuzzyMatcher` is critical for robustness:
- Handles whitespace differences
- Tolerates minor mistakes by the LLM
- Finds the best match for `old_text` in the buffer
- Reports ambiguous matches to user if multiple locations found

---

## Tool System

### Tool Registration

Tools are registered when a Thread is created:

```rust
impl Thread {
    fn add_default_tools(&mut self, environment: Rc<dyn ThreadEnvironment>, cx: &mut Context<Self>) {
        // Built-in tools
        self.register_tool(Arc::new(ReadFileTool::new(...)));
        self.register_tool(Arc::new(EditFileTool::new(...)));
        self.register_tool(Arc::new(TerminalTool::new(...)));
        // ... all other tools
        
        // Context server tools (external)
        for tool in context_servers.tools() {
            self.register_tool(tool);
        }
    }
}
```

### Tool Execution Flow

```
User Input → Thread.run_next_turn()
    ↓
Build LLM Request with tool schemas
    ↓
Stream LLM Response
    ↓
Parse tool_use blocks
    ↓
For each tool_use:
    ├─→ Look up tool by name
    ├─→ Deserialize input JSON
    ├─→ Request authorization (if needed)
    ├─→ Execute tool.run()
    └─→ Collect ToolResult
    ↓
Append tool results to conversation
    ↓
Continue turn (LLM responds to tool results)
```

### Tool Input Validation

Tools use `schemars` for JSON Schema generation:

```rust
#[derive(Serialize, Deserialize, JsonSchema)]
pub struct ReadFileToolInput {
    /// The relative path of the file to read
    pub path: String,
    /// Optional line number to start reading (1-based)
    #[serde(default)]
    pub start_line: Option<u32>,
    /// Optional line number to end reading (1-based)
    #[serde(default)]
    pub end_line: Option<u32>,
}
```

This schema is:
1. Converted to JSON Schema format
2. Sent to the LLM as part of tool definition
3. LLM generates JSON matching the schema
4. Deserialized back into the typed struct

### Security Considerations

**File Access Controls**:
- Tools respect `WorktreeSettings`:
  - `file_scan_exclusions`: Files that should not be visible
  - `private_files`: Sensitive files (e.g., `.env`, `*.key`)
- All paths are validated against project roots
- Absolute path access outside worktrees is denied
- Path traversal (`../`) is prevented

**Command Execution**:
- Terminal commands run in project directory by default
- Can specify working directory per command
- Output can be limited with `output_byte_limit`
- Timeout support to prevent indefinite execution

---

## LLM Integration

### Language Model Abstraction

Zed abstracts over multiple LLM providers through the `language_model` crate:

```rust
pub trait LanguageModel: Send + Sync {
    fn id(&self) -> &LanguageModelId;
    fn name(&self) -> LanguageModelName;
    fn provider_id(&self) -> &LanguageModelProviderId;
    fn provider_name(&self) -> LanguageModelProviderName;
    fn max_token_count(&self) -> usize;
    
    fn count_tokens(&self, request: LanguageModelRequest, cx: &App) 
        -> BoxFuture<'static, Result<usize>>;
    
    fn stream_completion(
        &self,
        request: LanguageModelRequest,
        cx: &AsyncApp,
    ) -> BoxFuture<'static, Result<impl Stream<Item = Result<LanguageModelCompletionEvent>>>>;
}
```

**Supported Providers**:
- Anthropic (Claude)
- OpenAI (GPT)
- Google (Gemini)
- Ollama (local models)
- LM Studio (local models)
- Copilot Chat
- Zed Cloud (proxied access)

### Request Structure

```rust
pub struct LanguageModelRequest {
    pub messages: Vec<LanguageModelRequestMessage>,
    pub tools: Vec<LanguageModelRequestTool>,
    pub tool_choice: Option<LanguageModelToolChoice>,
    pub stop_sequences: Vec<String>,
    pub temperature: Option<f32>,
}

pub struct LanguageModelRequestMessage {
    pub role: Role,  // System, User, Assistant
    pub content: Vec<MessageContent>,
    pub cache: bool,  // Use prompt caching if available
    pub reasoning_details: Option<serde_json::Value>,
}

pub enum MessageContent {
    Text(String),
    Image(LanguageModelImage),
    ToolUse(LanguageModelToolUse),
    ToolResult(LanguageModelToolResult),
    Thinking { text: String, signature: Option<String> },
    RedactedThinking(String),
}
```

### Streaming Response

LLM responses stream in as events:

```rust
pub enum LanguageModelCompletionEvent {
    StartMessage { id: Option<String> },
    Text(String),              // Incremental text
    ToolUse {
        id: ToolUseId,
        name: String,
        input: serde_json::Value,
    },
    Thinking { text: String },
    Stop(StopReason),
    UsageDetails(TokenUsage),
}

pub enum StopReason {
    EndTurn,          // Natural completion
    MaxTokens,        // Hit token limit
    StopSequence,     // Hit stop sequence
    ToolUse,          // Agent wants to use tools
    ContentFilter,    // Blocked by safety filter
}
```

### Prompt Construction

The system prompt is generated from a Handlebars template (`system_prompt.hbs`):

```rust
let system_prompt = SystemPromptTemplate {
    project: &project_context,      // Worktrees, rules files
    available_tools: tool_names,    // List of tool names
    model_name: Some(model.name()), // Which model is being used
}.render(&templates)?;
```

**System Prompt Includes**:
1. Role description ("You are a highly skilled software engineer...")
2. Communication guidelines
3. Tool use instructions
4. Code block formatting rules (path-based: ````path/to/file.rs#L10-20`)
5. Project structure (root directories)
6. Tool-specific guidance (when to use grep vs find_path, etc.)
7. Custom user rules (from .zed/rules or prompt store)

### Context Management

**Mentions** allow users to explicitly add context:
- `@file`: Entire file contents
- `@directory`: Directory listing
- `@symbol`: Specific function/class
- `@selection`: Selected code range
- `@thread`: Previous conversation

These are packaged into structured XML in the user message:
```xml
<context>
<files>
```rust path/to/file.rs
(file contents)
```
</files>
<symbols>
```rust path/to/other.rs:10-50
(symbol contents)
```
</symbols>
</context>
```

---

## File Editing System

### Architecture

The editing system has two layers:

1. **Primary Agent** (`Thread`): Decides *what* to edit and *why*
2. **Edit Agent** (`EditAgent`): Figures out *how* to edit it

### Edit Flow Detail

```
Primary Agent decides to edit file
    ↓
Calls EditFileTool with:
    - path: "src/main.rs"
    - mode: Edit | Create | Overwrite
    - display_description: "Add error handling to main function"
    ↓
EditFileTool reads current file (if exists)
    ↓
Constructs detailed prompt for EditAgent:
    - Full file contents
    - Edit description
    - Line numbers
    - Format instructions (XML or diff)
    ↓
EditAgent streams back edit instructions
    ↓
EditParser parses streaming XML/diff
    ↓
For each edit:
    ├─→ Extract old_text and new_text
    ├─→ StreamingFuzzyMatcher finds best match
    ├─→ Report location to UI
    └─→ Apply edit to buffer
    ↓
Format file (if format_on_save enabled)
    ↓
Return diff to primary agent
```

### Edit Modes

**1. Edit Mode** (incremental changes):
- Best for modifying existing files
- Preserves most of the file
- Uses fuzzy matching to handle whitespace

**2. Create Mode** (new file):
- For files that don't exist yet
- No old_text matching needed
- Can also overwrite if file exists

**3. Overwrite Mode** (full replacement):
- When file needs complete rewrite
- More efficient than many small edits
- Used when structure changes significantly

### Streaming Diff Application

The streaming application is sophisticated:

```rust
// As chunks arrive from LLM
while let Some(chunk) = stream.next().await {
    match chunk {
        "<old_text>" => {
            current_old_text.clear();
            in_old_text = true;
        }
        "</old_text>" => {
            in_old_text = false;
        }
        "<new_text>" => {
            // Start collecting new text
            current_new_text.clear();
            in_new_text = true;
        }
        "</new_text>" => {
            // Apply edit now
            let range = fuzzy_match(&current_old_text, &buffer)?;
            buffer.edit(range, &current_new_text);
            // UI updates immediately
        }
        _ if in_old_text => current_old_text.push_str(&chunk),
        _ if in_new_text => current_new_text.push_str(&chunk),
    }
}
```

### Error Recovery

**Fuzzy Matching**:
- Normalizes whitespace before comparing
- Uses Levenshtein distance for similarity
- Reports ambiguous matches (multiple candidates)
- Falls back to line numbers if provided

**User Feedback**:
```rust
pub enum EditAgentOutputEvent {
    ResolvingEditRange(Range<Anchor>),  // Searching for match
    UnresolvedEditRange,                 // Couldn't find old_text
    AmbiguousEditRange(Vec<Range>),      // Multiple matches found
    Edited(Range<Anchor>),               // Successfully applied
}
```

---

## UI Integration

### ACP Thread Layer

The `AcpThread` (Agent Client Protocol Thread) is the bridge between core agent logic and UI:

```rust
pub struct AcpThread {
    id: SessionId,
    title: SharedString,
    entries: Vec<AgentThreadEntry>,        // Rendered conversation
    connection: Rc<dyn AgentConnection>,   // Callback interface
    project: Entity<Project>,
    action_log: Entity<ActionLog>,
    // ... UI state
}
```

**AgentConnection Trait**:
```rust
pub trait AgentConnection {
    fn run_prompt(&self, params: RunPromptParams) -> Task<Result<()>>;
    fn stop_prompt(&self) -> Task<Result<()>>;
    fn update_configuration(&self, ...) -> Task<Result<()>>;
    fn model_list(&self) -> Task<Result<AgentModelList>>;
}
```

### Event Stream

Tools communicate with UI through `ToolCallEventStream`:

```rust
pub struct ToolCallEventStream {
    tx: UnboundedSender<ToolCallUpdate>,
}

impl ToolCallEventStream {
    pub fn update_fields(&self, fields: ToolCallUpdateFields) {
        // Update tool call title, content, status, locations, etc.
    }
    
    pub fn request_authorization(&self, options: Vec<PermissionOption>) 
        -> oneshot::Receiver<PermissionOptionId> {
        // Pause execution, show modal to user, wait for response
    }
}
```

### Diff Visualization

File edits are shown as diffs in the UI:

```rust
pub struct Diff {
    buffer: Entity<Buffer>,
    base_text: Arc<String>,        // Text before edit
    current_snapshot: BufferSnapshot,
    diff_hunks: Vec<DiffHunk>,     // What changed
}

pub struct DiffHunk {
    base_range: Range<usize>,
    buffer_range: Range<Anchor>,
    status: DiffHunkStatus,        // Added, Removed, Modified
}
```

### Terminal Integration

Commands run through the terminal tool create persistent terminals:

```rust
pub struct Terminal {
    id: TerminalId,
    command: String,
    status: TerminalStatus,
    output: Arc<RwLock<Vec<u8>>>,     // Buffered output
    exit_status: Shared<Task<ExitStatus>>,
}

pub enum TerminalStatus {
    Running,
    Completed,
    Cancelled,
}
```

---

## Data Flow

### Complete Request Flow

```
1. User types message and presses Enter
    ↓
2. UI (agent_ui) → AcpThread.submit_user_message()
    ↓
3. AcpThread → NativeAgentConnection.run_prompt()
    ↓
4. NativeAgent finds Session → Thread
    ↓
5. Thread.run_next_turn() begins
    ├─→ Build system prompt from template
    ├─→ Convert message history to LLM format
    ├─→ Attach tool schemas
    └─→ Call model.stream_completion()
    ↓
6. Stream LLM response
    ├─→ Parse text chunks
    ├─→ Parse tool_use blocks
    └─→ Send events to AcpThread for UI updates
    ↓
7. For each tool_use:
    ├─→ Deserialize tool input
    ├─→ Look up tool implementation
    ├─→ Create ToolCallEventStream
    ├─→ Call tool.run() asynchronously
    ├─→ Stream tool updates to UI
    ├─→ Wait for authorization if needed
    └─→ Collect tool result
    ↓
8. If tool results exist:
    ├─→ Append to message history
    └─→ GOTO step 5 (continue turn)
    ↓
9. If stop reason is EndTurn:
    ├─→ Save message to database
    ├─→ Update title/summary if needed
    └─→ Notify UI of completion
```

### Authorization Flow

```
1. Tool needs authorization (e.g., EditFileTool with mode=Overwrite)
    ↓
2. Tool calls event_stream.request_authorization()
    ↓
3. ToolCallEventStream sends AuthorizationRequest to AcpThread
    ↓
4. AcpThread emits event to UI
    ↓
5. UI shows modal with options ("Allow", "Deny", "Allow for session")
    ↓
6. User clicks option
    ↓
7. UI sends response back through AcpThread
    ↓
8. Response sent through oneshot channel
    ↓
9. Tool receives PermissionOptionId
    ↓
10. Tool proceeds or aborts based on response
```

### Persistence Flow

```
Thread state changes (new message, edit, etc.)
    ↓
Thread.notify() called
    ↓
NativeAgent observes change
    ↓
Debounced save task triggered
    ↓
Thread.to_db_thread() serializes state
    ↓
HistoryStore.save_thread() writes to SQLite
    ↓
On restart:
    ├─→ HistoryStore.load_threads() reads DB
    ├─→ DbThread.from_db() deserializes
    └─→ Thread reconstructed with full history
```

---

## Implementation Guide

If you want to build your own agentic coding system based on Zed's architecture, here's how:

### Phase 1: Core Infrastructure

**1. Project Abstraction**
```rust
trait Project {
    fn root_directories(&self) -> Vec<PathBuf>;
    fn find_file(&self, relative_path: &str) -> Option<PathBuf>;
    fn open_buffer(&self, path: &Path) -> Result<Buffer>;
    fn filesystem(&self) -> &dyn Fs;
}
```

**2. Buffer Abstraction**
```rust
trait Buffer {
    fn text(&self) -> String;
    fn edit(&mut self, range: Range<usize>, new_text: &str);
    fn save(&mut self) -> Result<()>;
}
```

**3. Async Runtime**
- Use Tokio or async-std
- Wrap in your UI framework's async (if not using GPUI)

### Phase 2: LLM Integration

**1. Model Trait**
```rust
trait LanguageModel {
    async fn complete(
        &self,
        messages: Vec<Message>,
        tools: Vec<Tool>,
    ) -> Result<Stream<CompletionChunk>>;
}
```

**2. Implement Providers**
- Start with one (e.g., Anthropic Claude)
- Use their official SDK
- Handle rate limiting, retries, errors

**3. Tool Calling**
- Parse tool_use blocks from JSON
- Deserialize with serde_json
- Execute and format results

### Phase 3: Tool System

**1. Tool Trait**
```rust
trait Tool {
    type Input: Deserialize;
    type Output;
    
    fn name() -> &'static str;
    fn description() -> &'static str;
    fn schema() -> schemars::Schema;
    
    async fn execute(
        &self,
        input: Self::Input,
        project: &Project,
    ) -> Result<Self::Output>;
}
```

**2. Implement Core Tools**
- Start with: ReadFile, WriteFile, ListDirectory, RunCommand
- Add: Grep, FindFile, Diagnostics
- Each tool needs JSON schema for LLM

**3. Authorization System**
- Async channel for authorization requests
- UI callback when authorization needed
- Store "allow for session" decisions

### Phase 4: Edit Agent

**1. Simple Version First**
- Full file replacement only
- Prompt: "Generate complete file contents"
- No streaming, no fuzzy matching

**2. Add Streaming Edits**
- XML format is simpler than diff format
- Parse as you stream
- Apply edits incrementally

**3. Fuzzy Matching**
- Normalize whitespace
- Use edit distance algorithm
- Provide user feedback on ambiguity

### Phase 5: Conversation Management

**1. Thread State**
```rust
struct Thread {
    id: ThreadId,
    messages: Vec<Message>,
    active_tool_calls: HashMap<ToolCallId, ToolCall>,
    model: Box<dyn LanguageModel>,
}
```

**2. Turn Execution**
```rust
async fn run_turn(&mut self) {
    loop {
        let mut response = self.model.complete(...).await?;
        
        while let Some(chunk) = response.next().await {
            match chunk {
                Text(s) => self.append_text(s),
                ToolUse(tool) => {
                    let result = self.execute_tool(tool).await?;
                    self.append_tool_result(result);
                }
                Stop(reason) => {
                    if reason == StopReason::ToolUse {
                        continue; // Next iteration with tool results
                    }
                    return Ok(());
                }
            }
        }
    }
}
```

**3. Context Building**
- System prompt template
- Convert project structure to text
- Format message history for LLM

### Phase 6: UI Integration

**1. Message View**
- Show user messages
- Show agent messages
- Show tool calls with expandable details

**2. Diff Visualization**
- Use a diff library (e.g., similar, diff)
- Show before/after or unified diff
- Highlight changed lines

**3. Terminal Output**
- Capture stdout/stderr
- ANSI color support
- Scrollable view

### Phase 7: Persistence

**1. Database Schema**
```sql
CREATE TABLE threads (
    id TEXT PRIMARY KEY,
    title TEXT,
    created_at INTEGER,
    updated_at INTEGER
);

CREATE TABLE messages (
    id TEXT PRIMARY KEY,
    thread_id TEXT,
    role TEXT,  -- 'user' or 'assistant'
    content TEXT,  -- JSON
    created_at INTEGER,
    FOREIGN KEY (thread_id) REFERENCES threads(id)
);
```

**2. Serialization**
- Use serde_json for message content
- Compress large messages (zstd)
- Lazy load old messages

### Phase 8: Advanced Features

**1. Multiple Tools in Parallel**
- When LLM requests multiple tool calls
- Execute concurrently where safe
- Aggregate results

**2. Context Servers**
- MCP (Model Context Protocol) support
- External tools from other processes
- JSON-RPC communication

**3. Prompt Caching**
- Claude has prompt caching
- Cache system prompt + project context
- Significantly reduces latency/cost

**4. Streaming UI Updates**
- Show text as it streams
- Show tool calls as they're parsed
- Update tool status in real-time

### Common Pitfalls to Avoid

1. **Don't block the UI thread**
   - All LLM calls must be async
   - Tool execution must be async
   - Use background threads for heavy work

2. **Handle tool errors gracefully**
   - File not found → descriptive error to LLM
   - Permission denied → explain to LLM
   - Let LLM recover or ask user

3. **Rate limit LLM requests**
   - Implement exponential backoff
   - Respect provider limits
   - Handle quota exhaustion

4. **Validate all inputs**
   - Path traversal attacks
   - Command injection
   - Infinite loops in LLM responses

5. **Manage token budgets**
   - Track cumulative token usage
   - Truncate old messages when needed
   - Provide "resume" mechanism

### Testing Strategy

**1. Tool Tests**
```rust
#[test]
async fn test_read_file() {
    let project = TestProject::new();
    project.write_file("test.txt", "hello");
    
    let tool = ReadFileTool::new(&project);
    let result = tool.execute(ReadFileInput {
        path: "test.txt".into(),
        start_line: None,
        end_line: None,
    }).await.unwrap();
    
    assert_eq!(result, "hello");
}
```

**2. Edit Agent Tests**
```rust
#[test]
async fn test_edit_agent() {
    let buffer = Buffer::from_text("fn main() {}");
    let agent = EditAgent::new(test_model());
    
    agent.edit(
        buffer,
        "Add println! statement".into(),
    ).await.unwrap();
    
    assert!(buffer.text().contains("println!"));
}
```

**3. Integration Tests**
```rust
#[test]
async fn test_full_agent_flow() {
    let project = TestProject::new();
    let thread = Thread::new(project.clone());
    
    thread.submit_message("Create a hello world program").await;
    
    // Wait for completion
    thread.wait_for_completion().await;
    
    // Verify file was created
    assert!(project.file_exists("main.rs"));
    assert!(project.read_file("main.rs").contains("fn main"));
}
```

**4. Mock LLM for Tests**
```rust
struct MockLLM {
    responses: VecDeque<String>,
}

impl LanguageModel for MockLLM {
    async fn complete(&self, ...) -> Stream<Chunk> {
        // Return pre-scripted responses
        // Simulate tool calls
    }
}
```

---

## Key Takeaways

1. **Layered Architecture**: Separate concerns cleanly (LLM, tools, UI, persistence)

2. **Tool-First Design**: Agent capabilities are well-defined, testable tools

3. **Streaming Everything**: Text, edits, tool results all stream for responsiveness

4. **Safety by Design**: Authorization, validation, sandboxing at every layer

5. **Async All the Way**: No blocking operations in agent logic

6. **State Management**: Clear separation of conversation state vs. execution state

7. **Error Recovery**: Graceful degradation when things go wrong

8. **Extensibility**: Easy to add new tools, models, UI components

---

## Further Reading

- Zed's source code: https://github.com/zed-industries/zed
- Model Context Protocol: https://modelcontextprotocol.io
- Anthropic Tool Use: https://docs.anthropic.com/claude/docs/tool-use
- OpenAI Function Calling: https://platform.openai.com/docs/guides/function-calling
- GPUI Framework: https://www.gpui.rs

This architecture has evolved over months of real-world usage and represents a production-ready approach to agentic coding. Good luck building your own!
