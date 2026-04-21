# MindForge Knowledge Base Manager

**Integration:** MindForge MCP Server for multi-knowledgebase research management
**Repository:** https://github.com/AcceleratedIndustries/MindForgeForHermes

---

## Overview

MindForge transforms unstructured content (transcripts, papers, conversations) into structured, queryable knowledge bases with semantic search and relationship graphs. This skill configures the MCP server and provides workflows for managing topic-based knowledge bases.

---

## Configuration

The MCP server is configured in `~/.hermes/config.yaml`:

```yaml
mcp_servers:
  mindforge:
    command: /home/will/.hermes/hermes-agent/venv/bin/python
    args:
    - -m
    - mindforge.mcp.server
    env:
      PYTHONPATH: /home/will/MindForge
      MINDFORGE_ROOT: /home/will/.hermes/mindforge
    timeout: 60
    connect_timeout: 30
```

**Storage location:** `~/.hermes/mindforge/kbs/`

---

## Quick Start

### 1. List Your Knowledge Bases

```
User: What KBs do I have?
→ Calls: mcp_mindforge_kb_list
```

**Sample output:**
```json
[
  {
    "id": "foundations-of-ai",
    "name": "Foundations of AI",
    "concept_count": 112,
    "is_active": false
  },
  {
    "id": "hermes-agent-transcripts",
    "name": "Hermes Agent Transcripts",
    "concept_count": 7,
    "is_active": true
  }
]
```

### 2. Create a New KB

```
User: Create a KB for "Quantum Computing" research
→ Calls: mcp_mindforge_kb_create(name="Quantum Computing")
```

**Result:** Creates `~/.hermes/mindforge/kbs/quantum-computing/`

### 3. Switch Active KB

```
User: Switch to quantum-computing
→ Calls: mcp_mindforge_kb_select(id="quantum-computing")
```

### 4. Search

```
User: Search for "entanglement"
→ Calls: mcp_mindforge_search(query="entanglement")
```

---

## Complete Tool Reference

### Knowledge Base Management

| Tool | Purpose | Required Args |
|------|---------|---------------|
| `mcp_mindforge_kb_list` | List all KBs with metadata | — |
| `mcp_mindforge_kb_create` | Create new KB | `name` |
| `mcp_mindforge_kb_select` | Activate a KB | `id` |
| `mcp_mindforge_kb_get_current` | Show active KB | — |
| `mcp_mindforge_kb_rename` | Rename KB | `old_id`, `new_name` |
| `mcp_mindforge_kb_delete` | Move to trash | `id` |

### Search Modes

| Tool | Scope | Use When |
|------|-------|----------|
| `mcp_mindforge_search` | **Active KB only** | Focused research on current topic |
| `mcp_mindforge_search_all` | **All KBs** | Cross-cutting concepts |
| `mcp_mindforge_search_selected` | **Specific KBs** | Compare topics |

### Concept Operations (Active KB)

| Tool | Purpose |
|------|---------|
| `mcp_mindforge_get_concept` | Full concept details by name/slug |
| `mcp_mindforge_list_concepts` | All concepts (opt: tag filter) |
| `mcp_mindforge_get_neighbors` | Related concepts via graph |
| `mcp_mindforge_get_stats` | KB statistics |

---

## Naming Convention

KB IDs use **kebab-case** (auto-converted):
- "Neural Networks" → `neural-networks`
- "MLOps Best Practices" → `mlops-best-practices`
- "2024 Research Notes" → `2024-research-notes`

---

## Workflows

### Research Project Setup

```
1. Create KB for the project
   → kb_create(name="Project Alpha Research")

2. Select as active
   → kb_select(id="project-alpha-research")

3. Start adding concepts (via ingestion or manual)

4. Search within project scope
   → search(query="relevant concept")
```

### Cross-Project Discovery

```
User: Has "attention mechanism" appeared in any of my KBs?
→ search_all(query="attention mechanism")

User: Compare AI papers vs my transcripts on "optimization"
→ search_selected(
     query="optimization",
     kb_ids=["foundations-of-ai", "hermes-agent-transcripts"]
   )
```

### Cleanup

```
User: Delete the empty test KB
→ kb_delete(id="test-kb")
# → Moved to ~/.hermes/mindforge/trash/
```

---

## Storage Structure

```
~/.hermes/mindforge/
├── kbs/
│   ├── foundations-of-ai/
│   │   ├── concepts.json      # Concept manifest
│   │   ├── graph.json         # Knowledge graph
│   │   └── concepts/          # Individual concept MD files
│   └── your-kb/
│       └── ...
├── trash/                     # Soft-deleted KBs
└── registry.json              # KB metadata index
```

---

## Troubleshooting

### "No knowledge base selected"

The MCP tools require an active KB for `search`, `get_concept`, etc. Use `kb_select` first.

### Stale data in Telegram

The Telegram gateway caches MCP connections. If you just created a KB but don't see it:
```bash
# Restart gateway
kill <gateway_pid>
```

Then reload config in Telegram session.

### Tools not appearing

Ensure `~/.hermes/config.yaml` has the `mcp_servers.mindforge` entry and restart Hermes Agent.

---

## Future Roadmap

| Feature | Status |
|---------|--------|
| Multi-user support | Planned for SaaS |
| KB sharing/export | Future |
| Remote KB urls | Future |
| Automated ingestion | Via cron jobs |

---

## See Also

- **Multi-KB Skill:** `mindforge-multikb` (architecture details)
- **Repository:** https://github.com/AcceleratedIndustries/MindForgeForHermes
