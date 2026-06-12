# PRD: pi-vision-proxy 1.4.0

**Status:** Final
**Target version:** pi-vision-proxy 1.4.0
**Date:** 2026-05-03

---

## Changes since v1.3.0

1. **`analyze_image` tool** — agent-facing tool for targeted re-querying of images with multi-form crop support and optional grounding.
2. **Multi-image batched comparison** — adaptive joint vision calls when ≥2 images arrive together.
3. **`/vision-proxy describe` slash command** — user-facing re-query and re-describe with extended crop syntax.
4. **Optional grounded-coordinate output** — per-model native-format grounding with `grounding_format` metadata.
5. **Spatial-awareness fix** — image dimensions, filenames, and `crop_origin` in all fence types so agents can reason about coordinates without prior knowledge of image geometry.

---

## Background

pi-vision-proxy currently provides automatic, transparent image description for non-vision models in Pi. It runs in `before_agent_start`, sends each attached image to a configured vision model once, persists the description in the session keyed by image hash, and re-injects the description on every subsequent LLM call so descriptions survive across turns.

Three structural limitations remain:

1. **Generic descriptions, no question context.** Detail-level questions ("what error code?", "what is the y-axis maximum?") often require information the generic pass omitted.
2. **Per-image isolation.** Comparison questions cannot be answered from independently generated per-image descriptions.
3. **No user-side override.** No user-facing way to refresh or re-query a wrong description.

---

## Summary of features

| # | Feature | Type | Default |
|---|---|---|---|
| 1 | `analyze_image` tool with multi-form crop & question support | Agent-facing tool | enabled when proxy enabled |
| 2 | Multi-image batched comparison with adaptive prompting | Behaviour change | enabled when ≥ 2 images in one turn |
| 3 | `/vision-proxy describe` slash command | User-facing command | always available |
| 4 | Optional grounded-coordinate output with format metadata | Capability flag per vision model | curated Tier 1 list pre-populated |

---

## Image metadata and dimensions

All three fence tags carry image dimensions and filename. Dimensions are stored in an in-memory map populated on first ingestion — no session-entry persistence needed (images are always re-decoded each session).

```typescript
interface ImageMeta {
  width: number;
  height: number;
  filename?: string; // basename only
}

const _imageMeta = new Map<string, ImageMeta>();
```

When `readImageFileWithReason` returns an image, it also returns the basename. The caller stores it in `_imageMeta` alongside dimensions extracted via `image-size` (header-only, no full decode).

**Open question:** Whether `sharp` (for cropping and pHash) should be a hard or optional dependency, and what the degradation behavior is when it's absent. `image-size` is always a hard dependency.

---

## Fence tag reference

Three distinct tags — each has a clear semantic role so the agent can distinguish generic descriptions from targeted analyses from joint comparisons.

| Tag | Producer | Semantics |
|---|---|---|
| `<vision_proxy_description>` | Auto-proxy (`before_agent_start`) | Generic, per-image, always produced for attached images |
| `<vision_proxy_analysis>` | `analyze_image` tool | Targeted, question-driven, possibly cropped/grounded |
| `<vision_proxy_joint_description>` | Auto-proxy or tool (≥2 images) | Multi-image comparison |

Closing-tag neutralisation is applied to all three fence types. The `fenceUntrusted` function is updated to handle all three tags.

**Precedence rule:** `analyze_image` results (`<vision_proxy_analysis>`) are authoritative for the specific question asked. The cached generic description (`<vision_proxy_description>`) remains the default for everything else. The agent receives both in context and resolves contradictions using question specificity and recency. No correction entry is created.

---

## Feature 1 — Targeted analysis via `analyze_image` tool

### Problem

The automatic proxy generates one generic description per image. When the agent later needs a specific detail, it has no recourse — and even with a vision-capable active model, images attached many turns ago tend to fall out of effective attention.

### Functional requirements

**FR-1.1** When the proxy is enabled (`mode != off`), pi-vision-proxy registers a tool named `analyze_image` and exposes it to the active model alongside Pi's other tools, including when the active model supports images natively.

