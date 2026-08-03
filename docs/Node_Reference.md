# Node Reference

This page documents the HDA nodes shipped with the Houdini ComfyUI Bridge. Nodes are grouped by function.

> **Naming convention:** `<context>_<scope>.<name>.<major>.<minor>.hda`
> - `cop_xxx.` — COP (compositing) context
> - `lop_xxx.` — LOP (Solaris) context
> - `top_xxx.` — TOP / PDG context
> - Multiple versions of the same node may coexist for backward compatibility.

## Core submission nodes

### `comfyui_compound_graph_submit` (current: 1.3)

The primary node for building and submitting ComfyUI workflows. It is a **subnet**: dive inside to assemble a ComfyUI graph from `comfyui_partial_graph` nodes connected by COP wires.

- **Inputs:** 0–1 RGBA image
- **Outputs:** `dst` (RGBA image), `geo` (geometry)
- **Key parameters:** `base_url`, `do_realtime_cook`, `image_batch_index`, `comfyui_api_key`
- **Action:** Press **Compute** to build the combined graph, upload inputs, submit to ComfyUI, and download results.

This is the node used in the [First Steps](First_steps_1.md) tutorials.

### `comfyui_graph_submit` (current: 1.2)

An older, simpler submitter with a **fixed 8 RGBA inputs / 1 RGBA output** layout. The entire ComfyUI graph is stored as a single JSON string in the `cui_graph` parameter. Input replacement is declared via the `cui_inputs` multiparm.

Prefer `comfyui_compound_graph_submit` for new work. This node remains for legacy scenes.

## Graph building blocks

### `comfyui_partial_graph` (1.3 / 1.4)

Wraps a single ComfyUI node type (e.g. `KSampler`, `CheckpointLoaderSimple`). This is the basic building block inside a compound graph. Non-wire inputs (INT/FLOAT/STRING/BOOLEAN/COMBO) are promoted to subnet parameters; wire inputs (IMAGE/MASK/MODEL/…) become COP input connectors.

Version 1.4 extends the input/output count to 64 (1.3 supports 16). Both can coexist inside a compound graph.

### `comfyui_graph_preview` (1.3)

Previews an intermediate result inside a compound graph without leaving the subnet. Connect any image output to it, set the display flag, and press **Generate Preview**. See [First Steps 3](First_steps_3.md).

### `comfyui_partial_graph_outputs` (1.0)

A special node that performs **recursive submission**: it submits an inner sub-graph as an independent ComfyUI prompt, then injects the results back into the outer graph as string variables (referenced via `@{{var}}`). Use this for multi-stage pipelines (e.g. generate a mesh, then render it).

## 3D / geometry nodes

### `comfyui_geo_uploader` (1.3)

Exports Houdini geometry to GLB or OBJ and uploads it to ComfyUI's input directory. Used as a source node in 3D workflows.

### `lop_xxx.comfyui_hy3d_generate_asset` (current: 1.2)

LOP-context node for generating 3D assets with Hunyuan3D. Requires the [Hy3D Wrapper](Dependencies.md#comfyui-custom-node-dependencies) ComfyUI node pack. Output type `TRIMESH` is exported via `Hy3DExportMesh` and returned to Houdini as a geometry file.

## Sequence & batch nodes

### `comfyui_range_to_batch` (1.3)

Converts a Houdini frame range into a ComfyUI `Image Batch`. Generates one `LoadImage` per frame and chains `ImageBatch` nodes to merge them. Each frame is uploaded once and cached to avoid re-uploading.

### `comfyui_batch_upload_control` (current: 1.0)

Controls batch image upload, paired with `comfyui_range_to_batch` for sequence workflows.

### `comfyui_compute_in_order` (current: 1.0)

Orders sequential computation, useful for batch/sequence submission where results must be processed in order.

## Utility nodes

### `cop_xxx.autodepth` (current: 1.0)

Generates a depth map from an input image (e.g. via a depth-estimation model). The depth map can feed ControlNet or similar conditioning.

### `cop_xxx.pick_points` (current: 1.0)

Interactive point-picker example node. Demonstrates how to collect control points from the viewport for use in a workflow.

### `comfyui_image_upload` (current: 1.2)

Standalone single-image upload node. Exposes `comfyui_image_name()`, `comfyui_image_subdir()`, and `upload_input()` for programmatic use.

## TOPs / PDG

### `top_xxx.comfyui_submit_graph` (current: 1.0)

TOP/PDG submitter for building automated pipelines that combine CG rendering with diffusion models. Allows batch processing of ComfyUI jobs from a Houdini TOP network.

## Deprecated nodes

The following nodes are **no longer shipped in releases** (excluded by `otl-deploy-rules`) but may still appear in the source repository or old scenes. Do not use them in new work.

| Deprecated node | Replaced by |
|------------------|-------------|
| `cop_xxx.diffusion_rendering` | `comfyui_diffusion_rendering_SDXL` |
| `cop_xxx.generate_image` | `comfyui_generate_image_SDXL` |
| `cop_xxx.comfyui_diffusion_rendering_SDXL` | `comfyui_compound_graph_submit` (build the workflow manually) |
| `cop_xxx.comfyui_diffusion_rendering_FLUX` | `comfyui_compound_graph_submit` |
| `cop_xxx.comfyui_diffusion_rendering_SDXL_inpainting` | `comfyui_compound_graph_submit` |
| `cop_xxx.comfyui_generate_image_SDXL` | `comfyui_compound_graph_submit` |
| `cop_xxx.comfyui_inpainting_IPA` | `comfyui_compound_graph_submit` |
| `cop_xxx.comfyui_demo_prompt_gen` | (removed) |

These were pre-configured workflow HDAs. The compound graph approach is more flexible — build the equivalent workflow inside a `comfyui_compound_graph_submit` node instead.

## Other

### `Recipes.hda`

A collection of example workflow recipes. Not deployed in releases; available in the source repository for reference.
