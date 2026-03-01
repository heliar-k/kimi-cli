# Tool Execution Domain - Technical Documentation

## 1. Overview

The Tool Execution Domain is a comprehensive, extensible framework that enables AI agents to interact with external systems through a standardized interface. Built on the `kosong.tooling` framework, it provides a plugin architecture with strong typing, approval workflows, and comprehensive error handling.

**Core Purpose:** Enable AI agents to perform concrete actions in the development environment while maintaining security, user control, and type safety.

## 2. Architecture

### 2.1 Core Framework Components

The domain is structured in three layers:

```
┌─────────────────────────────────────────┐
│     Tool Definition Layer               │
│  (Tool, CallableTool, CallableTool2)    │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│     Execution Layer                     │
│  (SimpleToolset, Approval, Wire)        │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│     Result Layer                        │
│  (ToolReturnValue, DisplayBlock)        │
└─────────────────────────────────────────┘
```

### 2.2 Key Classes

#### Tool Definition (`packages/kosong/src/kosong/tooling/__init__.py`)

**Tool** - Base definition for LLM consumption:
```python
class Tool(BaseModel):
    name: str                    # Unique identifier
    description: str             # Purpose and usage
    parameters: ParametersType   # JSON Schema format
```

**CallableTool2[Params]** - Modern typed tool interface:
```python
class CallableTool2[Params: BaseModel](ABC):
    name: str
    description: str
    params: type[Params]  # Pydantic model for validation
    
    async def __call__(self, params: Params) -> ToolReturnValue:
        ...
```

**Key Features:**
- Automatic JSON Schema generation from Pydantic models
- Built-in parameter validation
- Type-safe execution
- Custom schema generator removes unnecessary titles

#### Execution Engine (`packages/kosong/src/kosong/tooling/simple.py`)

**SimpleToolset** - Concurrent tool execution manager:
```python
class SimpleToolset:
    def handle(self, tool_call: ToolCall) -> HandleResult:
        # Returns Future[ToolResult] for async execution
        return asyncio.create_task(_call())
```

**Features:**
- Concurrent execution via `asyncio.create_task()`
- JSON argument parsing with error handling
- Tool lookup by name
- Exception wrapping in ToolRuntimeError

#### Result Types

**ToolReturnValue** - Base result class:
```python
class ToolReturnValue(BaseModel):
    is_error: bool                      # Success/failure flag
    output: str | list[ContentPart]     # For LLM consumption
    message: str                        # Explanatory message
    display: list[DisplayBlock]         # For UI rendering
    extras: dict[str, JsonType] | None  # Debug/testing data
```

**Convenience Constructors:**
- `ToolOk(output, message, brief)` - Success result
- `ToolError(message, brief, output)` - Error result

## 3. Tool Categories

### 3.1 File Operations (`src/kimi_cli/tools/file/`)

**ReadFile** - Read file content with chunking:
```python
class ReadFile(CallableTool2[Params]):
    # Limits: MAX_LINES=1000, MAX_BYTES=100KB, MAX_LINE_LENGTH=2000
    # Features:
    # - Line-numbered output (cat -n style)
    # - Workspace boundary validation
    # - Media file detection
    # - Automatic truncation with warnings
```

**WriteFile** - Write/append with approval:
```python
class WriteFile(CallableTool2[Params]):
    # Modes: overwrite, append
    # Features:
    # - Diff preview before execution
    # - Path validation (absolute required for outside workspace)
    # - Approval workflow integration
    # - Parent directory existence check
```

**Security Features:**
- `is_within_workspace()` validation prevents directory traversal
- Canonical path resolution via `KaosPath.canonical()`
- Absolute path requirement for outside-workspace access

### 3.2 Shell Execution (`src/kimi_cli/tools/shell/`)

**Shell** - Safe command execution:
```python
class Shell(CallableTool2[Params]):
    # Platform support: bash (Unix), PowerShell (Windows)
    # Timeout: 1-300 seconds (MAX_TIMEOUT = 5 minutes)
    # Features:
    # - Real-time stdout/stderr streaming
    # - Clean environment via get_clean_env()
    # - Approval with command preview
    # - Async subprocess management
```

**Execution Flow:**
1. User approval with `ShellDisplayBlock` preview
2. Spawn subprocess via `kaos.exec()`
3. Stream output through callbacks
4. Enforce timeout with `asyncio.wait_for()`
5. Return exit code and captured output

