# Zed Agentic Coding System Diagrams

Visual representations of how Zed's agentic coding system works.

## System Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                              User Interface                             │
│                                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────┐  ┌────────────────┐ │
│  │   Thread    │  │  Tool Call   │  │   Diff   │  │    Terminal    │ │
│  │   View      │  │    View      │  │  Viewer  │  │     View       │ │
│  └─────────────┘  └──────────────┘  └──────────┘  └────────────────┘ │
│                                                                         │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │ Events & UI Updates
                                    │
┌───────────────────────────────────▼─────────────────────────────────────┐
│                            AcpThread Layer                               │
│                                                                          │
│  • Protocol Translation (Core ↔ UI)                                     │
│  • Event Streaming                                                      │
│  • Authorization Management                                             │
│  • Visual State (diffs, terminals, locations)                          │
│                                                                          │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │ Agent Connection Interface
                                    │
┌───────────────────────────────────▼─────────────────────────────────────┐
│                            NativeAgent                                   │
│                                                                          │
│  ┌────────────────┐     ┌──────────────────┐     ┌─────────────────┐  │
│  │   Session      │     │  Language Models │     │     Project     │  │
│  │   Management   │────▶│    Registry      │     │     Context     │  │
│  └────────────────┘     └──────────────────┘     └─────────────────┘  │
│         │                                                               │
│         │ manages                                                       │
│         ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                           Session                                 │  │
│  │  ┌───────────────────┐              ┌───────────────────────┐   │  │
│  │  │      Thread       │              │     AcpThread         │   │  │
│  │  │  (Core Logic)     │◀────────────▶│   (UI Protocol)       │   │  │
│  │  └───────────────────┘              └───────────────────────┘   │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘


┌───────────────────────────────────────────────────────────────────────┐
│                            Thread (Core)                               │
│                                                                        │
│  ┌──────────────┐   ┌─────────────┐   ┌────────────────────────┐    │
│  │   Message    │   │    Tools    │   │   Language Model       │    │
│  │   History    │   │   Registry  │   │                        │    │
│  └──────────────┘   └─────────────┘   └────────────────────────┘    │
│         │                   │                      │                  │
│         │                   │                      │                  │
│         ▼                   ▼                      ▼                  │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                    Turn Execution Engine                        │  │
│  │                                                                 │  │
│  │  • Build LLM Request                                           │  │
│  │  • Stream Response                                             │  │
│  │  • Execute Tools                                               │  │
│  │  • Handle Results                                              │  │
│  │  • Loop until completion                                       │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘


┌───────────────────────────────────────────────────────────────────────┐
│                          Tool System                                   │
│                                                                        │
│  Built-in Tools              Context Tools         External Tools     │
│  ┌──────────────────┐       ┌──────────────┐     ┌───────────────┐  │
│  │ • ReadFile       │       │ • OpenTool   │     │ Context       │  │
│  │ • EditFile       │       │ • Thinking   │     │ Servers       │  │
│  │ • Terminal       │       │ • WebSearch  │     │ (MCP)         │  │
│  │ • Grep           │       │ • Fetch      │     │               │  │
│  │ • FindPath       │       └──────────────┘     └───────────────┘  │
│  │ • ListDirectory  │                                                │
│  │ • Diagnostics    │                                                │
│  │ • DeletePath     │                                                │
│  │ • CopyPath       │                                                │
│  │ • MovePath       │                                                │
│  │ • SaveFile       │                                                │
│  │ • RestoreFile    │                                                │
│  │ • CreateDirectory│                                                │
│  └──────────────────┘                                                │
│                                                                        │
│  All tools implement: AgentTool trait                                 │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ trait AgentTool {                                              │  │
│  │   type Input: JsonSchema;                                      │  │
│  │   type Output;                                                 │  │
│  │   fn name() -> &str;                                           │  │
│  │   fn description() -> &str;                                    │  │
│  │   fn schema() -> Schema;                                       │  │
│  │   fn run(input, event_stream, cx) -> Task<Output>;           │  │
│  │ }                                                              │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

## Request/Response Flow

