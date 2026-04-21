# Phase 0 + Phase 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rename repo to MindForgeUniversal, universalize paths and integration docs, add MCP + source adapter seams, then deliver Phase 1 "Trust" (provenance, eval harness, knowledge hygiene).

**Architecture:** Phase 0 is structural prep — central path resolution, one consolidated MCP server behind a client-adapter seam, a `SourceAdapter` protocol the current markdown parser implements, and per-harness integration docs. Phase 1 threads `SourceRef` citations through the whole pipeline, adds a fixture-corpus evaluation runner, and adds rule-based conflict/decay/orphan detection with a review TUI.

**Tech Stack:** Python 3.10+ stdlib, `pyyaml`, `networkx`, `pytest`, `argparse`. No new runtime deps in core. Optional `[eval]` extra adds `jsonschema`.

**Working branch:** `claude/phase0-and-1`. Commit cadence: one commit per task. Do not push until Phase 1 exit review.

---

## Scope summary

- **Phase 0 (Plan 0):** rename, paths module, MCP server consolidation + adapter seam, source adapter seam, integration docs. Auto-executable except GitHub repo rename (requires user confirmation).
- **Phase 1.1 (Plan 1):** concept provenance — `SourceRef` threaded through pipeline.
- **Phase 1.2 (Plan 2):** evaluation harness — fixture corpus + scorer + CLI + CI.
- **Phase 1.3 (Plan 3):** knowledge hygiene — conflicts, decay, orphans, review TUI.
- **Exit (Plan 4):** verify exit criteria, push branch, summarize.

All work lands on `claude/phase0-and-1`. No PR from plan execution itself.

---

## Plan 0: Phase 0 — Rename, universalization, seams

### Task 0.1: Create working branch

**Files:** (none — git op)

- [ ] Create and switch to branch

```bash
git checkout -b claude/phase0-and-1
```

- [ ] Verify branch

```bash
git status
```
Expected: `On branch claude/phase0-and-1`, clean tree.

---

### Task 0.2: Introduce `mindforge/paths.py`

**Files:**
- Create: `mindforge/paths.py`
- Create: `tests/test_paths.py`

- [ ] Write failing tests first

`tests/test_paths.py`:
```python
"""Tests for centralized path resolution."""

from __future__ import annotations

import os
from pathlib import Path

import pytest

from mindforge.paths import MindForgePaths, resolve_root


def test_default_root_is_user_mindforge(monkeypatch):
    monkeypatch.delenv("MINDFORGE_ROOT", raising=False)
    monkeypatch.delenv("MINDFORGE_CONFIG", raising=False)
    paths = MindForgePaths.resolve()
    assert paths.root == Path.home() / ".mindforge"


def test_env_var_overrides_default(monkeypatch, tmp_path):
    monkeypatch.setenv("MINDFORGE_ROOT", str(tmp_path))
    paths = MindForgePaths.resolve()
    assert paths.root == tmp_path


def test_explicit_root_wins_over_env(monkeypatch, tmp_path):
    other = tmp_path / "other"
    monkeypatch.setenv("MINDFORGE_ROOT", str(tmp_path))
    paths = MindForgePaths.resolve(explicit_root=other)
    assert paths.root == other


def test_derived_paths(tmp_path):
    paths = MindForgePaths.resolve(explicit_root=tmp_path)
    assert paths.kbs_dir == tmp_path / "kbs"
    assert paths.trash_dir == tmp_path / "trash"
    assert paths.registry_file == tmp_path / "registry.json"


def test_config_file_path_respects_env(monkeypatch, tmp_path):
    custom_config = tmp_path / "config.yaml"
    monkeypatch.setenv("MINDFORGE_CONFIG", str(custom_config))
    paths = MindForgePaths.resolve()
    assert paths.config_file == custom_config


def test_ensure_dirs_creates_structure(tmp_path):
    paths = MindForgePaths.resolve(explicit_root=tmp_path)
    paths.ensure_dirs()
    assert paths.kbs_dir.is_dir()
    assert paths.trash_dir.is_dir()


def test_resolve_root_env_expansion(monkeypatch):
    monkeypatch.setenv("MINDFORGE_ROOT", "~/custom-mindforge")
    resolved = resolve_root()
    assert resolved == Path.home() / "custom-mindforge"
```

- [ ] Run tests — expect failure

```bash
pytest tests/test_paths.py -v
```
Expected: ImportError / ModuleNotFoundError on `mindforge.paths`.

- [ ] Implement `mindforge/paths.py`

```python
"""Centralized path resolution for MindForge.

Precedence for the root directory:
    1. Explicit (passed by caller, e.g. CLI --root)
    2. Env var MINDFORGE_ROOT
    3. Config file at $MINDFORGE_CONFIG or ~/.mindforge/config.yaml
    4. Default ~/.mindforge

No third-party deps; stdlib only.
"""

from __future__ import annotations

import os
from dataclasses import dataclass
from pathlib import Path


def _expand(p: str | Path) -> Path:
    return Path(os.path.expanduser(os.path.expandvars(str(p)))).resolve()


def resolve_root(explicit: str | Path | None = None) -> Path:
    """Return the MindForge root directory per the documented precedence."""
    if explicit is not None:
        return _expand(explicit)
    env = os.environ.get("MINDFORGE_ROOT")
    if env:
        return _expand(env)
    return _expand("~/.mindforge")


def resolve_config_file() -> Path:
    """Return the path to the optional config file."""
    env = os.environ.get("MINDFORGE_CONFIG")
    if env:
        return _expand(env)
    return resolve_root() / "config.yaml"


@dataclass(frozen=True)
class MindForgePaths:
    """Resolved paths for a MindForge installation."""

    root: Path
    kbs_dir: Path
    trash_dir: Path
    registry_file: Path
    config_file: Path

    @classmethod
    def resolve(cls, explicit_root: str | Path | None = None) -> "MindForgePaths":
        root = resolve_root(explicit_root)
        return cls(
            root=root,
            kbs_dir=root / "kbs",
            trash_dir=root / "trash",
            registry_file=root / "registry.json",
            config_file=resolve_config_file(),
        )

    def ensure_dirs(self) -> None:
        self.root.mkdir(parents=True, exist_ok=True)
        self.kbs_dir.mkdir(parents=True, exist_ok=True)
        self.trash_dir.mkdir(parents=True, exist_ok=True)
```

- [ ] Run tests — expect pass

```bash
pytest tests/test_paths.py -v
```
Expected: 7 passed.

- [ ] Commit

```bash
git add mindforge/paths.py tests/test_paths.py
git commit -m "feat(paths): add centralized path resolution with env/config precedence"
```

---

### Task 0.3: MCP server consolidation

The repo carries three server files (`server.py` is the multikb version, `server_multikb.py` and `server_original.py` are drift). Keep `server.py`, delete the other two, and replace its `MINDFORGE_ROOT` hardcode with `MindForgePaths`.

**Files:**
- Modify: `mindforge/mcp/server.py`
- Delete: `mindforge/mcp/server_multikb.py`
- Delete: `mindforge/mcp/server_original.py`

- [ ] Verify existing test coverage

```bash
pytest tests/test_mcp.py -v
```
Expected: all pass (baseline).

- [ ] Replace Hermes-specific path hardcode in `mindforge/mcp/server.py`

Find the block:
```python
MINDFORGE_ROOT = Path(os.environ.get("MINDFORGE_ROOT", os.path.expanduser("~/.hermes/mindforge")))
KBS_DIR = MINDFORGE_ROOT / "kbs"
TRASH_DIR = MINDFORGE_ROOT / "trash"
REGISTRY_FILE = MINDFORGE_ROOT / "registry.json"
```

Replace with:
```python
from mindforge.paths import MindForgePaths

_PATHS = MindForgePaths.resolve()
MINDFORGE_ROOT = _PATHS.root
KBS_DIR = _PATHS.kbs_dir
TRASH_DIR = _PATHS.trash_dir
REGISTRY_FILE = _PATHS.registry_file
```

Also remove the stray `sys.path.insert(0, ...)` at the top that hardcodes `/home/will/MindForge` — it's no longer relevant.

- [ ] Delete legacy server files

```bash
git rm mindforge/mcp/server_multikb.py mindforge/mcp/server_original.py
```

- [ ] Run MCP tests

```bash
pytest tests/test_mcp.py -v
```
Expected: all pass (behavior unchanged).

- [ ] Commit

```bash
git add mindforge/mcp/server.py
git commit -m "refactor(mcp): consolidate to single server; use MindForgePaths for root"
```

---

### Task 0.4: MCP client adapter seam

**Files:**
- Create: `mindforge/mcp/adapter.py`
- Create: `tests/test_mcp_adapter.py`
- Modify: `mindforge/mcp/server.py` (use adapter for tool-descriptions in the `list_tools` response)

- [ ] Write failing tests

`tests/test_mcp_adapter.py`:
```python
"""Tests for the MCP client adapter seam."""

from __future__ import annotations

from mindforge.mcp.adapter import ClientAdapter, DefaultAdapter, get_adapter


def test_default_adapter_is_registered():
    adapter = get_adapter("default")
    assert isinstance(adapter, DefaultAdapter)


def test_unknown_adapter_name_falls_back_to_default():
    adapter = get_adapter("does-not-exist")
    assert isinstance(adapter, DefaultAdapter)


def test_adapter_can_be_overridden_by_env(monkeypatch):
    monkeypatch.setenv("MINDFORGE_MCP_ADAPTER", "default")
    adapter = get_adapter()
    assert isinstance(adapter, DefaultAdapter)


def test_default_adapter_passes_description_through():
    adapter = DefaultAdapter()
    desc = "Long " * 100
    assert adapter.format_tool_description(desc) == desc


def test_adapter_is_extensible():
    class TruncatingAdapter(ClientAdapter):
        def format_tool_description(self, description: str) -> str:
            return description[:50]

    adapter = TruncatingAdapter()
    assert len(adapter.format_tool_description("x" * 500)) == 50
```

