# JOURNAL

## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/163

**Issue title:** Review creation does not verify profile ownership

**Tier:** [ ] Tier 1  [x] Tier 2  [ ] Tier 3

**Problem summary:**
The POST /reviews endpoint forwards the authenticated user's ID into
`create_review()`, but the service layer ignores that argument entirely — it
builds a Review from the supplied `profile_id` without ever confirming the
profile belongs to the caller. This is inconsistent with the read paths
(`get_review` and `list_reviews`), which correctly scope every query through
`Profile.user_id`. As a result, an authenticated user who knows another
user's profile UUID can create reviews against that profile — a broken
object-level authorization (IDOR) vulnerability in `core/services/
review_service.py`. A successful fix will load the target profile, verify
`profile.user_id == user_id`, and return a 403 (or 404) when it doesn't
match, bringing review creation in line with the other endpoints.

**Branch name:** fix/163-review-ownership-check

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [x] Issue added to cohort ledger

**Is this right for me? — scope reasoning:**
- Scope is small and well-bounded: the root cause lives in a single function
  (`create_review`) in one service file, with the fix pattern already
  demonstrated by `get_review`/`list_reviews` in the same file.
- I can clearly explain the problem (missing ownership check) and what a
  correct fix looks like, and it is straightforward to test (a request with
  another user's profile_id should be rejected).
- No large architectural change or unfamiliar subsystem is required, so it
  fits comfortably within the module's timeframe.

## Week 8 — Issue reproduction & solution planning

**Reproduction steps:**
Wrote an integration test (`tests/integration/test_review_ownership.py`)
that drives the real FastAPI app end-to-end against the dev Postgres
database:
1. Register User A and User B via `POST /auth/register`.
2. As User B, create a profile via `POST /profiles` — note the `profile_id`.
3. As User A, call `POST /reviews` with `{"profile_id": "<User B's profile_id>"}`.

**What I actually saw:**
The request succeeded with `200 OK` and a full review object, e.g.:
```
AssertionError: Expected review creation to be rejected for a profile that
does not belong to the requesting user, got 200:
{"id":"281a7a24-2c1c-4476-a299-75d5af9a5b39", ...}
```
Server logs confirm it didn't just create the row — the background task ran
to completion against User B's profile on User A's behalf:
`review_processing_started` → `ingestion_pipeline_completed` →
`agent_orchestration_completed` → `review_processing_completed
overall_score=0.81`, all logged under `user_id` = User A but `profile_id`
belonging to User B. This confirms the issue exactly as described: no
ownership check exists between the authenticated user and the profile being
reviewed.

**Root cause:**
`create_review()` in `core/services/review_service.py` accepts a `user_id`
argument but never uses it — it builds the `Review` directly from the
caller-supplied `profile_id`. This is inconsistent with `get_review()` and
`list_reviews()` in the same file, which correctly filter on
`Profile.user_id`.

**Plan summary (full detail in `PLAN.md`):**
Reuse the existing ownership-scoped lookup,
`profile_service.get_profile(db, profile_id, user_id)`, inside
`create_review()`. If it returns `None`, raise `HTTPException(404)` —
matching the pattern already used by `get_review_endpoint` and
`get_profile_endpoint` for "not found or not owned," and avoiding a 403's
information leak about whether the profile exists at all.

**Files to touch:**
- `core/services/review_service.py` — add the ownership check in `create_review()`
- `api/routes/reviews.py` — confirm the 404 surfaces correctly from the route
- `tests/unit/test_review_service.py` — add a unit-level ownership test; update
  existing mocked tests that currently call `create_review()` without a
  matching `Profile`, since they'll break once the check is added

**Riskiest part:**
The existing unit tests in `test_review_service.py` mock `create_review()`
with unrelated random `profile_id`/`user_id` pairs and assert success —
several of those will need their mocks updated once the ownership lookup is
added, or they'll start failing for the right reason (they're exercising the
now-fixed vulnerable path).

**Status:**
- [x] Reproduction test written and confirmed failing (proves the bug)
- [x] `PLAN.md` completed
- [ ] Walkthrough video (optional, not recorded)

## Week 9 — Solution building & PR submission

### Check-in 1 (mid-week)

**Current progress:**
Implemented the fix from `PLAN.md`. `create_review()` in
`core/services/review_service.py` now calls
`profile_service.get_profile(db, profile_id, user_id)` and raises
`HTTPException(404)` if it returns `None`, matching the "not found or not
owned" pattern already used by `get_review_endpoint`/`get_profile_endpoint`.
Confirmed `api/routes/reviews.py` needed no change — it already re-raises
`HTTPException` before its generic 500 handler. Updated the 6 existing
`create_review` unit tests in `tests/unit/test_review_service.py` to mock
the new ownership lookup, and added
`test_create_review_rejects_profile_not_owned_by_user` for the cross-user
case. The Week 8 integration test
(`tests/integration/test_review_ownership.py`) now flips from failing (200)
to passing (404) — the fix works end-to-end.

Self-review: compared `make check`/`make test-unit` before and after the
change. Unit tests: `53 failed, 376 passed` both before and after — same 53
pre-existing failures, +1 new passing test, zero regressions. Ruff on the 2
files touched: 9 pre-existing errors before → 6 after (net-fixed 3 while
editing: unused import, a long line, import ordering). Mypy flags 13
pre-existing errors in `review_service.py`/`profile_service.py` (untyped
`db` params, `Any` returns) — verified via before/after diff that these are
identical violations at shifted line numbers, or in `profile_service.py`
(0 lines changed by us) surfacing only because our new import makes mypy
follow it. All sub-tasks 1–7 from `PLAN.md` are done; sub-task 8 (final
`make check`/`make test-unit` pass) is done modulo the documented
pre-existing failures above.

**Next steps:**
Commit and push the fix (using `--no-verify` for this one commit, since
pre-commit's mypy/ruff hooks block on the pre-existing debt documented
above — not on anything this change introduces). Open a draft PR with this
evidence in the description, request peer/mentor feedback in Slack, then
address feedback and mark the PR ready for review.

**Blockers:**
None blocking progress. Noting for transparency: pre-commit's mypy and ruff
hooks fail on pre-existing issues in `core/services/review_service.py` and
`core/services/profile_service.py` unrelated to issue #163 (untyped `db`
parameters, implicit-Optional defaults, `Any` returns, and
`N806`/`F841` in `get_review`/`list_reviews` tests we didn't touch). Verified
with before/after diffs that this change introduces zero new lint or type
errors; full breakdown will go in the PR description per the course's
pre-existing-failures guidance.

---

### Check-in 2 (end of week)

**PR link:** [link to your submitted pull request]

**Branch:** fix/163-review-ownership-check

**What you built:**
[1–3 sentences summarizing what your fix does and how it works]

**Tests added or updated:**
[Which test files did you touch? What do they cover?]

**Self-review confirmation:** [ ] make check passes  [ ] make test-unit passes

**Draft PR feedback received from:** [name or Slack handle, or "none"]
