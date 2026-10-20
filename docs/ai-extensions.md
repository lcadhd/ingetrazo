# Extensions over MCP — using IngeTrazo plugins from an AI agent

Companion to [ai-bridge.md](ai-bridge.md). The bridge has five fixed tools (`run_python`,
`query_model`, `screenshot`, `undo`, `redo`); plugins cannot add tools. But `run_python` runs
inside the live app, where every plugin is already imported, so an agent can call a plugin's
own functions directly. This page says how, and documents each installed extension.

Written against IngeTrazo 0.5.9 and the extensions catalogue of 2026-10-20
(<https://ingetrazo.com/extensiones>, <https://github.com/ingelibre/ingetrazo-extensions>).
The plugin API is not stable; re-check a plugin's entry points after updating it.

## 1. Where plugins live

- Bundled: `<app>/plugins/` (AI Assistant, AI Bridge, Python Console, Model Info, Solid
  Inspector, Render with Blender). Example extensions (Levels, Polygon properties, Windowizer)
  ship in `<app>/examples/extensions/` and, since 0.5.8, are installed from
  **Extensions ▸ Manage extensions…** like any catalogue extension.
- Per-user: `$XDG_DATA_HOME/ingetrazo/plugins/` (Linux, default `~/.local/share/...`),
  `%APPDATA%\ingetrazo\plugins\` (Windows). **Extensions ▸ Open plugins folder** opens it.
  A plugin is one `.py` file or one package folder. **Extensions ▸ Manage extensions…**
  (0.5.8+) installs, updates and removes catalogue extensions after checking their SHA-256.
  By hand: install = copy + restart; uninstall = delete + restart; verify the SHA-256 first.
  The stem is whatever folder or file name the extension ships with (case and hyphens
  included: `planArq-v1`, `igz-uv`, `OpenTrace_BIM`), and it can change between versions.
- A plugin that fails to import or whose `setup(app)` raises shows as a disabled
  `⚠ name (load error)` entry in the Extensions menu (tooltip = the exception); the app still
  starts. Check from a script:

```python
import sys
from PySide6.QtWidgets import QMenu
print(sorted(n[17:] for n in sys.modules if n.startswith('ingetrazo_plugin_') and '.' not in n))
print([a.text() for m in viewport.window().menuBar().findChildren(QMenu)
       for a in m.actions() if 'load error' in a.text()])
```

## 2. Reaching a plugin from `run_python`

- Plugins are imported **by file path** as `ingetrazo_plugin_<stem>` (stem = file name without
  `.py`, or the package folder name). Use `sys.modules['ingetrazo_plugin_<stem>']`; never
  `import <stem>`. A package's submodules are `ingetrazo_plugin_<stem>.<sub>`.
  Exceptions (they register their own names): 3D Modifiers → `sys.modules['igz_mirror3d']`,
  `['igz_rotate3d']` …; Styles/Shadows toolbars → `igz_tb_style_igz_tb_style`,
  `igz_tb_style_igz_tb_shadows`; Scenes → `igz_scenes_igz_tb_scenes`.
- `Tool` plugins are instantiated once: `viewport.window()._tools['plugin_<stem>_<ClassName>']`.
- `setup(app)` plugins get an `ExtensionApp`. Build an equivalent to read or write their
  document data: `from views.extension_api import ExtensionApp; app = ExtensionApp(viewport.window(), '<stem>')`.
  State a plugin keeps in closures or widgets is only reachable through module globals,
  window attributes or the dock widget (`viewport.window().findChildren(PanelClass)`); each
  section below says where.
- Geometry types: vertex positions and `g.xform.map(p)` results are `QVector3D` — use
  `.x() .y() .z()`, not indexing. A group's `xform` is `None` when it has no transform
  (identity), so guard: `g.xform.map(p) if g.xform is not None else p`.
- **Prefer the pure builders.** Most geometry plugins have a function that returns a finished
  `Group` (or a new `Mesh`) without touching the document: call it and `groups.append(g)`
  yourself. That is one clean undo step and nothing else runs.

## 3. Rules that keep the bridge safe

**Never open a modal dialog.** `QDialog.exec()`, `QMessageBox.*`, `QFileDialog.*`,
`QInputDialog.*`, `QColorDialog.getColor` block the UI thread, so the bridge stalls until a
human closes the window. Most plugins' menu actions, "Edit…" functions and error paths use
them; the per-plugin Pitfalls lines name the ones to avoid. When calling plugin code you have
not read, turn dialogs into exceptions for the duration of the call:

```python
from PySide6.QtWidgets import QDialog, QMessageBox, QFileDialog, QInputDialog
def _no(*a, **k): raise RuntimeError('plugin tried to open a modal dialog')
_saved = [(c, n, c.__dict__.get(n, getattr(c, n))) for c, ns in (
    (QDialog, ('exec',)), (QMessageBox, ('exec', 'information', 'warning', 'critical', 'question')),
    (QFileDialog, ('getOpenFileName', 'getSaveFileName', 'getExistingDirectory')),
    (QInputDialog, ('exec', 'getText', 'getInt', 'getDouble', 'getItem'))) for n in ns]
for c, n, _ in _saved: setattr(c, n, _no if n == 'exec' else staticmethod(_no))
try:
    ...  # the plugin call
finally:
    for c, n, f in reversed(_saved): setattr(c, n, f)
```

**Answer startup dialogs first.** A modal dialog left open in the app (for example "Recovered
drawing" after a restart with unsaved work) blocks the UI thread, so every bridge request queues
up unanswered until a human closes it. OpenTrace BIM's "EXPERIMENTAL · Initial project setup"
window, shown on a new document, is not modal and does not block the bridge.

**Know what one `run_python` step can undo.** The step is a `SnapshotImport`: it restores the
active mesh (`scene.mesh`) and removes groups, layers, saved views, dimensions and text labels
**added** during the step. It does **not** restore:

- in-place edits to groups that already existed (their mesh, transform, `hidden`, children,
  `group.ext`), or groups removed from `model.groups`;
- `scene.plugin_data` (all extension document data), `scene.materials`, `scene.shadows`,
  display style, units, or any widget/panel state.

So: create new geometry in the step itself; change existing objects or plugin data only
through the plugin's (or the app's) own `Command` via `viewport.history.execute(cmd)`.

**Nested commands.** Plugin code that calls `viewport.history.execute(cmd)` inside
`run_python` pushes its own undo entry inside the outer step: that is two `undo` calls to take
it back (one if the outer step changed nothing it tracks, since an unchanged outer step leaves
no entry). If the inner command **fails it is rolled back silently**: the bridge still
answers `"error": null` (the traceback only appears in `stderr`). After any such call, check
and re-raise:

```python
viewport.history.execute(cmd)
if viewport.history.last_error:
    raise RuntimeError(viewport.history.last_error)
```

**Global side effects worth knowing.**

- ArchXQ IT Lite replaces `History.execute` for the whole app (after every command it checks
  whether ArchXQ groups were moved).
- Organize Toolbar replaces `MainWindow._on_unhide_last` with its own version.
- OpenTrace BIM has its own self-updater, which downloads releases from its author's GitHub, outside
  the catalogue review. It is off unless enabled in its Home panel. It can also download the
  Uniclass classification tables on request.
- Texture Library downloads from Poly Haven only when asked. Scene Automation runs `ffmpeg` /
  `ffplay` (if installed) for MP4 export and sounds. Display Scale can restart the app.
- Coloured Tools + Space moves the **Space** shortcut from Select to command search.
- Automation appends to `automacao-debug.log` in its own plugin folder while it runs (an entry per
  window that opens), so that file keeps growing.
- ComfyUI Renderer probes its server URL (3 s timeout) at startup.
- Point Cloud & Splatting saves invisible proxy box groups in the `.igz`; the clouds themselves are panel state and are not saved.

## 4. Built-in plugins

| Plugin | From MCP |
|---|---|
| AI Bridge | It *is* the MCP link. |
| Python Console | Same executor as `run_python`; nothing extra to call. |
| AI Assistant | In-app chat; not for an external agent. |
| Model Info | Read-only dialog. Use `query_model` / `core.bim.collect_objects(scene)` instead. |
| Solid Inspector | Dialog. Its per-edge diagnosis (0 faces = stray, 1 = open border, >2 = non-manifold) is easy to replicate on `mesh.edges`. |
| Render with Blender | Needs Blender installed in the app's environment (`core/render_blender.py` holds the logic). |

## 5. Index of installed extensions

"Full" = everything useful is callable without the UI; "Partial" = the core works, some
features need the UI; "UI only" = use the core APIs named in its section instead.
"Live-tested" = a test exercising its entry points was run through the bridge in
IngeTrazo 0.5.9 (2026-10-20), passed, and left the document unchanged afterwards.

| Extension | Stem | From MCP | Catalogue status | Live-tested |
|---|---|---|---|---|
| Array | `array_tool` | Full | Community | yes |
| IngeTrazo Bevel | `ingetrazo_bevel` | Full | Reviewed | yes |
| Taper Tool for IngeTrazo | `ingetrazo_taper` | Full | Reviewed | yes |
| Loop Cut | `loop_cut` | Partial | Community | yes |
| Vertex Editor | `vertex_editor` | Partial | Reviewed | yes |
| IngeTrazo Curve Tools | `ingetrazo_curve_tools` | Full | Community | yes |
| 3D Modifiers | `igz_modify3d` | Partial | Community | yes |
| Weld | `weld_tool` | Full | Community | yes |
| Path Select | `path_select_tool` | Full | Community | yes |
| Quadrify | `quadrify_tool` | Full | Community | yes |
| Membrane | `membrane_tool` | Full | Community | yes |
| JA LineaStyle | `lineas_color` | Full | Community | yes |
| Angle Dimension | `Angle_Dimension` | Full | Community | yes |
| Parametric Node Editor | `node_editor` | Partial | Community | yes |
| Parametric solids | `igz_solids` | Full | Community | yes |
| SubD (subdivision modelling) | `subdiv` | Full | Community | yes |
| Divide Points | `divide_points` | Full | Community | yes |
| IngeTrazo Unfold | `unfold` | Full | Community | yes |
| UV (UV mapping tools) | `igz-uv` | Full | Community | yes |
| Proxies | `proxy_tool` | Full | Community | yes |
| OpenTrace BIM | `OpenTrace_BIM` | Full | Community | yes |
| ArchXQ IT Lite · Architecture by phases | `archxq_it_lite` | Partial | Community | yes |
| IFC data — read and edit | `ifc_editor` | Full | Community | yes |
| Walls, doors and windows | `openings` | Full | Community | yes |
| Walls (masonry from floor plans) | `walls_tool` | Full | Community | yes |
| Plan Import (PDF floor plans) | `plan_import_tool` | Partial | Community | yes |
| CAD2IngeTrazo · CAD plans to BIM | `cad2ingetrazo` | Partial | Community | yes |
| planArq | `planArq-v1` | Full | Community | yes |
| AISC steel members | `igz_steel` | Full | Community | yes |
| Windowizer | `windowizer` | Full | Bundled | yes |
| IngeTrazo Stair Maker with Railings | `make_stair` | Full | Reviewed | yes |
| IngeTrazo Kitchen Maker | `ingetrazo_kitchen_maker` | Full | Reviewed | yes |
| IngeFloor · Wood floors | `floor_generator` | Partial | Community | yes |
| Roof Tiles | `roof_tile_tool` | Full | Community | yes |
| Levels (Niveles) | `niveles` | Full | Reviewed | yes |
| Polygon properties | `propiedades_poligono` | Full | Reviewed | yes |
| Topology (terrain) | `topology_tool` | Full | Community | yes |
| Grading | `grading_tool` | Full | Community | yes |
| Cut List | `cutlist` | Full | not catalogued | yes |
| Woodworking Cut List (Lista de corte para marcenaria) | `lista_corte_marcenaria` | Full | Community | yes |
| STEP export | `stepexport` | Full | Community | yes |
| 3D PDF export | `igz_pdf3d` | Full | Community | yes |
| Point Cloud & 3D Gaussian Splatting | `pointcloud_splat` | Partial | Community | yes |
| JA Sun · Sun and shadows | `ja_sun` | Full | Community | yes |
| igz_sunpath · Sun Path | `igz_sunpath` | Partial | Community | yes |
| Styles, Shadows and Map toolbars | `igz_tb_toolbar` | Partial | Community | yes |
| Scenes (scene manager) | `igz_tb_scenes` | Partial | Community | yes |
| Scene Automation | `scenes_play` | Full | Community | yes |
| Automation (Automação) | `automacao` | Partial | Community | yes |
| Phasing Animator for IngeTrazo | `phasing_animator` | Partial | Community | yes |
| ComfyUI Viewport Renderer | `comfyui_renderer` | Partial | Community | yes |
| Perspective Matcher | `perspective_matcher` | Partial | Community | yes |
| Texture Library for IngeTrazo | `texture_library` | Full | Community | yes |
| MultiView for IngeTrazo | `ingetrazo_multiview` | UI only | Community | yes |
| Standard Views 2 | `standard_views_2` | Partial | Community | yes |
| JA Tree · Model outliner | `outliner` | Full | Community | yes |
| JA Units · Units | `ja_units` | Full | Community | yes |
| Organize Toolbar | `organize_toolbar` | UI only | Community | yes |
| Isolate / Unisolate Selection | `isolate_unisolate_selection_toolbar` | Full | not catalogued | yes |
| Coloured Tools + Space (UI Colores + Space) | `ui_colores_space` | UI only | Community | yes |
| Custom Icons · SVG icon themes | `custom_icons` | Partial | Community | yes |
| Display Scale for IngeTrazo | `display_scale` | UI only | Community | yes |

## 6. Modelling and editing

### Array — `array_tool`
1.2.1 · Pesi (pesi3d.de) · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Copies the selection (groups, components, loose faces/edges) in a 1D/2D/3D grid with per-copy move, rotate and scale. **In the app:** Extensions ▸ Array…, right-click ▸ Array…, and a one-button PESI3D toolbar (all open a non-modal dialog).

**From MCP:** Full — the whole array is a pure parameter dict + a `Command` that works on the current selection, no UI needed (1.2.1 only adds the version to the dialog title).

- Module: `ingetrazo_plugin_array_tool`
- Entry points:
  - `default_params()` → dict: `move`/`rotate`/`scale` rows `{"mode": "inc"|"tot", "inc": [x,y,z], "tot": [x,y,z]}` (move in metres, rotate in degrees, scale in %, 100 = same size); `reorient` bool; `uniform` bool (scale X for all axes); `center` `"selection"|"pivot"|"world"`; `type` `"copy"|"instance"`; `dim` 1|2|3; `count` `[n1, n2, n3]` (totals per dimension, original included); `row2`/`row3` `[x,y,z]` metres offset between rows/layers.
  - `total_in_array(p)` → int (original included).
  - `make_array_command(scene, p)` → a `Command` for the CURRENT `scene.selection` (or `None` if nothing selected / total ≤ 1). `cmd.do(scene)` builds the copies; `cmd.copies` = number of grid cells made (each holds every selected object).
  - `run_array(viewport, p)` → executes via `viewport.history.execute` and restores the selection to the originals.
  - `copy_matrix(i, j, k, p, center, ref)` → `QMatrix4x4` of one copy (pure).
- Saved data: QSettings `plugins/array_tool/params` (JSON of the last dialog run; `load_params()` / `save_params(p)`); `plugins/pesi3d/placed_toolbars`. Nothing in the document.
- Needs: nothing extra.
- Pitfalls:
  - Select first: groups/faces/edges in `scene.selection`; edges bounding a selected face are ignored (copied with the face).
  - Prefer `make_array_command(scene, p).do(scene)` inside `run_python` (one undo step; copies are new groups / loose-mesh faces, both reverted by the outer transaction). `run_array` pushes a second undo entry.
  - `type="instance"` first converts classic groups to components in place (`GroupToComponentCommand`); the outer `run_python` undo does NOT revert that — use `run_array` + check `viewport.history.last_error` if you need instances.
  - Do not run it while a group is open for editing: copies go into that group's children, which the outer undo does not track.
  - `cmd.do` leaves the selection on the last inserted copy. Max 20000 objects (`MAX_OBJECTS`) is enforced only by the dialog.

```python
import sys
m = sys.modules['ingetrazo_plugin_array_tool']
g = extrude([(10, 0), (10.5, 0), (10.5, 0.5), (10, 0.5)], 0, 1, name="Post")
selection.clear(); selection.add(g)
p = m.default_params()
p["move"]["inc"] = [1.0, 0.0, 0.0]          # 1 m between copies
p["dim"] = 2; p["count"] = [5, 3, 1]; p["row2"] = [0.0, 2.0, 0.0]
cmd = m.make_array_command(scene, p); cmd.do(scene)
print(cmd.copies)                              # 14 new posts
```

### IngeTrazo Bevel — `ingetrazo_bevel`
1.0.2 · aashishzharbade-arch · GPL-3.0-or-later · Reviewed

*Live-tested in IngeTrazo 0.5.9.*

Chamfers or rounds edges of closed, consistently oriented polygon solids: either the selected loose edges/faces or every sharp edge of one selected group. **In the app:** Extensions ▸ IngeTrazo Bevel V1.0 ▸ Bevel selected edges / solid, and the same button on its toolbar (opens a preview dialog).

**From MCP:** Full — `bevel_mesh()` is a pure function that returns a new mesh; only the dialog is UI.

- Module: `ingetrazo_plugin_ingetrazo_bevel`
- Entry points:
  - `bevel_mesh(source, chosen=None, width=.01, segments=1, angle=30., soften=True)` → `(new_mesh, n_beveled_edges)`. `source`: a `Mesh` made only of closed shells (every edge exactly 2 faces, planar faces without holes, outward winding, ≤ 6000 faces). `chosen`: set of `Edge`s of `source`, or `None` = all edges whose dihedral turn ≥ `angle` (degrees). `width`: offset along each face in **metres** (≥ 0.001). `segments`: 1 = chamfer, 2–12 = rounded profile. Source is not modified. Raises `ValueError` with the reason.
  - `ReplaceMesh(target_mesh, result, group=None)` — the Apply command: with `group` it sets `group.mesh = result`; without it replaces the whole active mesh's content.
  - `target_from(app)` → `(mesh, chosen_edges|None, group|None)` from the current selection (UI helper; raises `ValueError`).
- Saved data: none (`window._ingetrazo_bevel = (toolbar, dialogs)`).
- Needs: NumPy (bundled with the app).
- Pitfalls: the loose path bevels the WHOLE loose mesh (all of it must be closed solids) and replaces it. Groups with children or a non-unit scaled `xform` are refused by the UI; `width` is in the group's local units. To apply to an existing group use `viewport.history.execute(m.ReplaceMesh(g.mesh, result, g))` then check `viewport.history.last_error` — a direct `g.mesh = result` is NOT undone by the run_python transaction unless `g` was created in that same call. Rounded corners where three rounded edges meet become spherical patches; other junctions get flat fans. Big widths fail with "offset collides near a corner".

```python
import sys; m = sys.modules['ingetrazo_plugin_ingetrazo_bevel']
g = extrude([(12, 2), (12.5, 2), (12.5, 2.5), (12, 2.5)], 0, 0.5, name="cube")
vertical = {e for e in g.mesh.edges if abs(e.a.z() - e.b.z()) > 1e-9}
new, n = m.bevel_mesh(g.mesh, vertical, width=0.05, segments=4)
g.mesh = new; print(n, len(new.faces))
```

### Taper Tool for IngeTrazo — `ingetrazo_taper`
1.0.0 · aashishzharbade-arch · GPL-3.0-or-later · Reviewed

*Live-tested in IngeTrazo 0.5.9.*

Tapers selected groups or loose faces/edges along a 3D axis: the radial distance from the axis is scaled linearly from 1 at the axis start to target/reference at its end (optionally only in one direction, or as a rigid move). **In the app:** Extensions ▸ Taper Tool for IngeTrazo V1.0 and a toolbar button; then four clicks in the viewport (axis start, axis end, reference, target) with a small options window.

**From MCP:** Full — the mapping and deformation are pure; only the click-driven tool is UI.

- Module: `ingetrazo_plugin_ingetrazo_taper`
- Entry points (all metres):
  - `Mapping(a, b, direction, reference, target, flat=False, infinite=True)` — `a`, `b`: axis start/end `QVector3D`; `direction`: vector towards the reference side (its part perpendicular to the axis is used); `reference`/`target`: distances from the axis, scale at `b` = target/reference (≤ 1000); `flat`: scale only along `direction`; `infinite=False` clamps the taper to the a–b span. Callable: `mp(point)` → new `QVector3D`. Raises `ValueError`.
  - `deform_mesh(source_mesh, mapping, rigid=False, divisions=4)` → new `Mesh` (world coords): all faces triangulated and cut at `divisions` stations along the axis (smooth, soft edges), original edges kept as hard edges. `rigid=True` just translates the mesh by the displacement of its bbox centre.
  - `Selection(scene)` (from `scene.selection`: groups and/or COMPLETE loose objects) → `.build(mapping, rigid, divisions)` → list of new classic `Group`s named "<name> · Taper Tool" (layer, material, ifc copied).
  - `TaperCommand(selection, result_groups, copy_result)` — replaces (or with `copy_result=True` keeps) the originals.
- Saved data: none (`window._ingetrazo_taper_tool = (toolbar, state, launch)`).
- Needs: nothing extra.
- Pitfalls: `launch()` shows a QMessageBox if the selection is invalid — do not call it. `TaperCommand` removes original groups/faces: run it with `viewport.history.execute(...)` (nested, own undo step) and check `viewport.history.last_error`; it REASSIGNS `scene.selection`, so the `selection` name in scope is stale for the rest of that call. Removing groups yourself in run_python is not undoable (the transaction only restores the loose mesh and drops added groups). Result face counts grow fast (triangles × stations, cap 200 000).

```python
import sys; m = sys.modules['ingetrazo_plugin_ingetrazo_taper']
g = extrude([(12, 2), (13, 2), (13, 3), (12, 3)], 0.0, 2.0, name="box")
mp = m.Mapping(QVector3D(12.5, 2.5, 0), QVector3D(12.5, 2.5, 2), QVector3D(1, 0, 0), 0.5, 0.25)
groups.append(Group(m.deform_mesh(g.mesh, mp, False, 4), name="box · tapered"))
groups.remove(g)   # fine here: g was created in this same call
```

### Loop Cut — `loop_cut`
1.1.0 · Daniel Wagner · MIT · Community

*Live-tested in IngeTrazo 0.5.9.*

Inserts one edge loop through a closed strip of quads (midpoints of the crossed edges) or, failing that, through a closed planar cross-section perpendicular to the picked edge; 1.1 adds face picking (picks the longest quad ring through the face — cylinder walls), Alt+click to select the green Edge Loop and Shift+click to add the blue Edge Ring. **In the app:** the «LoopTools» toolbar button «Corte em loop» (an interactive tool: hover for a pink preview, click to cut). UI texts are Portuguese.

**From MCP:** Partial — no public cut function: the `_plan*` functions are pure, the cut itself (10 lines in `LoopCutTool.on_click`) must be replicated, as below.

- Module: `ingetrazo_plugin_loop_cut`
- Entry points (all read-only, raise `ValueError` with a Portuguese message when no clean ring exists):
  - `_plan(mesh, seed_edge)` → `(pairs, crossed_edges, points)`: `pairs` = `{face: (edge_in, edge_out)}`, `crossed_edges` = set, `points` = `{edge: QVector3D}` where each crossed edge is split. `seed_edge` is an edge the ring CROSSES (box side strip: a vertical edge → horizontal cut at mid-height). Tries `_loop_plan(mesh, seed)` (quads only, returns `(pairs, edges)`) then `_plane_plan(mesh, seed)` (plane through the seed's midpoint, normal along it).
  - `_quad_plan_from_face(mesh, face)` → same triple: the LONGEST closed quad loop through a quad face (what a click on a face uses).
  - `_ring_from_edge(mesh, edge)` → set of edges: the closed coplanar end-to-end ring containing `edge` (e.g. the contour made by a previous cut). Select it with `scene.select(list(ring), mode="add")`.
  - Edge Loop selection = `scene.select(list(crossed_edges), mode="replace")`.
  - `_tool` (module global) / `viewport.window()._tools['plugin_loop_cut_LoopCutTool']` — the tool; its `on_click` needs a screen pick, not useful from a script.
- Saved data: none.
- Needs: nothing extra (`core.topology.split_face_by_chord`).
- Pitfalls:
  - Works on whichever mesh you pass, but only `scene.mesh` is restored by `undo`; cut a group's mesh only after opening it, or on geometry created in the same call.
  - Faces with holes, open borders, a plane through an existing vertex, or branching sections are refused (`ValueError`); catch it or let the call roll back.
  - The cut replaces every face of the ring (new `Face` objects, attrs copied); hold no references to old faces.
  - `_ring_from_edge` scans every edge of the mesh (slow on very large meshes). No dialogs.

```python
import sys
from core.topology import split_face_by_chord
lc = sys.modules['ingetrazo_plugin_loop_cut']
def loop_cut(mesh, plan):                       # same as LoopCutTool.on_click's mutate
    pairs, _crossed, points = plan
    for e, p in points.items():
        mesh.split_edge(e, p)
    for f, (a, b) in pairs.items():
        loops = split_face_by_chord(f, points[a], points[b])
        if loops is None: raise RuntimeError("face would not split")
        attrs = dict(f.attrs); mesh.remove_face(f)
        for lp in loops: mesh.add_face(lp).attrs.update(attrs)
    return len(pairs)
M = scene.mesh                                  # a loose box at x 10..11, z 0..2 exists
seed = M.find_edge(M.vertex_at(QVector3D(10, 0, 0)), M.vertex_at(QVector3D(10, 0, 2)))
print(loop_cut(M, lc._plan(M, seed)))           # 4 faces cut at z = 1
```

### Vertex Editor — `vertex_editor`
1.1 · Daniel Wagner · MIT · Reviewed

*Live-tested in IngeTrazo 0.5.9.*

Moves one vertex of the active mesh (click it, then click the target or type a distance); landing on another vertex welds them, and the move is rolled back if it would add open, loose or non-manifold edges or a degenerate face. **In the app:** Extensions ▸ «Editar vertice (2 cliques)» and the «Vertex» toolbar button (an interactive tool; red dots on every vertex). UI texts are Portuguese.

**From MCP:** Partial — no public function; the move+weld+validate logic lives in `VertexEditorTool._commit`, which can be driven through the tool instance (below). A plain move without the checks is just `scene.mesh.place_vertex(v, target)`.

- Module: `ingetrazo_plugin_vertex_editor`
- Entry points:
  - `_editor` (module global; also `viewport.window()._tools['plugin_vertex_editor_VertexEditorTool']`) — the tool instance.
  - `_editor._commit(viewport, delta)` — moves the vertex at `_editor.start_point` by `delta` (`QVector3D`, metres) in `viewport.scene.mesh`; if a vertex already sits at the target it welds them (`weld_coincident` + `prune_orphan_vertices`) and refuses (rolls back) a weld that leaves a face with a repeated vertex or increases boundary/loose/non-manifold edge counts. Set `_editor.selected = v; _editor.start_point = QVector3D(v.position)` first. Returns nothing; the result is in `viewport.history.last_error` (None = applied). Resets the tool afterwards.
- Saved data: none.
- Needs: nothing extra.
- Pitfalls:
  - `_commit` calls `viewport.history.execute(...)` itself: a nested undo entry, and a refused weld is swallowed by the outer `run_python` (reports ok). Pass the shim below instead of `viewport` so it runs inside the one `run_python` step and `last_error` can be checked.
  - Only `scene.mesh` (loose geometry, or the group open for editing) is used; vertices are matched by position (`mesh.vertex_at`, 0.1 mm weld grid).
  - Error messages are Portuguese. No dialogs.

```python
import sys
ve = sys.modules['ingetrazo_plugin_vertex_editor']
class OneStep:                       # viewport stand-in: no nested undo entry
    def __init__(s, vp): s.vp, s.scene, s.history, s.last_error = vp, vp.scene, s, None
    def execute(s, cmd):
        snap = s.scene.mesh.capture_state(); s.last_error = None
        try: cmd.do(s.scene)
        except Exception as e: s.scene.mesh.restore_state(snap); s.last_error = str(e)
    def __getattr__(s, n): return getattr(s.vp, n)
vp = OneStep(viewport); t = ve._editor
v = scene.mesh.vertex_at(QVector3D(11, 1.2, 0))
t.selected, t.start_point = v, QVector3D(v.position)
t._commit(vp, QVector3D(0, -0.2, 0))            # onto (11, 1, 0): welds
if vp.last_error: raise RuntimeError(vp.last_error)
```

### IngeTrazo Curve Tools — `ingetrazo_curve_tools`
1.0.0 · aashishzharbade-arch · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Creates editable sampled curves (Bézier, cubic Bézier chains, B-spline, centripetal Catmull–Rom, polyline; open or closed) as wire-only groups whose recipe is stored on the group, and converts an existing edge chain into an editable polyline. **In the app:** Extensions ▸ IngeTrazo Curve Tools V1.0 and its toolbar (one button per curve type, Edit, Convert), right-click ▸ Edit Curve Tools curve; an editor window plus click/drag of control points in the viewport.

**From MCP:** Full — sampling and group building are pure functions; only the editor and the click tool are UI.

- Module: `ingetrazo_plugin_ingetrazo_curve_tools`
- Entry points (coordinates in **metres**, local to the curve group, whose `xform` starts as identity = world):
  - `recipe(kind='Bézier', points=[[x, y, z], …], segments=24, closed=False)` → dict. `kind` ∈ `KINDS` = 'Bézier', 'Cubic Bézier' (open: 4+3n controls, closed: 3+3n), 'B-spline', 'Catmull–Rom', 'Polyline'. `segments` must be an `int` 2–256 (total for Bézier/B-spline, per span for cubic Bézier/Catmull–Rom).
  - `sample(data)` → list of `[x, y, z]` along the curve (closed curves repeat the first point at the end); raises `ValueError`.
  - `curve_mesh(data)` → `Mesh` of edges sharing one `curve` id (selects as one curve). `make_group(data)` → `Group` "Curve — <kind>" (not yet in the document): `groups.append(g)`.
  - `ordered_path(edges)` → `(points, closed)` for one unbranched chain/loop (≤ 512 vertices); `conversion_data(scene)` → Polyline recipe from the selected wire group or loose edges.
  - `data_of(g)` → stored recipe or None; `validate_source(g)` raises if the group no longer matches its recipe.
  - Edit an existing curve: `EditCurve(g, new_data)` via `viewport.history.execute` (nested; replaces `g.mesh`).
- Saved data: `group.ext["ingetrazo_curve_tools"]` = the recipe dict (version, kind, points, segments, closed).
- Needs: NumPy (bundled with the app).
- Pitfalls: `launch(...)` / `Editor.apply` / `to_polyline` show QMessageBox on errors — avoid them. Curves are edges only (no faces); use them as Follow-Me paths or outlines. Points are de-duplicated at 0.1 mm. Replacing `g.mesh` directly is not undone by the run_python transaction unless `g` was created in the same call.

```python
import sys; m = sys.modules['ingetrazo_plugin_ingetrazo_curve_tools']
k = 0.5523
d = m.recipe('Cubic Bézier', [[13, 2, 0], [13, 2 + k, 0], [12 + k, 3, 0], [12, 3, 0]], segments=16)
g = m.make_group(d); groups.append(g)
print(len(m.sample(d)), len(g.mesh.edges))
```

### 3D Modifiers — `igz_modify3d`
1.0.1 · Ezequiel M Rezende · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Six point-driven tools: Mirror (plane by 3 points), Scale (ratio of two distances), Rotate (angle between two directions), Align (3 source → 3 target points), Array (step vector × N) and Polar Array (axis, N, total angle). **In the app:** the «3D Modifiers» toolbar (interactive tools: click points; no menu entries).

**From MCP:** Partial — the transform matrices are pure functions and the group helpers work from a script; the tools' own flows are click-driven and Mirror/Array/Polar open modal dialogs.

- Modules: `ingetrazo_plugin_igz_modify3d` (only `setup`); the tools are imported as TOP-LEVEL modules (the package dir is put on `sys.path` in setup): `sys.modules['igz_mirror3d']`, `['igz_scale3d']`, `['igz_rotate3d']`, `['igz_align3d']`, `['igz_array3d']`, `['igz_polararray3d']`, `['igz_xform_command']`; toolbar module `['igz_modify3d_main']`.
- Entry points (points are `QVector3D`, metres; each returns a `QMatrix4x4` or `None` if degenerate):
  - `igz_mirror3d._reflection_matrix(p1, p2, p3)` — reflection across the plane through 3 points.
  - `igz_scale3d._scale_matrix(p1, p2, p3)` — uniform scale about `p1` by `|p3-p1| / |p2-p1|`.
  - `igz_rotate3d._rotation_matrix(p1, p2, p3)` — rotation about `p1` taking direction `p2-p1` onto `p3-p1` (axis = their cross product).
  - `igz_align3d._align_matrix_cad(s1, t1, s2, t2, s3, t3)` — rigid move: `s1→t1`, `s2-s1` along `t2-t1`, plane `s1 s2 s3` onto `t1 t2 t3`.
  - `igz_mirror3d._copy_group(g, m)` → new transformed `Group` named "<name> (Espelho)" (NOT added; `groups.append` it). `igz_mirror3d._copy_loose(scene, items, m)` adds mirrored copies of `Face`/`Edge` items to `scene.mesh` (winding reversed).
  - `igz_xform_command.XformGroupsCommand(transform, groups, remove_after=False)` — undoable in-place `transform(g)` per group (snapshots xform/mesh/axes; optional removal from `scene.groups`).
  - Array / Polar: no functions; the copy loop is a closure in `_finish` (copies = `copy.deepcopy(group)` + translation / rotation about the axis by `total_deg / count * i`, i = 1..count-1).
- Saved data: none.
- Needs: nothing extra.
- Pitfalls:
  - NEVER activate the tools or call `Mirror3dTool._finish` (QMessageBox "delete originals?"), `Array3dTool.on_click` / `PolarArray3dTool.on_click` (QInputDialog): they stall the bridge.
  - Scale/Rotate/Align/Array/Polar act on selected GROUPS only; loose geometry is ignored (only Mirror copies loose faces/edges).
  - In-place transforms of EXISTING groups are not undone by the `run_python` step: run `viewport.history.execute(XformGroupsCommand(...))` (its own undo entry; the bridge then reports `changed: false`) and check `history.last_error`. For groups created in the same call, `cmd.do(scene)` is fine.
  - A mirrored classic group keeps its vertex order, so its faces come out inside-out (confirmed live: the outer face normal points inward) — reverse them afterwards; Array/Polar copies keep the original's `uid` (deepcopy).
  - The modules log to stderr (`DEBUG = True`): expect `[igz_…]` lines in `stderr`.

```python
import sys
from core.group import transformed_mesh
rot, xc = sys.modules['igz_rotate3d'], sys.modules['igz_xform_command']
g = extrude([(10, 0), (11, 0), (11, 1), (10, 1)], 0, 1, name="Box")
m = rot._rotation_matrix(QVector3D(10, 0, 0), QVector3D(11, 0, 0), QVector3D(10, 1, 0))  # 90° about Z
def apply(gr):                                   # same as the tools' apply_transform
    if gr.xform is not None: gr.xform = m * gr.xform
    else: gr.mesh = transformed_mesh(gr.mesh, m)
xc.XformGroupsCommand(apply, [g]).do(scene)      # new group: undone with the call
```

### Weld — `weld_tool`
1.2 · Pesi (pesi3d.de) · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Joins chains of loose edges into one curve (selects with one click), optionally closing small gaps between free ends and dropping collinear points; Unweld clears the curve ids. **In the app:** Extensions ▸ Weld ▸ Weld Edges… / Unweld Edges, right-click on selected edges, and a PESI3D toolbar (Weld opens a non-modal dialog; 1.2 adds a "Live Preview" button, "Follow selection", "↻ Read selection" and Reset).

**From MCP:** Full — `weld()` / `unweld()` are pure functions on a `Mesh`, no UI (unchanged since 1.1; only the dialog changed).

- Module: `ingetrazo_plugin_weld_tool`
- Entry points:
  - `weld(mesh, edges, tol=0.0, collinear=False)` → `{"curves", "gaps", "points", "edges"}`: mutates `mesh` in place. `tol` (metres) closes gaps between open chain ends closer than it (two free ends meet in the middle; a free end next to attached geometry moves onto it; faces never move); `collinear=True` removes interior straight-run vertices not used by faces (within 0.1°); every chain of ≥ 2 edges gets one new `edge.curve` id. Returns counts and the resulting edge list.
  - `unweld(mesh, edges)` → int: clears `edge.curve` on `edges`.
  - `selected_edges(scene)` → selected `Edge`s that live in `scene.mesh`.
  - `build_chains(edges)` → `[(edge_list, vertex_list, closed)]`; `find_gaps(edges, tol)` → `(pairs, blocked)` (read-only).
  - `run_weld(viewport, edges, tol, collinear)` / `run_unweld(viewport)` — UI commands (own `history.execute`, status-bar messages).
- Saved data: QSettings `plugins/weld_tool/gaps`, `tol_m` (default 0.01), `collinear`, `preview`, `follow`; nothing in the document.
- Needs: nothing extra.
- Pitfalls:
  - Call `weld(scene.mesh, …)` directly inside `run_python` (one undo step). `run_weld` pushes its own nested undo entry.
  - Do not call `open_dialog` from a script: with Live Preview on (the remembered default) its timers execute and undo history commands on their own while it stays open.
  - Only `scene.mesh` is snapshotted by the transaction; welding a closed group's `g.mesh` is not undone. Open the group first or build new geometry.
  - Chains break at junctions (3+ edges) and where unselected geometry is attached; single-edge chains get no curve id.
  - Edge objects change when gaps or collinear points are removed: use the returned `"edges"`.

```python
import sys
w = sys.modules['ingetrazo_plugin_weld_tool']
pts = [(10, 5), (11, 5), (12, 5), (12, 6)]
es = [scene.mesh.add_edge(QVector3D(*a, 0), QVector3D(*b, 0)) for a, b in zip(pts, pts[1:])]
es.append(scene.mesh.add_edge(QVector3D(12, 6.004, 0), QVector3D(12, 7, 0)))
r = w.weld(scene.mesh, es, tol=0.01, collinear=True)
print(r["curves"], r["gaps"], r["points"], len(r["edges"]))   # 1 1 2 2
```

### Path Select — `path_select_tool`
1.1 · Pesi (pesi3d.de) · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Selects a path of edges by clicking a few points along it: the shortest way along the edges between clicks is filled in; Select Contour grows selected edges along their chains. **In the app:** Extensions ▸ Path Select (Alt+C, an interactive tool), right-click on edges ▸ Select Contour, and a PESI3D toolbar.

**From MCP:** Full — the graph search and contour walk are pure functions; the result is just a selection.

- Module: `ingetrazo_plugin_path_select_tool`
- Entry points:
  - `shortest_route(sources, targets, blocked_edges=(), blocked_verts=())` → `(source_vertex, target_vertex, [edges])` or `None`: A* along edges (metres) from any `Vertex` in `sources` to the first `Vertex` of `targets` reached; hidden edges are never walked.
  - `contour_of(edge)` → `[edges]` in order: the chain through `edge`, walking both ways while exactly two visible edges meet (stops at junctions and loose ends).
  - `PathState()` — the click-by-click path: `.start(edge)`, `.route_to(edge)` → preview `(end_index, edges_to_add, new_end_vertex, closes)` or `None`, `.commit(preview)`, `.undo()`; result in `.edges` (ordered), `.ends`, `.closed`.
  - Select the result with `scene.select(edges)` (`mode="replace"|"add"|"toggle"|"remove"`).
- Saved data: none (only `plugins/pesi3d/placed_toolbars` in QSettings for the toolbar).
- Needs: nothing extra.
- Pitfalls:
  - Nothing here mutates geometry: a `run_python` call that only changes the selection makes no undo entry (`changed: false`), and `undo` does not restore the previous selection.
  - Works on `Vertex`/`Edge` objects of one mesh — get vertices with `scene.mesh.vertex_at(QVector3D(...))`; a group's edges live in `g.mesh`.
  - The interactive tool instance lives in a `setup()` closure (not in `window._tools`); there is nothing to gain from driving it.

```python
import sys
ps = sys.modules['ingetrazo_plugin_path_select_tool']
M = scene.mesh
a = M.vertex_at(QVector3D(10, 10, 0)); b = M.vertex_at(QVector3D(12, 12, 0))
src, dst, route = ps.shortest_route([a], [b])
scene.select(route)                       # the path is now the selection
print(len(route), sum(e.length() for e in route))
```

### Quadrify — `quadrify_tool`
1.2 · Pesi (pesi3d.de) · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Pairs triangles into four-sided faces: flat pairs become one real quad face, curved pairs keep their triangles with a soft (or hidden) diagonal; Grid Remesh rebuilds a height-field surface (terrain) as a regular grid of quads clipped to its outline. **In the app:** Extensions ▸ Quadrify ▸ Quadrify… / Grid Remesh…, its own toolbar, and the right-click menu on selected faces.

**From MCP:** Full — `plan`/`apply_plan` and `grid_plan`/`apply_grid` are pure and work on any `Mesh` (unchanged since 1.1; 1.2 only rebuilt the dialogs).

- Module: `ingetrazo_plugin_quadrify_tool` (single file)
- Entry points:
  - `plan(mesh, faces, p)` → `{"pairs": [(edge, f0, f1, quad, kind, flipped)], "tris": n, "left": [faces]}`; `faces` = faces of `mesh` to consider (only holeless triangles count). `p = dict(DEFAULTS)`: `face_angle` (° max fold), `shape_angle` (° max corner deviation from 90), `planar_tol` (°, ≤ 2.5: flatter pairs merge), `curved` "soft"/"hidden"/"keep", `materials` (pair only same paint), `maximize`.
  - `apply_plan(mesh, pairs)` → `{"merged", "softened", "faces": new quad faces}` — edits `mesh` in place.
  - `grid_plan(faces, gp)` → `{"faces", "full", "border", "overlap", "spacing", "error"}` (`error` None / "empty" / "vertical" / "too_many"); `gp = dict(GRID_DEFAULTS)`: `spacing_m` (m, 0 = auto), `angle` (° about Z; `auto_angle(faces)` fits the site), `clip`, `planar_tol`.
  - `apply_grid(mesh, faces, plan)` → `{"faces", "soft"}` — replaces `faces` in `mesh` by the grid (third argument is the PLAN from `grid_plan`).
  - `scope_faces(scene)` → `(selected faces of scene.mesh or all its faces, from_selection)`.
- Saved data: none in the document. QSettings `plugins/quadrify_tool/<key>` and `…/grid_<key>` (dialog defaults).
- Needs: manifold3d for Grid Remesh (ships with IngeTrazo).
- Pitfalls: `open_dialog`/`open_grid_dialog` are non-modal (no message boxes) but must not be opened from a script: since 1.2 their Live Preview (on by default, remembered in QSettings) executes and undoes `SnapshotImport` steps by itself on timers while open. `run_quadrify`/`run_grid` wrap `apply_*` in their own `SnapshotImport` (nested undo step) and never report failure — call `apply_*` directly. **Undo:** run_python's snapshot covers `scene.mesh` (loose geometry, or the group currently open for editing) and added groups only; editing the mesh of an existing, closed group is NOT undone — work on `mesh`, or ask the user to open the group first. Grid Remesh needs a height field (`plan["overlap"]` True = steep/overhanging parts).

```python
import sys; m = sys.modules['ingetrazo_plugin_quadrify_tool']
faces = [f for f in mesh.faces if len(f.loop) == 3]        # loose triangles (undoable)
pl = m.plan(mesh, faces, dict(m.DEFAULTS))
r = m.apply_plan(mesh, pl["pairs"])
print(r["merged"], "quads,", r["softened"], "soft diagonals,", len(pl["left"]), "left")
gp = dict(m.GRID_DEFAULTS, spacing_m=0.5)                   # Grid Remesh of a terrain
terr = [f for f in mesh.faces if f.normal().z() > 0.2]
plan = m.grid_plan(terr, gp)
if plan["error"] is None: m.apply_grid(mesh, terr, plan)
```

### Membrane — `membrane_tool`
1.1 · Pesi (pesi3d.de) · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Stretches a minimal-surface (soap-film) skin over a closed loop of edges that may be bent in 3D — tents, sails, hypars — optionally inflated or sucked in by pressure, with inner loops as mast rings/holes, open chains as cables and guide points as fixed points. **In the app:** Extensions ▸ Membrane ▸ Skin from Edges… / Edit Membrane…, its own toolbar, and the right-click menu.

**From MCP:** Full — `make_membrane` is pure and returns a ready `Group`; append it yourself.

- Module: `ingetrazo_plugin_membrane_tool` (single file)
- Entry points:
  - `make_membrane(data, p, name="Membrane")` → `(Group, info)`. `data = {"segs": [(x0,y0,z0,x1,y1,z1), …], "points": [(x,y,z), …]}` (m); the largest closed loop (in its best-fit plane) is the boundary. `p` from `default_params()`: `res_mode` "div"/"size", `div` (4–250), `size` (m), `iters` (relaxation rounds, 1–500), `pressure` (−100…100 %, 100 = hemisphere over a circle; + bulges along the plane normal, which points up for a level loop), `ratio` (warp:weft tension, 0.1–10), `inner` "holes"/"lines", `soft`, `flip`. `info`: triangles, seconds, move (last round's motion / size; < 0.002 = settled), inner, lines, points, ignored.
  - `gather(obj)` → `data` from `obj.selection` (edges, groups of edges, guide points).
  - `compute(data, p)` → `(V, T, info)` numpy arrays; `classify(data, normalize_params(p))` + `estimate_triangles(dom)` for a cheap count.
- Saved data: `group.ext["membrane_tool"]` (version, params, segs, points). QSettings `plugins/membrane_tool/params` (dialog defaults).
- Needs: numpy (ships with IngeTrazo).
- Pitfalls: `show_skin`/`show_edit` open a non-modal dialog that uses `QMessageBox` on errors — avoid them. `run_membrane(viewport, data, p)` executes its own command (nested undo step) but raises on `history.last_error`. Raises `MembraneError` without a closed loop, when the skin does not settle (pressure too high), or above 150 000 triangles (`force=True`). The result is a single open surface (one side), no thickness. Check `info["move"]`: high = raise `iters`.

```python
import sys; m = sys.modules['ingetrazo_plugin_membrane_tool']
loop = lambda P: [P[i] + P[(i+1) % len(P)] for i in range(len(P))]
hypar = [(10, 0, 0.0), (14, 0, 1.0), (14, 4, 0.0), (10, 4, 1.0)]
p = m.default_params(); p.update(div=20, iters=30, pressure=0.0)
g, info = m.make_membrane({"segs": loop(hypar), "points": []}, p, name="Membrane")
groups.append(g); print(info["triangles"], round(info["move"], 4))
```

### JA LineaStyle — `lineas_color`
0.1.4 · Julio Angulo · GPL-3.0-or-later · Community (catalogue id `ja_lineastyle`)

*Live-tested in IngeTrazo 0.5.9.*

Draws 3D lines with a display colour, pixel weight (1–12 px) and pattern (solid, dotted, dashed, long_dash, dash_dot, dash_dot_dot) in one "JA LineaStyle" group mesh; closed coplanar outlines become faces. Also restyles selected line groups/loose wires. Styles are drawn by a patched viewport pass (also in `render_image`/screenshots). **In the app:** side-tray tab «LineaStyle», Extensions ▸ «JA LineaStyle…», right-click ▸ JA LineaStyle ▸ colour / Restaurar estilo, and a line tool started from the panel.

**From MCP:** Full — the model functions/commands are plain code; only the interactive line tool needs the mouse.

- Module: `ingetrazo_plugin_lineas_color` (+ `.model`, `.styles`, `.controller`, `.render`, `.tool`)
- Entry points (`md = sys.modules['ingetrazo_plugin_lineas_color.model']`):
  - `md.make_line(a, b, color, layer=None, flags=None, width_px=1.0, pattern="solid")` → a new `Group` "Línea de color" with one edge (a, b QVector3D in m; color `"#rrggbb"`). Not added: `groups.append(g)`.
  - Polyline/outline in one styled mesh: `s = md.DrawingSession(viewport)`, then per segment `md.AddSessionSegment(s, a, b, color, width_px, pattern).do(scene)` — welds, splits crossings, closed planar loops get faces; `s.group` is the container (added to `scene.groups` on first segment).
  - `md.ColorSelection(scene, color, width_px=None, pattern=None)` — restyle the current selection (line-only groups, or loose face-less edges which are moved into a new styled group); `color=None` resets. `.changed`; raises `ValueError` for bad selections.
  - Reading: `md.group_color/group_width/group_pattern(g)`, `md.is_line_group(g)`, `md.styled_segments(scene, interactive=False)` → `{(color, width, pattern): [(a, b), …]}` world segments.
  - Controller `viewport.window()._lineas_color_controllers['lineas_color']`: `.choose(color)`, `.width_px`, `.pattern`, `.apply(reset=False)` (errors only to the status bar).
- Saved data: `group.ext["lineas_color"]` = `{color, width_px, pattern, drawing_mesh, edges: [{a, b, color, width_px, pattern}]}` (in the .igz).
- Needs: nothing extra (refuses to load on viewports lacking its patched methods).
- Pitfalls: `controller.choose_custom()` opens a modal `QColorDialog`. On groups created in the same script use `cmd.do(scene)` (one outer undo step covers them); on EXISTING groups use `viewport.history.execute(cmd)` + check `last_error` (the outer snapshot does not track `group.ext`). `AddSessionSegment`/`ColorSelection` clear the selection. Width/pattern are on-screen only; exports carry plain edges. Messages are Spanish.

```python
import sys
md = sys.modules['ingetrazo_plugin_lineas_color.model']
g = md.make_line(QVector3D(10,0,0), QVector3D(12,0,0), "#e53935", width_px=3, pattern="dashed"); groups.append(g)
s = md.DrawingSession(viewport)
pts = [QVector3D(10,1,0), QVector3D(11,1,0), QVector3D(11,2,0), QVector3D(10,2,0)]
for a, b in zip(pts, pts[1:] + pts[:1]):
    md.AddSessionSegment(s, a, b, "#1e88e5", 2.0, "solid").do(scene)
print(len(s.group.mesh.edges), len(s.group.mesh.faces), md.group_color(s.group))
```

### Angle Dimension — `Angle_Dimension`
1.0.0 · Ryszard Jurdeczka · MIT · Community

*Live-tested in IngeTrazo 0.5.9.*

Measures the angle at a vertex between two arms (three clicks) and keeps it as a permanent orange overlay — two arms, an arc and a "60.0°" label — stored in the drawing. **In the app:** Extensions ▸ Draw angle dim (Ctrl+Alt+A); Extensions ▸ Angle dim tools ▸ Undo last / Delete (pick) / Clear all / About.

**From MCP:** Full — the maths are pure functions and the stored list is a module global plus `_commit()`; only Clear all / About open message boxes (easy to replicate).

- Module: `ingetrazo_plugin_Angle_Dimension` (single file; private helpers, no public API besides `setup`)
- Entry points:
  - `angle_at(vertex, a, b)` → degrees 0–180 (unsigned) between vertex→a and vertex→b, or None if an arm has zero length. `angle_between(u, w)` same for two direction vectors. All args `QVector3D`, world metres.
  - `arc_points(vertex, a, b, radius, n=48)` → n+1 `QVector3D` on the arc from arm a to arm b (radius in m; the overlay uses 0.3 × shorter arm).
  - `_DIMS` — working list of `(vertex, a, b, degrees)` tuples (QVector3D ×3, float). Append/delete, then `_commit()` → bool: writes `_APP.set_document_data(_encode(_DIMS))` (nested `SetPluginDataCommand`, one undo step; errors are swallowed → False + status message).
  - `_encode(dims)` → `{"format": 1, "dimensions": [{"v","a","b": [x,y,z]}]}`; `_decode(data)` → list of tuples (angle recomputed). `_refresh_from_document()` reloads `_DIMS`; `_undo_last()` pops the newest and commits (no dialog).
  - `_APP` = the plugin's real `ExtensionApp` (key `Angle_Dimension`); `_APP.document_data(None)` reads the stored dict.
- Saved data: `scene.plugin_data['Angle_Dimension']` (format above). If that key is absent it shows, read-only, data from a key starting `Draw_Angle` (dev revisions). No QSettings, no group data.
- Needs: nothing extra.
- Pitfalls:
  - NEVER call `_clear_all` (QMessageBox.question) or `_show_about` (QMessageBox.information). Clear with `_DIMS.clear(); _commit()`, or `_APP.set_document_data(None)` to remove the key.
  - The dimensions are free 3D points, not tied to geometry (moving/deleting the model does not update them) and are not core dimensions (`scene.dimensions`). Arms are drawn to exactly the points given.
  - If the script changes nothing else, run_python drops its own empty step, so the dimension is ONE undo entry; with geometry changes too there are two.
  - MCP `undo` reverts the data but does not emit `sceneVersionChanged`, so the overlay stays stale until the next run_python (which refreshes it) — or call `_refresh_from_document()`.
  - The interactive tool (`viewport.set_active_tool(m._tool_class()())`) needs mouse clicks; useless over MCP.

```python
import sys
m = sys.modules['ingetrazo_plugin_Angle_Dimension']
V, A, B = QVector3D(10, 0, 0), QVector3D(12, 0, 0), QVector3D(11, 1.732, 0)
ang = m.angle_at(V, A, B)                       # 60.0
m._DIMS.append((V, A, B, ang))
if not m._commit() or viewport.history.last_error:
    raise RuntimeError(viewport.history.last_error or "angle dim not saved")
print(round(ang, 1), len(m._APP.document_data({}).get("dimensions", [])))
```

### Parametric Node Editor — `node_editor`
1.1 · seghier · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

A visual node graph (sliders, maths, points, curves, boxes/cylinders/spheres, extrude, roofs, image sampler…) whose "IngeTrazo Output" node streams the result into the model live or bakes it as a group. **In the app:** side-tray tab "Node Editor" (can float), Extensions ▸ Node Editor ▸ Open, tool shortcut Ctrl+Shift+N.

**From MCP:** Partial — the graph engine and the ~40 node classes run headless; drawing/wiring the visual graph and .itgraph save/load are UI.

- Module: `ingetrazo_plugin_node_editor` (+ `.engine` NodeGraph, `.nodes_library` node classes + `NODE_REGISTRY`, `.models` Point3D/Vector3D/MeshData/`create_box(center, dx, dy, dz, centered)`, `create_cylinder(base, r, h, segments)`, `create_sphere`, `extrude_profile(points, vector)`). It also puts its own folder on `sys.path`.
- Entry points:
  - `G = engine.NodeGraph()`; `n = G.add_node(NL.BoxNode())`; `G.connect(src.outputs[i], dst.inputs[j])`; `G.evaluate(context=None)` → ms; errors land in `node.error` (never raised).
  - Unwired inputs: set `node.inputs[i].value` (a `widget_values[<port name>]` entry wins over it); sliders read `widget_values['value']`. Box inputs: Center (Point3D, min corner unless Centered), Size X/Y/Z (m). Rectangle: Origin, Width, Length, Centered → outputs Profile, Mesh. Extrude: Profile, Height (m), Direction.
  - `IngeTrazoOutputNode` (inputs Geometry, Texture, Group Name, Layer, Material): after evaluate, `out.last_mesh_data` is a `MeshData` (`faces[].vertices/holes` of Point3D with `.x .y .z`).
  - `out.bake(app, is_live=False)` → nested `SnapshotImport` adding group "<Group Name>[ n]"; `is_live=True` appends/updates a group tagged `g.ext["node_editor_id"]` directly.
  - The live panel: `viewport.window()._node_editor_panel` (`.graph`, `.run_evaluation()`, `.build_example_box()` / `_house` / `_spiral_staircase` / `_column_grid` / `_wave_surface`).
- Saved data: none in the document (graphs only in .itgraph files via dialogs); live groups keep `ext["node_editor_id"]`.
- Needs: nothing extra (shapely optional for roof/offset nodes, numpy for terrain bakes).
- Pitfalls: both `bake` modes set `scene.terrain = None` (drops a georef terrain, not undone) and swallow exceptions (only logged), and non-live bake adds a nested undo step — prefer converting `last_mesh_data` yourself. Evaluating with `context={"app": app}` streams into the scene. Bake drops colours/materials. Panel Bake with no output node, Save/Open graph, reload errors → QMessageBox/QFileDialog.

```python
import sys
NL = sys.modules['ingetrazo_plugin_node_editor.nodes_library']
E = sys.modules['ingetrazo_plugin_node_editor.engine']
G = E.NodeGraph(); box = G.add_node(NL.BoxNode()); out = G.add_node(NL.IngeTrazoOutputNode())
box.inputs[0].value = NL.Point3D(12, 0, 0); box.inputs[3].value = 0.5
G.connect(box.outputs[0], out.inputs[0]); G.evaluate(context=None)
m = Mesh()
for f in out.last_mesh_data.faces:
    m.add_face([QVector3D(p.x, p.y, p.z) for p in f.vertices])
groups.append(Group(m, name='Box from graph'))
```

### Parametric solids — `igz_solids`
1.0.0 · Ezequiel M. Rezende · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

14 parametric solids (box, wedge, capsule, cylinder, tube, sphere, domus, spherical wedge, cone, pyramid, geodesic sphere/football, helix, prism, torus), each inserted as its own watertight group. **In the app:** «Solids» toolbar (object name `igz_solids`) and Extensions ▸ Solids ▸ submenu; each button opens a parameter dialog, then places at the cursor (click) or at the origin.

**From MCP:** Full — every shape is a pure `build(params, origin)` returning faces; only the parameter dialog is modal.

- Module: `ingetrazo_plugin_igz_solids` (only re-exports `setup`), `._core`, and one submodule per shape: `.tb_box`, `.tb_capsule`, `.tb_cylinder`, `.tb_sphere`, `.tb_frustrum`, `.tb_geodesic`, `.tb_helix`, `.tb_prism`, `.tb_torus` (relative imports; get them with `importlib.import_module('ingetrazo_plugin_igz_solids.tb_box')`).
- Entry points:
  - `tb_<shape>.build(params: dict, origin: QVector3D)` → `(faces, cap_indices, smooth)`; faces are QVector3D loops (metres) or `(outer, [holes])`. Missing keys fall back to defaults. `origin` = base corner (box/wedge), base centre (cylinder, cone, prism, helix axis, capsule south pole), centre (sphere, geodesic, torus — these extend below `origin.z`). Keys (dialog defaults):
    - box: `width` X 1, `length` Y 1, `height` Z 2, `shape` "box"|"wedge" · capsule: `radius` 0.5, `body_height` 1.5, `h_segments` 24, `v_segments` 8 (per cap)
    - cylinder: `shape` "cylinder"|"tube", `outer_radius` 0.5, `inner_radius` 0.25, `segments` 24, `height` 2
    - sphere: `shape` "sphere"|"domus"|"segment", `radius` 1, `h_segments` 24, `v_segments` 12, `angle` 45 (degrees, segment)
    - frustrum: `shape` "cone"|"pyramid" (cone = smooth), `base_radius` 1, `top_radius` 0 (=apex), `height` 2, `sides` 24 (pyramid button uses 4)
    - geodesic: `base` "tetra"|"octa"|"icosa"|"football", `radius` 1, `frequency` 0–4 (2), `finish` "flat"|"smooth"
    - helix: `helix_radius` 1, `tube_radius` 0.25, `height` 2, `turns` 2, `helix_segments` 24 (per turn), `tube_segments` 12
    - prism: `radius` 1 (circumradius), `height` 2, `sides` 6 (≥12 → smoothed) · torus: `major_radius` 0.75, `minor_radius` 0.5, `major_segments` 24, `minor_segments` 16
  - `_core.add_solid(viewport, faces, caps=(), smooth=False, name=None)` → new `Group` (fresh mesh, `orient_outward`, soft seams except caps), via `InsertGroupCommand` (selects it; goes inside an open group).
  - `_core.load_modules()` → the tb modules in toolbar order; `_core.module_actions(mod)` → the buttons (`name`, `icon`, preset `params`).
- Saved data: none (plain groups named after the button, e.g. "Box", "Tube").
- Needs: nothing extra.
- Pitfalls: never call `run_tool` (modal `ParamDialog.exec()`) or `SolidPlacementTool` (waits for a click). `add_solid` is a nested undo step — check `viewport.history.last_error` and raise; or rebuild its five lines yourself (below) for a single step. Out-of-range values are silently clamped (tube inner ≥ outer → 0.7·outer, torus minor ≥ major → 0.2·major, helix tube ≥ helix radius → 0.4·helix).

```python
import importlib
from core.orient import orient_outward
tb = importlib.import_module('ingetrazo_plugin_igz_solids.tb_cylinder')
faces, caps, smooth = tb.build({'shape': 'tube', 'outer_radius': 0.3, 'inner_radius': 0.25,
                                'height': 2.0, 'segments': 32}, QVector3D(10, 0, 0))
m = Mesh()
for f in faces:
    m.add_face(*f) if isinstance(f, tuple) else m.add_face(f)
orient_outward(m)
if smooth:
    capf = {m.faces[i] for i in caps}
    for e in m.edges:
        if len(e.faces) == 2 and not any(f in capf for f in e.faces):
            e.soft = True
g = Group(mesh=m, name='Tube'); scene.groups.append(g)
print(g.name, len(m.faces))
```

### SubD (subdivision modelling) — `subdiv`
1.0 · Nicolas Marc · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Catmull-Clark subdivision modelling: a coarse control cage (kept in `group.ext["subdiv"]`) is edited with 14 tools (create, edit, crease, grow face/edge, stretch, offset, squeeze, divide, delete, close, bridge, shell) plus symmetry, join/split and convert; the group's mesh is the smoothed surface, and Finish turns it into an ordinary group or solid. **In the app:** side-tray tab «SubD» (also Extensions ▸ SubD panel); cage picking/dragging in the viewport via an overlay + pickable.

**From MCP:** Full — cage, tools, smoothing and group building are pure Python (no Qt); only the panel is UI.

- Module: `ingetrazo_plugin_subdiv` (setup only) + pure `.edit` (Cage), `.ops` (tools), `.group` (IngeTrazo adapter), `.subdivide`, `.follow`; `.panel` (UI, loaded by setup). Relative imports only; use `importlib.import_module('ingetrazo_plugin_subdiv.edit')`.
- Entry points (coordinates are `(x, y, z)` tuples in metres; faces are vertex-index tuples):
  - `edit.Cage(verts, faces, creases=None, colors=None, sym=None)`; `Cage.box(size=(1,1,1), center=(0,0,0))` (the panel's Create puts the lower corner on the origin). Methods: `crease_by_angle(deg=30, sharpness=4)` → count, `set_crease(edges, s)`, `move/rotate/stretch/squeeze`, `extrude_face(fi, h)`, `inset_face`, `divide_face`, `loop_cut`, `close_hole`, `bridge`, `shell(t, "outside"|"inside"|"center")`, `orient()`, `is_closed()`, `signed_volume()`, `solid_report()` → `(ok, problems, volume)`, `bounds()`, `face_normal(fi)`, `to_dict()/from_dict()`.
  - `ops.apply_tool(cage, tool, sel, value=0.0, axis="auto", pending=None, sym=None)` → `Result(selection, message, pending)`; edits the cage in place. `tool` ∈ `ops.SPECS` keys except "create": edit, crease, grow_face, grow_edge, stretch, inset_face, offset_edge, squeeze, divide, delete_face, close, bridge, shell. `sel` = `("v", i)`, `("e", a, b)` or `("f", i)`; `axis` "auto"|"x"|"y"|"z" (divide: "face" = that face only; shell: the mode). Raises `ValueError` (status-bar text) when the selection doesn't fit. Multi-item: `crease_items`, `grow_items`, `squeeze_items`, `bridge_two(cage, first, second, twist)`.
  - `group.build_group(cage, levels, Group, Mesh, QVector3D, name=None, lines="cage"|"all"|"none", merge_flat=False)` → new Group (not added to the scene): triangulated smooth surface + `ext["subdiv"]` = `{cage, levels, lines, ref}`. `merge_flat=True` merges each flat patch into one polygon (what Finish uses; then set `g.ext` without `"subdiv"`).
  - `group.read(g)` → `(cage, levels, lines)` or None; `is_subdiv(g)`; `cage_from_mesh(mesh)` → `(cage, holes_dropped)` (Convert; then `crease_by_angle`); `auto_levels(cage, budget=25000)`, `triangle_count(cage, levels)`; `join_cages(a, b)`; `carry_over(old, new)` (name, layer, ifc, uid…).
- Saved data: `group.ext["subdiv"]` per SubD group (cage dict, levels, lines, follow ref). No document_data or QSettings.
- Needs: nothing extra (NumPy only for the panel).
- Pitfalls: the panel commits with its own `_Swap` command (nested undo) and reads its widgets — from a script use the pure functions and swap groups yourself (`scene.groups[i] = new` after `carry_over(old, new)`). A cage moved with the group is only followed via the panel's `StateCache` (`sys.modules['ingetrazo_plugin_subdiv.panel']._state(g)`); `read(g)` returns the cage as built. Keep cages coarse (`cage_from_mesh` refuses > 7000 faces; level 3–4 multiplies faces by 64–256). Face holes are dropped on convert.

```python
import importlib
ed, ops, sg = (importlib.import_module('ingetrazo_plugin_subdiv.' + n) for n in ('edit', 'ops', 'group'))
cage = ed.Cage.box((1, 1, 1), center=(10.5, 0.5, 0.5))
top = next(i for i in range(len(cage.faces)) if cage.face_normal(i)[2] > 0.9)
ops.apply_tool(cage, 'grow_face', ('f', top), 0.5)
cage.crease_by_angle(30, 4)                      # keep it boxy; skip for a pebble
g = sg.build_group(cage, 2, Group, Mesh, QVector3D, name='SubD', merge_flat=True)
g.ext = {}                                       # finished (ordinary group); keep ext to stay editable
scene.groups.append(g)
print(len(g.mesh.faces))
```

### Divide Points — `divide_points`
0.2.1 · G. Andreotti · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Places IngeTrazo guide points (snappable, saved in the .igz) along the selected lines/curves, by number of points or by distance, starting from both ends or from the middle; connected edges are joined into paths. **In the app:** Extensions ▸ Divide Points ▸ Divide Points… (modeless dialog with orange preview crosses) / Show in toolbar; right-click "Divide Points…" when edges are selected.

**From MCP:** Full — pure placement maths in `geometry`, and one function that adds the guide points + registry as a single command.

- Module: `ingetrazo_plugin_divide_points` + `.geometry` (pure, no Qt), `.scene_ops`, `.ui`, `.toolbar`, `.texts`. Relative imports only. Load with `importlib.import_module(P + '.geometry')`.
- Entry points (metres, world coordinates, points as `(x, y, z)` tuples):
  - `scene_ops.selected_paths(scene)` → list of `geometry.Path` from the Edge objects in `scene.selection` (active mesh only; faces/groups ignored). `geometry.build_paths([(key_a, key_b, (x,y,z), (x,y,z)), …])` does the same from raw segments (keys = vertex identity; branch vertices split paths; loops become closed paths). `Path(points, closed=False)`, `.length`.
  - `geometry.place(paths, mode, *, count=None, spacing=None, origin='ends', limit=None)` → de-duplicated points. `mode` `'count'` (N points; 'ends' puts the first/last ON the ends, gap L/(N−1); 'middle' gap L/(N+1), off the ends; closed: N gaps of L/N) or `'distance'` (every `spacing` m: 'ends' walks in from both ends, remainder in the middle; 'middle' walks out from the midpoint; `limit` caps the count). Raises `geometry.PlacementError` (count 1–10000, spacing ≥ 0.1 mm). `geometry.positions(length, mode, …, closed=False)` → arc-length parameters; `geometry.point_at(path, s)` → point.
  - `scene_ops.add_points(viewport, 'divide_points', points)` → number added (skips places that already have a guide point); adds `core.guide.Guide` points + updates the registry in ONE nested command; raises RuntimeError if the history reports an error. `scene_ops.delete_points(viewport, 'divide_points')` → removes every guide point this plugin registered (all of them, not just the last batch). `scene_ops.live_points(scene, 'divide_points')` → registered points still present.
- Saved data: `plugin_data['divide_points']` = {version: 1, points: [[x,y,z], …]}; QSettings `plugins/divide_points/show_toolbar`.
- Needs: nothing extra.
- Pitfalls: guide points are not geometry — `query_model` counts won't show them; read `scene.guides` (`direction is None`). add/delete run their own `viewport.history.execute` (separate undo step inside run_python; run_python's own snapshot does NOT track `scene.guides`, so always go through `add_points`, never append Guides by hand). Points are rounded to float32 (`scene_ops.as_stored`). The dialog is modeless (`show()`), harmless but unnecessary.

```python
import importlib
P = 'ingetrazo_plugin_divide_points'
G, S = (importlib.import_module(P + s) for s in ('.geometry', '.scene_ops'))
path = G.Path([(10, 0, 0), (14, 0, 0), (14, 3, 0)])          # L-shaped, 7 m
pts = G.place([path], 'count', count=8, origin='ends')        # a point every 1 m
print(S.add_points(viewport, 'divide_points', pts), len(S.live_points(scene, 'divide_points')))
```

### IngeTrazo Unfold — `unfold`
1.0 · Thomas Lai · MIT · Community (catalogue id `unfold`)

*Live-tested in IngeTrazo 0.5.9.*

Rotates a selected group of faces about the edge they share with a reference face, so they lie flat in the reference face's plane (one hinge per operation; repeat to unfold a net). **In the app:** Extensions ▸ Unfold ▸ Unfold starts a click workflow (single-click a face or double-click for its coplanar group, then click the adjacent reference face; Esc/Space exits).

**From MCP:** Full — the hinge rotation is a plain function taking the reference face; the selection comes from `scene.selection`.

- Module: `ingetrazo_plugin_unfold` (single file).
- Entry points:
  - `_unfold_to_reference(viewport, ref)` → True/False. Rotates every Face in `scene.selection` that lives in `scene.mesh` (except `ref`) about the first `ref` edge shared with one of them, by the angle that takes that face's normal onto `ref.normal()`. Faces are removed and rebuilt (new Face objects, attrs copied), orphan edges deleted, sheet edges made hard/visible; afterwards `scene.selection` = rebuilt faces + `ref`. On failure (no shared edge, already coplanar, degenerate) it flashes a status message and returns False — no dialog.
  - `_coplanar_faces(scene, ref)` → list of faces in `scene.mesh` edge-connected and coplanar with `ref` (hidden edges block). What the double-click selects.
  - `_same_plane(face_a, face_b)` → bool.
- Saved data: none (geometry only).
- Needs: nothing extra.
- Pitfalls: works only on the ACTIVE mesh (`scene.mesh`: loose geometry, or the group currently open for editing); faces of closed groups are ignored. The selected faces must be edge-connected to `ref` through a real shared mesh edge. Which way the sheet swings follows the face normals (outward box normals unfold outward; inconsistent normals fold onto the reference). Nested undo: it runs `viewport.history.execute(UnfoldCommand(...))` (own undo step); it catches exceptions and returns False, so check the return value and `viewport.history.last_error`. The interactive workflow installs an event filter on the viewport — don't call `begin()` from a script.

```python
import sys
U = sys.modules['ingetrazo_plugin_unfold']
m = scene.mesh
ref = m.add_face([QVector3D(10, 0, 1), QVector3D(11, 0, 1), QVector3D(11, 1, 1), QVector3D(10, 1, 1)])
side = m.add_face([QVector3D(11, 0, 0), QVector3D(11, 1, 0), QVector3D(11, 1, 1), QVector3D(11, 0, 1)])
scene.selection.clear(); scene.selection.update(U._coplanar_faces(scene, side))
if not U._unfold_to_reference(viewport, ref) or viewport.history.last_error:
    raise RuntimeError('unfold failed')
print([[(v.x(), v.y(), v.z()) for v in f.vertices] for f in scene.selection if f is not ref])
```

### UV (UV mapping tools) — `igz-uv`
1.0.0 · Ezequiel M. Rezende · GPL-3.0-or-later · Community (catalogue id `igz_uv`)

*Live-tested in IngeTrazo 0.5.9.*

Writes per-face texture UV maps (`face.attrs['texture']['uvw']`) with six projections (planar, spherical, cylindrical around Z, box, quad 0–1, tube), a fan triangulator that keeps UVs, and U/V offset/rotate/scale. Faces with no texture get the bundled UV-grid image. **In the app:** "igz-UV" toolbar, Extensions ▸ igz-UV submenu, side-tray panel "Ajuste UV" (sliders), UV Transform tool (arrow keys).

**From MCP:** Full — the map commands only need faces in `scene.selection` and a viewport; the pure UV maths is reachable too.

- Module: package `ingetrazo_plugin_igz-uv` only forwards; the code is a separate module loaded by path as `sys.modules['igz_uv_igz_UV']` (also `sys.modules['ingetrazo_plugin_igz-uv']._UV`). Not under the package name.
- Entry points (all act on the Face objects in `scene.selection`; faces of a group mesh work, points are mesh-local):
  - `planar_map(viewport)` (face-plane projection, .skp basis, uses existing texture sw/sh/rot), `spherical_map(viewport)` (around the selection centroid), `cylindrical_map(viewport)` (Z axis; V = 0 at min Z → 1 at max Z), `box_map(viewport)` (dominant plane per face, scaled by the selection's largest bbox side, so UV 0–1 spans that size), `quad_face_map(viewport)` (4-vertex faces → (0,0),(1,0),(1,1),(0,1)), `tube_map(viewport)`, `triangulate_selection(viewport)`. Each runs its own `viewport.history.execute(SnapshotImport(...))`; return None.
  - Pure helpers: `_planar_uvs(pts, normal, sw, sh, rot)`, `_spherical_uvs(pts, center)`, `_cylindrical_uvs(pts, (zmax, zmin), center)` → [(u, v)] per point (tuples); `_ensure_uv_texture(face)` (adds grid texture if none); `_apply_uvw_to_face(face, uvw)`; `_read_uvw(face)` → 8 floats or None; `_uv_at(uvw, p)` → (u, v); `_compose_uvw(uvw, cu, cv, su, sv, cos, sin, du, dv)` (panel's block transform). Fit with `core.texture.fit_uv_affine(points, uvs)`.
- Saved data: only face attrs — `texture` {path, sw, sh, rot, uvw} and `mat` = "igz-UV Texture" when the grid was auto-applied. No document data, no QSettings.
- Needs: nothing extra (grid image `icons/texture_grid.jpg` inside the plugin folder; its absolute path is written into the face).
- Pitfalls: with no faces selected every map command calls `QMessageBox.information` (modal → stalls the bridge); `quad_face_map` also does when no selected face has 4 vertices; Help… is a QMessageBox. Always fill `scene.selection` with faces first. Nested undo: each command pushes its own entry — check `viewport.history.last_error` after it and `raise`. Its `SnapshotImport` only snapshots `scene.mesh` (loose geometry) + added groups, so UV changes on faces of an EXISTING group are not reverted by undo (neither the plugin's step nor run_python's) — save `dict(f.attrs['texture'])` first if you may need to restore. `triangulate_selection` adds the triangles to `scene.mesh` (the active/loose mesh) and removes the originals from it, so use it only on loose geometry or inside the open group, never on faces of a closed group. The panel's Reload/sliders and the arrow-key tool also mutate (preview writes history per slider move) — don't drive them from scripts.

```python
import sys
U = sys.modules['igz_uv_igz_UV']
g = extrude([(10, 0), (12, 0), (12, 1), (10, 1)], 0.0, 0.5, name='uv test')
scene.selection.clear(); scene.selection.update(g.mesh.faces)
U.box_map(viewport)
if viewport.history.last_error: raise RuntimeError(viewport.history.last_error)
print([U._read_uvw(f) is not None for f in g.mesh.faces])
```

### Proxies — `proxy_tool`
1.0 · Pesi (pesi3d.de) · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Swaps the geometry of selected groups/component instances (trees, furniture, cars) for a light proxy — bounding box, convex hull, vertex cloud (crosses) or the original reduced by vertex clustering — keeping name, place, layer and instances; the full geometry is written to external `.igz` files (one per object, all together, or per layer) and swapped back with Show Originals / Show Proxies. **In the app:** Extensions ▸ Proxies ▸ Create Proxies… (non-modal dialog with Live Preview), Show Originals, Show Proxies, Select All Proxies; same entries on a «Proxies» PESI3D toolbar and in the right-click menu.

**From MCP:** Full — the proxy builders are pure NumPy functions and `create_proxies` / `show_originals` / `show_proxies` run without the dialog (one swap command each); only the dialog's folder and colour pickers are modal.

- Module: `ingetrazo_plugin_proxy_tool` (single file).
- Entry points:
  - Pure: `gather_geometry(group)` → `(P (n,3) positions in the definition frame, T (m,3) triangles, A per-triangle attrs, S loose edges)`; `face_count(group)`; `box_spec(P)`, `hull_spec(P, points=64)`, `cloud_spec(P, pct=5.0, max_points=2000)`, `source_spec(P, T, A, pct=10.0, keep_mat=True)`, `make_spec(P, T, A, params)` → spec `{P, faces, attrs, segs}`; `spec_face_count(spec)`; `build_mesh(spec, params)` → new `Mesh` (proxy colour / see-through, soft seams).
  - `params` = `dict(DEFAULTS, …)`: `type` box|hull|cloud|source, `hull_points`, `cloud_pct`, `cloud_max`, `source_pct`, `keep_mat`, `color` [r,g,b] 0–1, `see_through`, `min_faces_on`, `min_faces`, `mode` individual|together|layer.
  - `plan_proxies(viewport, groups, params, folder)` → plan dict (meshes, swap items, files) without touching document or disk.
  - `create_proxies(viewport, groups, params, folder)` → `{"definitions", "objects"}` or None: writes the originals to `folder` (`<name>_<id6>.igz`, `proxies.igz` or `<layer>.igz`; an existing file is merged), then swaps. `groups` = top-level groups (`selected_groups(scene)`); instances sharing a definition share one proxy.
  - `show_originals(viewport)` / `show_proxies(viewport)` → count; act on the SELECTED proxies, or on every proxy in the document when none is selected. `select_all_proxies(viewport)`.
  - `proxy_info(group)` → `group.ext['proxy_tool']` or None; `read_originals(path)` → `({proxy id: Group}, materials)`.
- Saved data: `group.ext['proxy_tool']` = `{id, file, rel, state 'proxy'|'original', faces, proxy_faces, local, name, params, serial}`; the original `.igz` files on disk; QSettings `plugins/proxy_tool/*` (dialog options, `last_folder`), `plugins/pesi3d/placed_toolbars`.
- Needs: NumPy (bundled). Disk space for the originals.
- Pitfalls:
  - Writes files: ALWAYS pass an explicit absolute `folder` (the dialog's default is `<document>_proxies` next to the saved document; with an unsaved document and `folder=""` the files land in the process's current directory).
  - `create_proxies`/`show_*` execute their own swap command → nested undo step inside `run_python`; undo does NOT delete the written `.igz` files. `show_proxies` rewrites the original file if the original was edited while shown.
  - Never call `open_dialog` from a script (non-modal, but its Folder…/colour buttons are modal and it keeps a live preview step). Groups that are already proxies are skipped by create. A missing original file is reported only with `flash_status` (the returned count is lower).

```python
import sys
pt = sys.modules['ingetrazo_plugin_proxy_tool']
groups = pt.selected_groups(scene)                       # top-level groups / instances
res = pt.create_proxies(viewport, groups, dict(pt.DEFAULTS, type='hull', hull_points=48),
                        '/tmp/my_proxies')               # explicit folder
print(res, [(g.name, pt.proxy_info(g)['proxy_faces']) for g in groups])
pt.show_originals(viewport)                              # e.g. before rendering/exporting
```

## 7. Architecture, BIM and structures

### OpenTrace BIM — `OpenTrace_BIM`
0.44.0 · Leandro Lopes · GPL-3.0-or-later · Community (catalogue id `opentrace_bim`; formerly stem `arquitetura_parametrica`)

*Live-tested in IngeTrazo 0.5.9.*

Parametric BIM: straight/curved multilayer walls with hosted openings, parametric doors/windows, polygon slabs, columns, beams, IFC4 (IFC4.3 experimental) export/import, classification catalogues; bundles Layer Combinations. **In the app:** one "OpenTrace BIM" side dock (pages per element + BIM/project), a tool row and overlay in the viewport; UI translated to English at startup.

**From MCP:** Full — elements are built by pure `make_*` functions and inserted by plain `Create*/Edit*` commands whose `do(scene)` can be called directly.

- Module: `ingetrazo_plugin_OpenTrace_BIM` + `.model`/`.commands` (walls), `.door_window_commands`, `.slab_model`/`.slab_commands`, `.column_model`/`.column_commands`, `.beam_model`/`.beam_commands`, `.bim`, `.ifc_export`. Relative imports only (no bare-name self-import). Load with `importlib.import_module(P + '.model')`.
- Entry points (world metres, XY points as `QVector3D`, Z of points ignored):
  - `model.make_wall_segment(start, end, values)` → Group. values: `thickness`, `height`, `base` (bottom Z), `alignment` `'left'|'center'|'right'` (body to the LEFT of start→end for 'left'); optional `material_name`, `structure` 'simple'|'composite' + `layers`, `openings` [{id, position (centre, m from start), width, sill, height}], `top_mode`/`top_level`/`top_offset`.
  - `model.make_arc_wall(start, end, sagitta, values)` → curved wall (sagitta + = bulge left, ≤ 5×chord). values must still contain a `length` key (any number, recomputed) or it raises.
  - `door_window_commands.CreateHostedFill(scene, wall, spec)` → window/door group + wall cut in one command (wall must already be in `scene.groups`). spec: `kind` 'window'|'door', `id` (unique str), `host_id=str(wall.uid)`, `opening_id` (new id → new cut), `anchor` 'left'|'center'|'right', `position` (station of the anchor, m), `width`, `height`, `sill`; window `panes` 1–16; door `family_key` 'swing_single'|'sliding_surface'|'sliding_pocket', `hinge`, `swing` ±1. Must stay ≥ 2 cm from wall ends.
  - `slab_model.make_slab(world_polygon, values, edge_specs=None, openings=None)` (start from `dict(slab_model.DEFAULTS, …)`; `base_z`, `thickness`; openings [{polygon: [[x,y,0],…] world}]); `column_model.make_column(point, values)` (`profile` 'rect'|'circle', `width`, `depth`, `diameter`, `height`, `base_z`, `rotation`°, `anchor` 'tl'…'mc'…'br'); `beam_model.make_beam(start, end, values)` (`width`, `height`, `base_z` = axis Z, `anchor` default 'mc').
  - Insert: `commands.CreateWall(g).do(scene)`, `slab_commands.CreateSlab`, `column_commands.CreateColumn`, `beam_commands.CreateBeam` (stamp IFC identity, layer, classification). Read: `model.read_wall`, `slab_model.read_slab`, `column_model.read_column`, `beam_model.read_beam`. Edit: `commands.EditWall(scene, g, values)`, `EditColumn(scene, g, values)`, `EditBeam`, `EditSlab(scene, g, values=…, …)`; walls also `PaintWallBodyMaterial(scene, g, name)`.
  - `bim.scan_scene(scene)` → {total, counts, openings…}; `bim.element_kind(g)`; `ifc_export.export_ifc(app, Path, *, data=None)` (no dialogs; `app = ExtensionApp(viewport.window(), 'OpenTrace_BIM')`).
- Saved data: per element `group.ext['arquitetura_parametrica']` (kind + parameters; key unchanged) and `…_derived`, `group.ifc`. Document: `plugin_data['OpenTrace_BIM']` (NEW key = stem; files saved under the old stem keep `plugin_data['arquitetura_parametrica']`, which this version does not read), `plugin_data['layer_combinations']`. QSettings: `arquitetura_parametrica/element_presets_v2`, `…/complex_profiles_v1`, `opentrace/bim_library_v1`, `opentrace/composition_library_v1`, `opentrace_bim/check_updates_automatically` (default off).
- Needs: nothing extra. Network only on user action: updater (GitHub API, auto-check off by default), Uniclass/bSDD classification downloads. IfcOpenShell optional for import.
- **Known bug (0.44.0 on IngeTrazo 0.5.9): windows fail.** `door_window_geometry.build_fill_parts` reads `getattr(part, "ext", {}).get(...)`, but a new group's `ext` is `None` in 0.5.9, so every window placement (`CreateHostedFill` with kind 'window', and the same tool in the app) raises `AttributeError` and rolls back. Doors take another path. Walls, slabs, columns, beams and IFC export pass.
- Pitfalls: on File ▸ New of a clean untitled doc it shows "EXPERIMENTAL · Initial project setup" — NON-modal (`show()`), so the bridge is not blocked; close it with `[d.reject() for d in viewport.window().findChildren(QDialog) if d.windowTitle().startswith('EXPERIMENTAL') and d.isVisible()]`. The Home "Configure new project", library manager, profile editor, catalogue dialogs use `exec()` — never call those slots. Create+edit in one call with `.do(scene)` directly; editing pre-existing elements needs `viewport.history.execute(...)` + `raise` on `viewport.history.last_error`. Don't start walls from `model.DEFAULTS` (its `top_profile [3,3]` overrides `height`). `scene.edit_group` must be None. Walls must not be scaled/sheared or mesh-edited (read_* raises).

```python
import importlib
P = 'ingetrazo_plugin_OpenTrace_BIM'
M, C, DW = (importlib.import_module(P + s) for s in ('.model', '.commands', '.door_window_commands'))
w = M.make_wall_segment(QVector3D(10, 0, 0), QVector3D(15, 0, 0),
        {'thickness': 0.2, 'height': 2.7, 'base': 0.0, 'alignment': 'left'})
C.CreateWall(w).do(scene)
# Fails in 0.44.0 on 0.5.9 (see Known bug); kept for when it is fixed:
DW.CreateHostedFill(scene, w, {'kind': 'window', 'id': 'win-a', 'host_id': str(w.uid),
        'opening_id': 'op-a', 'anchor': 'center', 'position': 2.5,
        'width': 1.2, 'height': 1.2, 'sill': 0.9}).do(scene)
print(M.read_wall(w)['length'], len(M.read_wall(w)['openings']))
```

### ArchXQ IT Lite · Architecture by phases — `archxq_it_lite`
0.10.2 (0.10.2-lite) · Orlando Souza · XQ · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Phase-driven architecture workflow (levels → terrain → structure → walls): every element is a record in one JSON document and the whole building is regenerated from it (walls auto-join, mitre, T/X cuts). Lite draws levels, terrain (plot, excavations), walls (line/arc/circle, and since 0.10 curtain walls) and columns; beams, slabs, doors/windows, roofs, ramps, stairs are Pro (kept and shown if present). **In the app:** "⏻ ArchXQ" switch over the 3D view, then its floating bars, levels strip, grips/radial menu and a side-tray tab (library + outliner + properties); an "ArchXQ" submenu in Extensions; right-click entries on ArchXQ objects.

**From MCP:** Partial — no public function API, but the document model is pure Python and the live instance's `_commit_all(doc)` rebuilds the building without dialogs; drawing tools, grips and edit windows are UI/modal.

- Module: `ingetrazo_plugin_archxq_it_lite` (live object: `sys.modules[...]._instance`, class `ui.ArchXQ`; None if another ArchXQ copy is already loaded) + `.model` (`load`, `new_walls`, `new_elements`, `elevations`, `add_floor`), `.walls` (pure wall engine, `why_not`), `.structure` (`build`, `levels_info`, `fit_column`, `why_not`), `.curtain` (`DEFAULTS`), `.wallmove` (`reshape`), `.compat` (`wall_groups`, `BUILT_KINDS`, `_rec`, `commit_build`). Relative imports only (no bare `archxq_it` import).
- Entry points:
  - `X._live(X.doc())` → current doc (dict; elements whose groups were deleted dropped). `doc['levels']` = [{id, name, kind 'basement'|'ground'|'floor', height m}] bottom→top; ground floor Z = `doc['project']['ground_level']` (0 without project). `structure.levels_info(doc, model.elevations)[lv]` → {z0, height, slab, under}.
  - Wall record (append via `model.new_walls(doc['walls'], [rec…])`, adds id + "Wall N"): `{'kind':'line'|'arc'|'circle', 'a':[x,y], 'b':[x,y]` (arc also `'m'` point on curve; circle `'c'`, `'r'`), `'t'` m, `'align'` 'outside'|'centre'|'inside', `'side'` ±1 (inside left/right of a→b), `'height'` 'level' or m, `'base'` m, `'level'` id}. Curtain wall: a straight wall + `'cw': True` + `curtain.DEFAULTS` (cw_s mullion spacing, cw_band, colours, cw_glass_tr…).
  - Column record (via `model.new_elements(doc['structure'], [rec…])`): `{'type':'column', 'level', 'x', 'y', 'shape' 'rect'|'round', 'w', 'd' m, 'angle'` deg, `'anchor'` 'centre'|'sw'|…, `'base'`, `'height'` 'level'|m, `'fit'` bool}. `structure.fit_column(rec, level_walls)` snaps it into a wall.
  - `X._commit_all(doc)` → rebuilds EVERY ArchXQ group from `doc` and stores it (plus terrain follow-up if a plot exists, joined into the same step). Edits: `X.apply_wall(rec)` (moved ends now drag joined walls via `wallmove.reshape`), `X.apply_element(rec)`, `X.delete_wall(id)`, `X.delete_element(id)`. Levels: `X._save_levels(doc, model.add_floor(doc['levels'], 3.0))`.
- Saved data: `plugin_data['archxq']` (`compat.KEY`, not the stem; data under the loader key is moved on read) — levels, walls, structure, openings, digs, plot, project, view state, `built`, `guides_off`. Groups: `group.ext['archxq']` = {type, id, level…}; layers `"ArchXQ · <level>"`, `"ArchXQ · Terrain"`. QSettings `archxq/…`.
- Needs: nothing extra for walls/columns; curtain walls and room outlines use the host's shapely.
- Pitfalls: `_commit_all` runs its own history command → TWO undo steps (undo both); check `viewport.history.last_error` after it and `raise`. Never edit ArchXQ meshes directly (regenerated each commit; a global `History.execute` patch adopts host Move/Rotate). Column tops now stop 1 cm under a floor above (`TOP_GAP`). Avoid `_walls_done`/`_struct_done` (need UI state; setbacks `QMessageBox`), `edit_*` windows (modal), `on_add_floor` without a project. Startup: if another ArchXQ (Pro, old `archxq_it`) is installed, `onecopy.claim` shows a `QMessageBox.information` 1.5 s after load (via QTimer) and the Lite stays off — keep a single copy. Do not author Pro-only record types.

```python
import sys, importlib
P = 'ingetrazo_plugin_archxq_it_lite'
X = sys.modules[P]._instance
M = importlib.import_module(P + '.model')
doc = X._live(X.doc())
lv = next(l for l in doc['levels'] if l['kind'] == 'ground')['id']
pts = [(10, 0), (16, 0), (16, 4), (10, 4)]
doc['walls'] += M.new_walls(doc['walls'], [
    {'kind': 'line', 'a': list(a), 'b': list(b), 't': 0.2, 'align': 'outside', 'side': 1,
     'height': 'level', 'base': 0.0, 'level': lv} for a, b in zip(pts, pts[1:] + pts[:1])])
doc['structure'] += M.new_elements(doc['structure'], [
    {'type': 'column', 'level': lv, 'x': 13, 'y': 2, 'shape': 'rect', 'w': 0.3, 'd': 0.3,
     'anchor': 'centre', 'base': 0.0, 'height': 'level'}])
X._commit_all(doc)
if viewport.history.last_error: raise RuntimeError(viewport.history.last_error)
print(len(X.doc()['walls']), len(X.doc()['structure']))
```

### IFC data — read and edit — `ifc_editor`
1.0 · smokey7428 · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Edits every attribute and property set (standard `Pset_*` from buildingSMART's IFC4/IFC2X3 tables, plus your own) of an object already tagged in the BIM panel, and writes a full IFC4 file that keeps them, with `IfcOpeningElement`/`IfcRelVoidsElement`/`IfcRelFillsElement` for walls made by the `openings` plugin. **In the app:** side-tray tab «IFC» (also Window ▸ Panels), Extensions ▸ IFC fields (Ctrl+Shift+F) and Extensions ▸ Export IFC (full)…

**From MCP:** Full — the record store, the edit command, the schema tables and the IFC writer are plain functions; only the panel's "add property set/property" and the export menu entry use dialogs.

- Module: `ingetrazo_plugin_ifc_editor` + `.store`, `.schema` (loaded at startup) and `.writer` (lazy: `importlib.import_module('ingetrazo_plugin_ifc_editor.writer')`). Relative imports only.
- Entry points:
  - `store.object_key(entity)` → `"uid:<group.uid>"` for a group with `group.ifc` set, `"ifc:<id>"` for a face with `attrs["ifc"]`, else None (untagged = not editable; tag first: `group.ifc = {"class": "IfcWall", "name": "W1"}` or `core.bim.tag_group`).
  - `store.record(scene, entity)` → `{key, class, name, schema, fields, psets, pset_types, custom, tagged}` or None; `store.records(scene)` → all tagged objects; `store.resolve(scene, key)` → `(faces, groups)`.
  - `store.edit(viewport, key, fields=None, psets=None, pset_types=None, custom=None, schema=None, ifc_class=None, name=None)` → runs `EditIfcObject` (one undo step). Each given argument REPLACES the stored one: `fields` = `{attr: value}` (e.g. `Description`, `ObjectType`, `Tag`, `PredefinedType`); `psets` = `{pset: {prop: value}}`; `pset_types` = `{pset: {prop: "IfcLabel"…}}` for non-standard props; `custom` = names of your own psets. `ifc_class`/`name` rewrite the core BIM tag too.
  - `schema.load("IFC4"|"IFC2X3")` → `Table`: `.has_entity(c)`, `.attributes(c)` (ordered, with `type`, `optional`, `enum`), `.predefined_types(c)`, `.psets_for(c, predefined_type=None)` → `[(name, entry)]`, `.pset(name)["properties"]`, `.resolve_kind(type)`, `.product_names()`.
  - `writer.save(scene, path, project_name="IngeTrazo project", schema="IFC4")` → number of elements written (EVERY tagged object in the document, one storey). Raises `ValueError` if nothing is tagged. Unknown classes are written as `IfcBuildingElementProxy`; enum values not legal for the attribute become `$`.
- Saved data: `scene.plugin_data["ifc_editor"] = {"objects": {key: {schema, fields, psets, pset_types, custom}}}`; class/name stay in `group.ifc` / `face.attrs["ifc"]`. Reads `group.ext["openings"]` (walls/hosts of the `openings` plugin) when exporting.
- Needs: nothing extra (schema tables in `data/*.json.gz`).
- Pitfalls:
  - Never call `export(app)` (save-file dialog) or the panel's `_on_add_pset`/`_on_add_property` (QInputDialog / modal dialog via a timer).
  - `store.edit` executes its own command → a nested undo step inside `run_python` (two steps; undo twice). The outer step does NOT track `plugin_data`, so the nested one is what reverts it. Check `viewport.history.last_error` right after and `raise`.
  - Only top-level groups (`scene.groups`) and loose faces count. For groups the writer uses `group.mesh` only (read from code): a container whose geometry is all in children — e.g. the `openings` plugin's walls/doors — has no faces and is SKIPPED, and a component's `xform` is not applied.
  - The written file is ASCII (`\X2\` escapes); output path is the caller's — use /tmp for tests.

```python
import sys, importlib
P = 'ingetrazo_plugin_ifc_editor'
st, sc = sys.modules[P + '.store'], sys.modules[P + '.schema']
wr = importlib.import_module(P + '.writer')
g = extrude([(10, 0), (14, 0), (14, 0.2), (10, 0.2)], 0, 3, name='W1')
g.ifc = {"class": "IfcWall", "name": "W1"}
key = st.object_key(g)
st.edit(viewport, key, fields={"Description": "Party wall", "PredefinedType": "SOLIDWALL"},
        psets={"Pset_WallCommon": {"IsExternal": True, "FireRating": "EI60"}})
if viewport.history.last_error: raise RuntimeError(viewport.history.last_error)
print(st.record(scene, g)["psets"], wr.save(scene, '/tmp/model.ifc'))
```

### Walls, doors and windows — `openings`
1.1 · smokey7428 · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Draws parametric layered walls along a plan polyline (skins from outside in, centred on the line; regenerated from `group.ext`) and places parametric doors/windows (frame, leaf/sash, glass, handle; swing symbol on its own layer). In one of its own walls an opening is data on the wall (the wall regenerates with the hole and follows resizes/erase); in a plain face wall it cuts a hole through to the back face and reveals. Tags `IfcWall`/`IfcDoor`/`IfcWindow`; take-off per skin (m², m³, kg). **In the app:** side-tray tabs «Openings», «Walls», «Take-off»; tools «Wall» and «Door»; Extensions ▸ Doors & windows ▸ (Place…, Draw a wall…, panels, Edit…, Erase, Make plan/elevation/3D, Redraw the swing symbols); right-click ▸ Edit / Erase on an opening.

**From MCP:** Full — walls, hosted and freeform openings, rebuild, erase and the take-off are plain functions run inside the plugin's `DoorCommand`; only the tools need clicks and the take-off CSV export uses a save dialog.

- Module: `ingetrazo_plugin_openings` + `.walls`, `.wall_object`, `.object`, `.hosting`, `.params`, `.profiles`, `.schedule`, `.opening`, `.geometry`, `.wall_geometry` (get them with `importlib.import_module(P + '.<name>')`; relative imports).
- Entry points (all lengths m, plan XY, Z up):
  - `walls.defaults('spouwmuur'|'binnenwand')` → wall dict; set `path` `[[x, y], …]` (≥ 2 points), `base` (Z), `height`, `skins` `[{name, thickness, material}]` (outside→in), `openings` (data, normally left to `hosting`). `walls.validate(w)` → problems.
  - `wall_object.create(scene, w, name=None)` → container Group (`IfcWall`, one child group per skin, materials created). `wall_object.regenerate(scene, wall, w)`; `wall_object.erase(scene, wall)`; `wall_object.wall_params(wall)`; `wall_object.quantities(w)` → `[{name, material, area, volume, mass}]`.
  - `params.defaults('door'|'window', type)` (door `hinged`; window `fixed`|`swing`) → params dict (`width`, `height`, `sill`, `handing` left|right, `swing` in|out, `open_angle` °, section sizes, `*_mat` names, `symbol`…); `params.validate(p)`.
  - `hosting.add(scene, wall, click, normal, p)` → opening Group in an `openings` wall: `click` = QVector3D near the wall line (picks segment + centre along it), `normal` = QVector3D of the face looked at (sets handing side). `hosting.opening_of(scene, item)` → `(wall, opening dict)`.
  - `object.create(scene, face, click, p, name=None)` → Group cut into a LOOSE face of `scene.mesh` with a parallel back face ≤ 0.80 m behind (`OpeningError` if it does not fit).
  - `object.rebuild(scene, item, p)` (hosted: wall hole follows); `object.erase(scene, item)` → bool (False = could not close the freeform hole); `object.opening_params(item)`.
  - `schedule.rows(scene)` → `{walls: [per skin], openings: [...], totals}`; `schedule.csv(rows)` → CSV text.
  - `object.DoorCommand(mutate)` / `object.run(viewport, mutate)` — one undo step capturing loose mesh, groups (children, ext, ifc, xform, layer), `plugin_data['openings']`, materials, layers.
- Saved data: `group.ext['openings']` (wall: `kind: wall` + path/skins/openings; opening: params, frame, depth, `host` {wall uid, opening id} or `outline`); `scene.plugin_data['openings'] = {"doors": {uid: {record, symbol}}}`; layers «Doors», «Windows», «Openings — plan»; materials per skin/part.
- Needs: nothing extra.
- Pitfalls:
  - Wrap every change in `viewport.history.execute(object.DoorCommand(mutate))` and check `last_error` (nested undo step; the outer `run_python` step does not track materials, plugin_data or children of existing groups). `object.run` does the same but only flashes errors and returns False.
  - Hosted openings do not register their swing symbol: `rebuild` adds a second «Swing symbol» group and `erase` leaves the first one behind (read from code; test prints the count). Use `fixed` windows or delete stray symbols.
  - Freeform cutting only sees top-level `scene.mesh`, not a face inside a group. `takeoff_panel` CSV export opens a save dialog; «Make plan/elevation/3D» changes saved views and layer visibility.

```python
import importlib
P = 'ingetrazo_plugin_openings'
om, wo, wl, hs, pm = (importlib.import_module(f'{P}.{n}') for n in ('object', 'wall_object', 'walls', 'hosting', 'params'))
w = wl.defaults('spouwmuur'); w['path'] = [[10, 0], [14, 0]]
out = {}
def mutate(s):
    out['wall'] = wo.create(s, w)
    out['win'] = hs.add(s, out['wall'], QVector3D(12, 0, 1.5), QVector3D(0, -1, 0), pm.defaults('window', 'fixed'))
viewport.history.execute(om.DoorCommand(mutate))
if viewport.history.last_error: raise RuntimeError(viewport.history.last_error)
print(out['wall'].name, out['win'].name, hs.opening_of(scene, out['win'])[1])
```

### Walls (masonry from floor plans) — `walls_tool`
1.2 · Pesi (pesi3d.de) · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Finds the walls, windows and doors in a flat floor plan (a Plan Import 1.2+ sheet, picked by line style, or plain edges, e.g. a DXF) and extrudes them into ONE solid group tagged IfcWall, with parapets under windows and lintels over openings. **In the app:** Extensions ▸ Walls ▸ Walls from Plan…, the same entry in the right-click menu, a PESI3D toolbar button; a non-modal dialog with live preview plus a click tool and overlay to correct the preselection.

**From MCP:** Full — detection (`detect` / `styled_detect`) and `build_group` are pure; append the group yourself.

- Module: `ingetrazo_plugin_walls_tool` (single file)
- Entry points:
  - `normalize_params(p)` / `default_params()`: `t_min`, `t_max` (wall thickness, m), `pairs` (walls drawn as two lines), `openings`, `open_min`, `open_max` (m), `height`, `sill`, `lintel` (m above the plan), `min_piece` (m²).
  - `read_source(scene)` → `{"src", "segs", "z", "label", "key"}` from `scene.selection` (any object with `.selection`): `src` set for a Plan Import sheet (`plan_source`), else `segs = [(x0,y0,x1,y1)]` from loose edges / selected groups (`gather(scene)` → `(segs, z)`, raises `WallsError` if empty or not flat).
  - `detect(segs, p)` → `{"walls": [{"poly" (shapely Polygon), "on", …}], "openings": [{"poly", "kind" "window"|"door", "on", "w"}], "faces", "rejected"}` (stairs and dimension chains rejected).
  - Line styles: `source_from_record(rec, matrix=None)` (rec = `group.ext["plan_import_tool"]`) → `{"styles", "segs", "fills", "all", "sid_all", "z"}`; `guess_wall_styles(src, p)` → style ids; `styled_detect(src, chosen_ids, p)` → same dict as `detect`; `style_label(st)`.
  - `build_group(walls_geom, openings, p, z, name)` → Group (closed mesh, `g.ifc = {"class": "IfcWall"}`); `walls_geom` = `shapely.ops.unary_union` of the `on` wall polys, `openings` = the dicts with kind window/door. `solid_layers` / `solid_faces` are the raw steps.
  - `run_build(viewport, walls_geom, openings, p, z, select=True)` executes its own command (nested undo step; raises on `last_error`).
- Saved data: `group.ext["walls_tool"]` (`version`, `params`, `z`, `walls` WKT, `openings` [{kind, poly WKT}]); QSettings `plugins/walls_tool/params`. Reads Niveles data `niveles` for the storey height (`level_height`).
- Needs: shapely (+ numpy) in the app's Python — NOT in IngeTrazo 0.5.9's requirements.txt; earlier notes found it present (host's shapely). Without it the dialog warns "Shapely is missing" (QMessageBox).
- Pitfalls: `open_dialog(app)` shows `QMessageBox.warning` on any error (and appends to `~/IngeTrazo/walls_tool.log`, `%LOCALAPPDATA%` on Windows) — never call it from MCP. Openings are heuristic (two facing jambs 0.40–4.00 m apart; window if a glass line spans the gap). The plan must lie flat; a sheet group moved/rotated after import is handled via its `xform` (`plan_source`). Texts follow the UI language (de/es/en).

```python
import sys; m = sys.modules['ingetrazo_plugin_walls_tool']
from shapely.ops import unary_union
p = m.normalize_params({"height": 2.5})
s = m.read_source(scene)                       # the selected plan group / edges
r = m.styled_detect(s["src"], m.guess_wall_styles(s["src"], p), p) if s["src"] else m.detect(s["segs"], p)
walls = unary_union([w["poly"] for w in r["walls"] if w.get("on", True)])
ops = [o for o in r["openings"] if o.get("on", True) and o["kind"] in ("window", "door")]
g = m.build_group(walls, ops, p, s["z"], "Masonry"); groups.append(g); print(len(r["walls"]), [o["kind"] for o in ops])
```

### Plan Import (PDF floor plans) — `plan_import_tool`
1.2.1 · Pesi (pesi3d.de) · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Imports floor plans or whole plan sets from vector PDFs as edges at their plan scale: one group per sheet, plans stacked on their storey heights, elevations/sections stood upright, with auto crop, filters (small parts, colours, thin lines, hatches), clean-up, reference point, levels and layers. **In the app:** Extensions ▸ Plan Import… and its PESI3D toolbar button (a non-modal dialog with live preview).

**From MCP:** Partial — the PDF reading, filtering and sheet-group building are pure helpers; the page list, crops, stacking/levels and layers live only in the dialog.

- Module: `ingetrazo_plugin_plan_import_tool` (package; lazy relative imports of `.pdf_vectors`, `.cleanup`, `.autocrop`, `.page_view` (Qt) — get them with `importlib.import_module('ingetrazo_plugin_plan_import_tool.pdf_vectors')`)
- Entry points:
  - `pdf_vectors.page_count(path)`, `page_info(path)` → `[(w_pt, h_pt)]`; `extract(path, page_index=0, tol=0.1)` → `PageVectors(width, height, paths=[PdfPath(runs, stroke, fill, stroke_rgb, fill_rgb, width, evenodd)], counts)` in page points; `page_texts(path, page_index)` → `(name, scale_n, [PageText])` (reads «Erdgeschoss 1:100»); `classify_page(name, texts)` → `(kind "plan"/"site"/"elev"/"section", sides, sure)`; `suggest_levels(...)`, `story_key(name)`.
  - `default_params()` (content/filter keys: `strokes`, `fills`, `skip_white`, `min_part_mm`, `min_width_mm`, `skip_hatch`, `skip_colours` ["#rrggbb"], `dedupe`, `gap_mm`).
  - `collect_runs(vec, p)` → `(runs, stats)`; `clean_runs(runs, p, mpp)` → `(runs, {"merged", "closed"})`; `metres_per_point(scale_n)`; `clip_runs(runs, (x0,y0,x1,y1))`, `runs_bbox(runs)`; `autocrop.suggest_crops(w, h, runs, texts)` → crop rects (pt).
  - `build_group(runs, mpp, name, offset=(x,y,z), axes=FLAT)` → Group of edges in metres (sheet point (x,y) → offset + x·mpp·U + y·mpp·V); `upright_placement(...)` / `layout_offsets(...)` give offset+axes for views/stacks.
  - `style_record(vec, p, crop=None)` → line-style record (what Walls reads); `unpack_style_record(rec)`.
  - `insert_groups(viewport, groups, select=True, extra=())` executes ONE command (nested undo step, raises on `last_error`); `layer_commands(groups, names)`, `merge_levels(old, [(name, z)])` for the Niveles data key `niveles`.
- Saved data: each sheet group gets `group.ext["plan_import_tool"]` = style record + `version`, `page`, `scale`, `mpp`, `offset`, `axes`. With Stacked + to_levels, document data `niveles` (merged). QSettings `plugins/plan_import_tool/params`.
- Needs: numpy (app); pypdf 6.19 vendored in `_vendor` (appended to `sys.path` only if no pypdf is importable; topology_tool ships the same copy).
- Pitfalls: the dialog uses `QFileDialog` (open), `QInputDialog` (Measure scale), a colour `QDialog.exec()` and `QMessageBox` — never call `show_dialog` from MCP. `build_group` raises `ValueError` when nothing is left after filters. `build_group` does not set `ext` or layers — add `g.ext = {KEY: rec}` yourself if Walls should read it. Big CAD sheets: `extract`/`page_texts` can take seconds per page (120 s limit).

```python
import sys, importlib; m = sys.modules['ingetrazo_plugin_plan_import_tool']
pv = importlib.import_module('ingetrazo_plugin_plan_import_tool.pdf_vectors')
path = "/tmp/plan.pdf"                       # an existing vector PDF
vec = pv.extract(path, 0); name, scale, texts = pv.page_texts(path, 0)
p = m.default_params(); mpp = m.metres_per_point(scale or 100)
runs, _ = m.collect_runs(vec, p); runs, _ = m.clean_runs(runs, p, mpp)
g = m.build_group(runs, mpp, m.sheet_name(path, 1, name, scale or 100), offset=(10, 0, 0))
rec = m.style_record(vec, p); rec.update(scale=scale or 100, mpp=mpp, offset=[10, 0, 0], axes=[list(a) for a in m.FLAT])
g.ext = {m.KEY: rec}; groups.append(g); print(g.name, len(g.mesh.edges))
```

### CAD2IngeTrazo · CAD plans to BIM — `cad2ingetrazo`
3.11 · M. Tariq · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Reads DXF/DWG floor plans by layer and builds a BIM model level by level (walls, doors, windows, columns, beams, slabs, stairs, roof, plot), all regenerated from one JSON project; also makes plan/elevation/section sheets. **In the app:** side-tray tab "CAD2IngeTrazo", Extensions ▸ CAD2IngeTrazo submenu, its own toolbar (3D / Plan / Sheets / SketchUp / CAD lines), right-click on its elements.

**From MCP:** Partial — the import → rebuild pipeline is plain functions on a dict; the panel's many settings, dialogs and exports are UI.

- Module: `ingetrazo_plugin_cad2ingetrazo` (+ `.project` = PJ, `.engine.model` = M, `.cadread`, `.sheets`); the live panel is the module global `_panel`.
- Entry points (all pure: they edit `doc`, nothing reaches the scene until `commit`):
  - `PJ.get(scene)` → the whole project dict (deep copy of `scene.plugin_data["cad2ingetrazo"]`: `arch` levels/walls/openings/structure/rooms, `imports`, `settings`…).
  - `PJ.import_level(doc, lid, path, how)` → report dict (`walls`, `openings`, `columns`, `beams`, `slab`, `rooms`, `skipped`…). `lid` = level id (ground is `"lv0"`); `path` = DXF/DWG on the app's machine; `how` = `{"unit": m per drawing unit or None=auto, "base": "origin"|"insbase"|"extents_ll"|"extents_c"|"custom", "bx","by" (custom base), "ix","iy" insertion (m), "rot" (deg), "auto": layer auto-detect, "region": [x0,y0,x1,y1] crop, + layer patterns "walls","doors","windows","columns","slab",… like "*WALL*, A-WALL*"}`. Replaces what an earlier import of that level made.
  - Levels: `doc["arch"]["levels"] = M.add_floor(levels, height)` / `M.add_basement(...)`; new id = the one not there before.
  - Extras on `doc`: `PJ.add_corner_columns(doc, lid, size=0.30)`, `PJ.add_beams_on_walls(doc, lid, w, h)`, `PJ.add_footings(doc)`, `PJ.set_roof(doc, kind="hip"|"gable"|"flat"|"none", slope=25, overhang=0.6)`, `PJ.auto_slabs(doc, t=0.15)` (return counts/bool).
  - `PJ.commit(viewport, doc, opts={})` → stores `doc` and rebuilds EVERY cad2ingetrazo group (tagged in `g.ext["cad2ingetrazo"]`; read with `PJ.tag_of(g)` → `{"type","id","level"}`) on layers `"C2I · <level>"`.
  - Sheets: `planes, views, sh = sheets.make(doc["arch"], M.elevations, {"paper":"A3","scale":"auto","cut":1.2,"elev":"yes","sect":"yes","dims":"no","rooms":"yes","prefix":"A-"}); sheets.commit(viewport, planes, views, sh)`.
- Saved data: `scene.plugin_data["cad2ingetrazo"]` (the project dict); panel choices in QSettings.
- Needs: `ezdxf` (in IngeTrazo) and `shapely` (both bundled with IngeTrazo; slabs and rooms need shapely); DWG needs LibreDWG `dwg2dxf` (else save as DXF; the other converter fallbacks are Windows-only).
- Pitfalls: `commit` and `sheets.commit` execute their own history commands → extra nested undo step; both raise if it failed (good). `commit` replaces all plugin groups and clears the selection. Avoid `_panel._import`, `export_pdf`, `export_skp`, `export_dxf`, Remove level, About (QFileDialog / QMessageBox). `_panel.show_3d()`/`show_plan()` are safe (view switches). Wall detection needs double-line walls 0.07–0.46 m apart.

```python
import importlib
PJ = importlib.import_module('ingetrazo_plugin_cad2ingetrazo.project')
doc = PJ.get(scene)
rep = PJ.import_level(doc, 'lv0', '/projects/plan.dxf',
                      {'unit': None, 'base': 'origin', 'auto': True})
print(rep['walls'], rep['openings'], rep['skipped'][:3])
PJ.commit(viewport, doc, {})
```

### planArq — `planArq-v1`
1.0.1 · José Castro Basso (FADU – UDELAR) · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Walls, pipes and steel/timber profiles (IPN, IPE, UPN, rectangular/circular tube, PGC steel framing, timber) swept along an editable polyline axis; axis and parameters are stored so the element can be re-edited. Schematic, pre-sizing only. **In the app:** side-tray tab "planArq" (Spanish UI: draw wall / pipe / profile, edit nodes, apply parameters to the selection). 1.0.1 only changes drawing interaction (points stay on the working plane; H-pan resumes the tool) — the build and data API are unchanged.

**From MCP:** Full — an element is one record dict; the plugin's `_Create` command builds it exactly as the draw tool does.

- Module: `ingetrazo_plugin_planArq-v1` (package, everything in `__init__.py`). Data key = `app.key` = the stem **`"planArq-v1"`** (it was `"planarq"` under the old stem).
- Entry points:
  - `_Create(key, group, record)` — a history Command: `viewport.history.execute(_Create('planArq-v1', Group(name=…), record))`; builds the mesh, appends the group, stores the record, selects the group.
  - `record` (metres, world coords, Z up): `{"type": "wall"|"pipe"|"profile", "nodes": [[x,y,z], …] (≥2, the axis), …}`; wall: `"width", "height", "alignment": 0 left / 1 centre / 2 right`; pipe: `"diameter"` (outer), `"wall"` (thickness, UI uses 8 % of Ø), `"color"` (key of `COLORS`: Celeste, Azul, Naranja, Rojo…); profile: `"profile_data": {"family", "name", "tipo_forma", "dimensions" (mm)}, "rotation": 0|90`. UI also stores `"level"`, `"fixed_level"` (drawing aids only, not used by the build).
  - Catalogue: `PROFILE_CATALOG` (perfiles.json), `_family_data(key)` → `{"tipo_forma", "items": {name: dims}, …}`; keys IPN, IPE, UPN, Tubo_Rectangular, Tubo_Circular, Steel_Framing_PGC, escuadrias_madera.
  - `_build(group, record)` — pure: sets `group.mesh` and `group.ifc` (IfcWall / IfcPipeSegment / IfcMember with `path_length`), raises `ValueError` (Spanish) on bad input.
  - `_Rebuild(key, group, before, after)` / `_BatchParameters(key, [(group, before, after)])` — change an existing element's record.
  - `_records(scene, key)` → `{group.uid: record}`. Live panel: `viewport.window().findChildren(P.ConstructionPanel)[0].app.key` confirms the key.
- Saved data: `scene.plugin_data["planArq-v1"]["elements"]` = `{group uid: record}`.
- Needs: nothing extra (perfiles.json + perfiles/*.jpg beside the plugin).
- Pitfalls: Documents saved with the old stem keep their records under `plugin_data["planarq"]` — the panel no longer sees them (geometry stays, but is not re-editable) unless the data is copied to the new key. `_Create`/`_Rebuild` are nested undo steps and fail silently — check `viewport.history.last_error` and `raise`. A bare `_build` + `groups.append` makes geometry the panel cannot edit. Two coincident consecutive nodes or a 180° reversal fail the sweep. Undoing a `_Create` empties the records but leaves `plugin_data["planArq-v1"] = {"elements": {}}` behind (harmless; delete the key if you need the document exactly as before). No modal dialogs. setup installs an event filter on the viewport (H-pan resume) — harmless for scripts.

```python
import sys
P = sys.modules['ingetrazo_plugin_planArq-v1']
g = Group(name='Muro')
viewport.history.execute(P._Create('planArq-v1', g, {
    'type': 'wall', 'nodes': [[10, 0, 0], [14, 0, 0], [14, 3, 0]],
    'width': 0.20, 'height': 2.5, 'alignment': 1}))
if viewport.history.last_error: raise RuntimeError(viewport.history.last_error)
print(g.ifc['class'], len(g.mesh.faces))
```

### AISC steel members — `igz_steel`
1.1.0 · David Songer · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Places structural steel members with AISC Shapes Database v16 dimensions (2,299 shapes in 13 families: W, M, S, HP, C, MC, L, WT, MT, ST, 2L, HSS, PIPE) along any direction, rebuilds them in place (stretch/edit), tags them IfcColumn/IfcBeam/IfcMember, produces a Bill of Materials, and keeps named reference grids (rectangular/radial, with levels) that draw over the viewport and snap. **In the app:** «Steel» toolbar (Place / Stretch / Edit / Bill of Materials / Grids), Extensions ▸ Steel ▸ same entries, right-click ▸ Steel grids… / Edit steel member… / Stretch steel member.

**From MCP:** Full — shape lookup, member building, insert/rebuild commands and the BOM are plain functions; the place/stretch tools need clicks; the member/BOM/grid windows are non-modal, but BOM Export CSV opens a save dialog and the grid window's Add levels… a modal dialog.

- Module: `ingetrazo_plugin_igz_steel` + `._db`, `._host`, `._bom`, `._units`, `._grid`, `._grid_host`, `._core` (all loaded by setup; relative imports). Live session: `next(iter(sys.modules[P + '._core']._SESSIONS.values()))` (`.db`, `.grids`).
- Entry points (metres, world Z up):
  - `_db.load()` → `Database`: `.find(label)` (imperial `W12X26` or metric `W310X38.7`, case-insensitive) → `Shape` (`family`, `label`, `label_metric`, `dims` in, `weight_per_m()` kg/m) or None; `.families()`, `.shapes(family)`, `.search(text, family=None)`.
  - `_host.new_params(label, unit_system 'imperial'|'metric', insertion 'centroid'|'top'|'bottom'|'left'|'right'|'top_left'|… (profile point ids), roll_deg, fillets bool, segments 8–96)` → params dict.
  - `_host.member_parts(shape, params, start (x,y,z), direction (x,y,z), length)` → parts dict (mesh in local coords + `xform`, name, ifc, ext). Length < 1 mm makes a flat "profile" object. `_host.make_group(parts, layer=None)` → Group (not yet in the scene).
  - `_host.run(viewport, _host.insert_command(group, layer=None))` → error text or None (adds the group, assigns/creates the layer). `_host.run(viewport, _host.rebuild_command(group, parts, layer=None))` — replace a member in place (stretch = same start/direction, new length).
  - `_host.member_frame(group)` → `(start, direction, length, roll_deg)` read from the live xform; `member_params(group)`; `is_steel_group(group)`; `iter_groups(scene)` (nested too); `member_record(group, db)`.
  - `_bom.collect(records)` → `BOM(rows[mark, label, length_m, weight_kg, layer, ifc_class, uid], summary, count, total_length_m, total_weight_kg)`; `_bom.to_csv(bom, _units.fmt_len, _units.fmt_weight)` → CSV text.
  - Grids (pure): `_grid.parse_spacing_list("25', 3@30'", units)` → metres; `_grid.RectGrid(id, name, origin, rotation_deg, x_spacings, y_spacings, …)` → `.intersections()` `[Node(names, at)]`, `.lines()`; `_grid.dump_block(grids, active_id, units)` / `parse_block(raw)`.
- Saved data: `group.ext['igz_steel']` = params + `start`, `direction`, `length`; document data `plugin_data['igz_steel']` = grid block `{version, grids, active_grid, units}` (write with `session.grids.save_block(block)` or `ExtensionApp(viewport.window(), 'igz_steel').set_document_data(block)` — one nested undo step).
- Needs: nothing extra (`data/aisc_v16.json` ships with it).
- Pitfalls:
  - Avoid `SteelSession.insert/rebuild/open_*`, `single_selected`, `on_update` — they pop `QMessageBox` on errors or with a group open; `grids.delete` asks `QMessageBox.question`.
  - `run(...)` executes its own command: a nested undo step inside `run_python` (two undo steps). It returns the error text; raise on it.
  - Members are refused (by the session) inside an open group; `insert_command` would put them in the open group. Close groups first.
  - A member is recognised by `ext['igz_steel']` + a non-None `xform`; exploding/making unique loses that.

```python
import sys
P = 'ingetrazo_plugin_igz_steel'
H, DB, BOM = (sys.modules[f'{P}.{n}'] for n in ('_host', '_db', '_bom'))
db = DB.load(); shape = db.find('W12X26')
params = H.new_params(shape.label, 'imperial', 'centroid', 0.0, True, 24)
g = H.make_group(H.member_parts(shape, params, (10, 0, 3), (1, 0, 0), 4.0))
err = H.run(viewport, H.insert_command(g, 'Steel'))
if err: raise RuntimeError(err)
bom = BOM.collect([H.member_record(x, db) for x in H.iter_groups(scene) if H.is_steel_group(x)])
print(g.ifc['class'], bom.count, round(bom.total_weight_kg, 1))
```

### Windowizer — `windowizer`
0.5.9 (ships with the app) · Bane Andreev (port of Rick Wilson's Windowizer 3) · GPL-3.0-or-later · Bundled example

*Live-tested in IngeTrazo 0.5.9.*

Turns a face drawn on a wall into a parametric window: one container group (`IfcWindow`) holding a Frame and Glass panes (rows × columns), and opens the wall (through to its back face, reveals, or just a hole). Edit rebuilds it in place from its stored parameters; Erase removes it and closes the wall. **In the app:** Extensions ▸ Windowizer ▸ Windowize selected faces / Edit window… / Inherit settings / Erase window, and the same in the right-click menu. Since 0.5.9 each window shows as «Window» in the Model Browser, with panes and size in its tooltip.

**From MCP:** Full — `make_window` / `rebuild_window` / `delete_window` take a scene and plain dicts; only the menu commands open dialogs.

- Module: `ingetrazo_plugin_windowizer`
- Entry points:
  - `ensure_default_materials(scene)` — adds the «Windowizer frame» / «Windowizer glass» materials; call it first (the builders refuse unknown material names).
  - `make_window(scene, face, params, name=None)` → container `Group` (added to `scene.groups`; name defaults to `Window-001`…). `face` = a LOOSE face in `scene.mesh` without holes (it is consumed). `params` = dict, missing keys from `DEFAULTS`: `rows`/`cols` `"2"` or weights `"1,3,1"` (bottom→top, left→right); `frame_w`, `frame_h`, `mull_w`, `mull_h`, `frame_inset`, `frame_depth`, `glass_inset`, `glass_t` in metres; `frame_mat`, `glass_mat`; `opening` `"auto"|"reveal"|"hole"`; `wall_max` (m, auto: cut through only if the back face is this close). Raises `WindowizerError` (frame wider than the opening, bad counts, face with holes…).
  - `rebuild_window(scene, group, params)` — new parts in place (the wall opening is not recut). `delete_window(scene, group)` → bool (False = wall edited since, opening left open; the window goes anyway).
  - `window_data(group)` → `{"params", "outline", "pose"}` or None; `current_params(scene)`; `WindowizerCommand(mutate)` — one undo step restoring mesh, groups, children, ext, data and materials.
- Saved data: `group.ext['windowizer']` (params, local outline, pose) on each window; document data `scene.plugin_data['windowizer']` = `{"version": 2, "current": params, "openings": {uid: record}}` (the record closes the wall on Erase).
- Needs: nothing extra.
- Pitfalls:
  - NEVER call `windowize`, `edit` (modal QDialog) or `inherit`/`erase`/`_warn` (QMessageBox on any problem).
  - The `run_python` step only reverts the loose mesh and new groups; Edit/Erase (in-place children, group removal), the document data and the materials need `viewport.history.execute(wz.WindowizerCommand(mutate))` + check `history.last_error`. That makes 2 undo entries when the mesh also changed: undo twice to revert fully.
  - Run at top level (no group open). The host face is matched to the wall by `is_bounded` (its edges shared with one coplanar face): draw the window face INSIDE the wall face.

```python
import sys
wz = sys.modules['ingetrazo_plugin_windowizer']
face = next(f for f in selection if isinstance(f, Face))      # a rectangle drawn on a wall
out = {}
def mutate(sc):
    wz.ensure_default_materials(sc)
    out["g"] = wz.make_window(sc, face, {"rows": "1", "cols": "2", "frame_w": 0.05})
viewport.history.execute(wz.WindowizerCommand(mutate))
if viewport.history.last_error: raise RuntimeError(viewport.history.last_error)
print(out["g"].name, len(out["g"].children))                   # Window-001 3
```

### IngeTrazo Stair Maker with Railings — `make_stair`
1.1.0 · aashishzharbade-arch · GPL-3.0-or-later · Reviewed

*Live-tested in IngeTrazo 0.5.9.*

Parametric straight, straight-with-landing, L- and U-shaped stairs (treads, riser boards, landings) with optional glass or steel railings, built as one container group of box solids. **In the app:** Extensions ▸ IngeTrazo Stair maker V.1 (Create / Edit selected), a floating "stair" toolbar, and right-click ▸ Edit stair… on a stair.

**From MCP:** Full — the builder is pure module-level code; only the dialog is UI.

- Module: `ingetrazo_plugin_make_stair`
- Entry points (all lengths in **millimetres**, rotation in degrees; missing keys take `DEFAULTS`):
  - `resolve(values)` → `(params, risers, riser_mm, first_flight_risers)`; raises `ValueError` with a readable reason.
  - `make_group(values)` → new `Group` (not yet in the document): `kind` ∈ `TYPES` ("Straight", "Straight with landing", "L-shaped", "U-shaped"), `height`, `width`, `going`, `count_mode` ("Target riser"/"Riser count"), `target_riser`, `risers`, `landing_after` (0 = halfway), `landing_depth`, `turn` ("Left"/"Right"), `gap` (U), `nosing`, `thickness`, `closed`, `riser_thickness`, `top_landing`, `top_depth`, `origin_x/y/z`, `rotation`, `railing` ("None"/"Glass"/"Steel"), `rail_sides`, `rail_height`, … (see `DEFAULTS`). Add it with `groups.append(g)`.
  - `stair_parts(values)` / `geometry_parts(values)` → list of `(name, bounds_or_8_corners_mm, role)` — dry run, no geometry.
  - `build_children(values)` → list of child `Group`s in local metres (for rebuilding an existing stair).
  - `selected_stairs(scene)` → stair groups in the selection.
- Saved data: `group.ext["make_stair"] = {"version", "parameters"}` (full resolved params, mm); `group.ifc = {"class": "IfcStair"}`.
- Needs: nothing extra.
- Pitfalls: the first flight runs along local +X from the origin; placement lives in `g.xform` (children are local), so read world coordinates with `g.xform.map(p)`. Edit an existing stair with `StairCommand(g, build_children(p), p)` via `viewport.history.execute(...)` — a nested command (own undo step; check `viewport.history.last_error`). Setting `g.children` directly is NOT undone by the run_python transaction (it only reverts the loose mesh and newly added groups). Never call `StairDialog.apply` (QMessageBox on error).

```python
import sys; m = sys.modules['ingetrazo_plugin_make_stair']
p, n, r, first = m.resolve(dict(kind="L-shaped", railing="Steel", origin_x=12000.0))
g = m.make_group(p); groups.append(g)
print(n, round(r, 1), len(g.children))
```

### IngeTrazo Kitchen Maker — `ingetrazo_kitchen_maker`
1.0.2 · aashishzharbade-arch · GPL-3.0-or-later · Reviewed

*Live-tested in IngeTrazo 0.5.9.*

Parametric kitchen cabinets (base, wall, tall, high, blind corners, fillers; doors, drawers, glass, open shelves, worktop) as one container group of box parts each, plus a grouped cost report. **In the app:** Extensions ▸ IngeTrazo Kitchen Maker V.1 and its toolbar (Library, Add next, Edit, Open / close, Swap hinge, Finishes, Cost report), and right-click ▸ IngeTrazo Kitchen Maker V.1 on a cabinet.

**From MCP:** Full — builders, placement and report are pure functions; only the editor/report dialogs are UI.

- Module: `ingetrazo_plugin_ingetrazo_kitchen_maker`
- Entry points (recipe values in **mm**, model in metres; local +X = width, front faces local −Y, Z up from the floor):
  - `preset(family="Base", style="Double doors", width=600.)` → complete params dict (`family` ∈ `FAMILIES`, `style` ∈ `STYLES`, `finish` ∈ `FINISHES`, `top_finish` ∈ `TOPS`, `hinge`, `opening` °, `shelves`, `drawers`, `worktop`, `price`, `currency`, `name`, …). `catalog()` → the 48 library presets.
  - `validate(p)` raises `ValueError`; every `DEFAULT` key must be present with the right type (floats for lengths).
  - `make_group(p)` → new `Group` (identity `xform`, children = parts, not in the document). Place with `g.xform.translate(x, y, z)` (m) then `g.xform.rotate(deg, 0, 0, 1)`; add with `groups.append(g)`.
  - `adjacent_pose(reference, p, gap_mm=0.)` → `QMatrix4x4` beside `reference` along its +X; assign to `g.xform`.
  - `geometry(p)` → `[(name, 8 corners mm, QUADS, role)]` (dry run). `params_of(g)`, `selected(scene)`.
  - `report_rows(groups)` → rows per identical configuration (`REPORT_HEADERS`: … Qty, Currency, Unit price, Total); `export_csv(path, rows)` writes UTF-8 CSV (use /tmp or a mounted folder).
  - Edit existing cabinets: `UpdateCabinets([(g, new_p), …])` via `viewport.history.execute` (nested).
- Saved data: `group.ext["ingetrazo_kitchen_maker"] = {"version", "parameters"}`; `window._ingetrazo_kitchen_maker = (toolbar, dialogs, action)`.
- Needs: NumPy (bundled with the app) for the preview only.
- Pitfalls: never call `action("Finishes")` (QInputDialog) or any `action(...)` that can fail (QMessageBox), nor `Editor.apply` in "Click to place" mode (arms PlaceGroupTool and waits for a click). `UpdateCabinets` edits existing groups: as a nested command it is its own undo step; check `viewport.history.last_error`. Wall units: plinth 0 and worktop False, z set by you (≈1.5 m).

```python
import sys; m = sys.modules['ingetrazo_plugin_ingetrazo_kitchen_maker']
p = m.preset("Base", "Double doors", 600.0); p.update(price=250.0); m.validate(p)
a = m.make_group(p); a.xform.translate(12.0, 3.0, 0.0); groups.append(a)
b = m.make_group(p); b.xform = m.adjacent_pose(a, p, 0.0); groups.append(b)
print(len(a.children), m.report_rows([a, b]))
```

### IngeFloor · Wood floors — `floor_generator`
1.0 · Hris (Krzysztof) · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Generates parametric plank flooring, decking or parquet (11 patterns: running bond, herringbone 1–3x, chevron, basketweave, Versailles, mixed widths …) on a room face, with holes/columns, a mitered border, skirting boards and a material & cut report. **In the app:** Extensions ▸ Floor & Decking (Podłoga) ▸ Generate / Edit Floor… / Material & Cut Report…, a toolbar button, a tool (Ctrl+Shift+F), and the right-click menu on faces or a generated floor.

**From MCP:** Partial — `generate_floor_for_face` is pure and returns a ready `Group`; every UI path (create, edit, report) goes through modal dialogs.

- Module: `ingetrazo_plugin_floor_generator` (single file); Tool `viewport.window()._tools['plugin_floor_generator_FloorGeneratorTool']` (do not activate).
- Entry points:
  - `generate_floor_for_face(verts, params, normal=None, holes_3d=None)` → `(Group, plank_count, report)`. `verts` = outline as `QVector3D` list (or a `Face`, whose own holes are used), any winding; `normal` = side the planks grow to (`QVector3D`); `holes_3d` = list of `QVector3D` loops. `params = dict(DEFAULT_PARAMS)` with `pattern` (`PATTERN_KEYS`: pat_half, pat_third, pat_random, pat_straight, pat_herringbone, pat_herringbone_double, pat_herringbone_triple, pat_chevron, pat_basket, pat_versailles, pat_mixed_widths), `plank_w_cm`, `plank_l_cm` (0 = continuous), `thickness_mm`, `gap_mm`, `angle_deg`, `wood_type` (`WOOD_KEYS`: wood_oak/pine/walnut/ash/teak), `color_var_pct`, `use_components`, `use_border`, `border_planks`, `border_plank_w_cm`, `use_hole_border`, `use_skirting`, `skirting_holes`, `skirting_h_cm`, `skirting_th_mm`, `skirting_color` "wood"/"white", `lang` "en"/"pl". `report`: room_area_m2, holes_count, full/cut/border_planks_count, skirting_count, skirting_linear_m, waste_pct, recommended_order_m2, geometry_backend …
  - `_resolve_rooms_and_holes(faces)` → `[(outline, normal, holes)]` — groups coplanar faces, smaller faces inside become holes.
  - Edit an existing floor: `new, n, rep = _rebuild_geometry(g, params)`, then `viewport.history.execute(FloorCommand(lambda sc: _apply_rebuild(g, new, params, rep), targets=[g]))` (private; own undo step — check `viewport.history.last_error`).
- Saved data: `group.ext["floor_generator"]` (params, outline_3d, normal_3d, holes_3d, report); `scene.plugin_data["floor_generator"]` = last dialog params (set only by the UI path). `g.ifc = {"class": "IfcCovering"}`.
- Needs: manifold3d (ships with IngeTrazo); falls back to shapely or pure Python (no holes) — see `report["geometry_backend"]`.
- Pitfalls: `run_floor_generator`, the Tool, `show_material_report` and `_execute` use `QDialog.exec()` / `QMessageBox` — they stall the bridge; never call them. `rebuild_floor()` edits the group in place, which run_python's snapshot does not undo — use the `FloorCommand` route above. The floor group always carries an identity `xform`; whole planks are component instances in `g.children`.

```python
import sys; m = sys.modules['ingetrazo_plugin_floor_generator']
room = [QVector3D(10, 0, 0), QVector3D(14, 0, 0), QVector3D(14, 3, 0), QVector3D(10, 3, 0)]
col = [QVector3D(11.8, 1.3, 0), QVector3D(12.2, 1.3, 0), QVector3D(12.2, 1.7, 0), QVector3D(11.8, 1.7, 0)]
p = dict(m.DEFAULT_PARAMS, pattern="pat_herringbone", use_border=True, use_skirting=True)
g, n, rep = m.generate_floor_for_face(room, p, normal=QVector3D(0, 0, 1), holes_3d=[col])
groups.append(g); print(n, "planks,", round(rep["recommended_order_m2"], 2), "m2 to order")
```

### Roof Tiles — `roof_tile_tool`
1.2.1 · Pesi (pesi3d.de) · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Builds roofs on a footprint (20 shapes: gable, hip, half-hip, mansard, butterfly, dome …, with gable walls and a slab) and covers sloped faces with real 3D tiles (verge tiles or a flush verge finish, ridge/hip tiles, end caps, cut tiles, valley/hole flashing); whole tiles are instanced, now combined into "Tile block" components. **In the app:** Extensions ▸ Roof Tiles ▸ Create Roof… / Edit Roof Shape… / Cover Roof… / Edit Roof Tiles…, its PESI3D toolbar, and the right-click menu.

**From MCP:** Full — both builders are pure functions that return a ready `Group`; append it yourself.

- Module: `ingetrazo_plugin_roof_tile_tool` (single file)
- Entry points:
  - `make_shape(fp, p, name="Roof")` → `(Group, info)`. `fp = {"outline": [(x, y), …], "z0": z}` (m, footprint at wall-top height, any winding); `p` from `gen_default_params()`: `type` (`ROOF_TYPES` keys: gable, hip, halfhip, dutch, pyramid, mono, split, saltbox, mansard, gambrel, bonnet, butterfly, mroof, sawtooth, barrel, spire, dome, helm, aframe, flat), `pitch`/`pitch2` (°), `ratio`, `brk`, `offset`, `count`, `overhang`, `gable_ov`, `thick` (m), `walls`, `rotate`, `auto`/`gables` (edge indices), `eave` (-1 = longest). `info`: faces, walls, height, seconds. Lower level: `build_roof(outline, z0, rtype, p)` → `(roof_polys, wall_polys)`; `gather_footprint(scene)` → `fp` from selected horizontal faces / a building body.
  - `gather(scene)` → `{"faces": [{"outer": [(x,y,z)…], "holes": […]}], "shapes": [uids]}` from `scene.selection` (any object with `.selection`: faces or groups; a group's upward sloped faces).
  - `make_roof(data, p, name="Roof Tiles")` → `(Group, info)`; `p` from `default_params()`: `preset` (`PRESET_BY_ID`: frankfurt, harz, toscana, flat, marseille, biber, monk, pantile, plain_uk, spanish), `W`, `Lmin`, `Lmax` (m), `detail` low/medium/high, `color` (name in `COLORS`), `verge` mode (`VERGE_MODES`: tiles, mortar, board, metal, cut), `ridge`/`hips`/`mono`/`endcaps`/`flashing`/`holeflash`/`cutcomp` (blocks) bools, `endstyle`, gaps/overhangs (m), `offset`. `info`: tiles, cut, cut_shared, blocks, verge, ridge, hip, mono, gauge… `result_text(info)` = one-line summary; `estimate(data, p)` = fast count (`tiles`, `objects`, `area`, `edges`).
  - `materials_of(p)` → Material list (the command registers them in `scene.materials`; optional, faces carry `attrs["color"]`/`["mat"]`).
- Saved data: `group.ext["roof_tile_tool"]` (shape: `kind="roof_shape"`, params, outline, z0; covering: version, params, faces, shape uids). QSettings `plugins/roof_tile_tool/params` and `/shape` (dialog defaults only).
- Needs: numpy and manifold3d (both ship with IngeTrazo).
- Pitfalls: `show_*` open non-modal dialogs that use `QMessageBox` on errors and `QFileDialog` for custom tiles — never call them (`custom_from_scene(scene)` / `custom_from_file(path)` are the headless way to a custom tile). `run_roof`/`run_shape`/`recover_tiles` execute their own commands (nested undo steps) but raise on `history.last_error`. `tile_spec` uses `p["W"]/Lmin/Lmax`, not the preset's: copy them from `PRESET_BY_ID[id]` when changing preset. `RoofError` above 40 000 objects (`p["force"]=True`). Big roofs with `detail="high"` are slow (120 s limit). Whole tiles are components in `g.children` (single tiles and "Tile block"s).

```python
import sys; m = sys.modules['ingetrazo_plugin_roof_tile_tool']
fp = {"outline": [(10, 0), (14, 0), (14, 3), (10, 3)], "z0": 2.5}
sp = m.gen_default_params(); sp.update(type="gable", pitch=35.0, overhang=0.3)
shape, si = m.make_shape(fp, sp, name="Roof"); groups.append(shape)
class _Sel: selection = {shape}
tp = m.default_params(); tp.update(preset="frankfurt", detail="low")
tiles, info = m.make_roof(m.gather(_Sel()), tp, name="Roof Tiles")
groups.append(tiles); print(m.result_text(info))
```

### Levels (Niveles) — `niveles`
0.5.7 · José Castro Basso (FADU–UDELAR) · GPL-3.0-or-later · Reviewed

*Live-tested in IngeTrazo 0.5.9.*

Named building levels (PB, PA, Azotea…) with a height, kept in the document; in a parallel-projection elevation/section it draws a dashed guide per level and the cursor snaps to those heights. **In the app:** side-tray tab «Levels»/«Niveles» (list + Add/Edit/Delete + "Show elevation").

**From MCP:** Full — all state is one document-data value; read/write it through `ExtensionApp`.

- Module: `ingetrazo_plugin_niveles`
- Entry points:
  - `_state(app)` → `{"levels": [{"name": str, "z": float m}, …] sorted by z, "guides": bool}` (sanitised read).
  - `app.set_document_data({"levels": [{"name", "z"}…], "guides": True})` — writes levels (z in metres, absolute). The panel refreshes itself.
  - Elevation where guides show: `viewport.camera.set_view("front"); viewport.camera.perspective = False`.
  - `snap_to_levels(app, viewport, snap, px, py)` / `draw_guides(app, viewport, painter)` are viewport callbacks; not useful from a script.
- Saved data: `scene.plugin_data["niveles"]` = `{"levels": [...], "guides": bool}`.
- Needs: nothing extra.
- Pitfalls: Add/Edit open a modal `_LevelDialog` — never click/call `LevelsPanel._add/_edit`. `set_document_data` runs its own `SetPluginDataCommand`: from `run_python` it is its own undo step and is NOT rolled back if the script raises later (the outer snapshot ignores `plugin_data`); writing `scene.plugin_data` directly is not undoable at all. Guides/snap only when the camera is parallel and |pitch| ≤ 15°.

```python
from views.extension_api import ExtensionApp
import sys
m = sys.modules['ingetrazo_plugin_niveles']
app = ExtensionApp(viewport.window(), 'niveles')
app.set_document_data({"levels": [{"name": "PB", "z": 0.0}, {"name": "PA", "z": 2.8}], "guides": True})
print(m._state(app))
viewport.camera.set_view("front"); viewport.camera.perspective = False; viewport.update()
```

### Polygon properties — `propiedades_poligono`
0.5.7 · Rony Leonel Janampa Monago · MIT · Reviewed

*Live-tested in IngeTrazo 0.5.9.*

Area, perimeter, centroid, second moments (Ix, Iy, Ixy about centroid and model axes), principal moments + angle, radii of gyration and in-plane bbox of the selected coplanar faces (holes subtract). Read-only. **In the app:** Extensions ▸ Polygon properties, and right-click ▸ Polygon properties when faces are selected.

**From MCP:** Full — the maths and the selection reader are plain functions; only the report window is UI.

- Module: `ingetrazo_plugin_propiedades_poligono`
- Entry points:
  - `region_properties(outers, holes=())` — lists of 2D loops `[(u, v), …]` (m, loop not closed, any winding; outers add, holes subtract). Returns dict: `area` m², `perimeter` m, `centroid` (u, v), `Ix_centroid`/`Iy_centroid`/`Ixy_centroid` and `*_origin` m⁴, `I1`, `I2`, `theta` rad (I1 axis from u), `rx`, `ry` m, `bbox` (u0, v0, u1, v1), `vertices`. Raises `PolygonError` if area ≈ 0.
  - `polygon_properties(pts)` — one simple polygon, same dict.
  - `selection_properties(scene)` → `(props, (origin, u, v, n), (n_faces, n_holes))` for selected `Face`s of `scene.mesh`; `props["centroid_world"]` is a `QVector3D`. Raises `PolygonError` (nothing selected / not coplanar).
  - `format_report(props, plane, counts)` → the dialog's text, in the document's units.
- Saved data: none.
- Needs: nothing extra.
- Pitfalls: `show_properties(viewport)` opens a modal `QDialog.exec()` — never call it. Only faces of the active mesh (`scene.mesh`) count: faces inside a closed group are ignored unless that group is open for editing (else use `region_properties` with your own loops). Plane axes follow `core.axes.plane_axes(normal)` (u ≈ red axis). Selection changes are not undoable — restore it.

```python
import sys
m = sys.modules['ingetrazo_plugin_propiedades_poligono']
f = mesh.add_face([QVector3D(10,0,0), QVector3D(12,0,0), QVector3D(12,1,0), QVector3D(10,1,0)],
                  [[QVector3D(10.5,.25,0), QVector3D(10.5,.75,0), QVector3D(11,.75,0), QVector3D(11,.25,0)]])
selection.clear(); selection.add(f)
props, plane, counts = m.selection_properties(scene)
print(props["area"], props["centroid_world"], props["I1"], props["I2"])
print(m.format_report(props, plane, counts))
```

## 8. Terrain

### Topology (terrain) — `topology_tool`
1.2.1 · Pesi (pesi3d.de) · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Builds a terrain group from contour lines (regular grid, harmonic interpolation, optional quads), from points (Delaunay TIN) or from the spot heights of a vector site-plan PDF, with optional skirt, closed bottom, embedded contours and a flat contour map. **In the app:** Extensions ▸ Topology ▸ Terrain from Contours… / Terrain from Points… / Terrain from Plan… / Edit Terrain…, its own PESI3D toolbar, and the right-click menu (plus an Altitude Editor overlay inside the dialog).

**From MCP:** Full — `make_terrain` is pure and returns a ready `Group`; the PDF reader (`survey_pdf.read_pdf`) is headless too.

- Module: `ingetrazo_plugin_topology_tool` (package; submodules loaded lazily by relative import: `.survey_pdf`, `.pdf_vectors`, `.page_view` (Qt) — get them with `importlib.import_module('ingetrazo_plugin_topology_tool.survey_pdf')`)
- Entry points:
  - `make_terrain(data, p, name="Terrain")` → `(Group, (faces, seconds, "triangles"|"quads"))`. `data = {"mode": "contours", "segs": [(x0,y0,z0,x1,y1,z1), …]}` (m, at their heights; optional `"cset"`) or `{"mode": "points", "points": [(x,y,z), …], "source": {…}}` (≥ 3, not collinear). `p` from `default_params()`: `res_mode` "cells"/"size", `cells` (long side, 10–2000), `cell_size` (m), `boundary` "hull"/"box"/"cut", `peaks` "round"/"flat", `skirt`, `skirt_depth` (m below lowest point), `bottom`, `soft`, `quads`, `smooth_r` (m), `max_edge` (m, points), `contours`, `map`.
  - `gather_segments(scene)` / `gather_points(scene)` → source data from `scene.selection` (edges, faces, groups, guide points; points fall back to `scene.geo_points`). `gather_points` returns `(points, label)`.
  - `estimate_triangles(segs, p)` → `(n, (h, nx, ny))`; `terrain_from_contours(segs, p)` → `(V, T, C)`, `terrain_from_points(pts, p)` → `(V, T)` numpy arrays.
  - Clean-up: `cs = ContourSet.from_segs(segs); cs.set_clean(simplify_m, smooth_passes, min_len_m); segs = cs.segs()`.
  - Plan PDF: `survey_pdf.read_pdf(path, page_index=0)` → `{"recs", "scale" (1:n or None), "pages", "vec", "texts", "info"}` (paper points); then `recs = survey_from_pdf(d["recs"], scale)`, `survey_apply(recs, codes_on={})` → used count, `survey_to_local(recs, origin="min", keep_height=False)` → `(pts, (e0, n0, h0))` (lowest height at z 0), `dedupe_points(pts)`.
- Saved data: `group.ext["topology_tool"]` (`mode`, `version`, params, segs/lines or points + `source`); legacy keys `terrain_tool`, `topoform`. QSettings `plugins/topology_tool/params`, `plugins/topology_tool/survey_dir` (dialog only).
- Needs: numpy (ships with IngeTrazo); pypdf 6.19 vendored in `_vendor` (put on `sys.path` on first PDF use; shared with plan_import_tool, whichever loads first wins).
- Pitfalls: `show_contours/points/edit` open non-modal dialogs (`QMessageBox.warning` on errors); `show_survey(viewport, path=None)` opens a `QFileDialog` when no path is given — avoid all `show_*`. `run_terrain(viewport, data, p, old=None)` executes its own command (nested undo step) but raises on `history.last_error`. Raises `TopoError` if all contours share one height or > 1 000 000 triangles (`p["force"]=True`). Keep `cells` ≤ 300 for the 120 s limit. Spot heights only pair with a symbol within 12 mm of the text; codes like FFB/OK start unticked.

```python
import sys, math; m = sys.modules['ingetrazo_plugin_topology_tool']
segs = []
for r, z in ((6, 0.0), (4, 1.0), (2, 2.0)):
    P = [(20 + r*math.cos(a/36*2*math.pi), 10 + r*math.sin(a/36*2*math.pi), z) for a in range(36)]
    segs += [P[i] + P[(i+1) % 36] for i in range(36)]
p = m.default_params(); p.update(cells=40, skirt_depth=0.5)
g, (n, secs, unit) = m.make_terrain({"mode": "contours", "segs": segs}, p, name="Terrain")
groups.append(g); print(n, unit)
```

### Grading — `grading_tool`
1.1 · Pesi (pesi3d.de) · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Fills the open ground inside a closed loop of edges (which may lie at different heights) with a smooth triangulated terrain that meets every edge; inner loops become holes or fixed lines, open chains are fixed lines, guide points are spot heights. **In the app:** Extensions ▸ Grading ▸ Fill from Edges… / Edit Grading…, its own toolbar, and the right-click menu.

**From MCP:** Full — `make_grading` is pure and returns a ready `Group`; append it yourself.

- Module: `ingetrazo_plugin_grading_tool` (single file)
- Entry points:
  - `make_grading(data, p, name="Grading")` → `(Group, info)`. `data = {"segs": [(x0,y0,z0,x1,y1,z1), …], "points": [(x,y,z), …]}` (m). Segments are chained in plan; the largest closed loop is the outer contour. `p` from `default_params()`: `res_mode` "div"/"size", `div` (divisions on the long side, 4–250), `size` (m), `smooth` (0–100 %: straight slopes → rounded), `inner` "holes"/"lines", `soft`, `flip`. `info`: triangles, seconds, slope (max °), inner, lines, points, ignored.
  - `gather(obj)` → `data` from `obj.selection` (edges, groups of edges, guide points).
  - `classify(data, normalize_params(p))` → domain dict; `estimate_triangles(dom)` for a cheap count. `compute(data, p)` → `(V, T, info)` numpy arrays without IngeTrazo objects. `slope_text(deg)` formats the slope.
- Saved data: `group.ext["grading_tool"]` (version, params, segs, points). QSettings `plugins/grading_tool/params` (dialog defaults).
- Needs: numpy (ships with IngeTrazo).
- Pitfalls: `show_fill`/`show_edit` open a non-modal dialog that uses `QMessageBox` on errors — avoid them. `run_grading(viewport, data, p)` executes its own command (nested undo step) but raises on `history.last_error`. Raises `GradingError` without a closed loop (seen from above) or above 150 000 triangles (`force=True` overrides). Pure-Python Delaunay: keep `div` ≤ ~120 for the 120 s limit. The result is a single surface (no skirt/solid); smooth=100 can overshoot edge heights slightly.

```python
import sys; m = sys.modules['ingetrazo_plugin_grading_tool']
loop = lambda P: [P[i] + P[(i+1) % len(P)] for i in range(len(P))]
outer = [(10, 0, 0.0), (20, 0, 0.0), (20, 8, 1.0), (10, 8, 1.0)]
inner = [(14, 3, 1.5), (16, 3, 1.5), (16, 5, 1.5), (14, 5, 1.5)]   # platform → hole
p = m.default_params(); p.update(div=20, smooth=100.0)
g, info = m.make_grading({"segs": loop(outer) + loop(inner), "points": []}, p, name="Grading")
groups.append(g); print(info["triangles"], m.slope_text(info["slope"]))
```

## 9. Fabrication and export

### Cut List — `cutlist`
0.0.1 · Ronaldo Alves · GPL-3.0-or-later · GitHub, not catalogued (ronaldogalves/cutlist-ingetrazo @ 6a9166e)

*Live-tested in IngeTrazo 0.5.9.*

Builds a woodworking cut list from leaf groups/components with faces: measures each part's length × width × thickness, finds the board (core), face coverings and edge bands by material role, merges equal parts into lines with quantities, sectioned by board and thickness, flags odd parts, and exports it (CSV/text, .xlsx, clipboard) through user-named supplier profiles. Read-only on geometry. **In the app:** side-tray tab «Cut List» (Refresh, tag filter, merge toggle, Export…, click a line to highlight its parts), Extensions ▸ Cut List ▸ (Show panel / Materials… / Settings… / Export…), right-click ▸ «Cut List: part settings…».

**From MCP:** Full — extraction, grouping and the export engine (profiles → rows → bytes) are plain functions; only the settings/materials/export windows are dialogs, and a CSV can be written to /tmp without them.

- Module: `ingetrazo_plugin_cutlist` (+ `.host.extract`, `.host.store`, `.host.app`, `.model.parts`, `.model.grouping`, `.model.materials`, `.model.fields`, `.export.profile`, `.export.rows`, `.export.writers`). The `.export.*` modules load lazily: get them with `importlib.import_module('ingetrazo_plugin_cutlist.export.rows')`, not `sys.modules[...]`.
- Entry points:
  - `extract(scene, use_selection=True, excluded_tags=frozenset(), include_groups=True)` → `Extraction` (`parts` [RawPart], `groups` uid→Group, `scope` "selection"|"model", `materials`, `tags`, `notices`). Walks selected groups, else every visible group.
  - `read_part(raw, library, override=None)` → `Part` (`uid`, `name` (spaces tidied), `length`/`width`/`thickness` m, `core`, `face1`, `face2`, `edges`, `tag`, `grain`, `can_rotate`, `note`, `flags`, `excluded`). Material names = paint label: registry name, texture stem or `"#rrggbb"`.
  - `build(parts, tolerance=0.001, *, by_name=False)` → `CutList` (`sections[].material/thickness/lines`, line `core/length/width/qty/display_names()/uids`, `flagged`, `qty`).
  - Export: `profile.generic()` → `Profile` "Simple CSV" (`;`, `utf-8-sig`, CRLF, whole mm, `,` decimals, header Qty;Name;Length;Width;Thickness;Material;Band L1/L2/W1/W2;Note). `Profile(...)`/`Profile.from_dict(d)`/`dataclasses.replace` for others (`format` text|xlsx|clipboard, `separator`, `rows` merged|per_piece, `split` none|board|field:<name>, `filename` template, `columns` of `Column(header, kind=field|text|template, value)`, `material_codes`/`band_codes`).
  - `rows.make_rows(cut_list, {uid: Part}, library, profile, part_fields=None, context={"model": stem, "date": iso}, skip_boards=frozenset())` → `Export` (`files[]` `OutFile(name, header, rows of Cell)`, `problems[]` (missing codes/material), `ok`). `name` = safe file name from `profile.filename` (`{model}` → `cutlist_mcp_test.csv`; empty → `cut_list.csv`).
  - `writers.unencodable(file, profile)` → encoding problems; `writers.file_bytes(file, profile)` → bytes (CSV or .xlsx); `xlsx_bytes(file)`, `clipboard_text(files, profile)`. Write the bytes yourself: `open('/tmp/x/' + f.name, 'wb').write(...)`.
  - `Store(app)`: `.library()`, `.settings()`, `.overrides(groups)`, `.fields()` (custom fields; `fields.values(tag, own)`), `.profiles()` (user's saved profiles, generic if none), `.save_materials(...)`, `.clear_materials(names, remember=False)`, `.set_overrides({group: PartOverride(...).with_fields({...})})`.
  - Live controller: `viewport.window().extension_panels()['extension_cutlist'].widget()._controller` — `.refresh(ask=False)` then `.export_session().rows(profile)` gives the panel's exact export (overrides, "Group #n" names, custom fields).
- Saved data: `scene.plugin_data["cutlist"]` (model settings, materials, `fields`), `plugin_data["cutlist.numbers"]` (written without undo), `group.ext["cutlist"]` (part overrides incl. `fields`); user file `<user plugins dir>/../cutlist/defaults.json` (defaults, materials, fields, `export_profiles`, `last_export_profile`; `CUTLIST_USER_DIR` overrides the folder).
- Needs: Python ≥ 3.11 and NumPy.
- Pitfalls: never call `open_export` (modal ExportDialog, and `refresh(ask=True)` first), `open_materials`, `open_settings`, `open_part_settings_for`; `controller.write_files` asks with a QMessageBox if a file exists; `ExportDialog.export()` uses QFileDialog. Check `ex.problems` + `unencodable` before writing — the dialog refuses to export with any. `Store.save_profiles` and `remember=True` write the user's defaults file (no undo). Settings/materials/overrides writes are nested undo steps. With nothing selected the whole model is listed.

```python
import sys, importlib
ex = sys.modules['ingetrazo_plugin_cutlist.host.extract']
pa, gr, ma = (sys.modules[f'ingetrazo_plugin_cutlist.model.{n}'] for n in ('parts', 'grouping', 'materials'))
rw, wr, pf = (importlib.import_module(f'ingetrazo_plugin_cutlist.export.{n}') for n in ('rows', 'writers', 'profile'))
e = ex.extract(scene, use_selection=True); lib = ma.Library()
parts = [pa.read_part(r, lib) for r in e.parts]
cl, prof = gr.build(parts, 0.001), pf.generic()
xp = rw.make_rows(cl, {p.uid: p for p in parts}, lib, prof, context={"model": "job"})
assert not xp.problems and not any(wr.unencodable(f, prof) for f in xp.files)
for f in xp.files:
    open('/tmp/' + f.name, 'wb').write(wr.file_bytes(f, prof))   # /tmp/job.csv
print([f.name for f in xp.files], sum(len(f.rows) for f in xp.files))
```

### Woodworking Cut List (Lista de corte para marcenaria) — `lista_corte_marcenaria`
1.0.0 · Victor Jardim · GPL-3.0-or-later · Community (catalogue file `lista_corte.py`, installed here as `lista_corte_marcenaria.py`)

*Live-tested in IngeTrazo 0.5.9.*

Measures every visible group/component mesh in the model as one board (length × width × thickness in mm, largest first, in the part's own axes so rotated components are measured correctly), flags non-rectangular or odd-thickness parts, merges identical boards (0.1 mm) and exports a semicolon CSV with edge-banding counts, area and tape length. Read-only on the model. **In the app:** Extensions ▸ «Lista de corte — Marcenaria» (a modal table dialog; edge-band counts and names are edited in the table, not saved).

**From MCP:** Full — `collect_pieces`, `group_pieces` and `make_csv` are pure functions; only the dialog is modal.

**vs `cutlist`:** this is a small single-file, dialog-only tool: whole model only (no selection or tag filter), no materials (boards are not split by material, no face coverings), edge banding is just 0–2 long/short sides typed per line and never saved, CSV only (pt-BR headers, `,` decimals), nothing stored in the model. `cutlist` is material-aware (board/covering/band roles), works on the selection, keeps settings and per-part overrides in the document, has a side panel and supplier export profiles (CSV/xlsx/clipboard).

- Module: `ingetrazo_plugin_lista_corte_marcenaria`; tool `viewport.window()._tools['plugin_lista_corte_marcenaria_ListaCorteMarcenaria']`.
- Entry points:
  - `collect_pieces(scene)` → `(pieces, skipped)`: one `Piece(name, qty=1, length, width, thickness, issue)` (mm) per visible, non-billboard group mesh with faces, nested placements included (name `"Parent / Child"`, transforms composed); `issue` holds "REVISAR: …" notes (non-rectangular solid → bounding box used; thickness < 0.2 or > 80 mm; shear; unmeasurable). `skipped` = counts `empty`, `hidden`, `billboard`, `loose` (loose faces are never pieces).
  - `group_pieces(pieces)` → merged `Piece` list (dims rounded to 0.1 mm, names joined with " | ", qty summed; pieces with an issue never merge), sorted by thickness, length, width desc. `Piece.area` = qty × L × W in m².
  - `make_csv(path, rows)` writes UTF-8-BOM `;` CSV; `rows` = `(name, qty, length_mm, width_mm, thickness_mm, bands_long 0–2, bands_short 0–2, note)`; adds total area (m²) and tape length (m) columns; text cells guarded against formula injection (`_safe_csv`).
- Saved data: none.
- Needs: nothing extra.
- Pitfalls: activating the tool opens a modal `QDialog.exec()` (plus QFileDialog/QMessageBox on export) — never call `on_activate`/`_show_dialog`. Always the whole model: filter the returned pieces yourself (by name or by checking `scene.selection`). A group that has both its own faces and children yields a piece for itself and one per child. Materials are ignored. Write CSVs only where intended (e.g. /tmp).

```python
import sys
lc = sys.modules['ingetrazo_plugin_lista_corte_marcenaria']
pieces, skipped = lc.collect_pieces(scene)
rows = lc.group_pieces(pieces)
for r in rows:
    print(r.qty, r.name, r.length, r.width, r.thickness, r.issue)
lc.make_csv('/tmp/lista_de_corte.csv',
            [(r.name, r.qty, r.length, r.width, r.thickness, 2, 0, r.issue) for r in rows if not r.issue])
```

### STEP export — `stepexport`
0.3.0 · Nicolas Marc · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Exports groups (one part per top-level group/component, nested pieces folded in, world coordinates) and the loose geometry as STEP AP214 solids (`MANIFOLD_SOLID_BREP`, or open shells when not closed): planar faces with holes, faceted cylinders recognised as true `CYLINDRICAL_SURFACE`s (holes, rods, fillets), optional repair (T-junctions, small caps) and coplanar-facet merging, colours; a report (solids, open shells, volumes, cylinders found) before writing. **In the app:** Extensions ▸ Export to STEP… opens the «STEP» side-tray panel (hidden at start): content All/Selection, unit mm/cm/m, cylinders (soft edges / geometry), simplify tolerance, colours, author, «Analyse» and «Export…».

**From MCP:** Full — `collect_parts` → `write_step` → (optional) `reader.read_solids` are plain functions (numpy, no Qt); `write_step(path=None)` returns the text without writing.

- Module: `ingetrazo_plugin_stepexport` + `.panel`, `.collect`, `.writer`, `.model`, `.texts` (loaded by setup); `.reader` is only used by tests — get it with `importlib.import_module(P + '.reader')`.
- Entry points:
  - `collect.collect_parts(scene, selection_only=False)` → `[model.Part]` (visible, non-billboard top-level groups + one part for the visible loose faces; `True` = only selected groups / selected loose faces).
  - `model.Part(name, faces, color=None, soft_edges=None)`, `model.Face(outer, holes=[], src=None)` — plain `(x, y, z)` tuples in metres, any winding: build parts without the scene.
  - `writer.write_step(parts, path=None, scale=1000.0, unit='mm'|'cm'|'m', author='', filename='model.stp', cylinders=False, cyl_mode='soft'|'geometry', cyl_tol=1e-5, angle_max=20.0, repair_tol=None, simplify_tol=None)` → `(text, report)`; writes ASCII to `path` when given. `scale` must match `unit` (mm 1000, cm 100, m 1). The panel uses `repair_tol=1e-5` and `simplify_tol=<mm>/1000`. `report` = `{solids, shells, faces, cylinders, parts: [{name, shells: [{name, closed, volume (unit³), faces, facets, cylinders, rejected, problems}]}], notes}`.
  - `collect.format_report(report, unit='mm', lang='en'|'fr')` → the panel's text.
  - `reader.read_solids(text)` → `[{name, kind 'solid'|'shell', faces, volume, edge_ok, units, surfaces {PLANE, CYLINDRICAL_SURFACE}, radii}]` (verification).
- Saved data: QSettings `ingetrazo/stepexport`: `unit`, `cylinders`, `cyl_mode`, `colors`, `author`, `simplify`, `simplify_tol`, `last_dir`. No document data.
- Needs: NumPy (bundled with IngeTrazo).
- Pitfalls: read-only on the model, no undo step. Do not call `StepPanel.export()` (`QFileDialog`); `StepPanel.run(path)` shows `QMessageBox.warning` if writing fails and saves the panel options to QSettings. Colours: pass `color=None` on parts to drop them (the panel's checkbox does that). Part names are unique-ified (`Name (2)`); a part with several shells writes `Name (1)`, `Name (2)`….

```python
import importlib
P = 'ingetrazo_plugin_stepexport'
col, wr = (importlib.import_module(f'{P}.{n}') for n in ('collect', 'writer'))
parts = col.collect_parts(scene, selection_only=True)
text, rep = wr.write_step(parts, path='/tmp/model.stp', scale=1000.0, unit='mm',
                          cylinders=True, cyl_mode='soft', repair_tol=1e-5)
print(col.format_report(rep, 'mm', 'en'))
```

### 3D PDF export — `igz_pdf3d`
1.0.1 · David Songer · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Exports the selection (groups whole, loose faces as one «Loose geometry» node per layer) or, with nothing selected, every visible group and loose face as a one-page PDF 1.7 with an interactive U3D model (tree by layer/object, materials and textures, the exported view plus Iso/Top/Front/Back/Left/Right) and a viewport snapshot for viewers without 3D. Pure-Python writer, no dependencies. **In the app:** «3D PDF» toolbar button and Extensions ▸ Export 3D PDF… (a non-modal dialog: title, note, units, page size, Save as, Export).

**From MCP:** Full — the dialog's pipeline (`collect` → `write_u3d` → `make_views` → `write_pdf`) is plain functions returning bytes; write them to a path yourself.

- Module: `ingetrazo_plugin_igz_pdf3d` (+ `._core`; live `Session` in `._core._SESSIONS`) + `._collect`, `._u3d`, `._views`, `._pdf` — these load lazily (via `._dialog`, when the dialog first opens): get them with `importlib.import_module(P + '._collect')`, not `sys.modules`.
- Entry points:
  - `_collect.collect(scene, units, viewport=None, poster_size=None)` → plain dict (`meshes`, `materials`, `textures`, `layers`, `camera`, `bbox` in U3D space (Y-up, recentred), `counts {groups, faces}`, `fallbacks`, `poster`). `units` `'in'` (1 unit = 1 inch, default in the UI) or `'m'`. `viewport` gives the "Exported view" camera; `poster_size=(width_px, None)` also renders a JPEG snapshot of the viewport (the dialog uses 1600). Raises `_collect.NothingToExport` (only edges selected / nothing visible).
  - `_collect.resolve(scene)` → `(groups, faces, whole)`; `describe_selection(scene)` → text.
  - `_u3d.write_u3d(data)` → U3D bytes (starts `b'U3D\x00'`).
  - `_views.make_views(data['camera'], data['bbox'])` → list of 3D view dicts.
  - `_pdf.write_pdf(options, u3d_bytes, views, poster=None, poster_size=None)` → PDF bytes. `options` = `{title, note, date 'YYYY-MM-DD', units, units_label ('inches'|'metres'), page 'letter'|'a4'|'tabloid'|'archd'}` (landscape).
- Saved data: none (no document data, no settings).
- Needs: nothing extra. Interactive 3D only in Adobe Acrobat Reader / Bluebeam Revu; others show the snapshot.
- Pitfalls: read-only on the model, no undo step. `collect` uses `scene.selection` — set it (and restore it) to choose what is exported. The dialog's Browse… is a `QFileDialog`; its export runs in a worker thread and writes to the "Save as" field (default `~/Untitled model.pdf` or the document's name) — do not drive the dialog. The poster calls `viewport.render_image` (slow on big models; skip with `poster_size=None`). Edges and billboards are not exported.

```python
import importlib
P = 'ingetrazo_plugin_igz_pdf3d'
C, U3, VW, PDF = (importlib.import_module(f'{P}.{n}') for n in ('_collect', '_u3d', '_views', '_pdf'))
data = C.collect(scene, 'm', viewport, (1600, None))          # current selection, or all visible
pdf = PDF.write_pdf({'title': 'Model', 'note': '', 'date': '2026-10-20', 'units': 'm',
                     'units_label': 'metres', 'page': 'a4'},
                    U3.write_u3d(data), VW.make_views(data['camera'], data['bbox']),
                    data['poster'], data['poster_size'])
open('/tmp/model.pdf', 'wb').write(pdf)
print(data['counts'], len(pdf))
```

## 10. Viewing, rendering and organising

### Point Cloud & 3D Gaussian Splatting — `pointcloud_splat`
1.0.0 · archimades27 · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Loads point clouds (.ply, .las, .xyz/.pts/.txt/.csv; not .laz) and 3D Gaussian splats (.ply with f_dc/opacity/scale fields, .splat), draws them as a viewport overlay, snaps tools to points, and offers Scan-to-BIM helpers (RANSAC plane, bounding box, guide points). **In the app:** side-tray tab "Point Clouds & Splats", Extensions ▸ Point Cloud & Gaussian Splatting… (Ctrl+Shift+P); opening a .ply/.splat/.xyz/.pts/.las/.laz goes to it.

**From MCP:** Partial — parsers and Scan-to-BIM helpers are plain functions; clouds are panel session state (not in the document), and display, clipping box and transforms are panel UI.

- Module: `ingetrazo_plugin_pointcloud_splat` (+ `.parsers`, `.data_models`, `.cad_tools`, `.scene_proxy`, `.renderer`); panel = `viewport.window()._pointcloud_panel`.
- Entry points:
  - `parsers.read_ply(path)` → `PointCloudDataset` or `GaussianSplatDataset` (decided by the field names); also `read_las`, `read_xyz`, `read_splat`, `generate_demo_pointcloud()`. Datasets hold numpy `coords` (N,3), `colors` (N,4 uint8), splats also `scales`, `rotations`, `opacities`; transform fields `offset`, `rotation_[xyz]_deg`, `scale` (about `pivot`); `.bounds` = world (min, max).
  - `panel.add_dataset(ds)` → shows it, adds an invisible proxy group "Point Cloud: <name>" (`ext {"type": "pointcloud_proxy"}`), selects it, zooms all viewports. After changing `offset`/rotations call `scene_proxy.sync_proxy_from_dataset(ds)` (the proxy's corners define the transform).
  - `cad_tools.create_plane_surface_group(ds, scene, name=None)` → Group (one face on the RANSAC dominant plane, ±6 cm) or None; `create_bounding_box_group(ds, scene, name=None)` → Group; `convert_to_guide_points(ds, scene, max_points=120)` → count. All take world coords and append to the scene directly.
- Saved data: none — datasets vanish on restart/New/Open; only the proxy box groups are saved (opacity 0, left behind).
- Needs: numpy. The accelerator is a macOS arm64 `fast_splat.dylib`; on Linux it fails to load (logged) and numpy fallbacks run. Rebuilding `fast_splat.c` needs Apple GCD/blocks (`dispatch/dispatch.h`): gcc cannot compile it, and the loader tries the .dylib first and stops there, so a Linux `.so` would only load if the .dylib were removed. Fallback: point clouds = vectorised numpy, OK up to ~1 M points; splats = pure-Python per-pixel loop in the paint path (UI and screenshots freeze for seconds). **Show splats as points instead** (example).
- Pitfalls: `panel.load_file` and the Import buttons show QMessageBox/QFileDialog on error → use the parsers. Undo removes the proxy/helper groups but not the panel dataset, nor guide points (`scene.guides` is not snapshotted). The file must be readable by the app (in a container: a mounted folder). Spans >1000 units are auto-scaled ×0.001. Photogrammetry output (COLMAP, OpenSplat) is in arbitrary units and orientation: set `ds.scale` from a known real dimension, and crop splat outliers by percentile.

```python
import sys, numpy as np
PR = sys.modules['ingetrazo_plugin_pointcloud_splat.parsers']
DM = sys.modules['ingetrazo_plugin_pointcloud_splat.data_models']
ds = PR.read_ply('/projects/scan_splat.ply')
if type(ds).__name__ == 'GaussianSplatDataset':          # splat -> plain points
    lo, hi = np.percentile(ds.coords, [2, 98], axis=0)
    k = (ds.opacities > 0.3) & np.all((ds.coords >= lo) & (ds.coords <= hi), axis=1)
    c = ds.colors[k].copy(); c[:, 3] = 255
    ds = DM.PointCloudDataset(name=ds.name, filepath=ds.filepath,
                              coords=np.ascontiguousarray(ds.coords[k]), colors=c)
viewport.window()._pointcloud_panel.add_dataset(ds)
print(ds.count, ds.bounds)
```

### JA Sun · Sun and shadows — `ja_sun`
0.1.0 · Julio Angulo · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Solar-study panel over the app's native sun/shadows: date, time, darkness, UTC offset, site (lat/lon/north), sun-path chart, day playback, named presets and "save as scene". **In the app:** side-tray tab «JA Sun», Extensions ▸ «JA Sun · Sol y sombras…» (Ctrl+Alt+S) and a one-button «JA Sun» toolbar.

**From MCP:** Full — a reachable `Controller` plus pure solar maths; every edit is a host `Command`.

- Module: `ingetrazo_plugin_ja_sun` (+ `.model`, `.controller`, `.ui`)
- Entry points:
  - `ctrl = viewport.window()._ja_sun_controllers['ja_sun']` (the live Controller).
  - `ctrl.change(patch)` — patch keys `enabled` bool, `month`/`day`/`hour`/`minute` int (local time, current year), `darkness` 0..1, `utc_offset` int −12..14 or None (= from longitude). Validates, executes `SetShadows`, refreshes panel + native shadows; raises `ValueError`/`RuntimeError`. No-op (no command) if nothing changes.
  - `ctrl.execute(model.SetLocation(lat°, lon°, north°))` — georef site (lat −80..84, lon −180..180); `ctrl.save_preset("name")`, `ctrl.load_preset("name")`, `ctrl.delete_preset("name")`, `ctrl.save_scene()` (adds a SavedView named «Sol dd-mm · hh:mm»).
  - `model.solar_details(scene)` → dict `elevation`, `azimuth` (deg), `sunrise`/`sunset`/`noon` (minutes after local midnight or None), `daylight`, `all_day`, `offset`, `ratio` (shadow length per unit height, None at night), `path`. `model.site(scene)` → (lat, lon, north). `model.path_for(lat, lon, month, day, offset, year)` → 10-min samples `(minute, elevation, azimuth)`.
- Saved data: shadows in `scene.shadows` (native), site in `scene.georef`; presets in `scene.plugin_data["ja_sun"]["presets"]` = {name: shadow dict}.
- Needs: nothing extra.
- Pitfalls: `ctrl.save_preset()` WITHOUT a name opens a modal `QInputDialog` — always pass the name. Every edit is its own nested undo step and is not rolled back if the script raises afterwards; `model.apply_shadow()` directly is not undoable at all. Panel strings and errors are Spanish. A running playback (`ctrl.playback`) is stopped by any edit.

```python
import sys
md = sys.modules['ingetrazo_plugin_ja_sun.model']
ctrl = viewport.window()._ja_sun_controllers['ja_sun']
ctrl.change({"enabled": True, "month": 6, "day": 21, "hour": 15, "minute": 0})
d = md.solar_details(scene)
print(round(d["elevation"], 1), round(d["azimuth"], 1), d["sunrise"], d["sunset"], md.site(scene))
```

### igz_sunpath · Sun Path — `igz_sunpath`
1.0.0 · Ezequiel M. Rezende · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Solar-study toolbar: toggles the app's shadows, draws a 2D sun arrow, a 3D sun vector and a solar dome (day paths for the 15th of each month + the current date, analemmas, hour labels, compass, hover cells that set the date/time) over the viewport, picks the dome centre, sets date/time and site, and exports the view as PNG. Draws only — it creates no geometry. **In the app:** «Sunpath» toolbar (object name `igz_sunpath`: Shadows on/off, Sun Arrow, Sun Vector, Sun Dome, Set Sun Tool, Set Center, Calendar, Set Location, Save PNG); Tab toggles Jan–Jun/Jul–Dec cells while the dome is on.

**From MCP:** Partial — the solar maths and dome geometry are callable and the overlays can be switched without dialogs, but there are no public entry points (everything is `_private`) and the output is an overlay, not model data.

- Module: `ingetrazo_plugin_igz_sunpath` (just wraps `setup`); the code is in `ingetrazo_plugin_igz_sunpath.igz_sunpath` (call it `m`).
- Entry points:
  - `m._solar_position(sh, latitude_deg)` → `(azimuth°, elevation°)`, azimuth clockwise from north; `sh` = anything with `month, day, hour, minute` (normally `scene.shadows`). Simple declination + equation-of-time formula; ignores longitude and time zone (clock = local solar time); within ~1° of `core.sun.solar_position(lat, 0, utc)`. For exact positions use `core.sun.solar_position(lat, lon, when_utc)` / `sun_direction(...)` instead.
  - `m._build_dome_cache(radius_m, lat_deg, month, day)` → dict `meridians` (each `pts` [(x,y,z) m], `marcados` hour points, `nascente`/`poente` sunrise/sunset, `label`, `month`, `day`, `special`), `analemas`, `cells`, `radius`, `lat`. Centred on `m._DOME_CENTER_XYZ` (QVector3D or None = origin), rotated by `m._NORTH_ROTATION_DEG`. Cached: call `m._invalidate_dome_cache()` after changing globals. Points can be turned into real edges with `mesh.add_edge` if a 3D dome is wanted.
  - Overlay switches (what the toolbar does, no dialogs): `for a, en, _ in m._LABELED_ACTIONS: if en == 'Sun Dome': a.setChecked(True)` ('Sun Arrow', 'Sun Vector', 'Sun Dome'). Dome size: `m._DOME_DIAGRAM_SCALE` (0.2–20, radius = 5 m × scale); centre: set `m._DOME_CENTER_XYZ = QVector3D(x, y, z)`, then `m._save_state_to_scene(mw)`, `m._invalidate_dome_cache()`, `m._repaint(mw)`.
  - Date/time: set `scene.shadows.month/day/hour/minute` (core `ShadowSettings`) and `enabled = True`; the plugin polls it every 100 ms.
  - `m._capture_viewport_to_png(mw, path)` → bool: viewport + overlay to PNG (transparent background if `m._EXPORT_TRANSPARENT_BG`).
- Saved data: plain attributes on the Scene object (`scene.igz_dome_active`, `igz_sun_vector_active`, `igz_overlay_visible`, `igz_dome_diagram_scale`, `igz_dome_show_*`, `igz_export_transparent`, `igz_dome_center`) — not plugin_data, so not saved in the .igz and not undoable. Location from `scene.georef` (fallback Belo Horizonte −19.92, −43.93); north from `scene.georef.north`.
- Needs: nothing for the maths. Set Location clicks the native base-map panel's Find (geocoding/tiles: network).
- Pitfalls: Calendar (`_trigger_calendar`), Save PNG (`_export_screenshot`: QDialog + QFileDialog + QMessageBox) and setup's error path are modal. Set Sun Tool opens a non-modal dialog (`m._DIALOG`); Set Center waits for a viewport click. Don't press the Shadows toggle from a script (it flips `scene.shadows.enabled` and also triggers the first menu action whose text contains "shadow"/"sombra"); set `scene.shadows.enabled` instead. `_change_location` writes `georef.lat/lon` in place (no undo, UTM offsets of the datum not recomputed). Module globals and scene attributes survive undo — restore them yourself. The overlay is a child widget of the viewport; whether the bridge's `screenshot` shows it is untested.

```python
import sys, math
m = sys.modules['ingetrazo_plugin_igz_sunpath.igz_sunpath']
sh = scene.shadows; sh.month, sh.day, sh.hour, sh.minute, sh.enabled = 12, 21, 15, 0, True
lat = getattr(scene.georef, 'lat', m._LATITUDE_DEG)
az, el = m._solar_position(sh, lat)
print(f'sun az {az:.1f} el {el:.1f}')
for a, en, _ in m._LABELED_ACTIONS:          # show the 3D sun vector overlay
    if en == 'Sun Vector':
        a.setChecked(True)
```

### Styles, Shadows and Map toolbars — `igz_tb_toolbar`
1.5.2 · Ezequiel M. Rezende · GPL-3.0-or-later · Community (catalogue id `igz_toolbars`)

*Live-tested in IngeTrazo 0.5.9.*

Three toolbars: «Styles» (13 buttons that trigger the native Camera ▸ Style actions), «Shadows» (on/off, date and time sliders with sunrise/sunset marks, calendar, intensity) and «Map» (tile source + location box that drives the Terrain panel's "Find location"). **In the app:** toolbars only (no menu, panel or context entry).

**From MCP:** Partial — the helpers are thin shortcuts over native state; everything they do is reachable more safely with core APIs, and the Map/calendar paths open dialogs.

- Module: `ingetrazo_plugin_igz_tb_toolbar`; submodules are NOT under that name: `sys.modules['igz_tb_style_igz_tb_style']` and `sys.modules['igz_tb_style_igz_tb_shadows']` (also `pkg._STYLE`, `pkg._SHADOWS`).
- Entry points (`mw = viewport.window()`):
  - Style: `_STYLE._trigger_action(mw, english)` → bool; `english` from `_STYLE.MENU_ACTIONS` ("Default", "Architectural", "Shaded", "Hidden line", "Monochrome", "Wireframe", "X-ray", "Toggle X-ray", "Edges", "Profiles", "Back edges", "Hidden Objects", "Hidden Geometry"). Native equivalent: `from core.style import style_by_name; mw._apply_display_style(style_by_name("Wireframe"))`.
  - Shadows (write `scene.shadows` directly): `_SHADOWS._toggle_shadows(mw, bool)`, `_change_date(mw, day_of_year)` (1–366, also enables), `_change_time(mw, minutes_after_midnight)` (also enables), `_change_intensity(mw, 0..100)` (→ darkness 0..1).
  - Pure: `_SHADOWS._sun_times(lat°, lon°, datetime.date, utc_offset_h)` → `(sunrise, sunset)` as `datetime.time` (NOAA approximation).
- Saved data: none of its own (changes native `scene.display_style` / `scene.shadows`, saved in the .igz).
- Needs: Map: network tiles via the native Terrain panel.
- Pitfalls: nothing is undoable or marks the document dirty (direct attribute writes / native style actions). `_open_native_georef_dialog` clicks the Terrain panel's Find button (a modal dialog) and `_make_calendar_button`'s `ShadowsCalendarDialog.exec()` is modal — never call them. `_change_location` is effectively a no-op (ShadowSettings has no lat/lon; use JA Sun or `scene.georef`). Sunrise marks default to lat −19.92, lon −43.93, UTC−3 when unset. Text-matching of native actions depends on translations.

```python
import sys
st = sys.modules['igz_tb_style_igz_tb_style']; sh = sys.modules['igz_tb_style_igz_tb_shadows']
mw = viewport.window()
print(st._trigger_action(mw, "Wireframe"), scene.display_style.face_mode)
sh._change_time(mw, 15 * 60); sh._change_intensity(mw, 40)
import datetime; print(sh._sun_times(40.4, -3.7, datetime.date(2026, 6, 21), 2))
```

### Scenes (scene manager) — `igz_tb_scenes`
1.0.0 · Ezequiel M. Rezende · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Camera keyframes ("scenes": eye, target, fov, hold and transition seconds) kept in the document, played as a smooth fly-through (Linear / Smooth / Catmull-Rom) and exported as PNG frames or MP4 (FFmpeg). Separate from the native saved views. **In the app:** «Scenes» toolbar (add, remove, clear, play, stop, timings, export, FFmpeg) and a «Scenes» side-tray panel (dock `igz_tb_scenes_dock`).

**From MCP:** Partial — the scene list, persistence and camera sampling are scriptable; play/export/timing go through dialogs (render frames yourself instead).

- Module: `ingetrazo_plugin_igz_tb_scenes`; the code lives in `sys.modules['igz_scenes_igz_tb_scenes']` (= `pkg._SCENES`).
- Entry points (`sm = sys.modules['igz_scenes_igz_tb_scenes']`; `ctrl = viewport.window().findChildren(sm.ScenesPanel)[0].controller`):
  - `ctrl.cenas` — list of `sm.Scene(nome, eye[x,y,z] m, target[x,y,z] m, fov°, hold_s=1.0, transicao_s=2.0)`; `ctrl.fps`, `ctrl.suavizacao` ("Linear"|"Smooth"|"Catmull-Rom"), `ctrl.loop`.
  - `ctrl.adicionar_cena_atual()` — append the current camera as "Scene N" and save to the document. After editing `ctrl.cenas` yourself call `ctrl.salvar_no_documento()` (else the next document change reloads the old list).
  - `ctrl.aplicar_camera(eye, target, fov=None)` — move the viewport camera (`look_from`).
  - `sm.sequence_duration(cenas)` → seconds; `sm.sample_sequence(cenas, t_s, smoothing)` → `(eye, target, fov)` at time t. Frames: loop t over `n = round(duration*fps)`, `ctrl.aplicar_camera(*sample)`, `viewport.render_image(w, h).save(path)`.
- Saved data: `scene.plugin_data["igz_tb_scenes"]` = `{"cenas": [Scene.to_dict()], fps, suavizacao, loop, transicao_padrao_s, hold_padrao_s, export_w, export_h}`; defaults in `<AppData>/igz_tb_scenes/config.json`.
- Needs: FFmpeg only for MP4 (`DependencyDialog` can download it — network).
- Pitfalls: `limpar_cenas()` (QMessageBox.question), `remover_cena_selecionada()` (message box or chooser dialog unless `ctrl.indice_selecionado` is valid), `tocar_animacao()` with no scenes, `configurar_tempos()`, `exportar_animacao()`, `_render()` (QFileDialog/QMessageBox) all block — remove with `del ctrl.cenas[i]` + `salvar_no_documento()`. Each save is its own nested undo step. Playback runs on a QTimer after the call returns. Loading a document without scene data keeps the previous list in memory.

```python
import sys
sm = sys.modules['igz_scenes_igz_tb_scenes']
ctrl = viewport.window().findChildren(sm.ScenesPanel)[0].controller
ctrl.cenas.append(sm.Scene("Front", [11, -8, 3], [11, 0, 1], 40.0, 1.0, 2.0))
ctrl.adicionar_cena_atual()          # current camera as the next keyframe, saved
print(sm.sequence_duration(ctrl.cenas), sm.sample_sequence(ctrl.cenas, 1.5, "Smooth"))
```

### Scene Automation — `scenes_play`
0.2.1 · G. Andreotti · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Plays the model's scenes (saved views) in sequence with eased camera moves (per scene: start/stop speed, travel and stay seconds, optional sound), loops, full screen with a floating player bar (pause = Look Around, Esc stops), MP4 export, and automations saved to a .json or kept in the model. **In the app:** Extensions ▸ Scene Automation… (non-modal window `scenes_play_dialog`).

**From MCP:** Full — the automation model, timing and camera interpolation are pure Python, the camera adapter and frame renderer are plain functions; only the window, file pickers and the MP4 progress use dialogs. Sounds and MP4 need FFmpeg.

- Module: `ingetrazo_plugin_scenes_play` (`show_dialog(app)`, `_STATE["dialog"]`) + `.model`, `.motion`, `.player` (pure), `.host` (IngeTrazo adapter), `.media` (ffmpeg/ffprobe/ffplay), `.export`, `.ui_dialog`, `.ui_player` (UI, loaded lazily — use `importlib.import_module('ingetrazo_plugin_scenes_play.<sub>')`).
- Entry points:
  - `model.Entry(scene, enabled=True, start="medium", stop="medium", travel=3.0, wait=5.0, audio=None, audio_length=0.0)` — scene by NAME; start/stop ∈ "slow"|"medium"|"fast"; seconds (travel ≤ 600, wait ≤ 3600). `model.Automation(entries, loops=1)` (0 = continuous): `.played(names)`, `.single_runtime(names)`, `.total_runtime(names)` (s; None if continuous), `.to_dict()` / `Automation.from_dict(d)` (validates, raises `AutomationError`).
  - `host.scene_names(scene)`, `host.find_view(scene, name)`, `host.build_steps(scene, entries)` → `[player.Step]` (missing scenes/sound files dropped).
  - `player.Sequencer(steps, loops, camera, on_change=None, audio=None)`: `.start()`, `.advance(dt_s)`, `.pause()/.resume()/.stop()`, `.phase` idle|travel|wait|done, `.time`. `player.timeline(steps, loops, start_pose)` → `(duration_s, [(time, file, offset)])`. `player.NullCamera(pose)` for dry runs.
  - `motion.Pose(target_xyz, distance, yaw, pitch, fov)`; `motion.Move.build(a, b, travel, start, stop).pose_at(t)`; `motion.interpolate(a, b, s)`.
  - `host.Camera(window)`: `.pose()`, `.set_pose(pose)`, `.arrive(step)` (recalls the scene like a double-click in the Scenes panel: layers, style, sections too). `host.show_scene(window, view)`.
  - `host.render_frame(viewport, width, height)` → QImage RGB32 (exact size, no overlays) — frames can be saved as PNGs to /tmp without FFmpeg.
  - Real-time playback (async, after the script returns): `importlib.import_module(P + '.ui_player').PlaybackController(window, "Scene Automation").play(steps, loops, full_screen=False, sound=None)`; stop with `.stop()`.
- Saved data: `scene.plugin_data["scenes_play"]` = `{"version": 1, "name", "automation": Automation.to_dict()}` (the window's Apply; one undo step). QSettings `plugins/scenes_play/{full_screen, geometry, last_dir, ffmpeg_dir}`. Automation files: JSON `{"format": "scenes_play.automation", "version": 2, ...}`.
- Needs: FFmpeg (ffmpeg, ffprobe, ffplay) on PATH or in the folder from QSettings — only for sounds (`media.probe_length`, `SoundPlayer`) and MP4 (`export.export_video` / `media.Encoder`). Never start them from a test.
- Pitfalls: `export.export_video` (ExportDialog.exec, QFileDialog, QProgressDialog, QMessageBox), `pick_ffmpeg_dir`, the window's Save/Load/attach sound, and `_error`/close-with-unsaved-changes are modal. `show_dialog` is non-modal but its Play with no valid scene pops a QMessageBox. Playback moves the live camera, may switch to Look Around and full screen and installs an app-wide Esc filter. Steps hold SavedView objects: they stop if the scene list changes. Adding scenes from a script: `scene.saved_views.append(SavedView(name, target=(x, y, z), distance, yaw, pitch))` (names must be unique).

```python
import importlib
P = 'ingetrazo_plugin_scenes_play'
md, host, pl = (importlib.import_module(f'{P}.{n}') for n in ('model', 'host', 'player'))
names = host.scene_names(scene)
auto = md.Automation([md.Entry(n, travel=3.0, wait=2.0) for n in names[:3]], loops=1)
steps = host.build_steps(scene, auto.played(names))
cam = host.Camera(viewport.window()); seq = pl.Sequencer(steps, 1, cam); seq.start()
for k in range(int(auto.total_runtime(names) * 2)):      # 2 frames per second to /tmp
    seq.advance(0.5)
    host.render_frame(viewport, 640, 360).save(f'/tmp/sp_{k:03d}.png')
```

### Automation (Automação) — `automacao`
1.0.0 · Daniel Wagner · MIT · Community

*Live-tested in IngeTrazo 0.5.9.*

Per-component motion preview: each component gets ordered blocks of rotation (axis by X/Y/Z or two picked points, rpm, direction, angle per cycle, accumulate/ping-pong) or linear motion (speed, distance, direction, one-way/ping-pong/accumulate), with repetitions and a delay before each cycle; Play/reverse/step/stop animate all configured components. The animation is drawn from GPU copies only — the model's transforms are never changed. **In the app:** «Automação» toolbar (gear, object name `automacao_toolbar`), Extensions ▸ Automação… (non-modal window `automacao_dialog`, UI in Portuguese/Spanish/English).

**From MCP:** Partial — the configuration is plain document data and the timeline/pose maths can be run from a script; the preview itself is viewport-only (visible in `screenshot`, not in `query_model`), and there is no export.

- Module: `ingetrazo_plugin_automacao` (single `__init__.py`); live controller `viewport.window()._motor_controller` (`MotorController`).
- Entry points:
  - Configuration (write it directly): `ExtensionApp(window, 'automacao').set_document_data({"version": 3, "motors": {component.uid: {"blocks": [block, ...]}}})`. Start each block from `MotorController._new_block("rotation"|"linear")` (static) and set: `axis_a`, `axis_b` — points in the component's LOCAL coordinates (metres; mapped by `group.xform`); rotation: `rpm`, `direction` 1|-1, `angle_degrees` per cycle (360), `rotation_mode` "accumulate"|"ping_pong"; linear: `speed_mm_s`, `distance_mm` (along a→b), `linear_direction` 1|-1, `linear_mode` "one_way"|"ping_pong"|"accumulate"; both: `repetitions` (0 = forever), `delay_s` (pause before every cycle). Blocks of one component compose as a matrix product in block order (M = B1·B2·…, so the last block acts on the points first); rotation axes are world lines through the mapped a→b.
  - `ctl._configured()` → `[{"group", "blocks": [runtime]}]` for components in `scene.groups` (only `group.is_component()`).
  - `ctl._advance(dt_s, redraw=False)` steps `ctl._motors` (runtime fields `angle` °, `phase` 0–1 per length, `done`); returns False when all are done. Pose of a component: `_rotation(a, b, angle)` (module function, a/b world QVector3D) and `translate((b - a) * phase)`.
  - Preview without the window: `ctl._ensure_preview()` (True if something is configured), `ctl._seek(t_s)` (exact time), `ctl.play_forward()` / `play_reverse()` / `pause()` (async QTimer), `ctl.stop()`.
- Saved data: `scene.plugin_data["automacao"]` as above (older v1/v2 layouts are still read). Components are referred to by `uid`.
- Needs: nothing extra. Uses viewport internals (`begin_groups_preview`, `_group_chunk`, GL buffers): fragile across IngeTrazo versions.
- Pitfalls: only component instances qualify (a plain group is ignored; a test can make one with `g.xform = QMatrix4x4()` and `g.component = True`). Any document change (edit, undo, another run_python that modifies the model) stops the preview, so configure in one call and preview in a later, read-only one. Its debug log `automacao-debug.log` is appended in the plugin folder on every action and every top-level window shown (an app-wide event filter), and setup replaces `sys.excepthook`. The window's fields auto-save as nested undo steps through a 250 ms timer (`ctl._pending_save_data`); don't edit the window and the data from a script at the same time. `pick_axis` switches to a two-click tool. No modal dialogs.

```python
import sys
from views.extension_api import ExtensionApp
am = sys.modules['ingetrazo_plugin_automacao']
comp = next(g for g in scene.groups if g.is_component() and g.name == 'Fan')   # an existing component
blk = am.MotorController._new_block('rotation')
blk.update(axis_a=[0, 0, 0], axis_b=[0, 0, 1], rpm=30.0, angle_degrees=360.0, repetitions=0)
data = ExtensionApp(viewport.window(), 'automacao').document_data({'version': 3, 'motors': {}})
data['motors'][comp.uid] = {'blocks': [blk]}
ExtensionApp(viewport.window(), 'automacao').set_document_data(data)
# next run_python (read-only): ctl = viewport.window()._motor_controller; ctl._ensure_preview(); ctl._seek(0.5)
```

### Phasing Animator for IngeTrazo — `phasing_animator`
1.0.0 · aashishzharbade-arch · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Construction-sequence animation: one phase track per layer (show / fade / rise from below, keep or hide after), object pose keyframes and camera keyframes, previewed on an isolated copy of the model and exported as a PNG sequence (or MP4/WebM via FFmpeg). **In the app:** Extensions ▸ Phasing Animator V1.0 and a toolbar (open, keyframes, play, export) — all in a non-modal "Studio" window with a timeline and its own preview viewport.

**From MCP:** Partial — the animation data and its evaluation are pure (`engine`); preview, playback and export live in the Studio window.

- Module: `ingetrazo_plugin_phasing_animator` (package; `setup` only) + **`ingetrazo_plugin_phasing_animator.engine`** (data/evaluation) and `.ui` (Studio, Controller).
- Entry points (engine; times in seconds, distances in metres):
  - `empty_data()` → data dict; `validated(d)` → checked deep copy (raises `ValueError`).
  - `add_phase(d, layer_name, t)` → phase dict `{id, name, layer, start, end, effect: 'show'|'fade'|'rise', after: 'keep'|'hide', rise, enabled}` appended to `d['phases']` (one per layer, 3 s long from `t`).
  - `pose(xform_or_None)` → `{p, s, q}`; `put_key(keys, t, value)`; object tracks: `d['objects'][group.uid] = {'name', 'keys': [{'t', 'value'}]}`; camera: `put_key(d['camera'], t, camera_data(viewport.camera))`.
  - `phase_value(phase, t)` → `(visible, alpha, z_offset)`; `Snapshot(scene)` (deep copy of the document) `.apply(d, t)` poses/hides/fades the copy (`.scene`) for time `t`.
  - `EditAnimation(d, prepare_groups=())` — the command that stores `d`; `prepare_groups` get an identity `xform` so they can carry keys. Run via `viewport.history.execute` (nested, own undo step) and check `last_error`.
  - Studio (UI): `viewport.window()._phasing_animator` (Controller) `.editor` (Studio, None until opened). `Studio.begin_export(folder, cfg, encoder='')` starts a timer-driven export without the folder dialog once the Studio is open.
- Saved data: `scene.plugin_data["phasing_animator"]` (schema 1: duration, fps, phases, objects, camera, export) — written by `EditAnimation`, not `set_document_data`; QSettings `IngeTrazo/PhasingAnimator` key `ffmpeg`. Imports legacy `plugin_data["ingetrazo_animation"]`.
- Needs: OpenGL for preview/export; FFmpeg only for MP4/WebM (not in the container).
- Pitfalls: `Controller.open()` shows a QMessageBox on error; `start_export`/`browse_ffmpeg` open QFileDialogs. The run_python transaction does NOT snapshot `plugin_data` — writing it directly is not undoable; use `EditAnimation`. Export renders asynchronously after the call returns, into `<folder>/Phasing-Animator-<timestamp>/`. `Snapshot` copies the whole model (slow on big models).

```python
import sys; e = sys.modules['ingetrazo_plugin_phasing_animator.engine']
d = e.validated(scene.plugin_data.get(e.KEY))      # current data or empty
p = e.add_phase(d, "Layer 0", 0.0); p.update(effect='rise', rise=3.0)
viewport.history.execute(e.EditAnimation(d))
if viewport.history.last_error: raise RuntimeError(viewport.history.last_error)
```

### ComfyUI Viewport Renderer — `comfyui_renderer`
1.0 · seghier · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Captures the viewport, sends it to a ComfyUI server with an API-format workflow (bundled: Flux Canny, Qwen-Image ControlNet, Anima LLLite), patches prompt/steps/denoise/cfg/seed, and shows the AI render as a viewport overlay (opacity, fit, clear on camera move). **In the app:** side-tray tab «ComfyUI» (shared panel `comfyui_render`), Extensions ▸ «ComfyUI Viewport Renderer…».

**From MCP:** Partial — rendering is asynchronous (QThread) and needs a ComfyUI server; failures pop a `QMessageBox.critical` later. Workflow analysis/patching is pure.

- Module: `ingetrazo_plugin_comfyui_renderer` (+ `.panel`, `.workflow_utils`, `.comfy_client`, `.worker`, `.overlay`); live panel: `viewport.window()._comfyui_render_panel`.
- Entry points:
  - `wu.discover_workflows(custom_dir=None)` → `[(display_name, Path)]`; `wu.analyze_workflow(dict)` → node ids (`image_node`, `positive_prompt_node`, `negative_prompt_node`, `sampler_node`) + defaults (`steps`, `denoise`, `cfg`, `seed`, prompt texts); `wu.patch_workflow(wf, uploaded_image_name, positive_prompt, negative_prompt, steps, denoise, cfg, seed, analysis)` → patched deep copy.
  - Drive the panel: `p.combo_workflows.setCurrentIndex(i)` (loads it), `p.txt_positive_prompt.setPlainText(...)`, `p.spin_steps/spin_denoise/spin_cfg.setValue(...)`, `p.combo_res` (capture size), then `p.start_render()` → returns at once; poll in later calls: `p.active_worker.isRunning()`, `p.lbl_status.text()`, result `p.latest_rendered_bytes` (PNG) — write it to a file yourself. `p.stop_render()`, `p.clear_overlay()`, `p.overlay.enabled/opacity/fit_mode`.
  - `p.client` (`ComfyClient`): `check_connection()` → (ok, msg, stats); `upload_image`, `queue_prompt`, `poll_history`, `download_image` are blocking network calls — keep them off the bridge (use `start_render`).
- Saved data: none in the document. QSettings `IngeTrazo/ComfyUIRenderer`: `server_url` (default `http://127.0.0.1:8000`), workflow folder.
- Needs: a reachable ComfyUI server with the workflow's models/custom nodes (none is installed in this container).
- Pitfalls: `start_render()` with no workflow, a capture failure, or ANY worker error → modal `QMessageBox` (the error one fires after the call, blocking later bridge calls until a human closes it) — check `p.client.check_connection()[0]` first. `save_image()`, folder/file pickers are `QFileDialog`s. The panel checks the server (3 s timeout) at startup. Setting `txt_server` text rewrites QSettings. The overlay is screen-only (not in `render_image` exports).

```python
import sys
wu = sys.modules['ingetrazo_plugin_comfyui_renderer.workflow_utils']
p = viewport.window()._comfyui_render_panel
ok, msg, _ = p.client.check_connection(); print(ok, msg)
if ok and p.combo_workflows.count():
    p.combo_workflows.setCurrentIndex(0)
    p.txt_positive_prompt.setPlainText("photorealistic timber house at dusk")
    p.start_render()          # later call: p.latest_rendered_bytes -> open('/tmp/r.png','wb').write(...)
```

### Perspective Matcher — `perspective_matcher`
1.5 · seghier · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Match the viewport camera to a photo: photo overlay, draggable red/green(/blue) vanishing-line handles and an origin pin, 2- or 3-point solver (focal, FOV, pitch/yaw, eye), photo-tab "scenes", whole-model rescaling to a known length, and projecting the photo as textures onto faces. **In the app:** side-tray tab «Perspective Match», Extensions ▸ «Perspective Matcher…» (Ctrl+Shift+M), viewport overlay with handles/tabs, right-click ▸ «📷 Project Textures from Photo».

**From MCP:** Partial — the solver and calibration state are scriptable; handle placement is meant for the mouse, and texture projection / scale dialog always show message boxes.

- Module: `ingetrazo_plugin_perspective_matcher` (single `__init__.py`); live plugin: `m._GLOBAL_PLUGIN` (`.data`, `.solved`, `.pixmap`, `.panel`).
- Entry points:
  - `m.solve_perspective(data, view_w_px, view_h_px, img_w=0, img_h=0)` → `SolvedCameraParams` (`valid`, `status_msg`, `focal_px`, `focal_35mm`, `fov_deg`, `pitch_deg`, `yaw_deg`, `eye`/`target`/`forward` QVector3D, `vp_x`/`vp_y`/`vp_z` px). Pure.
  - `m.PerspectiveMatchData()` — handles are `[rx, ry]` in 0..1 of the photo's letterboxed rect: `x1_a/x1_b/x2_a/x2_b` (two lines along X), `y1_*`, `y2_*` (Y), `z1_*`, `z2_*` (Z, 3-point), `origin`; `mode` "2point"|"3point"; `distance` m (eye to origin); `fix_verticals`. `to_dict()/from_dict()`.
  - `p.load_image(path)` — load photo (no dialog), embed base64 (<16 MB), enable, solve, save. `p.update_camera_from_guides()` — solve with the viewport size and set the camera. `p.toggle_enabled(bool)`, `p.switch_to_view(i)`, `p.add_current_view()`, `p.set_camera_locked(bool)`, `p.save_state()`.
  - `p.scale_model_to_edge(current_m, target_m)` / `p.apply_scale_factor(f)` — scales the WHOLE scene about the origin (ScaleMatchSceneCommand).
  - `m.get_camera_state(cam)` / `m.apply_camera_state(cam, dict)` — JSON camera snapshot.
- Saved data: `scene.plugin_data["perspective_matcher"]` = `PerspectiveMatchData.to_dict()` (includes the photo as base64; `enabled` is always reset to False on load).
- Needs: a photo file on the app's machine.
- Pitfalls: `project_textures_from_photo()` ends in a `QMessageBox` on every path; `open_scale_dialog()` and `rename_view()` (QInputDialog), Load Photo (QFileDialog) block. Every save is a nested `set_document_data` undo step; scaling is a command whose undo re-scales by 1/f (not a snapshot) and touches all geometry, guides, dimensions — only on explicit request. While `data.camera_locked`, the event filter freezes camera navigation. Handle coords depend on the viewport/photo aspect.

```python
import sys
m = sys.modules['ingetrazo_plugin_perspective_matcher']
d = m.PerspectiveMatchData()            # adjust d.x1_a … d.origin from the photo first
r = m.solve_perspective(d, 1600, 1000, 1600, 1000)
print(r.valid, r.status_msg, round(r.fov_deg, 1), round(r.focal_35mm, 1), r.eye)
```

### Texture Library for IngeTrazo — `texture_library`
1.0.0 · aashishzharbade-arch · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

A texture browser for a folder of images (sub-folders = categories; PBR sets shown once by their colour map), with real-world tile size per texture, painting faces/groups with it as a named material, import of images/ZIPs, and optional Poly Haven (CC0) downloads. **In the app:** side-tray tab «Textures» + Extensions ▸ Texture library… (Ctrl+Shift+T).

**From MCP:** Full — the library scan and the paint step are plain functions (`scan_library`, `texture_dict`, `paint_commands`); only the panel buttons use dialogs/network.

- Module: `ingetrazo_plugin_texture_library` (single file)
- Entry points:
  - `scan_library(root, limit=5000)` → [`Entry(name, path, category, tile, source)`]: colour/plain images under `root`; `tile` = (sw, sh) m from the `<image>.json` sidecar. `classify(stem)` → (`"color"|"other"|"plain"`, name).
  - `texture_dict(path, sw, sh, rot=0.0)` → `{"path", "sw", "sh"[, "rot"]}` (tile size in metres, rot degrees).
  - `paint_commands(scene, faces, groups, texture, base_name)` → (`[commands]`, `Material`): face texture + clear opacity + `attrs["mat"]` tag, registers the material (name made unique by `unique_material_name`); groups get `group.material = {"texture", "mat"}`. Run them with `viewport.history.execute(CompoundCommand(cmds))`.
  - `selected_faces(scene)`, `selected_groups(scene)`; `read_sidecar(img)`, `save_sidecar(img, **fields)`; `import_into_library(root, category, files)` → images copied (ZIP members too); `default_root()` → `~/.local/share/IngeTrazo/texture_library`.
  - The panel: `viewport.window().findChild(QDockWidget, 'extension_texture_library')`; its widget `._root` is the library folder in use.
- Saved data: on faces `attrs["texture"]`, `attrs["mat"]`; `scene.materials[name]`; QSettings `("IngeTrazo", "TextureLibraryExtension")`: `root`, `unit`; sidecar `.json` beside each image.
- Needs: the image files on disk (the face keeps the path). Poly Haven download needs network (`download_poly_asset`, `_http_get`; https polyhaven.com only).
- Pitfalls:
  - Don't press «Change…», «Import…» (QFileDialog/QInputDialog) or «Get online…» (dialog + network threads). `paint_with()` arms the Paint tool (UI state only).
  - The material registry is NOT reverted by the run_python rollback: use `history.execute(CompoundCommand(cmds))` (own undo step) and check `viewport.history.last_error`.
  - The panel's «Remember size» rewrites the image's sidecar whenever you paint.
  - Several faces → `tools.paint._surface_commands` wraps the image across them (curved surfaces).

```python
import sys
from core.history import CompoundCommand
tl = sys.modules['ingetrazo_plugin_texture_library']
f = mesh.add_face([QVector3D(10,0,0), QVector3D(12,0,0), QVector3D(12,2,0), QVector3D(10,2,0)])
tex = tl.texture_dict('/path/to/oak_diff_1k.jpg', 0.6, 0.6)
cmds, mat = tl.paint_commands(scene, [f], [], tex, 'Oak')
viewport.history.execute(CompoundCommand(cmds))
if viewport.history.last_error: raise RuntimeError(viewport.history.last_error)
print(mat.name, f.attrs['mat'], f.attrs['texture']['sw'])
```

### MultiView for IngeTrazo — `ingetrazo_multiview`
1.0.5 · aashishzharbade-arch · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Replaces the single viewport with a 2×2 (or two-view) layout — Top, Perspective (the main viewport), Front, Right — sharing the document and undo history, with enlarge/restore by double Space and F4 to toggle. **In the app:** a "IngeTrazo MultiView" toolbar and Extensions ▸ IngeTrazo MultiView (Toggle, Enlarge/Restore, Zoom Extents All, Top/Perspective/Front/Right view).

**From MCP:** UI only — it is a screen layout; an AI gets every view through `viewport.camera` (`set_view`, `fit_box`) and `viewport.render_image`. Its controller has no dialogs, so it can be inspected or switched if a user asks.

- Module: `ingetrazo_plugin_ingetrazo_multiview`
- Entry points (all on the module global `_multiview_controller`, a `MultiViewController`; nothing takes geometry):
  - `.is_active` (bool), `.activate()` / `.deactivate()` / `.toggle()` — switch layouts; `.toggle_two_views(bool)`, `.toggle_maximize()`.
  - `.apply_preset_view(view_key, perspective)` — `view_key` ∈ "top", "iso", "front", "right"; acts on the active cell (or the main viewport when inactive).
  - `.zoom_extents_all()` — fits every visible cell to the model bounds.
  - `.top_vp`, `.front_vp`, `.right_vp` — secondary `Viewport`s (None until first activation); `.toolbar` (object name `ingetrazo_multiview_toolbar`).
- Saved data: none (an error log `multiview.log` in the app-data folder).
- Needs: nothing extra.
- Pitfalls: while MultiView is active the main viewport is only one cell, so the bridge `screenshot` aspect changes — call `viewport.camera.set_aspect(w, h)` first, or `deactivate()`. Activating reparents the GL widget (refuses while an extension workspace is shown; it only flashes a status message). It wraps `viewport.set_nav_mode` and installs an application-wide key filter that eats F4 and double Space. Camera changes are not undo steps.

```python
import sys; c = sys.modules['ingetrazo_plugin_ingetrazo_multiview']._multiview_controller
print(c.is_active, c.top_vp is not None)
# if c.is_active: c.deactivate()   # back to one viewport before screenshots
```

### Standard Views 2 — `standard_views_2`
1.0.0 · TB · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Two extra toolbars: «Standard Views 2» (Isometric, Top, Front, Right, Left, Back, Bottom with 48 px icons, a projection-cycle button Parallel → Perspective → 2-Point, and Smart Zoom = zoom selection, else extents) and «Theme Control» (one light/dark toggle). **In the app:** toolbars `StandardViews-2 Toolbar` and `Theme Toggler Toolbar`; no menu entries.

**From MCP:** Partial — the camera helpers are plain functions taking an `ExtensionApp` (no dialogs, no model change); the theme toggle is a persistent setting. The core camera (`viewport.camera.set_view(name)`, `.set_projection(mode)`) does the same without the plugin.

- Module: `ingetrazo_plugin_standard_views_2` (single file)
- Entry points (`app = ExtensionApp(viewport.window(), 'standard_views_2')`):
  - `trigger_camera_view(app, name)` — `name` in `"top" "bottom" "front" "back" "right" "left"` (also `"iso"`): `camera.set_view(name)` then forces `perspective = False` (parallel).
  - `trigger_standard_isometric(app)` — yaw −45°, pitch 30°, `perspective = True` (does NOT clear `two_point`).
  - `cycle_projection_modes(app)` — parallel → perspective → two_point → parallel (`camera.projection_mode()` tells where you are).
  - `trigger_smart_zoom_combo(app, window)` — triggers `window._act_zoom_selection` if `scene.selection` is non-empty, else `window._act_zoom_extents`.
  - `toggle_global_preferences_theme(window)` — flips light/dark via `views.theme.save_theme` + `apply_theme`.
- Saved data: none of its own. The theme toggle writes the app's general setting (theme) — persistent.
- Needs: nothing extra.
- Pitfalls:
  - Every helper swallows exceptions and only `print`s them (no error raised): check the camera afterwards.
  - Camera changes are not undo steps (the run_python rollback does not restore the view); save/restore yaw, pitch, target, distance, perspective, two_point yourself if the view matters.
  - Do not call `toggle_global_preferences_theme` unless asked — it changes the user's saved theme.
  - An overlay redraws the toolbar icons on every paint when theme/projection change (harmless).

```python
import sys, math
from views.extension_api import ExtensionApp
sv = sys.modules['ingetrazo_plugin_standard_views_2']
app = ExtensionApp(viewport.window(), 'standard_views_2')
sv.trigger_camera_view(app, 'front')
print(viewport.camera.projection_mode(), round(math.degrees(viewport.camera.yaw)))   # parallel -90
```

### JA Tree · Model outliner — `outliner`
0.1.0 · Julio Angulo · GPL-3.0-or-later · Community (catalogue id `ja_tree`)

*Live-tested in IngeTrazo 0.5.9.*

Tree of groups, components (nested) and section planes: search/filter, rename, hide/show, assign tag, delete, zoom to, open for editing, and drag to re-parent/reorder keeping world position. **In the app:** side-tray tab «Tree», Extensions ▸ «JA Tree…» (Ctrl+Alt+O), viewport right-click ▸ «Localizar en JA Tree».

**From MCP:** Full — the hierarchy reader and the Rename/Reparent commands are plain code; the rest are native host commands.

- Module: `ingetrazo_plugin_outliner` (+ `.model`, `.controller`, `.ui`, `.i18n`)
- Entry points (`md = sys.modules['ingetrazo_plugin_outliner.model']`):
  - `md.index_scene(scene)` → ordered dict `key → Node`; key = `"group:<uid>"` / `"section:<uid>"`; Node has `entity`, `parent`, `ancestors` (tuple of groups), `world` (QMatrix4x4), `kind` ("group"|"component"|"section"), `visible`, `locked`, `name`, `tag`, `parent_key`. `md.entity_key(group)` builds a key.
  - `md.Reparent(scene, keys, destination=None, position=None)` — move groups (keys; nested selections reduced to topmost) into `destination` key (None = model root) at list index `position` (None = end); world placement kept via `xform = inv(parent_world) * world`. Validation raises `ValueError` in the constructor; `.changed` False = no-op.
  - `md.Rename(entity, name)` (1–256 chars). `md.bounds_of(nodes, keys)` → `(QVector3D lo, hi)` world bounds or `(None, None)`. `md.matching_keys(nodes, query, kind)` → `(included, direct)` search.
  - Controller: `viewport.window()._outliner_controllers['outliner']` (`zoom()`, `edit(key)`, `leave()` — UI glue; errors go to the status bar, not exceptions).
- Saved data: none in the document (structure is native). QSettings `outliner/sort`, `outliner/tags`, `outliner/language`.
- Needs: nothing extra.
- Pitfalls: `assign_tag()` opens a modal `QInputDialog`; `ctrl.move/toggle/delete` swallow errors — build commands yourself. On groups created in the SAME script call `cmd.do(scene)` (the outer step then undoes everything); on EXISTING groups use `viewport.history.execute(cmd)` + check `last_error` — its own undo step, and moving a nested group to the root also leaves the outer step "changed": undo twice. `Reparent` refuses while a group is open for editing (`scene.edit_group`). Re-parenting gives the parent an identity `xform` (it becomes a non-component container).

```python
import sys
md = sys.modules['ingetrazo_plugin_outliner.model']
a = extrude([(10,0),(11,0),(11,1),(10,1)], 0, 1, name="Parent")
b = extrude([(12,0),(13,0),(13,1),(12,1)], 0, 1, name="Child")
md.Reparent(scene, [md.entity_key(b)], md.entity_key(a)).do(scene)   # new groups: plain do()
nodes = md.index_scene(scene)
print([(n.name, n.kind, n.parent_key) for n in nodes.values()], md.bounds_of(nodes, [md.entity_key(a)]))
```

### JA Units · Units — `ja_units`
0.1.0 · Julio Angulo · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Sets the document's length unit (mm, cm, m, in, ft, ft-in, fractional in/ft-in) and precision through the host's native unit engine — display and typed input only, geometry stays in metres; plus built-in/personal presets and two toolbar preset slots. **In the app:** side-tray tab «Units/Unidades», a «JA Units» toolbar, and entries in the Extensions menu.

**From MCP:** Full — the `Controller` methods take plain values; each change is one `SetUnitsCommand`.

- Module: `ingetrazo_plugin_ja_units` (single file package)
- Entry points (`ctrl = viewport.window()._ja_units_controllers['ja_units']`):
  - `ctrl.apply({"length": code, "precision": n})` — code in `UNIT_ORDER`; n = decimals 0..6 (fractional units: 0..4 = 1", 1/4", 1/16", 1/32", 1/64"). Raises `ValueError` on bad values, `RuntimeError` if the command failed; no command if nothing changes.
  - `ctrl.set_unit(code)`, `ctrl.set_precision(n)`, `ctrl.step_precision(±1)`, `ctrl.apply_builtin(key)` (keys of `PRESETS`: `metric_mm`, `metric_cm`, `metric_m`, `feet_inches`, `fractional_inches`), `ctrl.current()` → `{"length", "precision"}`, `ctrl.summary()` → e.g. "mm · 0".
  - `SetUnitsCommand(chosen)` — the undoable command itself; `validated(dict)` → clean dict or None.
- Saved data: `scene.units` and `scene.dimension_style["units"/"decimals"]` (native, in the .igz). Personal presets/slots/toolbar in QSettings `extensions/ja_units/{presets,slots,toolbar}`; `remember_default()` writes the app's new-document units.
- Needs: nothing extra.
- Pitfalls: `ctrl.run(fn)` and every toolbar/menu action show a modal `QMessageBox` on error, `ask_save_slot()` opens a `QInputDialog` — call `apply`/`set_*` directly. Each apply is its own nested undo step (not rolled back if the script raises later). `save_user`, `remember_default` change the user's settings, not the document — avoid unless asked. Unit choice affects only display (`core.units.fmt_len`), never coordinates.

```python
import sys
m = sys.modules['ingetrazo_plugin_ja_units']
ctrl = viewport.window()._ja_units_controllers['ja_units']
ctrl.apply({"length": "cm", "precision": 1})
from core import units
print(ctrl.summary(), units.fmt_len(1.2345))
```

### Organize Toolbar — `organize_toolbar`
1.2 · seghier · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

A toolbar and an Extensions ▸ Organize submenu with six buttons for existing core commands: Hide, Unhide Last, Unhide All, Reverse Faces, Make Group, Explode Group. **In the app:** «Organize» toolbar + Extensions ▸ Organize ▸ …

**From MCP:** UI only — every button calls a `MainWindow._on_*` method (or its own Unhide Last); an AI should use the same core commands directly.

- Module: `ingetrazo_plugin_organize_toolbar`
- Entry points (all UI handlers, taking the MainWindow `viewport.window()`):
  - `_on_hide_impl(win)` → `win._on_hide()` on the current selection, then remembers the batch in `_SESSION_HIDDEN_HISTORY`.
  - `_on_unhide_last_impl(win)` — unhides the most recent Hide still hidden (searches the undo stack, then the session list).
  - `_safe_call(win, "_on_unhide_all" | "_on_reverse_faces" | "_on_make_group" | "_on_explode_group")`.
  - Core equivalents for scripts (`core.history`): `HideCommand(entities, hidden=True|False)`, `FlipFacesCommand(faces)`, `MakeGroupCommand(faces, edges, name=None)`, `MakeNestedGroupCommand(faces, edges, groups)`, `ExplodeGroupCommand(group)`.
- Saved data: none (`_SESSION_HIDDEN_HISTORY` is in memory only).
- Needs: nothing extra.
- Pitfalls:
  - `setup()` REPLACES `win._on_unhide_last` (Edit ▸ Unhide ▸ Last) with the plugin's version.
  - Every handler runs `viewport.history.execute(...)` itself (nested undo entry, status-bar message, no error raised).
  - With core commands inside `run_python`: `cmd.do(scene)` is undone with the call only for changes to `scene.mesh` and NEW groups. Hiding a group (`group.hidden`), flipping a group's faces, or exploding/removing a group are not; use `viewport.history.execute(cmd)` for those (own undo entry).
  - Make Group / Explode refuse while a group is open for editing (`scene.edit_group`). No dialogs.

```python
from core.history import HideCommand, MakeGroupCommand
f = mesh.add_face([QVector3D(10, 0, 0), QVector3D(11, 0, 0), QVector3D(11, 1, 0), QVector3D(10, 1, 0)])
MakeGroupCommand([f], [], name="Slab").do(scene)        # loose face -> new group
g = groups[-1]
viewport.history.execute(HideCommand([g], hidden=True)) # hide: its own undo step
```

### Isolate / Unisolate Selection — `isolate_unisolate_selection_toolbar`
no version · sudhanva-gh · GPL-3.0 (repository licence; none in the file) · GitHub, not catalogued (sudhanva-gh/isolate-unisolate-ingetrazo-plugin @ ad4405f)

*Live-tested in IngeTrazo 0.5.9.*

Hides every group/component except the selected ones, remembering the previous visibility; Unisolate restores it. **In the app:** Extensions ▸ Isolate Selected (Ctrl+Shift+Alt+I) / Unisolate / Restore (Ctrl+Shift+Alt+U), the same two in Extensions ▸ Isolate Selection ▸, and an «Isolate Selection» toolbar.

**From MCP:** Full — `isolate_selected(viewport)` / `unisolate(viewport)` need only a selection of groups; no dialogs. Mind the undo caveat below.

- Module: `ingetrazo_plugin_isolate_unisolate_selection_toolbar`
- Entry points:
  - `isolate_selected(viewport)` — selected groups (`scene.selection`, nested ones included) stay/become visible; EVERY other group in the current context (top level, or the open group's children, at all depths) gets `hidden = True`. The previous `hidden` flags are stored by `uid`. If an isolation is already active it is restored first. No selected group → status message only.
  - `unisolate(viewport)` — puts back the stored `hidden` flags and clears the data; "Nothing is isolated." otherwise.
- Saved data: document data under the key `isolate_selection_v2` (NOT the stem): `scene.plugin_data['isolate_selection_v2'] = {"active": True, "states": {uid: was_hidden}}`; absent when not isolated.
- Needs: nothing extra.
- Pitfalls:
  - Undo does NOT bring the groups back: the hide is done with `SnapshotImport`, which does not track `group.hidden`, so `undo` only reverts the stored data and the groups stay hidden (and Unisolate then says nothing is isolated). Undo an isolation with `unisolate(viewport)`, or Edit ▸ Unhide ▸ All.
  - Each call pushes 2 undo entries of its own (4 when re-isolating) besides the `run_python` step.
  - Loose geometry is never hidden; only groups/components.
  - Isolating a container also hides the groups nested inside it (they are not "selected"), so it can look empty.

```python
import sys
iso = sys.modules['ingetrazo_plugin_isolate_unisolate_selection_toolbar']
g = groups[0]
selection.clear(); selection.add(g)
iso.isolate_selected(viewport)                       # everything else hidden
print(scene.plugin_data.get('isolate_selection_v2', {}).get('active'))
# … later, in another call:  iso.unisolate(viewport)
```

## 11. Interface

### Coloured Tools + Space (UI Colores + Space) — `ui_colores_space`
0.1.0 · sanask24 · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Recolours the toolbar icons by tool family (drawing blue, modelling orange, sections violet, solids red, annotation green, navigation turquoise, materials magenta…, lighter hues in the dark theme) and makes **Space** open the native command search (Select moves from Space to V if V is free). **In the app:** Extensions ▸ UI Colores + Space ▸ two check items (colours, Space) + «Buscar comandos…».

**From MCP:** UI only — it changes icons and keyboard shortcuts; nothing touches the model. An AI can read its state, or get a family colour / icon for a tool key.

- Module: `ingetrazo_plugin_ui_colores_space` (+ `.controller`, `.host`, `.icons`; relative imports only)
- Entry points (read-only use recommended):
  - `viewport.window()._ui_colores_space_controller` — the `Controller` (QObject): `.colors_enabled`, `.search_enabled` (bools), `.original_icons` {action: QIcon}, `.shortcut_changes` {action: (before, applied)}, `.color_action` / `.space_action` (the checkable menu items).
  - `icons.icon_category(key)` → family name (`"drawing"`, `"modeling"`, …, `"utility"` default); `icons.icon_color(key)` → QColor for the current palette; `icons.tool_icon(key)` → coloured QIcon (null for an unknown key). Keys are the window's `_icon_actions` keys (`"line"`, `"pushpull"`, `"orbit"`…).
  - `host.search_action(win)` (objectName `command_search`), `host.select_action(win)`, `host.icon_actions(win)` → [(action, key)].
- Saved data: QSettings (app default org/app) `ui_colores_space/colors`, `ui_colores_space/space` (bools). No document data.
- Needs: nothing extra (IngeTrazo ≥ 0.5.7 internals `_icon_actions`, `_tool_actions["select"]`).
- Pitfalls:
  - `Controller.set_colors(b)` / `set_search(b)` (and toggling the menu items) WRITE those QSettings and rebind shortcuts — do not call from a script unless asked; if you do, restore the previous value.
  - It keeps its own copy of the icon renderer: with `custom_icons` installed, whichever refreshes last wins (palette change / new child widget re-triggers this one).
  - No dialogs; «Buscar comandos…» only triggers the native search popup (keyboard UI, useless over MCP).

```python
import sys
ic = sys.modules['ingetrazo_plugin_ui_colores_space.icons']
c = viewport.window()._ui_colores_space_controller
print(c.colors_enabled, c.search_enabled, ic.icon_category('pushpull'), ic.icon_color('line').name())
```

### Custom Icons · SVG icon themes — `custom_icons`
1.1 · APBureauEtude · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Replaces toolbar icons and 3D-view tool cursors with the user's SVG files: one folder per theme with a `mapping.json` (icon/cursor key → SVG path), paired by drag and drop on a local web page. **In the app:** Extensions ▸ Custom icons ▸ Choose icons… (opens the browser page) / Open icons folder / Reload icons.

**From MCP:** Partial — no model effect; the theme files and `reload(window)` are plain Python (no dialogs), but the pairing editor is a browser page.

- Module: `ingetrazo_plugin_custom_icons` (+ `.editor`, the page's HTTP server; relative import)
- Entry points:
  - `root_folder()` → Path `~/.local/share/ingetrazo/custom_icons` (beside the user plugins dir); `custom_dir()` (`Icônes custom/`), `originals_dir()` (`Icônes originales/`).
  - `themes()` → [theme folder names]; `active_theme()` → name or None (IngeTrazo's own); `save_theme(name|None)` writes `settings.json`.
  - `theme_pool(theme)` → {relative path: Path} of its SVGs; `load_mapping(theme, pool)` / `save_mapping(theme, {"icons": {key: path}, "cursors": {slot: path}})`; `suggested_mapping(paths)` pairs by file name (`line.svg` → icon `line`, `cursors/line.svg` → cursor `line`, `cursors/tape+.svg` → guide-mode slot).
  - `reload(window=None)` → int (icons + cursors replaced): re-exports the originals, re-reads the active theme, redraws when given the window.
  - Pure renderers: `render_svg(bytes, size_px, ink=None)` → QImage|None, `svg_icon(bytes, ink=None)` → QIcon|None, `svg_cursor(bytes, kind, plus=False, dpr=1.0)` → QCursor|None, `hotspot_kind(slot)` → `"tip"|"point"|"drip"|(fx, fy)`, `find_hotspot(QImage, kind)` → (x, y). `currentColor` in an SVG = interface ink.
- Saved data: files under `root_folder()` (`settings.json`, each theme's `mapping.json`, `Sauvegardes/`, `Importées/`); no QSettings, no document data.
- Needs: QtSvg (falls back to Qt's svg image plugin). The editor binds an HTTP server on 127.0.0.1 (random port, token URL).
- Pitfalls:
  - `setup()` monkey-patches `views.icons.tool_icon` / `tool_cursor` in EVERY loaded module (`custom_icons_original` attribute holds IngeTrazo's).
  - Never call `open_editor` (starts the server + opens a browser), `open_custom_folder` / `_open_folder` (file manager).
  - `reload()` writes one SVG per built-in icon into `Icônes originales/` and `save_theme`/`save_mapping` change what the user sees at next start: only on request.
  - With `ui_colores_space` also installed, the two fight over toolbar icons (last refresh wins).

```python
import sys
ci = sys.modules['ingetrazo_plugin_custom_icons']
svg = b'<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 48 48"><path d="M8 40 L40 8" stroke="currentColor" stroke-width="3"/></svg>'
print(ci.themes(), ci.active_theme(), ci.svg_icon(svg) is not None, ci.hotspot_kind('line'))
```

### Display Scale for IngeTrazo — `display_scale`
1.2.0 · aashishzharbade-arch · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.9.*

Scales the interface (font, toolbar/tab/list icon sizes) live with presets that make the screen lay out like 720p/1080p/2K/4K, optional restart at a true `QT_SCALE_FACTOR`, and a Focus Mode that hides every visible side panel. **In the app:** «Display Scale» toolbar (monitor icon → dialog) + Extensions ▸ Display Scale... / Display Scale: Smaller (more space) / Display Scale: Larger / Focus Mode (hide panels) / Display Scale: Toolbar.

**From MCP:** UI only — it changes the user's interface size and settings, not the model. An AI can read the saved scale, compute workspace sizes, or use Focus Mode for a bigger viewport (restore afterwards).

- Module: `ingetrazo_plugin_display_scale` (single file; Tool keys `plugin_display_scale_DisplayScaleTool`, `…_ScaleSmallerTool`, `…_ScaleLargerTool`, `…_FocusModeTool`, `…_DisplayScaleToolbarTool`)
- Entry points (pure / harmless):
  - `screen_info(win=None)` → (physical_w_px, physical_h_px, os_dpr).
  - `scale_for_workspace(virtual_w, phys_w, os_dpr)` → UI scale (clamped 0.40–3.00); `workspace_for_scale(scale, phys_w, phys_h, os_dpr)` → (w, h) px the UI lays out like.
  - `restart_command(scale)` → (argv list, env dict) — only BUILDS the command; `permanent_command(scale)` → shell text.
  - `LIVE` (`_Live`): `.scale`, `.icon`, `.text`, `.active`; `LIVE.apply(scale, icon, text)` / `LIVE.reset()` change the running UI (not saved).
  - `toggle_focus(win)` → True when panels were hidden; call again to restore them (`_focus["hidden"]` holds the docks).
- Saved data: QSettings `("ingetrazo-plugins", "display_scale")`: `scale`, `icon`, `text` (floats), `auto` ("true"/"false"; re-applied at every start).
- Needs: nothing extra; «Restart at true scale» relaunches the app process (`subprocess.Popen`).
- Pitfalls:
  - Never activate `DisplayScaleTool` or the toolbar button (modal `_Dialog.exec()`); never call `restart_at_scale` (QMessageBox, closes the window, starts a new process).
  - `ScaleSmallerTool` / `ScaleLargerTool` (`_step`) WRITE QSettings `scale` and rescale the live UI.
  - `LIVE.apply` installs an app-wide event filter and changes the app font; leave it alone unless asked, and `LIVE.apply(old values)` afterwards.
  - Module import schedules timers (2.2 s auto-apply, 2.7 s toolbar) — re-importing the file would apply the saved scale again.

```python
import sys
ds = sys.modules['ingetrazo_plugin_display_scale']
w, h, dpr = ds.screen_info(viewport.window())
print(w, h, dpr, ds.workspace_for_scale(ds.LIVE.scale, w, h, dpr))
```
