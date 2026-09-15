---
name: flowkon-icp-outreach-campaign
description: Build a complete Flowkon LinkedIn outreach campaign from an ICP down to individually researched, personalized messages per lead. Use this skill whenever the user asks to create, set up, or build a LinkedIn/Flowkon outreach campaign, wants ICP-based lead filtering or scoring, asks for a "campaign with warmup and follow-ups," wants "personalized messages for each lead," or wants deep research done on leads before outreach. Trigger this even if the user only asks for one piece (e.g. "just score these leads by ICP" or "write personalized connect messages") — those are sub-steps of this same skill. Always prefer this skill over ad hoc Flowkon tool calls for anything beyond a single simple lookup.
---

# Flowkon ICP Outreach Campaign Builder

Turns "I want to sell X to Y audience" into a fully staged Flowkon campaign: a
Sales Navigator search, an ICP-scored lead list, a warmup+follow-up automation
flow, and individually researched, hand-written personalized messages per lead
— all saved and verified in Flowkon, in draft status, ready for human review
before launch.

This is a **long, multi-step workflow**. Move through the phases below in
order. Confirm scope with the user at the marked checkpoints rather than
guessing — each checkpoint is cheap to ask and expensive to redo at scale
across 20+ leads.

## Phase 0 — Clarify the ICP