**FR-1.2** Tool schema:

```typescript
analyze_image({
  images:   string[],              // 1..maxImagesPerCall; path or "sha256:<hex>"
  question: string,                // required, non-empty, max 4000 chars
  model?:   string,                // optional; provider/model-id
  crop?:    CropEntry[],           // optional; per-image crop
  reason?:  string                 // optional; logged for analytics only
}) -> string

type CropEntry = {
  image_index: number;
} & (
  | { region: NamedRegion }
  | { normalized: { x: number; y: number; width: number; height: number } }
  | { pixels:     { x: number; y: number; width: number; height: number } }
);

type NamedRegion =
  | "top-left" | "top-right" | "bottom-left" | "bottom-right"
  | "top" | "bottom" | "left" | "right" | "center"
  | "top-half" | "bottom-half" | "left-half" | "right-half";
```

Each `CropEntry` must include exactly one of `region`, `normalized`, or `pixels`. Specifying more than one is a tool error.

**FR-1.2.1** *Crop semantics.*

- **`region`** — proxy resolves to a normalized rectangle internally. Quadrants (`top-left` etc.) cover 50% × 50%. `top` / `bottom` / `left` / `right` cover the indicated 50% × 100% strip. `center` is the centre 50% × 50%. `*-half` aliases are explicit names for the 50%-strip forms. Always valid; never errors.
- **`normalized`** — `x`, `y`, `width`, `height` ∈ [0.0, 1.0]. Resolved to pixels by multiplying by image dimensions. Out-of-bounds values are clamped; if the resulting rectangle has zero area, tool error.
- **`pixels`** — absolute pixel coordinates. Out-of-bounds values are clamped to image dimensions. Zero-area after clamping → tool error. Use this form only when the agent has authoritative pixel coordinates from a prior fence or from a previous grounded response.

The proxy converts all three forms to pixels internally before cropping locally and transmitting the cropped region to the vision model.

**FR-1.2.2** *`reason` field.* Optional, logged when supplied. No semantic role — purely for analytics.

**FR-1.2.3** *Tool description text.* The tool description registered with the active model must include all three crop forms with concrete examples. Required text:

> Use `analyze_image` when (a) the cached description of an image lacks a detail you need, (b) you need to compare or cross-reference multiple images, or (c) you need to focus on a specific region.
>
> **Cropping.** Three forms, in order of preference:
>
> - **`region`** — coarse cut by name. Use when you don't have exact dimensions: `{ image_index: 0, region: "bottom-right" }`.
> - **`normalized`** — fractional coordinates 0.0–1.0. Default choice for precise crops without knowing image dimensions: `{ image_index: 0, normalized: { x: 0.5, y: 0.5, width: 0.4, height: 0.4 } }`.
> - **`pixels`** — absolute pixels. Use only when you have authoritative coordinates from a prior `<vision_proxy_description>` or `<vision_proxy_analysis>` (which carry `width` and `height` attributes) or from a previous grounded response. Example: `{ image_index: 0, pixels: { x: 1840, y: 120, width: 840, height: 360 } }`.
>
> Image dimensions and filenames are available in the `width`, `height`, and `filename` attributes of `<vision_proxy_description>`, `<vision_proxy_analysis>`, and `<vision_proxy_joint_description>` blocks in your context.
>
> When a crop is applied, the response fence carries a `crop_origin` attribute (e.g. `crop_origin="1840,120"`). Add the origin's x to any returned x-coordinate and the origin's y to any returned y-coordinate to map coordinates back to the original full image.
>
> The tool result is authoritative for the specific question asked; the cached generic description remains the default for everything else.

**FR-1.3** Path resolution and security. Delegates path policy to Pi's `read` tool. Adds: `..` segments rejected, symlink-escape rejected, `images.length ≤ maxImagesPerCall`.

**FR-1.4** Model override. Must be registered, must support image input. Agent-initiated overrides honoured even when `PI_VISION_PROXY_MODEL` is env-locked.

