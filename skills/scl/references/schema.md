# .scg.yaml Schema Reference

## Entity fields

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `id` | yes | string | Unique. Convention: `e01`, `e02`... |
| `type` | yes | enum | `CORE` `INV` `PROHIBIT` `FACT` `OVERRIDE` `RUNTIME` |
| `key` | yes | string | snake_case. Used for retrieval and relations. |
| `value` | yes | string | The knowledge chunk. Multiline with `\|`. Keep under 200 words. |
| `layer` | yes | int | `0` = Core · `1` = Invariant · `2` = Task-specific |
| `entity_type` | no | enum | `fact` `concept` `procedure` `knowledge` |
| `immutable` | no | bool | If true, cannot be overridden. Default: false |

## Type priorities

| Type | Priority | Semantic boost |
|------|----------|----------------|
| CORE | 1000 | — |
| INV | 100 | ×2.5 if value contains: must, critical, required |
| PROHIBIT | 100 | ×2.5 if value contains: never, prohibit, wrong |
| FACT | 10 | — |
| OVERRIDE | 50 | — |
| RUNTIME | 20 | — |

## Relation fields

| Field | Required | Values |
|-------|----------|--------|
| `from` | yes | entity `id` |
| `to` | yes | entity `id` |
| `type` | yes | `REQUIRES` `DEPENDS_ON` `CONSTRAINS` `RELATED_TO` |

All relations are forward-only. `A REQUIRES B` = "retrieve A → also retrieve B". Not the reverse.

## Layer guidelines

- **Layer 0**: 3–6 entities. File format, primary tool, validation. Always loaded.
- **Layer 1**: 5–12 entities. Hard rules, CRITICAL/NEVER patterns. Always loaded.
- **Layer 2**: 10–20 entities. Task procedures, subtopic patterns. Retrieved on demand.

## Common mistakes

- Too many L0/L1 entities defeats compression — keep layers 0+1 under 15 total
- Reverse `DEPENDS_ON` edges cause noise from sibling subtopics — forward only
- Catalog descriptions in wrong language — must match user's query language
- Entity values over 200 words — split into two linked entities instead

---

## content_hash algorithm

The hash covers only `entities` and `relations` — not metadata like `description` or `language`. Run after completing the full entity/relation list.

```python
import hashlib, yaml, json

def scg_hash(data):
    entities = sorted([
        {"id": e["id"], "type": e["type"], "key": e["key"],
         "value": e["value"], "layer": e["layer"]}
        for e in data["entities"]
    ], key=lambda x: x["id"])
    relations = sorted([
        {"from": r["from"], "to": r["to"], "type": r["type"]}
        for r in data["relations"]
    ], key=lambda x: (x["from"], x["to"]))
    canonical = json.dumps({"entities": entities, "relations": relations},
                           sort_keys=True, ensure_ascii=False)
    return "sha256:" + hashlib.sha256(canonical.encode()).hexdigest()[:16]

# Stamp
data = yaml.safe_load(open("skill.scg.yaml"))
data["content_hash"] = scg_hash(data)
yaml.dump(data, open("skill.scg.yaml", "w"), allow_unicode=True)

# Verify (Consumer Mode)
def verify(data):
    stored = data.get("content_hash", "")
    return True if not stored else stored == scg_hash(data)
```
