# Nonica-Revit-MCP-Skill

Skill files for the Nonica Revit MCP, enabling AI assistants to query and control Revit models via natural language. Covers model auditing, data extraction, view/sheet management, and geometry. Includes a standard edition and a local-LLM-optimized edition (Gemma 4 27B and smaller).

---

## Requirements

- [Nonica AI Connector for Revit](https://tools.nonica.io/AIConnector) — the MCP server that exposes Revit to AI assistants  
- [Nonica Autodesk App Store listing](https://apps.autodesk.com/RVT/en/Detail/Index?id=9212699819557407848&appLang=en&os=Win64)  
- Autodesk Revit 2022 or later  
- An MCP-compatible AI client (Claude, Cursor, Windsurf, or any local LLM with MCP support)

---
## Get Nonica Pro

Access 50+ MCP tools for Revit, Rhino, and more in a few clicks.  
Use the link below for a **5% discount** on Nonica Pro:

👉 /[Get Nonica Pro — 5% off with code RINO](https://buy.stripe.com/3cscNl1Hb2iY2SQ3cc?prefilled_promo_code=RINO)

Promo code: `RINO`

---
## Files

| File | Description |
|------|-------------|
| `SKILL.md` | Full reference for all 50+ Nonica Revit MCP tools. Covers parameters, return shapes, 26 gotchas, and common workflows. For use with frontier models (Claude, GPT-4o, Gemini, etc.) |
| `SKILL_LocalLLM.md` | Lean edition optimized for Gemma 4 27B and smaller local LLMs. Decision-tree structure, exact JSON call syntax, and hard rules to prevent the most common mistakes |

---
## Tutorials

| Tutorial | Link |
|----------|------|
| Claude + Revit MCP | [▶ Watch on YouTube](https://youtu.be/MAQ9hm3Crao?si=0A1uSM6zNCfqaiUb) |
| ChatGPT + Revit MCP | [▶ Watch on YouTube](https://youtu.be/Il5ohk_HMps?si=4lrsdynQZJzNGHSP) |


---
## Workflows Covered

### A - Model Health Audit
Check warnings, family bloat, worksets, and model quality indicators without touching any elements.

### B - Data Extraction
Read and write parameters in bulk across any category. Discover parameter IDs, extract values, and push changes back to Revit.

### C - View & Sheet Management
List views and sheets, report viewport placement, create new floor plan views, and apply or clear graphic color overrides on any element in any view.

### D - Geometry & Spatial
Get element location points, bounding boxes, boundary lines, and hosting information. Identify face-hosted elements and infer floor levels from Z coordinates.

---

## How to Use

1. Install the [Nonica AI Connector](https://tools.nonica.io/AIConnector) and enable it in Revit's Nonica tab
2. Connect your MCP-compatible AI client to the Nonica server
3. Paste the contents of the appropriate skill file into your AI client's system prompt or skill slot
4. Open a Revit model and start asking questions

**For frontier models** → use `SKILL.md`  
**For local LLMs (Gemma, Llama, Mistral, etc.)** → use `SKILL_LocalLLM.md`

---

## Validated Models

These skills were developed and validated live against:
- Snowdon Towers Sample Architectural (Revit default sample)
- Snowdon Towers Sample Plumbing (Revit default sample)
- RAC Basic Sample Project (Revit default sample)

The skills are fully generic — compatible with any Revit model in any language.

---

## Key Gotchas (from live testing)

- All coordinates are in **feet** in Revit's internal system — not screen-relative
- `get_all_elements_shown_in_view` **overflows** on complex 3D views — use `get_elements_by_category` instead
- Element creation always requires **three steps**: `create_tool_names_explorer` → `create_tool_arguments_explorer` → `create_tools_invoker`
- `model_title` in every response — if it changes, the user switched documents; re-orient immediately
- Graphic overrides can target **any view by ID**, not just the currently active view

See `SKILL.md` for the full list of 26 gotchas.

---

## License

MIT
