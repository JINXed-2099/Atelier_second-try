---
id: W-TRUST-01
version: 1
tier: 1b
type: ward
domain: TRUST
status: draft
created: 2026-09-17
last_modified: 2026-09-17
produces: { trust_tier: string, score: number, can_bypass_moderation: boolean }
consumes: { user_id: string }
imports: [R-DATA-01.v1, R-EVENT-01.v1]
error_codes: [TRUST_CALC_FAILED, INSUFFICIENT_HISTORY]
deprecated_versions: []
tags: [tier/1b, type/ward, domain/TRUST]
---

# W-TRUST-01 — Calculate Trust Score

## Purpose
Computes a user's current trust tier based on their contribution history.

## Rune composition
| Rune | Why it's needed | What it provides |
|---|---|---|
| [[R-DATA-01.v1]] | Fetches the user record for account age | User record with join date |
| [[R-EVENT-01.v1]] | Retrieves contribution events for scoring | Event count and types |

## Tier compliance
- [x] Imports only Runes (Tier 0)
- [x] No imports from Arcanes, other Wards, Rituals, or Spaces
- [x] Independent of the composition track (Arcanes)

## Interface

### Input (`consumes`)
```
{
  user_id: string
}
```

### Output (`produces`)
```
{
  trust_tier: string,             // "new" | "member" | "trusted" | "elder"
  score: number,                  // 0–100
  can_bypass_moderation: boolean  // true for "trusted" and "elder"
}
```

## Governance logic
Trust tiers are calculated from three weighted signals:
- Account age (days since joined) — 20% weight
- Contribution count (helpful critiques, discussions, uploads) — 50% weight
- Moderation record (flags, warnings, bans) — 30% weight (negative)

Tier thresholds:
- `new`: score 0–24
- `member`: score 25–54
- `trusted`: score 55–84
- `elder`: score 85–100

`can_bypass_moderation` is true only for `trusted` and `elder`.

## Used by Rituals
- [[X-GUARD-01.v1]] — combines trust with moderation for auto-approval decisions

## Error handling
| Caught Sigil (from Rune) | Wrapped as | Ward error code |
|---|---|---|
| R-DATA-01 → RECORD_NOT_FOUND | TRUST_CALC_FAILED | Cannot score unknown user |
| R-EVENT-01 → any error | INSUFFICIENT_HISTORY | No contribution history available |

## Notes
This Ward is critical to the maturity gradient. The trust score directly governs what a user can do across all Spaces.
