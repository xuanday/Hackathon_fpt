# RepoIntel Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a working local repository-intelligence MCP plugin that indexes a Python codebase, builds a symbol/dependency graph, persists repository memory and Git history, and gives GitHub Copilot a compact task-specific context pack.

**Architecture:** RepoIntel uses one local SQLite database as the source of truth. Repository files are indexed into chunks and FTS5, Python AST produces symbols and graph edges, Git provides historical evidence, and persistent memories capture reusable repository knowledge. A retrieval engine combines all four sources and exposes them through four MCP tools.

**Tech Stack:** Python 3.12, `uv`, official MCP Python SDK v2, SQLite/FTS5, Python `ast`, Pydantic, standard-library Git subprocesses, pytest. Optional P1: `sqlite-vec`.

**Spec:** `docs/superpowers/specs/2026-09-07-repointel-design.md`

**Plan target:** `docs/superpowers/plans/2026-09-07-repointel-mvp.md`

## Global Constraints

- Hard implementation window: **120 minutes**.
- Team size: **5 developers**.
- Primary demo client: **GitHub Copilot**.
- Claude Code and Codex compatibility are demonstrated only if setup costs <10 minutes.
- Runtime must be local-first.
- Persistent state lives under `.repointel/`.
- MVP supports **Python repositories only**.
- SQLite is the only mandatory persistence dependency.
- FTS5 is the mandatory retrieval baseline.
- Vector search must never block the critical path.
- Graph accuracy may be heuristic; demo usefulness matters more than static-analysis completeness.
- No web dashboard.
- No authentication.
- No cloud database.
- No universal parser.
- No autonomous code editing inside RepoIntel.
- RepoIntel provides intelligence; Copilot remains the coding agent.
- Freeze new features at **minute 90**.

---

# 1. Definition of Done

At minute 120, this exact flow must work:

```text
Python repository
       │
       ▼
repointel init
       │
       ├── files indexed
       ├── chunks indexed
       ├── symbols extracted
       ├── graph built
       ├── git history captured
       └── local memory persisted
               │
               ▼
         RepoIntel MCP
               │
               ▼
        GitHub Copilot
               │
               ▼
get_task_context(
  "Implement refresh-token rotation"
)
               │
               ▼
compact context pack
               │
               ▼
Copilot edits code
               │
               ▼
tests pass
```

The demo must expose four measurable results:

```text
Repository tokens / chars
Retrieved context size
Relevant files/symbols
Final tests passing
```

---

# 2. Repository Layout

```text
repointel/
├── pyproject.toml
├── README.md
├── src/
│   └── repointel/
│       ├── __init__.py
│       ├── cli.py
│       ├── config.py
│       ├── models.py
│       │
│       ├── storage/
│       │   ├── __init__.py
│       │   ├── db.py
│       │   └── schema.sql
│       │
│       ├── indexer/
│       │   ├── __init__.py
│       │   ├── scanner.py
│       │   ├── chunker.py
│       │   └── service.py
│       │
│       ├── graph/
│       │   ├── __init__.py
│       │   ├── parser.py
│       │   ├── resolver.py
│       │   └── service.py
│       │
│       ├── gitintel/
│       │   ├── __init__.py
│       │   └── service.py
│       │
│       ├── memory/
│       │   ├── __init__.py
│       │   └── service.py
│       │
│       ├── retrieval/
│       │   ├── __init__.py
│       │   ├── ranker.py
│       │   └── context.py
│       │
│       └── mcp/
│           ├── __init__.py
│           └── server.py
│
├── tests/
│   ├── fixtures/
│   │   └── demo_repo/
│   ├── test_storage.py
│   ├── test_indexer.py
│   ├── test_graph.py
│   ├── test_gitintel.py
│   ├── test_memory.py
│   ├── test_retrieval.py
│   └── test_mcp.py
│
└── benchmarks/
    ├── tasks.json
    └── run_benchmark.py
```

## Ownership rule

Only **Member 1** edits:

```text
models.py
storage/*
cli.py
mcp/server.py
pyproject.toml
```

Other members consume those interfaces.

This prevents merge conflicts during the two-hour sprint.

---

# 3. Persistent Storage Schema

All state lives at:

```text
.repointel/repointel.db
```

## `schema.sql`

