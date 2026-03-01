# Agent System Domain - Technical Implementation Documentation

## 1. Overview

The Agent System Domain serves as the central orchestration layer for Kimi CLI's AI capabilities. It coordinates LLM interactions, tool execution, conversation management, and implements sophisticated features like context compaction, time-travel debugging, and flow-based skill execution.

**Domain Purpose:** Provide a flexible, extensible framework for defining AI agent behaviors, managing conversation state, orchestrating tool usage, and enabling complex multi-step workflows through a turn-based execution model.

---

## 2. Core Architecture Components

### 2.1 KimiSoul - The Central Orchestrator

**Location:** `src/kimi_cli/soul/kimisoul.py`

KimiSoul implements the main agent execution loop and serves as the primary interface for agent interactions.

#### Key Responsibilities:
- **Turn-based execution** with configurable max steps per turn
- **Retry logic** with exponential jittered backoff for transient errors
- **Approval workflow coordination** via background asyncio tasks
- **Context mutation protection** using `asyncio.shield()`
- **Real-time steering** through synthetic tool call injection
- **Slash command dispatch** for special user commands

#### Technical Implementation:

```python
class KimiSoul:
    def __init__(self, agent: Agent, *, context: Context):
        self._agent = agent
        self._runtime = agent.runtime
        self._context = context
        self._loop_control = agent.runtime.config.loop_control
        self._compaction = SimpleCompaction()
        self._steer_queue: asyncio.Queue[str | list[ContentPart]] = asyncio.Queue()
```

**Retry Strategy:**
- Initial delay: 0.3s
- Maximum delay: 5s
- Jitter: 0.5s
- Retryable errors: `APIConnectionError`, `APITimeoutError`, `APIEmptyResponseError`, HTTP 429/500/502/503

**Turn Execution Flow:**
1. Create checkpoint (restore point)
2. Append user message to context
3. Enter agent loop with max steps limit
4. For each step:
   - Check context size and compact if needed
   - Call LLM via `kosong.step()`
   - Process tool calls with approval workflow
   - Grow context with results
   - Check stop conditions

**Stop Conditions:**
- `no_tool_calls`: LLM response contains no tool requests
- `tool_rejected`: User declined a tool execution

---

### 2.2 Context Management

**Location:** `src/kimi_cli/soul/context.py`

Implements persistent conversation history with checkpoint-based time-travel capabilities.

#### Storage Architecture:

**File Backend:** JSONL format with three entry types:
- **Message entries:** Standard conversation messages
- **`_checkpoint` entries:** Restore point markers with sequential IDs
- **`_usage` entries:** Token count tracking

#### Key Operations:

**Checkpoint Creation:**
```python
async def checkpoint(self, add_user_message: bool):
    checkpoint_id = self._next_checkpoint_id
    self._next_checkpoint_id += 1
    # Write checkpoint marker to file
    # Optionally inject user-visible checkpoint message
```

**Time-Travel Revert:**
```python
async def revert_to(self, checkpoint_id: int):
    # Rotate current file to backup
    # Restore context up to specified checkpoint
    # Rebuild in-memory state
```

**Context Compaction:**
- Triggered when token count + reserved size exceeds LLM context limit
- Uses `SimpleCompaction` strategy by default
- Preserves last N user/assistant message pairs (default: 2)
- Generates summary via LLM call for older messages

---

### 2.3 Tool Management System

**Location:** `src/kimi_cli/soul/toolset.py`

Provides unified interface for three tool types: internal Python tools, MCP server tools, and external API tools.

#### Tool Loading Architecture:

**Three-Tier Loading:**
1. **Internal Tools:** Python modules with dependency injection
2. **MCP Tools:** External servers via JSON-RPC protocol
3. **External Tools:** API-based tools registered at runtime

**Dependency Injection:**
```python
def _load_tool(tool_path: str, dependencies: dict[type[Any], Any]):
    # Inspect tool constructor signature
    # Inject matching dependencies from runtime
    # Instantiate and return tool
```

