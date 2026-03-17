# SCL Benchmark

How we measured token savings and what the numbers mean.

---

## Methodology

Token counts use the standard approximation of 4 characters = 1 token, consistent with Claude's tokenizer for mixed English/code content.

**Baseline** = the full context Claude loads when a skill triggers without SCL:
- Skills with a single SKILL.md: that file's token count
- Skills with reference files (pdf, pptx): SKILL.md + all reference files loaded

**SCL retrieval** = tokens in the pruned context block for a given query:
- Layer 0 + Layer 1 entities (always loaded)
- Layer 2 entities matched by the query + multi-hop dependencies

Each skill was tested against 5–10 representative queries covering its full surface area. The SCL avg is the mean across those queries.

---

## Results

### Context savings (SCG retrieval vs full skill load)

| Skill | Full load | SCL avg/query | Saved |
|-------|-----------|---------------|-------|
| pdf   | 9,145 tok | 527 tok | **94%** |
| pptx  | 7,284 tok | 556 tok | **92%** |
| docx  | 5,014 tok | 606 tok | **88%** |
| xlsx  | 2,863 tok | 558 tok | **81%** |
| kubernetes | 2,695 tok | 588 tok | **78%** |

### Real cost per query

The SCL skill itself loads 446 tokens at session start (one-time cost).

| | First query | 2nd+ query |
|--|------------|-----------|
| **pdf** | 973 tok (89% saved) | 527 tok (94% saved) |
| **pptx** | 1,002 tok (86% saved) | 556 tok (92% saved) |
| **docx** | 1,052 tok (79% saved) | 606 tok (88% saved) |
| **xlsx** | 1,004 tok (65% saved) | 558 tok (81% saved) |
| **kubernetes** | 1,034 tok (62% saved) | 588 tok (78% saved) |

---

## Why savings vary

**pdf and pptx save the most** because their baselines include large reference files (REFERENCE.md, FORMS.md, editing.md, pptxgenjs.md) that are rarely needed in full for any single query. SCL loads only the relevant slice.

**xlsx and kubernetes save less on first query** because their SKILL.md files are already compact (~2,700 tok). The 446-token SCL overhead is proportionally larger. From the second query onward the overhead disappears and savings are 78–81%.

**The break-even point** is roughly 2 queries per session for small skills, and 1 query for skills over 5,000 tokens.

---

## What SCL does not help with

Sequential workflow skills — where the user always needs the full process from start to finish — see at most 20–28% savings. Examples: skill-creator, CI/CD pipeline skills, multi-step agent orchestration. For those, load the SKILL.md in full.

A skill is a good SCL candidate if it has 10+ independently answerable subtopics. See `references/author_mode.md` for the full qualification criteria.

---

## SCL overhead

The SCL SKILL.md is 446 tokens — deliberately minimal. Evolution during development:

| Stage | Tokens | Change |
|-------|--------|--------|
| Initial draft | 1,232 | — |
| + versioning code | 1,863 | Added hash algorithm inline |
| − benchmark table → README | 1,784 | Not needed at runtime |
| − Python code → schema.md | 1,061 | Code is a reference, not instruction |
| − Author Mode → reference file | 562 | Only Consumer Mode needed at runtime |
| − selection criteria → author_mode.md | **446** | Final |

The principle: SKILL.md contains only what Claude needs to execute Consumer Mode. Everything else loads on demand.
