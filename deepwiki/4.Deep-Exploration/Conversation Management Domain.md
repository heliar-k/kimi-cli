Based on my comprehensive analysis of the codebase, I'll now write detailed technical documentation for the Conversation Management Domain.

# Conversation Management Domain - Technical Documentation

## 1. Domain Overview

### 1.1 Purpose and Scope

The Conversation Management Domain is the central orchestration layer responsible for managing the complete lifecycle of AI conversation sessions in Kimi CLI. It provides persistent, resumable conversation contexts that survive application restarts, supports concurrent multi-session workflows, and enables advanced features like session forking and history replay.

**Core Responsibilities:**
- Session lifecycle management (create, read, update, delete, archive)
- Real-time bidirectional communication via WebSocket streaming
- Conversation history persistence using event sourcing patterns
- Session forking for conversation branching
- Work directory management and file isolation
- Auto-archival of inactive sessions
- Process orchestration for multi-session concurrency

**Business Value:**
- **Context Preservation**: Users can pause and resume conversations without losing context
- **Multi-tasking**: Support for multiple concurrent AI conversations across different projects
- **Conversation Branching**: Fork sessions at any point to explore alternative approaches
- **Auditability**: Complete conversation history with wire protocol event logs
- **Performance**: TTL-based caching reduces disk I/O for frequently accessed sessions

### 1.2 Architecture Position

```
┌─────────────────────────────────────────────────────────────┐
│                    Presentation Layer                        │
│  ┌──────────────────┐              ┌──────────────────┐    │
│  │  Web Interface   │              │  CLI Interface   │    │
│  │   (React/TS)     │              │    (Python)      │    │
│  └────────┬─────────┘              └────────┬─────────┘    │
└───────────┼──────────────────────────────────┼──────────────┘
            │                                  │
            │ REST API + WebSocket             │ Direct
            │                                  │
┌───────────▼──────────────────────────────────▼──────────────┐
│          Conversation Management Domain (Core)               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   Session    │  │   History    │  │   Session    │     │
│  │  Lifecycle   │  │  Management  │  │  Streaming   │     │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │
└─────────┼──────────────────┼──────────────────┼─────────────┘
          │                  │                  │
          │ Coordinates      │ Persists         │ Streams
          │                  │                  │
┌─────────▼──────────────────▼──────────────────▼─────────────┐
│              Supporting Domains                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │     Agent    │  │     LLM      │  │     Tool     │     │
│  │    System    │  │  Providers   │  │  Execution   │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└──────────────────────────────────────────────────────────────┘
```

## 2. Sub-Module Architecture

### 2.1 Session Lifecycle Management

**Purpose**: Manages the complete lifecycle of conversation sessions from creation through deletion, including metadata management, work directory coordination, and state transitions.

#### 2.1.1 Core Components

**Backend Components:**

```python
# src/kimi_cli/web/store/sessions.py
class SessionMetadata(BaseModel):
    """Session metadata stored in metadata.json"""
    session_id: str
    title: str = "Untitled"
    title_generated: bool = False
    title_generate_attempts: int = 0
    wire_mtime: float | None = None
    archived: bool = False
    archived_at: float | None = None
    auto_archive_exempt: bool = False

class JointSession(Session):
    """Combined session model with web UI and kimi-cli session data"""
    kimi_cli_session: KimiCLISession = Field(exclude=True)

class SessionIndexEntry:
    """In-memory index entry for fast session lookups"""
    session_id: UUID
    session_dir: Path
    context_file: Path
    work_dir: str
    work_dir_meta: WorkDirMeta
    last_updated: datetime
    title: str
    metadata: SessionMetadata | None
```

**Frontend Components:**

```typescript
// web/src/hooks/useSessions.ts
interface SessionsState {
  sessions: Session[];
  archivedSessions: Session[];
  selectedSessionId: string | null;
  loading: boolean;
  error: string | null;
}

// web/src/lib/api/models/Session.ts
interface Session {
  session_id: string;
  title: string;
  last_updated: string;
  is_running: boolean;
  status: SessionStatus | null;
  work_dir: string;
  session_dir: string;
  archived: boolean;
}
```

#### 2.1.2 Session Creation Flow

**API Endpoint**: `POST /api/sessions/`

