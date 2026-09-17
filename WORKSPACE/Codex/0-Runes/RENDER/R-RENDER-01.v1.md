---
id: R-RENDER-01
version: 1
tier: 0
type: rune
domain: RENDER
status: draft
created: 2026-09-17
last_modified: 2026-09-17
produces: { html: string, css_class: string }
consumes: { template_id: string, data: object }
error_codes: [TEMPLATE_NOT_FOUND, RENDER_FAILED]
imports: []
deprecated_versions: []
tags: [tier/0, type/rune, domain/RENDER]
---

# R-RENDER-01 — Render Template

## Purpose
Renders a UI template with provided data into an HTML string.

## Qualification gates
- [x] **Gate 1 — Space neutral:** Zero references to any Space
- [x] **Gate 2 — Single responsibility:** Renders one template
- [x] **Gate 3 — Zero dependencies:** No Codex imports
- [x] **Gate 4 — Generic interface:** String + object in, string out
- [x] **Gate 5 — Independent testability:** No Codex mocks needed

## Interface

### Input (`consumes`)
```
{
  template_id: string,
  data: object            // key-value pairs the template expects
}
```

### Output (`produces`)
```
{
  html: string,
  css_class: string       // the template's root CSS class
}
```

## Error codes
| Code | When it fires | Sigil message |
|---|---|---|
| TEMPLATE_NOT_FOUND | No template registered under template_id | Template not found: {template_id} |
| RENDER_FAILED | Data doesn't satisfy template requirements | Render failed: missing key {key} |

## Behavior
1. Look up template by `template_id`.
2. Validate that `data` contains all required keys. If not → throw RENDER_FAILED Sigil.
3. Interpolate data into template.
4. Return rendered HTML and root CSS class.

## External dependencies
- Template engine (e.g., Handlebars, EJS)

## Notes
Generic renderer. Spaces define which templates exist; this Rune only processes them.