**FR-1.5** *(removed — consent system removed in 1.6.0)*

**FR-1.6** Persistence. Image bytes by sha256. Image dimensions are computed on first ingestion and stored in the in-memory `_imageMeta` map (see "Image metadata and dimensions" above). Tool results are not persisted as canonical descriptions. Result LRU keyed by `(sorted_image_hashes, crop_signature, question_hash, model_id)` — `crop_signature` is a stable hash over the resolved-pixels rectangle, so semantically equivalent crops in different forms hit the same cache entry.

**FR-1.7** Configuration:
- `/vision-proxy tool on|off` — default `on` at GA, `off` during beta.
- `/vision-proxy max-images-per-call <n>` — default 10, range 1–20.
- `/vision-proxy cache-size <n>` — default 50, range 0–500.

**FR-1.8** *Result fencing with image metadata.* The tool result is wrapped in a `<vision_proxy_analysis>` fence that carries the dimensions, filename, and (when applicable) grounding format of the source image:

```
<vision_proxy_analysis
    image="sha256:abc123..."
    width="3840"
    height="2160"
    filename="screenshot.png"
    grounding_format="qwen_pixels">
The error dialog [1840, 120, 2680, 480] shows "Connection timed out"...
</vision_proxy_analysis>
```

Required attributes: `image` (always), `width`, `height` (always, in pixels). Optional attributes: `filename` (basename only — full paths are never exposed in fence attributes, per security note), `grounding_format` (only present when grounding is enabled and the model is in `groundingModels`; see FR-4.2 and FR-4.7).

For cropped calls, `width` and `height` are the dimensions of the **cropped region** sent to the vision model, not the original image, and `image` is suffixed with the crop signature: `image="sha256:abc...#crop:1840,120,840,360"`. A `crop_origin` attribute is added:

```
<vision_proxy_analysis
    image="sha256:abc123...#crop:1840,120,840,360"
    width="840"
    height="360"
    crop_origin="1840,120"
    filename="screenshot.png"
    grounding_format="qwen_pixels">
The error dialog [1840, 120, 2680, 480] shows "Connection timed out"...
</vision_proxy_analysis>
```

**`crop_origin`** — `"<x>,<y>"` comma-separated pixel offset of the crop's top-left corner within the original image. Present **only** on cropped calls. Absent on uncropped calls. The agent adds `crop_origin.x` to any returned x-coordinate and `crop_origin.y` to any returned y-coordinate to map coordinates back to full-image space.

Closing-tag neutralisation is applied to the body.

**FR-1.9** Telemetry. `vision_proxy.tool_call` entries with timestamp, image hashes, crop form and resolved pixels, question, supplied `reason`, model id, latency_ms, provider-reported token usage, `cache_hit` flag.

**FR-1.10** *Precedence between tool results and cached descriptions.* `analyze_image` results (`<vision_proxy_analysis>`) are authoritative for the specific question asked. The cached generic description (`<vision_proxy_description>`) remains the default for everything else. The agent receives both in context and resolves contradictions using question specificity and recency. No correction entry is created.

---

## Feature 2 — Multi-image batched comparison

### Problem

VLMs natively struggle to link visual cues across images even when both images are in the same prompt. But forcing a contrastive structure when the user just said "describe both" over-constrains.

### Approach

The vision model itself routes between contrastive and co-presented output structure — it's already multilingual and is already reading the user's question. The proxy supplies language-independent structural hints (filenames, perceptual similarity) as additional context.

### Functional requirements

**FR-2.1** When the automatic proxy processes a turn containing `N` ≥ 2 images:
- N independent vision calls produce per-image canonical descriptions (unchanged).
- One additional vision call produces a "joint description" cached by `(sorted_hashes, turn_index)`. Injected for *this turn only*.

**FR-2.2** When `analyze_image` is invoked with `2 ≤ N ≤ maxImagesPerCall` images, one vision call is made with all N images.

**FR-2.3** Joint calls obey the prompt-injection fence and `include_context` setting.