```python
# src/kimi_cli/web/api/sessions.py
@router.post("/", summary="Create a new session")
async def create_session(
    request: CreateSessionRequest,
    runner: KimiCLIRunner = Depends(get_runner),
) -> Session:
    """Create a new session with specified work directory."""
    work_dir = Path(request.work_dir).expanduser().resolve()
    
    # Validate work directory
    if not work_dir.exists():
        work_dir.mkdir(parents=True, exist_ok=True)
    
    # Create session using KimiCLISession
    kimi_session = await KimiCLISession.create(work_dir=work_dir)
    session_id = UUID(kimi_session.id)
    
    # Initialize metadata
    metadata = SessionMetadata(
        session_id=str(session_id),
        title="Untitled",
    )
    save_session_metadata(kimi_session.dir, metadata)
    
    # Invalidate cache to reflect new session
    invalidate_sessions_cache()
    
    return load_session_by_id(session_id)
```

**Directory Structure Created:**

```
~/.kimi-cli/sessions/{session_id}/
├── metadata.json          # Session metadata
├── wire.jsonl            # Wire protocol events (JSON-RPC)
├── context.jsonl         # Conversation history for LLM
├── state.json            # Session state (agent config, etc.)
└── uploads/              # User-uploaded files
```

#### 2.1.3 Session Listing with Caching

**Cache Strategy**: Cache-aside pattern with 5-second TTL

```python
# src/kimi_cli/web/store/sessions.py
CACHE_TTL = 5.0  # seconds
_sessions_cache: list[JointSession] | None = None
_cache_timestamp: float = 0.0

def load_sessions_page(
    limit: int = 100,
    offset: int = 0,
    query: str | None = None,
    archived: bool | None = None,
) -> list[JointSession]:
    """Load sessions with caching and filtering."""
    global _sessions_cache, _cache_timestamp
    
    now = time.time()
    
    # Check cache validity
    if _sessions_cache is None or (now - _cache_timestamp) > CACHE_TTL:
        # Cache miss or expired - rebuild from disk
        entries = _build_sessions_index()
        _sessions_cache = [_build_joint_session(e) for e in entries]
        _cache_timestamp = now
    
    # Filter by archived status
    filtered = [
        s for s in _sessions_cache
        if archived is None and not s.archived
        or archived is True and s.archived
        or archived is False and not s.archived
    ]
    
    # Apply search query
    if query:
        query_lower = query.lower()
        filtered = [
            s for s in filtered
            if query_lower in s.title.lower()
            or query_lower in s.work_dir.lower()
        ]
    
    # Apply pagination
    return filtered[offset:offset + limit]
```

**Cache Invalidation Strategy:**

```python
def invalidate_sessions_cache() -> None:
    """Clear cache after mutations (create/update/delete)."""
    global _sessions_cache, _cache_timestamp
    _sessions_cache = None
    _cache_timestamp = 0.0
```

#### 2.1.4 Auto-Archive Mechanism

**Configuration:**

```python
AUTO_ARCHIVE_DAYS = 15  # Sessions older than 15 days
AUTO_ARCHIVE_INTERVAL = 300.0  # Run at most once per 5 minutes
```

**Implementation:**

```python
# src/kimi_cli/web/store/sessions.py
_last_auto_archive_time: float = 0.0

def run_auto_archive() -> int:
    """Auto-archive old sessions (throttled execution)."""
    global _last_auto_archive_time
    
    now = time.time()
    if now - _last_auto_archive_time < AUTO_ARCHIVE_INTERVAL:
        return 0  # Skip if ran recently
    
    _last_auto_archive_time = now
    archived_count = 0
    
    entries = _build_sessions_index()
    
    for entry in entries:
        if entry.metadata and _should_auto_archive(
            entry.last_updated, 
            entry.metadata
        ):
            # Update metadata to archived
            updated_metadata = entry.metadata.model_copy(
                update={
                    "archived": True,
                    "archived_at": time.time(),
                }
            )
            save_session_metadata(entry.session_dir, updated_metadata)
            archived_count += 1
    
    if archived_count > 0:
        invalidate_sessions_cache()
    
    return archived_count

def _should_auto_archive(
    last_updated: datetime, 
    metadata: SessionMetadata
) -> bool:
    """Check if session should be auto-archived."""
    if metadata.archived:
        return False  # Already archived
    
    if metadata.auto_archive_exempt:
        return False  # User manually unarchived
    
    now = datetime.now(tz=UTC)
    age_days = (now - last_updated).days
    return age_days >= AUTO_ARCHIVE_DAYS
```

**Trigger Points:**
- On session list API call: `await asyncio.to_thread(run_auto_archive)`
- Throttled to run at most once per 5 minutes
- Non-blocking background execution