```sql
PRAGMA foreign_keys = ON;
PRAGMA journal_mode = WAL;

CREATE TABLE IF NOT EXISTS meta (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS files (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    path TEXT NOT NULL UNIQUE,
    language TEXT NOT NULL,
    sha256 TEXT NOT NULL,
    size_bytes INTEGER NOT NULL,
    is_test INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE IF NOT EXISTS chunks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    file_id INTEGER NOT NULL,
    symbol_id INTEGER,
    start_line INTEGER NOT NULL,
    end_line INTEGER NOT NULL,
    content TEXT NOT NULL,
    content_hash TEXT NOT NULL,
    FOREIGN KEY(file_id) REFERENCES files(id) ON DELETE CASCADE
);

CREATE VIRTUAL TABLE IF NOT EXISTS chunks_fts USING fts5(
    chunk_id UNINDEXED,
    path,
    symbol,
    content,
    tokenize='porter unicode61'
);

CREATE TABLE IF NOT EXISTS symbols (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    file_id INTEGER NOT NULL,
    qualified_name TEXT NOT NULL,
    name TEXT NOT NULL,
    kind TEXT NOT NULL,
    start_line INTEGER NOT NULL,
    end_line INTEGER NOT NULL,
    signature TEXT,
    docstring TEXT,
    UNIQUE(file_id, qualified_name, start_line),
    FOREIGN KEY(file_id) REFERENCES files(id) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_symbols_name
ON symbols(name);

CREATE INDEX IF NOT EXISTS idx_symbols_qualified
ON symbols(qualified_name);

CREATE TABLE IF NOT EXISTS edges (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    source_type TEXT NOT NULL,
    source_id INTEGER NOT NULL,
    target_type TEXT NOT NULL,
    target_id INTEGER NOT NULL,
    relation TEXT NOT NULL,
    metadata_json TEXT,
    UNIQUE(
        source_type,
        source_id,
        target_type,
        target_id,
        relation
    )
);

CREATE INDEX IF NOT EXISTS idx_edges_source
ON edges(source_type, source_id);

CREATE INDEX IF NOT EXISTS idx_edges_target
ON edges(target_type, target_id);

CREATE TABLE IF NOT EXISTS commits (
    sha TEXT PRIMARY KEY,
    author_time INTEGER NOT NULL,
    message TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS file_commits (
    file_id INTEGER NOT NULL,
    commit_sha TEXT NOT NULL,
    PRIMARY KEY(file_id, commit_sha),
    FOREIGN KEY(file_id) REFERENCES files(id) ON DELETE CASCADE,
    FOREIGN KEY(commit_sha) REFERENCES commits(sha) ON DELETE CASCADE
);

CREATE TABLE IF NOT EXISTS memories (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    kind TEXT NOT NULL,
    content TEXT NOT NULL,
    source_file_id INTEGER,
    source_start_line INTEGER,
    source_end_line INTEGER,
    source_hash TEXT,
    created_commit TEXT,
    confidence REAL NOT NULL DEFAULT 1.0,
    valid INTEGER NOT NULL DEFAULT 1,
    created_at INTEGER NOT NULL,
    FOREIGN KEY(source_file_id) REFERENCES files(id)
);

CREATE VIRTUAL TABLE IF NOT EXISTS memories_fts USING fts5(
    memory_id UNINDEXED,
    content,
    tokenize='porter unicode61'
);

CREATE TABLE IF NOT EXISTS task_runs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    task TEXT NOT NULL,
    repo_commit TEXT,
    context_json TEXT NOT NULL,
    context_chars INTEGER NOT NULL,
    created_at INTEGER NOT NULL
);
```

---

# 4. Shared Domain Models

`src/repointel/models.py`

```python
from __future__ import annotations

from typing import Literal
from pydantic import BaseModel, Field


NodeType = Literal["file", "symbol"]
RelationType = Literal[
    "imports",
    "defines",
    "calls",
    "tests",
    "references",
]


class FileRecord(BaseModel):
    id: int | None = None
    path: str
    language: str = "python"
    sha256: str
    size_bytes: int
    is_test: bool = False


class SymbolRecord(BaseModel):
    id: int | None = None
    file_id: int
    qualified_name: str
    name: str
    kind: Literal["class", "function", "method"]
    start_line: int
    end_line: int
    signature: str | None = None
    docstring: str | None = None


class EdgeRecord(BaseModel):
    source_type: NodeType
    source_id: int
    target_type: NodeType
    target_id: int
    relation: RelationType
    metadata: dict[str, str] = Field(default_factory=dict)


class ChunkRecord(BaseModel):
    id: int | None = None
    file_id: int
    path: str
    symbol_id: int | None = None
    symbol: str | None = None
    start_line: int
    end_line: int
    content: str
    content_hash: str


class Evidence(BaseModel):
    kind: Literal["file", "git", "memory"]
    ref: str
    detail: str | None = None


class SearchHit(BaseModel):
    chunk_id: int
    file_id: int
    path: str
    symbol_id: int | None
    symbol: str | None
    start_line: int
    end_line: int
    content: str
    score: float


class ContextItem(BaseModel):
    kind: Literal["code", "symbol", "memory", "history"]
    title: str
    content: str
    score: float
    evidence: list[Evidence]


class GraphLink(BaseModel):
    source: str
    relation: str
    target: str


class TaskContextStats(BaseModel):
    repository_files: int
    repository_chars: int
    context_chars: int
    reduction_percent: float


class TaskContextResponse(BaseModel):
    task: str
    repo_commit: str | None
    items: list[ContextItem]
    relationships: list[GraphLink]
    stats: TaskContextStats


class SymbolExplanation(BaseModel):
    symbol: str
    location: str
    summary: str
    callers: list[str]
    callees: list[str]
    related_files: list[str]
    history: list[str]


class ChangeImpact(BaseModel):
    target: str
    direct_dependents: list[str]
    indirect_dependents: list[str]
    likely_tests: list[str]


class HistoryItem(BaseModel):
    sha: str
    message: str
    files: list[str]


class HistoryResponse(BaseModel):
    query: str
    results: list[HistoryItem]
```

