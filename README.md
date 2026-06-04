# ooda agent skills

Installable [Agent Skills](https://code.claude.com/docs/en/skills) that teach a
coding agent (Claude Code, Cursor, etc.) how to use [ooda](https://ooda.run).

## `ooda`

Publish and manage static websites on `ooda.run` from the CLI — publish a built
site to a shareable `{slug}-p.ooda.run` URL, list/unpublish sites, and control
per-site access (public / password / login). See
[`skills/ooda/SKILL.md`](./skills/ooda/SKILL.md).

## Install

Using the [`skills`](https://github.com/vercel-labs/skills) CLI:

```bash
# Global (all projects)
npx skills add toy-studio/ooda-skills -g

# Or per-project
npx skills add toy-studio/ooda-skills
```

Or install manually — copy the skill into your agent's skills directory:

```bash
mkdir -p ~/.claude/skills
curl -fsSL https://raw.githubusercontent.com/toy-studio/ooda-skills/main/skills/ooda/SKILL.md \
  -o ~/.claude/skills/ooda/SKILL.md
```

## Requirements

- Node.js 20+ (the skill drives `npx @oodarun/cli`).
- An ooda account in an organization. The agent authenticates via `ooda login`
  (email code), a saved login (`~/.ooda/auth.json`), or the
  `OODA_ACCESS_TOKEN` + `OODA_ORG_ID` environment variables.

## Releasing

Keep the skill in sync with the published [`@oodarun/cli`](https://www.npmjs.com/package/@oodarun/cli).
When a CLI command or flag changes, update `skills/ooda/SKILL.md` here.