- [ ] Run tests — expect failure

```bash
pytest tests/test_mcp_adapter.py -v
```

- [ ] Implement `mindforge/mcp/adapter.py`

```python
"""Client adapter seam for the MCP server.

Purpose: future MCP adaptations (client-specific quirks like tool-description
length limits, response shape tweaks, schema compatibility) plug in here
without touching the core server. This session ships ONE DefaultAdapter —
it is a behavior-preserving pass-through. The seam exists so future sessions
have somewhere to hang per-client logic.

Selection order:
    1. Explicit argument to get_adapter(name)
    2. Env var MINDFORGE_MCP_ADAPTER
    3. "default"
"""

from __future__ import annotations

import os
from typing import Any


class ClientAdapter:
    """Base class for MCP client adapters. Override to customize per client."""

    name: str = "base"

    def format_tool_description(self, description: str) -> str:
        """Return a description string safe for the target client."""
        return description

    def format_tool_response(self, payload: Any) -> Any:
        """Return a response payload safe for the target client."""
        return payload


class DefaultAdapter(ClientAdapter):
    """Pass-through adapter. Matches current behavior."""

    name = "default"


_REGISTRY: dict[str, type[ClientAdapter]] = {
    "default": DefaultAdapter,
}


def register_adapter(name: str, cls: type[ClientAdapter]) -> None:
    _REGISTRY[name] = cls


def get_adapter(name: str | None = None) -> ClientAdapter:
    chosen = name or os.environ.get("MINDFORGE_MCP_ADAPTER") or "default"
    cls = _REGISTRY.get(chosen, DefaultAdapter)
    return cls()
```

- [ ] Wire adapter into `mindforge/mcp/server.py`

Near the top, after imports and path resolution:
```python
from mindforge.mcp.adapter import get_adapter

_ADAPTER = get_adapter()
```

In the `list_tools` handler (find the function that returns the list of `Tool(...)` definitions), wrap each tool's description:
```python
Tool(name="...", description=_ADAPTER.format_tool_description("..."), ...)
```

Behavior-preserving: `DefaultAdapter.format_tool_description` is identity.

- [ ] Run tests

```bash
pytest tests/test_mcp_adapter.py tests/test_mcp.py -v
```
Expected: all pass.

- [ ] Commit

```bash
git add mindforge/mcp/adapter.py tests/test_mcp_adapter.py mindforge/mcp/server.py
git commit -m "feat(mcp): add ClientAdapter seam for future per-client quirks"
```

---

### Task 0.5: Source adapter protocol

**Files:**
- Create: `mindforge/ingestion/sources.py`
- Create: `tests/test_source_adapter.py`
- Modify: `mindforge/ingestion/parser.py` (expose `MarkdownSourceAdapter` as the canonical markdown-file adapter; keep existing `parse_transcript`/`parse_all_transcripts` functions for back-compat)

- [ ] Write failing tests

`tests/test_source_adapter.py`:
```python
"""Tests for SourceAdapter protocol and MarkdownSourceAdapter."""

from __future__ import annotations

from pathlib import Path

import pytest

from mindforge.ingestion.parser import MarkdownSourceAdapter, parse_transcript
from mindforge.ingestion.sources import SourceAdapter, get_adapter_for


def test_protocol_accepts_markdown_adapter():
    adapter = MarkdownSourceAdapter()
    # Duck-typing: if it parses, it qualifies.
    assert callable(getattr(adapter, "parse", None))


def test_markdown_adapter_matches_existing_parser(tmp_path: Path):
    transcript = tmp_path / "t.md"
    transcript.write_text(
        "Human: What is attention?\n\n"
        "Assistant: A mechanism that weighs tokens.\n"
    )
    adapter = MarkdownSourceAdapter()
    adapter_result = adapter.parse(transcript)
    direct_result = parse_transcript(transcript)
    assert [t.role for t in adapter_result.turns] == [t.role for t in direct_result.turns]
    assert [t.content for t in adapter_result.turns] == [t.content for t in direct_result.turns]


def test_get_adapter_for_markdown_returns_markdown_adapter(tmp_path: Path):
    p = tmp_path / "x.md"
    p.write_text("Assistant: hi")
    adapter = get_adapter_for(p)
    assert isinstance(adapter, MarkdownSourceAdapter)


def test_get_adapter_for_unknown_extension_raises(tmp_path: Path):
    p = tmp_path / "x.pdf"
    p.write_text("")
    with pytest.raises(ValueError):
        get_adapter_for(p)
```

- [ ] Run tests — expect failure

- [ ] Implement `mindforge/ingestion/sources.py`

```python
"""SourceAdapter protocol.

The pipeline ingests from multiple source types: markdown transcripts today;
Claude Code project JSONL, ChatGPT exports, Cursor logs, Hermes transcripts
tomorrow (Phase 4). All of them boil down to a list of turns.

This module defines the adapter protocol. One adapter ships today:
MarkdownSourceAdapter (see parser.py). Future adapters register themselves
via register_adapter().
"""

from __future__ import annotations

from pathlib import Path
from typing import Callable, Protocol, runtime_checkable

from mindforge.ingestion.parser import Transcript


@runtime_checkable
class SourceAdapter(Protocol):
    """A source adapter produces a Transcript from a path or stream."""

    def parse(self, path: Path) -> Transcript: ...


# extension -> adapter factory
_ADAPTERS: dict[str, Callable[[], SourceAdapter]] = {}


def register_adapter(extension: str, factory: Callable[[], SourceAdapter]) -> None:
    """Register an adapter factory for a file extension (include the dot)."""
    _ADAPTERS[extension.lower()] = factory


def get_adapter_for(path: Path) -> SourceAdapter:
    """Return a configured adapter for the given path, by extension."""
    ext = path.suffix.lower()
    factory = _ADAPTERS.get(ext)
    if factory is None:
        raise ValueError(f"No SourceAdapter registered for extension: {ext!r}")
    return factory()
```

- [ ] Refactor `mindforge/ingestion/parser.py` to expose `MarkdownSourceAdapter`

Add to the bottom of `parser.py`:

```python
class MarkdownSourceAdapter:
    """SourceAdapter for markdown and plain-text transcript files."""

    def parse(self, path: Path) -> Transcript:
        return parse_transcript(path)


# Register at import time for .md and .txt.
from mindforge.ingestion import sources as _sources  # noqa: E402
_sources.register_adapter(".md", MarkdownSourceAdapter)
_sources.register_adapter(".txt", MarkdownSourceAdapter)
```

Note: the late import-and-register is deliberate to break the circular-import between `parser` and `sources`.

- [ ] Run tests

```bash
pytest tests/test_source_adapter.py tests/test_ingestion.py -v
```
Expected: all pass.

- [ ] Commit

```bash
git add mindforge/ingestion/sources.py mindforge/ingestion/parser.py tests/test_source_adapter.py
git commit -m "feat(ingestion): add SourceAdapter protocol; MarkdownSourceAdapter wraps current parser"
```

---

### Task 0.6: Create `docs/integrations/` with README + 7 harness guides

**Files:**
- Create: `docs/integrations/README.md`
- Create: `docs/integrations/claude-code.md`
- Create: `docs/integrations/claude-desktop.md`
- Create: `docs/integrations/hermes-agent.md`
- Create: `docs/integrations/openclaw.md`
- Create: `docs/integrations/codex-cli.md`
- Create: `docs/integrations/openai-agents-sdk.md`
- Create: `docs/integrations/generic-mcp.md`

This is documentation, not code — TDD doesn't apply. Follow this template for each harness doc:

```markdown
# Integrating MindForge with <Harness>

## Prerequisites
- Python 3.10+, `pip install -e .` (or `pip install mindforge` when published)
- (harness-specific deps)

## Configuration

(harness-specific config snippet pointing at `python -m mindforge.mcp.server` with `MINDFORGE_ROOT` env var)

## Verification

Start the harness, ensure the MCP tools `kb_list`, `search`, `get_concept`, etc. are listed.

## Known limitations

(Any quirks for this harness — leave sections empty / "none observed" if none.)
```

- [ ] Write `docs/integrations/README.md` — compatibility matrix

```markdown
# MindForge Integrations

MindForge speaks the Model Context Protocol (MCP) over stdio JSON-RPC. Any MCP-compatible harness can use it. Common integrations are documented here.

## Compatibility matrix

| Harness | Install method | Config path | MCP stdio | Status |
|---|---|---|---|---|
| [Claude Code](claude-code.md) | Anthropic CLI | `~/.claude/mcp_servers.json` or project `.mcp.json` | ✓ | Supported |
| [Claude Desktop](claude-desktop.md) | macOS/Windows app | `~/Library/Application Support/Claude/claude_desktop_config.json` | ✓ | Supported |
| [Hermes Agent](hermes-agent.md) | Self-hosted | `~/.hermes/config.yaml` | ✓ | Supported |
| [OpenClaw](openclaw.md) | `github.com/openclaw/openclaw` | Project `.openclaw/config.yaml` | ✓ | Community-supported |
| [Codex CLI](codex-cli.md) | OpenAI Codex | `~/.codex/config.toml` | ✓ | Supported |
| [OpenAI Agents SDK](openai-agents-sdk.md) | Python library | Programmatic | ✓ | Supported |
| [Generic MCP client](generic-mcp.md) | Any stdio JSON-RPC MCP client | — | ✓ | — |

## Common environment

Every integration sets one env var:

    MINDFORGE_ROOT=<path to your KB root>

Default: `~/.mindforge`. Hermes-style installs may prefer `~/.hermes/mindforge`.

## Command

The MCP server runs as:

    python -m mindforge.mcp.server

No args. Tool list, tool shapes, and response formats are documented in each harness guide.
```

