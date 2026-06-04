---
name: ooda
description: >
  Publish and manage static websites on ooda.run from the command line. Use when
  the user wants to publish a built static site or SPA to a shareable
  {slug}-p.ooda.run URL, list/update/unpublish their published sites, or set
  per-site access (public, password-protected, or ooda-login). Every command runs
  non-interactively via `npx @oodarun/cli`, so an agent can drive the whole
  lifecycle. Triggers: "publish this site", "put this online", "deploy my built
  site", "share this as a URL", "list my ooda sites", "password-protect / make
  public / unpublish a site".
---

# ooda

ooda publishes static sites to a permanent, shareable URL at `{slug}-p.ooda.run`
and lets you manage them — all from the CLI, non-interactively.

Use this skill when the user wants to:

- **Publish** a built static site / SPA to a public URL.
- **List** the sites they've already published.
- **Control access** to a site: public, password-protected, or ooda-login-only.
- **Unpublish** a site.

## What else ooda does (mostly interactive — outside this skill)

ooda's other half is **cloud dev environments**: each project runs in its own
cloud sandbox with Claude Code and a live URL. Those commands are **interactive**
(they open the dashboard or spawn a TTY Claude session), so they're for the human
to run, not for an agent to drive headlessly:

- `npx @oodarun/cli` — interactive project menu + local dashboard.
- `ooda deploy [path|github-url]` — spin up a cloud dev environment from a folder
  or GitHub repo.
- `ooda connect <project>` — open an existing project and run Claude in it.
- `ooda list` — list the org's projects (this one *is* non-interactive; `--json`
  works).

This skill covers the **non-interactive** surface an agent can drive on its own:
authentication, publishing static sites, and managing published sites. When a
user wants a running dev environment (not a static publish), point them at
`ooda deploy` / the dashboard rather than `ooda publish`.

## Requirements

- Node.js 20+ (so `npx` is available).
- An ooda account that belongs to an organization — publishing is org-scoped.

No install step is required for the CLI itself: run `npx @oodarun/cli@latest
<command>`. Optionally `npm install -g @oodarun/cli` for a shorter `ooda <command>`.

To (re)install this skill: `npx skills add toy-studio/ooda-skills -g` (see the
repo README).

## Authentication (do this first)

ooda commands need an org session — an **existing** account (signup is invite-only).
Pick whichever fits:

1. **Email-code login — agent-drivable** (recommended; CLI 0.1.15+).
   You can run the whole login from chat — no password, no terminal prompt:
   ```bash
   npx @oodarun/cli@latest login --email <their-email>            # emails a 6-digit code
   # ask the user for the code from their inbox, then:
   npx @oodarun/cli@latest login --email <their-email> --code <code> [--org <id>] [--json]
   ```
   On success a session is saved to `~/.ooda/auth.json` and reused by every later
   command. If the account is in several orgs, pass `--org <id>` (the error lists
   the options).

2. **Saved login (any version).** The user runs `npx @oodarun/cli` (or
   `npx @oodarun/cli login`) once and signs in interactively; the saved session is
   reused afterwards.

3. **Environment variables (headless / CI).** Set both and the CLI skips login:
   - `OODA_ACCESS_TOKEN` — the user's JWT.
   - `OODA_ORG_ID` — the org to act in.

**Before publishing, check the session — don't just try a command** (an unauthed
command may trigger an interactive prompt you can't answer):

```bash
npx @oodarun/cli@latest whoami   # exits 0 + prints the org when signed in, non-zero otherwise
```

If it exits non-zero, authenticate with one of the options above. If you can't,
tell the user:
> "Run `npx @oodarun/cli` and log in once, then I can publish for you."

## Publish a site

```bash
# Run from the PROJECT ROOT — not the build folder.
npx @oodarun/cli@latest publish [--slug <slug>] [--json]
```

- Publishes an **already-built** static site — it does **not** run your build.
  Run the project's build first (`npm run build`, `pnpm build`, etc.).
