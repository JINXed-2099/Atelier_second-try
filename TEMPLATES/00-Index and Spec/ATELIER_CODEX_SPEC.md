# Atelier Codex — Architecture Specification

> The Codex is Atelier's layered component architecture. Everything built on the platform draws from this tiered system. Dependencies only flow downward. When something breaks, the damage stays in the layer it broke in.

---

## 1. Tier definitions

### Tier 0 — Runes
The smallest indivisible units of functionality. A Rune does exactly one thing, knows nothing about which Space calls it, and depends on nothing — not even other Runes. Runes are the bedrock — if this layer is solid, nothing forces a rebuild from scratch.

**Examples:** fetch a user record, validate an input field, resize an image, generate a timestamp, hash a password.

### Tier 1a — Arcanes
Composed spells. An Arcane bundles multiple Runes into a higher-order operation that is still generic (not tied to any specific Space). Think of it as a class that orchestrates atomic methods.

**Examples:** build a user profile card (calls R-DATA for user record + R-MEDIA for avatar + R-RENDER for layout), compose a threaded conversation (calls R-DATA for messages + R-AUTH for session verification + R-RENDER for thread view).

**Dependency rule:** Arcanes may import Runes only. Never other Arcanes, never Wards, never Rituals, never Spaces.

### Tier 1b — Wards
Persistent enchantments — cross-cutting services that govern behavior across all Spaces. Wards handle concerns that don't belong to any single Space but affect all of them: trust/reputation, permissions, notifications, moderation, audit logging.

**Dependency rule:** Wards may import Runes only. Wards and Arcanes are independent peers — neither may import the other.

**Why parallel, not stacked:** If Wards depended on Arcanes, a broken Arcane could cascade into permissions or trust — the very systems meant to be most stable. By keeping them on the same tier with no lateral dependency, either can break without affecting the other.

### Tier 1c — Rituals
Ward orchestration spells. A Ritual exists for one reason: to compose multiple Wards (and optionally Runes) into a coordinated decision. Wards are deliberately kept independent of each other, but real-world logic often needs their outputs combined — "check the user's trust score, then decide whether to auto-approve or send to moderation." That coordination is a Ritual.

**Examples:** auto-moderation decision (calls W-TRUST for score + W-MOD for queue routing + W-RATE for throttle check), access gate (calls W-PERM for role check + W-TRUST for tier verification).

**Dependency rule:** Rituals may import Wards and Runes. Never Arcanes, never other Rituals, never Spaces.

**Why Rituals exist:** Without them, Ward orchestration would be duplicated across every Space that needs it. Five Spaces each wiring W-MOD + W-TRUST the same way is five places to maintain, five places for inconsistency to creep in. A Ritual centralizes that logic in one place. Spaces call the Ritual instead of wiring the Wards themselves.

**Why Rituals cannot import Arcanes:** Rituals govern safety, trust, and access — the most sensitive decisions in the platform. If a Ritual depended on an Arcane (a UI composition or content pipeline), a broken Arcane could cascade into moderation or permissions. Rituals stay in the governance track (Wards + Runes only), completely independent of the composition track (Arcanes).

### Tier 2 — Spaces (Framework)
The five major Spaces (Odyssey, Discussion, Critique, Showcase, Resources) and any future Spaces. This is where all space-specific logic lives. Spaces consume Runes, Arcanes, Wards, and Rituals freely.

**This is the layer where things are allowed to break.** A broken Critique flow should never require touching a Rune, Ward, or Ritual — only rewiring how Critique calls them.

**Cross-Space isolation:** Spaces cannot import from other Spaces. Critique cannot reach into Showcase's logic, and vice versa. If two Spaces need similar behavior, they both call the same Arcanes, Wards, or Rituals independently. This ensures a broken Space never cascades into another Space.

---

## 2. Rune qualification criteria

A function must pass **all five gates** to qualify as a Rune. If it fails any one, it belongs in a higher tier.

### Gate 1 — Space neutrality
The function must contain zero references to any specific Space (Odyssey, Discussion, Critique, Showcase, Resources). Not in its name, not in its parameters, not in its internal logic, not in its documentation.

**Test:** Remove all Space names from the codebase. Does this function still make complete sense? If yes → pass. If it becomes meaningless without a Space name → it belongs in the framework.

### Gate 2 — Single responsibility
The function does exactly one thing. If you need the word "and" to describe what it does, it's two Runes or an Arcane.