**FR-2.4** `maxBatch` config (slash: `/vision-proxy max-batch <n>`; range 1–10; default 4).

**FR-2.5** *Adaptive joint-call system prompt.*

```
You are analysing N images that the user has provided together.
Refer to them as Image 1 (filename), Image 2 (filename), ... .
Each image's dimensions will be visible to your reasoning as
"Image N: WxH pixels".

Read the user's question carefully. If the user is asking about
comparison, difference, change, or relationship between the images,
structure your response as:
  (1) similarities across the images,
  (2) specific differences,
  (3) a direct, step-by-step answer to the user's question.

Otherwise, describe each image in turn and note any obvious relationships
between them.

[Structural hints, if available: ...]
[User's prompt: ...]
```

This works in any language the underlying VLM supports — comparison cues in German, Italian, Japanese, Chinese, etc. are handled natively. No per-language code in the proxy.

**FR-2.5.1** *Filename hints.* See Appendix D. Basename-only.

**FR-2.5.2** *Perceptual similarity hint.* pHash computed after resizing both images to a fixed square with letterboxing (e.g. 64×64 with gray padding) to normalise aspect ratio before hashing. Threshold default 0.80 (lowered to account for residual aspect-ratio distortion in content-heavy crops). Configurable via `PI_VISION_PROXY_PHASH_THRESHOLD`.

*Known limitation:* Heavy crops (e.g. a small modal window cropped from a full-screen screenshot) may still fall below threshold. This is acceptable — the hint is advisory (FR-2.5.3) and the VLM can still compare the images without it.

**FR-2.5.3** Hints are advisory; the user's question wins.

**FR-2.5.4** Hints suppressed for `analyze_image` tool path — the agent's `question` is already explicit.

**FR-2.6** *Joint description fencing with per-image metadata.*

```
<vision_proxy_joint_description
    images="2"
    dimensions='[{"image":"sha256:aaa","width":1920,"height":1080,"filename":"before.png"},
                 {"image":"sha256:bbb","width":1920,"height":1080,"filename":"after.png"}]'
    grounding_format="none">
Image 1 and Image 2 differ primarily in the navigation sidebar...
</vision_proxy_joint_description>
```

`dimensions` is a JSON-encoded array, one entry per image, in the order presented to the vision model. Closing-tag neutralisation is applied.

**FR-2.7** Cost telemetry. Provider-reported tokens only.

### Edge cases

| Case | Behaviour |
|---|---|
| 1 image in a turn | No joint call |
| `maxBatch` exceeded | Joint call skipped; per-image descriptions still produced |
| One image fails to decode | Joint call skipped |
| `maxBatch=1` | Joint calls disabled |
| Filenames contain no recognised pattern | No filename hint; pHash hint may still apply |
| pHash computation fails | Skip pHash hint; proceed |
| Cropped images in joint call | Each cropped image's dimensions in the `dimensions` array reflect the cropped region |

---

## Feature 3 — `/vision-proxy describe` slash command

### Functional requirements

**FR-3.1** New slash subcommands. Crop syntax supports all three coordinate forms:

```
/vision-proxy describe <path-or-hash> [<path-or-hash> ...]
                                      [--question "<text>"]
                                      [--crop <image_index>:<form>]
                                      [--model <provider/model-id>]
                                      [--save]

/vision-proxy redescribe <path-or-hash> [--model <provider/model-id>]
```

`<form>` syntax for `--crop`:

| Form | Syntax | Example |
|---|---|---|
| Named region | `r=<name>` | `--crop 0:r=top-right` |
| Normalized | `n=<x>,<y>,<w>,<h>` | `--crop 0:n=0.5,0.5,0.4,0.4` |
| Pixels | `p=<x>,<y>,<w>,<h>` | `--crop 0:p=1840,120,840,360` |

Specifying more than one form per crop entry is rejected with a clear error message.