Before touching any tools, nail down:
- **Product being sold** and its core value prop (what pain does it solve, for whom)
- **Target segments** (e.g. "SaaS founders" and "agencies" are different buyers
  with different pitches — don't merge them into one generic campaign)
- **Geography** — ask, don't assume; scope changes lead volume a lot
- **Which segments to prioritize** if there are multiple

Use `ask_user_input_v0` for these if the user's initial ask is broad (e.g. "sell
my product to B2B companies"). Keep to 1-3 tight questions, not a full brief.

Write out the ICP as a short table (title/seniority, industry, company size,
signals) before searching — this becomes your filter logic in Phase 1 and your
scoring rubric in Phase 2.

## Phase 1 — Build the lead search

Use `Flowkon:flowkon_create_lead_search` with `search_type: "sales_navigator"`.

**Critical constraint: do not fabricate Sales Navigator facet IDs.** If the
user supplies a real Sales Navigator URL, reuse its verified facet IDs (region,
headcount, etc.) when building follow-on searches — these are real LinkedIn
taxonomy codes you can't safely guess. For new segments without a source URL,
prefer free-text `keywords` combined only with facets you've already verified
(e.g. a REGION id from an existing URL) rather than inventing title/industry/
headcount facet IDs. Tell the user plainly when you've done this — it's an
honest tradeoff, not a hidden shortcut.

Run one search per segment (e.g. one for "SaaS founders," one for "agencies")
rather than conflating them — this keeps downstream ICP scoring and messaging
cleanly separable.

Check `Flowkon:flowkon_get_search_progress` before moving on; searches are
async.

## Phase 2 — Create campaigns and score leads into them

1. Create one campaign per segment via `Flowkon:flowkon_create_campaign`
   (`search_entry_id` links it to the search, but does **not** auto-enroll
   leads — that's a separate step).
2. Pull the lead pool via `Flowkon:flowkon_get_leads_with_placeholders`
   (`limit: 100`). Note: this tool does not filter by `search_entry_id` — you
   get the workspace's recent lead pool, so score/dedupe by name and headline
   content, not by trusting a search-scoped result set.
3. **Score each lead manually against the ICP table from Phase 0** using their
   `headline` and `company_name`. Classify as high-fit (title matches +
   industry/company-type matches) or excluded (wrong seniority, unrelated
   industry, vague/unverifiable). State the scoring rule explicitly to the
   user before running it, e.g.: "High fit = Founder/CEO/Co-Founder title AND
   company is a SaaS product or marketing/creative agency."
4. Enroll the high-fit leads with `Flowkon:flowkon_add_campaign_prospects`,
   passing the matching `search_id`. Watch the `conflicts` array in the
   response — leads already enrolled in another campaign are silently skipped;
   report this to the user rather than assuming full enrollment.

## Phase 3 — Build the automation flow (warmup + follow-ups)

Call `Flowkon:flowkon_get_flow_node_types` first — always, every time — to get
the current authoritative list of node types/operations and structural rules.
Do not rely on memory of past flows; the API is the source of truth.

**Standard expert-pattern flow** (adjust delays per the user's stated
preference, but this is a sane default):

```
START → VIEW_PROFILE → LIKE → CONNECT → IF_CONNECTED (condition)
  yes → FOLLOW_UP 1 → FOLLOW_UP 2 → FOLLOW_UP 3 → END
  no  → WITHDRAW_CONNECT → END
```

Rationale for each piece:
- **View + Like before Connect**: warms the profile up, makes the connect
  request feel less cold/automated
- **Delay of ~1-2 days between each pre-connect touch**: back-to-back same-day
  actions look scripted
- **3-4 day wait before checking IF_CONNECTED**: gives a real human time to
  notice and accept
- **Small delay (not 0) before Follow-up 1 after acceptance**: messaging the
  instant someone accepts reads as bot behavior
- **Escalating follow-up cadence** (~3, then ~5 days apart): curiosity
  question → proof/value pitch → soft break-up with low-friction CTA
- **Withdraw after ~14 days if not accepted**: frees up the request slot per
  LinkedIn norms rather than leaving it pending indefinitely

Message nodes should reference **placeholder merge tags**
(`{{connect_message}}`, `{{follow_up_message_1}}`, `{{follow_up_message_2}}`)
rather than static text, so each lead's personalized copy (Phase 4) merges in
per-lead. See `references/placeholder-fields.md` for how to find/reuse the
right placeholder definition IDs — never create new custom fields per lead;
reuse the same 2-3 field IDs across every lead in the campaign.

Save with `Flowkon:flowkon_build_campaign_flow`, then confirm with
`Flowkon:flowkon_get_campaign_flow_view` and show the result to the user
(don't just describe it from memory — the tool's summary view may only show
one branch of a conditional, so cross-check against what you actually sent if
anything looks incomplete).

## Phase 4 — Research and write per-lead personalized messages

This is the highest-effort, highest-value phase. Do not template-merge names
into generic copy — that's what a first draft looks like, not a finished one.
If the user says messages "aren't impressive" or asks for "more research,"
this phase is what's missing.

For each lead:
1. **Research before writing.** Use `web_search` on the person's name + company
   to find something real and specific: notable clients, an award, a founding
   story, a stat, a recent campaign. Not every lead will have searchable
   results — that's fine, see step 2.
2. **Write 3 unique messages per lead** by hand:
   - Connect note: opens with the specific researched detail (or, absent
     research, a sharp specific read of their LinkedIn headline — never a
     generic "would love to connect")
   - Follow-up 1: a real, pointed question tied to their specific business
     model, not a templated "how's business going"
   - Follow-up 2: the value/proof pitch, phrased to connect back to what
     *they specifically* do — not a copy-pasted product blurb
3. **Save immediately per lead** with `Flowkon:flowkon_set_lead_placeholder_values`,
   passing the `lead_id` and the same 2-3 `placeholder_definition_id`s
   established in Phase 3.
4. **Batch by research depth, and say so.** Some leads will get deep
   research-backed messages, others will only get sharp copy off the headline
   alone (search quality varies). Report this split honestly to the user
   rather than implying uniform depth — see `references/quality-tiers.md`.

Copywriting bar (apply to every message):
- No "would love to connect" as the *entire* opener — earn the ask with a
  specific detail first
- No question a template could answer for them ("how's business?") — ask
  something only true of their specific company
- Avoid repeating the exact same sentence structure across leads even when
  the underlying pitch is the same; read back a batch and vary the phrasing

## Phase 5 — Verify, don't assume

After any batch of saves, **read the data back from Flowkon** — do not tell
the user "saved" based on the write call's success response alone if they ask
you to confirm. Use `Flowkon:flowkon_get_leads_with_placeholders` filtered by
lead name (or unfiltered with a high limit) to pull the actual current values
and show them directly. This matters because:
- Write calls can silently no-op or be overwritten by a later call
- The user may be viewing a different UI surface than what you wrote to
  (e.g. unrelated "Personalization Message 1/2/3" fields the app itself may
  auto-provision, which are not the same as your `connect_message` /
  `follow_up_message_1/2` fields — don't confuse the two, and tell the user
  plainly if you spot fields you didn't create)

When asked "which tool did you use" or "how did you send this," answer with
the literal tool name and parameters, not a vague description — this keeps
trust intact and makes the workflow auditable.

## Minor updates to a single lead

For a small edit request ("update just this one," "make a minor tweak"):
call `Flowkon:flowkon_set_lead_placeholder_values` with only the changed
field(s) for that one `lead_id` — don't resend unrelated fields, and don't
touch other leads. Show a clear before/after so the user can see exactly what
changed.

## Common pitfalls (from real runs of this workflow)

- **Don't fabricate Sales Navigator facet IDs.** Free-text keywords + verified
  facets only.
- **`flowkon_get_leads_with_placeholders` doesn't scope by search_entry_id** —
  score/dedupe leads yourself rather than trusting the result set is exactly
  your search's output.
- **`add_campaign_prospects` conflicts are silent** unless you check the
  `conflicts` array — always report skipped leads to the user.
- **Flow's `IF_CONNECTED` branches can't reconverge** — give each branch its
  own END node (see `flowkon_get_flow_node_types` structural rules).
- **First-draft messages are templated by default** — expect the user to ask
  for a research pass; budget for it rather than treating it as a surprise
  follow-up ask.
