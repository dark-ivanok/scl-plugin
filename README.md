# SCL — System Cognitive Layer

**Compress any domain knowledge skill into a typed graph. 80–90% fewer tokens per query, same output quality.**

## The problem

A typical domain knowledge skill (docx, kubernetes, pdf...) loads its entire `SKILL.md` on every query — even if the user only needs one subtopic. A 5,000-token skill loads 5,000 tokens whether you're asking about tables or tracked changes.

## How SCL fixes it

SCL decomposes a `SKILL.md` into a typed knowledge graph (`.scg.yaml`). When a query arrives, only the relevant nodes are retrieved and injected — everything else stays on disk.

| Skill | Baseline | SCL | Savings |
|-------|----------|-----|---------|
| docx | 5,014 tok | 554 tok | **89%** |
| pdf (3 files) | 9,144 tok | 870 tok | **90%** |
| kubernetes | 2,695 tok | 588 tok | **78%** |

No external API calls. No embeddings. Claude reads a compact Key Catalog and maps the query to nodes using its own semantic reasoning.

## Benchmark results

Tested against Anthropic's public skills (baseline = full skill load including all reference files):

| Skill | Baseline | SCL avg/query | Saved |
|-------|----------|---------------|-------|
| pdf | 9,145 tok | 527 tok | 94% |
| pptx | 7,284 tok | 556 tok | 92% |
| docx | 5,014 tok | 606 tok | 88% |
| xlsx | 2,863 tok | 558 tok | 80% |
| kubernetes | 2,695 tok | 588 tok | 78% |

Baseline = full skill load including all reference files.

**Real cost per query** = SCL SKILL.md (582 tok, Consumer Mode only) + SCL avg above.
Author Mode lives in `references/author.md` and only loads when indexing a skill.

| Skill | Baseline | Real total | Real saving |
|-------|----------|-----------|-------------|
| pdf | 9,145 tok | 1,109 tok | 88% |
| pptx | 7,284 tok | 1,138 tok | 84% |
| docx | 5,014 tok | 1,188 tok | 76% |
| xlsx | 2,863 tok | 1,140 tok | 60% |
| kubernetes | 2,695 tok | 1,170 tok | 57% |


## Install

```bash
/plugin marketplace add dark-ivanok/scl-plugin
/plugin install scl
```

Or directly:
```bash
claude plugin install github:dark-ivanok/scl-plugin
```

## Usage

**Index an existing skill (Author Mode):**
> "index my skill" / "apply SCL to this SKILL.md" / "supercharge my skill"

Claude reads your `SKILL.md`, decomposes it into a `.scg.yaml`, and injects a Key Catalog back into the skill automatically.

**Using an indexed skill (Consumer Mode):**

Automatic. Once a `.scg.yaml` exists alongside a `SKILL.md`, SCL loads only the relevant nodes on every query. No configuration needed.

**Debug:**
> "which nodes did you use" / "debug SCL"

```
SCL: 6 nodes · ~380 tokens (vs ~5,000 full skill · 92% saved)
Keys: image_in_existing_doc, image_run_type, xml_edit_workflow [+2 invariants]
```

## How it works

Each entity has a **layer** (0=always load, 1=invariants, 2=on demand) and a **type** (CORE, INV, PROHIBIT, FACT). Relations between nodes (REQUIRES, DEPENDS_ON, CONSTRAINS) let Claude pull in dependencies via forward BFS — without loading unrelated subtopics.

See `skills/scl/references/worked_example.md` for a complete annotated example.

## Files

```
scl-plugin/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
└── skills/scl/
    ├── SKILL.md
    └── references/
        ├── schema.md
        └── worked_example.md
```

## ❤️ Support this project

This project was built using the free version of Claude, driven purely by curiosity and a desire to explore new ideas.

If you find it useful or interesting, consider supporting its development through donations or sponsorship. Any support helps keep the project evolving and allows for more time improving and expanding it.

Thanks for being part of the journey 🚀

## License

MIT
