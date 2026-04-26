# Nonica Revit MCP — SKILL.md

> Comprehensive reference for all Nonica Revit MCP tools.  
> Covers tool parameters, return shapes, and lessons learned from live testing.

---

## Quick-start Workflow

```
1. Orient yourself          → get_active_view_in_revit
2. Find a category          → get_category_by_keyword("Wall")
3. Get elements             → get_elements_by_category(categoryId)
4. Inspect one element      → get_parameters_from_elementid(elementId)
5. Read a param in bulk     → get_parameter_value_for_element_ids(ids[], paramId)
6. Write / act              → set_parameter_value_for_elements / set_* tools
```

---

## Tool Reference

### GROUP 1 — Zero-parameter Context Tools

These require no input and are safe to call anytime for orientation.

| Tool | Returns |
|------|---------|
| `get_active_view_in_revit` | Current view ID, title, `screen_up_direction`, `screen_right_direction`, open views list, units |
| `get_all_project_units` | 150 unit specs: Spec name → unit string → symbol |
| `get_all_warnings_in_the_model` | All warnings: failureGuid, description, failing element IDs, severity |
| `get_all_workset_information` | Workset list + `is_document_workshared` flag |
| `get_model_categories` | All categories: `{Id, Name}` (OST_ prefix names) |
| `get_user_selection_in_revit` | Currently selected IDs in main doc + linked docs |
| `create_tool_names_explorer` | 14 available creation tool names |
| `get_all_used_families_in_model` | All loadable family IDs + names (NOT system families) |

**⚠️ Lesson:** `get_model_categories` can return hundreds of rows — expensive. Prefer `get_category_by_keyword` first.

---

### GROUP 2 — Discovery / Navigation

#### `get_category_by_keyword(keyword)`
- **Use first** to find category IDs efficiently
- `keyword` must be in the **model's language** (check `language` field returned by any tool)
- Returns matching categories — often 2 results: the main category + its tags category

#### `get_all_used_families_of_category(categoryId)`
- Returns loadable family names + IDs for a category
- Does NOT include system families (walls, floors, etc.)

#### `get_all_used_types_of_families(familyNames[])`
- Returns all type IDs + names per family
- **Works for both loadable AND system families**
- `familyNames` must match exactly (case-sensitive)

#### `get_all_elementids_for_specific_type_ids(list_elementIds[])`
- Returns instance element IDs for each type ID
- **⚠️ Returns empty array for types that exist but have no placed instances** — this is correct, not an error
- A type ID that returns `[]` means the type definition exists in the model but no instances have been placed
- MAX: 50 type IDs per request

#### `get_all_elements_of_specific_families(familyNames[])`
- Returns all instance IDs per family name
- More direct than type-based lookup when you want all instances of a family
- `familyNames` must match exactly; MAX: 30 names per request

#### `get_elements_by_category(categoryId)`
- Returns all instance element IDs for a category
- Fast — use this for bulk element discovery

#### `get_all_elements_shown_in_view(viewOrSheetId)`
- Works with views, schedules, and sheets
- **⚠️ WARNING: Causes token overflow on complex 3D views** (output can exceed 60,000 characters on large models)
- **Best used with**: sheet IDs, schedule IDs, or simple plan views
- For 3D views with many elements, prefer `get_elements_by_category` instead

---

### GROUP 3 — Element Properties

#### `get_parameters_from_elementid(elementId)`
- **Always call this first** on any new element type to discover parameter IDs
- Returns `par_write_in_element` + `par_read_only_in_element`, each with `{Id, Name, Value}`
- Note at end: "More as additional properties" → use `get_all_additional_properties_from_elementid` for API-level props
- Common built-in parameter IDs (consistent across all Revit models):
  - `-1010106` → Comments (String, writable)
  - `-1001203` → Mark (writable)
  - `-1001362` → Head Height (doors/windows)
  - `-1002002` → Family Name (read-only)
  - `-1002001` → Type Name (read-only)
  - `-1002108` → Host Id (read-only)
  - `-1007401` → Sheet Number (sheets)
  - `-1007400` → Sheet Name (sheets)

#### `get_parameter_value_for_element_ids(list_elementIds[], idParameter)`
- Bulk read of one parameter across many elements
- Get `idParameter` from `get_parameters_from_elementid` first
- MAX: 500 element IDs per request
- Groups elements with identical values together in output

