# Placeholder fields for per-lead personalization

Flowkon workspaces come with a mix of system-default placeholders
(firstName, companyName, jobTitle, etc.) and custom ones a team may have
created previously (Custom Message 1/2, Pain Point, message ending, etc.).

## How to find the right fields

1. Call `Flowkon:flowkon_get_placeholder_definitions` (use `custom_only: true`
   to skip the system defaults) to see what already exists in the workspace.
2. **Reuse existing fields rather than creating new ones per lead.** A
   workspace typically already has 2-3 generic slots meant for exactly this
   purpose — e.g. `connect_message`, `follow_up_message_1`,
   `follow_up_message_2`. Look for slugs that map to your flow's message
   nodes before assuming you need something new.
3. Note the `id` (placeholder_definition_id) for each field you'll use — you
   need these exact IDs for every `flowkon_set_lead_placeholder_values` call.
   They're workspace-level and identical across all leads.

## How the merge works

In the campaign flow (see main SKILL.md Phase 3), message nodes reference
these fields via `{{slug}}` syntax, e.g.:

```json
{"data": {"body": "{{connect_message}}"}, "id": "connect", "operation": "CONNECT", "type": "ACTION"}
```

At send time, Flowkon substitutes each lead's saved value for that field. If
a lead has no value set for a referenced field, it typically renders blank or
falls back to the field's configured `fallback_value` — so every enrolled
lead needs values set for every field referenced in the flow, or you'll send
empty/generic messages to whoever you missed.

## Example set call

```
Flowkon:flowkon_set_lead_placeholder_values(
  lead_id: "<lead-uuid>",
  values: [
    {"placeholder_definition_id": "<connect_message-id>", "value": "<written connect note>"},
    {"placeholder_definition_id": "<follow_up_message_1-id>", "value": "<written follow-up 1>"},
    {"placeholder_definition_id": "<follow_up_message_2-id>", "value": "<written follow-up 2>"}
  ]
)
```

Do this once per lead — one call, all fields for that lead together — rather
than one call per field, to keep the number of tool calls proportional to
lead count rather than lead count × field count.

## Unrelated fields you may encounter

Some workspaces have auto-provisioned fields (e.g. "Personalization Message
1/2/3") that appear in `flowkon_get_placeholder_definitions` but aren't
referenced by any flow you built. Don't assume these are yours to fill or
that they're broken — check whether any flow node actually references them
before touching them, and flag their existence to the user rather than
guessing at their purpose.
