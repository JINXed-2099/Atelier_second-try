---
id: R-MEDIA-01
version: 1
tier: 0
type: rune
domain: MEDIA
status: draft
created: 2026-09-17
last_modified: 2026-09-17
produces: { url: string, width: number, height: number, format: string }
consumes: { source_url: string, target_width: number, target_height: number }
error_codes: [SOURCE_NOT_FOUND, RESIZE_FAILED, UNSUPPORTED_FORMAT]
imports: []
deprecated_versions: []
tags: [tier/0, type/rune, domain/MEDIA]
---

# R-MEDIA-01 — Resize Image

## Purpose
Resizes an image from a source URL to the specified dimensions.

## Qualification gates
- [x] **Gate 1 — Space neutral:** Zero references to any Space
- [x] **Gate 2 — Single responsibility:** Resizes one image
- [x] **Gate 3 — Zero dependencies:** No Codex imports
- [x] **Gate 4 — Generic interface:** Strings and numbers in/out
- [x] **Gate 5 — Independent testability:** Only needs a storage mock

## Interface

### Input (`consumes`)
```
{
  source_url: string,
  target_width: number,
  target_height: number
}
```

### Output (`produces`)
```
{
  url: string,          // URL of the resized image
  width: number,
  height: number,
  format: string        // "jpg", "png", "webp"
}
```

## Error codes
| Code | When it fires | Sigil message |
|---|---|---|
| SOURCE_NOT_FOUND | source_url returns 404 or is unreachable | Source image not found at {source_url} |
| RESIZE_FAILED | Processing error during resize | Image resize failed: {reason} |
| UNSUPPORTED_FORMAT | Source image is not jpg, png, or webp | Unsupported image format: {format} |

## Behavior
1. Fetch the image from `source_url`.
2. Validate format is supported. If not → throw UNSUPPORTED_FORMAT Sigil.
3. Resize to target dimensions, preserving aspect ratio.
4. Upload resized image to storage.
5. Return the new URL and dimensions.

## External dependencies
- Image processing library (e.g., Sharp)
- Cloud storage client

## Notes
Used heavily by A-PROFILE-01 for avatar processing and A-GALLERY-01 for thumbnails.
