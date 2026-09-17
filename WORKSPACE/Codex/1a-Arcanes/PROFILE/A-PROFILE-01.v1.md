---
id: A-PROFILE-01
version: 1
tier: 1a
type: arcane
domain: PROFILE
status: draft
created: 2026-09-17
last_modified: 2026-09-17
produces: { html: string, css_class: string, user_id: string, display_name: string }
consumes: { user_id: string, avatar_size: number }
imports: [R-DATA-01.v1, R-MEDIA-01.v1, R-RENDER-01.v1]
error_codes: [PROFILE_BUILD_FAILED, PARTIAL_RENDER]
deprecated_versions: []
tags: [tier/1a, type/arcane, domain/PROFILE]
---

# A-PROFILE-01 — Build Profile Card

## Purpose
Assembles a complete user profile card by fetching user data, processing the avatar, and rendering the result.

## Rune composition
| Rune | Why it's needed | What it provides |
|---|---|---|
| [[R-DATA-01.v1]] | Fetches the user record | id, username, display_name, avatar_url |
| [[R-MEDIA-01.v1]] | Resizes avatar to card dimensions | Processed avatar URL |
| [[R-RENDER-01.v1]] | Renders the profile card template | Final HTML output |

## Tier compliance
- [x] Imports only Runes (Tier 0)
- [x] No imports from other Arcanes, Wards, Rituals, or Spaces
- [x] Space-neutral — no reference to any specific Space

## Interface

### Input (`consumes`)
```
{
  user_id: string,
  avatar_size: number     // pixels, square
}
```

### Output (`produces`)
```
{
  html: string,
  css_class: string,
  user_id: string,
  display_name: string
}
```

## Orchestration flow
1. Call [[R-DATA-01.v1]] with `user_id` → get user record.
2. Call [[R-MEDIA-01.v1]] with `user.avatar_url` and `avatar_size` → get resized avatar.
3. Call [[R-RENDER-01.v1]] with template `"profile-card"` and combined data → get HTML.
4. Return assembled profile card.

If step 1 fails → throw PROFILE_BUILD_FAILED (no user, nothing to build).
If step 2 fails → continue with a placeholder avatar, set code to PARTIAL_RENDER.
If step 3 fails → throw PROFILE_BUILD_FAILED.

## Error handling
| Caught Sigil (from Rune) | Wrapped as | Arcane error code |
|---|---|---|
| R-DATA-01 → RECORD_NOT_FOUND | PROFILE_BUILD_FAILED | User not found, cannot build profile |
| R-DATA-01 → DB_TIMEOUT | PROFILE_BUILD_FAILED | Data layer unreachable |
| R-MEDIA-01 → any error | PARTIAL_RENDER | Profile built with placeholder avatar |
| R-RENDER-01 → RENDER_FAILED | PROFILE_BUILD_FAILED | Template render failed |

## Notes
This is one of the most-used Arcanes — nearly every Space needs to display user profiles. The graceful degradation on avatar failure (PARTIAL_RENDER instead of full failure) is deliberate — a missing avatar shouldn't prevent showing the profile.
