---
name: revit-mcp-local-llm
description: >
  Use this skill whenever a user asks anything about a Revit model connected via the Nonica MCP.
  This includes: reading element data, checking model quality, auditing warnings, extracting
  parameters, creating views or sheets, exporting PDFs, inspecting geometry, changing colors,
  moving elements, or writing parameter values. Trigger on any mention of Revit, families,
  walls, doors, rooms, sheets, schedules, levels, views, or BIM data.
---

# Revit MCP Skill — Gemma 4 / Local LLM Edition

Optimized for Gemma 4 27B (and smaller) connected to the Nonica Revit MCP client.
Rules come first. Follow them exactly — they prevent the most common mistakes.

---

## RULES — Read Before Every Task

**Rule 1 — Orient first.**
Always call `get_active_view_in_revit` at the start of a conversation. It gives you the model name, language, current view ID, and coordinate directions. You need these for almost everything.

**Rule 2 — Keywords must match the model language.**
`get_category_by_keyword` only works if the keyword is in the model's language (shown in the `language` field of any response). For English models use "Door", "Wall", "Room". Do not guess translations.

**Rule 3 — Get parameter IDs before bulk reads or writes.**
Never guess a parameter ID. Call `get_parameters_from_elementid` on one element first. Copy the exact integer ID from the result, then use it in `get_parameter_value_for_element_ids` or `set_parameter_value_for_elements`.

**Rule 4 — Material layers need TYPE IDs, not instance IDs.**
`get_material_layers_from_types` silently returns nothing if you pass instance IDs.
Correct workflow: `get_element_types_for_elementids([wallInstanceId])` → take TypeId → pass to material layers tool.

**Rule 5 — Avoid `get_all_elements_shown_in_view` on 3D views.**
It returns 60,000+ characters on complex views and will overflow your context.
Use `get_elements_by_category(categoryId)` for targeted queries instead.
`get_all_elements_shown_in_view` is safe for schedule IDs and simple plan views.

**Rule 6 — Type IDs with zero instances are not errors.**
`get_all_elementids_for_specific_type_ids` returns `[]` for type IDs that exist but have no placed instances. This is correct — not a bug.

**Rule 7 — Coordinates are internal, not screen-relative.**
All XYZ values are in Revit's internal coordinate system (feet). To interpret "up", "left", "north" in a view, use `screen_up_direction` and `screen_right_direction` from `get_active_view_in_revit`.

**Rule 8 — Isolation has no reset tool.**
`set_isolated_elements_in_view` cannot be undone via MCP. Warn the user before calling it. They must reset manually in Revit: View tab → Reset Temporary Hide/Isolate.

**Rule 9 — Creation is always three steps.**
`create_tool_names_explorer` → `create_tool_arguments_explorer` → `create_tools_invoker`.
Never skip the middle step — argument names must match exactly or the call will fail.

**Rule 10 — Delete cascades.**
`set_delete_elements` also deletes hosted elements and annotations. Always confirm with the user before calling it.

**Rule 11 — Monitor `model_title` in every response.**
Every MCP response contains a `model_title` field. If this value changes between calls, the user has switched the active Revit document. Stop immediately, call `get_active_view_in_revit`, and re-orient before continuing. All IDs, family names, and view IDs from the previous model are invalid.

---

## DECISION TREE — What to Call

Use this to pick the right tool without reading the full reference.