---

# 5. Storage API Contract

All members code against one API owned by Member 1.

`src/repointel/storage/db.py`

```python
class RepoStore:
    def __init__(self, db_path: Path) -> None: ...

    def initialize(self) -> None: ...

    def clear_index(self) -> None: ...

    def set_meta(self, key: str, value: str) -> None: ...

    def get_meta(self, key: str) -> str | None: ...

    def upsert_file(self, record: FileRecord) -> int: ...

    def get_file_by_path(self, path: str) -> FileRecord | None: ...

    def list_files(self) -> list[FileRecord]: ...

    def replace_chunks(
        self,
        file_id: int,
        chunks: list[ChunkRecord],
    ) -> None: ...

    def search_chunks(
        self,
        query: str,
        limit: int = 20,
    ) -> list[SearchHit]: ...

    def replace_symbols(
        self,
        file_id: int,
        symbols: list[SymbolRecord],
    ) -> list[SymbolRecord]: ...

    def find_symbols(
        self,
        query: str,
        limit: int = 20,
    ) -> list[SymbolRecord]: ...

    def get_symbol(
        self,
        qualified_name: str,
    ) -> SymbolRecord | None: ...

    def replace_edges_from_file(
        self,
        file_id: int,
        edges: list[EdgeRecord],
    ) -> None: ...

    def neighbors(
        self,
        node_type: NodeType,
        node_id: int,
        relation: str | None = None,
    ) -> list[tuple[EdgeRecord, str]]: ...

    def upsert_commit(
        self,
        sha: str,
        author_time: int,
        message: str,
    ) -> None: ...

    def link_file_commit(
        self,
        file_id: int,
        commit_sha: str,
    ) -> None: ...

    def add_memory(
        self,
        content: str,
        kind: str,
        source_file_id: int | None,
        source_start_line: int | None,
        source_end_line: int | None,
        source_hash: str | None,
        created_commit: str | None,
        confidence: float = 1.0,
    ) -> int: ...

    def search_memories(
        self,
        query: str,
        limit: int = 5,
    ) -> list[ContextItem]: ...

    def record_task_run(
        self,
        task: str,
        repo_commit: str | None,
        context_json: str,
        context_chars: int,
    ) -> None: ...
```

No member changes these method names after minute 15.

---

# 6. Indexing Architecture

## File scanning

Support:

```text
*.py
README.md
ARCHITECTURE.md
CONTRIBUTING.md
```

Ignore:

```text
.git/
.venv/
venv/
__pycache__/
dist/
build/
node_modules/
.coverage/
.pytest_cache/
```

## Chunking

Python source:

```text
Prefer symbol-level chunks.
Fallback to 80-line chunks for module-level code.
```

Markdown:

```text
Split by headings.
```

Target:

```text
~500–2,500 characters per chunk
```

---

# 7. Graph Model

P0 graph relations:

```text
file --defines--> symbol
file --imports--> file
symbol --calls--> symbol
```

P1:

```text
symbol --tested_by--> symbol/file
```

Python parser uses stdlib `ast`.

Example:

```python
class SymbolVisitor(ast.NodeVisitor):
    def __init__(self) -> None:
        self.scope: list[str] = []
        self.symbols: list[ParsedSymbol] = []
        self.calls: list[ParsedCall] = []

    def visit_ClassDef(self, node: ast.ClassDef) -> None:
        ...

    def visit_FunctionDef(self, node: ast.FunctionDef) -> None:
        ...

    def visit_AsyncFunctionDef(
        self,
        node: ast.AsyncFunctionDef,
    ) -> None:
        ...

    def visit_Call(self, node: ast.Call) -> None:
        ...
```

Call resolution may be heuristic:

```text
TokenService.rotate()
       ↓
search known symbol:
name == "rotate"

Prefer:
same module
imported module
same class
unique repository symbol
```

Perfect resolution is explicitly not required.

---

# 8. Local Memory Model

RepoIntel uses two memory types in the MVP.

## Fact memory

Persistent repository conventions:

```text
"Token lifecycle logic belongs in TokenService."
"Integration tests live under tests/integration."
```

Evidence:

```text
source file
line range
source hash
commit
```

## Episodic memory

Every successful `get_task_context()` call is persisted in `task_runs`.

This means RepoIntel gradually remembers:

```text
task
→ files
→ symbols
→ relationships
```

A future similar task may boost previously useful context.

Memory validation:

```text
if source_hash != current_file_hash:
    memory.valid = false
```

P0 requires storing memory.

P1 adds automatic invalidation during `sync`.

---

# 9. Git Archaeology API

`src/repointel/gitintel/service.py`

```python
class GitService:
    def __init__(self, repo_root: Path) -> None: ...

    def head_sha(self) -> str | None: ...

    def recent_commits(
        self,
        limit: int = 50,
    ) -> list[HistoryItem]: ...

    def history_for_file(
        self,
        path: str,
        limit: int = 10,
    ) -> list[HistoryItem]: ...

    def history_for_symbol(
        self,
        symbol: str,
        limit: int = 10,
    ) -> list[HistoryItem]: ...

    def search_history(
        self,
        query: str,
        limit: int = 5,
    ) -> list[HistoryItem]: ...
```

