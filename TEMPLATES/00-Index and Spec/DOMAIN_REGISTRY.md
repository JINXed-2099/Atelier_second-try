---
type: registry
last_modified: 2026-09-17
---

# Domain Registry

> No component uses a domain not listed here. To add one, propose to the team and log in [[DECISION_LOG]].

## Tier 0 — Rune domains

| Domain | Covers | Added |
|---|---|---|
| AUTH | Identity verification, session tokens, hashing | 2026-09-17 |
| DATA | CRUD operations, queries, record retrieval | 2026-09-17 |
| MEDIA | Image/file upload, resize, format conversion | 2026-09-17 |
| RENDER | Layout assembly, template rendering, formatting | 2026-09-17 |
| VALID | Input validation, sanitization, type checking | 2026-09-17 |
| SEARCH | Indexing, querying, filtering, sorting | 2026-09-17 |
| TIME | Timestamps, scheduling, date math | 2026-09-17 |
| NOTIFY | Message dispatch (the act of sending — not policy) | 2026-09-17 |
| EVENT | Event emission, logging hooks, activity recording | 2026-09-17 |

## Tier 1a — Arcane domains

| Domain | Covers | Added |
|---|---|---|
| PROFILE | User profile assembly, display, editing flows | 2026-09-17 |
| THREAD | Conversation/thread lifecycle management | 2026-09-17 |
| CONTENT | Content creation, editing, publishing pipelines | 2026-09-17 |
| FEED | Feed generation, aggregation, sorting | 2026-09-17 |
| GALLERY | Collection display, grid assembly, browsing | 2026-09-17 |
| ONBOARD | Signup flows, first-run experiences, tutorials | 2026-09-17 |

## Tier 1b — Ward domains

| Domain | Covers | Added |
|---|---|---|
| TRUST | Reputation scoring, trust tiers, maturity tracking | 2026-09-17 |
| PERM | Role-based access, visibility controls, gating | 2026-09-17 |
| MOD | Content moderation, flagging, review queues | 2026-09-17 |
| AUDIT | Action logging, change history, accountability trail | 2026-09-17 |
| RATE | Rate limiting, throttling, abuse prevention | 2026-09-17 |

## Tier 1c — Ritual domains

| Domain | Covers | Added |
|---|---|---|
| GUARD | Safety decisions (moderation + trust + rate limiting) | 2026-09-17 |
| GATE | Access decisions (authentication + permissions) | 2026-09-17 |
| SIGNAL | Event orchestration (audit + notification dispatch) | 2026-09-17 |

## Tier 2 — Space domains

| Domain | Covers | Added |
|---|---|---|
| ODYSSEY | Odyssey-specific flows | 2026-09-17 |
| DISC | Discussion-specific flows | 2026-09-17 |
| CRIT | Critique-specific flows | 2026-09-17 |
| SHOW | Showcase-specific flows | 2026-09-17 |
| RES | Resources-specific flows | 2026-09-17 |

---

## Adding a new domain

1. Search this file — does an existing domain cover your case?
2. Write: `[NAME] — [what it covers]`
3. Confirm with team it doesn't overlap.
4. Add to the table. Log in [[DECISION_LOG]].
