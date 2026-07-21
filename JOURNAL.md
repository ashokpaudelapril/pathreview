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

**Setup confirmation:** [ ] App runs locally at localhost:5173

**Cohort ledger:** [ ] Issue added to cohort ledger

**Is this right for me? — scope reasoning:**
- Scope is small and well-bounded: the root cause lives in a single function
  (`create_review`) in one service file, with the fix pattern already
  demonstrated by `get_review`/`list_reviews` in the same file.
- I can clearly explain the problem (missing ownership check) and what a
  correct fix looks like, and it is straightforward to test (a request with
  another user's profile_id should be rejected).
- No large architectural change or unfamiliar subsystem is required, so it
  fits comfortably within the module's timeframe.