Commands:

```bash
git rev-parse HEAD
git log --format=...
git log -- <path>
git log -S "<symbol>" --all
```

No Git library dependency required.

---

# 10. Retrieval Algorithm

P0 candidate generation:

```text
Task
 │
 ├─ FTS5 chunks        top 20
 ├─ symbol match       top 10
 ├─ memory search      top 5
 └─ graph expansion    depth 1
          │
          ▼
      rerank
          │
          ▼
  context budget
```

## P0 scoring

```python
final_score = (
    0.55 * lexical_score
    + 0.20 * symbol_score
    + 0.15 * graph_score
    + 0.10 * memory_score
)
```

Where absent signals contribute `0`.

Exact ranking quality is less important than deterministic useful results.

## Graph expansion

For top five initial symbols/files:

```text
IMPORTS neighbor      +0.12
CALLS neighbor        +0.15
DEFINES relationship  +0.08
test file             +0.15
```

Depth:

```text
1 hop P0
2 hops only for get_change_impact
```

## Context budget

Default:

```text
max_chars = 8,000
```

Approximate displayed tokens:

```python
estimated_tokens = round(chars / 4)
```

No tokenizer dependency required.

---

# 11. MCP API

`src/repointel/mcp/server.py`

```python
from mcp.server import MCPServer

mcp = MCPServer("RepoIntel")
```

## Tool 1 — `get_task_context`

```python
@mcp.tool()
def get_task_context(
    task: str,
    max_chars: int = 8000,
) -> TaskContextResponse:
    """Return evidence-backed repository context for a coding task."""
```

Agent use:

```text
Before editing unfamiliar code, call this once with the full task.
```

---

## Tool 2 — `explain_symbol`

```python
@mcp.tool()
def explain_symbol(
    symbol: str,
) -> SymbolExplanation:
    """Explain a repository symbol and its structural context."""
```

Output includes:

```text
location
callers
callees
related files
history
```

---

## Tool 3 — `get_change_impact`

```python
@mcp.tool()
def get_change_impact(
    target: str,
    depth: int = 2,
) -> ChangeImpact:
    """Return likely blast radius for changing a symbol or file."""
```

`depth` is clamped:

```text
1 <= depth <= 2
```

---

## Tool 4 — `search_history`

```python
@mcp.tool()
def search_history(
    query: str,
    limit: int = 5,
) -> HistoryResponse:
    """Find Git history relevant to a repository decision or symbol."""
```

Maximum:

```text
limit <= 10
```

---

# 12. CLI API

```bash
repointel init [PATH]
repointel sync [PATH]
repointel mcp [PATH]
repointel stats [PATH]
```

## `init`

```text
full repository scan
→ DB
→ chunks
→ symbols
→ graph
→ git metadata
```

Expected:

```text
RepoIntel initialized

Files:       481
Chunks:      923
Symbols:   3,842
Edges:     6,912
Memories:     12
Head:       abc1234
```

## `sync`

P1:

```text
git diff / file hashes
→ changed files
→ re-index only changed files
```

If incremental sync is not ready:

```text
sync may safely rerun init
```

That is acceptable for hackathon MVP.

---

# 13. Five-Member Ownership

| Member | Branch | Owns | Critical output |
|---|---|---|---|
| **M1 — Integration/MCP** | `feat/core-mcp` | shared models, DB, CLI, MCP | Copilot can call server |
| **M2 — Index** | `feat/index` | scanner, chunker, FTS | natural-language task finds code |
| **M3 — Graph** | `feat/graph` | AST, symbols, edges | symbol/dependency graph |
| **M4 — Git/Memory** | `feat/git-memory` | archaeology + persistent memories | evidence/why |
| **M5 — Retrieval/Demo** | `feat/retrieval-demo` | ranking, context pack, benchmark/demo repo | killer result |

No one adds features outside ownership before minute 90.

---

# 14. Parallel Timeline

```text
00:00
│
├─ M1 contracts + DB + MCP stub
├─ M2 scanner/index
├─ M3 AST/graph
├─ M4 git/memory
└─ M5 golden demo + retrieval skeleton

00:15
│
└─ SHARED CONTRACT FREEZE

00:20
│
└─ Copilot successfully calls dummy MCP tool

00:50
│
├─ index works
├─ graph works
├─ git works
└─ retrieval golden test works

01:00
│
└─ first real get_task_context()

01:15
│
└─ Copilot connected to real RepoIntel

01:30
│
└─ feature freeze

01:30–01:45
│
└─ benchmark + fix only

01:45–02:00
│
└─ demo rehearsal / pitch / backup
```

---

# 15. Task 1 — Foundation, Contracts and MCP Stub

**Owner:** Member 1  
**Deadline:** minute 20

**Files:**
- Create: `pyproject.toml`
- Create: `src/repointel/models.py`
- Create: `src/repointel/storage/schema.sql`
- Create: `src/repointel/storage/db.py`
- Create: `src/repointel/mcp/server.py`
- Create: `src/repointel/cli.py`
- Test: `tests/test_storage.py`
- Test: `tests/test_mcp.py`

**Produces:**