- [ ] Write each per-harness doc

For each of the six harnesses, create a short doc (50-100 lines) that covers prerequisites, config, verification, and known limitations. See the template above. Include actual commands and actual config snippets. Keep Hermes-specific skill content in `hermes-agent.md` (generalized — no hardcoded user paths). For OpenAI Agents SDK use a Python snippet showing `MCPServerStdio` usage.

- [ ] Commit

```bash
git add docs/integrations/
git commit -m "docs(integrations): add per-harness guides + compatibility matrix"
```

---

### Task 0.7: Update `skills/SKILL.md` as a pointer

**Files:**
- Modify: `skills/SKILL.md`

- [ ] Shorten to a pointer with Hermes-skill-system affordances only

Replace the current Hermes-centric body with a short version:

```markdown
# MindForge Skill (Hermes Agent)

> The full integration guide moved to `docs/integrations/hermes-agent.md`. This file stays as a Hermes skill entry for the Hermes skill system.

**Repository:** https://github.com/AcceleratedIndustries/MindForgeUniversal

See `docs/integrations/README.md` for all supported harnesses.

## Quick Hermes config

```yaml
mcp_servers:
  mindforge:
    command: python
    args: ["-m", "mindforge.mcp.server"]
    env:
      MINDFORGE_ROOT: /home/you/.hermes/mindforge
    timeout: 60
```

See `docs/integrations/hermes-agent.md` for the full workflow and tool reference.
```

- [ ] Commit

```bash
git add skills/SKILL.md
git commit -m "docs(skills): shrink Hermes skill to pointer + minimal config"
```

---

### Task 0.8: Update roadmap + README references

**Files:**
- Modify: `docs/ROADMAP.md` (line 16 — change `(Hermes, `dade9ab`)` to `(ref commit `dade9ab`)`)
- Modify: `README.md` (no Hermes references; verify nothing needs change — if not, skip)

- [ ] Grep for stale references

```bash
grep -rn "MindForgeForHermes\|\.hermes/mindforge" docs/ README.md skills/ 2>/dev/null
```

- [ ] Replace `MindForgeForHermes` with `MindForgeUniversal` wherever the string appears in docs (repo URL references). Leave the local working directory name untouched.

- [ ] Commit

```bash
git add -u docs/ROADMAP.md README.md skills/
git commit -m "docs: update URLs from MindForgeForHermes to MindForgeUniversal"
```

---

### Task 0.9: Wire `MindForgePaths` into `MindForgeConfig`

**Files:**
- Modify: `mindforge/config.py`

- [ ] Add optional `kb_root` attribute that defaults to `MindForgePaths.resolve().root`

```python
from mindforge.paths import MindForgePaths

@dataclass
class MindForgeConfig:
    # ...existing fields...
    kb_root: Path | None = None   # if None, resolved from env/defaults

    def __post_init__(self) -> None:
        if self.kb_root is None:
            self.kb_root = MindForgePaths.resolve().root
        self.concepts_dir = self.output_dir / "concepts"
        self.graph_dir = self.output_dir / "graph"
        self.embeddings_dir = self.output_dir / "embeddings"
```

- [ ] Run all tests

```bash
pytest -q
```
Expected: all pass.

- [ ] Commit

```bash
git add mindforge/config.py
git commit -m "feat(config): derive kb_root from MindForgePaths by default"
```

---

### Task 0.10: GitHub repo rename (GATED — requires user confirmation)

**This task must not execute without explicit user confirmation at the time.** When reached, pause and ask:

> "About to run: gh repo rename MindForgeUniversal. Proceed?"

After confirmation:

```bash
gh repo rename MindForgeUniversal
git remote set-url origin https://github.com/AcceleratedIndustries/MindForgeUniversal.git
git remote -v   # verify
```

Commit nothing for this task — it's a pure metadata op.

---

## Plan 1: Phase 1.1 — Concept Provenance

### Task 1.1.1: Add `SourceRef` dataclass

**Files:**
- Create: `mindforge/distillation/source_ref.py`
- Create: `tests/test_source_ref.py`

- [ ] Write failing tests

`tests/test_source_ref.py`:
```python
from __future__ import annotations

from mindforge.distillation.source_ref import SourceRef


def test_to_dict_roundtrip():
    ref = SourceRef(
        transcript_path="2025-03-14_llm_internals.md",
        transcript_hash="abcd1234",
        turn_indices=[4, 7],
        extracted_at="2025-03-14T11:22:00Z",
        chunk_id="2025-03-14_llm_internals.md:t4:c0",
    )
    data = ref.to_dict()
    assert data["transcript_path"] == "2025-03-14_llm_internals.md"
    assert data["turn_indices"] == [4, 7]
    restored = SourceRef.from_dict(data)
    assert restored == ref


def test_defaults():
    ref = SourceRef(
        transcript_path="t.md",
        transcript_hash="h",
        turn_indices=[0],
        extracted_at="2025-01-01T00:00:00Z",
    )
    assert ref.chunk_id is None
    assert ref.snippet is None


def test_snippet_is_capped_on_init():
    long = "x" * 5000
    ref = SourceRef(
        transcript_path="t.md", transcript_hash="h",
        turn_indices=[0], extracted_at="2025-01-01T00:00:00Z",
        snippet=long,
    )
    assert len(ref.snippet) == 500
```

- [ ] Run tests — expect failure

- [ ] Implement `mindforge/distillation/source_ref.py`

```python
"""SourceRef: a citation pointer from a concept to its originating turns."""

from __future__ import annotations

from dataclasses import dataclass, field
from typing import Optional


SNIPPET_MAX_CHARS = 500


@dataclass
class SourceRef:
    transcript_path: str
    transcript_hash: str
    turn_indices: list[int]
    extracted_at: str  # ISO 8601 UTC
    chunk_id: Optional[str] = None
    snippet: Optional[str] = None

    def __post_init__(self) -> None:
        if self.snippet is not None and len(self.snippet) > SNIPPET_MAX_CHARS:
            object.__setattr__(self, "snippet", self.snippet[:SNIPPET_MAX_CHARS])

    def to_dict(self) -> dict:
        return {
            "transcript_path": self.transcript_path,
            "transcript_hash": self.transcript_hash,
            "turn_indices": list(self.turn_indices),
            "extracted_at": self.extracted_at,
            "chunk_id": self.chunk_id,
            "snippet": self.snippet,
        }

    @classmethod
    def from_dict(cls, data: dict) -> "SourceRef":
        return cls(
            transcript_path=data["transcript_path"],
            transcript_hash=data["transcript_hash"],
            turn_indices=list(data["turn_indices"]),
            extracted_at=data["extracted_at"],
            chunk_id=data.get("chunk_id"),
            snippet=data.get("snippet"),
        )
```

- [ ] Run tests — expect pass

- [ ] Commit

```bash
git add mindforge/distillation/source_ref.py tests/test_source_ref.py
git commit -m "feat(provenance): add SourceRef dataclass with 500-char snippet cap"
```

---

### Task 1.1.2: Add `sources` field to `Concept`

**Files:**
- Modify: `mindforge/distillation/concept.py`
- Modify: `tests/test_distillation.py` (if it references concept serialization)

- [ ] Extend `Concept` dataclass

In `concept.py`:
```python
from mindforge.distillation.source_ref import SourceRef

@dataclass
class Concept:
    # ...existing fields...
    sources: list[SourceRef] = field(default_factory=list)
```

In `to_dict`:
```python
"sources": [s.to_dict() for s in self.sources],
```

In `from_dict`:
```python
sources = [SourceRef.from_dict(s) for s in data.get("sources", [])]
# pass `sources=sources` to cls(...)
```

In `merge_with`:
```python
merged_sources_refs = self.sources + other.sources
# dedup by (transcript_path, transcript_hash, tuple(turn_indices))
seen: set[tuple] = set()
deduped: list[SourceRef] = []
for s in merged_sources_refs:
    key = (s.transcript_path, s.transcript_hash, tuple(s.turn_indices))
    if key in seen:
        continue
    seen.add(key)
    deduped.append(s)
# pass `sources=deduped` to Concept(...)
```

- [ ] Write a test — roundtrip with sources

Add to `tests/test_source_ref.py`:
```python
from mindforge.distillation.concept import Concept
from mindforge.distillation.source_ref import SourceRef


def test_concept_roundtrip_preserves_sources():
    c = Concept(
        name="KV Cache",
        definition="d",
        explanation="e",
        sources=[SourceRef(
            transcript_path="t.md", transcript_hash="h",
            turn_indices=[0], extracted_at="2025-01-01T00:00:00Z",
        )],
    )
    restored = Concept.from_dict(c.to_dict())
    assert restored.sources == c.sources


def test_concept_merge_dedups_sources():
    ref = SourceRef(
        transcript_path="t.md", transcript_hash="h",
        turn_indices=[0], extracted_at="2025-01-01T00:00:00Z",
    )
    a = Concept(name="X", definition="d", explanation="e", sources=[ref])
    b = Concept(name="X", definition="d", explanation="e", sources=[ref])
    merged = a.merge_with(b)
    assert len(merged.sources) == 1
```

- [ ] Run tests

```bash
pytest tests/test_source_ref.py tests/test_distillation.py -v
```
Expected: all pass.

- [ ] Commit

```bash
git add mindforge/distillation/concept.py tests/test_source_ref.py
git commit -m "feat(provenance): add sources field to Concept with dedup on merge"
```

---

### Task 1.1.3: Thread turn indices through `Chunk` (already present as `turn_index` — verify and promote)

**Files:**
- Modify: `mindforge/ingestion/chunker.py` (only if needed — `Chunk` already carries `turn_index` and `source_file`)

- [ ] Verify existing `Chunk` shape is sufficient

```bash
grep -n "class Chunk\|turn_index\|source_file" mindforge/ingestion/chunker.py
```