**MCP Integration:**
- Background loading with status tracking
- OAuth authentication support
- Timeout configuration per server
- Status states: pending → connecting → connected/failed/unauthorized

**Current Tool Call Context:**
```python
current_tool_call = ContextVar[ToolCall | None]("current_tool_call", default=None)

def get_current_tool_call_or_none() -> ToolCall | None:
    """Access current tool call from within tool execution"""
    return current_tool_call.get()
```

---

### 2.4 Approval Workflow System

**Location:** `src/kimi_cli/soul/approval.py`

Implements human-in-the-loop approval for tool executions with session-level auto-approval.

#### Architecture:

**Request Queue Pattern:**
```python
class Approval:
    def __init__(self):
        self._request_queue = Queue[Request]()
        self._requests: dict[str, tuple[Request, asyncio.Future[bool]]] = {}
        self._state = ApprovalState(yolo=False)
```

**Approval States:**
- **YOLO mode:** Auto-approve all tool executions
- **Auto-approve actions:** Set of action names approved for session
- **Per-request approval:** User decides for each tool call

**Response Types:**
- `approve`: Allow this single execution
- `approve_for_session`: Auto-approve this action type for session
- `reject`: Decline execution

**Coordination Flow:**
1. Tool calls `approval.request()` → queues request
2. Soul runs `_pipe_approval_to_wire()` task → sends to UI
3. UI responds via wire protocol
4. Soul calls `resolve_request()` → resolves tool's future

---

### 2.5 Agent Runtime

**Location:** `src/kimi_cli/soul/agent.py`

Manages agent configuration, system prompt rendering, and runtime dependencies.

#### Runtime Components:

**Builtin System Prompt Arguments:**
```python
@dataclass
class BuiltinSystemPromptArgs:
    KIMI_NOW: str              # Current datetime
    KIMI_WORK_DIR: KaosPath    # Working directory
    KIMI_WORK_DIR_LS: str      # Directory listing
    KIMI_AGENTS_MD: str        # AGENTS.md content
    KIMI_SKILLS: str           # Available skills
    KIMI_ADDITIONAL_DIRS_INFO: str  # Additional workspace dirs
```

**Jinja2 Template Rendering:**
- Uses `StrictUndefined` for error detection
- Injects builtin args automatically
- Supports custom args from agent spec

**LaborMarket - Subagent Management:**
- **Fixed subagents:** Isolated LaborMarket, own DenwaRenji
- **Dynamic subagents:** Share parent's LaborMarket

**Runtime Cloning Strategies:**
```python
def copy_for_fixed_subagent(self) -> Runtime:
    # New DenwaRenji, new LaborMarket
    
def copy_for_dynamic_subagent(self) -> Runtime:
    # New DenwaRenji, shared LaborMarket
```

---

### 2.6 Flow Execution System

**Location:** `src/kimi_cli/skill/flow/`

Enables multi-step workflows defined in Mermaid or D2 flowchart syntax.

#### Flow Graph Structure:

**Node Types:**
- `begin`: Entry point (exactly one required)
- `end`: Exit point (exactly one required)
- `task`: Execute agent turn with node label as prompt
- `decision`: Branch based on LLM choice extraction

**Edge Structure:**
```python
@dataclass
class FlowEdge:
    src: str           # Source node ID
    dst: str           # Destination node ID
    label: str | None  # Branch label for decisions
```

**Validation Rules:**
1. Exactly one begin and one end node
2. End node reachable from begin
3. Decision nodes must have labeled edges
4. No duplicate edge labels per node

**Choice Extraction:**
```python
def parse_choice(text: str) -> str | None:
    # Extract from <choice>...</choice> XML tags
    # Returns last match if multiple found
```

**Ralph Loop Generator:**
- Built-in flow for automated iteration
- Configurable max iterations
- Continues until LLM signals completion

---

### 2.7 D-Mail Time-Travel System

**Location:** `src/kimi_cli/soul/denwarenji.py`