**FR-3.2** `describe` semantics:
- Resolves inputs identical to FR-1.3.
- Omitting `--question` triggers default generic system prompt.
- Multiple inputs trigger a joint call (FR-2.5 adaptive prompt + filename/pHash hints).
- Result printed to TUI only by default; `--save` overwrites canonical session description (single image, no `--question`, no `--crop`).

**FR-3.3** `redescribe` is sugar for `describe <hash> --save` with no question and no crop.

**FR-3.4** *(removed — consent system removed in 1.6.0)*

**FR-3.5** TUI output uses distinct visual style with `[Vision Proxy]` prefix.

**FR-3.6** Logged as `vision_proxy.command` entries.

---

## Feature 4 — Optional grounded-coordinate output

### Functional requirements

**FR-4.1** Per-model capability flag in the model registry: `supportsGrounding: boolean`.

**FR-4.1.1** *Curated default list* (1.4.0 ships with these marked `supportsGrounding: true`):

**Tier 1 — designed-in grounding:**
- `Qwen/Qwen2.5-VL-3B-Instruct`, `-7B-Instruct`, `-32B-Instruct`, `-72B-Instruct`
- `Qwen/Qwen3-VL` family
- `allenai/Molmo2-8B`, `allenai/Molmo2-72B`
- `deepseek-ai/deepseek-vl2-tiny`, `-small`, `-base`
- `OpenGVLab/InternVL3` family

**Tier 2 — opt-in with quality caveat:**
- `google/gemini-2.5-pro`, `google/gemini-3-pro`

**Excluded (warned on `add` attempt):**
- `anthropic/claude-*`, `openai/gpt-4o`, `gpt-5`, `meta/llama-*-vision`

**FR-4.1.2** Slash commands:

```
/vision-proxy grounding-models add <provider/model-id> [--format <fmt>]
/vision-proxy grounding-models remove <provider/model-id>
/vision-proxy grounding-models list
/vision-proxy grounding-models reset
```

`add` for an excluded model triggers a confirmation prompt explaining the unreliability.

**FR-4.2** *Native-format grounding output.* When `supportsGrounding: true`, the system prompt for both per-image and joint calls is appended with a model-specific instruction matching the model's training format. The proxy does **not** force a uniform output format across models. Instead, the proxy:

1. Looks up the model's `grounding_format` in the registry (FR-4.7).
2. Appends an instruction phrased in the model's native convention.
3. Records the format in the response fence's `grounding_format` attribute so the agent knows what convention to interpret.

Example instruction for `qwen_pixels` models:

> When you describe a spatial element, follow the description with bounding-box coordinates as `[x1, y1, x2, y2]` in absolute pixels relative to the image. Use `Image-N:` prefix for multi-image inputs.

Example instruction for `molmo_points` models:

> When you describe a spatial element, follow the description with point coordinates as `<point x="..." y="..." alt="..."/>` using your standard percentage-based convention.

**FR-4.3** When `supportsGrounding: false`, system prompt unchanged; `grounding_format` attribute on the fence is `"none"`.

**FR-4.4** The proxy does not parse, validate, or rewrite returned coordinates. Downstream agents and tooling can extract them using the `grounding_format` attribute as a key.

**FR-4.5** Purely additive — a model that ignores the grounding instruction simply produces a description without coordinates; `grounding_format` will still be set per the registry but the response body will contain no coordinates.

**FR-4.6** *Crop-and-grounding interaction.* When grounding is enabled and the call includes a `crop`, returned coordinates are relative to the **cropped region** sent to the vision model. The fence includes a `crop_origin` attribute (e.g. `crop_origin="1840,120"`) giving the pixel offset of the crop's top-left corner in the original image. Agents can recover full-image coordinates by adding `crop_origin` to any returned coordinate. The fence's `width` and `height` reflect the cropped dimensions, and the `image` attribute is suffixed with the crop signature (FR-1.8). Documented in the tool description (FR-1.2.3).

**FR-4.7** *Per-model `grounding_format` registry.* New required field on every entry in `groundingModels`. The registry maps each grounding-capable model to one of the following format identifiers:

