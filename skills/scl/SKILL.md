---
name: scl
description: >
  Build an SCG: "index my skill", "create SCG for", "make my skill use less tokens".
  Use an SCG: automatic when a .scg.yaml exists alongside a SKILL.md.
  Inspect: "which nodes did you use", "debug SCL", "show SCG".
---

## CONSUMER MODE

Activate **at the start of any response** where the skill's directory has a `.scg.yaml` alongside its `SKILL.md`. Check before loading any skill content.

- `.scg.yaml` present → follow steps below, do not load the full SKILL.md
- `.scg.yaml` absent → load SKILL.md normally

### 1. Load and verify

Read `.scg.yaml`. If `content_hash` is present, recompute it (algorithm in `references/schema.md`) and compare. If mismatch, fall back to full SKILL.md and warn:
```
⚠️ SCL: content_hash mismatch — SCG may be out of sync. Falling back to full skill load.
```

### 2. Map query → keys

Read only the `## SCL Key Catalog` block (between `SCL_CATALOG_START` / `SCL_CATALOG_END`). Match user query to Layer-2 keys by semantic reasoning. Decompose multi-part queries first. Layer 0+1 load automatically.

### 3. Multi-hop retrieval (forward BFS only)

From matched keys, traverse `REQUIRES` / `DEPENDS_ON` / `CONSTRAINS` edges up to depth 2. Never traverse reverse edges.

### 4. Resolve and inject

Priority: Layer 2 > 1 > 0, then type priority, then id. Render and inject. Do not load the full SKILL.md.

### 5. Debug (on request)

Report **actual** numbers from this retrieval:
```
SCL: <N> nodes · ~<tokens> tok (vs ~<baseline> full · <pct>% saved)
Keys: <keys> [+<N> invariants]
```

---

## Reference files

- `references/author_mode.md` — how to build an SCG for a skill
- `references/schema.md` — full field reference + hash algorithm
- `references/worked_example.md` — complete annotated SCG for the docx skill