```python
RepoStore
TaskContextResponse
SymbolExplanation
ChangeImpact
HistoryResponse
```

and running:

```bash
uv run repointel mcp .
```

### Steps

- [ ] **Step 1: Bootstrap**

```bash
uv init --python 3.12
uv add "mcp[cli]>=2,<3" pydantic
uv add --dev pytest
```

- [ ] **Step 2: Add package script**

```toml
[project.scripts]
repointel = "repointel.cli:main"
```

- [ ] **Step 3: Write storage smoke test**

```python
def test_store_initializes_schema(tmp_path):
    store = RepoStore(tmp_path / "repo.db")
    store.initialize()

    assert store.get_meta("missing") is None
```

Run:

```bash
uv run pytest tests/test_storage.py -q
```

Expected:

```text
FAIL: RepoStore not implemented
```

- [ ] **Step 4: Implement schema and `RepoStore.initialize()`**

Use the exact SQL schema above.

- [ ] **Step 5: Run storage test**

Expected:

```text
PASS
```

- [ ] **Step 6: Create dummy MCP tool**

```python
@mcp.tool()
def get_task_context(
    task: str,
    max_chars: int = 8000,
) -> TaskContextResponse:
    return TaskContextResponse(
        task=task,
        repo_commit=None,
        items=[],
        relationships=[],
        stats=TaskContextStats(
            repository_files=0,
            repository_chars=0,
            context_chars=0,
            reduction_percent=0.0,
        ),
    )
```

- [ ] **Step 7: MCP smoke test**

Use in-process MCP client or Inspector to verify tool exists.

- [ ] **Step 8: Commit**

```bash
git add .
git commit -m "feat: establish RepoIntel core contracts and MCP server"
```

### M1 Copilot kickoff prompt

```text
Implement only the RepoIntel shared contracts described in the plan:
models.py, SQLite RepoStore, CLI skeleton and an MCPServer v2 server.
Do not implement indexing, graph, git analysis or retrieval.
Keep method signatures exactly as specified because four parallel
developers will code against them.
```

---

# 16. Task 2 — Repository Scanner + FTS Index

**Owner:** Member 2  
**Deadline:** minute 55

**Files:**
- Create: `src/repointel/indexer/scanner.py`
- Create: `src/repointel/indexer/chunker.py`
- Create: `src/repointel/indexer/service.py`
- Test: `tests/test_indexer.py`

**Consumes:**

```python
RepoStore.upsert_file()
RepoStore.replace_chunks()
RepoStore.search_chunks()
```

**Produces:**

```python
class IndexService:
    def index_repository(self, root: Path) -> IndexStats: ...
```

### Steps

- [ ] **Step 1: Scanner failing test**

```python
def test_scanner_ignores_venv_and_finds_python(tmp_path):
    (tmp_path / "app.py").write_text("print('hi')")
    (tmp_path / ".venv").mkdir()
    (tmp_path / ".venv" / "bad.py").write_text("x = 1")

    paths = scan_repository(tmp_path)

    assert Path("app.py") in paths
    assert Path(".venv/bad.py") not in paths
```

- [ ] **Step 2: Implement scanner**

Support:

```text
.py
.md
```

- [ ] **Step 3: Chunker failing test**

```python
def test_python_function_becomes_chunk():
    source = """
def login(user):
    return user.id
"""
    chunks = chunk_python("auth.py", source)

    assert chunks[0].start_line == 2
    assert "def login" in chunks[0].content
```

- [ ] **Step 4: Implement chunker**

Use AST line ranges where available.

Fallback:

```python
WINDOW_LINES = 80
```

- [ ] **Step 5: Persist chunks + FTS**

Each chunk writes to both:

```text
chunks
chunks_fts
```

- [ ] **Step 6: Golden retrieval test**

```python
hits = store.search_chunks(
    "refresh token",
    limit=5,
)

assert any("token" in hit.path for hit in hits)
```

- [ ] **Step 7: Commit**

```bash
git add src/repointel/indexer tests/test_indexer.py
git commit -m "feat: index repository files into local search"
```

### M2 Copilot kickoff prompt

```text
Own only RepoIntel repository scanning, chunking and SQLite FTS indexing.
Use the shared RepoStore interfaces exactly as defined. Optimize for
Python repos and deterministic hackathon behavior. Do not edit shared
models, MCP, graph or retrieval code.
```

---

# 17. Task 3 — Python Symbol Graph

**Owner:** Member 3  
**Deadline:** minute 55

**Files:**
- Create: `src/repointel/graph/parser.py`
- Create: `src/repointel/graph/resolver.py`
- Create: `src/repointel/graph/service.py`
- Test: `tests/test_graph.py`

**Consumes:**

```python
RepoStore.replace_symbols()
RepoStore.find_symbols()
RepoStore.replace_edges_from_file()
```

**Produces:**

```python
GraphService.index_file(path, source, file_id)
GraphService.explain_symbol(...)
GraphService.impact(...)
```

### Steps

- [ ] **Step 1: Symbol extraction failing test**

```python
def test_extracts_class_method():
    source = """
class TokenService:
    def rotate(self, token: str) -> str:
        return token
"""

    result = parse_python_symbols(source)

    names = [x.qualified_name for x in result.symbols]

    assert "TokenService" in names
    assert "TokenService.rotate" in names
```

