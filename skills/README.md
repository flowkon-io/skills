# Skills

Flowkon's Claude Code skills, organized by category. Each category is a
subfolder with its own README listing the skills in it; each skill is a
folder containing a `SKILL.md` (frontmatter `name` + `description`, plus
step-by-step instructions) and optional `references/`, `scripts/`, or
`assets/` subfolders.

```
skills/
  <category>/
    README.md              # lists the skills in this category
    <skill-name>/
      SKILL.md              # required
      references/           # optional: longer docs, loaded only when relevant
      scripts/               # optional: helper scripts the skill can run
      assets/                 # optional: templates/files the skill uses or outputs
```

This repo is a dedicated skills-distribution repo:
skills live at
top-level `skills/`, and `.claude-plugin/plugin.json` +
`marketplace.json` at the repo root turn the whole repo into an installable
Claude Code plugin marketplace. See the root [README.md](../README.md) for
install instructions.

## Categories

| Category | Description |
| --- | --- |
| [outreach](outreach/README.md) | Building and running Flowkon LinkedIn outreach campaigns. |
| [engineering](engineering/README.md) | Working in Flowkon's own codebase (social-bot backend, campaign flow engine). |

## Using these skills locally, in this repo

Because skills live under top-level `skills/` (not `.claude/skills/`),
Claude Code won't auto-discover them just from being opened in this repo —
that's intentional, matching the reference repo, since this repo's job is to
be installed elsewhere, not to run itself. To use these skills while working
directly in this repo (e.g. to test a skill before publishing it), install
the plugin from the local path:

```
/plugin marketplace add .
/plugin install flowkon-skills@flowkon
```

## Adding a new skill

1. Pick (or create) a category folder under `skills/`.
2. Create `<category>/<skill-name>/SKILL.md` with frontmatter:
   ```yaml
   ---
   name: skill-name
   description: What it does and when Claude should trigger it — be specific about trigger phrases.
   ---
   ```
3. Add the skill's step-by-step instructions below the frontmatter.
4. Add a row for it to the category's `README.md`.
5. Add the skill's path to the `skills` array in
   [`.claude-plugin/plugin.json`](../.claude-plugin/plugin.json).