#### `get_all_additional_properties_from_elementid(elementId)`
- Returns API-level (Revit DB class) properties — writable + read-only separated
- Examples for `FamilyInstance`: `Pinned`, `HandFlipped`, `FacingFlipped`, `FromRoom`, `ToRoom`, `UniqueId`, `Location`, `CreatedPhaseId`
- Use only when `get_parameters_from_elementid` doesn't expose what you need

#### `get_additional_property_for_all_elementids(propertyName, list_elementIds[])`
- Bulk read of one API-level property
- `propertyName` is **case-sensitive** and must match exactly (e.g., `"Name"`, `"Pinned"`)
- MAX: 500 IDs per request

---

### GROUP 4 — Geometry

#### `get_boundingboxes_for_element_ids(list_elementIds[])`
- Returns `[min XYZ, max XYZ]` in **feet** for each element
- Optional: `idSheet` for sheet-space bounding boxes
- MAX: 500 IDs per request
- Use for arrangement, spacing, and collision checks

#### `get_location_for_element_ids(list_elementIds[])`
- Returns `LocationPoint` or `LocationCurve` per element
- Units: feet; rotation in radians
- **⚠️ Coordinates are Revit's internal axes — NOT screen-relative**
- Use `get_active_view_in_revit` → `screen_up_direction` + `screen_right_direction` to interpret spatial relationships visually
- **Does NOT work with floors**
- MAX: 500 IDs per request

#### `get_boundary_lines(list_elementIds[])`
- Returns actual edge geometry as start/end point pairs
- **Only works with: Walls, Floors, Rooms**
- More precise than bounding boxes for spatial analysis
- MAX: 30 IDs per request

---

### GROUP 5 — Type / Category Inspection

#### `get_categories_from_elementids(list_elementIds[])`
- Returns category ID + name for each element, grouped
- MAX: 1000 IDs per request

#### `get_object_classes_from_elementids(list_elementIds[])`
- Returns full Revit API class name (e.g., `Autodesk.Revit.DB.FamilyInstance`)
- Useful to determine which additional properties are available
- MAX: 1000 IDs per request

#### `get_element_types_for_elementids(list_elementIds[])`
- Returns `{TypeId, TypeName}` with the instance IDs that share each type
- MAX: 500 IDs per request

#### `get_host_id_for_element_ids(list_elementIds[])`
- Returns the host/tagged element ID for hosted elements
- Works with: doors, windows, tags, MEP fixtures
- Use to trace doors → walls, tags → tagged elements

#### `get_material_layers_from_types(list_elementIds[])`
- Returns material layers: `{MaterialId, MaterialName, MaterialWidth}` in mm
- **⚠️ Must receive TYPE IDs** (WallType, FloorType, RoofType, CeilingType) — NOT instance IDs
- Workflow: `get_element_types_for_elementids` → take TypeId → pass to this tool
- MAX: 100 IDs per request

---

### GROUP 6 — Worksets (Collaborative Models)

#### `get_worksets_from_elementids(list_elementIds[])`
- Returns workset assignment per element
- **Graceful response** if model is not workshared — returns friendly message
- MAX: 1000 IDs per request

#### `get_worksharing_information_for_element_ids(list_elementIds[])`
- Returns workset name, creator, owner, last changed by per element
- **Graceful response** if not workshared
- MAX: 100 IDs per request

---

### GROUP 7 — Views, Sheets, Schedules

#### `get_schedules_info_and_columns(list_elementIds[])`
- Input: schedule element IDs (category `OST_Schedules`)
- Returns: schedule title, category, columns with `{ParId, ParName, Heading, IsHidden}`
- Chain with `get_all_elements_shown_in_view` + `get_parameter_value_for_element_ids` to read schedule data

#### `get_viewports_and_schedules_on_sheets(list_elementIds[])`
- Input: sheet IDs
- Returns: title block ID, sheet frame, viewports + schedule instances with their frames and referenced view IDs
- MAX: 100 IDs per request

#### `get_graphic_filters_applied_to_views(list_elementIds[])`
- Returns all view filters per view: filter ID, name, type (`ParameterFilter`), filtered categories
- MAX: 50 IDs per request