Implements checkpoint-based context rewinding via "D-Mail" messages from the future.

#### Mechanism:

**DMail Structure:**
```python
class DMail(BaseModel):
    message: str          # Message to inject at checkpoint
    checkpoint_id: int    # Target checkpoint ID
```

**Constraints:**
- Only one pending D-Mail at a time
- Checkpoint ID must be valid (< current checkpoint count)
- Sent from tools, fetched by soul

**Execution Flow:**
1. Tool sends D-Mail via `DenwaRenji.send_dmail()`
2. Raises `BackToTheFuture` exception
3. Soul catches exception, fetches pending D-Mail
4. Context reverts to target checkpoint
5. D-Mail message injected as user message
6. Agent loop continues from that point

---

### 2.8 Skill Discovery System

**Location:** `src/kimi_cli/skill/__init__.py`

Discovers and loads skills from layered directory structure.

#### Directory Resolution Priority:
1. **Built-in:** `src/kimi_cli/skills/` (when supported by KAOS backend)
2. **User-level:** `~/.config/agents/skills/`, `~/.agents/skills/`, etc.
3. **Project-level:** `<work_dir>/.agents/skills/`, `<work_dir>/.kimi/skills/`, etc.

**Skill Types:**
- `standard`: Regular prompt-based skills
- `flow`: Flowchart-based multi-step workflows

**Frontmatter Parsing:**
```yaml
---
name: skill-name
description: Skill description
type: flow  # or standard
---
```

**Skill Indexing:**
- Case-insensitive name lookup
- Later directories override earlier ones
- Normalized name mapping

---

## 3. Key Interaction Patterns

### 3.1 Turn Execution Sequence

```
User Input
    ↓
Slash Command Check
    ↓ (if normal turn)
Checkpoint Creation
    ↓
Append User Message
    ↓
Agent Loop Entry
    ↓
┌─────────────────────────┐
│  Step Loop (max steps)  │
│                         │
│  1. Check context size  │
│  2. Compact if needed   │
│  3. Call LLM            │
│  4. Process tool calls  │
│  5. Grow context        │
│  6. Check stop reason   │
└─────────────────────────┘
    ↓
Turn Complete
```

### 3.2 Approval Coordination

```
Tool Execution
    ↓
approval.request()
    ↓
Queue Request
    ↓
Soul: _pipe_approval_to_wire()
    ↓
Wire: ApprovalRequest
    ↓
UI: Display Dialog
    ↓
User Decision
    ↓
Wire: ApprovalResponse
    ↓
Soul: resolve_request()
    ↓
Tool: Future Resolved
```

### 3.3 Context Compaction Flow

```
Token Count Check
    ↓
Threshold Exceeded?
    ↓ (yes)
Wire: CompactionBegin
    ↓
Prepare Messages
    ↓
Call LLM for Summary
    ↓
Clear Context
    ↓
Create Checkpoint
    ↓
Append Compacted Messages
    ↓
Wire: CompactionEnd
```

---

## 4. Configuration and Extensibility

### 4.1 Agent Specification

**Location:** `src/kimi_cli/agentspec.py`

**Agent Spec Structure:**
```yaml
extend: path/to/base/agent.yaml  # Optional inheritance
name: agent-name
system_prompt_path: prompts/system.md
system_prompt_args:
  CUSTOM_ARG: value
tools:
  - kimi_cli.tools.file:ReadFile
  - kimi_cli.tools.shell:Shell
exclude_tools:
  - tool_to_exclude
subagents:
  researcher:
    path: agents/researcher/agent.yaml
    description: Research specialist
```

**Inheritance Mechanism:**
- Recursive resolution of `extend` chain
- Child specs override parent values
- `Inherit` marker for explicit inheritance

### 4.2 Loop Control Configuration

**Max Steps Per Turn:** Configurable limit to prevent infinite loops
**Reserved Context Size:** Buffer for tool results and system messages
**Max Ralph Iterations:** Limit for automated iteration flows

