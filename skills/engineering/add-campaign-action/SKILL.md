---
name: add-campaign-action
description: Add, modify, or audit campaign action / node types (VIEW_PROFILE, LIKE, ENDORSE, CONNECT, EMAIL, COMPANY_FOLLOW_INVITE, IF_* conditions, etc.) in social-bot's campaign flow engine. Covers the action-handler/ACTION_MAP pattern, the quota/limit registries, retry special-casing, and cross-registry consistency checks. Use when asked to add a new campaign action, change how an existing action behaves (limits, quota, retry, payload), or find/fix inconsistencies across campaign action registries in social-bot.
---

# Campaign actions (social-bot)

Every campaign action is dispatched through one code path (`run_action` in
`backend/app/service/CampaignActions.py`), but is *wired up* through several
independently-maintained registries with **no cross-validation between them**.
Whether you're adding a new action, changing an existing one, or investigating
why an action behaves oddly, start by knowing which registries govern it — see
`references/action-registry.md` for a full current snapshot, including two live
gaps (`COMMENT`, `IF_INMAIL_OPENED` — enum entries with no handler) and a
divergence between the two "exclude from limit" sets. Re-derive it with the
grep recipe at the bottom of this file rather than trusting the snapshot blindly
— it drifts.

Use `ENDORSE` as the reference implementation for a simple, non-conditional
action with no retry special-casing:
- Handler: `endorse_profile_activity` — `backend/app/utils/campaign_actions.py` ~L548
- Enum: `campaign_enums.py` ~L25
- Quota-window opt-out: in `ACTIONS_WITHOUT_QUOTA_LIMIT`
- Job-builder branch: `CampaignService.campaign_job_info` ~L4677 (function moved
  since first written — grep `def campaign_job_info` if the line's stale)

## Task: add a new action

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
     This is the only place `ACTION_MAP` is looked up (`run_action` ~L161) — a
     missing entry raises `ValueError: Unsupported action` **at execution time**,
     not at startup or flow-save time. `COMMENT` is stuck in exactly this state
     today (see `references/action-registry.md`) — don't repeat it silently.

3. **Limit-map key** — `campaign_enums.py`: add an entry to `ACTION_TYPE_LIMIT_MAP`
   (usually identity: `"<ACTION>": "<ACTION>"`). This is what daily
   "Settings > Actions" limits key off
   (`CampaignService._remaining_allowance_for_action` ~L5084 — the live
   enforcement path; `CampaignService.is_action_limit_available` ~L3525 looks
   unreferenced, don't assume it's the one that matters).

4. **Quota-window opt-out (decide)** — `backend/app/service/CampaignActions.py`:
   `ACTIONS_WITHOUT_QUOTA_LIMIT` set (~L48). Add your action here only if it
   should bypass the per-account LinkedIn-style throttle window
   (`ActionQuotaStateService`) — e.g. low-risk / no distinct provider rate limit,
   like ENDORSE/LIKE. Leave it out if it should be throttled like CONNECT/VIEW_PROFILE/IN_MAIL.
   - Also update `ACTIONS_WITHOUT_LIMIT` in `campaign_enums.py` (~L65) to match —
     a **second, separately-maintained** set that today already disagrees with
     the first one for `IF_EMAIL_BOUNCED`/`IF_EMAIL_REPLIED` (see reference doc).
     Don't assume they're one list; update both deliberately.

5. **Frontend/flow-validation schema (only if `backend/app/service/campaign_flow_validation/`
   exists in your checkout)** — that module currently lives only on the unmerged
   branch `feat/campaign-validation`, not on `main`/`hf/campaign_mail`
   (`git ls-tree -r <ref> -- backend/app/service/campaign_flow_validation` to
   check your own checkout). If it's present: add an entry to `SUPPORTED_ACTIONS`
   in `campaign_flow_validation/constants.py` with `operation`, `label`,
   `topology` (usually `"linear"`), `required_fields`, `optional_fields` — this
   is what `GET /campaign-flow/rules` returns to the UI. Add a
   `STEP_PLACEMENT_RULES` entry (+ matching reason in `get_disabled_reason`) if
   the action has placement constraints, and a `NODE_CONFIG_RULES` entry if it
   needs required config fields validated. If the module isn't present yet,
   there's currently no schema check at all — the frontend/flow JSON can
   reference actions the backend can't run, with no error until a lead reaches
   that node.

6. **Legacy adapter** — `backend/app/utils/flow_adapter.py`:
   `LEGACY_ACTION_TO_DEFINITION` — add an entry mapping the action to itself (or
   to whatever old token should resolve to it), for backward-compat with
   previously-saved flow JSON.

7. **Payload enrichment (only if needed)** — if the action needs extra data
   resolved before it can run (ENDORSE needs `endorsement_id`, LIKE needs
   `post_id`), add a branch in `CampaignService.campaign_job_info`
   (`backend/app/service/CampaignService.py` — the per-lead job builder; there
   is no function literally named `execute_campaign_flow`). Follow the ENDORSE
   or LIKE branch as a template. If the action only needs generic fields already
   present (`account_id`, `target_profile_id`, `platform`), skip this.

8. **Retry/failure special-casing (only if needed)** — `run_action` (~L159)
   already raises a generic retryable exception for `statusCode in
   {408,500,502,503,504}` or infra-error keywords, handled uniformly for every
   action; your handler just needs accurate `statusCode`/`errorType`, not its
   own retry logic. Add action-specific handling only if you need something
   like EMAIL's post-send inbox record, CONNECT's note-limit fallback, or
   COMPANY_FOLLOW_INVITE's monthly-exhaustion handling — see the relevant
   segment of `run_user_profile_action` (`CampaignActions.py`, organized in 4
   commented segments).

## Task: modify how an existing action behaves

Pin down which registry actually governs the behavior you're changing before
editing — they're independent, so the fix is rarely "everywhere":

- **"It's hitting rate limits too often / not often enough"** → daily limit:
  `ACTION_TYPE_LIMIT_MAP` + `CampaignService._remaining_allowance_for_action`.
  Per-account throttle window instead: `ACTIONS_WITHOUT_QUOTA_LIMIT` membership
  + `ActionQuotaStateService`.
- **"It should/shouldn't retry on failure"** → the generic path is
  status-code/keyword driven in `run_action`; action-specific behavior lives in
  the 4 segments of `run_user_profile_action`. Don't add per-action retry logic
  inside the handler itself (`campaign_actions.py`) — that's not where retries
  are decided.
- **"Its response shape needs adjusting for a specific error"** — see
  `_normalize_company_follow_credit_exhausted` in `campaign_actions.py` as the
  existing pattern for reshaping a provider error into a specific
  `errorType`/`statusCode` for one action before `parse_action_response` runs.
- **"It needs different data before it runs"** → the `campaign_job_info` branch
  for that action (step 7 above).
- **"The frontend should offer/hide it differently"** → only actionable once
  `campaign_flow_validation/` exists in your checkout (step 5).

After any behavior change, re-check the two limit-exclusion sets and the
`ACTION_MAP`/enum pairing haven't drifted out of sync as a side effect —
that's the easiest way to silently break an unrelated action.

## Task: audit for registry drift

Quick recipe (run from `backend/`) to list every action name across the
registries that matter, so you can eyeball what's missing where:

```bash
grep -E "^\s+[A-Z_]+ = \"" app/utils/campaign_enums.py                     # enum members
grep -oE '"[A-Z_]+":' app/utils/campaign_actions.py                        # ACTION_MAP keys
grep -n "ACTIONS_WITHOUT_QUOTA_LIMIT" -A 20 app/service/CampaignActions.py | grep -oE '"[A-Z_]+"'
grep -n "ACTIONS_WITHOUT_LIMIT " -A 15 app/utils/campaign_enums.py | grep -oE '"[A-Z_]+"'
```

Cross-reference the outputs; any enum member missing from `ACTION_MAP` is
unrunnable, and any name in one exclusion set but not the other is worth
confirming is intentional. See `references/action-registry.md` for a
pre-built table and the two gaps/divergences already found this way.

## Tests

- `backend/tests/test_run_user_profile_action_session_boundaries.py` — template
  for `run_user_profile_action`/`run_action` tests (uses `action="VIEW_PROFILE"`,
  a quota-branch action). If your action is in `ACTIONS_WITHOUT_QUOTA_LIMIT`,
  also add a case for the `quota_branch=False` path — nothing currently covers it.
- `backend/tests/test_campaign_scheduler_tick.py` — template for
  `fetch_scheduler_tick_jobs`/job-building tests across different `action` values.
- Quota-window coverage (only if NOT in `ACTIONS_WITHOUT_QUOTA_LIMIT`):
  `backend/tests/test_action_quota_state_service.py`.
- Failure/retry coverage patterns: `test_campaign_connect_note_fallback.py`,
  `test_campaign_provider_throttle.py`.
- `backend/app/tests/test_campaign_flow.py` / `test_campaign_flow_validation.py`
  — only exist once `campaign_flow_validation/` is merged (step 5); not present
  on `main`/`hf/campaign_mail` today.

## After implementing

Run from `backend/`: `uv run pytest tests/ -v` and `cd .. && ./lint.sh --check-only`.