```
User asks about...
│
├── "what model / what view am I looking at?"
│     → get_active_view_in_revit
│
├── "find all [category] elements" (walls, doors, rooms, etc.)
│     → get_category_by_keyword("Door")          # get categoryId
│     → get_elements_by_category(categoryId)     # get instance IDs
│
├── "what families / types exist for [category]?"
│     → get_all_used_families_of_category(categoryId)
│     → get_all_used_types_of_families(["FamilyName"])
│
├── "read a parameter value" (mark, level, comments, area, etc.)
│     → get_parameters_from_elementid(oneElementId)   # discover param ID
│     → get_parameter_value_for_element_ids(ids, paramId)
│
├── "write / change a parameter value"
│     → get_parameters_from_elementid(oneElementId)   # find paramId
│     → set_parameter_value_for_elements(ids, paramId, ["value"])
│
├── "model health / warnings / issues"
│     → get_all_warnings_in_the_model
│     → get_all_used_families_in_model + get_size_in_mb_of_families (for bloat)
│
├── "wall / floor / roof material layers"
│     → get_element_types_for_elementids(instanceIds)   # get TypeId
│     → get_material_layers_from_types([TypeId])
│
├── "where is this element / geometry / coordinates"
│     → get_location_for_element_ids(ids)               # point/curve
│     → get_boundingboxes_for_element_ids(ids)          # min/max box
│     → get_boundary_lines(ids)                         # walls/floors/rooms only
│
├── "what sheet / view / schedule shows this?"
│     → get_viewports_and_schedules_on_sheets([sheetId])
│     → get_schedules_info_and_columns([scheduleId])
│
├── "color / highlight elements in a view"
│     → set_graphic_overrides_for_elements_in_view(ids, viewId, R, G, B)
│     → (to reset) same call with clearGraphics: true
│
├── "select / highlight elements for user to see"
│     → set_user_selection_in_revit(ids)
│
├── "create a view / sheet / grid / level"
│     → create_tool_names_explorer
│     → create_tool_arguments_explorer(["create_viewplans"])
│     → create_tools_invoker("create_viewplans", {...})
│
└── "export PDF"
      → get_elements_by_category(-2003100)               # -2003100 = Sheets (standard)
      → create_tool_arguments_explorer(["create_pdf_export_print"])
      → create_tools_invoker("create_pdf_export_print", {...})
```

---

## WORKFLOW A — Model Auditing

### Check warnings and model health

```jsonc
// Step 1: get all warnings
{ "tool": "get_all_warnings_in_the_model" }
// Returns: description, failingElementIds[], warningSeverity per warning

// Step 2 (optional): investigate a specific failing element
{ "tool": "get_parameters_from_elementid", "arguments": { "elementId": <failingElementId> } }
```

### Find oversized families (performance audit)

```jsonc
// Step 1: get all family IDs
{ "tool": "get_all_used_families_in_model" }
// Returns: { "<familyId>": "<FamilyName>", ... }

// Step 2: check sizes — pass the family IDs you got above (max 30 per call)
{ "tool": "get_size_in_mb_of_families",
  "arguments": { "list_elementIds": [<familyId1>, <familyId2>, <familyId3>] } }
// Returns: ordered list by MB + total model file size
// Note: some families return size=0 and "Cannot get file size" — this is normal
//       for in-place families or non-loadable types; skip them, not an error
```

### Find duplicate / overlapping elements

```jsonc
// Warnings tool already flags these — look for descriptions containing:
// "identical instances in the same place"
// "Highlighted walls overlap"
// "Highlighted floors overlap"
// Then get their IDs from failingElementIds[] and select them for the user:
{ "tool": "set_user_selection_in_revit",
  "arguments": { "list_elementIds": [<failingElementId1>, <failingElementId2>] } }
```

---

## WORKFLOW B — Data Extraction

### Read a parameter across all elements of a category

```jsonc
// Step 1: get category ID
{ "tool": "get_category_by_keyword", "arguments": { "keyword": "Door" } }
// → categoryId (e.g. -2000023 for Doors — standard across all models)

// Step 2: get all instance IDs for that category
{ "tool": "get_elements_by_category", "arguments": { "categoryId": <categoryId> } }
// → [<id1>, <id2>, ...]

// Step 3: discover parameter IDs on one element
{ "tool": "get_parameters_from_elementid", "arguments": { "elementId": <id1> } }
// → find paramId for "Mark" (-1001203), "Level" (-1001352), etc.
// Note: built-in param IDs like -1001203 are standard across all Revit models

// Step 4: bulk read that parameter across all instances
{ "tool": "get_parameter_value_for_element_ids",
  "arguments": { "list_elementIds": [<id1>, <id2>, <id3>], "idParameter": <paramId> } }
// → groups elements by shared value
```

### Read schedule data