- [ ] **Step 2: Implement AST extraction**

Capture:

```text
class
function
method
async function
```

- [ ] **Step 3: Import graph test**

```python
source = "from auth.repository import TokenRepository"

result = parse_python_symbols(source)

assert result.imports[0].module == "auth.repository"
```

- [ ] **Step 4: Resolve module → repository file**

Examples:

```text
auth.repository
→ auth/repository.py

app.auth.repository
→ app/auth/repository.py
```

- [ ] **Step 5: Calls extraction**

Support:

```python
rotate()
service.rotate()
TokenService.rotate()
```

Heuristic name resolution is sufficient.

- [ ] **Step 6: Persist edges**

Create:

```text
file --defines--> symbol
file --imports--> file
symbol --calls--> symbol
```

- [ ] **Step 7: Impact smoke test**

```python
impact = graph.impact(
    "TokenService.rotate",
    depth=2,
)

assert "AuthRouter.refresh" in impact.direct_dependents
```

- [ ] **Step 8: Commit**

```bash
git add src/repointel/graph tests/test_graph.py
git commit -m "feat: build repository symbol dependency graph"
```

### M3 Copilot kickoff prompt

```text
Build RepoIntel's Python AST symbol/dependency graph. Prioritize useful
heuristics over perfect static analysis. Extract classes/functions/methods,
imports and basic calls, then persist graph edges through RepoStore.
Do not change shared contracts.
```

---

# 18. Task 4 — Git Archaeology + Local Memory

**Owner:** Member 4  
**Deadline:** minute 55

**Files:**
- Create: `src/repointel/gitintel/service.py`
- Create: `src/repointel/memory/service.py`
- Test: `tests/test_gitintel.py`
- Test: `tests/test_memory.py`

**Produces:**

```python
GitService
MemoryService
```

### Steps

- [ ] **Step 1: Git HEAD test**

```python
def test_head_sha(repo_fixture):
    git = GitService(repo_fixture)

    sha = git.head_sha()

    assert sha is not None
    assert len(sha) >= 7
```

- [ ] **Step 2: Implement subprocess helper**

```python
def _git(
    root: Path,
    *args: str,
) -> str:
    result = subprocess.run(
        ["git", *args],
        cwd=root,
        check=True,
        text=True,
        capture_output=True,
    )
    return result.stdout
```

- [ ] **Step 3: Implement file history**

```bash
git log --format=... -- path
```

- [ ] **Step 4: Implement symbol archaeology**

```bash
git log -S "rotate_refresh_token" --all
```

- [ ] **Step 5: Memory write/read test**

```python
memory_id = memory.add_fact(
    content="Token logic belongs in TokenService.",
    source_path="auth/service.py",
    start_line=1,
    end_line=30,
)

results = memory.search("token logic")

assert results[0].content.startswith("Token logic")
```

- [ ] **Step 6: Implement fact memory**

Memory records include:

```text
file hash
current HEAD
confidence
```

- [ ] **Step 7: Implement episodic memory helper**

```python
memory.record_task_context(
    task="implement refresh rotation",
    context=response,
)
```

- [ ] **Step 8: Commit**

```bash
git add src/repointel/gitintel src/repointel/memory
git add tests/test_gitintel.py tests/test_memory.py
git commit -m "feat: add git archaeology and persistent repository memory"
```

### M4 Copilot kickoff prompt

```text
Implement RepoIntel Git archaeology and persistent local memory only.
Use native git subprocesses, not an extra Git library. Every memory
should retain evidence: path/line/hash/commit where available.
Keep the implementation small enough to finish within 45 minutes.
```

---

# 19. Task 5 — Retrieval Engine + Context Pack

**Owner:** Member 5  
**Deadline:** minute 65

**Files:**
- Create: `src/repointel/retrieval/ranker.py`
- Create: `src/repointel/retrieval/context.py`
- Create: `tests/test_retrieval.py`
- Create: `tests/fixtures/demo_repo/*`
- Create: `benchmarks/tasks.json`
- Create: `benchmarks/run_benchmark.py`

**Consumes all other subsystems.**

**Produces:**

```python
class ContextService:
    def get_task_context(
        self,
        task: str,
        max_chars: int = 8000,
    ) -> TaskContextResponse: ...
```

### Golden task

```text
Implement refresh-token rotation.
```

Fixture must include:

```text
auth/router.py
auth/token_service.py
auth/token_repository.py
tests/test_refresh.py
```

plus irrelevant modules.

### Steps

- [ ] **Step 1: Golden failing test**

```python
response = context.get_task_context(
    "Implement refresh-token rotation"
)

titles = [item.title for item in response.items]

assert any(
    "token_service.py" in title
    for title in titles
)
```

- [ ] **Step 2: Implement FTS candidate retrieval**

Top:

```text
20 chunks
```

- [ ] **Step 3: Add symbol candidates**

Use:

```python
store.find_symbols(task)
```

after tokenizing:

```text
refresh
token
rotation
```

- [ ] **Step 4: Graph expansion**

Expand top five candidates by one hop.