### 3.3 Web Tools (`src/kimi_cli/tools/web/`)

**SearchWeb** - Moonshot API integration:
```python
class SearchWeb(CallableTool2[Params]):
    # Parameters:
    # - query: Search text
    # - limit: 1-20 results (default 5)
    # - include_content: Fetch full page content
    
    # Features:
    # - OAuth credential resolution
    # - Custom headers support
    # - Result parsing with validation
    # - Graceful service unavailability handling
```

**FetchURL** - Content retrieval with fallback strategy

### 3.4 Multi-Agent Orchestration (`src/kimi_cli/tools/multiagent/`)

**Task** - Subagent delegation:
```python
class Task(CallableTool2[Params]):
    # Features:
    # - Context isolation (separate history files)
    # - Wire event bridging (SubagentEvent wrapping)
    # - Continuation prompts for brief responses
    # - Max steps protection
    # - Labor market integration
```

**Subagent Execution:**
1. Generate unique context file via `next_available_rotation()`
2. Create isolated `Context` and `KimiSoul`
3. Bridge wire events to parent (approval requests stay at root)
4. Run with `run_soul()` and optional continuation
5. Return final response or error

## 4. Approval System (`src/kimi_cli/soul/approval.py`)

### 4.1 Architecture

```python
class Approval:
    def __init__(self, yolo: bool = False, *, state: ApprovalState | None = None):
        self._request_queue = Queue[Request]()
        self._requests: dict[str, tuple[Request, asyncio.Future[bool]]] = {}
        self._state = state or ApprovalState(yolo=yolo)
```

### 4.2 Workflow

**Request Flow:**
1. Tool calls `approval.request(sender, action, description, display)`
2. System checks YOLO mode or auto-approve list
3. If approval needed, creates `Request` with UUID
4. Queues request and returns `Future[bool]`
5. Tool awaits approval decision

**Resolution:**
- `approve` - Single approval
- `approve_for_session` - Auto-approve this action type
- `reject` - Decline execution

**State Sharing:**
- `approval.share()` creates new queue with shared state
- Enables consistent auto-approve across tool instances

## 5. Display System (`src/kimi_cli/tools/display.py`)

### 5.1 Display Block Types

**DiffDisplayBlock** - File change visualization:
```python
class DiffDisplayBlock(DisplayBlock):
    type: str = "diff"
    path: str
    old_text: str
    new_text: str
```

**ShellDisplayBlock** - Command preview:
```python
class ShellDisplayBlock(DisplayBlock):
    type: str = "shell"
    language: str  # "bash" or "powershell"
    command: str
```

**TodoDisplayBlock** - Task list updates:
```python
class TodoDisplayBlock(DisplayBlock):
    type: str = "todo"
    items: list[TodoDisplayItem]
```

### 5.2 Registry Pattern

Display blocks use a class-level registry for polymorphic deserialization:
```python
__display_block_registry: ClassVar[dict[str, type["DisplayBlock"]]] = {}
```

Unknown types fall back to `UnknownDisplayBlock` with raw data preservation.

## 6. Error Handling (`packages/kosong/src/kosong/tooling/error.py`)

### 6.1 Error Hierarchy

```
ToolError (ToolReturnValue with is_error=True)
├── ToolNotFoundError      # Tool name not in toolset
├── ToolParseError         # Invalid JSON arguments
├── ToolValidateError      # Schema validation failure
├── ToolRuntimeError       # Execution exception
└── ToolRejectedError      # User declined approval
```

### 6.2 Error Handling Strategy

**Tools MUST NOT raise exceptions** (except `asyncio.CancelledError`):
```python
try:
    result = await tool.call(arguments)
    return ToolResult(tool_call_id=tool_call.id, return_value=result)
except Exception as e:
    return ToolResult(tool_call_id=tool_call.id, return_value=ToolRuntimeError(str(e)))
```

## 7. Utility Components

### 7.1 ToolResultBuilder (`src/kimi_cli/tools/utils.py`)

**Purpose:** Manage output with automatic truncation:
```python
class ToolResultBuilder:
    def __init__(self, max_chars=50_000, max_line_length=2000):
        # Features:
        # - Character limit enforcement
        # - Line-by-line truncation
        # - Display block accumulation
        # - Extras dictionary for metadata
```

**Usage Pattern:**
```python
builder = ToolResultBuilder()
builder.write("output line\n")
builder.display(DiffDisplayBlock(...))
builder.extras(file_size=1024)
return builder.ok("Success message", brief="Done")
```