```jsonc
// Step 1: find schedule element IDs (Schedules category = -2000573)
{ "tool": "get_elements_by_category", "arguments": { "categoryId": -2000573 } }

// Step 2: get schedule columns + parameter IDs
{ "tool": "get_schedules_info_and_columns",
  "arguments": { "list_elementIds": [<scheduleId>] } }
// → SchColumns[]: { ParId, ParName, Heading }

// Step 3: get rows listed in the schedule (safe — schedule IDs don't overflow)
{ "tool": "get_all_elements_shown_in_view", "arguments": { "viewOrSheetId": <scheduleId> } }

// Step 4: read values for a specific column using its ParId
{ "tool": "get_parameter_value_for_element_ids",
  "arguments": { "list_elementIds": [<rowId1>, <rowId2>], "idParameter": <colParId> } }
```

### Get wall material composition

```jsonc
// Step 1: get wall instance IDs
{ "tool": "get_elements_by_category", "arguments": { "categoryId": -2000011 } }

// Step 2: get wall TYPE IDs  ← required, not instance IDs
{ "tool": "get_element_types_for_elementids",
  "arguments": { "list_elementIds": [<wallInstanceId1>, <wallInstanceId2>] } }
// → TypeId: <typeId> = "<WallTypeName>"

// Step 3: get material layers using the TYPE IDs
{ "tool": "get_material_layers_from_types",
  "arguments": { "list_elementIds": [<typeId>] } }
// → MaterialName, MaterialWidth in mm
```

---

## WORKFLOW C — View & Sheet Management

### Create a floor plan view for a level

```jsonc
// Step 1: find level IDs (Levels category = -2000240)
{ "tool": "get_elements_by_category", "arguments": { "categoryId": -2000240 } }
// → [<levelId1>, <levelId2>, ...]

// Step 2: get arguments for view creation
{ "tool": "create_tool_arguments_explorer",
  "arguments": { "list_toolNames": ["create_viewplans"] } }
// → Arguments: (list_names, list_levelids, list_isceilingplan)

// Step 3: create the view using the level ID from Step 1
{ "tool": "create_tools_invoker",
  "arguments": {
    "toolName": "create_viewplans",
    "argumentIdsAndValues": {
      "list_names": ["Level 1 - New Plan"],
      "list_levelids": [<levelId1>],
      "list_isceilingplan": [false]
    }
  }
}
```

### Export sheets to PDF

```jsonc
// Step 1: get sheet IDs (Sheets category = -2003100)
{ "tool": "get_elements_by_category", "arguments": { "categoryId": -2003100 } }

// Step 2: get sheet numbers to sort them (-1007401 = Sheet Number, built-in)
{ "tool": "get_parameter_value_for_element_ids",
  "arguments": { "list_elementIds": [<sheetId1>, <sheetId2>], "idParameter": -1007401 } }
// Sort by sheet number before exporting

// Step 3: export (folderpath MUST end with \\)
{ "tool": "create_tools_invoker",
  "arguments": {
    "toolName": "create_pdf_export_print",
    "argumentIdsAndValues": {
      "list_filenames": ["ProjectSheets.pdf"],
      "list_viewids": [<sheetId1>, <sheetId2>],
      "folderpath": "C:\\Users\\<username>\\Desktop\\",
      "combineinonefile": true
    }
  }
}
```

### Color-code elements by parameter value

```jsonc
// Example: color fire-rated doors red

// Step 1: find door family types
{ "tool": "get_all_used_types_of_families",
  "arguments": { "familyNames": ["<DoorFamilyName>"] } }
// → find types with fire-rating in name (e.g. "MIN", "FR", "60 min")

// Step 2: get instance IDs for those types
{ "tool": "get_all_elementids_for_specific_type_ids",
  "arguments": { "list_elementIds": [<fireRatedTypeId>] } }

// Step 3: apply red override in the active view
{ "tool": "set_graphic_overrides_for_elements_in_view",
  "arguments": {
    "list_elementIds": [<instanceId1>, <instanceId2>],
    "viewId": <activeViewId>,
    "red": 255, "green": 0, "blue": 0
  }
}

// Step 4: clear overrides when done
{ "tool": "set_graphic_overrides_for_elements_in_view",
  "arguments": {
    "list_elementIds": [<instanceId1>, <instanceId2>],
    "viewId": <activeViewId>,
    "red": 0, "green": 0, "blue": 0,
    "clearGraphics": true
  }
}
```