#### 2.1.5 Title Derivation

**Strategy**: Extract title from first user message in wire.jsonl

```python
def _derive_title_from_wire(session_dir: Path) -> str:
    """Derive session title from first TurnBegin event."""
    wire_file = session_dir / "wire.jsonl"
    if not wire_file.exists():
        return "Untitled"
    
    with open(wire_file, encoding="utf-8") as f:
        for line in f:
            record = json.loads(line.strip())
            message = record.get("message", {})
            
            if message.get("type") == "TurnBegin":
                user_input = message.get("payload", {}).get("user_input")
                if user_input:
                    msg = Message(role="user", content=user_input)
                    text = msg.extract_text(" ")
                    return shorten(text, width=300)
    
    return "Untitled"
```

**Title Update Logic:**

```python
def _ensure_title(entry: SessionIndexEntry, *, refresh: bool) -> None:
    """Ensure session has a title, updating metadata if needed."""
    wire_file = entry.session_dir / "wire.jsonl"
    wire_mtime = wire_file.stat().st_mtime if wire_file.exists() else None
    
    metadata = entry.metadata or load_session_metadata(
        entry.session_dir, 
        str(entry.session_id)
    )
    
    # Case 1: Title exists and is not "Untitled"
    if metadata.title and metadata.title != "Untitled":
        entry.title = metadata.title
        # Update wire_mtime if changed
        if metadata.wire_mtime != wire_mtime:
            metadata = metadata.model_copy(update={"wire_mtime": wire_mtime})
            save_session_metadata(entry.session_dir, metadata)
        return
    
    # Case 2: Title is empty or "Untitled" - derive from wire
    if refresh:
        title = _derive_title_from_wire(entry.session_dir)
        entry.title = title
        metadata = metadata.model_copy(
            update={"title": title, "wire_mtime": wire_mtime}
        )
        save_session_metadata(entry.session_dir, metadata)
```

### 2.2 History Management

**Purpose**: Handles conversation history persistence, replay, and forking capabilities using event sourcing patterns with wire protocol.

#### 2.2.1 Wire Protocol Architecture

**File Format**: JSON Lines (JSONL) with metadata header

```python
# src/kimi_cli/wire/file.py
class WireFileMetadata(BaseModel):
    """First line in wire.jsonl"""
    type: Literal["metadata"] = "metadata"
    protocol_version: str  # Current: "v1.3"

class WireMessageRecord(BaseModel):
    """Each subsequent line in wire.jsonl"""
    timestamp: float
    message: WireMessageEnvelope
```

**Example wire.jsonl:**

```jsonl
{"type":"metadata","protocol_version":"v1.3"}
{"timestamp":1709280954.123,"message":{"type":"TurnBegin","payload":{"user_input":"Hello"}}}
{"timestamp":1709280954.456,"message":{"type":"StepBegin","payload":{"step":1}}}
{"timestamp":1709280954.789,"message":{"type":"ContentPart","payload":{"text":"Hi there!"}}}
{"timestamp":1709280955.012,"message":{"type":"TurnEnd","payload":{}}}
```

#### 2.2.2 Wire File Operations

**Append Message:**

```python
# src/kimi_cli/wire/file.py
class WireFile:
    async def append_message(
        self, 
        msg: WireMessage, 
        *, 
        timestamp: float | None = None
    ) -> None:
        """Append a wire message to the file."""
        record = WireMessageRecord.from_wire_message(
            msg,
            timestamp=time.time() if timestamp is None else timestamp,
        )
        await self.append_record(record)
    
    async def append_record(self, record: WireMessageRecord) -> None:
        """Append a record to wire.jsonl."""
        self.path.parent.mkdir(parents=True, exist_ok=True)
        needs_header = not self.path.exists() or self.path.stat().st_size == 0
        
        async with aiofiles.open(self.path, mode="a", encoding="utf-8") as f:
            if needs_header:
                metadata = WireFileMetadata(
                    protocol_version=self.protocol_version
                )
                await f.write(_dump_line(metadata))
            await f.write(_dump_line(record))
```

**Iterate Records:**

```python
async def iter_records(self) -> AsyncIterator[WireMessageRecord]:
    """Iterate over all wire message records."""
    if not self.path.exists():
        return
    
    async with aiofiles.open(self.path, encoding="utf-8") as f:
        async for line in f:
            line = line.strip()
            if not line:
                continue
            
            parsed = parse_wire_file_line(line)
            if isinstance(parsed, WireFileMetadata):
                continue  # Skip metadata header
            
            yield parsed
```

