---
name: add-campaign-action
description: Add a new campaign action / node type (e.g. COMMENT, TAG_LEAD) to social-bot's campaign flow engine — wires the action handler, ACTION_MAP, enums, quota/limit config, and the frontend-facing flow-rules schema. Use when asked to add, implement, or wire up a new campaign action, campaign node type, or flow step in social-bot.
---

# Add a campaign action

Adding a new campaign action touches several independently-maintained registries
across the backend. **Nothing cross-validates them against each other** — a
mismatch fails silently (frontend can offer an action the backend can't run, or
vice versa) rather than at startup. `CampaignActionsEnum.COMMENT` already exists
in `campaign_enums.py` with no `ACTION_MAP` handler and no `SUPPORTED_ACTIONS`
entry — that's a live example of the trap, not a hypothetical.

Use `ENDORSE` as the reference implementation for a simple, non-conditional
action with no retry special-casing. Its pieces, referenced throughout below:
- Handler: `endorse_profile_activity` — `backend/app/utils/campaign_actions.py` ~L548
- Enum: `campaign_enums.py` ~L25
- Quota-window opt-out: `ACTIONS_WITHOUT_QUOTA_LIMIT` membership
- Job-builder branch: `CampaignService.campaign_job_info` ~L4677
- Flow-schema entry: `campaign_flow_validation/constants.py` ~L110
- Placement rule: `campaign_flow_validation/rules.py` ~L48

## Required changes (every new action)

1. **Enum** — `backend/app/utils/campaign_enums.py`: add to `CampaignActionsEnum`
   (or `CampaignConditionalActions` if it's an `IF_*` condition node).

2. **Handler + ACTION_MAP** — `backend/app/utils/campaign_actions.py`:
   - Write `async def <name>_activity(payload: dict) -> dict`. Pattern: validate
     required payload fields up front (return
     `{"status": False, "statusCode": 400, "response": {"errorType": "ValidationError", "message": ...}}`
     if missing — see `like_recent_post_activity`), call
     `automation_service.facade.<method>(...)`, then wrap the result through
     `parse_action_response(response, action_name=..., account_id=..., target_id=...)`
     and return it.
   - Add `"<ACTION>": <name>_activity` to `ACTION_MAP` at the bottom of the file.
     This is the only place `ACTION_MAP` is looked up (`run_action` in
     `CampaignActions.py` ~L161) — a missing entry raises
     `ValueError: Unsupported action` at runtime, not at startup.

3. **Limit-map key** — `campaign_enums.py`: add an entry to `ACTION_TYPE_LIMIT_MAP`
   (usually identity: `"<ACTION>": "<ACTION>"`). This is what daily
   "Settings > Actions" limits key off
   (`CampaignService._remaining_allowance_for_action` ~L5084 — the live
   enforcement path; `CampaignService.is_action_limit_available` ~L3525 looks
   unreferenced, don't assume it's the one that matters).

4. **Quota-window opt-out (decide)** — `backend/app/service/CampaignActions.py`:
   `ACTIONS_WITHOUT_QUOTA_LIMIT` set (~L47). Add your action here only if it
   should bypass the per-account LinkedIn-style throttle window
   (`ActionQuotaStateService`) — e.g. low-risk / no distinct provider rate limit,
   like ENDORSE/LIKE. Leave it out if it should be throttled like CONNECT/VIEW_PROFILE.
   - Also check `ACTIONS_WITHOUT_LIMIT` in `campaign_enums.py` (~L65) — a
     **second, not-identical** exclusion set used elsewhere. Keep your decision
     consistent across both; they are not the same list and nothing enforces that.

5. **Frontend/flow-validation schema** —
   `backend/app/service/campaign_flow_validation/constants.py`: add an entry to
   `SUPPORTED_ACTIONS` with `operation`, `label`, `topology` (usually `"linear"`),
   `required_fields`, `optional_fields`. This is what `GET /campaign-flow/rules`
   returns to the UI — without it, the frontend won't know the action exists.
   - If the action has placement constraints (only valid after a "connected"
     branch, like ENDORSE/COMPANY_FOLLOW_INVITE), add a rule to
     `STEP_PLACEMENT_RULES` in `campaign_flow_validation/rules.py`, and a matching
     reason in `get_disabled_reason`.
   - If it needs required config fields validated (like EMAIL/IN_MAIL/FOLLOW_UP/
     COMPANY_FOLLOW_INVITE), add an entry to `NODE_CONFIG_RULES` in the same file.

6. **Legacy adapter** — `backend/app/utils/flow_adapter.py`:
   `LEGACY_ACTION_TO_DEFINITION` — add an entry mapping the action to itself (or
   to whatever old token should resolve to it), for backward-compat with
   previously-saved flow JSON.

## Optional: payload enrichment before execution

If the action needs extra data resolved before it can run (ENDORSE needs
`endorsement_id`, LIKE needs `post_id`), add a branch in
`CampaignService.campaign_job_info` (`backend/app/service/CampaignService.py`,
~L4287-5060 — the per-lead job builder; there is no function literally named
`execute_campaign_flow`). Follow the ENDORSE branch (~L4677) or LIKE branch
(~L4714). If the action only needs generic fields already present
(`account_id`, `target_profile_id`, `platform`), skip this.

If it needs retry/failure special-casing beyond the generic "raise on 5xx/
timeout → retry" path (EMAIL's post-send inbox record, CONNECT's note-limit
fallback, COMPANY_FOLLOW_INVITE's monthly-exhaustion handling), add it in the
relevant segment of `run_user_profile_action`
(`backend/app/service/CampaignActions.py`, ~L339-880, organized in 4 commented
segments). Most simple actions need nothing here. Note `run_action` (~L159)
already raises a generic retryable exception for `statusCode in
{408,500,502,503,504}` or infra-error keywords — your handler just needs to
return accurate `statusCode`/`errorType`, not implement retry itself.

## Tests

- `backend/tests/test_run_user_profile_action_session_boundaries.py` — template
  for `run_user_profile_action`/`run_action` tests (uses `action="VIEW_PROFILE"`,
  a quota-branch action). If your new action is in `ACTIONS_WITHOUT_QUOTA_LIMIT`,
  also add a case for the `quota_branch=False` path — nothing currently covers it.
- `backend/tests/test_campaign_scheduler_tick.py` — template for
  `fetch_scheduler_tick_jobs`/job-building tests across different `action` values.
- `backend/app/tests/test_campaign_flow.py` — assert the new operation appears
  in `GET /campaign-flow/rules`'s `supported_actions`.
- `backend/app/tests/test_campaign_flow_validation.py` — template for placement/
  graph-integrity tests for a specific operation (uses `{"operation": "LIKE"}`).
- Quota-window coverage (only if NOT in `ACTIONS_WITHOUT_QUOTA_LIMIT`):
  `backend/tests/test_action_quota_state_service.py`.
- Failure/retry coverage patterns: `test_campaign_connect_note_fallback.py`,
  `test_campaign_provider_throttle.py`.

## After implementing

Run from `backend/`: `uv run pytest tests/ -v` and `cd .. && ./lint.sh --check-only`.