---

## WORKFLOW D — Geometry & Spatial

### Get element locations and sizes

```jsonc
// Location point (doors, walls, columns — NOT floors)
{ "tool": "get_location_for_element_ids",
  "arguments": { "list_elementIds": [<elementId1>, <elementId2>] } }
// → LocationPoint: "(X, Y, Z)" in feet

// Bounding box (any element)
{ "tool": "get_boundingboxes_for_element_ids",
  "arguments": { "list_elementIds": [<elementId1>, <elementId2>] } }
// → [(minX, minY, minZ), (maxX, maxY, maxZ)] in feet

// Actual edge geometry (walls, floors, rooms only — max 30 IDs)
{ "tool": "get_boundary_lines",
  "arguments": { "list_elementIds": [<wallOrRoomId>] } }
```

### Move elements

```jsonc
// Step 1: orient yourself — screen_right_direction tells you which axis is "right" in the view
{ "tool": "get_active_view_in_revit" }
// → screen_right_direction: "(X, Y, Z)"  ← use this to translate screen direction to model axes

// Step 2: move the element by a vector in feet
{ "tool": "set_movement_for_elements",
  "arguments": {
    "list_elementIds": [<elementId>],
    "mov_vect_X": [5.0],
    "mov_vect_Y": [0.0],
    "mov_vect_Z": [0.0]
  }
}
```

### Find which room a door connects

```jsonc
// get_all_additional_properties gives FromRoom and ToRoom on FamilyInstance elements
{ "tool": "get_all_additional_properties_from_elementid",
  "arguments": { "elementId": <doorElementId> } }
// → FromRoom: "<roomId> (<RoomName>)"
// → ToRoom:   "<roomId> (<RoomName>)"
```

---

## QUICK TOOL REFERENCE

| Tool | Required Args | Max IDs | Notes |
|------|--------------|---------|-------|
| `get_active_view_in_revit` | — | — | Call first every session |
| `get_category_by_keyword` | `keyword` (str) | — | Model language required |
| `get_elements_by_category` | `categoryId` (int) | — | Fastest element lookup |
| `get_all_used_families_in_model` | — | — | Loadable families only |
| `get_all_used_families_of_category` | `categoryId` | — | Loadable families only |
| `get_all_used_types_of_families` | `familyNames[]` | 30 names | Works for system families too |
| `get_all_elementids_for_specific_type_ids` | `list_elementIds[]` | 50 | Empty = unplaced type (not an error) |
| `get_all_elements_of_specific_families` | `familyNames[]` | 30 names | Exact match required |
| `get_parameters_from_elementid` | `elementId` | 1 | Always call first to find param IDs |
| `get_all_additional_properties_from_elementid` | `elementId` | 1 | API-level props: UniqueId, OwnerViewId, FromRoom/ToRoom |
| `get_parameter_value_for_element_ids` | `list_elementIds[]`, `idParameter` | 500 | Needs param ID from above |
| `get_element_types_for_elementids` | `list_elementIds[]` | 500 | Returns TypeId + TypeName |
| `get_material_layers_from_types` | `list_elementIds[]` | 100 | TYPE IDs only — not instances |
| `get_boundingboxes_for_element_ids` | `list_elementIds[]` | 500 | In feet |
| `get_location_for_element_ids` | `list_elementIds[]` | 500 | Not floors |
| `get_boundary_lines` | `list_elementIds[]` | 30 | Walls / Floors / Rooms only |
| `get_host_id_for_element_ids` | `list_elementIds[]` | 200 | Doors/windows → host wall |
| `get_all_warnings_in_the_model` | — | — | Model health check |
| `get_size_in_mb_of_families` | `list_elementIds[]` | 30 | Family IDs, not linked docs |
| `get_schedules_info_and_columns` | `list_elementIds[]` | 50 | Schedule element IDs only |
| `get_viewports_and_schedules_on_sheets` | `list_elementIds[]` | 100 | Sheet IDs only |
| `get_graphic_filters_applied_to_views` | `list_elementIds[]` | 50 | View IDs |
| `get_all_elements_shown_in_view` | `viewOrSheetId` | 1 | ⚠ Avoid on 3D views |
| `set_user_selection_in_revit` | `list_elementIds[]` | — | Accepts type/category IDs too |
| `set_isolated_elements_in_view` | `list_elementIds[]`, `viewId` | — | ⚠ No MCP reset — warn user |
| `set_graphic_overrides_for_elements_in_view` | `list_elementIds[]`, `viewId`, `red/green/blue` | — | `clearGraphics: true` to reset |
| `set_parameter_value_for_elements` | `list_elementIds[]`, `idParameter`, `list_newValues[]` | — | Values as strings with units |
| `set_movement_for_elements` | `list_elementIds[]`, `mov_vect_X/Y/Z[]` | 100 | In feet |
| `set_delete_elements` | `list_elementIds[]` | — | ⚠ Cascades — confirm first |
| `create_tool_names_explorer` | — | — | Step 1 of creation workflow |
| `create_tool_arguments_explorer` | `list_toolNames[]` | — | Step 2 of creation workflow |
| `create_tools_invoker` | `toolName`, `argumentIdsAndValues{}` | — | Step 3 — exact arg names |