| Format identifier | Convention | Example output |
|---|---|---|
| `qwen_pixels` | absolute pixels, two-corner `[x1, y1, x2, y2]` | `[1840, 120, 2680, 480]` |
| `molmo_points` | percentage-based points (0–100), Molmo's native `<point>` element | `<point x="42.5" y="62.3" alt="error dialog"/>` |
| `deepseek_bbox` | DeepSeek's `<\|ref\|>desc<\|/ref\|><\|det\|>[[x1,y1,x2,y2]]<\|/det\|>` format | (as documented by DeepSeek) |
| `internvl_pixels` | InternVL's native bbox format, absolute pixels | `[100, 200, 300, 400]` |
| `gemini_normalized_1000` | normalized 0–1000 coordinates per Gemini API convention | `[420, 124, 670, 248]` |
| `none` | no grounding | — |

**Default mappings shipped in 1.4.0:**

```json
{
  "Qwen/Qwen2.5-VL-7B-Instruct":  "qwen_pixels",
  "Qwen/Qwen2.5-VL-72B-Instruct": "qwen_pixels",
  "Qwen/Qwen3-VL-7B":             "qwen_pixels",
  "allenai/Molmo2-8B":            "molmo_points",
  "allenai/Molmo2-72B":           "molmo_points",
  "deepseek-ai/deepseek-vl2":     "deepseek_bbox",
  "deepseek-ai/deepseek-vl2-small": "deepseek_bbox",
  "OpenGVLab/InternVL3-8B":       "internvl_pixels",
  "google/gemini-2.5-pro":        "gemini_normalized_1000",
  "google/gemini-3-pro":          "gemini_normalized_1000"
}
```

When users add a model not in the default mapping, they must specify its format:

```
/vision-proxy grounding-models add <model-id> --format <format-identifier>
```

If `--format` is omitted, defaults to `qwen_pixels` (the most common convention) with a TUI warning.

---

## Cross-feature interaction

| Combination | Behaviour |
|---|---|
| `analyze_image` with N ≥ 2 images | Single joint vision call using FR-2.5 adaptive prompt; filename/pHash hints suppressed (FR-2.5.4) |
| `analyze_image` with `crop` and N ≥ 2 | Crops applied per-image before transmission, then batched. Joint description's `dimensions` array reflects cropped sizes |
| `analyze_image` with `pixels` crop on an image not previously described | Proxy clamps using actual dimensions (computed at ingestion); proceeds without warning |
| Automatic proxy + `analyze_image` follow-up in same turn | Both happen; tool result is authoritative for its specific question (FR-1.10) |
| `/vision-proxy describe` during automatic proxy turn | Serialised |
| Mode `off` | Tool unregistered; joint calls disabled; slash commands refuse |
| Grounding enabled + cropped call | Coordinates relative to cropped image; `crop_origin` provided for mapping back (FR-4.6) |
| User in non-English locale | FR-2.5 routing happens inside the VLM in user's language; hints language-independent |

---

## Configuration summary

| Setting | Slash | Env | Default | Range |
|---|---|---|---|---|
| Tool exposure | `/vision-proxy tool on\|off` | `PI_VISION_PROXY_TOOL` | `on` (GA) / `off` (beta) | — |
| Max images per call | `/vision-proxy max-images-per-call <n>` | `PI_VISION_PROXY_MAX_IMAGES_PER_CALL` | `10` | 1–20 |
| Max batch (auto) | `/vision-proxy max-batch <n>` | `PI_VISION_PROXY_MAX_BATCH` | `4` | 1–10 |
| Result cache size | `/vision-proxy cache-size <n>` | `PI_VISION_PROXY_CACHE_SIZE` | `50` | 0–500 |
| pHash similarity threshold | (`vision-proxy.json`) | `PI_VISION_PROXY_PHASH_THRESHOLD` | `0.80` | 0.0–1.0 |
| Grounding-capable models | `/vision-proxy grounding-models …` | — | Tier 1 list (FR-4.1.1) | — |

