# Security Audit Report: OwockiBot Platform

## 🔴 High Severity

### H-1 · Lack of Server-Side Validation on `reward_usdc`
**Description:** The API lacks proper validation for reward amounts, allowing inconsistent and invalid data (negative values, zeros, and extreme integers) to persist in the database.

**Evidence (Live in DB):**
- **Bounty #173:** `reward_usdc` = -100 ("negative test")
- **Bounty #174:** `reward_usdc` = 0 ("zero test")
- **Bounty #180:** `reward_usdc` = 999,999,999 ("huge bounty")

**Impact:** Attackers can manipulate reward values, poisoning downstream analytics and causing potential overflows in leaderboards or financial dashboards.

**Recommended Fix:** Implement a schema validator (Zod/Pydantic) on the `POST` handler to enforce:
`reward_usdc: int, min=1, max=10000`.

---

### H-2 · Bounty Creator Self-Claim Vulnerability
**Description:** The system allows bounty creators to claim and complete their own tasks, bypassing the intended collaborative nature of the platform.

**Evidence:**
- **Bounty #288:** `creator_address` == `claimer_address` (0xdc05f6ca...)
- **Bounty #193:** `creator_address` == `claimer_address` (0xC7Da3DbC...)

**Impact:** Enables reputation gaming and leaderboard farming. If escrow is implemented, this allows creators to "self-drain" funds.

**Recommended Fix:** Update the `PATCH {action:'claim'}` handler to reject requests if:
`claimer.toLowerCase() === bounty.creator_address.toLowerCase()`.

---

## 🟡 Medium Severity

### M-1 · Null/Empty `creator_address` Persistence
**Description:** The database accepts an empty string for the `creator_address` field upon bounty creation.

**Evidence:** Bounties #221, #222, and #223 all have `creator_address: ""`.

**Impact:** Breaks the authorization model. Anyone can claim or cancel these bounties by sending an empty string as the authority.

**Recommended Fix:** Add Regex validation (`^0x[0-9a-fA-F]{40}$`) on creation and apply a `NOT NULL` constraint at the DB level.

---

### M-2 · Systematic Timestamp Format Divergence
**Description:** `created_at` and `updated_at` use different ISO-8601 formats due to inconsistent generation layers (Postgres/Python vs. Node.js).

**Evidence:** - `created_at`: `2026-04-11T14:04:44.764727+00:00` (Microseconds)
- `updated_at`: `2026-04-11T14:05:40.622Z` (Milliseconds)

**Impact:** Strict parsers and sorting algorithms may fail or produce inconsistent results.

**Recommended Fix:** Standardize timestamp generation in a single layer (e.g., DB-level `now()` on both insert and update).

---

### M-3 · State-Machine Inconsistency (`claimed` + `submission_url`)
**Description:** Bounties are found in a `claimed` state while already possessing a `submission_url`.

**Evidence:** Bounty #244: `status: "claimed"` with a valid GitHub submission link.

**Impact:** Indicates a broken business logic flow where status transitions do not follow the data lifecycle.

**Recommended Fix:** Ensure status updates and submission writes are atomic. Add a DB constraint: 
`CHECK (status <> 'claimed' OR submission_url IS NULL)`.

---

## 🔵 Low Severity & Hygiene

### L-1 · Missing Sanitization on Title/Description
**Description:** The server accepts raw HTML tags and lacks length limits for bounty metadata.

**Evidence:** Bounty #172 title: `<script>alert(1)</script>`.

**Impact:** Potential Passive XSS for third-party consumers (bots, RSS feeds, indexers).

**Recommended Fix:** Sanitize HTML tags server-side and enforce limits (Title ≤ 120, Description ≤ 3000).

---

### L-2 · Production Database Pollution
**Description:** Test records from developers are visible in the public production API.

**Evidence:** Bounties #173, #174, #180, and #181 (e.g., "negative test", "string reward").

**Impact:** Reduces data integrity and professional credibility.

**Recommended Fix:** Hard-delete test artifacts and establish a strict staging vs. production environment separation.