#### 2.2.3 History Replay Mechanism

**Purpose**: When a WebSocket reconnects, replay all historical events to restore UI state.

**Implementation:**

```python
# src/kimi_cli/web/api/sessions.py
async def replay_history(ws: WebSocket, session_dir: Path) -> None:
    """Replay historical wire messages to a WebSocket."""
    wire_file = session_dir / "wire.jsonl"
    if not await asyncio.to_thread(wire_file.exists):
        return
    
    try:
        lines = await asyncio.to_thread(_read_wire_lines, wire_file)
        for event_text in lines:
            await ws.send_text(event_text)
    except Exception:
        pass  # Best-effort replay

def _read_wire_lines(wire_file: Path) -> list[str]:
    """Read and parse wire.jsonl into JSON-RPC event strings."""
    result: list[str] = []
    
    with open(wire_file, encoding="utf-8") as f:
        for line in f:
            line = line.strip()
            if not line:
                continue
            
            record = json.loads(line)
            if record.get("type") == "metadata":
                continue  # Skip metadata header
            
            message_raw = record.get("message")
            message = deserialize_wire_message(message_raw)
            is_req = is_request(message)
            
            # Convert to JSON-RPC format
            event_msg = {
                "jsonrpc": "2.0",
                "method": "request" if is_req else "event",
                "params": message_raw,
            }
            
            if is_req:
                event_msg["id"] = message.id
            
            result.append(json.dumps(event_msg, ensure_ascii=False))
    
    return result
```

**WebSocket Connection Flow:**

```
Client Connects
      ↓
Accept WebSocket
      ↓
Replay History (wire.jsonl)
      ↓
Send history_complete
      ↓
Start Worker Process
      ↓
Forward Live Events
```

#### 2.2.4 Session Forking

**Purpose**: Create a new session with conversation history up to a specific turn, enabling users to branch conversations and explore alternative approaches.

**API Endpoint**: `POST /api/sessions/{session_id}/fork`

```python
# src/kimi_cli/web/api/sessions.py
@router.post("/{session_id}/fork", summary="Fork a session at a specific turn")
async def fork_session(
    session_id: UUID,
    request: ForkSessionRequest,
    runner: KimiCLIRunner = Depends(get_runner),
) -> Session:
    """Fork a session, creating a new session with history up to specified turn."""
    source_session = get_editable_session(session_id, runner)
    source_dir = source_session.kimi_cli_session.dir
    wire_path = source_dir / "wire.jsonl"
    context_path = source_dir / "context.jsonl"
    
    # Truncate wire and context at specified turn
    truncated_wire_lines = truncate_wire_at_turn(wire_path, request.turn_index)
    truncated_context_lines = truncate_context_at_turn(
        context_path, 
        request.turn_index
    )
    
    # Create new session with same work_dir
    work_dir = source_session.kimi_cli_session.work_dir
    new_session = await KimiCLISession.create(work_dir=work_dir)
    new_session_dir = new_session.dir
    
    # Copy referenced video files
    source_uploads = source_dir / "uploads"
    if source_uploads.is_dir():
        referenced_videos = _extract_video_references(truncated_wire_lines)
        if referenced_videos:
            new_uploads = new_session_dir / "uploads"
            new_uploads.mkdir(exist_ok=True)
            for video_name in referenced_videos:
                source_file = source_uploads / video_name
                if source_file.is_file():
                    shutil.copy2(source_file, new_uploads / video_name)
    
    # Write truncated files
    (new_session_dir / "wire.jsonl").write_text(
        "\n".join(truncated_wire_lines) + "\n",
        encoding="utf-8",
    )
    (new_session_dir / "context.jsonl").write_text(
        "\n".join(truncated_context_lines) + "\n",
        encoding="utf-8",
    )
    
    # Create metadata with "Fork: " prefix
    metadata = SessionMetadata(
        session_id=new_session.id,
        title=f"Fork: {source_session.title}",
    )
    save_session_metadata(new_session_dir, metadata)
    
    invalidate_sessions_cache()
    return load_session_by_id(UUID(new_session.id))
```

**Wire Truncation Logic:**