- **Run it from the project root and let it auto-detect the build output**
  (`dist`, `build`, `out`, `.output/public`, or `.next/static`). The optional
  `[path]` argument is the **project directory**, not the build folder — it
  defaults to the current dir.
  - ✅ `npx @oodarun/cli@latest publish` (in the project root)
  - ✅ `npx @oodarun/cli@latest publish ./my-app` (path to a project root)
  - ❌ `npx @oodarun/cli@latest publish ./dist` — **wrong**: it looks for a build
    dir *inside* `./dist` and fails. Don't pass the build folder.
- On success it prints the live URL, e.g. `https://my-app-p.ooda.run`.
- `--json` gives machine-readable output: `{ ok, url, slug, version, fileCount, totalSize }`.

### Slugs are global and auto-deduplicated
`{slug}-p.ooda.run` is a global subdomain, so slugs are unique across all orgs.

- Without `--slug`, the CLI derives one from `ooda.json`'s `name` or the folder
  name. If that's already taken, it appends a short random suffix automatically.
- It writes the resolved slug back to `ooda.json`, so **re-publishing the same
  project keeps the same URL**.
- With `--slug <name>` you choose explicitly. An explicit slug is used as-is and
  is **not** auto-suffixed — if it's taken by another org you'll get an error and
  should pick a different one.

### Is the project publishable?
ooda serves a **static snapshot** (built HTML/CSS/JS) — there is no running
server. This works for:
- Static sites (Astro, Hugo, plain HTML)
- Client-side SPAs (React, Vue, Svelte with client-side routing)
- Static exports (Next.js `output: "export"`, Nuxt `ssr: false`)

It does **not** work for apps that need a server at runtime (SSR, `/api` routes,
a database, runtime secrets). If the project needs a server, tell the user it
can't be published as a static site.

## Manage published sites

```bash
npx @oodarun/cli sites list [--json]
npx @oodarun/cli sites access <slug> --mode <public|password|login> \
    [--password <pw> | --clear-password] [--clear-mode] [--json]
npx @oodarun/cli sites password <slug> [--json]
npx @oodarun/cli sites delete <slug> [--json]
```

- **`sites list`** — every site in the org, with its URL, effective access mode,
  and owner. Use `--json` to parse it.
- **`sites access <slug>`** — set the access policy:
  - `--mode public` — anyone can view.
  - `--mode password --password <pw>` — visitors must enter the password.
  - `--mode login` — only members of the user's ooda org can view (they sign in).
  - `--clear-password` removes the per-site password.
  - `--clear-mode` reverts to the org default.
- **`sites password <slug>`** — reveal the effective password of a
  password-protected site.
- **`sites delete <slug>`** — unpublish the site and remove its files.

Every command exits `0` on success and non-zero on failure, printing the server's
error message. Add `--json` to any command for structured output.

## What to tell the user

- After publishing, always share the live `https://<slug>-p.ooda.run` URL.
- **New sites may default to `login` access** (org members only). If the user
  wants it openly shareable, run
  `npx @oodarun/cli sites access <slug> --mode public` and tell them it's now
  public. If you leave it login-gated, tell them only members of their ooda org
  can open it.
- For a **password**-protected site, share both the URL and the password.

## Full reference

```bash
npx @oodarun/cli --help
```

## Troubleshooting

- **It tries to prompt for a login** → no saved session and no env vars. Have the
  user run `npx @oodarun/cli` and log in once, or set `OODA_ACCESS_TOKEN` +
  `OODA_ORG_ID`.
- **"No build output found"** → run the project's build first; the CLI publishes
  the built output directory, not the source.
- **"Slug … is taken by another organisation"** (only happens with an explicit
  `--slug`) → choose a different `--slug`, or omit it so the CLI picks a unique one.
- **Visiting the URL loops or shows "you don't have access"** → the site is
  `login`-gated and you're signed in to an account that isn't in the site's org.
  Make it public (`sites access <slug> --mode public`) or sign in with an account
  in that org.
- **Styles / fonts / images missing after publishing** → make sure those files
  are in the build output. The site is served at the root of its own subdomain,
  so absolute paths like `/assets/...` and `/_next/...` work.