`vision-proxy.json` schema additions:

```json
{
  "tool": "on",
  "maxImagesPerCall": 10,
  "maxBatch": 4,
  "cacheSize": 50,
  "pHashSimilarityThreshold": 0.80,
  "groundingModels": {
    "Qwen/Qwen2.5-VL-7B-Instruct":  { "format": "qwen_pixels" },
    "allenai/Molmo2-8B":            { "format": "molmo_points" },
    "deepseek-ai/deepseek-vl2":     { "format": "deepseek_bbox" },
    "OpenGVLab/InternVL3-8B":       { "format": "internvl_pixels" }
  }
}
```

(Shipped default is the full FR-4.7 mapping; example abbreviated.)

Backwards compatibility: existing 1.3.0 `vision-proxy.json` files load unchanged. `groundingModels`, `tool`, `maxImagesPerCall`, `maxBatch`, `cacheSize` were absent in 1.3.0, so the new defaults take effect on first 1.4.0 launch without losing user state.

---

## Telemetry / observability

Session entry types:

- `vision_proxy.tool_call` — includes `crop_form` (`region`/`normalized`/`pixels`/none) and `crop_resolved_pixels` for analytics.
- `vision_proxy.joint_description` — includes `filename_hint`, `phash_similarity`, and `dimensions` array.
- `vision_proxy.command`
- `vision_proxy.skip`

Each entry includes timestamp, image hashes, model id, `grounding_format` used, latency_ms, provider-reported token usage, `cache_hit` where applicable.

---

## Security considerations

- **Prompt injection.** Closing-tag neutralisation applied to all three fence types (`<vision_proxy_description>`, `<vision_proxy_analysis>`, `<vision_proxy_joint_description>`). Filename hints are basename-only — full paths never appear in fence attributes or hint strings.
- **Path policy.** Defers to Pi's `read` tool plus `..`/symlink-escape rejection.
- **Crop as exfiltration vector.** Cropped region still sent to configured vision provider. The three crop forms do not change the security posture — they just determine how the agent specifies what to send.
- **Dimension metadata leak.** Image dimensions in fence attributes are derived from the user's own attached images and pose no additional risk beyond what the proxy already sends to the vision model.

---

## Rollout

- **1.4.0-beta.1**: Feature 1 with `tool=off` default, all three crop forms supported, dimensions in description fence.
- **1.4.0-beta.2**: Feature 3 with extended crop syntax.
- **1.4.0-beta.3**: Feature 2 with `maxBatch=1` default; adaptive prompt, filename/pHash hints, joint-fence dimensions.
- **1.4.0-beta.4**: Feature 4 with curated Tier 1 default list and `grounding_format` registry; opt-in flag itself off by default.
- **1.4.0**: Flip defaults to `tool=on`, `maxBatch=4`. Grounding flag remains opt-in but list pre-populated.

---

## Out of scope (later versions)

- Cross-session image library
- Per-image TTL on cached descriptions
- Vision model fallback chain
- Streaming descriptions
- Preprocessing (resize, format normalisation) before vision call
- Dollar-cost telemetry
- Proxy-side parsing/normalisation of grounding coordinates into a unified format (currently the agent reads `grounding_format` and decides)
- Auto-detection of grounding capability and format from provider metadata

---

## Appendix A — References

- Dong et al., *Training Multi-Image Vision Agents via End2End Reinforcement Learning* (IMAgent), arXiv 2512.08980, December 2025.
- Chen et al., *MiCo: Multi-image Contrast for Reinforcement Visual Reasoning*, NeurIPS 2025, arXiv 2506.22434.
- *LaViT: Aligning Latent Visual Thoughts for Multi-modal Reasoning*, arXiv 2601.10129, January 2026.
- Wang et al., *CiQi-Agent: Aligning Vision, Tools and Aesthetics in Multimodal Agent for Cultural Reasoning*, arXiv 2603.28474, March 2026.
- Bai et al., *Qwen2.5-VL Technical Report*, 2025. Source for FR-4.7 `qwen_pixels` format.
- Deitke et al., *Molmo2: Open Vision-Language Video Model*, arXiv 2601.10611, January 2026. Source for FR-4.7 `molmo_points` format.
- DeepSeek AI, *DeepSeek-VL2: Mixture-of-Experts Vision-Language Models*, arXiv 2412.10302, December 2024. Source for FR-4.7 `deepseek_bbox` format.

