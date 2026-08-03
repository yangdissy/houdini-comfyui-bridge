# Architecture

How the Houdini ComfyUI Bridge works under the hood. Useful for contributors and advanced users.

## Two-sided design

The plugin runs on **both** Houdini and ComfyUI:

```
┌─────────────── Houdini side ───────────────┐         ┌─────────────── ComfyUI side ──────────────┐
│  HDA nodes (COP / LOP / TOP)               │         │  PromptServer                              │
│    ├─ compound_graph_submit (main)         │  HTTP   │    ├─ /sidefx_houdini/* routes             │
│    ├─ partial_graph (node wrapper)         │ ──────> │    ├─ HouCui* helper nodes (7)             │
│    └─ geo_uploader, range_to_batch, ...    │ <────── │    └─ web/js/listener.js (message poller)  │
│  Python connection library (9 modules)     │         │                                            │
└────────────────────────────────────────────┘         └────────────────────────────────────────────┘
```

- **Houdini side** — `houdini/python3.11libs/houdini_comfyui_connection/`: builds, merges, submits, and downloads graphs.
- **ComfyUI side** — `src/houconnect/`: provides helper nodes, custom HTTP routes, and a front-end message listener.

## The compound graph mechanism

The core idea: **a Houdini COP network is compiled into a single ComfyUI graph at cook time.**

### Build flow (on Compute)

1. **Traverse** — Starting from the output node, walk backwards along COP wire connections. The traversal penetrates `subnet`, `null`, bypassed, and `switch` nodes (`compound_graph_core_graph_helpers.py`).
2. **Per-node fragments** — Each `comfyui_partial_graph` node contributes a small ComfyUI graph fragment (one node, stored as JSON in its `cui_graph` parameter).
3. **Non-graph sources** — Ordinary COP nodes (images, geometry) are treated as upload sources: they generate a `LoadImage` (or `ImageToMask`) node and are queued for upload.
4. **Merge** — All fragments are combined into one graph. Each fragment is given a unique key prefix (`<offset>_<original_key>`) to avoid node-key collisions, and all connection references are rewritten (`combine_graph_parts` in `compound_graph_core.py`).
5. **Parameter injection** — Values promoted to subnet parameters are written back into the graph as `param_overrides` (`replace_params_in_graph_by_key`).
6. **Submit** — The combined graph is sent to ComfyUI via `POST /prompt`.
7. **Poll & download** — Houdini polls `GET /history/{prompt_id}` until the job completes, then downloads each output via `GET /view`.

### Output type → save node

The save node appended to the graph depends on the output's declared type:

| Output type | ComfyUI save nodes generated |
|-------------|------------------------------|
| `IMAGE` / (empty) | `SaveImage` |
| `MASK` | `MaskToImage` → `SaveImage` |
| `TRIMESH` (Hunyuan3D) | `Hy3DExportMesh` → `HouCuiStringAsImage` |
| `MESH` | `SaveGLB` |
| `STRING` | `HouCuiStringAsImage` |
| `VIDEO` | `SaveVideo` |

### Recursive submission

`comfyui_partial_graph_outputs` can submit an inner sub-graph as a **separate** ComfyUI prompt, then inject the results into the outer graph as string variables (`@{{var}}`). This enables multi-stage pipelines where one ComfyUI job's output feeds another.

## HTTP API endpoints

### Standard ComfyUI endpoints (used by the Houdini side)

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `POST` | `/prompt` | Submit a graph for execution |
| `GET` | `/queue` | Query the execution queue |
| `GET` | `/history/{prompt_id}` | Query job status / results |
| `GET` | `/view` | Download a result file |
| `POST` | `/upload/image` | Upload an input image |
| `GET` | `/object_info` / `/object_info/{type}` | Fetch ComfyUI node definitions |
| `POST` | `/history` | Delete prompt history |