If `Chunk.turn_index` exists (it does), no change needed. Otherwise add it.

- [ ] No commit if unchanged.

---

### Task 1.1.4: Track source chunks on `RawConcept` (heuristic + LLM extractors)

**Files:**
- Modify: `mindforge/ingestion/extractor.py`
- Modify: `mindforge/llm/extractor.py`
- Modify: `tests/test_ingestion.py`

- [ ] Read the current `RawConcept` definition

```bash
grep -n "RawConcept\|class RawConcept" mindforge/ingestion/extractor.py mindforge/llm/extractor.py
```

- [ ] Add `source_chunks: list[Chunk] = field(default_factory=list)` to the `RawConcept` dataclass (whichever module owns it — usually `mindforge/ingestion/extractor.py`).

- [ ] Update heuristic extractor: when it produces a `RawConcept`, include the originating chunk in `source_chunks`.

- [ ] Update LLM extractor similarly. The LLM may produce concepts spanning multiple chunks; collect all of them.

- [ ] Add a test

`tests/test_ingestion.py` — append:
```python
def test_heuristic_extractor_attaches_source_chunks(tmp_path):
    from mindforge.ingestion.parser import parse_transcript
    from mindforge.ingestion.chunker import chunk_turns
    from mindforge.ingestion.extractor import extract_concepts

    t = tmp_path / "t.md"
    t.write_text(
        "Assistant: KV Cache is a mechanism that stores Key and Value matrices.\n"
        "It avoids recomputation during autoregressive generation.\n"
    )
    transcript = parse_transcript(t)
    chunks = chunk_turns(transcript.turns)
    raws = extract_concepts(chunks)
    assert raws, "expected at least one RawConcept"
    for raw in raws:
        assert raw.source_chunks, f"concept {raw.name!r} missing source_chunks"
```

- [ ] Run tests

- [ ] Commit

```bash
git add mindforge/ingestion/extractor.py mindforge/llm/extractor.py tests/test_ingestion.py
git commit -m "feat(provenance): attach source chunks to RawConcept (heuristic + LLM)"
```

---

### Task 1.1.5: Distiller converts chunks → `SourceRef` on `Concept`

**Files:**
- Modify: `mindforge/distillation/distiller.py`
- Modify: `mindforge/utils/text.py` (add `content_hash_file` if not present — fallback: compute on content)

- [ ] Add transcript hash helper

If `content_hash` exists (it does — see `concept.py` imports), use it on transcript bytes for `SourceRef.transcript_hash`.

- [ ] In the distiller, at the point where a final `Concept` is constructed from merged `RawConcept`s:

```python
from datetime import datetime, timezone
from pathlib import Path

from mindforge.distillation.source_ref import SourceRef
from mindforge.utils.text import content_hash

def _source_refs_from_chunks(chunks) -> list[SourceRef]:
    # Group by (source_file, transcript_hash)
    by_file: dict[str, list] = {}
    for ch in chunks:
        by_file.setdefault(ch.source_file, []).append(ch)
    now = datetime.now(timezone.utc).isoformat()
    refs: list[SourceRef] = []
    for source_file, chs in by_file.items():
        try:
            text = Path(source_file).read_text(encoding="utf-8")
            h = content_hash(text)
        except OSError:
            h = ""
        turn_indices = sorted({ch.turn_index for ch in chs})
        # Snippet: first chunk content, capped inside SourceRef
        snippet = chs[0].content if chs else None
        refs.append(SourceRef(
            transcript_path=source_file,
            transcript_hash=h,
            turn_indices=turn_indices,
            extracted_at=now,
            chunk_id=chs[0].id if chs else None,
            snippet=snippet,
        ))
    return refs
```

Populate `concept.sources` from merged `RawConcept.source_chunks`.

- [ ] Add a test

`tests/test_distillation.py` — append:
```python
def test_distilled_concept_has_source_refs(tmp_path):
    # Use your existing distiller fixtures; the point is:
    # after distill(), concept.sources must be non-empty when source_chunks are non-empty.
    ...
```
(Use existing fixture patterns in `test_distillation.py`.)

- [ ] Run tests

- [ ] Commit

```bash
git add mindforge/distillation/distiller.py tests/test_distillation.py
git commit -m "feat(provenance): populate Concept.sources from RawConcept source_chunks"
```

---

### Task 1.1.6: Deduplicator merges source lists

**Files:**
- Modify: `mindforge/distillation/deduplicator.py`

- [ ] When two concepts are merged, combine their `source_chunks` / `sources`. The `Concept.merge_with` dedup logic from Task 1.1.2 already handles `sources`; ensure `RawConcept` merges similarly if the deduplicator runs before the distiller.

- [ ] Commit

```bash
git add mindforge/distillation/deduplicator.py
git commit -m "feat(provenance): merge source chunks on dedup"
```

---

### Task 1.1.7: Renderer emits `sources:` in YAML frontmatter

**Files:**
- Modify: `mindforge/distillation/renderer.py`

- [ ] Replace the current `sources:` frontmatter loop (which writes raw file paths) with structured `SourceRef` output:

```python
if concept.sources:
    lines.append("sources:")
    for src in concept.sources:
        lines.append(f"  - transcript: \"{src.transcript_path}\"")
        lines.append(f"    turns: [{', '.join(str(i) for i in src.turn_indices)}]")
        lines.append(f"    extracted_at: \"{src.extracted_at}\"")
# Keep source_files for back-compat if it's still populated
elif concept.source_files:
    lines.append("sources:")
    for src in concept.source_files:
        lines.append(f"  - \"{src}\"")
```

- [ ] Write a test

`tests/test_distillation.py`:
```python
def test_renderer_emits_structured_sources():
    from mindforge.distillation.renderer import render_concept
    from mindforge.distillation.concept import Concept
    from mindforge.distillation.source_ref import SourceRef

    c = Concept(
        name="KV Cache", definition="d", explanation="e",
        sources=[SourceRef(
            transcript_path="t.md", transcript_hash="h",
            turn_indices=[4, 7], extracted_at="2025-03-14T11:22:00Z",
        )],
    )
    md = render_concept(c)
    assert "transcript: \"t.md\"" in md
    assert "turns: [4, 7]" in md
    assert "extracted_at: \"2025-03-14T11:22:00Z\"" in md
```

- [ ] Run tests, commit

```bash
git add mindforge/distillation/renderer.py tests/test_distillation.py
git commit -m "feat(provenance): emit structured sources in YAML frontmatter"
```

---

### Task 1.1.8: Provenance JSON storage

**Files:**
- Modify: `mindforge/pipeline.py`
- Modify: `mindforge/config.py` (derived path)

- [ ] Add `provenance_dir` to config

In `MindForgeConfig.__post_init__`:
```python
self.provenance_dir = self.output_dir / "provenance"
```

In `ensure_dirs`:
```python
for d in [self.concepts_dir, self.graph_dir, self.embeddings_dir, self.provenance_dir]:
    d.mkdir(parents=True, exist_ok=True)
```

- [ ] In `pipeline.py` after concepts are written, write one JSON per concept:

```python
import json

def _write_provenance(concept, provenance_dir):
    if not concept.sources:
        return
    path = provenance_dir / f"{concept.slug}.json"
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(json.dumps({
        "slug": concept.slug,
        "name": concept.name,
        "sources": [s.to_dict() for s in concept.sources],
    }, indent=2))
```

Call `_write_provenance(concept, config.provenance_dir)` from wherever the pipeline iterates over final concepts to render them.

- [ ] Add a pipeline-level test

`tests/test_pipeline.py`:
```python
def test_pipeline_writes_provenance_json(tmp_path):
    # ... run pipeline on a single-transcript fixture ...
    prov_files = list((tmp_path / "output" / "provenance").glob("*.json"))
    assert prov_files, "expected provenance JSON per concept"
```

- [ ] Run tests, commit

```bash
git add mindforge/pipeline.py mindforge/config.py tests/test_pipeline.py
git commit -m "feat(provenance): write per-concept provenance JSON under output/provenance/"
```

---

### Task 1.1.9: MCP `get_concept` includes sources

**Files:**
- Modify: `mindforge/mcp/server.py`

- [ ] In the `get_concept` tool handler, add `sources` to the response dict:

```python
"sources": [s.to_dict() for s in concept.sources],
```

- [ ] Add a test

`tests/test_mcp.py`:
```python
def test_get_concept_returns_sources():
    # Build a KB with one concept that has sources, call get_concept, assert "sources" key.
    ...
```

- [ ] Run tests, commit

```bash
git add mindforge/mcp/server.py tests/test_mcp.py
git commit -m "feat(mcp): include sources in get_concept response"
```

---

### Task 1.1.10: CLI `show <slug> [--sources]`

**Files:**
- Modify: `mindforge/cli.py`

- [ ] Add subparser

```python
show = subparsers.add_parser("show", help="Show a single concept")
show.add_argument("slug", help="Concept slug")
show.add_argument("--sources", action="store_true", help="Print source citations")
show.add_argument("--output", "-o", type=Path, default=Path("output"))
```

- [ ] Handler

```python
def cmd_show(args):
    from mindforge.distillation.concept import ConceptStore
    config = MindForgeConfig(output_dir=args.output)
    store = ConceptStore.load(config.output_dir / "concepts.json")
    concept = store.get(args.slug)
    if not concept:
        print(f"Unknown concept: {args.slug}", file=sys.stderr)
        return 1
    print(f"{concept.name}")
    print(f"  {concept.definition}")
    if args.sources and concept.sources:
        print("Sources:")
        for s in concept.sources:
            turns = ", ".join(str(i) for i in s.turn_indices)
            print(f"  {s.transcript_path} (turns {turns})")
    return 0
```

Add to the `commands` dict: `"show": cmd_show`.

- [ ] Add a test