#### `get_graphic_overrides_for_element_ids_in_view(viewId, list_elementIds[])`
- Returns element-level graphic overrides (highest priority layer)
- Returns `"Graphics were not overrided in the view."` when using defaults — this is normal, not an error
- Also returns category-level overrides for the categories involved
- Priority: Element override > View filter override > Category override
- **Does NOT work with elements in linked documents**
- MAX: 100 IDs per request

#### `get_graphic_overrides_view_filters(viewId, list_filterIds[])`
- Returns filter-level graphic overrides (medium priority layer)
- Get filter IDs from `get_graphic_filters_applied_to_views` first
- MAX: 100 IDs per request

#### `get_if_elements_pass_filter(filterId, list_elementIds[])`
- Returns three lists: passing, not passing, invalid
- Useful for QA workflows — test elements against a view filter condition
- MAX: 1000 IDs per request

---

### GROUP 8 — Family Stats

#### `get_size_in_mb_of_families(list_elementIds[])`
- Input: **family element IDs** (from `get_all_used_families_in_model`)
- Returns ordered list by file size + model file size
- **⚠️ Do NOT use with elements from linked documents**
- Can be slow for large families
- MAX: 30 IDs per request
- Some families return `"Cannot get file size"` with size `0` — this is expected for in-place families or certain non-loadable types; not an error

---

### GROUP 9 — Document Switching (Linked Models)

#### `get_document_switched(elementId?, switchMainDoc?)`
- Switch context to a linked Revit document
- `elementId`: ID of a `RevitLinkInstance` (optional, default -1)
- `switchMainDoc`: set `true` to switch back to the main document
- After switching, **all subsequent tool calls operate in the switched document**
- Always switch back with `switchMainDoc: true` when done

---

### GROUP 10 — Non-Destructive SET Tools

Safe to use; visual or selection changes only.

#### `set_user_selection_in_revit(list_elementIds[])`
- Selects elements in Revit, visible to user
- **Accepts element IDs, type IDs, OR category IDs**
- Tip: pass a type ID to select all instances of that type — no need to expand to individual IDs first
- Overrides current selection

#### `set_isolated_elements_in_view(list_elementIds[], viewId)`
- Isolates elements in a view
- **Accepts element IDs, type IDs, OR category IDs**
- **⚠️ No reset/clear isolation tool exists in the MCP** — user must manually reset in Revit (View tab → Reset Temporary Hide/Isolate)

#### `set_graphic_overrides_for_elements_in_view(list_elementIds[], viewId, red, green, blue, clearGraphics?)`
- Sets element color override in a view using RGB values (0–255)
- Set `clearGraphics: true` to reset element back to default appearance (RGB values ignored when clearing)
- Does NOT work with elements in linked documents

---

### GROUP 11 — Destructive SET Tools

**Always confirm with user before executing. Changes persist in the Revit model.**

#### `set_parameter_value_for_elements(list_elementIds[], idParameter, list_newValues[])`
- Bulk-writes a parameter value across elements
- `idParameter`: get from `get_parameters_from_elementid`
- `list_newValues`: strings; include unit acronym for numeric values (e.g., `"2.12 m"`, `"5 ft"`, `"90 °"`)
- Supply one value to apply the same to all; or match list length to `list_elementIds` for per-element values
- Returns `StorageType`, `DataType`, and a grouped summary of what was written

#### `set_additional_property_for_all_elements(propertyName, list_elementIds[], list_newValues[])`
- Writes an API-level property (e.g., `"Pinned"`, `"Name"`)
- `propertyName` is **case-sensitive**
- Returns `expected_format` in response — check this if write fails
- MAX: 500 IDs per request

#### `set_copy_elements(list_elementIds[], mov_vect_X[], mov_vect_Y[], mov_vect_Z[])`
- Copies elements with a translation vector in **feet**
- Hosted elements are copied with their host
- Use `get_active_view_in_revit` screen directions to compute correct X/Y vectors
- MAX: 100 IDs per request

#### `set_movement_for_elements(list_elementIds[], mov_vect_X[], mov_vect_Y[], mov_vect_Z[])`
- Moves elements by a translation vector in **feet**
- Hosted elements are projected/corrected onto their host
- List order of IDs must match list order of vectors
- MAX: 100 IDs per request