```
User Input
    │
    ├──▶ "Create a Rust function that parses JSON"
    │
    ▼
┌────────────────────────────────────────────────────┐
│ 1. Thread.run_next_turn()                          │
└────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────┐
│ 2. Build LLM Request                               │
│    • System Prompt (from template)                 │
│    • Message History                               │
│    • Tool Schemas (JSON Schema for each tool)     │
│    • Model Configuration                           │
└────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────┐
│ 3. model.stream_completion()                       │
│    (Send request to LLM provider)                  │
└────────────────────────────────────────────────────┘
    │
    ▼ Stream of events
┌────────────────────────────────────────────────────┐
│ 4. Parse Completion Events                         │
│    ┌──────────────────────────────────┐           │
│    │ CompletionEvent::Text            │           │
│    │   ↓                               │           │
│    │ Append to agent message          │           │
│    │ Update UI                         │           │
│    └──────────────────────────────────┘           │
│    ┌──────────────────────────────────┐           │
│    │ CompletionEvent::ToolUse         │           │
│    │   ↓                               │           │
│    │ Parse tool name & input          │           │
│    │ Queue for execution              │           │
│    └──────────────────────────────────┘           │
│    ┌──────────────────────────────────┐           │
│    │ CompletionEvent::Stop            │           │
│    │   ↓                               │           │
│    │ Check stop reason                │           │
│    └──────────────────────────────────┘           │
└────────────────────────────────────────────────────┘
    │
    ▼ If tools were requested
┌────────────────────────────────────────────────────┐
│ 5. Execute Tools (in parallel where safe)          │
│    ┌──────────────────────────────────┐           │
│    │ For each tool_use:               │           │
│    │   • Lookup tool by name          │           │
│    │   • Deserialize input            │           │
│    │   • Create event stream          │           │
│    │   • Request auth (if needed)     │           │
│    │   • Execute tool.run()           │           │
│    │   • Collect result               │           │
│    └──────────────────────────────────┘           │
└────────────────────────────────────────────────────┘
    │
    ▼ Tool results ready
┌────────────────────────────────────────────────────┐
│ 6. Append Tool Results to Conversation             │
│    Format: ToolResult message                      │
└────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────┐
│ 7. Decision Point                                  │
│    ┌──────────────────────────────────┐           │
│    │ If stop_reason == ToolUse        │           │
│    │   → GOTO Step 2 (continue turn) │           │
│    │                                   │           │
│    │ If stop_reason == EndTurn        │           │
│    │   → Complete & save              │           │
│    │                                   │           │
│    │ If stop_reason == MaxTokens      │           │
│    │   → Offer to resume              │           │
│    └──────────────────────────────────┘           │
└────────────────────────────────────────────────────┘
```

## Edit Agent Flow

