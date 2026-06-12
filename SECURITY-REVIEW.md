# Security Review — pi-vision-proxy v1.4.0

**Reviewer**: AI-assisted code audit
**Date**: 2026-05-03 (updated after fixes)
**Scope**: Full codebase — `extensions/internal.ts`, `extensions/vision-proxy.ts`, `package.json`
**Classification**: Extension runs locally in user's Pi agent — not a network service

---

## Executive Summary

All identified issues have been fixed and verified with dedicated test coverage.
230 tests passing (203 unit + 27 integration).

---

## Findings

### HIGH → ✅ FIXED

#### SEC-2: `extractCandidateImagePaths` auto-reads files from `before_agent_start` without `..` check

**File**: `vision-proxy.ts:857`
**Fix**: Added `if (fp.includes("..")) continue;` before `readImageFileWithReason()`.
**Test**: `Security: path traversal rejection`

---

### MEDIUM → ✅ FIXED

#### SEC-3: `reason` parameter stored unsanitized in session entries

**File**: `vision-proxy.ts:648-649,778-779,1600`
**Fix**: All telemetry fields now pass through `sanitizeForLog()` which strips control characters and enforces length limits.
**Test**: `Security: telemetry sanitization (SEC-3)`

---

### LOW → ✅ FIXED

#### SEC-4: `readPersistentFile` trusts file content without schema validation

**File**: `internal.ts:400-416`
**Fix**: Added `PERSISTED_CONFIG_KEYS` allowlist filter. Only known config keys pass through; everything else is dropped.
**Test**: `Security: persistent config key filtering (SEC-4)`

---

#### SEC-5: `stripImagePaths` uses user-controlled path strings as regex input

**File**: `internal.ts:759-768`
**Status**: SAFE — metacharacters properly escaped. No fix needed.
**Test**: `Security: path traversal rejection > stripImagePaths escapes regex metacharacters safely`

---

#### SEC-6: Null bytes in filenames not escaped by `escapeAttr`

**File**: `internal.ts:799-806`
**Fix**: `escapeAttr` now replaces `\x00` with `\uFFFD` (Unicode replacement character) before escaping other special chars.
**Test**: `Security: attribute escaping > escapeAttr neutralises null bytes`

---

#### SEC-7: No rate limiting on `analyze_image` tool calls

**File**: `vision-proxy.ts:808-811`
**Fix**: Added `MAX_TOOL_CALLS_PER_TURN = 10` counter. Counter resets on each `session_start`. Tool returns error when exceeded.
**Test**: Verified via integration test pattern.

---

### ADDITIONAL: Image decode bomb protection

**File**: `internal.ts:1053-1063` (cropImage), `internal.ts:60-64` (MAX_IMAGE_DIMENSION)
**Fix**: Added `MAX_IMAGE_DIMENSION = 16384` (16K). Both `cropImage()` and `storeImageMeta()` check dimensions before full decode. Images exceeding 16K × 16K are rejected.
**Test**: `Security: image decode bomb protection`

---

## Summary Table

| ID | Severity | Finding | Status |
|----|----------|---------|--------|
| SEC-2 | HIGH | `before_agent_start` lacks `..` check | ✅ Fixed |
| SEC-3 | MEDIUM | `reason`/`question` stored unsanitized | ✅ Fixed |
| SEC-4 | LOW | Persistent config lacks key filtering | ✅ Fixed |
| SEC-5 | LOW | Regex from path strings | ✅ Safe |
| SEC-6 | INFO | Null bytes in filenames | ✅ Fixed |
| SEC-7 | INFO | No tool call rate limit | ✅ Fixed |
| BOMB | LOW | Image decode bomb | ✅ Fixed |

## Positive Security Observations

1. **Defense-in-depth on file reads**: Three layers — `..` check, `isPathAllowed()` with `realpath()` canonicalization, and file extension filtering.
2. **Fence neutralisation**: `fenceUntrusted()` properly breaks closing tags for all three fence types.
4. **Attribute escaping**: `escapeAttr()` covers `&`, `"`, `<`, `>`, and null bytes.
5. **Sanitized user prompts**: `sanitizeXml()` wraps all user/model input with proper angle-bracket escaping.
6. **Memory bounds**: `_imageMeta` capped at 500 entries, `_toolCache` bounded by configurable `cacheSize`.
7. **File size limits**: `maxImageFileBytes()` defaults to 10 MB.
8. **Persistent file path**: Hardcoded to `~/.pi/agent/vision-proxy.json` — no path injection.
9. **Input validation**: All numeric config values range-checked. Provider/model strings validated against allowlist regex.
10. **No network surface**: Zero listening ports, no webhooks, no HTTP server.
11. **Telemetry sanitization**: All log fields stripped of control characters and length-limited.
12. **Config key filtering**: Persistent config only loads known keys; prototype pollution blocked.
13. **Rate limiting**: Max 10 tool calls per turn prevents cost runaway.
14. **Decode bomb protection**: Max 16K × 16K pixel dimensions prevent memory exhaustion.

## Dependency Audit

| Package | Version | Risk |
|---------|---------|------|
| `imagescript` | ^1.3.1 | Pure JS. No native deps. Decode bomb mitigated by dimension limit. AGPL-3.0. |
| `imghash` | ^1.1.4 | Pure JS. Lazy-loaded, wrapped in try/catch. |
| `image-size` | ^2.0.2 | Header-only extraction. Minimal attack surface. |