```python
def truncate_wire_at_turn(wire_path: Path, turn_index: int) -> list[str]:
    """Return all wire lines up to and including the specified turn."""
    lines: list[str] = []
    current_turn = -1  # Will become 0 on first TurnBegin
    
    with open(wire_path, encoding="utf-8") as f:
        for line in f:
            stripped = line.strip()
            if not stripped:
                continue
            
            record = json.loads(stripped)
            
            # Always keep metadata header
            if record.get("type") == "metadata":
                lines.append(stripped)
                continue
            
            message = record.get("message", {})
            msg_type = message.get("type")
            
            if msg_type == "TurnBegin":
                current_turn += 1
                if current_turn > turn_index:
                    break  # Stop after target turn
            
            if current_turn <= turn_index:
                lines.append(stripped)
            
            # Stop after TurnEnd of target turn
            if msg_type == "TurnEnd" and current_turn == turn_index:
                break
    
    if current_turn < turn_index:
        raise ValueError(
            f"turn_index {turn_index} out of range (max turn: {current_turn})"
        )
    
    return lines
```

**Context Truncation Logic:**

```python
def truncate_context_at_turn(context_path: Path, turn_index: int) -> list[str]:
    """Return all context lines up to and including the specified turn.
    
    Turn detection based on real user messages, excluding synthetic 
    checkpoint entries like <system>CHECKPOINT N</system>.
    """
    lines: list[str] = []
    current_turn = -1
    
    with open(context_path, encoding="utf-8") as f:
        for line in f:
            stripped = line.strip()
            if not stripped:
                continue
            
            record = json.loads(stripped)
            
            # Count real user messages (not checkpoints)
            if (record.get("role") == "user" and 
                not _is_checkpoint_user_message(record)):
                current_turn += 1
                if current_turn > turn_index:
                    break
            
            if current_turn <= turn_index:
                lines.append(stripped)
    
    return lines
```

### 2.3 Session Streaming

**Purpose**: Implements real-time bidirectional communication between frontend and backend for live conversation updates using WebSocket and JSON-RPC protocol.

#### 2.3.1 WebSocket Architecture

**Connection Endpoint**: `ws://localhost:8899/api/sessions/{session_id}/stream`

**Protocol**: JSON-RPC 2.0 over WebSocket

```typescript
// web/src/hooks/useSessionStream.ts
interface WireMessage {
  jsonrpc: "2.0";
  method: "event" | "request";
  params: WireEvent;
  id?: string | number;  // Present for requests
}
```

#### 2.3.2 Backend WebSocket Handler

```python
# src/kimi_cli/web/api/sessions.py
@router.websocket("/{session_id}/stream")
async def session_stream(
    ws: WebSocket,
    session_id: UUID,
    runner: KimiCLIRunner = Depends(get_runner_ws),
) -> None:
    """WebSocket endpoint for real-time session streaming."""
    
    # Validate authentication
    token = ws.query_params.get("token")
    if not verify_token(token):
        await ws.close(code=status.WS_1008_POLICY_VIOLATION)
        return
    
    # Validate origin
    origin = ws.headers.get("origin")
    if not is_origin_allowed(origin):
        await ws.close(code=status.WS_1008_POLICY_VIOLATION)
        return
    
    # Load session
    session = load_session_by_id(session_id)
    if session is None:
        await ws.close(code=status.WS_1008_POLICY_VIOLATION)
        return
    
    await ws.accept()
    
    try:
        # Step 1: Replay history
        await replay_history(ws, session.kimi_cli_session.dir)
        await send_history_complete(ws)
        
        # Step 2: Get or create session process
        session_process = await runner.get_or_create_session(session_id)
        
        # Step 3: Send current status snapshot
        await session_process.send_status_snapshot(ws)
        
        # Step 4: Attach WebSocket for live updates
        await session_process.attach_websocket(ws)
        
        # Step 5: Message loop
        while True:
            data = await ws.receive_text()
            
            # Parse JSON-RPC message
            try:
                msg = JSONRPCInMessageAdapter.validate_json(data)
            except Exception:
                error_response = JSONRPCErrorResponse(
                    error=JSONRPCErrorObject(
                        code=ErrorCodes.PARSE_ERROR,
                        message="Invalid JSON-RPC message",
                    ),
                    id=None,
                )
                await ws.send_text(error_response.model_dump_json())
                continue
            
            # Handle message
            if isinstance(msg, JSONRPCPromptMessage):
                await session_process.send_prompt(msg, ws)
            elif isinstance(msg, JSONRPCCancelMessage):
                await session_process.send_cancel(msg)
            # ... handle other message types
    
    except WebSocketDisconnect:
        pass
    finally:
        await session_process.detach_websocket(ws)
```

