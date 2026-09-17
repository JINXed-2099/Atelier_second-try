---
id: X-GUARD-01
version: 1
tier: 1c
type: ritual
domain: GUARD
status: draft
created: 2026-09-17
last_modified: 2026-09-17
produces: { decision: string, reason: string, requires_review: boolean }
consumes: { user_id: string, content_id: string, action_type: string }
imports: [W-TRUST-01.v1, W-MOD-01.v1, W-RATE-01.v1]
error_codes: [GUARD_CHECK_FAILED, RATE_EXCEEDED]
deprecated_versions: []
tags: [tier/1c, type/ritual, domain/GUARD]
---

# X-GUARD-01 — Auto-Moderation Decision

## Purpose
Combines trust score, moderation rules, and rate limits into a single approve/queue/reject decision for user-submitted content.

## Ward composition
| Ward | Why it's needed | What it provides |
|---|---|---|
| [[W-TRUST-01.v1]] | Determines user's trust tier | Trust tier and bypass eligibility |
| [[W-MOD-01.v1]] | Checks content against moderation rules | Flag status and violation type |
| [[W-RATE-01.v1]] | Checks if user has exceeded submission rate limits | Rate status and cooldown remaining |

## Tier compliance
- [x] Imports only Wards (Tier 1b) and Runes (Tier 0)
- [x] No imports from Arcanes, other Rituals, or Spaces
- [x] Independent of the composition track

## Interface

### Input (`consumes`)
```
{
  user_id: string,
  content_id: string,
  action_type: string     // "post" | "critique" | "upload" | "comment"
}
```

### Output (`produces`)
```
{
  decision: string,         // "approve" | "queue" | "reject"
  reason: string,           // human-readable explanation
  requires_review: boolean  // true if queued for manual review
}
```

## Orchestration flow
1. Call [[W-RATE-01.v1]] → if rate exceeded → return `reject` immediately.
2. Call [[W-TRUST-01.v1]] → get trust tier and bypass eligibility.
3. Call [[W-MOD-01.v1]] → check content against rules.
4. Decision matrix:
   - Trust bypass = true AND mod clean → `approve`
   - Trust bypass = true AND mod flagged → `queue` (trusted users get review, not auto-reject)
   - Trust bypass = false AND mod clean → `approve`
   - Trust bypass = false AND mod flagged → `reject`

## Decision logic
Rate limiting is checked first because it's the cheapest check and the hardest gate — no trust level overrides a rate limit. Trust is checked before moderation because it changes how mod flags are handled (trusted users get queued for review instead of auto-rejected).

## Error handling
| Caught Sigil (from Ward) | Wrapped as | Ritual error code |
|---|---|---|
| W-RATE-01 → any error | GUARD_CHECK_FAILED | Rate check unavailable — defaulting to queue |
| W-TRUST-01 → TRUST_CALC_FAILED | GUARD_CHECK_FAILED | Trust check failed — defaulting to queue |
| W-MOD-01 → any error | GUARD_CHECK_FAILED | Moderation check unavailable — defaulting to queue |

**Fail-safe:** If any Ward fails, the Ritual defaults to `queue` (not approve, not reject). This ensures content is never silently approved when governance is down, and users aren't unfairly rejected by a system failure.

## Notes
This is the central safety Ritual. Every Space calls it before publishing user content. The fail-safe-to-queue design is deliberate — when in doubt, a human reviews.