**Test:** Can you describe it in one sentence without a conjunction? "Fetches a user record by ID" → pass. "Fetches a user record and checks their permissions" → fail (split into R-DATA + R-AUTH, or compose as an Arcane).

### Gate 3 — Zero dependencies
The function imports nothing from the Codex — not Arcanes, not Wards, not Rituals, not Spaces, and not other Runes. It may import standard libraries and external packages only. If it needs another Rune's output, that composition is an Arcane by definition.

**Test:** Delete the entire `/codex/` folder. Does this function still compile and run with only its standard library and external package imports? If yes → pass.

### Gate 4 — Generic interface
Inputs and outputs use generic types (strings, numbers, objects, arrays, booleans), not structures defined by a specific Space or Arcane. A Rune should not require `CritiqueThread` as an input — it should accept a generic `threadId: string`.

**Test:** Could a completely different platform (not Atelier) use this function without modification? If yes → pass.

### Gate 5 — Independent testability
The function can be unit-tested with no mocking of any Codex component — no Runes, no Arcanes, no Wards, no Rituals, no Spaces. It may need mocks for external services (database, API) but never for anything inside the Codex.

**Test:** Write the test file. If your imports include anything from `/runes/`, `/arcanes/`, `/wards/`, `/rituals/`, or `/spaces/` — even for test fixtures — the function fails this gate.

---

## 3. Versioning protocol

### Version format

```
[ID].v[MAJOR]
```

Examples: `R-AUTH-01.v1`, `A-PROFILE-01.v3`, `W-TRUST-01.v2`

There is no minor version. A change either preserves the interface (internal patch, no version bump needed — just update the implementation) or it breaks the interface (new major version).

### When to version

| Change type | Action | Example |
|---|---|---|
| Bug fix, performance improvement, refactor — same inputs, same outputs | Patch in place, no version change | Optimizing R-DATA-01's query without changing what it accepts or returns |
| New optional parameter added — old callers unaffected | Patch in place, no version change | Adding an optional `limit` param to R-DATA-01 |
| Required parameter added, parameter removed, output shape changed, parameter type changed | **New major version** | R-AUTH-01.v1 returns `boolean` → R-AUTH-01.v2 returns `{ allowed: boolean, reason: string }` |

### Breaking change rules

1. **The old version is never deleted.** R-AUTH-01.v1 remains callable indefinitely (or until an explicit sunset, see below).
2. **The new version gets a new file**, not an edit to the old one. `R-AUTH-01.v1.md` stays untouched. `R-AUTH-01.v2.md` is created alongside it.
3. **A migration note is mandatory.** The new version's file must include a `## Migration from v[N-1]` section explaining what changed, why, and how to update callers.
4. **Deprecation is explicit.** When a version is superseded, its file gets a `[DEPRECATED — use v2]` tag at the top. It still works; the tag is a signal to migrate.
5. **Sunset requires a team decision.** A deprecated version can only be removed after all callers have migrated AND the team agrees. This is logged in the decision log.

### Version in the component file

```markdown
---
id: R-AUTH-01
version: 2
tier: 0
domain: AUTH
status: active
deprecated_versions: [1]
created: 2026-09-16
last_modified: 2026-09-16
breaking_change: true
produces: { allowed: boolean, reason: string }
consumes: { token: string }
---

## Migration from v1
v1 returned `boolean`. v2 returns `{ allowed: boolean, reason: string }`.
Update all callers to destructure the response.
```

### Data interface fields — `produces` and `consumes`

Every component declares what data shape it accepts (`consumes`) and what shape it returns (`produces`) in its frontmatter. This makes the contract between tiers explicit rather than implicit.

**Rules:**
- A Rune's `produces` field is a promise. Any Arcane, Ward, or Ritual that calls this Rune can rely on that shape.
- An Arcane's `consumes` field references the shapes it expects from the Runes it imports. If R-DATA-01's `produces` changes, every component whose `consumes` references that shape is flagged.
- A change to a `produces` field is always a breaking change, even if the implementation is "just adding a field." The version protocol applies.
- The linter can cross-check: if A-PROFILE-01 `consumes: { id: string, name: string }` from R-DATA-01, but R-DATA-01 `produces: { id: string, display_name: string }` — the mismatch is caught before deployment.

