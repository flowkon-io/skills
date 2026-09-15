# Campaign action registry — snapshot

Snapshot taken on branch `hf/campaign_mail` (also true of `main` — verified via
`git ls-tree`). **Re-verify before relying on this for anything but orientation**:
these sets drift independently and nothing enforces consistency between them.
Regenerate with the grep recipe at the bottom of `../SKILL.md`.

Columns:
- **Enum** — defined in `CampaignActionsEnum` (A) or `CampaignConditionalActions` (C) in `backend/app/utils/campaign_enums.py`
- **Handler** — has a live entry in `ACTION_MAP` (`backend/app/utils/campaign_actions.py`)
- **LimitMap** — in `ACTION_TYPE_LIMIT_MAP` (campaign_enums.py) — daily "Settings > Actions" limit counting
- **NoQuota** — in `ACTIONS_WITHOUT_QUOTA_LIMIT` (`CampaignActions.py` ~L48) — bypasses the per-account `ActionQuotaStateService` throttle window
- **NoLimit** — in `ACTIONS_WITHOUT_LIMIT` (campaign_enums.py ~L65) — a **second, separately-maintained** set, not the same list as NoQuota
- **Conditional** — in `CONDITIONAL_ACTIONS` (campaign_enums.py) — excluded from daily-limit counting because it's a routing check, not a limited action

| Action | Enum | Handler | LimitMap | NoQuota | NoLimit | Conditional |
|---|---|---|---|---|---|---|
| LIKE | A | ✅ | ✅ | ✅ | ✅ | |
| VIEW_PROFILE | A | ✅ | ✅ | | | |
| COMMENT | A | ❌ **no handler — enum exists, ACTION_MAP doesn't. Unrunnable today; `run_action` raises `ValueError: Unsupported action` if scheduled.** | ✅ | | | |
| ENDORSE | A | ✅ | ✅ | ✅ | ✅ | |
| FOLLOW | A | ✅ | ✅ | ✅ | ✅ | |
| CONNECT | A | ✅ | ✅ | | | |
| COMPANY_FOLLOW_INVITE | A | ✅ | ✅ | | | |
| WITHDRAW_CONNECT | A | ✅ | ✅ | ✅ | ✅ | |
| FOLLOW_UP | A | ✅ | ✅ | ✅ | ✅ | |
| IN_MAIL | A | ✅ | ✅ | | | |
| FIND_EMAIL | A | ✅ | n/a (data-prep step, not limited — correct) | ✅ | ✅ | |
| EMAIL | A | ✅ | ✅ | ✅ | ✅ | |
| IF_CONNECTED | C | ✅ | n/a | ✅ | ✅ | ✅ |
| IF_INMAIL_OPENED | C | ❌ **no handler — same gap as COMMENT.** | n/a | ❌ not listed | ❌ not listed | ✅ |
| IF_EMAIL_EXISTS | C | ✅ | n/a | ✅ | ✅ | ✅ |
| IF_EMAIL_OPENED | C | ✅ | n/a | ✅ | ✅ | ✅ |
| IF_EMAIL_CLICKED | C | ✅ | n/a | ✅ | ✅ | ✅ |
| IF_EMAIL_BOUNCED | C | ✅ | n/a | ✅ | ❌ **missing from NoLimit — present in NoQuota. Concrete proof the two sets diverge.** | ✅ |
| IF_EMAIL_REPLIED | C | ✅ | n/a | ✅ | ❌ **same gap as IF_EMAIL_BOUNCED.** | ✅ |
| IF_REPLIED | C | ✅ | n/a | ✅ | ✅ | ✅ |
| END | — | n/a (terminal, doesn't execute) | n/a | n/a | n/a | in `TERMINAL_ACTIONS`, not `CONDITIONAL_ACTIONS` |

## Known live gaps (as of this snapshot)

1. **`COMMENT`** — full enum + `ACTION_TYPE_LIMIT_MAP` entry, zero implementation.
   If a flow somehow gets a `COMMENT` node scheduled, `run_action` raises
   `ValueError: Unsupported action: COMMENT` at execution time, not at flow
   save/validate time (`backend/app/ai/prompts/comment.py` exists for AI-generated
   comment text but nothing consumes it via the campaign engine).
2. **`IF_INMAIL_OPENED`** — same shape of gap: enum exists, no `ACTION_MAP` handler,
   and it's also missing from both `ACTIONS_WITHOUT_QUOTA_LIMIT` and
   `ACTIONS_WITHOUT_LIMIT` (though moot while it has no handler at all).
3. **`ACTIONS_WITHOUT_QUOTA_LIMIT` vs `ACTIONS_WITHOUT_LIMIT` divergence** —
   `IF_EMAIL_BOUNCED` and `IF_EMAIL_REPLIED` are in the former but not the latter.
   Confirm whether that's intentional before assuming the two sets are
   interchangeable.

## The flow-validation / frontend-schema module does not exist on this branch

`backend/app/service/campaign_flow_validation/` (constants.py `SUPPORTED_ACTIONS`,
rules.py `STEP_PLACEMENT_RULES`/`NODE_CONFIG_RULES`, `CampaignFlowValidationService`)
and the router `backend/app/routers/campaign_flow.py` (`GET /campaign-flow/rules`,
`POST /campaign-flow/validate`) **exist only on the unmerged branch
`feat/campaign-validation`** (`origin/feat/campaign-validation`), not on `main` or
`hf/campaign_mail`. Verified via `git ls-tree -r FETCH_HEAD -- backend/app/service/campaign_flow_validation`
returning empty against `main`.

If that branch has merged by the time you're reading this, re-check — once merged,
adding/modifying an action also requires a `SUPPORTED_ACTIONS` entry (see
`SKILL.md` step 5) for the frontend to know the action exists at all. Until then,
there is **no schema validation of node types** — the frontend and any hand-built
flow JSON can reference actions the backend can't run (like `COMMENT` above) with
no error until a lead actually reaches that node.
