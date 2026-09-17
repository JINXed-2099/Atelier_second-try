---
id: S-CRIT-01
version: 1
tier: 2
type: space
domain: CRIT
space: critique
status: draft
created: 2026-09-17
last_modified: 2026-09-17
produces: { thread_id: string, status: string }
consumes: { user_id: string, work_id: string, critique_body: string }
imports: [A-PROFILE-01.v1, X-GUARD-01.v1, R-DATA-01.v1]
error_codes: [SUBMISSION_BLOCKED, CRITIQUE_CREATE_FAILED]
deprecated_versions: []
tags: [tier/2, type/space, domain/CRIT]
---

# S-CRIT-01 — Submit Critique

## Purpose
Handles the full flow when a user submits a critique in the Critique Space — from safety check to profile display to thread creation.

## Lower-tier composition

### Arcanes
| Arcane | Why it's needed |
|---|---|
| [[A-PROFILE-01.v1]] | Displays the critique author's profile card alongside their submission |

### Rituals
| Ritual | Why it's needed |
|---|---|
| [[X-GUARD-01.v1]] | Runs the auto-moderation decision before the critique is published |

### Direct Runes
| Rune | Why not an Arcane? |
|---|---|
| [[R-DATA-01.v1]] | Simple record fetch for the work being critiqued — no composition needed |

## Tier compliance
- [x] Imports only from lower tiers
- [x] No imports from other Spaces (cross-Space isolation)
- [x] All shared logic lives in lower tiers

## User flow
1. User writes a critique and hits Submit.
2. S-CRIT-01 calls [[X-GUARD-01.v1]] with the user's ID and critique content.
3. If decision = `reject` → show rejection reason to user, stop.
4. If decision = `queue` → notify user their critique is pending review, save as draft.
5. If decision = `approve` → proceed:
   a. Call [[R-DATA-01.v1]] to fetch the work being critiqued.
   b. Call [[A-PROFILE-01.v1]] to build the author's profile card.
   c. Create the critique thread, attach profile card, publish.
6. Return thread_id and status.

## Error handling
| Caught Sigil | User-facing message | Logged detail |
|---|---|---|
| X-GUARD-01 → GUARD_CHECK_FAILED | "Your submission is being reviewed" | Guard system down, defaulted to queue |
| A-PROFILE-01 → PROFILE_BUILD_FAILED | "Something went wrong — please try again" | Profile assembly failed |
| A-PROFILE-01 → PARTIAL_RENDER | *(no user message — profile shows with placeholder avatar)* | Avatar unavailable, placeholder used |
| R-DATA-01 → RECORD_NOT_FOUND | "The work you're critiquing was not found" | Work ID invalid or deleted |

## Notes
This is the primary entry point for the Critique Space. Notice how the Sigil chain gives full traceability: if the user sees "Something went wrong," the log shows S-CRIT-01 → A-PROFILE-01 → R-DATA-01 → DB_TIMEOUT, pinpointing exactly where it broke.
