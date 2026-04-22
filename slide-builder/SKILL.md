---
name: slide-builder
description: Build and modify PowerPoint (PPTX) presentations programmatically using python-pptx. Clone existing decks, update content while preserving formatting, duplicate/delete/reorder slides, and export to PDF via Microsoft PowerPoint. Use when the user needs to create, update, or manipulate slide decks.
---

# slide-builder — Programmatic PowerPoint Deck Manipulation

Build, modify, and export PPTX presentations using `python-pptx`. Always clone an existing deck as the base — never build from scratch.

## Prerequisites

```bash
pip3 install python-pptx lxml
```

For PDF export: Microsoft PowerPoint must be installed at `/Applications/Microsoft PowerPoint.app`

## Core Principle: Clone, Don't Create

**NEVER build slides from scratch with `add_slide()`.** Branded decks embed custom slide masters, layouts, icons, gradients, diagrams, and fonts that cannot be recreated programmatically. Always:

1. Copy the source PPTX file
2. Analyze its structure
3. Modify slides in-place
4. Duplicate existing slides for new content
5. Delete slides you don't need

## Step 0: Ask the Operator

Before touching any file, gather:

1. **Which deck is the base?** Clone the most recent or most relevant one.
2. **What content goes in?** Get the actual text, not assumptions.
3. **Which slides to keep, modify, or remove?** Let the operator decide structure.
4. **Who are the presenters?** Names, emails, titles.
5. **Any updated stats or numbers?** Don't guess — ask.

## Step 1: Analyze the Deck

Map every slide before making changes.

```python
from pptx import Presentation
import shutil

# Always start from a copy
shutil.copy2("source.pptx", "output.pptx")
prs = Presentation("output.pptx")

# Map structure
for i, slide in enumerate(prs.slides):
    layout = slide.slide_layout.name
    print(f"\nSlide {i+1} | layout: '{layout}' | shapes: {len(slide.shapes)}")
    
    for shape in slide.shapes:
        # Check if placeholder
        ph = None
        try:
            ph = shape.placeholder_format.idx
        except:
            pass
        
        if shape.has_text_frame:
            text = shape.text_frame.text[:80]
            print(f"  {'ph=' + str(ph) if ph is not None else shape.name}: '{text}'")
```

### Placeholder Indices

Only two indices are standard across all PPTX files:

| Index | Purpose |
|---|---|
| 0 | Title |
| 1 | Body / Content / Subtitle (depends on layout) |

**Everything else is template-specific.** Presenter fields, CTA text, footers — their indices are defined by the slide master. Always discover them by analyzing the deck first, never assume.

**Not all text is in placeholders.** Presenter names, labels, disclaimers, and diagram annotations are often regular text boxes. Find them by text content or shape name, not by placeholder index.

## Step 2: Helper Functions

### Get a Placeholder

```python
def get_ph(slide, idx):
    for shape in slide.placeholders:
        if shape.placeholder_format.idx == idx:
            return shape
    return None
```

### Set Text on a Run (Preserves Formatting)

```python
def set_run_text(shape, para_idx, run_idx, text):
    tf = shape.text_frame
    if para_idx < len(tf.paragraphs) and run_idx < len(tf.paragraphs[para_idx].runs):
        tf.paragraphs[para_idx].runs[run_idx].text = text
```

### Replace Bullets (XML-Level, Preserves Formatting)

This is the most important helper. It clones paragraph formatting from existing paragraphs, so bold headers stay bold and bullet styles are preserved.

```python
import copy

def set_bullets_xml(shape, header, items):
    """Replace content placeholder with header + bullet items.
    First line inherits bold/header formatting, rest inherit bullet formatting."""
    if not shape or not shape.has_text_frame:
        return
    txBody = shape.text_frame._txBody
    a_ns = txBody.nsmap.get('a', 'http://schemas.openxmlformats.org/drawingml/2006/main')
    existing = txBody.findall(f'{{{a_ns}}}p')
    if not existing:
        return
    header_tmpl = existing[0]
    bullet_tmpl = existing[1] if len(existing) > 1 else existing[0]
    
    for p in existing:
        txBody.remove(p)
    
    for i, line in enumerate([header] + items):
        tmpl = header_tmpl if i == 0 else bullet_tmpl
        new_p = copy.deepcopy(tmpl)
        runs = new_p.findall(f'{{{a_ns}}}r')
        if runs:
            runs[0].find(f'{{{a_ns}}}t').text = line
            for r in runs[1:]:
                new_p.remove(r)
        txBody.append(new_p)
```

### Duplicate a Slide

```python
def duplicate_slide_after(prs, src_index, insert_after_index):
    """Duplicate slide at src_index, place after insert_after_index.
    Preserves all shapes, graphics, and formatting."""
    template = prs.slides[src_index]
    new_slide = prs.slides.add_slide(template.slide_layout)
    for shape in list(new_slide.shapes):
        shape._element.getparent().remove(shape._element)
    for shape in template.shapes:
        new_slide.shapes._spTree.append(copy.deepcopy(shape._element))
    sldIdLst = prs.slides._sldIdLst
    el = sldIdLst[-1]
    sldIdLst.remove(el)
    sldIdLst.insert(insert_after_index + 1, el)
    return prs.slides[insert_after_index + 1]
```

### Delete a Slide

