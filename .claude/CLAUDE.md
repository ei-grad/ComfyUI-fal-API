# ComfyUI-fal-API

ComfyUI custom node plugin wrapping [fal.ai](https://fal.ai) API endpoints for image generation, video generation, LLMs, VLMs, upscalers, and LoRA training.

Fork: `ei-grad/ComfyUI-fal-API` (upstream: `gokayfem/ComfyUI-fal-API`)

## Project structure

```
__init__.py              # Entry point — dynamically imports node modules, merges NODE_CLASS_MAPPINGS
nodes/
  fal_utils.py           # Shared: FalConfig (singleton, API key), ImageUtils, ResultProcessor, ApiHandler
  image_node.py          # Image generation nodes (FAL/Image category)
  video_node.py          # Video generation nodes (FAL/VideoGeneration category)
  llm_node.py            # LLM node via OpenRouter (FAL/LLM)
  vlm_node.py            # VLM node via OpenRouter (FAL/VLM)
  trainer_node.py        # LoRA trainer nodes (FAL/Training)
  upscaler_node.py       # Upscaler nodes (FAL/Image)
config.ini               # FAL_KEY placeholder (env var takes priority)
example_workflows/       # ComfyUI workflow JSONs
pyproject.toml           # Dependencies + [tool.comfy] for ComfyUI Registry
```

## Adding a new node

1. Define a class in the appropriate `nodes/*_node.py` with:
   - `INPUT_TYPES(cls)` classmethod — returns `{"required": {...}, "optional": {...}}`
   - `RETURN_TYPES` tuple (e.g. `("IMAGE",)`)
   - `FUNCTION` string (method name to call)
   - `CATEGORY` string (e.g. `"FAL/Image"`)
   - The method itself
2. Add to `NODE_CLASS_MAPPINGS` at bottom of file: `"NodeName_fal": ClassName`
3. Add to `NODE_DISPLAY_NAME_MAPPINGS`: `"NodeName_fal": "Display Name (fal)"`
4. Use `ApiHandler.submit_and_get_result(endpoint, arguments)` for API calls
5. Use `ResultProcessor.process_image_result(result)` for image outputs
6. Use `ImageUtils.prepare_images(images)` to upload input images

## Gotchas

- **widgets_values order in workflow JSONs must match INPUT_TYPES declaration order.** When adding/reordering parameters in a node, all example workflows using that node must be updated.
- **API headers:** All fal API submissions include `X-Fal-Store-IO: 0` (via `ApiHandler._HEADERS`) to prevent payload persistence on fal's platform.
- **Config priority:** env var `FAL_KEY` > `config.ini`. Code is in `FalConfig._initialize()`.
- **Key validation is fail-fast at runtime, NOT at import.** `_initialize()` must never raise — `video_node.py` instantiates `FalConfig()` at module import, so raising there would break the whole plugin load and the nodes would vanish from the UI. Validation lives in `FalConfig._require_key()`, called from `get_client()`/`get_key()` (runtime, inside node `FUNCTION` methods). An empty/placeholder/malformed key (must look like `<id>:<secret>`) raises `FalKeyError` at execution time.
- **`FalKeyError` must propagate, not be swallowed.** Node `FUNCTION` methods wrap their body in `except Exception` and route through `ApiHandler.handle_*_error`, which would otherwise turn the error into a blank image / `"Error: ..."` string (reproducing the confusing downstream `InvalidSchema` failure). The three `handle_*_error` helpers re-raise `FalKeyError`; the combined-video node's own `except Exception` blocks (`generate_all_videos`, `generate_videos`) re-raise it too. `ImageUtils.upload_image`/`upload_file` call `get_client()` BEFORE their `try` so the error isn't swallowed into `return None`. Any new broad `except Exception` around a `get_client()`/`get_key()`/upload path must re-raise `FalKeyError`.
- **No test suite exists.** Test changes by running ComfyUI with the plugin loaded and checking `/object_info` endpoint.
- **`[tool.setuptools] packages = []`** is required in pyproject.toml because the flat layout (nodes/, example_workflows/) confuses setuptools auto-discovery. This project is not a real pip package — it's a ComfyUI plugin loaded via `__init__.py`.

## Testing locally

```bash
cd /path/to/comfy-org/ComfyUI
# Symlink plugin into custom_nodes/
ln -s /path/to/ComfyUI-fal-API custom_nodes/ComfyUI-fal-API
# Install deps into ComfyUI venv
uv pip install custom_nodes/ComfyUI-fal-API
# Run (--cpu if no GPU)
FAL_KEY=... .venv/bin/python main.py --cpu
# Verify nodes loaded
curl -s http://127.0.0.1:8188/object_info | python3 -c "import json,sys; d=json.load(sys.stdin); print(len([k for k in d if 'fal' in k.lower()]),'fal nodes')"
```

## fal.ai API conventions

- Image generation endpoints return `{"images": [{"url": "..."}]}` — use `ResultProcessor.process_image_result`
- Single-image endpoints return `{"image": {"url": "..."}}` — use `ResultProcessor.process_single_image_result`
- Video endpoints return URLs as strings
- Check endpoint schemas at `https://fal.ai/models/fal-ai/<endpoint>/api`
- The fal-client `submit()` method accepts `headers` dict for per-request headers