`tests/test_cli.py` (new if needed):
```python
def test_show_sources_flag_prints_citations(capsys, tmp_path):
    # ... seed a KB, call main, assert "turns" appears in stdout ...
    ...
```

- [ ] Run tests, commit

```bash
git add mindforge/cli.py tests/test_cli.py
git commit -m "feat(cli): add show <slug> [--sources] subcommand"
```

---

### Task 1.1.11: Back-compat warning for old KBs with no sources

**Files:**
- Modify: `mindforge/distillation/concept.py` (in `ConceptStore.load`)

- [ ] Emit a single stderr warning on load if any concept lacks `sources`:

```python
@classmethod
def load(cls, path: Path) -> ConceptStore:
    store = cls()
    if path.exists():
        data = json.loads(path.read_text())
        missing = 0
        for slug, cdata in data.items():
            store.concepts[slug] = Concept.from_dict(cdata)
            if not cdata.get("sources"):
                missing += 1
        if missing:
            import sys
            print(
                f"[mindforge] warning: {missing} concept(s) have no provenance. "
                f"Re-ingest to populate sources.",
                file=sys.stderr,
            )
    return store
```

- [ ] Commit

```bash
git add mindforge/distillation/concept.py
git commit -m "feat(provenance): warn on load when legacy KB lacks sources"
```

---

### Task 1.1.12: Storage protocol (minimal)

**Files:**
- Create: `mindforge/storage/__init__.py`
- Create: `mindforge/storage/fs.py`

- [ ] Introduce a minimal `Storage` protocol to centralize filesystem writes (concepts, provenance, manifest). Default impl is filesystem. This is just the seam — full migration across the codebase is Phase 3 work. Put the Phase 1.1 writes (concept files, provenance JSON) through it.

`mindforge/storage/__init__.py`:
```python
from mindforge.storage.fs import FilesystemStorage, Storage

__all__ = ["Storage", "FilesystemStorage"]
```

`mindforge/storage/fs.py`:
```python
from __future__ import annotations

from pathlib import Path
from typing import Protocol, runtime_checkable


@runtime_checkable
class Storage(Protocol):
    def write_text(self, path: Path, text: str) -> None: ...
    def read_text(self, path: Path) -> str: ...
    def exists(self, path: Path) -> bool: ...


class FilesystemStorage:
    def write_text(self, path: Path, text: str) -> None:
        path.parent.mkdir(parents=True, exist_ok=True)
        path.write_text(text, encoding="utf-8")

    def read_text(self, path: Path) -> str:
        return path.read_text(encoding="utf-8")

    def exists(self, path: Path) -> bool:
        return path.exists()
```

- [ ] Commit (no wiring this task — the seam is available; deferred adoption is fine)

```bash
git add mindforge/storage/
git commit -m "feat(storage): add minimal Storage protocol + FilesystemStorage impl"
```

---

## Plan 2: Phase 1.2 — Evaluation Harness

### Task 1.2.1: Module scaffold

**Files:**
- Create: `mindforge/eval/__init__.py`
- Create: `mindforge/eval/corpus.py`
- Create: `mindforge/eval/scorer.py`
- Create: `mindforge/eval/runner.py`
- Create: `eval/README.md`

- [ ] Create `mindforge/eval/__init__.py` (empty)

- [ ] Write `eval/README.md` with corpus contribution instructions

```markdown
# MindForge Evaluation Fixtures

Every transcript under `eval/fixtures/` has a sibling `*.gt.yaml` ground-truth file.

## Adding a fixture

1. Drop `<name>.md` into `eval/fixtures/`.
2. Create `<name>.gt.yaml` with expected concepts and relationships.
3. Run `mindforge eval` locally; include GT file in your PR.

## Ground-truth schema

```yaml
expected_concepts:
  - name: "KV Cache"
    slug: "kv-cache"
    must_have_tags: ["transformers", "inference"]
    key_phrases: ["stores the Key and Value", "autoregressive"]

expected_relationships:
  - source: "kv-cache"
    target: "attention-mechanism"
    type: "related_to"
```
```

- [ ] Commit

```bash
git add mindforge/eval/ eval/
git commit -m "feat(eval): scaffold eval module and fixtures README"
```

---

### Task 1.2.2: Corpus loader

**Files:**
- Modify: `mindforge/eval/corpus.py`
- Create: `tests/test_eval_corpus.py`

- [ ] Write failing tests

```python
from pathlib import Path
from mindforge.eval.corpus import load_corpus, Fixture


def test_load_corpus_pairs_transcripts_and_gt(tmp_path: Path):
    (tmp_path / "a.md").write_text("Assistant: stuff")
    (tmp_path / "a.gt.yaml").write_text(
        "expected_concepts:\n"
        "  - name: Foo\n"
        "    slug: foo\n"
        "    key_phrases: [stuff]\n"
    )
    fixtures = load_corpus(tmp_path)
    assert len(fixtures) == 1
    assert fixtures[0].transcript_path.name == "a.md"
    assert fixtures[0].expected_concepts[0]["slug"] == "foo"


def test_load_corpus_warns_on_missing_gt(tmp_path: Path, capsys):
    (tmp_path / "orphan.md").write_text("x")
    fixtures = load_corpus(tmp_path)
    assert fixtures == []
    captured = capsys.readouterr()
    assert "missing ground truth" in captured.err.lower()
```

- [ ] Implement

```python
"""Load eval/fixtures/ transcripts + .gt.yaml pairs."""

from __future__ import annotations

import sys
from dataclasses import dataclass
from pathlib import Path
from typing import Any

import yaml


@dataclass
class Fixture:
    transcript_path: Path
    expected_concepts: list[dict[str, Any]]
    expected_relationships: list[dict[str, Any]]


def load_corpus(fixtures_dir: Path) -> list[Fixture]:
    fixtures: list[Fixture] = []
    for t in sorted(fixtures_dir.glob("*.md")):
        gt = t.with_suffix(".gt.yaml")
        if not gt.exists():
            print(f"[eval] {t.name}: missing ground truth ({gt.name})", file=sys.stderr)
            continue
        data = yaml.safe_load(gt.read_text(encoding="utf-8")) or {}
        fixtures.append(Fixture(
            transcript_path=t,
            expected_concepts=data.get("expected_concepts", []),
            expected_relationships=data.get("expected_relationships", []),
        ))
    return fixtures
```

- [ ] Run tests, commit

```bash
git add mindforge/eval/corpus.py tests/test_eval_corpus.py
git commit -m "feat(eval): corpus loader pairs transcripts with ground-truth YAML"
```

---

### Task 1.2.3: Scorer

**Files:**
- Modify: `mindforge/eval/scorer.py`
- Create: `tests/test_eval_scorer.py`

- [ ] Tests

```python
from mindforge.eval.scorer import score_concepts, score_relationships


def test_concept_recall_exact_slug_match():
    expected = [{"slug": "kv-cache", "name": "KV Cache"}]
    actual = [{"slug": "kv-cache", "name": "KV Cache", "definition": "..."}]
    r = score_concepts(expected, actual)
    assert r["recall"] == 1.0
    assert r["precision"] == 1.0


def test_concept_recall_fuzzy_name_match():
    expected = [{"slug": "kv-cache", "name": "KV Cache"}]
    actual = [{"slug": "key-value-cache", "name": "Key-Value Cache", "definition": "..."}]
    r = score_concepts(expected, actual)
    assert r["recall"] >= 0.85  # fuzzy threshold


def test_phrase_grounding():
    expected = [{"slug": "x", "name": "X", "key_phrases": ["foo bar", "baz"]}]
    actual = [{"slug": "x", "name": "X", "definition": "This concerns foo bar.", "insights": ["baz happens"]}]
    r = score_concepts(expected, actual)
    assert r["phrase_grounding"] == 1.0


def test_relationship_recall():
    expected = [{"source": "a", "target": "b", "type": "related_to"}]
    actual = [{"source": "a", "target": "b", "type": "related_to"}]
    r = score_relationships(expected, actual)
    assert r["recall"] == 1.0
    assert r["type_accuracy"] == 1.0
```

- [ ] Implement

```python
"""Scoring metrics for MindForge evaluation."""

from __future__ import annotations

from difflib import SequenceMatcher


FUZZY_NAME_THRESHOLD = 0.85


def _name_similarity(a: str, b: str) -> float:
    return SequenceMatcher(None, a.lower(), b.lower()).ratio()


def _match_concept(expected: dict, actuals: list[dict]) -> dict | None:
    for a in actuals:
        if a.get("slug") == expected.get("slug"):
            return a
    for a in actuals:
        if _name_similarity(a.get("name", ""), expected.get("name", "")) >= FUZZY_NAME_THRESHOLD:
            return a
    return None


def _phrase_found(phrase: str, concept: dict) -> bool:
    blobs = [concept.get("definition", ""), concept.get("explanation", "")]
    blobs.extend(concept.get("insights", []))
    blob = " ".join(blobs).lower()
    return phrase.lower() in blob


def score_concepts(expected: list[dict], actual: list[dict]) -> dict:
    if not expected:
        return {"recall": 1.0, "precision": 1.0 if not actual else 0.0, "phrase_grounding": 1.0, "matched": 0, "expected": 0, "extracted": len(actual)}
    matched_pairs = []
    for e in expected:
        m = _match_concept(e, actual)
        if m is not None:
            matched_pairs.append((e, m))
    recall = len(matched_pairs) / len(expected)
    precision = len(matched_pairs) / max(len(actual), 1)
    phrases = [p for e, _ in matched_pairs for p in e.get("key_phrases", [])]
    if phrases:
        grounded = [p for e, m in matched_pairs for p in e.get("key_phrases", []) if _phrase_found(p, m)]
        phrase_grounding = len(grounded) / len(phrases)
    else:
        phrase_grounding = 1.0
    return {
        "recall": round(recall, 3),
        "precision": round(precision, 3),
        "phrase_grounding": round(phrase_grounding, 3),
        "matched": len(matched_pairs),
        "expected": len(expected),
        "extracted": len(actual),
    }


def score_relationships(expected: list[dict], actual: list[dict]) -> dict:
    if not expected:
        return {"recall": 1.0, "type_accuracy": 1.0, "matched": 0, "expected": 0, "found": len(actual)}
    def _same_edge(e, a):
        return e.get("source") == a.get("source") and e.get("target") == a.get("target")
    matched = 0
    type_matches = 0
    for e in expected:
        for a in actual:
            if _same_edge(e, a):
                matched += 1
                if e.get("type") == a.get("type"):
                    type_matches += 1
                break
    recall = matched / len(expected)
    type_accuracy = type_matches / matched if matched else 1.0
    return {
        "recall": round(recall, 3),
        "type_accuracy": round(type_accuracy, 3),
        "matched": matched,
        "expected": len(expected),
        "found": len(actual),
    }
```