## Appendix B — Tool contract

```typescript
type AnalyzeImageInput = {
  images: string[];
  question: string;
  model?: string;
  crop?: CropEntry[];
  reason?: string;
};

type CropEntry = {
  image_index: number;
} & (
  | { region: NamedRegion }
  | { normalized: { x: number; y: number; width: number; height: number } }
  | { pixels:     { x: number; y: number; width: number; height: number } }
);

type NamedRegion =
  | "top-left" | "top-right" | "bottom-left" | "bottom-right"
  | "top" | "bottom" | "left" | "right" | "center"
  | "top-half" | "bottom-half" | "left-half" | "right-half";

type AnalyzeImageOutput = string;
```

## Appendix C — Slash commands

```
/vision-proxy tool on|off
/vision-proxy max-images-per-call <n>     # 1..20
/vision-proxy max-batch <n>               # 1..10
/vision-proxy cache-size <n>              # 0..500
/vision-proxy grounding-models add|remove|list|reset [<provider/model-id>] [--format <fmt>]
/vision-proxy describe <path|hash>...
                          [--question "<text>"]
                          [--crop <i>:r=<region> | n=<x>,<y>,<w>,<h> | p=<x>,<y>,<w>,<h>]
                          [--model <provider/model-id>]
                          [--save]
/vision-proxy redescribe <path|hash>
                          [--model <provider/model-id>]
```

## Appendix D — Filename hint patterns (FR-2.5.1)

Basename-only matching, case-insensitive.

```
before.*  ∧  after.*                       → "before/after pair"
old.*     ∧  new.*                         → "old/new pair"
{prefix}{version}  (≥2, same prefix)       → "versioned sequence"
*_1.*     ∧  *_2.*                         → "numbered sequence"
*-1.*     ∧  *-2.*                         → "numbered sequence"
YYYY-MM-DD_*.*  (sortable, ≥2 files)       → "time-ordered sequence"
(no match)                                 → no hint emitted
```

**Version extraction rule:** From each basename, extract a `(prefix, version_number)` tuple by matching the rightmost occurrence of `[vV]?(\d+(?:\.\d+)?)` immediately before the extension. If ≥2 files share the same prefix but have different version numbers → "versioned sequence".

Examples that match:
- `mockup_v2.png ∧ mockup_v4.png` → prefix=`mockup_v`, versions {2, 4}
- `draft_v1.1.png ∧ draft_v1.2.png` → prefix=`draft_v`, versions {1.1, 1.2}
- `app2.png ∧ app3.png` → prefix=`app`, versions {2, 3}

The "versioned sequence" rule is checked before the `_\d` and `-\d` numbered-sequence rules.

## Appendix E — Description fence reference

All three fence types share a common metadata schema. Required attributes are always present; optional attributes appear when applicable.

| Attribute | Required | Where applied | Notes |
|---|---|---|---|
| `image` | yes | `description`, `analysis` | sha256 hash; suffixed with `#crop:x,y,w,h` for cropped calls |
| `images` | yes | `joint_description` | image count |
| `dimensions` | yes | `joint_description` | JSON-encoded array, one entry per image |
| `width` | yes | `description`, `analysis` | pixels of what the vision model saw (cropped or original) |
| `height` | yes | `description`, `analysis` | as above |
| `crop_origin` | optional | `description`, `analysis` | `"x,y"` pixels; present only on cropped calls; top-left corner of crop within original image |
| `filename` | optional | all three | basename only; never full path |
| `grounding_format` | optional | all three | one of FR-4.7's identifiers; absent or `"none"` when grounding off |
