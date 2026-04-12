---
name: tikz
description: Audit and fix TikZ visual collisions in any .tex file. Use when labels overlap arrows, text sits on boxes, or arrows cross each other. Applies mathematical gap calculations and Bézier depth formulas — no eyeballing.
allowed-tools: Bash(pdflatex*), Bash(grep*), Bash(ls*), Read, Edit, Glob
argument-hint: [path/to/file.tex]
---

# TikZ Collision Audit

**Purpose**: Find and fix every visual collision in every TikZ figure in a given `.tex` file. Labels on arrows, text inside boxes, arrows crossing arrows — this skill catches them all using measurement, not intuition.

**The fundamental rule**: Claude cannot reliably eyeball where TikZ elements land. All placement must be verified mathematically before declaring it safe.

---

## Step 1: Identify the file

If the user specified a file, use it. If not, ask. Then:

```bash
grep -n "tikzpicture\|begin{frame}\|node\|draw\|bend\|foreach" [file].tex | head -100
```

Get a sense of scope: how many TikZ diagrams, how many frames, how many arrows.

---

## Step 2: For each TikZ diagram, run all 6 Passes in order

Full rules and formulas are in `~/mixtapetools/.claude/skills/compiledeck/tikz_rules.md`. Read it if you haven't. What follows is the operational checklist — the reference file has the formulas.

---

### Pass 0: Cross-slide consistency

```bash
grep -n "node.*{" [file].tex | grep -v "^[[:space:]]*%"
```

If the same diagram appears on multiple slides: colors, positions, and font sizes must be identical across all instances. Only the deliberate change (a new highlight, a new node) should differ.

---

### Pass 1: Bézier curves — do this FIRST

```bash
grep -n "bend" [file].tex
```

For **each** curved arrow found, compute:

```
chord_length = distance between the two endpoints
max_depth    = (chord_length / 2) × tan(bend_angle / 2)
safe_zone    = max_depth + 0.5cm
```

Quick reference:

| Bend | tan(angle/2) |
|------|-------------|
| 20°  | 0.176       |
| 25°  | 0.222       |
| 30°  | 0.268       |
| 35°  | 0.315       |
| 40°  | 0.364       |
| 45°  | 0.414       |

**Check 1**: Is any label within `safe_zone` of the baseline, in the direction the curve bends? If yes → move the label.

**Check 2**: Does the curve cross any other arrow in the same figure? Curves bending down cross vertical arrows. Fix: reverse the bend direction.

---

### Pass 2: Label-between-nodes gap calculation

For every arrow label placed between two nodes:

```
Available gap  = center-to-center distance − half-width(A) − half-width(B)
Usable space   = Available gap − 0.6cm   [0.3cm padding each side]
```

Estimate label width using character counts:

| Font         | Width/char |
|--------------|-----------|
| `\scriptsize`  | 0.10 cm   |
| `\footnotesize`| 0.12 cm   |
| `\small`       | 0.15 cm   |
| `\normalsize`  | 0.18 cm   |
| Bold           | +10%      |

If **estimated label width > usable space**: guaranteed collision. Fix: move above/below, or shorten text, or increase node spacing.

**The most common fix** for labels that exceed the gap: break the label out as a standalone `\node` positioned above the arrow midpoint — clear of both boxes — rather than as an inline edge label.

---

### Pass 3: Arrow label positioning keywords

```bash
grep -n "node\[" [file].tex | grep -v "above\|below\|left\|right\|anchor\|pos\|midway\|near"
```

Every arrow label must carry a directional keyword. Any `node[...]` on an arrow without `above`, `below`, `left`, `right`, `anchor=`, `pos=`, or `midway` will sit ON the arrow line. Fix: add the keyword.

---

### Pass 4: Labels vs. drawn shapes (Boundary Rule)

For every `\node`, `\fill`, `\draw` circle/rectangle: compute the boundary. Any text within 0.4cm of a shape edge is a collision.

**Circles**: center `(cx, cy)`, radius `r` → boundary from `cy − r` to `cy + r`  
**Rectangles/boxes**: bottom-left `(x,y)`, width `w`, height `h` → top edge at `y + h`

If a label coordinate puts it inside or touching a shape boundary — move it 0.4cm outside.

**Note on `scale`**: `scale=0.8` shrinks coordinates but NOT text. A 2 cm gap becomes 1.6 cm, but a `\small` label stays `\small`. Recalculate all gaps accounting for scale.

---

### Pass 5: Margin check

Minimum clearances:

| Pair                          | Minimum |
|-------------------------------|---------|
| Label ↔ label                 | 0.3 cm  |
| Label ↔ axis line             | 0.3 cm  |
| Label ↔ arrow                 | 0.3 cm  |
| Arrow origin ↔ box edge       | 0.15 cm |
| Label ↔ shape boundary        | 0.4 cm  |
| Any object ↔ slide edge       | 0.5 cm  |

Also check:
- Multi-line nodes have `align=center` or `align=left`?
- No node text clipped by the slide margin?
- If `scale` is used: did text get unintentionally large relative to shrunken coordinates?

---

### Pass 6: Open the PDF and visually confirm

```bash
open [file].pdf
```

Scroll through every slide with a TikZ diagram. Trust your eyes for crowding, not just the math. What fits mathematically can still feel cramped. If it looks wrong, it is wrong.

---

## Step 3: Fix, recompile, repeat

After making fixes:

```bash
pdflatex -interaction=nonstopmode [file].tex 2>&1 | grep -E "Overfull|Underfull|Error|Warning"
```

Must return zero lines. Fix any new warnings introduced by the repositioning. Repeat until clean.

---

## Step 4: Re-audit the ENTIRE file after any fix

One collision fix often reveals a second one nearby, or introduces a new label that now crowds a different object. After every change, re-run Passes 1–5 on **all** TikZ figures in the file — not just the one you just touched.

```bash
grep -c "tikzpicture" [file].tex
```

That count is how many diagrams need a clean bill of health.

---

## Common collision patterns and fast fixes

| Pattern | Symptom | Fix |
|---------|---------|-----|
| Label between wide boxes | Text bleeds into box edge | Move label above arrow as standalone `\node` |
| Step 1 / Step 2 labels on flow diagrams | Labels overlap box text | Place labels above the arrow band, not as edge labels |
| Diagonal arrow label at `below right` | Label lands inside another box | Use `pos=0.55` + compute landing position |
| Return curve crossing vertical arrow | Arrows intersect visually | Reverse bend direction (`bend left` ↔ `bend right`) |
| Scale mismatch | Text large, gaps small | Redesign at intended size; avoid `scale` on complex diagrams |
| Slope label on regression line | Label sits on the line | Use `node[anchor=west]` at a point 0.3cm off the line end |
| `\\` inside `\textcolor{}` | Hbox overflow | Break into two separate `\textcolor` calls on separate lines |