# ComfyUI Integration Pitfalls

Common issues and their solutions when working with ComfyUI from Hermes.

## 1. API Format vs UI Format

ComfyUI has two JSON formats that look similar but are NOT interchangeable:

**API Format** (works with POST /prompt):
```json
{"3": {"class_type": "KSampler", "inputs": {"seed": 42, "model": ["4", 0]}}}
```

**UI Format** (saved from ComfyUI frontend):
```json
{"nodes": [{"id": 3, "type": "KSampler", "pos": [300, 200], "widgets_values": [42]}], "links": [[1, 4, 0, 3, 0, "MODEL"]]}
```

If a user provides a UI format workflow, you CANNOT pass it to `queue_prompt()` directly.
Options:
- Ask the user to export as "API Format" from ComfyUI's menu
- Parse it manually: node IDs become string keys, `widgets_values` map to `inputs`
  by the order defined in `object_info`, and `links` become `["source_id", output_idx]`

## 2. Node IDs Must Be Strings

Wrong: `{3: {"class_type": "KSampler", ...}}`
Right: `{"3": {"class_type": "KSampler", ...}}`

Links must also reference string IDs: `["4", 0]` not `[4, 0]`.
Python's `json.dumps()` auto-converts int keys to strings, but if you build
dicts programmatically, use string keys from the start to avoid confusion.

## 3. Model Filename Must Be Exact

`list_models("checkpoints")` returns exact filenames like:
```
["sd_xl_base_1.0.safetensors", "v1-5-pruned-emaonly.safetensors"]
```

These include the extension and are case-sensitive. Common mistakes:
- Using display names: "SDXL Base" ✗ → "sd_xl_base_1.0.safetensors" ✓
- Missing extension: "v1-5-pruned-emaonly" ✗
- Wrong case: "V1-5-Pruned-Emaonly.safetensors" ✗

Always call `list_models()` first and use an exact match.

## 4. Custom Node Not Found

Error: `"class_type 'CannyEdgePreprocessor' not found"`

The workflow uses a node from a custom node pack that isn't installed.
Common required packs:
- `comfyui_controlnet_aux` — ControlNet preprocessors (Canny, Depth, Pose)
- `ComfyUI-AnimateDiff-Evolved` — video generation
- `ComfyUI-Impact-Pack` — face detection, segmentation
- `ComfyUI_IPAdapter_plus` — IP-Adapter image conditioning

Solution: Install via ComfyUI Manager or clone into `custom_nodes/`:
```bash
cd ComfyUI/custom_nodes
git clone https://github.com/<org>/<repo>.git
# Restart ComfyUI
```

Check available nodes: `comfy_api("GET", "/object_info")` — if the class isn't
listed, the custom node pack isn't installed.

## 5. VRAM Out of Memory

Symptoms: CUDA OOM error, generation hangs then crashes.

Mitigations:
- Free VRAM between generations: `comfy_api("POST", "/free", {"unload_models": True})`
- Start ComfyUI with `--lowvram` or `--novram` for large models
- Reduce image dimensions (1024→768) or batch_size (1)
- Use fp16 or quantized models instead of fp32

Check VRAM usage:
```python
stats = comfy_api("GET", "/system_stats")
for dev in stats.get("devices", []):
    used = dev["vram_total"] - dev["vram_free"]
    print(f"{dev['name']}: {used/1e9:.1f}GB / {dev['vram_total']/1e9:.1f}GB")
```

## 6. Generation Never Completes

`wait_for_completion()` times out but no error.

Possible causes:
- **Queue is full**: Another generation is ahead in line. Check `get_queue_status()`.
- **Node is hung**: Some custom nodes can hang on edge cases. Use `interrupt()`.
- **Server disconnected**: Verify ComfyUI is still running with `comfy_api("GET", "/system_stats")`.

Always use a timeout in `wait_for_completion()` and handle `TimeoutError` gracefully.

## 7. Output Images Not Found

`get_image()` returns 404 or empty.

Common causes:
- **Wrong type parameter**: Output images use `type="output"`, uploaded images use
  `type="input"`, temporary previews use `type="temp"`.
- **Subfolder mismatch**: Check the `subfolder` field in the output metadata.
- **PreviewImage vs SaveImage**: `PreviewImage` nodes save to `temp/` with auto-cleanup.
  Use `SaveImage` for persistent outputs.

## 8. Upload Image Not Found by Workflow

After `upload_image()`, the workflow can't find the file.

The filename returned by upload may differ from what you sent (deduplication,
sanitization). Always use the returned filename:
```python
result = upload_image("/tmp/input.png")
actual_name = result["name"]  # Use THIS in the workflow
workflow["2"]["inputs"]["image"] = actual_name
```

## 9. Connection Refused

`urllib.error.URLError: <urlopen error [Errno 111] Connection refused>`

ComfyUI isn't running or is on a different port/host.
- Default: `http://127.0.0.1:8188`
- Check: `curl http://127.0.0.1:8188/system_stats`
- Remote: Set `COMFYUI_URL` env var or pass URL explicitly
- Firewall: ComfyUI defaults to `--listen 127.0.0.1` (localhost only).
  For remote access, start with `--listen 0.0.0.0`

## 10. Sampler/Scheduler Compatibility

Not all sampler + scheduler combinations work with all models.

Safe defaults by model type:
- **SD 1.5/SDXL**: `euler_ancestral` + `normal`, CFG 7.0
- **Flux**: `euler` + `simple`, CFG 1.0
- **SD3**: `euler` + `sgm_uniform`, CFG 4.5

If generation produces noise or solid colors, try a different sampler/scheduler.
