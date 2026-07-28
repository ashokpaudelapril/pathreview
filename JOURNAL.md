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