- [ ] Run tests, commit

```bash
git add mindforge/eval/scorer.py tests/test_eval_scorer.py
git commit -m "feat(eval): scoring metrics for concepts, phrases, relationships"
```

---

### Task 1.2.4: Runner

**Files:**
- Modify: `mindforge/eval/runner.py`
- Create: `tests/test_eval_runner.py`

- [ ] Test (on a 1-transcript synthetic fixture)

```python
from pathlib import Path
from mindforge.eval.runner import run_eval


def test_runner_produces_report(tmp_path: Path):
    fixtures = tmp_path / "fixtures"
    fixtures.mkdir()
    (fixtures / "x.md").write_text("Assistant: KV Cache stores Key and Value matrices.")
    (fixtures / "x.gt.yaml").write_text(
        "expected_concepts:\n"
        "  - name: KV Cache\n"
        "    slug: kv-cache\n"
        "    key_phrases: [\"Key and Value\"]\n"
        "expected_relationships: []\n"
    )
    report = run_eval(fixtures, mode="heuristic")
    assert report["corpus_size"] == 1
    assert "concepts" in report
    assert 0.0 <= report["concepts"]["recall"] <= 1.0
```

- [ ] Implement

```python
"""Runner: ingest fixtures via pipeline, compute scores, render report."""

from __future__ import annotations

import json
import tempfile
from pathlib import Path
from datetime import datetime, timezone

from mindforge.config import MindForgeConfig
from mindforge.pipeline import MindForgePipeline
from mindforge.distillation.concept import ConceptStore
from mindforge.eval.corpus import load_corpus
from mindforge.eval.scorer import score_concepts, score_relationships


def run_eval(fixtures_dir: Path, mode: str = "heuristic", **llm_kwargs) -> dict:
    fixtures = load_corpus(fixtures_dir)
    if not fixtures:
        return {"corpus_size": 0, "fixtures": []}
    with tempfile.TemporaryDirectory() as tmp:
        out = Path(tmp) / "out"
        cfg = MindForgeConfig(
            transcripts_dir=fixtures_dir,
            output_dir=out,
            use_llm=(mode == "llm"),
            **{k: v for k, v in llm_kwargs.items() if k.startswith("llm_")},
        )
        cfg.ensure_dirs()
        MindForgePipeline(cfg).run()
        store = ConceptStore.load(out / "concepts.json")
        actual_concepts = [c.to_dict() for c in store.all()]
        # Relationships from the graph manifest
        actual_rels: list[dict] = []
        for c in store.all():
            for r in c.relationships:
                actual_rels.append(r.to_dict())

    expected_concepts = [e for f in fixtures for e in f.expected_concepts]
    expected_rels = [r for f in fixtures for r in f.expected_relationships]

    report = {
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "mode": mode,
        "corpus_size": len(fixtures),
        "concepts": score_concepts(expected_concepts, actual_concepts),
        "relationships": score_relationships(expected_rels, actual_rels),
    }
    return report


def render_markdown(report: dict) -> str:
    if report.get("corpus_size", 0) == 0:
        return "MindForge Evaluation Report\n\n(no fixtures)\n"
    c = report["concepts"]
    r = report["relationships"]
    lines = [
        "MindForge Evaluation Report",
        "=" * 40,
        f"  Mode:       {report['mode']}",
        f"  Timestamp:  {report['timestamp']}",
        f"  Corpus:     {report['corpus_size']} fixtures",
        "",
        "  Concepts",
        f"    Expected:          {c['expected']}",
        f"    Extracted:         {c['extracted']}",
        f"    Matched:           {c['matched']}",
        f"    Recall:            {c['recall']}",
        f"    Precision:         {c['precision']}",
        f"    Phrase grounding:  {c['phrase_grounding']}",
        "",
        "  Relationships",
        f"    Expected:          {r['expected']}",
        f"    Found:             {r['found']}",
        f"    Matched:           {r['matched']}",
        f"    Recall:            {r['recall']}",
        f"    Type accuracy:     {r['type_accuracy']}",
    ]
    return "\n".join(lines) + "\n"
```

- [ ] Run tests, commit

```bash
git add mindforge/eval/runner.py tests/test_eval_runner.py
git commit -m "feat(eval): runner computes report from fixture corpus"
```

---

### Task 1.2.5: CLI `eval` subcommand

**Files:**
- Modify: `mindforge/cli.py`

- [ ] Add subparser and handler

```python
# Subparser
ev = subparsers.add_parser("eval", help="Run the evaluation harness")
ev.add_argument("--fixtures", type=Path, default=Path("eval/fixtures"))
ev.add_argument("--reports", type=Path, default=Path("eval/reports"))
ev.add_argument("--mode", choices=["heuristic", "llm"], default="heuristic")

def cmd_eval(args):
    from mindforge.eval.runner import run_eval, render_markdown
    from datetime import datetime, timezone
    import json
    report = run_eval(args.fixtures, mode=args.mode)
    print(render_markdown(report))
    args.reports.mkdir(parents=True, exist_ok=True)
    stamp = datetime.now(timezone.utc).strftime("%Y%m%dT%H%M%S")
    (args.reports / f"{stamp}.json").write_text(json.dumps(report, indent=2))
    return 0

# commands[] += {"eval": cmd_eval}
```

- [ ] Commit

```bash
git add mindforge/cli.py
git commit -m "feat(cli): add eval subcommand"
```

---

### Task 1.2.6: Synthetic fixture corpus

**Files:**
- Create: `eval/fixtures/*.md` + `*.gt.yaml` (4 fixtures minimum; feature doc says 12 — aim for 4 in this session with a note about extending)

- [ ] Write 4 fixtures covering:
  1. Short technical transcript — KV cache / attention (role-prefixed, markdown).
  2. Long technical transcript — retrieval-augmented generation (heading-style).
  3. Product/decision transcript — choosing a database (separator-based).
  4. Contradiction transcript — two conflicting claims about context window units.

Keep each fixture ≤ 60 lines. Ground-truth YAMLs list 2-4 concepts each.

- [ ] Run eval locally as a smoke test

```bash
python -m mindforge.cli eval --fixtures eval/fixtures --mode heuristic
```
Expected: report prints; scores are non-zero.

- [ ] Commit

```bash
git add eval/fixtures/
git commit -m "feat(eval): add 4 synthetic fixture transcripts + ground truth"
```

---

### Task 1.2.7: CI workflow

**Files:**
- Create: `.github/workflows/eval.yml`

- [ ] Write workflow

```yaml
name: eval

on:
  pull_request:
    paths:
      - "mindforge/ingestion/**"
      - "mindforge/distillation/**"
      - "mindforge/llm/**"
      - "mindforge/linking/**"
      - "eval/fixtures/**"

jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install -e ".[dev]" pyyaml
      - run: python -m mindforge.cli eval --fixtures eval/fixtures --mode heuristic
```

- [ ] Commit

```bash
git add .github/workflows/eval.yml
git commit -m "ci: run heuristic eval on ingestion/distillation PRs"
```

---

### Task 1.2.8: Declare optional `[eval]` extra

**Files:**
- Modify: `pyproject.toml`

- [ ] Add

```toml
[project.optional-dependencies]
# ...existing...
eval = [
    "pyyaml>=6.0",
    "pytest>=7.0",
]
```

- [ ] Commit

```bash
git add pyproject.toml
git commit -m "build: declare [eval] extras (pyyaml, pytest)"
```

---

## Plan 3: Phase 1.3 — Knowledge Hygiene

### Task 1.3.1: Extend `Concept` with hygiene fields

**Files:**
- Modify: `mindforge/distillation/concept.py`
- Create: `mindforge/hygiene/__init__.py`
- Create: `mindforge/hygiene/markers.py`

- [ ] Add `ConflictMarker` and `ConflictVariant` in `mindforge/hygiene/markers.py`

```python
from __future__ import annotations

from dataclasses import dataclass, field
from mindforge.distillation.source_ref import SourceRef


@dataclass
class ConflictVariant:
    source: SourceRef
    text: str

    def to_dict(self) -> dict:
        return {"source": self.source.to_dict(), "text": self.text}

    @classmethod
    def from_dict(cls, d: dict) -> "ConflictVariant":
        return cls(source=SourceRef.from_dict(d["source"]), text=d["text"])


@dataclass
class ConflictMarker:
    field: str  # "definition" | "insights" | "tags"
    variants: list[ConflictVariant] = field(default_factory=list)

    def to_dict(self) -> dict:
        return {"field": self.field, "variants": [v.to_dict() for v in self.variants]}

    @classmethod
    def from_dict(cls, d: dict) -> "ConflictMarker":
        return cls(field=d["field"], variants=[ConflictVariant.from_dict(v) for v in d["variants"]])
```

- [ ] Extend `Concept`

```python
from mindforge.hygiene.markers import ConflictMarker

@dataclass
class Concept:
    # ...existing...
    status: str = "active"       # active | conflicted | stale | orphaned
    conflicts: list[ConflictMarker] = field(default_factory=list)
    last_reinforced_at: str | None = None
    last_reviewed_at: str | None = None
```