### Custom endpoints (provided by this plugin's ComfyUI side)

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `POST` | `/sidefx_houdini/messages/get` | Poll for messages from Houdini (consumed by `listener.js`) |
| `DELETE` | `/sidefx_houdini/image` | Delete an uploaded input image |
| `POST` | `/sidefx_houdini/interrupt` | Interrupt a running prompt |

## ComfyUI-side helper nodes

Registered in `src/houconnect/nodes.py`:

| Node class | Display name | Purpose |
|------------|--------------|---------|
| `HouCuiStringAsImage` | String To Image Helper | Treat a string (file path) as an image output |
| `HouCuiTrimeshUnwrapProperly` | Trimesh UV Unwrap | UV-unwrap a trimesh (requires `trimesh`) |
| `HouCuiInputPathToAbsolute` | Input Path To Absolute | Expand a relative input path to absolute |
| `HouCuiCopyInputToOutput` | Copy Input to Output | Copy a file from input dir to output dir |
| `HouStringPassThrough` | String Pass Through | No-op string passthrough |
| `HouStringToFile` | String Save | Save a string to a text file in the output dir |
| `HouCuiFixImageFix` | (image) | Normalize image tensor shape/dimensions |

## Front-end message listener

`web/js/listener.js` runs inside the ComfyUI web UI. It polls `POST /sidefx_houdini/messages/get` every 2 seconds and handles three commands from Houdini:

| Command | Action |
|---------|--------|
| `create_loader` | Create a `LoadImage` node on the ComfyUI canvas with the given image name |
| `refresh_all` | Force-refresh all `LoadImage` nodes whose filename starts with `imgtemp` |
| `refresh` | Force-refresh a specific `LoadImage` node |

This is how Houdini-side uploads trigger live preview updates in the ComfyUI UI.

## Drag-and-drop import

`houdini/scripts/externaldragdrop.py` implements Houdini's global drag-drop callback. When you drag a ComfyUI-generated PNG (or a workflow JSON) onto the network editor **inside** a `comfyui_compound_graph_submit` node, it:

1. Reads the `prompt` (API format) and `workflow` (UI format) metadata embedded in the PNG.
2. Reconstructs the ComfyUI graph as a Houdini COP network via `create_network_from_prompt`.
3. Connects `SaveImage` nodes to the subnet output, and `PreviewImage` nodes to `comfyui_graph_preview`.
4. Marks input/output nodes in red and prompts the user to verify them.

## Key source files

### Houdini side — `houdini/python3.11libs/houdini_comfyui_connection/`

| File | Responsibility |
|------|----------------|
| `graph_submission.py` | HTTP API communication (submit / poll / download / delete) |
| `compound_graph_core.py` | Compound graph core: build, merge, upload, submit |
| `compound_graph_core_graph_helpers.py` | Input/output connection tracing (through subnet/null/switch) |
| `compound_graph_tools.py` | Node-definition sync, toolbar generation, subnet wrapping |
| `workflow_deserialization_tools.py` | Deserialize ComfyUI workflow (PNG/JSON) into a Houdini network |
| `subnet_wrapper_helper.py` | Promote parameters to the subnet wrapper |
| `node_data.py` | Recursive userdata read/write (through locked HDAs) |
| `upload_common.py` | Generic image upload |
| `ui_tools.py` | Error dialogs |

### ComfyUI side — `src/houconnect/`

| File | Responsibility |
|------|----------------|
| `__init__.py` | Extension registration; declares `WEB_DIRECTORY` |
| `routes.py` | `/sidefx_houdini/*` routes (message bus, image delete, interrupt) |
| `nodes.py` | The 7 `HouCui*` helper nodes |

### Other

| File | Responsibility |
|------|----------------|
| `houdini/scripts/externaldragdrop.py` | Drag-and-drop workflow import |
| `houdini/PARMmenu.xml` | Right-click menu items inside compound graphs |
| `web/js/listener.js` | ComfyUI-side message listener |
| `tools/prune_hdas.py` | HDA version pruning tool for release packaging |
| `otl-deploy-rules` | JSON rules consumed by `prune_hdas.py` |