```
Primary Agent decides to edit
    │
    ├──▶ tool_use { name: "edit_file", input: { path, mode, description } }
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ EditFileTool.run()                                               │
│                                                                  │
│ 1. Validate path is in project                                  │
│ 2. Check file_scan_exclusions & private_files                   │
│ 3. Request authorization if overwrite mode                      │
│ 4. Read current file (if exists)                                │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ Construct EditAgent Request                                      │
│                                                                  │
│ Template: edit_file_prompt_xml.hbs                              │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ You MUST respond with edits in this format:                │ │
│ │ <edits>                                                    │ │
│ │   <old_text line=10>...</old_text>                        │ │
│ │   <new_text>...</new_text>                                │ │
│ │ </edits>                                                   │ │
│ │                                                            │ │
│ │ <file_to_edit>                                             │ │
│ │ {{path}}                                                   │ │
│ │ </file_to_edit>                                            │ │
│ │                                                            │ │
│ │ <current_file_contents>                                    │ │
│ │ (full file with line numbers)                              │ │
│ │ </current_file_contents>                                   │ │
│ │                                                            │ │
│ │ <edit_description>                                         │ │
│ │ {{display_description}}                                    │ │
│ │ </edit_description>                                        │ │
│ └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ EditAgent.edit() / EditAgent.overwrite()                         │
│                                                                  │
│ • Use specialized edit model (often different from chat model)  │
│ • Stream response                                                │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼ Streaming response
┌─────────────────────────────────────────────────────────────────┐
│ EditParser (State Machine)                                       │
│                                                                  │
│ State transitions:                                               │
│ ┌──────────────────────────────────────────────────────────────┐│
│ │ Idle                                                          ││
│ │   ↓ See "<edits>"                                            ││
│ │ InEdits                                                       ││
│ │   ↓ See "<old_text"                                          ││
│ │ InOldText { line_num, buffer }                               ││
│ │   ↓ Accumulate text until "</old_text>"                     ││
│ │ BetweenTags                                                   ││
│ │   ↓ See "<new_text>"                                         ││
│ │ InNewText { buffer }                                          ││
│ │   ↓ Accumulate text until "</new_text>"                     ││
│ │ ReadyToApply { old_text, new_text, line_num }               ││
│ │   ↓ Emit EditParserEvent::Edit                              ││
│ │ InEdits (ready for next edit)                                ││
│ └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│ Events emitted:                                                  │
│ • EditParserEvent::Edit { old_text, new_text, line_hint }      │
│ • EditParserEvent::Complete                                     │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼ For each EditParserEvent::Edit
┌─────────────────────────────────────────────────────────────────┐
│ StreamingFuzzyMatcher.find_match()                              │
│                                                                  │
│ 1. Normalize whitespace in old_text and buffer                  │
│ 2. Search for exact match                                        │
│ 3. If no exact match, fuzzy search with edit distance          │
│ 4. If line_hint provided, bias search around that line         │
│ 5. Return best match or error                                   │
│                                                                  │
│ Possible outcomes:                                               │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ • Unique match found                                        │ │
│ │   → Return Range<Anchor>                                    │ │
│ │                                                             │ │
│ │ • Multiple matches found                                    │ │
│ │   → Emit AmbiguousEditRange event                          │ │
│ │   → Return error (user must disambiguate)                  │ │
│ │                                                             │ │
│ │ • No match found                                            │ │
│ │   → Emit UnresolvedEditRange event                         │ │
│ │   → Return error                                            │ │
│ └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼ If match found
┌─────────────────────────────────────────────────────────────────┐
│ Apply Edit to Buffer                                             │
│                                                                  │
│ 1. Emit EditAgentOutputEvent::ResolvingEditRange(range)        │
│ 2. buffer.edit(range, new_text)                                 │
│ 3. Emit EditAgentOutputEvent::Edited(range)                    │
│ 4. Update action_log                                             │
│ 5. Update agent_location (cursor position in UI)                │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼ After all edits applied
┌─────────────────────────────────────────────────────────────────┐
│ Post-processing                                                  │
│                                                                  │
│ 1. Format buffer (if format_on_save enabled)                    │
│ 2. Generate diff (old_text vs new_text)                        │
│ 3. Create Diff entity for UI visualization                      │
│ 4. Return result to primary agent                               │
│    • Success: Diff summary                                       │
│    • Failure: Error description                                  │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
Primary Agent receives result
    │
    ├──▶ Continue turn with diff in context
```

## Tool Execution with Authorization

```
Agent requests tool use
    │
    ├──▶ tool_use { name: "edit_file", input: { mode: "overwrite", ... } }
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│ Thread looks up tool by name                             │
└──────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│ Deserialize input JSON → EditFileToolInput               │
└──────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│ Create ToolCallEventStream                                │
│ • Channel for sending updates to UI                       │
└──────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│ Call tool.run(input, event_stream, cx)                   │
│ (Async task begins)                                       │
└──────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│ Tool logic determines authorization needed                │
│                                                           │
│ Example: EditFileTool                                     │
│ ┌──────────────────────────────────────────────────────┐ │
│ │ fn authorize(...) {                                   │ │
│ │   if mode == Overwrite {                             │ │
│ │     event_stream.request_authorization(vec![         │ │
│ │       PermissionOption {                             │ │
│ │         id: "allow",                                 │ │
│ │         label: "Allow Overwrite"                     │ │
│ │       },                                             │ │
│ │       PermissionOption {                             │ │
│ │         id: "deny",                                  │ │
│ │         label: "Deny"                                │ │
│ │       },                                             │ │
│ │       PermissionOption {                             │ │
│ │         id: "allow_session",                         │ │
│ │         label: "Allow for Session"                   │ │
│ │       }                                              │ │
│ │     ])                                               │ │
│ │   }                                                   │ │
│ │ }                                                     │ │
│ └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│ ToolCallEventStream.request_authorization()              │
│                                                           │
│ 1. Create oneshot channel (tx, rx)                       │
│ 2. Send AuthorizationRequest to AcpThread                │
│ 3. Tool task awaits rx.recv()                            │
└──────────────────────────────────────────────────────────┘
    │
    ├──▶ UI shows modal
    │
    │   ┌─────────────────────────────────────────────┐
    │   │  Overwrite file src/main.rs?                │
    │   │                                             │
    │   │  [ Allow Overwrite ]                        │
    │   │  [ Deny ]                                   │
    │   │  [ Allow for Session ]                      │
    │   └─────────────────────────────────────────────┘
    │
    ▼ User clicks option
┌──────────────────────────────────────────────────────────┐
│ UI sends PermissionOptionId back                         │
└──────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│ AcpThread sends response through tx                      │
└──────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│ Tool task receives PermissionOptionId                    │
│                                                           │
│ match option_id {                                         │
│   "allow" | "allow_session" => {                        │
│     // Continue with tool execution                      │
│     if option_id == "allow_session" {                   │
│       // Store in session permissions                    │
│     }                                                     │
│   }                                                       │
│   "deny" => {                                            │
│     return Err("User denied authorization");            │
│   }                                                       │
│ }                                                         │
└──────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│ Tool continues execution                                  │
│ • Perform file operations                                │
│ • Send progress updates via event_stream                 │
│ • Return ToolResult                                       │
└──────────────────────────────────────────────────────────┘
```