#### `set_rotation_for_elements(list_elementIds[], axis_start_*, axis_end_*, list_angles[])`
- Rotates elements around an axis defined by start + end points
- `list_angles`: radians; **positive = counterclockwise**
- Default axis: element's location point + Z axis (if axis not specified)
- MAX: 100 IDs per request

#### `set_copy_view_filters(origen_viewId, list_filterIds[], target_viewIds[])`
- Copies filter definitions + graphics from one view to others
- Get filter IDs from `get_graphic_filters_applied_to_views`

#### `set_revisions_on_sheets(list_sheetIds[], list_revisionIds[], assignRevisions)`
- Assigns (`true`) or unassigns (`false`) revisions to sheets
- Cannot unassign if RevisionClouds exist on the sheet or any views on it

#### `set_delete_elements(list_elementIds[])`
- **Permanently deletes** elements from the document
- Also deletes: hosted elements, annotations, and dependent sub-elements
- Returns `IdsDeleted` + `IdsNotDeleted`
- ⚠️ When deleting a view, Revit deletes the view + all its dependent elements (10 IDs deleted for a single 3D view)

---

### GROUP 12 — Creation Tools (3-step workflow)

Always follow this sequence:

```
Step 1: create_tool_names_explorer()           → get list of 14 tool names
Step 2: create_tool_arguments_explorer([...])  → get argument signatures
Step 3: create_tools_invoker(toolName, {...})  → execute
```

#### Available creation tools and their key arguments:

| Tool Name | Key Arguments |
|-----------|--------------|
| `create_grids` | `list_names[]`, axis start/end XYZ (6 arrays) |
| `create_levels` | `list_names[]`, `list_levelheights[]` (in feet) |
| `create_viewplans` | `list_names[]`, `list_levelids[]`, `list_isceilingplan[]` |
| `create_view3ds` | `list_names[]` |
| `create_viewsections_or_detailviews` | names, section base start/end XYZ, depth, height, `list_isdetailview[]` |
| `create_sheets` | `list_names[]`, `list_specifictitleblocktypeids[]` (optional) |
| `create_viewports_and_schedules_on_sheet` | (use explorer) |
| `create_schedule` | `schedulename`, `categoryid`, `list_parameteridsforcolumns[]` |
| `create_legends_or_draftingviews` | (use explorer) |
| `create_textnotes_on_view` | (use explorer) |
| `create_room_elevationviews` | (use explorer) |
| `create_pdf_export_print` | `list_filenames[]`, `list_viewids[]`, `folderpath`, `combineinonefile` |
| `create_cad_export_print` | (use explorer) |
| `create_tags_on_view` | `list_ids[]`, `viewid`, `offset_x/y`, `addleader`, `addelbowhorizleader` |

**Critical notes for creation tools:**

- **Grids**: `+X` = East, `+Y` = North
- **Sections**: What's visible to the RIGHT when standing at `sectbase_start` looking toward `sectbase_end`
  - Sectbase along `+X` → View looks South
  - Sectbase along `+Y` → View looks East
- **Tags**: For `list_ids` — use category ID for all elements, type IDs for type-filtered, instance IDs for instance-filtered; never pass all instance IDs when a category/type ID works
- **PDF export**: `folderpath` must end with `\\`; order sheets by number when combining
- **`argumentIdsAndValues` in `create_tools_invoker`**: Must be a JSON object matching the argument names exactly

---

## Common Workflows

### Find and inspect all doors of a fire-rated type
```
1. get_category_by_keyword("Door")                       → categoryId (e.g. -2000023)
2. get_all_used_families_of_category(categoryId)         → family names + IDs
3. get_all_used_types_of_families(["<DoorFamilyName>"])  → TypeIds + names
   → identify fire-rated types by name (e.g. containing "MIN" or "FR")
4. get_all_elementids_for_specific_type_ids([typeId])    → instance IDs
5. get_parameters_from_elementid(firstInstanceId)        → discover param IDs
6. get_parameter_value_for_element_ids(instanceIds, -1001203)  → Mark values
```

