# Extensions over MCP — using IngeTrazo plugins from an AI agent

Companion to [ai-bridge.md](ai-bridge.md). The bridge has five fixed tools (`run_python`,
`query_model`, `screenshot`, `undo`, `redo`); plugins cannot add tools. But `run_python` runs
inside the live app, where every plugin is already imported, so an agent can call a plugin's
own functions directly. This page says how, and documents each installed extension.

Written against IngeTrazo 0.5.7 and the extensions catalogue of 2026-10-10
(<https://ingetrazo.com/extensiones>, <https://github.com/ingelibre/ingetrazo-extensions>).
The plugin API is not stable; re-check a plugin's entry points after updating it.

## 1. Where plugins live

- Bundled: `<app>/plugins/` (AI Assistant, AI Bridge, Python Console, Model Info, Solid
  Inspector, Render with Blender). Example extensions (Levels, Polygon properties, Windowizer)
  ship in `<app>/examples/extensions/` and are installed from **Extensions ▸ Example extensions**.
- Per-user: `$XDG_DATA_HOME/ingetrazo/plugins/` (Linux, default `~/.local/share/...`),
  `%APPDATA%\ingetrazo\plugins\` (Windows). **Extensions ▸ Open plugins folder** opens it.
  A plugin is one `.py` file or one package folder. Install = copy + restart; uninstall =
  delete + restart. Verify the SHA-256 against the catalogue entry before installing.
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
- OpenTrace BIM has a self-updater that downloads from the official catalogue. It is off unless enabled in its panel.
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
IngeTrazo 0.5.7 (2026-10-11), passed, and left the document unchanged afterwards.

| Extension | Stem | From MCP | Catalogue status | Live-tested |
|---|---|---|---|---|
| Array | `array_tool` | Full | Community | yes |
| IngeTrazo Bevel | `ingetrazo_bevel` | Full | Reviewed | yes |
| Taper Tool for IngeTrazo | `ingetrazo_taper` | Full | Reviewed | yes |
| Loop Cut | `loop_cut` | Partial | Reviewed | yes |
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
| OpenTrace BIM | `arquitetura_parametrica` | Full | Community | yes |
| ArchXQ IT Lite · Architecture by phases | `archxq_it` | Partial | Community | yes |
| CAD2IngeTrazo · CAD plans to BIM | `cad2ingetrazo` | Partial | Community | yes |
| planArq | `planarq` | Full | Reviewed | yes |
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
| Point Cloud & 3D Gaussian Splatting | `pointcloud_splat` | Partial | Community | yes |
| JA Sun · Sun and shadows | `ja_sun` | Full | Community | yes |
| Styles, Shadows and Map toolbars | `igz_tb_toolbar` | Partial | Community | yes |
| Scenes (scene manager) | `igz_tb_scenes` | Partial | Community | yes |
| Phasing Animator for IngeTrazo | `phasing_animator` | Partial | Community | yes |
| ComfyUI Viewport Renderer | `comfyui_renderer` | Partial | Community | yes |
| Perspective Matcher | `perspective_matcher` | Partial | Community | yes |
| MultiView for IngeTrazo | `ingetrazo_multiview` | UI only | Community | yes |
| JA Tree · Model outliner | `outliner` | Full | Community | yes |
| JA Units · Units | `ja_units` | Full | Community | yes |
| Organize Toolbar | `organize_toolbar` | UI only | Community | yes |
| Isolate / Unisolate Selection | `isolate_unisolate_selection_toolbar` | Full | not catalogued | yes |

## 6. Modelling and editing

### Array — `array_tool`
1.2 · Pesi (pesi3d.de) · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.7.*

Copies the selection (groups, components, loose faces/edges) in a 1D/2D/3D grid with per-copy move, rotate and scale. **In the app:** Extensions ▸ Array…, right-click ▸ Array…, and a one-button PESI3D toolbar (all open a non-modal dialog).

**From MCP:** Full — the whole array is a pure parameter dict + a `Command` that works on the current selection, no UI needed.

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

*Live-tested in IngeTrazo 0.5.7.*

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

*Live-tested in IngeTrazo 0.5.7.*

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
1.0 · Daniel Wagner · MIT · Reviewed

*Live-tested in IngeTrazo 0.5.7.*

Inserts one edge ring through a closed strip of quads (at the midpoints of the crossed edges) or, failing that, through a closed planar cross-section perpendicular to the picked edge; open, branching or vertex-touching rings are refused. **In the app:** Extensions ▸ «Corte em loop» and the «LoopTools» toolbar button (an interactive tool: hover an edge for a pink preview, click to cut). UI texts are Portuguese.

**From MCP:** Partial — there is no public cut function: `_plan()` is pure, the cut itself (10 lines in `LoopCutTool.on_click`) must be replicated, as below.

- Module: `ingetrazo_plugin_loop_cut`
- Entry points:
  - `_plan(mesh, seed_edge)` → `(pairs, crossed_edges, points)`: `pairs` = `{face: (edge_in, edge_out)}`, `points` = `{edge: QVector3D}` where each crossed edge is split. Raises `ValueError` (Portuguese message) when no clean ring exists. Read-only. `seed_edge` is an edge the ring CROSSES (for a box side strip: a vertical edge → horizontal cut at mid-height).
  - `_loop_plan(mesh, seed)` (quads only, midpoints) and `_plane_plan(mesh, seed)` (plane through the seed's midpoint, normal along the seed) — the two strategies `_plan` tries in that order.
  - `_tool` (module global) / `viewport.window()._tools['plugin_loop_cut_LoopCutTool']` — the tool; its `on_click` needs a screen pick, not useful from a script.
- Saved data: none.
- Needs: nothing extra (`core.topology.split_face_by_chord`).
- Pitfalls:
  - Works on whichever mesh you pass, but only `scene.mesh` is restored by `undo`; cut a group's mesh only after opening it, or on a group created in the same call.
  - Faces with holes, open borders, or a plane through an existing vertex are refused (`ValueError`); catch it or let the call roll back.
  - The cut replaces every face of the ring (new `Face` objects, attrs copied); hold no references to old faces.
  - No dialogs.

```python
import sys
from core.topology import split_face_by_chord
lc = sys.modules['ingetrazo_plugin_loop_cut']
def loop_cut(mesh, seed):                      # same as LoopCutTool.on_click's mutate
    pairs, _crossed, points = lc._plan(mesh, seed)
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
print(loop_cut(M, seed))                        # 4 faces cut at z = 1
```

### Vertex Editor — `vertex_editor`
1.1 · Daniel Wagner · MIT · Reviewed

*Live-tested in IngeTrazo 0.5.7.*

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

*Live-tested in IngeTrazo 0.5.7.*

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

*Live-tested in IngeTrazo 0.5.7.*

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
1.1 · Pesi (pesi3d.de) · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.7.*

Joins chains of loose edges into one curve (selects with one click), optionally closing small gaps between free ends and dropping collinear points; Unweld clears the curve ids. **In the app:** Extensions ▸ Weld ▸ Weld Edges… / Unweld Edges, right-click on selected edges, and a PESI3D toolbar (Weld opens a non-modal dialog with a preview overlay).

**From MCP:** Full — `weld()` / `unweld()` are pure functions on a `Mesh`, no UI.

- Module: `ingetrazo_plugin_weld_tool`
- Entry points:
  - `weld(mesh, edges, tol=0.0, collinear=False)` → `{"curves", "gaps", "points", "edges"}`: mutates `mesh` in place. `tol` (metres) closes gaps between open chain ends closer than it (two free ends meet in the middle; a free end next to attached geometry moves onto it; faces never move); `collinear=True` removes interior straight-run vertices not used by faces; every chain of ≥ 2 edges gets one new `edge.curve` id. Returns counts and the resulting edge list.
  - `unweld(mesh, edges)` → int: clears `edge.curve` on `edges`.
  - `selected_edges(scene)` → selected `Edge`s that live in `scene.mesh`.
  - `build_chains(edges)` → `[(edge_list, vertex_list, closed)]`; `find_gaps(edges, tol)` → `(pairs, blocked)` (preview, read-only).
  - `run_weld(viewport, edges, tol, collinear)` / `run_unweld(viewport)` — the UI commands (own `history.execute`, status-bar messages).
- Saved data: QSettings `plugins/weld_tool/gaps`, `plugins/weld_tool/tol_m` (default 0.01), `plugins/weld_tool/collinear`; nothing in the document.
- Needs: nothing extra.
- Pitfalls:
  - Call `weld(scene.mesh, …)` directly inside `run_python` (one undo step). `run_weld` pushes its own nested undo entry.
  - Only `scene.mesh` (the active mesh) is snapshotted by the transaction; welding a closed group's `g.mesh` is not undone by `undo`. Open the group first or build new geometry.
  - A chain breaks at junctions (3+ edges) and where unselected geometry is attached; single-edge chains get no curve id.
  - The edge objects change when gaps or collinear points are removed: use the returned `"edges"`, not your old list.

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

*Live-tested in IngeTrazo 0.5.7.*

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
1.1 · Pesi (pesi3d.de) · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.7.*

Pairs triangles into four-sided faces: flat pairs become one real quad face, curved pairs keep their triangles with a soft (or hidden) diagonal; Grid Remesh rebuilds a height-field surface (terrain) as a regular grid of quads clipped to its outline. **In the app:** Extensions ▸ Quadrify ▸ Quadrify… / Grid Remesh…, its own toolbar, and the right-click menu on selected faces.

**From MCP:** Full — `plan`/`apply_plan` and `grid_plan`/`apply_grid` are pure and work on any `Mesh`.

- Module: `ingetrazo_plugin_quadrify_tool` (single file)
- Entry points:
  - `plan(mesh, faces, p)` → `{"pairs": [(edge, f0, f1, quad, kind, flipped)], "tris": n, "left": [faces]}`; `faces` = faces of `mesh` to consider (only holeless triangles count). `p = dict(DEFAULTS)`: `face_angle` (° max fold), `shape_angle` (° max corner deviation from 90), `planar_tol` (°, ≤ 2.5: flatter pairs merge), `curved` "soft"/"hidden"/"keep", `materials` (pair only same paint), `maximize`.
  - `apply_plan(mesh, pairs)` → `{"merged", "softened", "faces": new quad faces}` — edits `mesh` in place.
  - `grid_plan(faces, gp)` → `{"faces", "full", "border", "overlap", "spacing", "error"}` (`error` None / "empty" / "vertical" / "too_many"); `gp = dict(GRID_DEFAULTS)`: `spacing_m` (m, 0 = auto), `angle` (° about Z; `auto_angle(faces)` fits the site), `clip`, `planar_tol`.
  - `apply_grid(mesh, faces, plan)` → `{"faces", "soft"}` — replaces `faces` in `mesh` by the grid (third argument is the PLAN from `grid_plan`).
  - `scope_faces(scene)` → `(selected faces of scene.mesh or all its faces, from_selection)`.
- Saved data: none in the document. QSettings `plugins/quadrify_tool/<key>` and `…/grid_<key>` (dialog defaults).
- Needs: manifold3d for Grid Remesh (ships with IngeTrazo).
- Pitfalls: `open_dialog`/`open_grid_dialog` are non-modal (no message boxes) but useless headless. `run_quadrify`/`run_grid` wrap `apply_*` in their own `SnapshotImport` (nested undo step) and never report failure — call `apply_*` directly. **Undo:** run_python's snapshot covers `scene.mesh` (loose geometry, or the group currently open for editing) and added groups only; editing the mesh of an existing, closed group is NOT undone — work on `mesh`, or ask the user to open the group first. Grid Remesh needs a height field (`plan["overlap"]` True = steep/overhanging parts).

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

*Live-tested in IngeTrazo 0.5.7.*

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

*Live-tested in IngeTrazo 0.5.7.*

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

*Live-tested in IngeTrazo 0.5.7.*

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

*Live-tested in IngeTrazo 0.5.7.*

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

## 7. Architecture, BIM and structures

### OpenTrace BIM — `arquitetura_parametrica`
0.12.9 · Leandro Lopes · GPL-3.0-or-later · Community (catalogue id `opentrace_bim`)

*Live-tested in IngeTrazo 0.5.7.*

Parametric BIM: straight/curved multilayer walls with hosted openings, polygon slabs (arc edges, holes), columns and beams (simple or catalogue profiles), presets, IFC4 export/import. Bundles Layer Combinations. **In the app:** one "Parametric Architecture" side dock (wall/slab/column/beam/profile/BIM pages) plus its tools; UI text is Portuguese, translated to English at startup.

**From MCP:** Full — every element is built by pure `make_*` functions and inserted by plain `Create*/Edit*` commands whose `do(scene)` can be called directly.

- Module: `ingetrazo_plugin_arquitetura_parametrica` + `.model`/`.commands` (walls), `.slab_model`/`.slab_commands`, `.column_model`/`.column_commands`, `.beam_model`/`.beam_commands`, `.bim`, `.ifc_export`, `.profile_library`, `.preset_catalog`. Load with `importlib.import_module(pkg + '.model')`.
- Entry points (world metres, XY points as `QVector3D`; Z of the points is ignored, the values dict gives Z):
  - `model.make_wall_segment(start, end, values)` → Group. values (all four required): `thickness`, `height`, `base` (bottom Z), `alignment` `'left'|'center'|'right'` (body grows to the LEFT of start→end for 'left'), optional `material_name`, `structure` `'simple'|'composite'` + `layers` [{name, thickness, role 'core'|'finish', material_name}], `openings` [{id, position (m along the path from start, centre), width, sill, height}] (must stay inside the wall, ≥2 cm from ends, above base, below top), `top_mode`/`top_level`/`top_offset` (bind the top to a Níveis level).
  - `model.make_arc_wall(start, end, sagitta, values)` → curved wall; sagitta m, + = bulges left of start→end, |sagitta| ≤ 5×chord. values must ALSO contain a `length` key (any number; recomputed) or it raises.
  - `slab_model.make_slab(world_polygon, values, edge_specs=None, openings=None)` → Group. values: `thickness`, `base_z` (bottom Z), `reference_plane` 'bottom'|'top' (REQUIRED, label only — start from `dict(slab_model.DEFAULTS, …)`), `material_name`, `structure`/`layers`. `edge_specs` per edge `{"type":"arc","sagitta":h}` or `{"type":"line"}`; `openings` [{polygon: [[x,y,0],…] WORLD XY}] fully inside, non-overlapping.
  - `column_model.make_column(point, values)` → Group. values: `profile` 'rect'|'circle', `width`, `depth`, `diameter`, `height`, `base_z`, `rotation` (deg about Z), `anchor` 'tl'…'mc'…'br' (point = that spot of the section), `inclination` (0–85°), `inclination_azimuth`, `curvature`; or `section_type='complex'` + `profile_ref` (id from `profile_library.load_profiles()`).
  - `beam_model.make_beam(start, end, values)` → Group. values: `width`, `height` (section), `base_z` (Z of the axis), `anchor` (default 'mc' = axis through section centre; 'tc' = top at base_z), `inclination` (±85°), `curvature` (plan sagitta), `vertical_curvature`, `rotation`, profile keys as columns.
  - Insert: `commands.CreateWall(g).do(scene)` (`CreateWalls([..])` for several), `slab_commands.CreateSlab`, `column_commands.CreateColumn`, `beam_commands.CreateBeam` — they stamp IFC identity + materials and append to `scene.groups`.
  - Read: `model.read_wall(g)`, `slab_model.read_slab(g)`, `column_model.read_column(g)`, `beam_model.read_beam(g)` → values dict (raise on hand-edited geometry). Edit: `commands.EditWall(scene, g, values)`, `slab_commands.EditSlab(scene, g, values=…, world_polygon=…, edge_specs=…, openings=…)`, `column_commands.EditColumn(scene, g, values)`, `beam_commands.EditBeam(scene, g, values)` (merge into the read values).
  - `bim.scan_scene(scene)` → counts per kind; `ifc_export.export_ifc(app, Path, data=None)` writes IFC4 without dialogs (`app = ExtensionApp(viewport.window(), 'arquitetura_parametrica')`).
- Saved data: per element `group.ext['arquitetura_parametrica']` (`kind` wall/slab/column/beam + all parameters, schema 1/3) and `group.ext['arquitetura_parametrica_derived']` (junction caps), `group.ifc` (class, global_id, predefined_type, psets). Document: `plugin_data['arquitetura_parametrica']` (IFC project/site/storeys, zones, export options), `plugin_data['layer_combinations']`. QSettings: `arquitetura_parametrica/element_presets_v2`, `arquitetura_parametrica/complex_profiles_v1`, `opentrace/bim_library_v1`, `opentrace/composition_library_v1`. Levels come from `plugin_data['niveles']['levels']` ([{name, z}]).
- Needs: nothing extra (Níveis optional; IfcOpenShell optional for import).
- Pitfalls: `run_python`'s snapshot undoes NEW groups but not in-place edits of EXISTING ones — create+edit in the same call with `.do(scene)` directly; to edit a pre-existing element use `viewport.history.execute(EditWall(...))` and `raise` if `viewport.history.last_error`. Do not start from `model.DEFAULTS` without dropping `base_profile`/`top_profile`/`top_xy` (its `top_profile [3,3]` overrides `height`). Columns: `stations` (from `read_column`) override `width/depth` — set `stations=None` when resizing. Walls must not be scaled/rotated off-axis or have their meshes edited (read_* raises). Junction cleanup and level-bound tops are re-derived on the next UI refresh, outside history. `scene.edit_group` must be None. The BIM page, preset import/export, profile editor and updater use dialogs — don't call their UI slots.

```python
import sys, importlib
P = 'ingetrazo_plugin_arquitetura_parametrica'
M, C = (importlib.import_module(P + s) for s in ('.model', '.commands'))
SM, SC = (importlib.import_module(P + s) for s in ('.slab_model', '.slab_commands'))
w = M.make_wall_segment(QVector3D(10, 0, 0), QVector3D(15, 0, 0),
        {'thickness': 0.2, 'height': 2.7, 'base': 0.0, 'alignment': 'left',
         'openings': [{'id': 'w1', 'position': 2.5, 'width': 1.2, 'sill': 0.9, 'height': 1.2}]})
C.CreateWall(w).do(scene)
s = SM.make_slab([QVector3D(10, 0, 0), QVector3D(15, 0, 0), QVector3D(15, 4, 0), QVector3D(10, 4, 0)],
                 dict(SM.DEFAULTS, thickness=0.15, base_z=2.7))
SC.CreateSlab(s).do(scene)
print(M.read_wall(w)['length'], SM.read_slab(s)['thickness'])
```

### ArchXQ IT Lite · Architecture by phases — `archxq_it`
0.9.5 (0.9.5-lite) · Orlando Souza · XQ · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.7.*

Phase-driven architecture workflow (Project/levels → Terrain → Structure → Walls…): every element is a record in one JSON document and the whole building is regenerated from it (walls auto-join, mitre, T/X cuts). Lite draws levels, terrain (plot, excavations), walls (line/arc/circle) and columns; beams, slabs, footings, doors/windows, roofs, ramps, stairs, rooms and sheets are Pro (a Pro file's elements are kept and shown). **In the app:** "⏻ ArchXQ" switch at the top left of the 3D view, then its own bars over the viewport, a levels strip and a side-tray tab (library + outliner + properties); right-click menu entries on ArchXQ objects.

**From MCP:** Partial — no parametric function API, but the document model is pure Python and the live instance's `_commit_all(doc)` rebuilds the building without dialogs; the drawing tools and element windows are UI/modal.

- Module: `ingetrazo_plugin_archxq_it` (live object: `sys.modules['ingetrazo_plugin_archxq_it']._instance`, class `ui.ArchXQ`) + `.model` (doc schema, `load`, `new_walls`, `new_elements`, `elevations`, `add_floor`), `.walls` (pure wall engine, `why_not`), `.structure` (`build`, `levels_info`, `fit_column`, `why_not`), `.compat` (`wall_groups`, `kind_of`, `_rec`, `commit_build`).
- Entry points:
  - `X._live(X.doc())` → the current doc (dict; deleted elements dropped). Levels `doc['levels']` = [{id, name, kind 'basement'|'ground'|'floor', height m}] bottom→top; ground floor default id `'lv0'`, its floor Z = `doc['project']['ground_level']` (0 if no project).
  - Wall record (append via `model.new_walls(doc['walls'], [rec…])`, which adds id + "Wall N"): `{'kind':'line'|'arc'|'circle', 'a':[x,y], 'b':[x,y]` (arc also `'m'`: a point on the curve; circle: `'c'`, `'r'`), `'t'` thickness m, `'align'` 'outside'|'centre'|'inside' (which face sits on the drawn line), `'side'` +1/−1 (inside on the left/right of a→b), `'height'` `'level'` (up to the next slab/level top) or m, `'base'` m above the level floor, `'level'` level id}.
  - Column record (append via `model.new_elements(doc['structure'], [rec…])`): `{'type':'column', 'level', 'x', 'y', 'shape' 'rect'|'round', 'w', 'd' m, 'angle'` deg, `'anchor'` 'centre'|'sw'|'s'|'se'|'e'|'ne'|'n'|'nw'|'w', `'base'` m, `'height'` 'level'|m, `'fit'` bool}. `structure.fit_column(rec, level_walls)` snaps it into a wall.
  - `X._commit_all(doc)` → rebuilds EVERY ArchXQ group from `doc` and stores it (one host undo step). Edits: `X.apply_wall(rec)`, `X.apply_element(rec)`, `X.delete_wall(id)`, `X.delete_element(id)` (each a commit + toast). Levels: `X._save_levels(doc, model.add_floor(doc['levels'], 3.0))`. Plot: `X._plot_done([[x,y],…])` (read, untested).
- Saved data: `plugin_data['archxq']` (NOT the stem: `compat.own_key`; data under `archxq_it` is moved on read) — levels, walls, structure, openings, digs, plot, project, view/hidden state, `built` ids. Groups: `group.ext['archxq']` = {type wall|column|…, id, level, build stamps}; layers `"ArchXQ · <level name>"` and `"ArchXQ · Terrain"`. QSettings `archxq/prefs/*` (floor_height, warn_setbacks…).
- Needs: nothing extra for walls/columns (verified building them without shapely; room/outline helpers import shapely lazily).
- Pitfalls: `_commit_all` runs its own history command inside `run_python` → TWO undo steps (undo both; one alone leaves records whose groups are gone, which ArchXQ then treats as deleted). Check `viewport.history.last_error` after it and `raise`. Never edit ArchXQ meshes directly — every commit regenerates them (it does adopt host Move/Rotate commands via a global `History.execute` patch that runs after every command). Avoid `_walls_done`/`_struct_done` (need the UI's current level/options; walls can raise a setbacks `QMessageBox`), `edit_wall`/`edit_element`/`edit_project` (modal), `on_add_floor` without a project (opens a dialog). Do not author Pro-only record types in the Lite.

```python
import sys, importlib
P = 'ingetrazo_plugin_archxq_it'
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

### CAD2IngeTrazo · CAD plans to BIM — `cad2ingetrazo`
3.11 · M. Tariq · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.7.*

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

### planArq — `planarq`
1.0 · José Castro Basso (FADU – UDELAR) · GPL-3.0-or-later · Reviewed

*Live-tested in IngeTrazo 0.5.7.*

Walls, pipes and steel/timber profiles (IPN, IPE, UPN, rectangular/circular tube, PGC steel framing, timber) swept along an editable polyline axis; axis and parameters are stored so the element can be re-edited. Schematic, pre-sizing only. **In the app:** side-tray tab "planArq" (Spanish UI: draw wall / pipe / profile, edit nodes, apply parameters to the selection).

**From MCP:** Full — an element is one record dict; the plugin's `_Create` command builds it exactly as the draw tool does.

- Module: `ingetrazo_plugin_planarq` (single file; the data key is `"planarq"`).
- Entry points:
  - `_Create(key, group, record)` — a history Command: run `viewport.history.execute(_Create('planarq', Group(name=…), record))`; builds the mesh, appends the group, stores the record, selects the group.
  - `record` (metres, world coords, Z up): `{"type": "wall"|"pipe"|"profile", "nodes": [[x,y,z], …] (≥2, the axis), …}`; wall: `"width", "height", "alignment": 0 left / 1 centre / 2 right of the axis` (wall axis must advance in plan); pipe: `"diameter"` (outer), `"wall"` (thickness, UI uses 8 % of Ø), `"color"` (a key of `COLORS`: Celeste, Azul, Naranja, Rojo…); profile: `"profile_data": {"family", "name", "tipo_forma", "dimensions" (mm, from the catalogue)}, "rotation": 0|90`. UI also stores `"level"`, `"fixed_level"` (not used by the build).
  - Catalogue: `PROFILE_CATALOG` (perfiles.json), `_family_data(key)` → `{"tipo_forma", "items": {name: dims}, "IMAGEN"}`; keys IPN, IPE, UPN, Tubo_Rectangular, Tubo_Circular, Steel_Framing_PGC, escuadrias_madera.
  - `_build(group, record)` — pure: sets `group.mesh` and `group.ifc` (IfcWall / IfcPipeSegment / IfcMember with `path_length`), raises `ValueError` (Spanish text) on bad input.
  - `_Rebuild(key, group, before, after)` / `_BatchParameters(key, [(group, before, after)])` — change an existing element's record.
  - `_records(scene, 'planarq')` → `{group.uid: record}`.
- Saved data: `scene.plugin_data["planarq"]["elements"]` = `{group uid: record}`.
- Needs: nothing extra (perfiles.json + perfiles/*.jpg beside the plugin).
- Pitfalls: `_Create`/`_Rebuild` are nested undo steps and fail silently — check `viewport.history.last_error` and `raise`. A bare `_build` + `groups.append` makes geometry the panel cannot edit (no record), and plugin_data written outside a command is not rolled back by `run_python`. Two coincident consecutive nodes or a 180° reversal fail the sweep. No modal dialogs in this plugin.

```python
import sys
P = sys.modules['ingetrazo_plugin_planarq']
g = Group(name='Muro')
viewport.history.execute(P._Create('planarq', g, {
    'type': 'wall', 'nodes': [[10, 0, 0], [14, 0, 0], [14, 3, 0]],
    'width': 0.20, 'height': 2.5, 'alignment': 1}))
if viewport.history.last_error: raise RuntimeError(viewport.history.last_error)
print(g.ifc['class'], len(g.mesh.faces))
```

### Windowizer — `windowizer`
0.5.7 (ships with the app) · Bane Andreev (port of Rick Wilson's Windowizer 3) · GPL-3.0-or-later · Bundled example

*Live-tested in IngeTrazo 0.5.7.*

Turns a face drawn on a wall into a parametric window: one container group (`IfcWindow`) holding a Frame and Glass panes (rows × columns), and opens the wall (through to its back face, reveals, or just a hole). Edit rebuilds it in place from its stored parameters; Erase removes it and closes the wall. **In the app:** Extensions ▸ Windowizer ▸ Windowize selected faces / Edit window… / Inherit settings / Erase window, and the same in the right-click menu.

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

*Live-tested in IngeTrazo 0.5.7.*

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

*Live-tested in IngeTrazo 0.5.7.*

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

*Live-tested in IngeTrazo 0.5.7.*

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
1.0 · Pesi (pesi3d.de) · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.7.*

Builds roofs on a footprint (20 shapes: gable, hip, half-hip, mansard, butterfly, dome …, with gable walls and a slab) and covers sloped faces with real 3D tiles (verge, ridge/hip tiles, end caps, cut tiles, flashing). **In the app:** Extensions ▸ Roof Tiles ▸ Create Roof… / Edit Roof Shape… / Cover Roof… / Edit Roof Tiles…, its own toolbar, and the right-click menu.

**From MCP:** Full — both builders are pure functions that return a ready `Group`; append it yourself.

- Module: `ingetrazo_plugin_roof_tile_tool` (single file)
- Entry points:
  - `make_shape(fp, p, name="Roof")` → `(Group, info)`. `fp = {"outline": [(x, y), …], "z0": z}` (m, footprint at wall-top height, any winding); `p` from `gen_default_params()` with `type` (one of `ROOF_TYPES` keys: gable, hip, halfhip, dutch, pyramid, mono, split, saltbox, mansard, gambrel, bonnet, butterfly, mroof, sawtooth, barrel, spire, dome, helm, aframe, flat), `pitch`/`pitch2` (°), `overhang`, `gable_ov`, `thick` (m), `walls`, `rotate`, `auto`/`gables` (edge indices), `eave`. `info`: faces, walls, height. Lower level: `build_roof(outline, z0, rtype, p)` → `(roof_polys, wall_polys)`.
  - `gather(obj)` → `{"faces": [{"outer": [(x,y,z)…], "holes": […]}], "shapes": [uids]}` from `obj.selection` (any object with a `selection` set: faces or groups; takes a group's upward sloped faces).
  - `make_roof(data, p, name="Roof Tiles")` → `(Group, info)`; `p` from `default_params()`: `preset` (`PRESET_BY_ID`: frankfurt, harz, toscana, flat, marseille, biber, monk, pantile, plain_uk, spanish), `W`, `Lmin`, `Lmax` (m), `detail` low/medium/high, `color` (name in `COLORS`), `verge`, `ridge`/`hips`/`endcaps`/`flashing` bools, gaps (m). `info`: tiles, cut, verge, ridge, hip, mono, gauge… `result_text(info)` gives a one-line summary. `estimate(data, p)` is a fast count without cutting.
  - `materials_of(p)` → Material list; the plugin's command registers them with `core.materials.register(scene.materials, m)` (optional: faces already carry `attrs["color"]`/`["mat"]`; run_python's undo does not remove registry entries).
- Saved data: `group.ext["roof_tile_tool"]` (shape: `kind="roof_shape"`, params, outline, z0; covering: params, faces, shape uids). QSettings `plugins/roof_tile_tool/params` and `/shape` (dialog defaults only).
- Needs: numpy and manifold3d (both ship with IngeTrazo).
- Pitfalls: `show_*` functions open non-modal dialogs that use `QMessageBox` on errors and `QFileDialog` for custom tiles — never call them. `run_roof`/`run_shape` execute their own command (nested undo step) but raise on `history.last_error`. `tile_spec` uses `p["W"]/Lmin/Lmax`, not the preset's: copy them from `PRESET_BY_ID[id]` when changing preset. Raises `RoofError` above 40 000 pieces (`p["force"]=True` overrides). Big roofs with `detail="high"` are slow (120 s limit). Whole tiles are component instances in `g.children`.

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

*Live-tested in IngeTrazo 0.5.7.*

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

*Live-tested in IngeTrazo 0.5.7.*

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
1.1 · Pesi (pesi3d.de) · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.7.*

Builds a terrain group from contour lines (regular grid, harmonic interpolation, optional quads) or from points (Delaunay TIN), with optional skirt, closed bottom, embedded contours and a flat contour map. **In the app:** Extensions ▸ Topology ▸ Terrain from Contours… / Terrain from Points… / Edit Terrain…, its own toolbar, and the right-click menu (plus an Altitude Editor inside the dialog).

**From MCP:** Full — `make_terrain` is pure and returns a ready `Group`; append it yourself.

- Module: `ingetrazo_plugin_topology_tool` (single file)
- Entry points:
  - `make_terrain(data, p, name="Terrain")` → `(Group, (faces, seconds, "triangles"|"quads"))`. `data = {"mode": "contours", "segs": [(x0,y0,z0,x1,y1,z1), …]}` (contours already at their heights, m) or `{"mode": "points", "points": [(x,y,z), …]}` (≥ 3, not collinear). `p` from `default_params()`: `res_mode` "cells"/"size", `cells` (on the long side, 10–2000), `cell_size` (m), `boundary` "hull"/"box"/"cut", `peaks` "round"/"flat", `skirt`, `skirt_depth` (m below the lowest point), `bottom`, `soft`, `quads`, `smooth_r` (m), `max_edge` (m, points: trim long rim triangles), `contours`, `map`.
  - `gather_segments(obj)` / `gather_points(obj)` → source data from `obj.selection` (edges, faces, groups, guide points; points fall back to the document's survey points). `gather_points` returns `(points, label)`.
  - `estimate_triangles(segs, p)` → `(n, (h, nx, ny))`; `terrain_from_contours(segs, p)` → `(V, T, C)` numpy arrays, `terrain_from_points(pts, p)` → `(V, T)` if you want the raw surface.
  - Contour clean-up: `cs = ContourSet.from_segs(segs); cs.set_clean(simplify_m, smooth_passes, min_len_m); segs = cs.segs()`; pass `"cset": cs` in `data` to store it.
- Saved data: `group.ext["topology_tool"]` (`mode`, params, segs or points, lines); legacy keys `terrain_tool`, `topoform`. QSettings `plugins/topology_tool/params` (dialog defaults).
- Needs: numpy (ships with IngeTrazo).
- Pitfalls: `show_*` open non-modal dialogs that use `QMessageBox` on errors — avoid them. `run_terrain(viewport, data, p, old=None)` executes its own command (nested undo step) but raises on `history.last_error`. Raises `TopoError` if all contours share one height or the estimate exceeds 1 000 000 triangles (`p["force"]=True` overrides). The grid is laid over the contours' XY bounds; keep `cells` modest (≤ 300) to stay well inside 120 s.

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

*Live-tested in IngeTrazo 0.5.7.*

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

## 9. Fabrication

### Cut List — `cutlist`
0.0.1 · Ronaldo Alves · GPL-3.0-or-later · GitHub, not catalogued (ronaldogalves/cutlist-ingetrazo @ 6a9166e)

*Live-tested in IngeTrazo 0.5.7.*

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

## 10. Viewing, rendering and organising

### Point Cloud & 3D Gaussian Splatting — `pointcloud_splat`
1.0.0 · archimades27 · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.7.*

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

*Live-tested in IngeTrazo 0.5.7.*

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

### Styles, Shadows and Map toolbars — `igz_tb_toolbar`
1.5.2 · Ezequiel M. Rezende · GPL-3.0-or-later · Community (catalogue id `igz_toolbars`)

*Live-tested in IngeTrazo 0.5.7.*

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

*Live-tested in IngeTrazo 0.5.7.*

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

### Phasing Animator for IngeTrazo — `phasing_animator`
1.0.0 · aashishzharbade-arch · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.7.*

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

*Live-tested in IngeTrazo 0.5.7.*

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

*Live-tested in IngeTrazo 0.5.7.*

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

### MultiView for IngeTrazo — `ingetrazo_multiview`
1.0.5 · aashishzharbade-arch · GPL-3.0-or-later · Community

*Live-tested in IngeTrazo 0.5.7.*

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

### JA Tree · Model outliner — `outliner`
0.1.0 · Julio Angulo · GPL-3.0-or-later · Community (catalogue id `ja_tree`)

*Live-tested in IngeTrazo 0.5.7.*

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

*Live-tested in IngeTrazo 0.5.7.*

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

*Live-tested in IngeTrazo 0.5.7.*

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

*Live-tested in IngeTrazo 0.5.7.*

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