- [ ] **Step 5: Memory candidates**

Top five relevant memories.

- [ ] **Step 6: Historical evidence**

Only request Git history for top three code/symbol candidates.

This prevents unnecessary Git work.

- [ ] **Step 7: Apply context budget**

Sort descending score.

Append until:

```text
total_chars <= max_chars
```

- [ ] **Step 8: Calculate stats**

```python
reduction = (
    1
    - response_chars / max(repository_chars, 1)
) * 100
```

- [ ] **Step 9: Persist episodic task run**

Every successful request writes `task_runs`.

- [ ] **Step 10: Commit**

```bash
git add src/repointel/retrieval tests benchmarks
git commit -m "feat: compile task-aware repository context"
```

### M5 Copilot kickoff prompt

```text
Build the RepoIntel task-aware retrieval and benchmark path. Optimize
specifically for one excellent end-to-end demo. Combine FTS candidates,
symbol matches, one-hop graph expansion, memory and limited Git evidence
into an 8,000-character context budget. Do not build new infrastructure.
```

---

# 20. Task 6 — Replace MCP Stub With Real Services

**Owner:** Member 1  
**Start:** minute 55  
**Deadline:** minute 75

**Modify:**
- `src/repointel/mcp/server.py`
- `src/repointel/cli.py`
- `tests/test_mcp.py`

### Dependency

Requires merge of:

```text
M2 index
M3 graph
M4 git/memory
M5 context service
```

### Service container

```python
class Services:
    store: RepoStore
    graph: GraphService
    git: GitService
    memory: MemoryService
    context: ContextService
```

### `get_task_context`

```python
@mcp.tool()
def get_task_context(
    task: str,
    max_chars: int = 8000,
) -> TaskContextResponse:
    return services.context.get_task_context(
        task=task,
        max_chars=max_chars,
    )
```

### `explain_symbol`

Delegates:

```text
store
graph
git
```

### `get_change_impact`

Delegates:

```text
graph traversal
```

### `search_history`

Delegates:

```text
GitService.search_history
```

### Verification

- [ ] MCP Inspector lists all four tools.
- [ ] `get_task_context` produces structured output.
- [ ] Unknown symbol returns empty-but-valid response instead of crashing.
- [ ] Git-less repo does not crash task retrieval.
- [ ] Commit.

```bash
git commit -am "feat: expose repository intelligence through MCP"
```

---

# 21. Task 7 — CLI `init`

**Owners:** M1 + M2 + M3 + M4  
**Deadline:** minute 80

Flow:

```python
def init_repository(root: Path) -> None:
    store.initialize()

    files = index_service.index_repository(root)

    for file in files:
        if file.path.endswith(".py"):
            graph_service.index_file(...)

    git_service.index_recent_history()

    store.set_meta(
        "repo_head",
        git_service.head_sha() or "",
    )
```

CLI output:

```text
RepoIntel

✓ 142 files
✓ 281 chunks
✓ 637 symbols
✓ 1,108 relationships
✓ 15 repository memories

Head: ae12bc8

MCP ready:
  uv run repointel mcp .
```

One person owns the final code; others only diagnose module problems.

---

# 22. Task 8 — End-to-End Copilot Demo

**Owners:** M1 + M5  
**Everyone else:** bug fixing only  
**Deadline:** minute 95

## Test 1

Prompt Copilot:

```text
Before implementing this task, use RepoIntel to understand the
repository.

Implement refresh-token rotation and update the relevant tests.
```

Expected:

```text
Copilot calls:
get_task_context(...)
```

Context should include at least:

```text
token_service
token_repository
router
refresh test
```

## Test 2

Ask:

```text
What could break if TokenService.rotate_refresh_token changes?
```

Expected:

```text
get_change_impact
```

## Test 3

Ask:

```text
Why is token lifecycle logic kept in TokenService?
```

Expected:

```text
search_history
or
get_task_context
```

---

# 23. Benchmark

`benchmarks/tasks.json`

```json
[
  {
    "id": "refresh-rotation",
    "task": "Implement refresh-token rotation",
    "expected_files": [
      "auth/token_service.py",
      "auth/token_repository.py",
      "tests/test_refresh.py"
    ]
  },
  {
    "id": "login-rate-limit",
    "task": "Add rate limiting to login",
    "expected_files": [
      "auth/router.py",
      "auth/token_service.py"
    ]
  },
  {
    "id": "token-impact",
    "task": "What changes if TokenService.rotate_refresh_token changes?",
    "expected_files": [
      "auth/token_service.py",
      "auth/router.py",
      "tests/test_refresh.py"
    ]
  }
]
```

Metrics:

```text
Recall@5 relevant files
context chars
estimated context tokens
repository chars
reduction %
retrieval milliseconds
```

Output:

```text
Task                 Recall@5   Context    Reduction
refresh-rotation       3/3      5.8k       94.2%
login-rate-limit       2/2      4.1k       95.9%
token-impact           3/3      6.2k       93.8%
```

Use only actual measured values in the presentation.

---

# 24. P1 Vector Search — Only If Ahead

Start only if:

```text
✓ MCP real call works
✓ context retrieval works
✓ graph works
✓ demo test works
```

