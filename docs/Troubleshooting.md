# Troubleshooting

Common issues and how to resolve them.

## Installation & connection

### "Update Node Definitions" fails with a connection error

**Symptom:** Pressing **Update Node Definitions** on a compound graph node shows an error mentioning a connection failure or timeout.

**Cause:** Houdini cannot reach the ComfyUI server.

**Fix:**
1. Confirm ComfyUI is running.
2. Check the `base_url` parameter on the Houdini node points to the correct address (default `http://127.0.0.1:8188`).
3. If ComfyUI runs on another machine, ensure it listens on `0.0.0.0` or an external interface (see [Installation 1](Installation_1_Houdini_and_ComfyUI.md)), and that the port is not blocked by a firewall.
4. If you use ComfyUI Desktop, the default port is `8000` — either change it to `8188` in ComfyUI settings, or change the `base_url` on every Houdini node to `http://127.0.0.1:8000`.

### Port mismatch between ComfyUI Desktop and Houdini

**Symptom:** Connection works only after manually editing the URL on every node.

**Fix:** ComfyUI Desktop defaults to port `8000`, while all Houdini nodes default to `8188`. Pick one and align both sides. Changing ComfyUI's port to `8188` (in Settings → Server Config) is usually easier than editing every Houdini node.

## Missing dependencies

### `no trimesh module! acting as a noop`

**Symptom:** This message appears in the ComfyUI server console when using 3D / mesh features.

**Cause:** The optional `trimesh` Python package is not installed in the ComfyUI environment.

**Fix:** See [Dependencies — trimesh](Dependencies.md#installing-trimesh-into-comfyui). The `Trimesh UV Unwrap` node will pass meshes through unchanged until `trimesh` is installed.

### `ImportError: No module named 'requests'` (or `PIL`)

**Symptom:** Errors appear in Houdini when cooking a ComfyUI node or drag-and-dropping an image.

**Cause:** The Houdini-side Python dependencies are missing.

**Fix:** Install `requests` and `Pillow` into Houdini's Python — see [Dependencies — Houdini-side](Dependencies.md#houdini-side-python-dependencies).

### Missing ComfyUI node type after drag-and-drop

**Symptom:** Dragging a ComfyUI-generated PNG into Houdini shows a `MissingNodeDefinitionError` mentioning an unknown node type.

**Cause:** The image's workflow references a ComfyUI custom node that is not installed.

**Fix:** The error message tries to identify the node pack name. Install the missing pack via ComfyUI Manager → Custom Nodes Manager, restart ComfyUI, then press **Update Node Definitions** in Houdini and retry.

## Workflow & cooking

### Cook hangs indefinitely

**Symptom:** Pressing **Compute** never returns; Houdini appears frozen.

**Cause:** The ComfyUI server is busy, stuck, or the result poll loop is waiting on a prompt that will never finish.

**Workaround:** Press the **Interrupt** button on the compound graph node (sends `POST /sidefx_houdini/interrupt` to ComfyUI). If Houdini itself is frozen, you may need to escape the cook from Houdini's side.

> **Known limitation:** The current HTTP calls have no timeout. If ComfyUI is unresponsive, Houdini will wait indefinitely. Restarting ComfyUI typically unblocks Houdini.

### Result image is stale / not updating

**Symptom:** After re-cooking, the result looks identical to the previous run.

**Cause:** ComfyUI caches results by filename. If the output filename did not change, the cached file is returned.

**Fix:** Enable the **delete prompt history** / **delete input image** cleanup options on the node, or manually clear ComfyUI's `output` and `temp` directories.

### Windows: "file is in use" when cleaning up

**Symptom:** On Windows, cleanup of temporary input/output images fails because the file is locked.

**Cause:** Another process (often ComfyUI's image cache or a viewer) holds the file open.

**Workaround:** The plugin already shifts locked files to a `__<shift>__A_` backup name instead of failing. If these accumulate, periodically clear ComfyUI's `input` directory manually.

## Node definitions

### Tab menu inside compound graph is missing ComfyUI nodes

**Symptom:** After diving into a compound graph node, the tab menu does not show ComfyUI nodes.

**Cause:** Node definitions have not been synced, or ComfyUI was not running when you last pressed **Update Node Definitions**.

**Fix:**
1. Ensure ComfyUI is running.
2. Go up to the compound graph node, expand the **Node Definitions** tab, press **Update Node Definitions**.
3. Dive back in — the tab menu should now list all ComfyUI nodes.

### Newly installed ComfyUI model does not appear in Houdini

**Symptom:** After downloading a model via ComfyUI Manager, it does not show up in Houdini node parameters.

**Fix:** Press **Update Node Definitions** on the compound graph node. Houdini caches ComfyUI's node/model definitions and only refreshes on demand.