---

## Common Category IDs

```
Walls         -2000011    Doors         -2000023    Windows       -2000014
Floors        -2000032    Roofs         -2000035    Ceilings      -2000038
Rooms         -2000160    Stairs        -2000120    Columns       -2000100
Furniture     -2000080    Sheets        -2003100    Schedules     -2000573
Levels        -2000240    Grids         -2000220    Views         -2000279
Generic Model -2000151    Casework      -2001000    Planting      -2001360
```

---

## Error Recovery

| Symptom | Cause | Fix |
|---------|-------|-----|
| `get_material_layers_from_types` returns empty | Passed instance IDs | Call `get_element_types_for_elementids` first, use TypeId |
| `get_all_elementids_for_specific_type_ids` returns `[]` | Type has no placed instances | Not an error — type exists but is unused |
| `get_category_by_keyword` returns no match | Wrong language | Check `language` field in any response; try in that language |
| `get_all_elements_shown_in_view` overflows | 3D view too complex | Switch to `get_elements_by_category(categoryId)` |
| `set_parameter_value_for_elements` writes nothing | Parameter is read-only | Check `par_read_only_in_element` list in `get_parameters_from_elementid` output |
| `create_tools_invoker` fails | Wrong argument name | Re-run `create_tool_arguments_explorer` and copy names exactly |
| Coordinates look wrong | Forgot screen directions | Call `get_active_view_in_revit` → use `screen_right_direction` / `screen_up_direction` |
| Any tool returns "Revit UI was blocked" | Temporary Revit UI lock | Retry the exact same call — it resolves automatically, no user action needed |
| `get_size_in_mb_of_families` returns size `0` | In-place or non-loadable family | Expected behaviour — skip those entries, the rest of the list is still valid |
| `model_title` changes between calls | User switched active Revit document | Stop. Call `get_active_view_in_revit`. Re-orient completely — all IDs from the previous model are invalid |
| Element has `LevelId: -1` | Hosted on a linked model face | Not an error — the element is face-hosted on a link, not level-based. Infer level from Z coordinate via `get_location_for_element_ids` |
| `HasSpatialElementFromToCalculationPoints: False` | Element type has no room connections | Expected for plumbing fixtures, MEP elements; FromRoom/ToRoom only available on doors and windows |
| Need a stable element identifier across model versions | Integer IDs can change after central file sync | Use `UniqueId` (GUID) from `get_all_additional_properties_from_elementid` — it persists across versions |
| Unsure if element is a model element or a view-specific detail | Can't tell from parameters | Check `OwnerViewId` in `get_all_additional_properties_from_elementid`: `-1` = model element, positive = view-specific |
| Need to color elements without switching the active view | Graphic override requires view ID, not active view | Pass any view ID to `set_graphic_overrides_for_elements_in_view` — the active view does not need to change |

---

*Nonica Revit MCP — Compatible with any Revit model | Optimized for Gemma 4 27B and smaller*