### Read schedule data
```
1. get_elements_by_category(-2000573)                  → schedule element IDs
   (or use OST_Schedules category)
2. get_schedules_info_and_columns([scheduleId])        → column ParIds
3. get_all_elements_shown_in_view(scheduleId)          → listed instance IDs
4. get_parameter_value_for_element_ids(instanceIds, paramId) → cell values
```

### Color-highlight elements by condition
```
1. get_elements_by_category(categoryId)
2. get_parameter_value_for_element_ids(ids, paramId)   → filter by value
3. set_graphic_overrides_for_elements_in_view(matchingIds, viewId, R, G, B)
4. set_graphic_overrides_for_elements_in_view(matchingIds, viewId, 0,0,0, clearGraphics:true)  → reset
```

### Get wall composition
```
1. get_elements_by_category(-2000011)                  → wall instance IDs
2. get_element_types_for_elementids(wallInstanceIds)   → wall TypeIds
3. get_material_layers_from_types(wallTypeIds)         → material layers in mm
```

### Export sheets to PDF
```
1. get_elements_by_category(-2003100)                        → sheet IDs (OST_Sheets)
2. get_parameter_value_for_element_ids(sheetIds, -1007401)   → sheet numbers (sort)
3. create_tools_invoker("create_pdf_export_print", {
     list_filenames: ["ProjectSheets.pdf"],
     list_viewids: [sheetId1, sheetId2, ...],  // sorted by number
     folderpath: "C:\\Users\\<username>\\Desktop\\",
     combineinonefile: true
   })
```

---

## Important Gotchas & Lessons Learned

| # | Lesson | Detail |
|---|--------|--------|
| 1 | **Keyword language** | `get_category_by_keyword` requires the model's language — check `language` field returned in any tool response |
| 2 | **Type IDs with no instances** | `get_all_elementids_for_specific_type_ids` correctly returns `[]` for type IDs that have no placed instances — not a bug |
| 3 | **View overflow** | `get_all_elements_shown_in_view` on a complex 3D view returns 60,000+ characters and causes token overflow — use `get_elements_by_category` for targeted queries instead |
| 4 | **Internal coordinates** | All XYZ are Revit's internal coordinate system — use `screen_up_direction` / `screen_right_direction` from `get_active_view_in_revit` to interpret spatial relationships |
| 5 | **Material layers need type IDs** | `get_material_layers_from_types` fails silently on instance IDs — always pass wall/floor/roof TYPE IDs |
| 6 | **Worksharing graceful** | Both workset tools return friendly messages on non-workshared models — no error handling needed |
| 7 | **Parameter ID discovery** | Always call `get_parameters_from_elementid` on a representative element before doing bulk parameter reads or writes |
| 8 | **3-step creation** | Never skip `create_tool_arguments_explorer` — argument names must match exactly in `create_tools_invoker` |
| 9 | **No isolation reset** | `set_isolated_elements_in_view` has no inverse tool — user must manually reset in Revit (View tab → Reset Temporary Hide/Isolate) |
| 10 | **Loadable vs system families** | `get_all_used_families_in_model` / `get_all_used_families_of_category` only return loadable families; system families (walls, floors, ceilings) only appear via `get_all_used_types_of_families` |
| 11 | **Parameter values as strings** | `set_parameter_value_for_elements` and `set_additional_property_for_all_elements` take string values; include unit acronym for numeric params (e.g., `"2.12 m"`) |
| 12 | **Type/category for selection** | `set_user_selection_in_revit` and `set_isolated_elements_in_view` accept type or category IDs directly — far more efficient than expanding to instance IDs |
| 13 | **Delete cascades** | `set_delete_elements` deletes hosted elements and annotations too; deleting a 3D view deleted 10 IDs (view + dependent sub-elements) |
| 14 | **get_boundary_lines scope** | Only works with Walls, Floors, Rooms — passing other element types silently returns nothing |
| 15 | **Location doesn't work on floors** | `get_location_for_element_ids` explicitly does not support Floor elements |
| 16 | **PDF folderpath format** | Must end with `\\` (e.g. `"C:\\Users\\name\\"`) or be empty string (same folder as .rvt file) |
| 17 | **Section orientation** | Section views show what's to the RIGHT when standing at `sectbase_start` and looking toward `sectbase_end` |
| 18 | **set_additional_property format** | Response includes `expected_format` (e.g., `System.Boolean`) — check this when a write fails |
| 19 | **Transient UI block** | Any tool can return "Revit UI was blocked by another command/tool or window" — this can happen mid-session with no user action; simply retry the call, it recovers automatically |
| 20 | **Family file size unavailable** | `get_size_in_mb_of_families` returns `0` and `"Cannot get file size"` for in-place families and certain non-loadable types — not an error, just no file to measure |
| 21 | **Mid-session model switch** | `model_title` in the response header can change between calls if the user switches the active Revit document. Always check `model_title` on every response; if it changes, call `get_active_view_in_revit` immediately to re-orient before continuing |
| 22 | **Linked model hosting** | Elements placed on linked model faces (e.g., roof drains on an architectural link) will have `LevelId: -1` and their `Host` / `HostFace` property will reference the linked model path. This is expected — the element is not level-based |
| 23 | **Plumbing fixtures have no FromRoom/ToRoom** | `get_all_additional_properties_from_elementid` returns `HasSpatialElementFromToCalculationPoints: False` for plumbing fixtures. Room connections only exist on doors, windows, and similar opening elements |
| 24 | **`get_all_additional_properties_from_elementid` reveals hidden metadata** | Returns fields not exposed by `get_parameters_from_elementid`: `OwnerViewId` (-1 = model element, positive = view-specific detail element), `UniqueId` (stable GUID that persists across model versions and central file syncs — unlike the integer ID which can change), and `MEPModel` (flags MEP connector data available on the element class) |
| 25 | **Graphic overrides target any view by ID — not just the active one** | `set_graphic_overrides_for_elements_in_view` accepts any valid view ID regardless of which view the user is currently looking at. You can pre-color elements in a background view without switching to it first |
| 26 | **Face-hosted elements: infer level from Z coordinate** | Elements with `LevelId: -1` (hosted on linked model faces) cannot be grouped by level using parameter queries. Use `get_location_for_element_ids` to get the Z coordinate in feet and compare against known level elevations to infer which floor they belong to |

