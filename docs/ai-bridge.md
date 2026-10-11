# AI Bridge — driving IngeTrazo from an AI agent (MCP)

An AI agent (Claude Code, Claude Desktop, Cursor, Codex CLI, Antigravity CLI,
any [MCP](https://modelcontextprotocol.io) client) can model in the **live,
open** IngeTrazo document: build, query, look at the result and iterate. Every
agent action is one undo step, and an action that fails is rolled back whole.

This page is the complete guide for the agent and for the person setting it
up. The recipe book the agent receives automatically lives in
`core/ai_recipes.py` (the single source for both AI doors); this page
summarises it and adds what it leaves out. Using installed extensions
(plugins) from the bridge: [ai-extensions.md](ai-extensions.md).

## 1. Connect

**Start the bridge** in IngeTrazo: side tray ▸ **AI** tab ▸ **AI bridge
(MCP)** ▸ **Start bridge** (or **Extensions ▸ AI Bridge (MCP)**). It listens on
`127.0.0.1:4763` only (`INGETRAZO_AI_PORT` changes the port), serves **one
client at a time**, and only answers while IngeTrazo is open with the bridge
on. Once started, the section shows the exact connect lines for your system
with a **Copy** button.

**Register the MCP server** (it ships inside the app; no Python needed):

| Installation | MCP server command |
|---|---|
| Windows (installer or zip) | `C:\Program Files\IngeTrazo\ingetrazo-mcp.exe` |
| Linux tarball / PyInstaller build | `/path/to/ingetrazo/ingetrazo-mcp` (or `<ingetrazo> --mcp`) |
| AppImage / Flatpak / snap | `<AppImage> --mcp` / `flatpak run <id> --mcp` / `/snap/bin/<name> --mcp` |
| From a checkout | `python3 /path/to/app/scripts/ingetrazo_mcp.py` |

    claude mcp add ingetrazo -- "/path/to/ingetrazo-mcp"          # Claude Code

    {"mcpServers": {"ingetrazo": {"command": "/path/to/ingetrazo-mcp", "args": []}}}
    # Claude Desktop: %APPDATA%\Claude\claude_desktop_config.json (Windows),
    # ~/.config/Claude/claude_desktop_config.json (Linux); restart it after editing.

**Agent and app on different machines or in containers.** The bridge has no
authentication and `run_python` executes code inside the app, so it never
listens beyond loopback — do not publish or port-forward it to a network.
Instead, either

- run the MCP server **where the app runs**, over a channel you already
  trust — e.g. IngeTrazo in Docker: `ssh -T host docker exec -i <container>
  /opt/ingetrazo/ingetrazo-mcp` as the MCP command; or
- tunnel the port yourself (`ssh -L 4763:127.0.0.1:4763 user@host`, `socat`…)
  and point the client at the tunnel's end:
  `INGETRAZO_AI_HOST=host.docker.internal INGETRAZO_AI_PORT=4763 ingetrazo-mcp`.
  `INGETRAZO_AI_HOST` (default `127.0.0.1`) only changes where the *client*
  connects.

**No MCP client?** Talk to the bridge directly — newline-delimited JSON over
the TCP port, one request per line:

    {"id": 1, "tool": "run_python", "args": {"code": "print(len(groups))"}}
    → {"id": 1, "ok": true, "result": {"stdout": "1\n", "stderr": "", "error": null, "changed": false}}

A 10-line stdlib client (`socket.create_connection(("127.0.0.1", 4763))`,
send one line, read until `\n`) is enough; run it on the app's machine (or
inside its container with `docker exec -i`). Screenshots come back as a file
**path on the app's machine** — copy the PNG out yourself.

**If nothing answers:** IngeTrazo closed or bridge stopped (the section must
say "Listening on…"), wrong command path, another client already connected,
or (Claude Desktop) the server shows an error under Settings ▸ Developer ▸ MCP.

## 2. Tools

| Tool | Arguments | Returns |
|---|---|---|
| `run_python` | `code` | `{stdout, stderr, error, changed}` — one undo step; on an exception the **whole call is rolled back** (`error` set, document unchanged); no undo entry if nothing changed. |
| `query_model` | — | Face/edge counts, group and component names, materials, layers, dimensions, section planes, selection size, bounds. Start here. |
| `screenshot` | `width`, `height` (default 1024×768, max 4096) | A render of the live viewport (an image over MCP; a PNG path over the raw bridge). |
| `undo` / `redo` | — | `{ok}` — the user's own history. |

Each tool call runs on the app's UI thread with a **120 s limit**.

## 3. The modelling contract

- **Z up, metres, radians.** The document unit setting (`scene.units`) only
  affects display.
- **Scope** (persists between calls — define helpers once, reuse them):
  `scene` (alias `model`), `mesh` (loose geometry), `selection`, `groups`
  (= `scene.groups`), `layers`, `viewport`, `QVector3D`, `Mesh`, `Group`,
  `Edge`, `Face`, `bim`, and the builders `revolve`, `extrude`, `prism`,
  `wall`, `house`. A call that is a single expression prints its value (and
  stores it in `_`).
- **Do not explore the API** with `dir()`/`inspect`: the recipes below are
  what you need; if something is missing, write it and read the error.
- A **new document contains a scale figure** ("Engineer"/"Ingeniero", 1.70 m
  billboard). It is not the model; leave it alone unless asked.
- Simple requests go in **one complete call**; big ones in steps. Verify every
  step with screenshots.

## 4. Builders and recipes

Each builder creates one solid, correctly wound **named group**, adds it to the
document and returns it (`Group`). `color` is `(r, g, b[, a])` in 0..1.

| Builder | Use for |
|---|---|
| `revolve([(r, z), …], segments=32, scallop=None, closed=False, name=, color=)` | Anything round (lathe around Z, profile bottom→top; open profiles are capped; `closed=True` for a closed section such as a cup wall; `scallop=(depth, lobes)` carves flutes). |
| `extrude([(x, y), …], z0, z1, name=, color=)` | Any box or plan-shaped prism — a board, leg, rail or step is a 4-point extrude, never face by face. |
| `prism([3D points of a flat polygon], (dx, dy, dz), holes=[[…]], name=, color=)` | A flat polygon (with holes) swept along **any** vector: slabs, tilted panels, frames (outer loop + hole), slanted members. |
| `wall((x, y), (x, y), height=3, thickness=0.2, openings=[(offset, sill, width, height)], peak=None)` | Walls; they grow to the LEFT of a→b; sill 0 = door, >0 = window. |
| `house(width, depth, wall_height, thickness, roof="gable"\|"hip"\|"flat", ridge_height, overhang, ridge="x", doors=[(side, offset, w, h)], windows=[(side, offset, sill, w, h)], origin, name)` | A whole house; sides S (front, y=y0), E, N, W; offsets counter-clockwise from the side's first corner. |

Lower level:

- Loose face: `f = mesh.add_face([QVector3D(…), …])` — loop counter-clockwise
  seen from outside; `f.attrs["color"] = (r, g, b, 1.0)`. Edge:
  `mesh.add_edge(a, b)`.
- Hand-built group: `m = Mesh(); m.add_face([...]); …; g = Group(m,
  name="…"); groups.append(g)`.
- Paint a whole group: `g.material = {"color": (r, g, b), "opacity": 1.0}` —
  do not touch `scene.materials` (the document's material registry).
- Move a group: `g.mesh.move_vertex(v, QVector3D(dx, dy, dz))` for each
  `v in list(g.mesh.vertices)`; for a component instance (`g.xform is not
  None`) compose the translation into `g.xform` instead.
- `print()` what you need to know — briefly; output travels every turn.

### Freeform and curved solids

`prism`/`extrude` sweep **flat** polygons. For a curved surface (a dished
seat, a bent panel) build a closed `Mesh` yourself:

- top surface as a grid or concentric rings of **triangles** (quads that are
  not planar must be split), bottom surface offset and **reversed**, vertical
  quads around the rim; every loop counter-clockwise seen from outside;
- **sample the outline densely** (every 1–2 cm): a long straight edge with no
  vertices flattens the surface between its ends;
- set `f.attrs["color"]` on every face, then mark interior edges smooth with
  `e.soft = True` (keep the rim edges hard), then `groups.append(Group(m,
  name=…))`.

Hand-built meshes are **not** validated by the engine (the hermeticity guard
belongs to the interactive tools) — keep them closed and consistently wound;
the Solid Inspector plugin can check them.

### Round or slanted members

A leg is `prism(circle_in_a_horizontal_plane, leg_vector)` — floor-parallel end
cuts, like real joinery. For **shallow** members (rails, braces) that would
flatten into a sheet, build the circle in the plane **perpendicular** to the
member's axis instead.

## 5. Camera, rendering, saving, exporting

- **Camera** `viewport.camera`: `target`, `distance`, `yaw`, `pitch`
  (radians), `look_from(eye, direction)`, `fit_box(min, max, margin=1.1)`,
  `set_view("top"|"bottom"|"front"|"back"|"left"|"right"|"iso")`,
  `turn(dyaw_deg, dpitch_deg)`, `zoom(steps)`, `toggle_projection()`,
  `set_aspect(w, h)`; then `viewport.update()`.
- **Several views in one call:** for each view set the camera, call
  `cam.set_aspect(w, h)`, then `viewport.render_image(w, h).save(path)`.
  Camera moves do not create undo steps.
- **Save the user's document** like File ▸ Save does (backup `.bak`, clean
  state, title, recent files): `viewport.window()._do_save(Path("….igz"))`
  (private API). `formats.igz.save_scene(scene, path)` only writes a copy —
  the document stays "unsaved". Paths are on the app's machine; in Docker,
  only the mounted folders survive.
- **Export** (import inside the call): `formats.gltf.save_glb(scene, path)`,
  `formats.obj.save_obj`, `formats.stl.save_stl`, `formats.dae.save_dae`,
  `formats.ifc.save_ifc`, `formats.skp_out.save_skp`,
  `formats.dxf_out.save_dxf_layers(path, groups)`. glTF is written Y-up.
- **Open / import:** `viewport.window().open_path(Path(...))`,
  `viewport.window().import_igz_path(path, at=None)`,
  `formats.<fmt>.load_<fmt>(scene, path)` for obj/stl/dae/dxf/glb/skp.

## 6. A workflow that works

1. `query_model`; screenshot the starting state.
2. Keep the model as **one parametric script outside the app** (every
   dimension derived from a few named numbers). Build it in one `run_python`
   call.
3. Render several views (front ¾, side, back, a low view) and compare them
   with the reference (photos, drawings, a scan).
4. To change it, `undo` the build and run the corrected script — never patch
   geometry incrementally.
5. Measure, do not guess: real sizes come from the user (a photo has no
   scale). State every assumed dimension so the user can correct it.
6. Save with `_do_save` and export what the user needs.

## 7. Pitfalls (each one has cost a real session)

- **Squashed screenshots.** `screenshot` passes an explicit width *and*
  height, so the image keeps the camera's aspect — that of the last browser
  or window that showed the viewport (a phone in portrait gives 0.42). Call
  `viewport.camera.set_aspect(w, h)` in `run_python` first, with the same
  numbers.
- **`scene.bounds()` is cached per document version** — inside the call that
  just added geometry it can be stale (`(None, None)` on a new document) and
  `lo.x()` raises, **rolling back the whole build**. Compute bounds from your
  own coordinates, or query after the call.
- **Any exception rolls back everything**, including a debug `print` at the
  end of a long build. Keep diagnostics safe.
- **…but "everything" has limits.** A step restores the active mesh and
  removes groups, layers, views, dimensions and labels it *added*. It does not
  undo in-place edits to existing groups (mesh, transform, `hidden`,
  children), removed groups, `scene.plugin_data`, materials or display
  settings. Change those through a `Command` (`viewport.history.execute`) —
  and since a failing nested command is rolled back silently (the reply says
  `"error": null`), check `viewport.history.last_error` right after it.
  Details: [ai-extensions.md §3](ai-extensions.md#3-rules-that-keep-the-bridge-safe).
- **Modal dialogs block the bridge.** UI paths that ask the user something
  (`open_path` with unsaved changes, a failed save) freeze the UI thread: the
  call times out after 120 s and the dialog stays on the user's screen.
  Prefer the `formats.*` functions.
- **Selkies/browser desktops:** the user may be connected from a phone;
  the viewport size and aspect then follow that client.
- **One client at a time** — a second MCP client cannot connect until the
  first disconnects.

## 8. Security

The bridge is unauthenticated remote code execution inside the app (and the
user's account). It binds to `127.0.0.1` on purpose: never expose it, and
reach it only through channels you already trust (local process, SSH,
`docker exec`). Code run there can read anything the app's user can read,
including secrets in its environment.

## 9. The in-app assistant

For users without an agent of their own: the side tray's **AI** tab
(Ctrl+Shift+A) is a chat inside IngeTrazo that writes the same transactional
recipes (same `core/ai_recipes.py`). Paste an API key — the provider is
detected from its prefix:

| Prefix | Provider | Default model |
|---|---|---|
| `sk-ant-` | Anthropic | claude-sonnet-5 |
| `gsk_` | Groq (free) | llama-3.3-70b-versatile |
| `sk-or-` | OpenRouter | anthropic/claude-sonnet-5 |
| `AIza` | Gemini | gemini-2.5-flash |
| `sk-` | OpenAI | gpt-4o |
| *(empty)* | local Ollama | llama3.2 |

With a vision-capable provider it also receives viewport captures to check its
work, and **Photo…** attaches a reference image (rescaled to 1280 px, sent only
with that message) to model an object part by part. Give real measurements in
the message — a photo carries no scale — and correct any dimension the
assistant declares as assumed. The key is stored in the local settings.