#### 2.3.3 Session Process Management

**Purpose**: Manages subprocess lifecycle, WebSocket fanout, and message routing for each session.

```python
# src/kimi_cli/web/runner/process.py
class SessionProcess:
    """Manages a single session's KimiCLI subprocess.
    
    Concurrency model:
    - is_alive/is_running: worker subprocess exists and hasn't exited
    - is_busy: at least one in-flight prompt ID
    - WebSocket fanout supports "join while running"
    """
    
    def __init__(self, session_id: UUID) -> None:
        self.session_id = session_id
        self._in_flight_prompt_ids: set[str] = set()
        self._process: asyncio.subprocess.Process | None = None
        self._websockets: set[WebSocket] = set()
        self._replay_buffers: dict[WebSocket, list[str]] = {}
        self._read_task: asyncio.Task[None] | None = None
        self._lock = asyncio.Lock()
        self._ws_lock = asyncio.Lock()
    
    @property
    def is_busy(self) -> bool:
        """Whether session is currently processing a prompt."""
        return len(self._in_flight_prompt_ids) > 0
    
    async def start(self, session_dir: Path) -> None:
        """Start the worker subprocess."""
        async with self._lock:
            if self._process is not None:
                return  # Already running
            
            # Start subprocess
            self._process = await asyncio.create_subprocess_exec(
                sys.executable,
                "-m",
                "kimi_cli.web.runner.worker",
                str(session_dir),
                stdin=asyncio.subprocess.PIPE,
                stdout=asyncio.subprocess.PIPE,
                stderr=asyncio.subprocess.PIPE,
                env=get_clean_env(),
                limit=16 * 1024 * 1024,  # 16MB buffer
            )
            
            # Start read loop
            self._read_task = asyncio.create_task(self._read_loop())
    
    async def _read_loop(self) -> None:
        """Read messages from worker stdout and broadcast to WebSockets."""
        assert self._process and self._process.stdout
        
        while True:
            line = await self._process.stdout.readline()
            if not line:
                break  # EOF
            
            try:
                msg = JSONRPCOutMessageAdapter.validate_json(line)
                await self._broadcast_message(msg)
            except Exception:
                continue
    
    async def _broadcast_message(self, msg: JSONRPCOutMessage) -> None:
        """Broadcast message to all connected WebSockets."""
        async with self._ws_lock:
            msg_text = msg.model_dump_json()
            
            # Send to all connected WebSockets
            for ws in list(self._websockets):
                try:
                    # If WebSocket is replaying, buffer the message
                    if ws in self._replay_buffers:
                        self._replay_buffers[ws].append(msg_text)
                    else:
                        await ws.send_text(msg_text)
                except Exception:
                    # Remove disconnected WebSocket
                    self._websockets.discard(ws)
    
    async def attach_websocket(self, ws: WebSocket) -> None:
        """Attach a WebSocket for message broadcasting."""
        async with self._ws_lock:
            self._websockets.add(ws)
            # Create replay buffer for this WebSocket
            self._replay_buffers[ws] = []
    
    async def flush_replay_buffer(self, ws: WebSocket) -> None:
        """Flush buffered messages after history replay completes."""
        async with self._ws_lock:
            if ws in self._replay_buffers:
                buffer = self._replay_buffers.pop(ws)
                for msg_text in buffer:
                    await ws.send_text(msg_text)
    
    async def send_prompt(
        self, 
        msg: JSONRPCPromptMessage, 
        ws: WebSocket
    ) -> None:
        """Send a prompt to the worker subprocess."""
        async with self._lock:
            if self.is_busy:
                # Reject if already processing
                error = JSONRPCErrorResponse(
                    error=JSONRPCErrorObject(
                        code=ErrorCodes.INVALID_REQUEST,
                        message="Session is busy",
                    ),
                    id=msg.id,
                )
                await ws.send_text(error.model_dump_json())
                return
            
            # Track prompt ID
            prompt_id = str(msg.id)
            self._in_flight_prompt_ids.add(prompt_id)
            
            # Send to worker stdin
            assert self._process and self._process.stdin
            self._process.stdin.write(
                msg.model_dump_json().encode("utf-8") + b"\n"
            )
            await self._process.stdin.drain()
```

#### 2.3.4 Frontend WebSocket Hook

**Purpose**: Manages WebSocket connection lifecycle and transforms JSON-RPC events into React state.

```typescript
// web/src/hooks/useSessionStream.ts
export function useSessionStream(sessionId: string | null) {