---

## Response Format Reference

Every Revit MCP response includes this header:
```yaml
type: ContextUpdate
source: RevitModel
timestamp: "2026-04-26T..."
language: English_USA           # ← use this for keyword searches
model_title: <Your Project Name>   # ← CHECK THIS EVERY RESPONSE — can change if user switches document
is_linked_doc: false            # ← true if switched to a linked doc
```

> **⚠️ Model switch detection:** If `model_title` changes between calls, stop and call `get_active_view_in_revit` immediately. All IDs, family names, and view IDs from the previous model are invalid in the new one.

Common response patterns:
- **Grouped by value**: `get_parameter_value_for_element_ids` groups elements sharing a value
- **Grouped by type**: `get_element_types_for_elementids` groups instances under their type
- **Grouped by category**: `get_categories_from_elementids` groups instances under category
- **Pass/fail/invalid**: `get_if_elements_pass_filter` returns three separate lists
- **Error "not overrided"**: Normal response from graphic override tools when defaults are in use

---

## Category ID Quick Reference (Standard Revit Built-in IDs)

| Category | ID |
|----------|----|
| Walls | `-2000011` |
| Doors | `-2000023` |
| Windows | `-2000014` |
| Floors | `-2000032` |
| Roofs | `-2000035` |
| Ceilings | `-2000038` |
| Rooms | `-2000160` |
| Stairs | `-2000120` |
| Columns | `-2000100` |
| Structural Framing | `-2001320` |
| Curtain Wall Panels | `-2000170` |
| Curtain Wall Mullions | `-2000171` |
| Furniture | `-2000080` |
| Casework | `-2001000` |
| Sheets | `-2003100` |
| Schedules | `-2000573` |
| Levels | `-2000240` |
| Grids | `-2000220` |
| Views | `-2000279` |
| Viewports | `-2000510` |
| Rooms | `-2000160` |
| MEP Spaces | `-2003600` |
| Plumbing Fixtures | `-2001160` |
| Lighting Fixtures | `-2001120` |
| Mechanical Equipment | `-2001140` |
| Electrical Equipment | `-2001040` |
| Generic Model | `-2000151` |

---

*Nonica Revit MCP — Compatible with any Revit model*