### 4.3 Tool Extension Points

**Custom Tool Registration:**
```python
toolset.register_external_tool(
    name="custom_tool",
    description="Tool description",
    parameters={"type": "object", "properties": {...}}
)
```

**MCP Server Integration:**
```python
await toolset.load_mcp_tools(
    mcp_configs=[...],
    runtime=runtime
)
```

---

## 5. Error Handling and Recovery

### 5.1 Retry Strategy

**Retryable Errors:**
- Network connection failures
- API timeouts
- Empty responses
- Rate limiting (429)
- Server errors (500, 502, 503)

**Recovery Mechanism:**
```python
@retry(
    retry=retry_if_exception(is_retryable_error),
    stop=stop_after_attempt(max_retries),
    wait=wait_exponential_jitter(initial=0.3, max=5.0, jitter=0.5)
)
async def _step(...):
    # LLM call with automatic retry
```

### 5.2 Exception Types

- `LLMNotSet`: No LLM configured for agent
- `LLMNotSupported`: LLM lacks required capabilities
- `MaxStepsReached`: Exceeded max steps per turn
- `BackToTheFuture`: D-Mail time-travel trigger
- `ToolRejectedError`: User declined tool execution

---

## 6. Performance Considerations

### 6.1 Context Management

**Token Estimation:**
- Conservative heuristic: ~4 chars per token
- Replaced by actual usage on next LLM call
- Separate tracking for summary vs preserved messages

**File I/O Optimization:**
- Async file operations via `aiofiles`
- Append-only writes for performance
- File rotation for revert operations

### 6.2 Concurrent Operations

**Approval Piping:**
- Background asyncio task
- Structured cancellation on step completion
- Queue-based decoupling

**MCP Loading:**
- Background loading with status tracking
- Non-blocking tool discovery
- Toast notifications for status updates

---

## 7. Integration Points

### 7.1 Wire Protocol Communication

**Event Types:**
- `TurnBegin`: Start of user turn
- `TurnEnd`: Completion of turn
- `StepBegin`: Start of LLM step
- `StepInterrupted`: Step cancelled
- `ApprovalRequest`: Tool approval needed
- `ApprovalResponse`: User decision
- `CompactionBegin/End`: Context compaction events
- `StatusUpdate`: Token usage, context usage

### 7.2 LLM Provider Integration

**Via Kosong Engine:**
```python
result = await kosong.step(
    chat_provider=llm.chat_provider,
    system_prompt=system_prompt,
    toolset=toolset,
    history=context.history
)
```

### 7.3 Session State Persistence

**Approval State:**
- YOLO mode flag
- Auto-approve actions set
- Persisted to session state on change

**Additional Directories:**
- Shared list reference across agents
- Pruned on runtime creation
- Saved to session state

---

## 8. Best Practices

### 8.1 Agent Development

1. **Use inheritance** for common agent configurations
2. **Validate system prompts** with Jinja2 StrictUndefined
3. **Limit tool sets** to necessary capabilities
4. **Test with max_steps_per_turn** to prevent infinite loops

### 8.2 Tool Development

1. **Request approval** for destructive operations
2. **Use current_tool_call context** for metadata access
3. **Handle ToolRejectedError** gracefully
4. **Provide clear descriptions** for approval dialogs

### 8.3 Flow Design

1. **Validate flows** before deployment
2. **Use clear edge labels** for decision nodes
3. **Limit flow depth** to prevent excessive turns
4. **Extract choices** with XML tags for reliability

---

## 9. Summary

The Agent System Domain provides a sophisticated orchestration layer that enables:

- **Flexible agent behaviors** through configurable specifications
- **Robust execution** with retry logic and error recovery
- **Human oversight** via approval workflows
- **Context management** with time-travel capabilities
- **Extensibility** through tools, skills, and flows
- **Performance optimization** via compaction and async operations

This architecture supports both simple conversational agents and complex multi-step workflows while maintaining clear separation of concerns and extensibility points for future enhancements.