before minute 75.

Member 2 may add:

```text
sqlite-vec
```

Behind interface:

```python
class SemanticIndex:
    def add(
        self,
        chunk_id: int,
        embedding: list[float],
    ) -> None: ...

    def search(
        self,
        embedding: list[float],
        limit: int,
    ) -> list[SearchHit]: ...
```

Retrieval weighting changes to:

```python
score = (
    0.35 * vector_score
    + 0.30 * lexical_score
    + 0.20 * symbol_score
    + 0.10 * graph_score
    + 0.05 * memory_score
)
```

If model download, native extension or embedding integration takes more than **8 minutes**, revert immediately to FTS5.

The pitch can still say:

> “The retrieval layer supports semantic indexing; this prototype falls back to deterministic local FTS for hackathon reliability.”

---

# 25. P1 Incremental Sync — Only If Ahead

`repointel sync`

Algorithm:

```text
stored file SHA
       ↕
current file SHA
       │
       ▼
changed files only
       │
       ├─ replace chunks
       ├─ replace symbols
       ├─ replace outgoing edges
       └─ invalidate memories referencing changed hash
```

No filesystem watcher.

No daemon.

No background thread.

---

# 26. Integration Order

Merge strictly:

```text
1. M1 contracts
2. M2 index
3. M3 graph
4. M4 git/memory
5. M5 retrieval
6. M1 MCP wiring
```

Every branch rebases from contract commit before coding deeply.

Do not let M2/M3/M4 independently modify common models to “make things easier”.

If a contract is missing:

```text
tell M1
→ M1 modifies contract
→ everyone pulls
```

---

# 27. Hard Feature Freeze

At minute 90:

```text
NO:
vector experiments
new graph relation
web UI
Claude integration
Codex integration
better parser
better memory generation
new MCP tool
```

Only:

```text
demo blocker fixes
MCP fixes
retrieval fixes
benchmark
README
pitch
```

---

# 28. Demo Script

### 0:00–0:20 — Problem

```text
“Every coding agent session starts almost from zero.

It greps files, opens modules, follows imports and rereads Git history
just to reconstruct a repository it has already seen.”
```

### 0:20–0:40 — Product

```text
“RepoIntel indexes the repository once and exposes a queryable mental
model through MCP.”
```

### 0:40–1:00 — Init

```bash
repointel init .
```

Show:

```text
files
symbols
relationships
memories
```

### 1:00–1:40 — Copilot

Prompt:

```text
Implement refresh-token rotation.
```

Show Copilot call:

```text
get_task_context
```

Then show:

```text
relevant symbols
relationships
Git evidence
tests
```

### 1:40–2:00 — Result

Copilot edits.

Run tests.

```text
✓ tests passed
```

### 2:00–2:20 — Metric

```text
Repository:       XX,XXX chars
RepoIntel context: X,XXX chars
Reduction:          XX.X%
Relevant files:      X/X
```

### Closing

> **“We don't give Copilot more context. We give it repository intelligence.”**

---

# 29. Emergency Degradation Plan

If graph calls are broken:

```text
get_task_context = FTS + symbols + Git
```

Still demo.

If Git fails:

```text
FTS + graph + memory
```

Still demo.

If memory fails:

```text
FTS + graph + Git
```

Still demo.

If vector fails:

```text
FTS5
```

Expected.

If `get_change_impact` fails:

```text
only demo get_task_context
```

Still valid.

The only unrecoverable failure is:

```text
Copilot cannot invoke MCP
```

Therefore MCP smoke test happens in the first 20 minutes.

---

# 30. Final Member Checklist

## Member 1

```text
0–15   shared API + DB
15–20  MCP stub callable
20–55  CLI/store fixes + assist
55–75  real MCP integration
75–95  Copilot integration
95–120 integration/demo
```

## Member 2

```text
0–10   pull contracts
10–35  scanner/chunker
35–50  FTS persistence/search
50–60  tests
60–75  help retrieval
75+    optional vector / bugs
```

## Member 3

```text
0–10   pull contracts
10–30  AST symbols
30–45  imports/calls
45–55  graph persistence
55–65  impact query
65+    graph/retrieval fixes
```

## Member 4

```text
0–10   pull contracts
10–30  Git service
30–45  memory persistence
45–55  history evidence
55–65  tests
65+    demo evidence / fixes
```

## Member 5

```text
0–15   demo fixture + golden tests
15–35  retrieval skeleton
35–55  graph/memory/history enrichment
55–65  context budgeting/stats
65–80  benchmark
80+    own demo flow with M1
```

---

# 31. Final Acceptance Tests

Run:

```bash
uv run pytest -q
```

Required:

```text
all critical tests pass
```

Then:

```bash
uv run repointel init tests/fixtures/demo_repo
```

Required:

```text
files > 0
symbols > 0
edges > 0
```

Then MCP:

```bash
uv run mcp dev src/repointel/mcp/server.py
```

Required:

```text
4 tools visible
```

Then real host:

```text
GitHub Copilot
→ get_task_context
→ structured RepoIntel output
```

Finally:

```text
Copilot code change
→ repository tests
→ PASS
```

That is the hackathon Definition of Done.