**Future evolution — Contracts layer:** When the Codex grows past ~30 components and `produces`/`consumes` fields become hard to manage inline, extract them into a dedicated `/codex/contracts/` folder. Each contract becomes a standalone definition file that components reference by name instead of inlining the shape. Contracts will have their own versioning and index. This is not needed now — the frontmatter fields give you 90% of the safety with none of the overhead. You'll know it's time to formalize when someone patches a Rune's output and silently breaks three Arcanes despite the frontmatter being "right there."

---

## 4. Component taxonomy and naming

### ID format

```
[PREFIX]-[DOMAIN]-[NN]
```

- **Prefix** identifies the tier
- **Domain** identifies the functional area (space-neutral for Runes, still generic for Arcanes/Wards/Rituals, space-specific for Spaces)
- **NN** is a two-digit sequential number within that domain

### Tier prefixes

| Tier | Prefix | Example |
|---|---|---|
| Rune | `R` | R-AUTH-01 |
| Arcane | `A` | A-PROFILE-01 |
| Ward | `W` | W-TRUST-01 |
| Ritual | `X` | X-GUARD-01 |
| Space | `S` | S-CRIT-01 |

### Domain registry

Domains are registered centrally. No one invents a new domain without adding it to the registry first. This prevents drift and duplication ("AUTH" vs "ACCESS" vs "LOGIN" for the same thing).

**Rune domains (Tier 0) — must be space-neutral:**

| Domain | Covers |
|---|---|
| AUTH | Identity verification, session tokens, hashing |
| DATA | CRUD operations, queries, record retrieval |
| MEDIA | Image/file upload, resize, format conversion |
| RENDER | Layout assembly, template rendering, formatting |
| VALID | Input validation, sanitization, type checking |
| SEARCH | Indexing, querying, filtering, sorting |
| TIME | Timestamps, scheduling, date math |
| NOTIFY | Message dispatch (the act of sending — not policy) |
| EVENT | Event emission, logging hooks, activity recording |

**Arcane domains (Tier 1a) — generic compositions:**

| Domain | Covers |
|---|---|
| PROFILE | User profile assembly, display, editing flows |
| THREAD | Conversation/thread lifecycle management |
| CONTENT | Content creation, editing, publishing pipelines |
| FEED | Feed generation, aggregation, sorting |
| GALLERY | Collection display, grid assembly, browsing |
| ONBOARD | Signup flows, first-run experiences, tutorials |

**Ward domains (Tier 1b) — cross-cutting services:**

| Domain | Covers |
|---|---|
| TRUST | Reputation scoring, trust tiers, maturity tracking |
| PERM | Role-based access, visibility controls, gating |
| MOD | Content moderation, flagging, review queues |
| AUDIT | Action logging, change history, accountability trail |
| RATE | Rate limiting, throttling, abuse prevention |

**Ritual domains (Tier 1c) — Ward orchestration:**

| Domain | Covers |
|---|---|
| GUARD | Safety decisions combining moderation, trust, and rate limiting |
| GATE | Access decisions combining authentication and permissions |
| SIGNAL | Event orchestration combining audit trails and notification dispatch |

**Space domains (Tier 2) — framework-specific:**

| Domain | Covers |
|---|---|
| ODYSSEY | Odyssey-specific flows |
| DISC | Discussion-specific flows |
| CRIT | Critique-specific flows |
| SHOW | Showcase-specific flows |
| RES | Resources-specific flows |

### Naming a new component — checklist

1. Does this domain already exist in the registry? → Use it.
2. Does a component in this domain already do what you need? → Use or extend it (see versioning).
3. Neither? → Propose the new domain/component to the team. Log the decision.

---

## 5. Dependency gates — enforcement

### The rule in one sentence
**A component may only import from tiers below it — never from its own tier, never from above, and never laterally across Tier 1.**

### Tier import matrix

| Caller | Can import Runes (T0) | Can import Arcanes (T1a) | Can import Wards (T1b) | Can import Rituals (T1c) | Can import Spaces (T2) |
|---|---|---|---|---|---|
| Rune (T0) | ✕ (fully self-contained) | ✕ | ✕ | ✕ | ✕ |
| Arcane (T1a) | ✓ | ✕ | ✕ | ✕ | ✕ |
| Ward (T1b) | ✓ | ✕ | ✕ | ✕ | ✕ |
| Ritual (T1c) | ✓ | ✕ | ✓ | ✕ | ✕ |
| Space (T2) | ✓ | ✓ | ✓ | ✓ | ✕ (not other Spaces) |

### How to enforce

Every component file includes a `tier` field in its frontmatter:

```markdown
---
id: R-AUTH-01
tier: 0
...
---
```

**Gate mechanism — three layers of defense:**

**Layer 1 — Folder structure (passive)**
The project enforces a directory layout where each tier has its own folder. Imports from the wrong folder are visually obvious in code review.

```
/codex
  /runes/          ← Tier 0
  /arcanes/        ← Tier 1a
  /wards/          ← Tier 1b
  /rituals/        ← Tier 1c
  /spaces/         ← Tier 2
    /odyssey/
    /discussion/
    /critique/
    /showcase/
    /resources/
```

**Layer 2 — Import validation (active)**
A lightweight linter or pre-commit hook scans each file's imports and checks them against the tier import matrix. If a component imports from a folder it shouldn't, the commit is rejected with a clear error:

```
ERROR: Tier violation in R-AUTH-01
  → imports R-VALID-01 (Tier 0)
  → Runes (Tier 0) are fully self-contained — no Codex imports allowed.
  → If you need both R-AUTH-01 and R-VALID-01 together, compose them as an Arcane.
```

```
ERROR: Tier violation in A-PROFILE-01
  → imports W-PERM-01 (Tier 1b)
  → Arcanes (Tier 1a) cannot import Wards (Tier 1b).
  → Permission-gated profile flows belong in a Space or a Ritual.
```

**Layer 3 — Review gate (human)**
Every new component or modification goes through review. The reviewer checks:
- Is the tier assignment correct? (Apply the 5 qualification gates for Runes)
- Do all imports respect the tier matrix?
- Is the domain correct, or should this live elsewhere?
- Does a component for this already exist? (Check the index)

### What to do when you need an upward or lateral dependency

This is the most common pressure point. **It is always a design signal, not a rule to bend:**

| Situation | Solution |
|---|---|
| Rune needs another Rune | That's an Arcane. Compose both Runes at Tier 1a. |
| Rune needs logic from an Arcane | The Rune is not a Rune. Move it up to an Arcane. |
| Rune needs to know about a Space concept | It belongs in the Space. No lower tier has knowledge of Spaces. |
| Arcane needs another Arcane's output | Build a new Arcane that directly composes the Runes both Arcanes use. Two Arcanes never talk to each other — they share Runes underneath. |
| Ward needs another Ward | That's a Ritual. Compose both Wards at Tier 1c. |
| Ward needs an Arcane's behavior | The Ward composes the same Runes the Arcane uses. Wards and Arcanes are independent peers that happen to draw from the same Rune pool. |
| Arcane needs a Ward | The orchestration belongs in a Space or a Ritual. Arcanes never know about governance. |
| Space needs another Space | Both Spaces call the same lower-tier components independently. Spaces never import from each other. |

---

## 6. Discovery — preventing duplicates

### The index

A single `CODEX_INDEX.md` file in the root lists every component with its ID, name, one-line description, tier, domain, version, and status. This is the first place anyone looks before creating a new component.

```markdown
| ID | Name | Description | Tier | Domain | Version | Status |
|---|---|---|---|---|---|---|
| R-AUTH-01 | Verify session token | Checks if a session token is valid and not expired | 0 | AUTH | v1 | Active |
| R-AUTH-02 | Hash password | Generates a salted hash from a plaintext password | 0 | AUTH | v1 | Active |
| A-PROFILE-01 | Build profile card | Assembles a user's profile display from data + media + layout | 1a | PROFILE | v1 | Active |
| W-TRUST-01 | Calculate trust score | Computes a user's current trust tier from contribution events | 1b | TRUST | v1 | Draft |
| X-GUARD-01 | Auto-moderation decision | Combines trust score + moderation rules + rate limits into approve/queue/reject | 1c | GUARD | v1 | Draft |
```

### Before creating anything new