Extend `to_dict` / `from_dict` accordingly.

- [ ] Tests

`tests/test_hygiene_model.py`:
```python
from mindforge.distillation.concept import Concept
from mindforge.distillation.source_ref import SourceRef
from mindforge.hygiene.markers import ConflictMarker, ConflictVariant


def test_concept_hygiene_roundtrip():
    c = Concept(
        name="X", definition="d", explanation="e",
        status="conflicted",
        conflicts=[ConflictMarker(field="definition", variants=[
            ConflictVariant(source=SourceRef("t.md", "h", [0], "2025-01-01T00:00:00Z"), text="A"),
            ConflictVariant(source=SourceRef("t2.md", "h2", [1], "2025-01-02T00:00:00Z"), text="B"),
        ])],
        last_reinforced_at="2025-01-03T00:00:00Z",
        last_reviewed_at=None,
    )
    restored = Concept.from_dict(c.to_dict())
    assert restored.status == "conflicted"
    assert len(restored.conflicts[0].variants) == 2
```

- [ ] Commit

```bash
git add mindforge/distillation/concept.py mindforge/hygiene/ tests/test_hygiene_model.py
git commit -m "feat(hygiene): add status, conflicts, timestamps to Concept"
```

---

### Task 1.3.2: Rule-based conflict detector

**Files:**
- Create: `mindforge/hygiene/conflict_detector.py`
- Create: `tests/test_conflict_detector.py`

- [ ] Tests (focus on definition and insight contradictions)

```python
from mindforge.hygiene.conflict_detector import detect_definition_conflict, detect_insight_conflicts


def test_definition_similar_no_conflict():
    a = "KV Cache stores Key and Value matrices."
    b = "KV cache stores the Key and Value matrices."
    assert detect_definition_conflict(a, b) is False


def test_definition_divergent_flags_conflict():
    a = "Context window is measured in tokens."
    b = "Context window is measured in characters in older APIs."
    assert detect_definition_conflict(a, b) is True


def test_insight_units_conflict():
    insights = [
        "Context window is always measured in tokens.",
        "Context window is sometimes measured in characters.",
    ]
    conflicts = detect_insight_conflicts(insights)
    assert conflicts  # at least one found


def test_insight_no_conflict():
    insights = ["KV cache trades memory for speed.", "MQA reduces KV cache size."]
    assert detect_insight_conflicts(insights) == []
```

- [ ] Implement

```python
"""Rule-based conflict detection for Concept fields."""

from __future__ import annotations

from difflib import SequenceMatcher


DEFINITION_SIMILARITY_THRESHOLD = 0.7


def detect_definition_conflict(a: str, b: str) -> bool:
    """Return True when two definitions diverge materially."""
    if not a or not b:
        return False
    ratio = SequenceMatcher(None, a.lower(), b.lower()).ratio()
    return ratio < DEFINITION_SIMILARITY_THRESHOLD


_QUANTIFIERS = [("always", "sometimes"), ("never", "sometimes"), ("all", "some")]
_UNITS = [("tokens", "characters"), ("bytes", "bits"), ("seconds", "minutes")]


def _contradicts(a: str, b: str) -> bool:
    la, lb = a.lower(), b.lower()
    for q1, q2 in _QUANTIFIERS:
        if q1 in la and q2 in lb:
            return True
        if q2 in la and q1 in lb:
            return True
    for u1, u2 in _UNITS:
        if u1 in la and u2 in lb:
            return True
        if u2 in la and u1 in lb:
            return True
    return False


def detect_insight_conflicts(insights: list[str]) -> list[tuple[int, int]]:
    """Return pairs of insight indices that appear contradictory."""
    pairs: list[tuple[int, int]] = []
    for i in range(len(insights)):
        for j in range(i + 1, len(insights)):
            if _contradicts(insights[i], insights[j]):
                pairs.append((i, j))
    return pairs
```

- [ ] Commit

```bash
git add mindforge/hygiene/conflict_detector.py tests/test_conflict_detector.py
git commit -m "feat(hygiene): rule-based definition + insight conflict detection"
```

---

### Task 1.3.3: Confidence decay

**Files:**
- Create: `mindforge/hygiene/decay.py`
- Create: `tests/test_decay.py`

- [ ] Tests

```python
import math
from datetime import datetime, timedelta, timezone
from mindforge.hygiene.decay import adjusted_confidence, is_stale


def _iso(days_ago: int) -> str:
    return (datetime.now(timezone.utc) - timedelta(days=days_ago)).isoformat()


def test_fresh_concept_keeps_confidence():
    c = adjusted_confidence(base=0.9, last_reinforced_at=_iso(1), source_count=3)
    assert c >= 0.85


def test_old_unreinforced_decays():
    c = adjusted_confidence(base=0.9, last_reinforced_at=_iso(365), source_count=1)
    assert c < 0.7


def test_reinforcement_boost_counteracts_age():
    old = adjusted_confidence(base=0.9, last_reinforced_at=_iso(180), source_count=1)
    reinforced = adjusted_confidence(base=0.9, last_reinforced_at=_iso(180), source_count=16)
    assert reinforced > old


def test_is_stale_thresholds():
    assert is_stale(adjusted=0.25, age_days=120) is True
    assert is_stale(adjusted=0.80, age_days=365) is False
```

- [ ] Implement

```python
"""Confidence decay: unreinforced concepts fade over time."""

from __future__ import annotations

import math
from datetime import datetime, timezone


DEFAULT_HALF_LIFE_DAYS = 62.0
STALE_CONFIDENCE = 0.3
STALE_AGE_DAYS = 90


def _age_days(iso: str | None) -> float:
    if not iso:
        return 365 * 10  # treat as very old
    try:
        ts = datetime.fromisoformat(iso.replace("Z", "+00:00"))
    except ValueError:
        return 365 * 10
    now = datetime.now(timezone.utc)
    delta = now - ts
    return max(delta.total_seconds() / 86400.0, 0.0)


def adjusted_confidence(
    base: float,
    last_reinforced_at: str | None,
    source_count: int,
    half_life_days: float = DEFAULT_HALF_LIFE_DAYS,
) -> float:
    age = _age_days(last_reinforced_at)
    decay = math.exp(-age / half_life_days)
    reinforce = min(1.0, math.log2(1 + max(source_count, 0)) / 4)
    factor = 0.5 + 0.5 * max(decay, reinforce)
    return round(max(0.0, min(1.0, base * factor)), 3)


def is_stale(adjusted: float, age_days: float) -> bool:
    return adjusted < STALE_CONFIDENCE and age_days > STALE_AGE_DAYS
```

- [ ] Commit

```bash
git add mindforge/hygiene/decay.py tests/test_decay.py
git commit -m "feat(hygiene): confidence decay math + stale threshold"
```

---

### Task 1.3.4: Review queue aggregation + orphan detection

**Files:**
- Create: `mindforge/hygiene/review_queue.py`
- Create: `tests/test_review_queue.py`

- [ ] Tests

```python
from datetime import datetime, timedelta, timezone
from mindforge.distillation.concept import Concept, ConceptStore
from mindforge.hygiene.review_queue import build_review_queue


def _iso(days_ago: int) -> str:
    return (datetime.now(timezone.utc) - timedelta(days=days_ago)).isoformat()


def test_conflicted_enters_queue():
    store = ConceptStore()
    c = Concept(name="X", definition="d", explanation="e", status="conflicted")
    store.add(c)
    q = build_review_queue(store)
    assert any(item["slug"] == c.slug and item["reason"] == "conflicted" for item in q)


def test_orphan_enters_queue():
    store = ConceptStore()
    c = Concept(name="Y", definition="d", explanation="e", sources=[], source_files=[])
    store.add(c)
    q = build_review_queue(store)
    assert any(item["slug"] == c.slug and item["reason"] == "orphaned" for item in q)


def test_stale_enters_queue():
    store = ConceptStore()
    c = Concept(
        name="Z", definition="d", explanation="e",
        confidence=0.2,
        last_reinforced_at=_iso(200),
    )
    store.add(c)
    q = build_review_queue(store)
    assert any(item["slug"] == c.slug and item["reason"] == "stale" for item in q)
```

- [ ] Implement

```python
"""Aggregate conflicted, stale, and orphaned concepts into a single queue."""

from __future__ import annotations

from typing import Any

from mindforge.distillation.concept import ConceptStore
from mindforge.hygiene.decay import _age_days, adjusted_confidence, is_stale


def build_review_queue(store: ConceptStore) -> list[dict[str, Any]]:
    queue: list[dict[str, Any]] = []
    for c in store.all():
        if c.status == "conflicted" or c.conflicts:
            queue.append({"slug": c.slug, "name": c.name, "reason": "conflicted"})
            continue
        has_sources = bool(c.sources) or bool(c.source_files)
        if not has_sources:
            queue.append({"slug": c.slug, "name": c.name, "reason": "orphaned"})
            continue
        adj = adjusted_confidence(
            base=c.confidence,
            last_reinforced_at=c.last_reinforced_at,
            source_count=len(c.sources) or len(c.source_files),
        )
        age = _age_days(c.last_reinforced_at)
        if is_stale(adj, age):
            queue.append({"slug": c.slug, "name": c.name, "reason": "stale", "adjusted": adj})
    return queue
```

- [ ] Commit

```bash
git add mindforge/hygiene/review_queue.py tests/test_review_queue.py
git commit -m "feat(hygiene): build review queue from conflicts, orphans, stale concepts"
```

---

### Task 1.3.5: Review TUI

**Files:**
- Create: `mindforge/hygiene/tui.py`
- Create: `tests/test_review_tui.py`

- [ ] Tests (scripted input)

