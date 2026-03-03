# Sveeri — Ballot Scanner Database

## Database Versions

### ver1
Tables: `users`, `positions`, `candidates`, `sessions`, `ballots`, `votes`

Extra fields: `created_by` on positions & candidates, `scanned_by` and `scanned_at` on ballots.

### ver2
Same tables. Removed `created_by`, `scanned_by`, and `scanned_at`.

### ver3 ✅ Current
Same tables. Added to `ballots`: `scan_attempts`, `is_successful`.  
Added to `votes`: `position_id`, `is_abstain` and made `candidate_id` nullable.

--------------------------------------------------------------------------------

Tables (ver3)

| Table | Fields |
|---|---|
| `users` | `id`, `username`, `email`, `password_hash`, `forgetpassword_token`, `fp_token_expires`, `created_at` |
| `positions` | `id`, `position_name` |
| `candidates` | `id`, `candidate_name`, `position_id` |
| `sessions` | `id`, `user_id`, `started_at`, `ended_at`, `is_exported` |
| `ballots` | `id`, `session_id`, `scan_attempts`, `is_successful` |
| `votes` | `id`, `ballot_id`, `candidate_id`, `position_id`, `is_abstain` |

---

What Each Field Does to the Code

### users
- `password_hash` — the code never stores the raw password. It hashes it with bcrypt before saving and checks it with bcrypt on login.
- `forgetpassword_token` — the code generates a random token and saves it here when forgot password is triggered. It checks this token to allow a password reset.
- `fp_token_expires` — the code checks this before allowing a reset. If the current time is past this value, the token is rejected.

### sessions
- `ended_at` — **nullable**. The code checks if this is `NULL` to know if a session is still active. If `NULL`, no new session can start.
- `is_exported` — the code sets this to `1` and fills `ended_at` at the same time when the user clicks Export.
- A session **cannot start** if the `candidates` table is empty. The code checks this first.

### ballots
- `scan_attempts` — **TINYINT, length 3** (not 1). The code increments this every time a scan fails. Must go up to 3 so do not set length to 1 or MySQL treats it as boolean.
- `is_successful` — the code sets this to `0` when `scan_attempts` reaches 3. The code uses this to count unscanned ballots on the total votes screen.

### votes
- `candidate_id` — **nullable**. The code sets this to `NULL` when recording an abstain. If your code expects this to always have a value it will break.
- `position_id` — required even for abstain rows. The code uses this to count abstains per position since `candidate_id` is NULL and cannot be used.
- `is_abstain` — the code sets this to `1` for every position that was not voted on in a scanned ballot. The code uses this to separate real votes from abstains in the results.

### candidates
- `position_id` — the code uses this to group candidates by position when loading the ballot form and when checking which positions were skipped (abstain detection).

### Foreign key cascades
- Deleting a `session` also deletes its `ballots` and their `votes`.
- Deleting a `position` also deletes its `candidates` and any `votes` linked to them.
- The code should never delete sessions or positions mid-scan or results will be lost.