1. Search the index for keywords related to what you need.
2. Check the domain registry — does a domain for this exist?
3. If a similar component exists, use it or version it — don't create a parallel one.
4. If nothing fits, propose the new component with its ID, tier justification (citing the 5 gates if it's a Rune), and domain assignment.

---

## 7. Sigil — error handling infrastructure

### Why Sigils live outside the Codex

Every tier needs to create and propagate errors. But Runes can't import other Runes, and no tier can import from its own level. So the error system can't be a Rune, an Arcane, a Ward, or a Ritual — wherever you put it inside the Codex, something that needs it would be prohibited from importing it.

The Sigil system is **external infrastructure** — it sits outside the Codex entirely, the same way a standard library or language runtime does. Components import it the way they import any external package. To the Codex's dependency rules, Sigils don't exist in any tier, so no tier rule is violated by using them.

```
/codex
  /runes/
  /arcanes/
  /wards/
  /rituals/
  /spaces/

/infrastructure          ← outside the Codex
  /sigils/               ← error handling system
```

### The Sigil shape

Every error in the platform follows this standard structure:

```
{
  sigil: "CODEX_ERROR",
  tier: 0,                            // which tier threw
  source: "R-DATA-01.v1",            // which component threw
  code: "RECORD_NOT_FOUND",          // machine-readable error type
  message: "No user record for id u_999",  // human-readable
  timestamp: "2026-09-17T10:30:00Z",
  chain: []                           // errors from lower tiers that caused this
}
```

### How errors propagate — the wrapping rule

Each tier catches errors from the tier below, wraps them in a new Sigil with its own context, and pushes the original into the `chain` array. Raw errors from lower tiers never leak upward — the calling tier always gets a Sigil at its own level with the full trace underneath.

**At Tier 0 (Runes):** A Rune is the first point of contact with external services (databases, APIs, file systems). If an external service fails, the Rune wraps the raw external error in a Sigil before throwing. Raw external errors never leave a Rune. The Sigil has `tier: 0`, the Rune's own ID as `source`, and an empty `chain` (there's nothing below a Rune).

```
// R-DATA-01 fails to reach the database
{
  sigil: "CODEX_ERROR",
  tier: 0,
  source: "R-DATA-01.v1",
  code: "DB_TIMEOUT",
  message: "Database query timed out after 5000ms",
  timestamp: "...",
  chain: []
}
```

**At Tier 1a/1b (Arcanes/Wards):** An Arcane calls Runes. If a Rune throws a Sigil, the Arcane catches it, creates a *new* Sigil at its own tier, and nests the Rune's Sigil in `chain`. The Space never sees a raw Rune error.

```
// A-PROFILE-01 catches R-DATA-01's failure
{
  sigil: "CODEX_ERROR",
  tier: "1a",
  source: "A-PROFILE-01.v1",
  code: "PROFILE_BUILD_FAILED",
  message: "Could not assemble profile — data layer unreachable",
  timestamp: "...",
  chain: [
    { tier: 0, source: "R-DATA-01.v1", code: "DB_TIMEOUT", ... }
  ]
}
```

**At Tier 1c (Rituals):** Same pattern — catches Ward Sigils, wraps with Ritual context, pushes Ward Sigils into chain.

**At Tier 2 (Spaces):** The Space receives a top-level Sigil from whatever it called. The `chain` gives the full trace from Space down to the root cause. The Space makes two decisions: what to show the user (a friendly message derived from the top-level Sigil) and what to log (the full Sigil chain for debugging).

### Multi-Rune failures in a single Arcane

When an Arcane calls three Runes and two of them fail, the Arcane's Sigil contains both failures in the chain:

```
{
  sigil: "CODEX_ERROR",
  tier: "1a",
  source: "A-PROFILE-01.v1",
  code: "PARTIAL_FAILURE",
  message: "Profile partially assembled — media and data both failed",
  chain: [
    { tier: 0, source: "R-DATA-01.v1", code: "DB_TIMEOUT", ... },
    { tier: 0, source: "R-MEDIA-01.v1", code: "STORAGE_UNAVAILABLE", ... }
  ]
}
```

### Error code registry

Like domains, error codes should be registered to prevent drift. Each component defines its possible error codes in its frontmatter:

```markdown
---
id: R-DATA-01
...
error_codes:
  - RECORD_NOT_FOUND
  - DB_TIMEOUT
  - INVALID_QUERY
---
```

This gives the linter another check: if a component throws a code that isn't in its registered list, flag it.

---

## Quick reference — the naming at a glance

| Old name | New name | What it is | Tier |
|---|---|---|---|
| Function Card | **Rune** | Atomic, indivisible function | 0 |
| Function Map | **Arcane** | Composed bundle of Runes | 1a |
| *(new)* | **Ward** | Cross-cutting service | 1b |
| *(new)* | **Ritual** | Ward orchestration spell | 1c |
| Space | **Space** | Framework destination | 2 |
| *(the whole system)* | **Codex** | The complete architecture | — |
| *(infrastructure)* | **Sigil** | Standard error object | outside Codex |
