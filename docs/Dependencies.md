# Dependencies

This page lists all dependencies required by the Houdini ComfyUI Bridge, beyond Houdini and ComfyUI themselves (see [Installation 1](Installation_1_Houdini_and_ComfyUI.md) for those).

## Houdini-side Python dependencies

These packages must be available in Houdini's Python 3.11 environment.

| Package | Required | Purpose |
|---------|----------|---------|
| `requests` | Yes | HTTP communication with the ComfyUI server (`graph_submission.py`) |
| `Pillow` (`PIL`) | Yes | Reading PNG metadata when drag-and-dropping ComfyUI-generated images (`scripts/externaldragdrop.py`) |

### Installing into Houdini's Python

Houdini ships with its own Python interpreter. Install packages targeting it directly:

```bash
# Windows (adjust path to your Houdini install)
"C:\Program Files\Side Effects Software\Houdini 20.5.613\bin\hython.exe" -m pip install requests Pillow

# Linux/macOS
hython -m pip install requests Pillow
```

> **Note:** If you run Houdini with a custom Python environment, install the packages there instead.

## ComfyUI-side Python dependencies

| Package | Required | Purpose |
|---------|----------|---------|
| `aiohttp` | Yes (bundled with ComfyUI) | HTTP routes for the `/sidefx_houdini/*` endpoints (`src/houconnect/routes.py`) |
| `trimesh` | Optional | UV unwrapping of generated meshes via the `HouCuiTrimeshUnwrapProperly` node (`src/houconnect/nodes.py`). When missing, the node logs a warning and acts as a no-op (passes the mesh through unchanged). |

### Installing trimesh into ComfyUI

If you use the 3D / mesh generation features (e.g. `lop_xxx.comfyui_hy3d_generate_asset`):

```bash
# Inside your ComfyUI Python environment
pip install trimesh
```

For ComfyUI Desktop on Windows, the environment is managed by the app. Use the integrated terminal or the Manager's pip installer if available.

## ComfyUI custom node dependencies

Some features rely on third-party ComfyUI node packs that are **not** bundled with this plugin. Install them through the [ComfyUI Manager](https://github.com/ltdrdata/ComfyUI-Manager).

| Node pack | Required for | Provides |
|-----------|--------------|----------|
| Hy3D Wrapper (Hunyuan3D) | 3D asset generation (`lop_xxx.comfyui_hy3d_generate_asset`) | `Hy3DExportMesh` and related Hunyuan3D nodes |

> **How to install:** Open ComfyUI → Manager → Custom Nodes Manager → search for `Hy3D` or `Hunyuan3D` → install → restart ComfyUI.

## Verifying the installation

After installing all dependencies, verify by:

1. **Houdini side** — In a COP context, create a `Comfyui compound graph submit` node and press **Update Node Definitions**. No import errors should appear.
2. **ComfyUI side** — In the ComfyUI node editor, double-click and search for `unwrap`. The `Trimesh UV Unwrap` node should appear (confirms both this plugin and `trimesh` are loaded). See [First Steps 1](First_steps_1.md).

## Next steps

- [Installation 2 — Plugin](Installation_2_Plugin.md)
- [First Steps 1](First_steps_1.md)