## Message Context Formatting

```
User message: "Fix the bug in authentication"
User mentions: @src/auth.rs, @docs/api.md

┌────────────────────────────────────────────────────────────┐
│ UserMessage.to_request() builds:                           │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ LanguageModelRequestMessage {                              │
│   role: User,                                              │
│   content: [                                               │
│     Text("Fix the bug in authentication"),                 │
│     Text("<context>                                        │
│       The following items were attached by the user.       │
│       They are up-to-date and don't need to be re-read.    │
│                                                            │
│       <files>                                              │
│       ```rust src/auth.rs                                  │
│       (full file contents here)                            │
│       ```                                                  │
│       </files>                                             │
│                                                            │
│       <files>                                              │
│       ```md docs/api.md                                    │
│       (full document here)                                 │
│       ```                                                  │
│       </files>                                             │
│     </context>")                                           │
│   ]                                                        │
│ }                                                          │
└────────────────────────────────────────────────────────────┘

Mention types and their formatting:
┌─────────────────────────────────────────────────────────────┐
│ MentionUri::File { abs_path }                               │
│   ↓                                                          │
│ <files>                                                      │
│ ```extension path/to/file.ext                               │
│ (contents)                                                   │
│ ```                                                          │
│ </files>                                                     │
├─────────────────────────────────────────────────────────────┤
│ MentionUri::Directory { abs_path }                          │
│   ↓                                                          │
│ <directories>                                                │
│ Directory: path/to/dir                                       │
│ - file1.txt                                                  │
│ - file2.rs                                                   │
│ - subdir/                                                    │
│ </directories>                                               │
├─────────────────────────────────────────────────────────────┤
│ MentionUri::Symbol { abs_path, line_range, name }          │
│   ↓                                                          │
│ <symbols>                                                    │
│ ```rust path/to/file.rs:10-50                               │
│ (symbol contents)                                            │
│ ```                                                          │
│ </symbols>                                                   │
├─────────────────────────────────────────────────────────────┤
│ MentionUri::Selection { abs_path, line_range }             │
│   ↓                                                          │
│ <selections>                                                 │
│ ```rust path/to/file.rs:25-30                               │
│ (selected lines)                                             │
│ ```                                                          │
│ </selections>                                                │
├─────────────────────────────────────────────────────────────┤
│ MentionUri::Thread { thread_id }                            │
│   ↓                                                          │
│ <threads>                                                    │
│ Previous conversation:                                       │
│ (markdown of previous thread)                                │
│ </threads>                                                   │
├─────────────────────────────────────────────────────────────┤
│ MentionUri::Rule { rule_name }                              │
│   ↓                                                          │
│ <user_rules>                                                 │
│ ```                                                          │
│ (rule contents)                                              │
│ ```                                                          │
│ </user_rules>                                                │
├─────────────────────────────────────────────────────────────┤
│ MentionUri::Fetch { url }                                   │
│   ↓                                                          │
│ <fetched_urls>                                               │
│ Fetch: https://example.com/api/docs                         │
│ (fetched content)                                            │
│ </fetched_urls>                                              │
└─────────────────────────────────────────────────────────────┘
```