```python
def delete_slide(prs, slide_index):
    rId = prs.slides._sldIdLst[slide_index].get(
        '{http://schemas.openxmlformats.org/officeDocument/2006/relationships}id')
    prs.part.drop_rel(rId)
    sldId = prs.slides._sldIdLst[slide_index]
    prs.slides._sldIdLst.remove(sldId)
```

### Move a Slide

```python
def move_slide(prs, old_index, new_index):
    sldIdLst = prs.slides._sldIdLst
    el = sldIdLst[old_index]
    sldIdLst.remove(el)
    sldIdLst.insert(new_index, el)
```

## Step 3: Modify Content

### Workflow

1. **Delete** unwanted slides (highest index first to avoid shifting issues)
2. **Reorder** remaining slides to match target structure
3. **Update text** on reused slides using `set_run_text` or `set_bullets_xml`
4. **Duplicate** existing slides for new content sections
5. **Populate** duplicated slides with new text

### Index Shifting Warning

When inserting or deleting slides, all subsequent indices shift. Always:
1. Delete from highest index to lowest
2. After any insert/delete, re-map indices before the next operation
3. Or find slides by title text rather than hardcoded index:

```python
def find_slide_by_title(prs, title_substring):
    for i, slide in enumerate(prs.slides):
        ph0 = get_ph(slide, 0)
        if ph0 and ph0.has_text_frame and title_substring in ph0.text_frame.text:
            return i
    return None
```

## Step 4: Content Guidelines

### Slide Density Limits

| Slide Type | Max Lines | Rule |
|---|---|---|
| Summary | 7 | Header + 6 items. Split if more. |
| Detail | 5 | Bold tagline + 3-4 bullets |
| Roadmap | 6 | Header + 5 items. Split if more. |

**python-pptx cannot detect visual overflow.** Stay within limits and have the operator verify in PowerPoint.

### Writing Style

Study the existing slides and match their tone. Common patterns in professional decks:

- **Bold tagline first** — short, benefit-focused phrase
- **Declarative bullets** — state facts, not commands. "Validates client certificates" NOT "Validate client certificates"
- **No duplication** — title, tagline, and bullets must each add new information
- **Crystal clear** — if a bullet isn't immediately understandable, drop it. The presenter explains verbally.
- **3-4 bullets max** on detail slides. Less is more.

### Tone Drift Warning

AI-generated bullets default to imperative/technical language ("Execute", "Implement", "Reduce", "Leverage"). Professional slide decks use declarative, benefit-focused language. Always compare new slides against originals side-by-side and rewrite to match.

## Step 5: Verify

```python
# Dump all text for review
for i, slide in enumerate(prs.slides):
    layout = slide.slide_layout.name
    print(f"\nSlide {i+1} | {layout}")
    for shape in slide.shapes:
        if shape.has_text_frame:
            for p in shape.text_frame.paragraphs:
                t = p.text.strip()
                if t:
                    print(f"  {t[:100]}")
```

Check:
- Slide count matches plan
- No slide exceeds density limits
- No stale or incorrect text from the source deck
- No duplicate content between title, tagline, and bullets

Then ask the operator to open in PowerPoint and verify:
- Graphics/diagrams still make sense with new text
- No text overflow
- Formatting looks correct

## Step 6: Export to PDF

**Use Microsoft PowerPoint, not LibreOffice.** LibreOffice breaks embedded custom fonts.

### Single File

```bash
osascript -e '
tell application "Microsoft PowerPoint"
  open POSIX file "/path/to/deck.pptx"
  delay 2
  set theDoc to active presentation
  save theDoc in POSIX file "/path/to/deck.pdf" as save as PDF
  close theDoc
end tell
'
```

### Batch Export

```bash
for f in /path/to/*.pptx; do
  fname=$(basename "$f")
  pdfpath="/path/to/${fname%.pptx}.pdf"
  osascript -e "
    tell application \"Microsoft PowerPoint\"
      open POSIX file \"$f\"
      delay 2
      set theDoc to active presentation
      save theDoc in POSIX file \"$pdfpath\" as save as PDF
      close theDoc
    end tell
  "
done
```

**Skip temp files** — filter out `~$*.pptx` (PowerPoint lock files).

## Common Pitfalls

1. **`add_slide()` loses everything** — graphics, icons, embedded formatting all gone. Always duplicate from existing slides.
2. **Index shifting is the #1 bug source** — after any insert/delete, re-verify all indices or use title-based lookup.
3. **`shape.text = "..."` destroys formatting** — use `set_run_text` or `set_bullets_xml` to preserve bold, italic, color, and font size.
4. **Placeholder idx=1 means different things** — on cover slides it's the subtitle (small, centered), on content slides it's the body (large, left-aligned). Same index, different layout.
5. **Not all text is in placeholders** — presenter names, diagram labels, disclaimers are often regular text boxes. Find by content, not index.
6. **LibreOffice PDF export breaks fonts** — always use PowerPoint via AppleScript on macOS.
7. **python-pptx can't detect overflow** — enforce density limits manually.
8. **Tables exist** — check `shape.has_table` when analyzing. Tables have their own cell-based text access pattern.
9. **Images and groups are opaque** — python-pptx can read their position/size but not meaningfully edit embedded graphics. Leave them alone or delete them.
10. **Save frequently** — `prs.save()` after each major operation. If something breaks, you lose less work.
