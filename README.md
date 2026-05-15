# ppt-template-overwriter

A Claude skill for injecting content into existing PowerPoint (`.pptx`) templates while fully preserving branding, theme, animations, fonts, slide masters, and color schemes.

---

## What It Does

This skill handles content population **into** existing `.pptx` templates — not creating decks from scratch. It is designed for use cases like:

- Filling corporate pitch deck templates with new data
- Populating report slides from JSON/YAML content files
- Batch-generating slide decks from structured inputs
- Replacing text/images in branded templates without breaking formatting

---

## When Claude Uses This Skill

Claude triggers this skill when you say things like:

> "Fill my template", "populate slides from data", "replace placeholders in pptx", "generate deck from JSON/YAML", "inject content into PowerPoint", "update template with new data", "add images to pptx template", "replace text in slides without breaking formatting"

---

## Repository Structure

```
ppt-template-overwriter/
├── SKILL.md                          # Main skill instructions for Claude
├── references/
│   ├── parsing.md                    # Template parsing & placeholder detection
│   ├── injection.md                  # Content injection, overflow & special elements
│   ├── images.md                     # Image replacement
│   └── colors.md                     # Theme color extraction & usage
├── scripts/
│   ├── engine.py                     # Core Python engine
│   └── run.py                        # CLI runner
└── assets/
    └── schemas/
        ├── content.schema.json        # JSON schema for content input
        └── example.yaml               # Example YAML content file
```

---

## Three Mandatory Rules

These rules are enforced by the skill and prevent empty slides, broken formatting, and color mismatches.

### Rule 1 — Always Ask for Content Keyed to Slide Numbers

Before writing any code, Claude detects the template and shows a slide map:

```
Slide 0 │ Cover Page                │ object 2, object 5, object 6
Slide 1 │ Introduction & Objectives │ title (ph0), TextBox 10
Slide 2 │ Problem Identification    │ title (ph0), body (ph1)
```

Claude then asks you to confirm which content belongs to which slide. It never guesses the mapping.

### Rule 2 — Verify the Live Slide Title Before Every Injection

Before writing to slide N, Claude reads its actual title from the file and checks it matches your stated topic:

```
✔ VERIFIED  slide=3  live title="Problem Identification"  → injecting
✗ MISMATCH  slide=3  live title="Conclusion"  → SKIPPED (wrong slide mapped)
```

Mismatched slides are skipped and reported — never silently overwritten.

### Rule 3 — Extract and Use the PPT's Own Theme Colors

Claude runs a color extractor before any injection and uses your template's exact palette values for text colors, headings, and table fills. No hardcoded hex values.

---

## Workflow

```
STEP 1  DETECT         → Build slide map, show to user
STEP 2  EXTRACT COLORS → Extract dk1, lt1, accent1–6 from theme
STEP 3  CONFIRM MAP    → Present slide map table, WAIT for user confirmation
STEP 4  BACKUP         → Store current text of every shape
STEP 5  VERIFY & INJECT → Per slide: verify title → inject with theme colors
STEP 6  AUDIT          → Flag empty/stale shapes, restore from backup if needed
STEP 7  SAVE & REPORT  → Output file + per-slide summary (✔ injected / ✗ skipped / ⚠ restored)
```

---

## Setup

```bash
pip install python-pptx Pillow pyyaml lxml
```

---

## Content Input Format

Content can be provided as **JSON** or **YAML**, keyed by slide number.

Example (`content.yaml`):

```yaml
slides:
  0:
    title: "Q3 Performance Report"
    subtitle: "Finance Team · May 2026"
  1:
    title: "Introduction"
    body:
      - "Revenue up 12% YoY"
      - "Three new market segments entered"
      - "Headcount grew from 80 to 105"
  2:
    title: "Problem Identification"
    body: "Supply chain delays impacted fulfilment in H1..."
```

See [`assets/schemas/content.schema.json`](assets/schemas/content.schema.json) for the full schema and [`assets/schemas/example.yaml`](assets/schemas/example.yaml) for a complete example.

---

## CLI Usage

```bash
python scripts/run.py \
  --template path/to/template.pptx \
  --content  path/to/content.yaml \
  --output   path/to/output.pptx
```

Optional flags:

| Flag | Description |
|------|-------------|
| `--chunk N` | Process N slides at a time (useful for large decks, 50+ slides) |
| `--dry-run` | Print the slide map and exit without writing |
| `--no-audit` | Skip the post-injection audit step |

---

## Edge Cases Handled

| Scenario | Behaviour |
|----------|-----------|
| Content without slide numbers | Stops, shows slide map, asks user to confirm mapping |
| Topic mismatch on verify | Skips slide, logs `[MISMATCH]`, reports before saving |
| Empty string provided as content | Skips the shape — never writes empty string |
| Shape empty after injection | Restores from backup, logs `[RESTORED]` |
| Stale placeholder text remaining | Flags as `[STALE]`, re-attempts or reports |
| Locked shapes | Detected via XML, skipped with `[SKIP]` warning |
| Charts | `ChartData.replace_data()` — preserves chart type and colors |
| Tables | Row/cell injection; extra rows/cols left unchanged |
| Multi-language fonts | `<a:rPr lang>` preserved; only text changes |
| Large decks (50+ slides) | Use `--chunk 10` CLI flag |
| Password-protected files | Not supported — decrypt with LibreOffice first |

---

## Reference Docs

| Topic | File |
|-------|------|
| Template parsing & placeholder detection | [`references/parsing.md`](references/parsing.md) |
| Text, list, chart & table injection | [`references/injection.md`](references/injection.md) |
| Image replacement | [`references/images.md`](references/images.md) |
| Theme color extraction | [`references/colors.md`](references/colors.md) |

---

## License

MIT