### 7.2 Path Validation

**Workspace Boundary Check:**
```python
def is_within_workspace(
    path: Path,
    work_dir: Path,
    additional_dirs: list[Path]
) -> bool:
    # Prevents directory traversal attacks
    # Requires absolute paths for outside access
```

### 7.3 Content Truncation

**Line Truncation:**
```python
def truncate_line(line: str, max_length: int, marker: str = "...") -> str:
    # Preserves line breaks
    # Adds marker at truncation point
    # Ensures output doesn't exceed max_length
```

## 8. Integration Points

### 8.1 LLM Provider Integration

Tools are exposed to LLMs via the `Tool` base class:
```python
toolset = SimpleToolset([ReadFile(...), WriteFile(...), Shell(...)])
tools_for_llm = toolset.tools  # list[Tool]
```

### 8.2 Wire Protocol

Tool execution events are communicated via wire messages:
- `ToolCallRequest` - Request approval
- `ApprovalRequest/ApprovalResponse` - Approval workflow
- `SubagentEvent` - Nested agent execution

### 8.3 Runtime Context

Tools receive runtime dependencies via constructor injection:
```python
class ReadFile(CallableTool2[Params]):
    def __init__(self, runtime: Runtime):
        self._work_dir = runtime.builtin_args.KIMI_WORK_DIR
        self._additional_dirs = runtime.additional_dirs
```

## 9. Performance Optimizations

### 9.1 Async-First Design

- All tool execution is async (`async def __call__`)
- Concurrent execution via `asyncio.create_task()`
- Streaming output for long-running commands

### 9.2 Resource Management

- **Ripgrep binary management** - Automatic download/installation
- **Result truncation** - 50K chars, 2000 chars/line limits
- **Lazy loading** - `SkipThisTool` exception for unavailable tools
- **Path rotation** - Prevents context file conflicts

### 9.3 Timeout Protection

Shell commands enforce maximum 5-minute timeout:
```python
await asyncio.wait_for(
    asyncio.gather(
        _read_stream(process.stdout, stdout_cb),
        _read_stream(process.stderr, stderr_cb),
    ),
    timeout,
)
```

## 10. Security Considerations

### 10.1 Path Security

- Canonical path resolution prevents symlink attacks
- Workspace boundary validation
- Absolute path requirement for outside access
- Parent directory existence checks

### 10.2 Approval Workflow

- User confirmation for destructive operations
- Session-level auto-approve for repeated actions
- YOLO mode for trusted environments
- Action-based approval granularity

### 10.3 Environment Isolation

- Clean environment via `get_clean_env()`
- Subprocess isolation
- Timeout enforcement
- Output sanitization (UTF-8 with error replacement)

## 11. Extension Guide

### 11.1 Creating Custom Tools

```python
from pydantic import BaseModel, Field
from kosong.tooling import CallableTool2, ToolReturnValue

class MyToolParams(BaseModel):
    arg1: str = Field(description="First argument")
    arg2: int = Field(description="Second argument", ge=0)

class MyTool(CallableTool2[MyToolParams]):
    name: str = "MyTool"
    description: str = "Tool description for LLM"
    params: type[MyToolParams] = MyToolParams
    
    async def __call__(self, params: MyToolParams) -> ToolReturnValue:
        # Implementation
        return ToolOk(output="result", message="Success")
```

### 11.2 Adding to Toolset

```python
toolset = SimpleToolset()
toolset += MyTool()
# or
toolset.add(MyTool())
```

### 11.3 Custom Display Blocks

```python
class MyDisplayBlock(DisplayBlock):
    type: str = "my_custom_type"
    custom_field: str
    
# Automatically registered in __display_block_registry
```

## 12. Best Practices

1. **Always use ToolResultBuilder** for output management
2. **Validate paths** before file operations
3. **Request approval** for destructive actions
4. **Handle exceptions** gracefully (return ToolError)
5. **Provide clear descriptions** for LLM understanding
6. **Use display blocks** for rich UI feedback
7. **Set reasonable timeouts** for long operations
8. **Truncate output** to prevent token overflow
9. **Test with invalid inputs** to ensure robustness
10. **Document tool capabilities** in description field

---

**Document Version:** 1.0  
**Last Updated:** 2026-03-01  
**Confidence Score:** 9.2/10