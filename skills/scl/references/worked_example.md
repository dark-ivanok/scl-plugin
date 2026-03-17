# Worked Example — docx skill SCG (annotated)

This is the complete `.scg.yaml` for the docx skill, with inline annotations
explaining each design decision. Use it as a template when building new SCGs.

The docx skill SKILL.md is ~5,000 tokens. This SCG is ~700 tokens.
With the SCL catalog injected and consumer mode active, a typical query
loads ~400 tokens — a 92% reduction.

---

## docx.scg.yaml

```yaml
scl_version: "1.0"
skill: docx
language: es
description: "Creación, edición y análisis de archivos Word (.docx)"

# ── LAYER 0: CORE ─────────────────────────────────────────────────────────
# Always loaded. 4 entities covering the three entry points + validation.
# Rule: if you need it for EVERY docx task, it goes here.

entities:
  - id: e01
    type: CORE
    key: docx_format
    value: "A .docx file is a ZIP archive containing XML files."
    layer: 0
    entity_type: fact
    immutable: true
    # ↑ immutable: true because this fact never changes and should never
    #   be overridden by a RUNTIME or OVERRIDE entity.

  - id: e02
    type: CORE
    key: tool_new_doc
    value: "Create new .docx with JavaScript: npm install -g docx. Use Document, Packer from 'docx' package."
    layer: 0
    entity_type: procedure

  - id: e03
    type: CORE
    key: tool_edit_doc
    value: "Edit existing .docx: Unpack → edit XML → repack. Scripts: unpack.py, pack.py"
    layer: 0
    entity_type: procedure

  - id: e04
    type: CORE
    key: tool_read_doc
    value: "Read/analyze content: pandoc or unpack for raw XML. pandoc --track-changes=all document.docx -o output.md"
    layer: 0
    entity_type: procedure

  - id: e05
    type: CORE
    key: validation
    value: "After creating a file, always validate: python scripts/office/validate.py doc.docx. If fails, unpack, fix XML, repack."
    layer: 0
    entity_type: procedure

# ── LAYER 1: GLOBAL INVARIANTS ────────────────────────────────────────────
# Always loaded. Hard rules. Note: PROHIBIT entities with "NEVER" in their
# value get a 2.5× semantic priority boost automatically.

  - id: e10
    type: INV
    key: page_size_rule
    value: "CRITICAL: docx-js defaults to A4. Always set page size explicitly. US Letter: width=12240, height=15840 DXA. 1440 DXA = 1 inch."
    layer: 1
    entity_type: fact

  - id: e11
    type: PROHIBIT
    key: no_unicode_bullets
    value: "NEVER use unicode bullet characters (• or \\u2022). ALWAYS use LevelFormat.BULLET with numbering config."
    layer: 1
    entity_type: fact

  - id: e12
    type: PROHIBIT
    key: no_newline_in_docx
    value: "NEVER use \\n newlines in docx-js. Use separate Paragraph elements instead."
    layer: 1
    entity_type: fact

  - id: e13
    type: INV
    key: table_dual_widths
    value: "CRITICAL: Tables need dual widths — set columnWidths on the table AND width on each cell."
    layer: 1
    entity_type: fact

  - id: e14
    type: PROHIBIT
    key: no_percentage_width
    value: "NEVER use WidthType.PERCENTAGE for tables — it breaks in Google Docs. Always use WidthType.DXA."
    layer: 1
    entity_type: fact

  - id: e15
    type: PROHIBIT
    key: no_solid_shading
    value: "NEVER use ShadingType.SOLID for table shading — causes black backgrounds. Use ShadingType.CLEAR."
    layer: 1
    entity_type: fact

  - id: e16
    type: PROHIBIT
    key: no_table_dividers
    value: "NEVER use tables as dividers/rules. Use border on Paragraph instead."
    layer: 1
    entity_type: fact

  - id: e17
    type: INV
    key: image_run_type
    value: "ImageRun always requires explicit type field (png/jpg/etc). Missing type causes errors."
    layer: 1
    entity_type: fact

  - id: e18
    type: INV
    key: page_break_in_paragraph
    value: "PageBreak must always be inside a Paragraph element. Standalone PageBreak creates invalid XML."
    layer: 1
    entity_type: fact

  - id: e19
    type: INV
    key: smart_quotes
    value: "When adding text via XML editing, use XML entities: &#x2019; (apostrophe), &#x201C; (left double), &#x201D; (right double)."
    layer: 1
    entity_type: fact

# ── LAYER 2: TASK-SPECIFIC ────────────────────────────────────────────────
# Only loaded when query matches. One entity per coherent subtopic.
# Design question for each: "could a user ask ONLY about this, nothing else?"

  - id: e20
    type: FACT
    key: styles_headings
    value: |
      Override built-in heading styles using exact IDs: "Heading1", "Heading2".
      Include outlineLevel (0=H1, 1=H2) — required for TOC.
      Default font: Arial 32pt H1, 28pt H2. Keep titles black.
    layer: 2
    entity_type: knowledge

  - id: e21
    type: INV
    key: toc_requires_headinglevel
    value: "TOC requires HeadingLevel only on heading paragraphs. No custom styles on heading paragraphs used for TOC."
    layer: 2
    entity_type: fact
    # ↑ Layer 2 INV: this rule only matters when TOC is involved.
    #   Putting it in Layer 1 would load it for every prompt unnecessarily.

  - id: e22
    type: FACT
    key: table_construction
    value: |
      Table construction:
      - width: { size: 9360, type: WidthType.DXA } (US Letter: 12240 - 2880 margins)
      - columnWidths must sum exactly to table width
      - Each cell: width: { size: N, type: WidthType.DXA }
      - Cell margins: { top: 80, bottom: 80, left: 120, right: 120 }
      - Borders: BorderStyle.SINGLE, size:1, color:"CCCCCC"
      - Shading: ShadingType.CLEAR
    layer: 2
    entity_type: procedure

  - id: e23
    type: FACT
    key: list_construction
    value: |
      Lists use numbering config with LevelFormat.BULLET.
      Same reference = continues numbering. Different reference = restarts.
    layer: 2
    entity_type: procedure

  - id: e24
    type: FACT
    key: image_in_new_doc
    value: "ImageRun: { data: buffer, type: 'png', width: N, height: N }. EMU: 914400 = 1 inch."
    layer: 2
    entity_type: procedure

  - id: e25
    type: FACT
    key: image_in_existing_doc
    value: |
      Adding image to existing .docx:
      1. Add file to word/media/
      2. Add relationship to word/_rels/document.xml.rels
      3. Add content type to [Content_Types].xml
      4. Add <w:drawing><wp:inline> block with EMU dimensions.
    layer: 2
    entity_type: procedure

  - id: e26
    type: FACT
    key: tracked_changes
    value: |
      Tracked changes XML:
      - Insertion: <w:ins w:author="Claude" ...><w:r><w:t>text</w:t></w:r></w:ins>
      - Deletion: <w:del ...><w:r><w:delText>text</w:delText></w:r></w:del>
      - Replace entire <w:r> blocks. Preserve <w:rPr> formatting.
    layer: 2
    entity_type: procedure

  - id: e27
    type: FACT
    key: comments
    value: |
      Comments via comment.py:
      python scripts/comment.py unpacked/ 0 "Text"
      python scripts/comment.py unpacked/ 1 "Reply" --parent 0
      Add markers in document.xml. commentRangeStart/End are siblings of <w:r>, never inside.
    layer: 2
    entity_type: procedure

  - id: e28
    type: FACT
    key: headers_footers
    value: "Headers/Footers: use Header/Footer objects. Two-column footers: tab stops, NOT tables. Page numbers: PageNumber.CURRENT."
    layer: 2
    entity_type: procedure

  - id: e29
    type: FACT
    key: landscape_orientation
    value: "Landscape: pass portrait dimensions (short=width, long=height) + orientation: PageOrientation.LANDSCAPE. docx-js swaps internally."
    layer: 2
    entity_type: procedure

  - id: e30
    type: FACT
    key: xml_edit_workflow
    value: |
      XML edit steps (in order):
      1. python scripts/office/unpack.py document.docx unpacked/
      2. Edit files in unpacked/word/ with Edit tool directly (NOT Python scripts)
      3. python scripts/office/pack.py unpacked/ output.docx --original document.docx
      Element order in <w:pPr>: pStyle → numPr → spacing → ind → jc → rPr.
      RSIDs: 8-digit hex. xml:space="preserve" on <w:t> with spaces.
    layer: 2
    entity_type: procedure

  - id: e31
    type: FACT
    key: doc_conversion
    value: "Legacy .doc: python scripts/office/soffice.py --headless --convert-to docx document.doc"
    layer: 2
    entity_type: procedure

# ── RELATIONS ─────────────────────────────────────────────────────────────
# All forward-only. "If user needs FROM, they also need TO."
# Do NOT add reverse edges — they cause noise from sibling subtopics.

relations:
  # new doc workflow requires these invariants
  - { from: e02, to: e10, type: REQUIRES }   # new doc → page size rule
  - { from: e02, to: e11, type: REQUIRES }   # new doc → no unicode bullets
  - { from: e02, to: e12, type: REQUIRES }   # new doc → no newlines
  - { from: e02, to: e05, type: REQUIRES }   # new doc → validation

  # edit workflow is the foundation for XML-based tasks
  - { from: e03, to: e30, type: REQUIRES }   # edit → xml workflow

  # table subtopic pulls its own invariants
  - { from: e22, to: e13, type: CONSTRAINS } # table → dual widths
  - { from: e22, to: e14, type: CONSTRAINS } # table → no percentage
  - { from: e22, to: e15, type: CONSTRAINS } # table → no solid shading

  # styles subtopic links to TOC invariant
  - { from: e20, to: e21, type: REQUIRES }   # styles → toc requires headinglevel

  # image subtopics require the type invariant
  - { from: e24, to: e17, type: REQUIRES }   # image new → imagerun type
  - { from: e25, to: e17, type: REQUIRES }   # image existing → imagerun type

  # XML-based tasks depend on workflow + smart quotes
  - { from: e26, to: e30, type: DEPENDS_ON } # tracked changes → xml workflow
  - { from: e27, to: e30, type: DEPENDS_ON } # comments → xml workflow
  - { from: e30, to: e19, type: REQUIRES }   # xml workflow → smart quotes

  # landscape is a specialisation of page size
  - { from: e29, to: e10, type: RELATED_TO } # landscape → page size
```

---

## What this produces for a sample query

Query: "Añade una tabla con 3 columnas y fondo de color"

1. Catalog match → `table_construction`
2. Forward BFS from `e22`:
   - e22 → e13 (CONSTRAINS) → loaded
   - e22 → e14 (CONSTRAINS) → loaded
   - e22 → e15 (CONSTRAINS) → loaded
3. Layer 0+1 auto-loaded: e01–e05, e10–e19
4. Final context: e01, e02, e05, e10–e16, e22 = **10 entities, ~420 tokens**
5. Baseline: full SKILL.md = **~5,000 tokens**
6. Savings: **91.6%** — with 100% coverage of needed knowledge.