## Persistence & Recovery

```
Thread state changes
    │
    ├──▶ New message added, edit applied, tool executed
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ Thread.notify() called                                      │
│ (Triggers observers)                                        │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ NativeAgent observes Thread                                 │
│ (via cx.observe subscription)                               │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ Debounced save task                                         │
│ (Avoid saving on every keystroke)                           │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ Thread.to_db_thread()                                       │
│                                                             │
│ Serializes:                                                 │
│ • Thread ID, title, timestamps                             │
│ • All messages (User & Agent)                              │
│ • Tool results                                              │
│ • Token usage                                               │
│ • Model selection                                           │
│                                                             │
│ Compression:                                                │
│ • Large message content compressed with zstd                │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ HistoryStore.save_thread()                                  │
│                                                             │
│ SQLite schema:                                              │
│ ┌────────────────────────────────────────────────────────┐ │
│ │ threads                                                 │ │
│ │   - id: TEXT PRIMARY KEY                               │ │
│ │   - title: TEXT                                        │ │
│ │   - created_at: INTEGER                                │ │
│ │   - updated_at: INTEGER                                │ │
│ │   - model: TEXT                                        │ │
│ │   - messages: BLOB (zstd compressed JSON)              │ │
│ │   - metadata: TEXT (JSON)                              │ │
│ └────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────┘

On application restart:
┌────────────────────────────────────────────────────────────┐
│ HistoryStore.load_threads()                                 │
│                                                             │
│ 1. Query SQLite for all threads                            │
│ 2. Deserialize DbThread from BLOB                          │
│ 3. Decompress message content                              │
│ 4. Return Vec<DbThread>                                    │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ UI displays thread list                                     │
└────────────────────────────────────────────────────────────┘
    │
    ▼ User selects thread
┌────────────────────────────────────────────────────────────┐
│ DbThread.from_db()                                          │
│                                                             │
│ Reconstructs:                                               │
│ • Full Thread entity                                        │
│ • Message history                                           │
│ • Tool registry (re-register all tools)                    │
│ • Model connection                                          │
│                                                             │
│ User can continue conversation exactly where they left off │
└────────────────────────────────────────────────────────────┘
```

## Model Selection & Registry

```
┌────────────────────────────────────────────────────────────┐
│ LanguageModelRegistry (Global Singleton)                    │
│                                                             │
│ Registered Providers:                                       │
│ ┌────────────────────────────────────────────────────────┐ │
│ │ AnthropicProvider                                       │ │
│ │   - claude-3-5-sonnet-20241022                         │ │
│ │   - claude-3-5-haiku-20241022                          │ │
│ │   - claude-3-opus-20240229                             │ │
│ ├────────────────────────────────────────────────────────┤ │
│ │ OpenAIProvider                                          │ │
│ │   - gpt-4o                                             │ │
│ │   - gpt-4-turbo                                        │ │
│ │   - gpt-3.5-turbo                                      │ │
│ ├────────────────────────────────────────────────────────┤ │
│ │ GoogleProvider                                          │ │
│ │   - gemini-2.0-flash-exp                               │ │
│ │   - gemini-1.5-pro                                     │ │
│ ├────────────────────────────────────────────────────────┤ │
│ │ OllamaProvider (local)                                  │ │
│ │   - (discovered from localhost:11434)                   │ │
│ ├────────────────────────────────────────────────────────┤ │
│ │ ZedCloudProvider                                        │ │
│ │   - (proxied access to various models)                  │ │
│ └────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ NativeAgent.LanguageModels                                  │
│                                                             │
│ On initialization:                                          │
│ 1. Authenticate all providers (background task)            │
│ 2. Gather provided_models() from each                      │
│ 3. Build model_list (grouped by provider)                  │
│ 4. Create models HashMap<ModelId, Arc<dyn LanguageModel>>  │
│                                                             │
│ Model List Structure:                                       │
│ ┌────────────────────────────────────────────────────────┐ │
│ │ AgentModelList::Grouped {                               │ │
│ │   "Recommended": [                                      │ │
│ │     { id: "anthropic/claude-3-5-sonnet", ... },        │ │
│ │     { id: "openai/gpt-4o", ... }                       │ │
│ │   ],                                                    │ │
│ │   "Anthropic": [                                        │ │
│ │     { id: "anthropic/claude-3-5-sonnet", ... },        │ │
│ │     { id: "anthropic/claude-3-opus", ... }             │ │
│ │   ],                                                    │ │
│ │   "OpenAI": [                                           │ │
│ │     { id: "openai/gpt-4o", ... },                      │ │
│ │     { id: "openai/gpt-4-turbo", ... }                  │ │
│ │   ],                                                    │ │
│ │   ...                                                   │ │
│ │ }                                                       │ │
│ └────────────────────────────────────────────────────────┘ │
│                                                             │
│ Watch for changes:                                          │
│ • Provider added/removed                                    │
│ • Provider authenticated/deauthenticated                    │
│ • New models available                                      │
│ • Refresh model_list & notify UI                           │
└────────────────────────────────────────────────────────────┘
    │
    ▼ User selects model in UI
┌────────────────────────────────────────────────────────────┐
│ Thread.set_model(model_id)                                  │
│                                                             │
│ 1. Look up model by ID in LanguageModels registry          │
│ 2. Store Arc<dyn LanguageModel> in thread                  │
│ 3. Use this model for all subsequent completions           │
└────────────────────────────────────────────────────────────┘
```

