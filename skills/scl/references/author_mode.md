# SCL Author Mode — indexing a skill

Read this when the user wants to build an SCG for a skill.
Output: a `skill.scg.yaml` + a `## SCL Key Catalog` block injected into the target SKILL.md.

---

## 0. Qualify the skill first

Count independently answerable subtopics — sections a user query would need alone.

- **10+ subtopics** → proceed
- **5–9 subtopics** → moderate savings, worth it if heavily used
- **Fewer than 5** or sequential workflow → tell the user SCL won't help much here

---

## 1. Analyse the SKILL.md

Decompose into entities — one per subtopic a user might ask about independently. Target 15–35 total.

| Layer | Type | Loads |
|-------|------|-------|
| 0 | `CORE` | Every prompt |
| 1 | `INV` / `PROHIBIT` | Every prompt |
| 2 | `FACT` | On demand only |

Keep layers 0+1 under 15 entities total.

---

## 2. Design relations (forward-only)

Ask: "if the user needs A, do they also always need B?"

| Type | Meaning |
|------|---------|
| `REQUIRES` | A cannot work correctly without B |
| `DEPENDS_ON` | A's procedure builds on B |
| `CONSTRAINS` | A sets hard limits on B |
| `RELATED_TO` | Soft topical link (optional traversal) |

---

## 3. Write the .scg.yaml

See `schema.md` for full field reference and the hash algorithm.

Minimal structure:

```yaml
scl_version: "1.0"
skill: my-skill
language: en
content_hash: ""

entities:
  - { id: e01, type: CORE, key: overview, value: "...", layer: 0 }

relations:
  - { from: e20, to: e10, type: REQUIRES }
```

Before stamping the hash, verify:
- Every entity has `id`, `type`, `key`, `value`, `layer`
- No duplicate `key` values
- Every relation `from`/`to` matches an existing `id`
- Layers 0+1 combined ≤ 15 entities

Then compute and write `content_hash` using the algorithm in `schema.md`.

---

## 4. Inject the Key Catalog into the target SKILL.md

Append to the target SKILL.md. Descriptions **must be in the user's query language**.

```markdown
## SCL Key Catalog
<!-- SCL_CATALOG_START -->
| Key | Layer | Type | Description |
|-----|-------|------|-------------|
| overview   | 0 | CORE     | What this covers |
| no_pattern | 1 | PROHIBIT | What this prevents |
| procedure  | 2 | FACT     | What task this handles |
<!-- SCL_CATALOG_END -->
```

---

## 5. Verify

Quick sanity check after injecting:
- Load the `.scg.yaml` and run 3–5 representative queries mentally
- Confirm the right L2 nodes are retrieved for each
- Confirm L0+L1 invariants load on every query
- If coverage feels low, check the Key Catalog descriptions — they must match how users actually phrase queries