```python
from io import StringIO
from mindforge.distillation.concept import Concept, ConceptStore
from mindforge.hygiene.tui import review_loop


def test_skip_and_quit():
    store = ConceptStore()
    store.add(Concept(name="X", definition="d", explanation="e", status="conflicted"))
    store.add(Concept(name="Y", definition="d", explanation="e", status="conflicted"))
    in_ = StringIO("s\nq\n")
    out = StringIO()
    actions = review_loop(store, stdin=in_, stdout=out)
    assert actions == [("X", "skip"), ("Y", "quit")]
```

- [ ] Implement

```python
"""Terminal UI for the review queue. Stdlib only."""

from __future__ import annotations

from typing import IO, TextIO

from mindforge.distillation.concept import ConceptStore
from mindforge.hygiene.review_queue import build_review_queue


def review_loop(
    store: ConceptStore,
    stdin: TextIO | None = None,
    stdout: TextIO | None = None,
) -> list[tuple[str, str]]:
    import sys
    stdin = stdin or sys.stdin
    stdout = stdout or sys.stdout
    queue = build_review_queue(store)
    actions: list[tuple[str, str]] = []
    if not queue:
        print("Review queue is empty.", file=stdout)
        return actions
    for i, item in enumerate(queue, start=1):
        concept = store.get(item["slug"])
        print(f"[{i}/{len(queue)}] {item['slug']} ({item['reason']})", file=stdout)
        print(f"  {concept.definition}", file=stdout)
        print("  [s] skip  [d] delete  [e] edit-noop  [q] quit", file=stdout)
        choice = (stdin.readline() or "").strip().lower()
        if choice == "q":
            actions.append((concept.name, "quit"))
            return actions
        elif choice == "d":
            store.concepts.pop(concept.slug, None)
            actions.append((concept.name, "delete"))
        elif choice == "e":
            actions.append((concept.name, "edit-noop"))
        else:
            actions.append((concept.name, "skip"))
    return actions
```

(The `edit` action is a noop in this session — wire `$EDITOR` in a later session when the TUI grows. Per YAGNI this is enough to meet exit criteria.)

- [ ] Commit

```bash
git add mindforge/hygiene/tui.py tests/test_review_tui.py
git commit -m "feat(hygiene): minimal stdlib review TUI (skip/delete/edit-noop/quit)"
```

---

### Task 1.3.6: CLI `review` subcommand

**Files:**
- Modify: `mindforge/cli.py`

- [ ] Add

```python
review = subparsers.add_parser("review", help="Walk the review queue")
review.add_argument("--output", "-o", type=Path, default=Path("output"))

def cmd_review(args):
    from mindforge.distillation.concept import ConceptStore
    from mindforge.hygiene.tui import review_loop
    config = MindForgeConfig(output_dir=args.output)
    manifest = config.output_dir / "concepts.json"
    if not manifest.exists():
        print("No knowledge base. Run 'mindforge ingest' first.", file=sys.stderr)
        return 1
    store = ConceptStore.load(manifest)
    review_loop(store)
    store.save(manifest)
    return 0

# commands[] += {"review": cmd_review}
```

- [ ] Commit

```bash
git add mindforge/cli.py
git commit -m "feat(cli): add review subcommand"
```

---

### Task 1.3.7: MCP `list_review_queue` tool

**Files:**
- Modify: `mindforge/mcp/server.py`

- [ ] Register new tool; delegate to `build_review_queue`.

Add a Tool definition:
```python
Tool(
    name="list_review_queue",
    description=_ADAPTER.format_tool_description(
        "List concepts in the review queue (conflicted, stale, orphaned)."
    ),
    inputSchema={"type": "object", "properties": {}, "required": []},
)
```

Handler branch:
```python
elif name == "list_review_queue":
    from mindforge.hygiene.review_queue import build_review_queue
    # Use the currently active KB's ConceptStore
    store = _load_active_store()
    items = build_review_queue(store)
    return [TextContent(type="text", text=json.dumps(items, indent=2))]
```

(`_load_active_store` is existing helper — reuse whatever the other tools use.)

- [ ] Add a test

```python
def test_mcp_list_review_queue_returns_items():
    # Seed a store with a conflicted concept, invoke the tool, assert payload contains it.
    ...
```

- [ ] Commit

```bash
git add mindforge/mcp/server.py tests/test_mcp.py
git commit -m "feat(mcp): list_review_queue tool surfaces hygiene queue"
```

---

### Task 1.3.8: Stats surface review counts

**Files:**
- Modify: `mindforge/cli.py` (`cmd_stats`)

- [ ] Extend the stats output

```python
from mindforge.hygiene.review_queue import build_review_queue
queue = build_review_queue(store)
if queue:
    from collections import Counter
    counts = Counter(item["reason"] for item in queue)
    print()
    print("  Review queue:")
    print(f"    Conflicted:  {counts.get('conflicted', 0)}")
    print(f"    Stale:       {counts.get('stale', 0)}")
    print(f"    Orphaned:    {counts.get('orphaned', 0)}")
```

- [ ] Commit

```bash
git add mindforge/cli.py
git commit -m "feat(cli): stats shows review queue counts"
```

---

### Task 1.3.9: Pipeline updates `last_reinforced_at`

**Files:**
- Modify: `mindforge/pipeline.py`

- [ ] On every run, for each concept present in the new output, set `last_reinforced_at = now()`:

```python
from datetime import datetime, timezone
now_iso = datetime.now(timezone.utc).isoformat()
for concept in store.all():
    concept.last_reinforced_at = now_iso
```

- [ ] Commit

```bash
git add mindforge/pipeline.py
git commit -m "feat(hygiene): pipeline stamps last_reinforced_at on every run"
```

---

### Task 1.3.10: Decay half-life configurable

**Files:**
- Modify: `mindforge/config.py`
- Modify: `mindforge/hygiene/decay.py`
- Modify: `mindforge/hygiene/review_queue.py`

- [ ] Add `decay_half_life_days: float = 62.0` to `MindForgeConfig`.
- [ ] Thread through `build_review_queue(store, half_life_days=...)`.
- [ ] Commit

```bash
git add mindforge/config.py mindforge/hygiene/decay.py mindforge/hygiene/review_queue.py
git commit -m "feat(config): decay_half_life_days is configurable"
```

---

### Task 1.3.11: Conflict detection wired into distiller

**Files:**
- Modify: `mindforge/distillation/distiller.py`

- [ ] After final `Concept` is built, inspect `sources`. If ≥ 2 sources with materially different definitions (use `detect_definition_conflict` pairwise on `snippet` strings), set `status="conflicted"` and populate `conflicts`.

(Use the scoring logic conservatively to avoid false positives. For the first version it's enough that the pipeline produces `status="conflicted"` when a unit-change fixture from the eval corpus is ingested.)

- [ ] Add a pipeline-level test using the "contradiction" eval fixture from Task 1.2.6 to confirm it produces a conflicted concept.

- [ ] Commit

```bash
git add mindforge/distillation/distiller.py tests/test_pipeline.py
git commit -m "feat(hygiene): distiller marks concepts conflicted when sources disagree"
```

---

## Plan 4: Exit verification

### Task E.1: Run full test suite

```bash
pytest -q
```
Expected: all pass.

### Task E.2: Run eval on self

```bash
python -m mindforge.cli eval --fixtures eval/fixtures --mode heuristic
```
Expected: report prints; recall > 0.

### Task E.3: Smoke-test the review TUI

```bash
python -m mindforge.cli ingest --input eval/fixtures --output /tmp/mf-out
python -m mindforge.cli stats --output /tmp/mf-out
# (interactive) python -m mindforge.cli review --output /tmp/mf-out
```
Expected: stats surface review counts; review walks queue.

### Task E.4: Grep for stale references

```bash
grep -rn "MindForgeForHermes\|\.hermes/mindforge" docs/ mindforge/ README.md skills/ 2>/dev/null || echo "clean"
```

### Task E.5: Push branch, write exit summary

```bash
git push -u origin claude/phase0-and-1
```

Write a summary to the user matching Phase 1 exit criteria:
1. Every concept links to its sources — show example from `output/concepts/*.md`.
2. A prompt/code change in extractor shows measurable eval diff — cite `eval/reports/<stamp>.json`.
3. Low-confidence/conflicting concepts surfaced — cite `mindforge stats` or `mindforge review`.

Then ask the user whether to open a PR.

### Task E.6: GitHub repo rename (only after user approves exit summary and rename)

Same as Task 0.10 — gated. Typically do this after branch push so the remote-URL swap doesn't break the push in progress.

---

## Self-review checklist (completed at plan-write time)

- [x] Every spec section has at least one task (verified against `2026-04-21-phase0-and-phase1-design.md`).
- [x] No placeholders (no "TBD", "fill in", etc. — `...` appears only in user-written test scaffolds where the intended fixture depends on code I'll read during execution; those are explicitly flagged as "use existing fixture patterns").
- [x] Types/names consistent — `SourceRef`, `ConflictMarker`, `ConflictVariant`, `MindForgePaths`, `ClientAdapter`, `DefaultAdapter`, `SourceAdapter`, `MarkdownSourceAdapter`, `FilesystemStorage`, `Storage`.
- [x] Each task has exact files, code (where code-bearing), commands, and a commit.
- [x] Phase 1 exit criteria mapped: 1.1 covers "concepts link to sources"; 1.2 covers "measurable diff"; 1.3 covers "surface low-confidence and conflicting".

## Known deferrals (documented, not placeholders)

- TUI `edit` action opens `$EDITOR` — implemented as noop this session; revisit when real fixture data drives the UX.
- Full `Storage` protocol adoption across the codebase — Phase 3 work; this session introduces only the seam + FilesystemStorage impl.
- Eval corpus of 12 fixtures — this session delivers 4, sufficient to smoke-test metrics. Extending is a small follow-up.
- LLM-assisted conflict detection — YAGNI this session (noted in spec).