## Token Management & Context Windows

```
┌────────────────────────────────────────────────────────────┐
│ Before each turn                                            │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ Build full request                                          │
│ • System prompt                                             │
│ • All message history                                       │
│ • Tool schemas                                              │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ Count tokens                                                │
│ model.count_tokens(request)                                 │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ Check against limits                                        │
│                                                             │
│ if token_count > model.max_token_count() {                 │
│   // Context window exceeded                               │
│ }                                                           │
└────────────────────────────────────────────────────────────┘
    │
    ├──▶ Context fits: Send request
    │
    └──▶ Context too large:
         │
         ▼
    ┌────────────────────────────────────────────────────────┐
    │ Truncation Strategy                                     │
    │                                                         │
    │ Keep:                                                   │
    │ • System prompt (always)                               │
    │ • Most recent user message (always)                    │
    │ • Tool schemas (always)                                │
    │                                                         │
    │ Truncate:                                               │
    │ • Older messages in middle of conversation             │
    │ • Keep first few messages (for context)                │
    │ • Keep last N messages (for recency)                   │
    │                                                         │
    │ Algorithm:                                              │
    │ 1. Calculate tokens needed for essentials              │
    │ 2. Calculate remaining budget                          │
    │ 3. Allocate budget to message history                  │
    │ 4. Drop messages from middle if needed                 │
    │ 5. Insert "[Earlier messages truncated]" marker       │
    └────────────────────────────────────────────────────────┘

Token tracking:
┌────────────────────────────────────────────────────────────┐
│ After each completion                                       │
│                                                             │
│ CompletionEvent::UsageDetails { input_tokens, output_tokens }│
│   ↓                                                          │
│ Thread.token_usage += TokenUsage {                          │
│   prompt_tokens: input_tokens,                              │
│   completion_tokens: output_tokens,                         │
│   total_tokens: input_tokens + output_tokens,               │
│ }                                                            │
│   ↓                                                          │
│ Emit TokenUsageUpdated event → UI shows cumulative usage    │
└────────────────────────────────────────────────────────────┘

Prompt caching (Claude):
┌────────────────────────────────────────────────────────────┐
│ LanguageModelRequestMessage {                               │
│   role: System,                                             │
│   content: [Text(system_prompt)],                           │
│   cache: true,  // ← Mark for caching                      │
│ }                                                            │
│                                                             │
│ Benefits:                                                   │
│ • First request: Normal cost                               │
│ • Subsequent requests with same system prompt:             │
│   - Cache hit: ~90% cost reduction for cached portion     │
│   - Only pay for new/changed content                       │
│                                                             │
│ Invalidation:                                               │
│ • Cache expires after ~5 minutes                           │
│ • Any change to cached content invalidates                 │
└────────────────────────────────────────────────────────────┘
```

---

These diagrams should give you a comprehensive visual understanding of how all the pieces fit together in Zed's agentic coding system!
