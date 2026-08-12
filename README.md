# ooda agent skills

Installable [Agent Skills](https://code.claude.com/docs/en/skills) that teach a
coding agent (Claude Code, Cursor, etc.) how to use [ooda](https://ooda.run).

## `ooda`

Publish and manage static websites on `ooda.run` from the CLI — publish a built
site to a shareable `{slug}.ooda.run` URL, list/unpublish sites, and control
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

- Node.js 20+ (the skill drives the [`@oodarun/cli`](https://www.npmjs.com/package/@oodarun/cli) npm package).
- An ooda account in an organization. The user logs in once with `ooda login`
  (the saved session in `~/.ooda/auth.json` is reused), or sets the
  `OODA_ACCESS_TOKEN` + `OODA_ORG_ID` environment variables for headless use.

## Security

The skill ships no code — it is markdown instructions for the published CLI.
See [SECURITY.md](./SECURITY.md) for what the CLI accesses and why, the
authentication rules, and the write-only secrets model.

## Releasing

Keep the skill in sync with the published [`@oodarun/cli`](https://www.npmjs.com/package/@oodarun/cli).
When a CLI command or flag changes, update `skills/ooda/SKILL.md` here.
