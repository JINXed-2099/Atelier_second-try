---
id: R-DATA-01
version: 1
tier: 0
type: rune
domain: DATA
status: draft
created: 2026-09-17
last_modified: 2026-09-17
produces: { id: string, username: string, display_name: string, avatar_url: string, joined: string }
consumes: { user_id: string }
error_codes: [RECORD_NOT_FOUND, DB_TIMEOUT, INVALID_QUERY]
imports: []
deprecated_versions: []
tags: [tier/0, type/rune, domain/DATA]
---

# R-DATA-01 — Fetch User Record

## Purpose
Retrieves a single user record from the database by user ID.

## Qualification gates
- [x] **Gate 1 — Space neutral:** Zero references to any Space
- [x] **Gate 2 — Single responsibility:** Fetches one user record
- [x] **Gate 3 — Zero dependencies:** No Codex imports
- [x] **Gate 4 — Generic interface:** String in, object out
- [x] **Gate 5 — Independent testability:** Only needs a DB mock

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
  id: string,
  username: string,
  display_name: string,
  avatar_url: string,
  joined: string       // ISO 8601 date
}
```

## Error codes
| Code | When it fires | Sigil message |
|---|---|---|
| RECORD_NOT_FOUND | No record exists for the given user_id | No user record found for id {user_id} |
| DB_TIMEOUT | Database query exceeds 5000ms | Database query timed out after 5000ms |
| INVALID_QUERY | user_id is null, empty, or wrong type | Invalid user_id: expected non-empty string |

## Behavior
1. Validate that `user_id` is a non-empty string. If not → throw INVALID_QUERY Sigil.
2. Query the users table by primary key.
3. If no record found → throw RECORD_NOT_FOUND Sigil.
4. If query exceeds 5000ms → throw DB_TIMEOUT Sigil.
5. Return the user object matching the `produces` shape.

## External dependencies
- Database client (e.g., Prisma, Knex, or native driver)

## Notes
This is the most fundamental data Rune. Nearly every Arcane that deals with users will import this.
