# Security

This repo contains an [Agent Skill](https://code.claude.com/docs/en/skills):
markdown instructions that teach a coding agent how to use the
[`@oodarun/cli`](https://www.npmjs.com/package/@oodarun/cli) npm package. The
skill ships **no executable code**. This document states what the skill tells
an agent to do, and what the CLI it drives can access.

## What the CLI accesses

| Access | Scope | Why |
|--------|-------|-----|
| Read the project directory | Build output (`dist`, `build`, `out`, …) and `ooda.json` | Upload the built site |
| Write `ooda.json` | Project root | Persist the site slug and display metadata between publishes |
| Write `~/.ooda/auth.json` (mode 0600) | User home | Store the org session after login, and the current org after `ooda switch` |
| Network | HTTPS to `api.ooda.run` only | Publish files, manage sites, set secrets |

The CLI does not read files outside the project directory and `~/.ooda/`. It
does not run the project's build. It does not talk to any host other than
`api.ooda.run` (or the host set in `OODA_API_BASE` for local development).

## Authentication

The skill instructs the agent, in order of preference:

1. Ask the user to run `ooda login` themselves in their own terminal.
2. Use the `OODA_ACCESS_TOKEN` + `OODA_ORG_ID` environment variables in
   headless or CI environments.
3. As a fallback, use the email-code flow, where the **user** reads a 6-digit
   code from their own inbox and relays it in chat. The code is single-use and
   expires after 10 minutes. The skill explicitly forbids the agent to open or
   search the user's mailbox, and forbids guessing the account email.

The agent never sees or handles an account password.

## Secrets are write-only

`ooda secrets set` stores a value; no CLI command prints one back. This is
deliberate: the CLI is often driven by an LLM, and any printed value would land
in a chat transcript. To read a value, an org admin reveals it in the
dashboard. The one exception is `ooda sites password <slug>`, which prints a
site's **visitor access password** — a shareable gate for viewers, not an
account credential.

On the input side, the skill instructs the agent to keep true secret values out
of the model's context: the user either runs `ooda secrets set` themselves, or
exports the value as a shell variable that the agent's command expands
(`ooda secrets set KEY="$KEY" …`). The agent must not ask the user to paste a
secret into the chat. Only non-secret `--env` config (URLs, flags, public/anon
keys) is passed inline.

## Reporting a vulnerability

Open an issue in this repo, or email jonny@jonnyburch.com. Do not put secrets
or tokens in an issue.
