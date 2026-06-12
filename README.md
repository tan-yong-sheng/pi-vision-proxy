# pi-vision-proxy

Automatic **image** description for any model in [Pi](https://pi.dev).

When images are sent, this extension routes them to a **vision-capable model**, collects descriptions, persists them in the session, and injects them into the agent's context — so even text-only models can "see" your images across turns.

## What's new in 1.6.0

- **`/vision-proxy`** — the primary configuration command.
- **Consent-free operation** — the extension operates transparently without requiring explicit per-session consent.

## What's new in 1.4.0

- **`analyze_image` tool** — the agent can re-query images with targeted questions, multi-form crop support (region, normalized, pixels), and optional model-native grounding coordinates. Crops are applied locally before upload — only the cropped region is sent to the vision model.
- **Multi-image batched comparison** — when ≥2 images arrive together, an adaptive joint vision call produces a comparison description alongside per-image descriptions.
- **`/vision-proxy describe` slash command** — user-facing re-query with extended crop syntax, model override, and `--save` to overwrite the canonical description.
- **Grounding format registry** — per-model native-format coordinate output (Qwen pixels, Molmo points, DeepSeek bbox, InternVL pixels, Gemini 0–1000) with curated Tier 1 defaults.
- **ImageScript + imghash** — zero-native-dep image cropping and perceptual hashing.

## Install

```bash
pi install git:github.com/tan-yong-sheng/pi-vision-proxy@main
```

## Modes

| Mode | Behavior |
|------|----------|
| **`fallback`** | Only activates when the active model lacks image support (default) |
| **`always`** | Always uses the proxy, even if the active model supports images |
| **`off`** | Disabled entirely |

## Configuration

Settings persist across sessions in `~/.pi/agent/vision-proxy.json`. Environment variables override file settings; in-session commands override both.

### Slash commands

```
/vision-proxy                                            → opens interactive config menu
/vision-proxy pick                                       → pick vision model (provider → model)
/vision-proxy model <provider/model-id>                  → change image vision model
/vision-proxy fallback | always | off                    → set mode
/vision-proxy context on | off                           → include / exclude recent chat in proxy prompt
/vision-proxy tool on | off                              → enable/disable analyze_image tool
/vision-proxy max-images-per-call <1-20>                 → max images per tool call
/vision-proxy max-batch <1-10>                           → max images in auto-proxy joint call
/vision-proxy cache-size <0-500>                         → tool result cache entries
/vision-proxy grounding-models list                      → show grounding-capable models
/vision-proxy grounding-models add <provider/id> [--format <fmt>]
/vision-proxy grounding-models remove <provider/id>
/vision-proxy grounding-models reset                     → restore Tier 1 defaults
/vision-proxy describe <path|hash>... [--question "<text>"] [--crop <i>:<form>] [--model <provider/id>] [--save]
/vision-proxy redescribe <path|hash> [--model <provider/id>]
```

### Environment variables (override persisted settings)

| Variable | Values | Default |
|----------|--------|---------|
| `PI_VISION_PROXY_MODE` | `fallback`, `always`, `off` | `fallback` |
| `PI_VISION_PROXY_MODEL` | `provider/model-id` | `anthropic/claude-sonnet-4-5` |
| `PI_VISION_PROXY_INCLUDE_CONTEXT` | bool | `true` |
| `PI_VISION_PROXY_TOOL` | `on`, `off` | `on` |
| `PI_VISION_PROXY_MAX_IMAGES_PER_CALL` | 1–20 | `10` |
| `PI_VISION_PROXY_MAX_BATCH` | 1–10 | `4` |
| `PI_VISION_PROXY_CACHE_SIZE` | 0–500 | `50` |
| `PI_VISION_PROXY_MAX_IMAGE_BYTES` | positive integer | `10485760` (10 MB) |
| `PI_VISION_PROXY_ALLOW_HOME` | `1` to allow files under your home directory on non-drive platforms/volumes | not set |
| `PI_VISION_PROXY_ALLOW_DRIVES` | `0`/`false`/`off` to disable local Windows drive paths; otherwise local drive paths like `D:\Downloads\image.png` are allowed | enabled by default |

When an env var is set, the matching `/vision-proxy` subcommand is locked.

## How it works

```
User sends prompt + image(s)
        │
        ▼
  before_agent_start
        │
        ├─ Mode "off" → skip
        ├─ Mode "fallback" + active model supports images → skip
        ├─ Mode "always" OR active model can't see images:
        │       │
        │       ├─ Send images IN PARALLEL to vision model
        │       ├─ If ≥2 images: joint comparison call with adaptive prompt
        │       ├─ Persist each description as session entry (keyed by image hash)
        │       └─ Inject fenced descriptions into system prompt
        │
        ▼
  context (every LLM call)
        │
        └─ Replace each image block with persisted description text,
           so descriptions survive across turns
        │
        ▼
  analyze_image tool (when enabled)
        │
        ├─ Agent sends targeted question + optional crop
        ├─ Image cropped locally (ImageScript), ONLY cropped region sent to vision model
        ├─ Result cached by (hashes, crop, question, model)
        ├─ Max 10 tool calls per turn (rate limit)
        └─ Returned in <vision_proxy_analysis> fence with metadata
```

### Fence tags

| Tag | Purpose |
|-----|---------|
| `<vision_proxy_description>` | Auto-proxy per-image generic description |
| `<vision_proxy_analysis>` | Tool or describe command targeted analysis |
| `<vision_proxy_joint_description>` | Multi-image comparison description |

All fences carry `width`, `height`, `filename`, and optional `crop_origin` and `grounding_format` attributes. Closing-tag neutralisation is applied to all fence bodies.

### Grounding formats

When a model is in the grounding registry, a format-specific instruction is appended to the system prompt. The model's native coordinate format is recorded in the response fence so the agent knows how to interpret it.

| Format | Models | Convention |
|--------|--------|------------|
| `qwen_pixels` | Qwen2.5-VL, Qwen3-VL | `[x1, y1, x2, y2]` absolute pixels |
| `molmo_points` | Molmo2 | `<point x="%" y="%" alt="..."/>` |
| `deepseek_bbox` | DeepSeek-VL2 | `<{skip}\|ref{skip}>...<{skip}\|det{skip}>[[x1,y1,x2,y2]]` |
| `internvl_pixels` | InternVL3 | `[x1, y1, x2, y2]` absolute pixels |
| `gemini_normalized_1000` | Gemini 2.5/3 Pro | Normalized 0–1000 |

## Privacy & security

This extension **sends image data to a third-party provider**. By default that is `anthropic/claude-sonnet-4-5`. Be aware:

1. **Image data is uploaded** to the configured provider on every proxied request. Crop coordinates are applied locally before upload — only the cropped region is sent.
2. **Recent conversation context** (last 8 messages, truncated) is uploaded with the image unless you set `/vision-proxy context off` or `PI_VISION_PROXY_INCLUDE_CONTEXT=false`. Disable it for sensitive sessions.

## Security

The extension includes multiple layers of defense:
- **Path traversal protection** — `..` segments and symlink escapes are rejected
- **Fence injection resistance** — closing tags in model responses are neutralised
- **Config sanitization** — persistent config only loads known keys; prototype pollution blocked
- **Image decode bomb protection** — max 16K × 16K pixel dimensions prevent memory exhaustion
- **Tool call rate limiting** — max 10 calls per turn
- **Telemetry sanitization** — all logged fields stripped of control characters and length-limited

See `SECURITY-REVIEW.md` for the full audit.
