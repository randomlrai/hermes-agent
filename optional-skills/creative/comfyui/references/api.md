# ComfyUI REST API Reference

Complete reference for ComfyUI's HTTP API. Default: `http://127.0.0.1:8188`.

## Workflow Execution

### POST /prompt — Queue a workflow

```json
{
  "prompt": { <workflow nodes dict> },
  "client_id": "optional-uuid",
  "prompt_id": "optional-custom-id",
  "front": false,
  "extra_data": { "extra_pnginfo": {} }
}
```

Response: `{"prompt_id": "uuid", "number": 1, "node_errors": {}}`

### GET /history — Execution history

Query params: `?max_items=200&offset=0`

Returns: `{ "prompt_id": { "prompt": [...], "outputs": {...}, "status": {...} } }`

### GET /history/{prompt_id} — Single prompt history

Returns the same structure for one prompt. Empty dict `{}` if not yet complete.

### POST /interrupt — Stop current generation

Optional body: `{"prompt_id": "..."}` for targeted interrupt.

### GET /queue — Queue status

Returns: `{"queue_running": [...], "queue_pending": [...]}`

### POST /queue — Clear queue

Body: `{"clear": true}` to clear all, or `{"delete": ["prompt_id1", ...]}` for specific items.

## Images

### GET /view — Download image

Query params:
- `filename` (required) — output filename
- `type` — `output` (default), `input`, `temp`
- `subfolder` — subdirectory within the type folder
- `channel` — `rgb`, `rgba`, `a` (alpha only)
- `preview` — format and quality, e.g. `webp;90`

### POST /upload/image — Upload image

Multipart form data:
- `image` — file data
- `type` — `input` (default), `temp`, `output`
- `subfolder` — target subdirectory
- `overwrite` — `true` or `false`

Response: `{"name": "filename.png", "subfolder": "", "type": "input"}`

### POST /upload/mask — Upload mask

Same as upload/image but includes `original_ref` field linking to the source image.

## Node/Model Information

### GET /object_info — All node types

Returns every registered node class with:
- `input` — required and optional inputs with types, defaults, min/max
- `input_order` — parameter ordering
- `output` — output types
- `output_name` — human-readable output names
- `category` — node category (sampling, conditioning, latent, etc.)
- `description` — node description

### GET /object_info/{class_type} — Single node info

Returns info for one node class (e.g., `/object_info/KSampler`).

### GET /models/{folder} — List models

Folders: `checkpoints`, `loras`, `vae`, `controlnet`, `clip`, `clip_vision`,
`upscale_models`, `embeddings`, `hypernetworks`, `unet`, `diffusion_models`.

Returns: array of filename strings.

### GET /embeddings — List embeddings

Returns: array of embedding names.

## System

### GET /system_stats — System information

Returns: OS, Python version, PyTorch version, VRAM per device, RAM total/free.

### POST /free — Free memory

Body: `{"unload_models": true, "free_memory": true}`

### GET /features — Feature flags

Returns server capability flags.

## Workflow JSON Format

### API Format (used with POST /prompt)

```json
{
  "node_id_string": {
    "class_type": "NodeClassName",
    "inputs": {
      "param_name": value,
      "linked_input": ["source_node_id", output_index]
    }
  }
}
```

Key rules:
- Node IDs are **strings** (`"3"`, not `3`)
- Links use `["node_id", output_index]` arrays (output_index is 0-based int)
- Values are native types: string, number, boolean
- `class_type` must match a registered node exactly (case-sensitive)

### UI Format (saved from ComfyUI frontend)

Contains additional layout data (`pos`, `size`, `widgets_values`, `links` array).
**NOT directly usable** with POST /prompt. Extract the API format by:
1. Export as "API Format" from ComfyUI menu
2. Or parse the `workflow` key and convert links to the `["id", index]` format

## WebSocket (optional, for real-time progress)

Connect to: `ws://host:8188/ws?clientId={uuid}`

Events (JSON):
- `status` — queue count update
- `execution_start` — prompt began
- `execution_cached` — cached nodes skipped
- `executing` — current node (null = done)
- `executed` — node finished with output
- `execution_success` — entire prompt done
- `execution_error` — node error with traceback
- `progress` — step progress within a node

Completion detection: `"executing"` event with `data.node == null`.

Binary events (first 4 bytes = type):
- Type 1: preview image during generation
- Type 3: progress text
