---
name: garden-planner
description: Update, improve, or add features to the Square Foot Garden Bed Planner website. Use this skill whenever the user mentions the garden planner, garden beds, wants to add plants, change the layout, fix something on the planner, update the website, or asks about the garden planning tool. Trigger even for small requests like "add a plant" or "change the color" or "the garden planner needs...".
---

# Garden Planner Skill

## Project overview

- **File**: `/home/rune-keldsen/garden-planner/index.html` — single self-contained file (all HTML, CSS, and JS inline, no build step, no dependencies)
- **GitHub**: `https://github.com/runekeldsen/garden-planner` — remote already configured with PAT, just `git push`
- **Live site**: `https://garden-planner-roan.vercel.app`
- **Deploy**: pushing to `main` auto-deploys via Vercel — no manual deploy step needed

## Concept

Square Foot Gardening (SFG) planner for 4 raised beds (185×90cm each = 6×3 sq ft grid per bed).  
Each grid cell = 1 square foot. Dragging a plant into a cell shows how many plants fit in that square (e.g. carrots = ×16, tomato = ×1).

Key data structures in the JS:
- `PLANTS` array — plant definitions with `id`, `name`, `emoji`, `perSqFt`, `color`, `bg`, `cat`
- `beds` state — array of 4 beds, each with `id`, `name`, `cells` (flat array of 18 items, each `null` or `{ type: plantId }`)
- Persisted to `localStorage` under key `sfg-planner-v1`

## Workflow for every update

1. **Read the file first** — always read `index.html` before editing, even for small changes
2. **Make the change** using the Edit tool (prefer targeted edits over full rewrites)
3. **Commit and push**:
   ```bash
   cd /home/rune-keldsen/garden-planner
   git add index.html
   git commit -m "<short description of change>"
   git push
   ```
4. **Report** the live URL so the user can verify: `https://garden-planner-roan.vercel.app`

Vercel deploys in ~10 seconds after push. No need to wait or check — just tell the user it'll be live shortly.

## Common change patterns

**Adding a new plant** — add an entry to the `PLANTS` array with all required fields:
```js
{ id: 'unique_id', name: 'Display Name', emoji: '🌿', perSqFt: 4, color: '#hex', bg: '#lighthex', cat: 'Category' }
```
Valid categories: `Fruiting`, `Leafy greens`, `Root vegetables`, `Alliums`, `Fruits`, `Herbs`, `Legumes`

**Adding a new category** — add plants with a new `cat` value; `renderSidebar()` auto-groups by category.

**Changing perSqFt counts** — edit the `perSqFt` value in the `PLANTS` array entry.

**Styling changes** — CSS is all inline in the `<style>` block at the top of the file. Key selectors:
- `.bed` — the grid container (wood border, soil background)
- `.sq-cell` — individual square foot cell
- `.sq-cell.filled` — occupied cell
- `.plant-item` — sidebar palette entry

**Bed dimensions** — `GRID_COLS = 6`, `GRID_ROWS = 3`, `CELL_PX = 82` (pixels per cell)

**Drag behaviour** — handled by pointer events in `startDrag` / `onDragMove` / `onDragEnd`
