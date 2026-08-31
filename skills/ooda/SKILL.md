---
name: ooda
description: >
  Publish and manage static websites on ooda.run from the command line. Use when
  the user wants to publish a built static site or SPA to a shareable
  {slug}.ooda.run URL, list/update/unpublish their published sites, or set
  per-site access (public, password-protected, or ooda-login), or set env vars &
  secrets for a site. Every command runs non-interactively via the
  `ooda` CLI, so an agent can drive the whole lifecycle. Triggers: "publish this
  site", "put this online", "deploy my built site", "share this as a URL", "list
  my ooda sites", "password-protect / make public / unpublish a site", "set an
  env var / API key / secret for my site", "use window.__OODA_ENV__",
  "call an authenticated/OpenAI API from a published site", "proxy a secret API
  key", "wire up the ooda.json secrets manifest".
---

# ooda

ooda publishes static sites to a permanent, shareable URL at `{slug}.ooda.run`
and lets you manage them — all from the CLI, non-interactively. A site in a paid
org also gets a short `{site}.{org}.ooda.run` address (see "Two addresses").

Use this skill when the user wants to:

- **Publish** a built static site / SPA to a public URL.
- **List** the sites they've already published.
- **Control access** to a site: public, password-protected, or ooda-login-only.
- **Set env vars & secrets** for a site (non-secret config reaches the
  page as `window.__OODA_ENV__`; true secrets never do).
- **Unpublish** a site.

## Install

Install the CLI globally and use the `ooda` command — it prints an "update
available" nudge when a newer version is published, so a global install won't go
stale:

```bash
npm install -g @oodarun/cli
```

Requirements: Node.js 20+, and an ooda account in an organization (publishing is
org-scoped).

> You can also run without installing via `npx @oodarun/cli@latest <command>`,
> but prefer the global `ooda` command — it is shorter and more reliable
> across shells. All examples below use `ooda`.

To (re)install this skill: `npx skills add toy-studio/ooda-skills -g` (see the
repo README).

## Authentication (do this first)

ooda commands need an org session — an **existing** account (signup is invite-only).

**Before publishing, check the session — don't just try a command** (an unauthed
command may trigger an interactive prompt you can't answer):

```bash
ooda whoami   # exits 0 + prints the org when signed in, non-zero otherwise
```

If it exits non-zero, authenticate with one of these paths, in order of
preference:

1. **The user logs in themselves (preferred).** Ask the user to run `ooda login`
   in their own terminal and sign in once. The session is saved to
   `~/.ooda/auth.json` and reused by every later command. Tell them:
   > "Run `ooda login` once, then I can publish for you."

2. **Environment variables (headless / CI).** Set both and the CLI skips login:
   - `OODA_ACCESS_TOKEN` — the user's JWT.
   - `OODA_ORG_ID` — the org to act in.

3. **Email-code login from chat (fallback; CLI 0.1.15+).** Use this only when
   the user can't run a terminal themselves. The user stays in control of the
   credential at every step:
   ```bash
   ooda login --email <their-email>            # emails a 6-digit code
   # the USER reads the code from their own inbox and types it into the chat:
   ooda login --email <their-email> --code <code> [--org <id>] [--json]
   ```
   Rules for this path:
   - **Ask the user which email their ooda account uses — do NOT guess it** (e.g.
     from git config or the repo). A wrong address silently sends nothing (the
     endpoint never reveals whether an account exists), so a guess just wastes a
     round-trip. If they've logged in before, `ooda whoami` prints the account email.
   - **Never open or search the user's mailbox for the code**, even if you have
     email tools. The user reads the email and relays the code themselves.
   - The code is single-use and expires after 10 minutes. Requesting a new code
     cancels any earlier one — use the latest email.
   - If the account is in several orgs, pass `--org <id>` (the error lists the
     options).

## Publish a site

```bash
# Run from the PROJECT ROOT — not the build folder.
ooda publish [--slug <slug>] [--title "<name>"] [--description "<text>"] [--tags <a,b,c>] [--message "<what changed>"] [--json]
```

- Publishes an **already-built** static site — it does **not** run your build.
  Run the project's build first (`npm run build`, `pnpm build`, etc.).
- **Run it from the project root and let it auto-detect the build output**
  (`dist`, `build`, `out`, `.output/public`, or `.next/static`). The optional
  `[path]` argument is the **project directory**, not the build folder — it
  defaults to the current dir.
  - ✅ `ooda publish` (in the project root)
  - ✅ `ooda publish ./my-app` (path to a project root)
  - ❌ `ooda publish ./dist` — **wrong**: it looks for a build dir *inside*
    `./dist` and fails. Don't pass the build folder.
- On success it prints the live URL, e.g. `https://my-app-k3x9.ooda.run` —
  note the short random suffix; every new site gets one (see "Naming the site").
- `--json` gives machine-readable output: `{ ok, url, slug, version, fileCount,
  totalSize, promoted, live }`, plus `requestedSlug` when the name you asked for
  wasn't available and the CLI published to a different one (CLI 0.1.35+), plus
  `prettyUrl` when the org has a short address (CLI 0.1.37+ — see "Two addresses").
  **Always read the slug/URL back from the output rather than assuming the one
  you passed** — see "Naming the site".
- Every publish appends a new **version** and makes it live. Pass **`--draft`**
  to append a version *without* making it live (it stays behind the current live
  version); the CLI prints a `?v=N` preview URL, and you promote it later with
  `ooda sites pin <slug> N` (see "Versions & rollback").

### Naming the site (choose a good slug)
The slug is the site's public name in the URL `{slug}.ooda.run`, so name it
after **the user's project**, not the tool.

- **Every NEW site's URL ends in a short random suffix** — 4 characters
  including a digit, e.g. `acme-marketing-k3x9.ooda.run`. The server requires
  it (clean bare names are kept for a future org-subdomain scheme), and the
  CLI appends it automatically — even to an explicit `--slug` (CLI 0.1.32+).
  Don't try to fight or strip it; just pick a good descriptive **base** name.
  Existing sites keep the exact URL they already have.
- **Don't name the site after ooda or the template it came from.** "ooda" is
  the tool, not the user's site, and `ooda-react-blog` is the starter's name —
  if you clone a template, publish under a slug describing *their* project, not
  the template. (An `ooda-…` slug is allowed and will publish; it's just rarely
  what the user wants.) Avoid generic tool names (`site`, `dist`, `app`) for the
  same reason.
- Derive a descriptive slug from the **project**: its `package.json`/`ooda.json`
  name, the repo name, or what the user calls it (e.g. `acme-marketing`,
  `portfolio-2026`). Pass it with `--slug <name>`.
- **If it's not obvious what the site should be called, ask the user** before
  publishing — the slug is in the URL you'll share, and changing it later means a
  new URL.
- Confirm the URL back to the user after publishing so a wrong name is caught
  early — share the **exact URL the CLI printed**, which includes the suffix.

### Reserved & disallowed names
The server enforces a naming policy on every publish. **An unavailable name
does not fail the publish** (CLI 0.1.35+): the CLI resolves it and prints where
the site actually landed, so you never need to hunt for a free slug or retry
with a different one.

- **Random suffix (every new site).** A NEW site's slug must end in a random
  suffix (4 trailing characters including a digit). The CLI handles this
  automatically (0.1.32+) whatever the slug's origin, so you normally never
  see the underlying `suffix_required` rejection — older CLIs surface it as an
  error on an explicit `--slug` (update the CLI). Republishing an existing
  site is unaffected, whatever its slug looks like.
- **Reserved names.** Names that look like official ooda pages or
  infrastructure (`login`, `privacy`, `terms`, `docs`, `www`, `api`, `drop`, …)
  can't be claimed as-is. The suffix every new site gets already clears them
  (`privacy-k3x9` is fine), so the publish just succeeds.
- **The `ooda-` prefix is not reserved.** `ooda-react-blog-k3x9` and friends
  publish normally, so a template's own name is never a blocker. The random
  suffix every new site carries is what separates a user site from ooda's own
  pages. (Bare `ooda` is a reserved name like the ones above — it publishes as
  `ooda-k3x9`.)
- **Disallowed words.** Profanity and slurs are blocked in the slug **and** in
  `--title`/`--description`/`--tags`. No suffix or rename by the CLI fixes
  these — the publish (or a later metadata update) is refused with a message
  naming the offending field. If a folder name trips this, pick a clean
  `--slug` instead of retrying.

### Title, description & tags (set these!)
The slug is just the URL. Each site also carries display/search metadata you
should set on publish (CLI 0.1.25+):

- **`--title "Human Name"`** — the display name shown in the dashboard instead
  of the slug. Defaults to a prettified slug if unset, so always pass a real one.
- **`--description "…"`** — what the site is: its purpose, key features, and
  tech. Be specific and keyword-rich — this powers search across many sites.
- **`--tags blog,astro,marketing`** — 3–6 short lowercase keywords (kind of
  site, framework, domain).
- Title/description/tags are saved to `ooda.json`, so re-publishing keeps them
  without re-passing the flags.
- **Brand colour — set `"color": "#rrggbb"` in `ooda.json`** (CLI 0.1.29+):
  the dominant accent/brand colour of the site's design. The dashboard tints
  the site's card and preview frame with it. There's no flag — add the field
  to `ooda.json` before publishing. Omit it only if the site has no
  distinctive colour (the dashboard falls back to its default).
- **`--message "what changed"`** (or `-m`) — a per-publish note, recorded
  against that version like a commit message. Use it on re-publishes.

### Slugs are global and auto-deduplicated
`{slug}.ooda.run` is a global subdomain, so slugs are unique across all orgs.
(Sites published under the older `{slug}-p.ooda.run` scheme still work — those
URLs 301-redirect to the bare `{slug}.ooda.run` — so old shared links don't break.)

- Without `--slug`, the CLI derives a base name from `ooda.json`'s `name` or the
  folder name (it never uses "ooda" unless that's literally your project/folder
  name) and appends the required random suffix for a new site.
- It writes the resolved slug back to `ooda.json`, so **re-publishing the same
  project keeps the same URL** — suffix included.
- With `--slug <name>` you choose the base explicitly. A new site still gains
  the random suffix (CLI 0.1.32+).
- **A taken or reserved name never fails the publish** (CLI 0.1.35+). Whatever
  the slug's origin — derived, `--slug`, or the one saved in `ooda.json` — the
  CLI rolls a fresh suffix (or takes the alternative the server suggests) and
  publishes there. **Don't try to pick a free slug yourself, and don't retry a
  publish with a different name because one was unavailable** — read the slug
  and URL the CLI printed instead.
- **It won't clobber a different site in your org.** If the slug already belongs
  to another project in your org, the publish goes to a new suffixed slug rather
  than overwriting it, and says so. Pass `--force` only if you really mean to
  replace that site's contents. (CLI 0.1.19+; before 0.1.35 an explicit `--slug`
  was refused here instead.)

### Two addresses (paid orgs get a short one)
A site always answers on its global address, `https://{slug}.ooda.run` — suffix
and all. That address never changes and always works.

An org on a paid plan that has chosen an organisation slug **also** gets a short
address for every one of its sites (CLI 0.1.37+):

```
https://acme-marketing-k3x9.ooda.run     the global address — always works
https://acme-marketing.near-future.ooda.run   the short address — one org only
```

- The short one drops the random suffix, because the org name already makes it
  unique. Two orgs can both own `blog`.
- **Exactly one of the two is canonical, and the other redirects to it.** While
  the org pays, the global address 301s to the short one. If the org stops
  paying, the redirect reverses. **No link ever breaks**, whichever one was
  shared.
- The org does not choose per site. Every site in the org gets its short
  address at once, with no republish.

What this means for you:

- **Share the address the CLI printed.** `ooda publish` prints the canonical one
  first and the other under `Also at:`. Don't build a URL yourself from the slug.
- With `--json`, read `prettyUrl` when it is there and `url` otherwise.
- `ooda publish` records both in `ooda.json` — `urls.canonical` (share this),
  `urls.plain`, and `siteName` (the short name). They are written by the CLI and
  are read-only: editing them by hand does nothing, because renaming a site is a
  server operation.
- **A brand-new organisation subdomain needs a minute or two** before its TLS
  certificate exists. The CLI says so when it happens. Share the plain address
  until then; it works immediately.

The random suffix rule is unchanged by any of this — the global slug is always
suffixed, and the short name is separate.

### Is the project publishable?
ooda serves a **static snapshot** (built HTML/CSS/JS) — there is no running
server. This works for:
- Static sites (Astro, Hugo, plain HTML)
- Client-side SPAs (React, Vue, Svelte with client-side routing)
- Static exports (Next.js `output: "export"`, Nuxt `ssr: false`)

It does **not** work for apps that need a **server** at runtime (SSR, `/api`
routes, a runtime database). If the project needs a server, tell the user it
can't be published as a static site.

**Non-secret runtime config does work**, though: API base URLs, feature flags,
and public/anon keys can be set as env vars and read in the browser as
`window.__OODA_ENV__.KEY` — no rebuild needed (see "Env vars & secrets" below).
A site that only needs public config can be published; one that needs a real
secret in the browser cannot (true secrets are never exposed to the page) — but a
site that needs to *call* a secret-authenticated API can, via the secret-injecting
proxy (see "Env vars & secrets" below).

## Manage published sites

```bash
ooda sites list [--json]
ooda sites access <slug> --mode <public|password|login> \
    [--password <pw> | --clear-password] [--clear-mode] [--json]
ooda sites password <slug> [--json]
ooda sites delete <slug> [--json]
```

- **`sites list`** — every site in the org, with its URL, effective access mode,
  and owner. Use `--json` to parse it: the JSON entries also include each site's
  `title`, `description`, and `tags`, so when you need to find the right site to
  re-publish or update, list with `--json` and match on those rather than
  guessing from the slug. (CLI 0.1.27+.)
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

## Versions & rollback

Every publish appends a version; the newest is normally live. You can roll back
to any earlier version — it's a pointer move, nothing is re-uploaded.

```bash
ooda sites versions <slug> [--json]      # list versions; marks the live one (live/pinned)
ooda sites pin <slug> <version>          # make an earlier version live ("pin" it)
ooda sites pin <slug> latest             # re-pin the newest version (undo a rollback)
```

- **`sites versions <slug>`** — the publish history (newest first) with each
  version's file count, size, and `--message` note; the live one is marked
  `live` when it's the newest, `pinned` when an older version is being served.
- **`sites pin <slug> <version|latest>`** — set which version is live. Pinning an
  older version keeps it serving until you pin another **or publish new content**
  — a normal publish appends past the newest version and goes live, superseding
  the pin. Use `ooda publish --draft` to add a version without disturbing the
  pinned/live one.

## Env vars & secrets

Give a site configuration without hardcoding it. Managed entirely from
the CLI, non-interactively. There are two kinds:

- **`--env` = non-secret config** (API base URL, feature flag, public/anon key).
  For a published site it's delivered to the page as `window.__OODA_ENV__.KEY` —
  read it there at runtime, **not** from `import.meta.env`/`process.env` (those
  are build-time only on a static host). Changing a value re-materializes the
  site with **no rebuild**. Safe to be public.
- **secret** (the default, no `--env`) = a true secret. It is **never** sent to a
  site's browser, and you can **never** read a value back from the CLI.

```bash
# Always pass --description (alias --desc) when setting a variable.
ooda secrets set API_URL=https://api.example.com --env --description "API base URL"   # global config (admin)
ooda secrets set STRIPE_KEY="$STRIPE_KEY" --site <slug> --description "Stripe live key" # this site only (private; shell-expanded — see below)
ooda secrets list                                                # the global catalog: key, kind + description
ooda secrets list --site <slug>                                  # masked: this site's keys (used global + private)
ooda secrets use KEY --site <slug>                               # record that this site uses a global var
ooda secrets rm KEY [--site <slug>] [--force]                    # global rm blocked while in use
```

- **Global vs per-site.** With no scope flag an **org admin** sets a *global*
  var — a flat, org-wide namespace of shared config/secrets. A site only
  **receives** a global once it's recorded as **using** it (`ooda secrets use`, or
  the manifest via `secrets check`), so globals don't silently leak everywhere.
  With `--site <slug>` the site's owner (or an admin) sets a *private* value
  pinned to that one site, which overrides a global of the same name.
- **Reuse globals when going live.** Before publishing, run `ooda secrets list
  --json` to see the global catalog (each var's `key`, `kind` + `description`,
  never values), suggest the relevant ones to the user, and `ooda secrets use
  <KEY> --site <slug>` to adopt them — don't re-create values that already exist.
- **There is no `reveal` command — ever.** Values are write-only from the CLI
  (the CLI is LLM-driven, so any printed value would leak). To **read** a value
  back, an admin reveals it in the dashboard. Don't try to print or echo secrets.
- **Keep true secret values out of the chat.** Do not ask the user to paste a
  secret (API key, token) into the conversation, and do not type one into a
  command yourself. Use one of these flows instead:
  1. The user runs the `ooda secrets set` command themselves in their terminal.
  2. The user exports the value as a shell variable (e.g. in their shell or a
     gitignored `.env` they source), and you run the command with **shell
     expansion**, so the value never enters the model's context or transcript:
     ```bash
     ooda secrets set STRIPE_KEY="$STRIPE_KEY" --site <slug> --desc "Stripe live key"
     ```
  Non-secret `--env` config (URLs, flags, public/anon keys) is fine to pass
  inline — it's public by design.
- **Always set `--description` (alias `--desc`) when adding a variable.** Pass a
  short human label on every `ooda secrets set` so `ooda secrets list` shows what
  each key is for and teammates/agents can pick the right global to reuse. It's
  metadata only — the value itself is still never printed — but an undescribed
  key is hard to identify later, so treat the description as required, not optional.

### Call an authenticated API from a published site (secret-injecting proxy)

A published site has no server, so it can't hold a true API key. ooda's backend can
proxy for it: the browser calls a **same-origin** path and ooda injects the stored
secret server-side — the key never reaches the browser. Drive it with two
site-scoped secrets, where `<name>` uppercases to `<NAME>` (non-alphanumerics → `_`):

```bash
ooda secrets set PROXY_OPENAI_URL=https://api.openai.com --env --site <slug>  # upstream base (public)
ooda secrets set PROXY_OPENAI_KEY="$OPENAI_API_KEY"            --site <slug>  # true secret — shell-expanded,
                                                                              # never pasted into chat
```

The site's JS then calls the proxy path (no key client-side):

```js
fetch("/__ooda/proxy/openai/v1/chat/completions", {
  method: "POST",
  body: JSON.stringify({ model, messages, stream: true }),
})
```

It works for any Bearer-auth API (not just OpenAI) and supports streaming. The proxy
inherits the site's access mode, so password/login-gate a public demo
(`ooda sites access <slug> --mode password|login`) and use a spend-capped key.

### Declared secrets (`ooda.json` `secrets` manifest)

A repo can declare the values it needs in a `secrets` array in `ooda.json`, so an
agent knows what to ask for. Each entry has a `key`, optional `kind` (`"env"` |
`"secret"`, default `"secret"`), `description`, `example`, `default`, and
`required` (default `true`):

```json
{
  "secrets": [
    { "key": "PROXY_OPENAI_URL", "kind": "env",
      "default": "https://api.openai.com", "description": "OpenAI upstream base" },
    { "key": "PROXY_OPENAI_KEY", "kind": "secret",
      "description": "OpenAI API key", "example": "sk-..." }
  ]
}
```

To wire them up, run `ooda secrets check --json --apply-defaults` (publish first so a
site slug exists). It records usage of any declared key already provided as a
**global** var, applies any entry with a `default` (as a per-site private value),
and returns the still-missing required keys, each with its `key`, `kind`,
`description`, and `example`. For each missing one, first check whether a global
already covers it (`ooda secrets list --json`) and `ooda secrets use` it.
Otherwise set it per-site — for an `"env"` entry, ask the user for the value
(show its `description`/`example`) and pass it inline with `--env`; for a
`"secret"` entry, use the transcript-safe flow above (the user runs the command
themselves, or exports a shell variable you expand):

```bash
ooda secrets set KEY=VALUE --env --site <slug> --description "<desc>"   # kind: env
ooda secrets set KEY="$KEY" --site <slug> --description "<desc>"        # kind: secret
```

Pass the entry's `description` through. Re-run `ooda secrets check` until it
reports nothing missing.

## What to tell the user

- After publishing, always share the live URL **exactly as the CLI printed it**.
  On a paid org that is the short `{site}.{org}.ooda.run` address; otherwise it
  is `{slug}.ooda.run`. See "Two addresses".
- **New sites may default to `login` access** (org members only). If the user
  wants it openly shareable, run `ooda sites access <slug> --mode public` and tell
  them it's now public. If you leave it login-gated, tell them only members of
  their ooda org can open it.
- For a **password**-protected site, share both the URL and the password.

## Security & data access

This skill is documentation only — it ships no code. It teaches the agent to
drive the published [`@oodarun/cli`](https://www.npmjs.com/package/@oodarun/cli).
What that CLI touches:

- **Reads** the project directory: the build output and `ooda.json`.
- **Writes** `ooda.json` (slug + metadata) and the session file
  `~/.ooda/auth.json` (mode 0600).
- **Network**: HTTPS to `api.ooda.run` only.
- **Secrets are write-only.** The CLI can set a secret but can never read one
  back — there is no reveal command, so a value can't leak into a chat
  transcript. Admins reveal values in the dashboard instead.
- **The agent never handles passwords** and must never read the user's mailbox.
  See the authentication section above.

Full disclosures:
[SECURITY.md](https://github.com/toy-studio/ooda-skills/blob/main/SECURITY.md).

## Full reference

```bash
ooda --help
```

## Troubleshooting

- **`npx @oodarun/cli` fails or behaves unexpectedly in your shell** → install
  the CLI globally (`npm install -g @oodarun/cli`) and use the `ooda` command.
- **It tries to prompt for a login** → no saved session and no env vars. Have the
  user run `ooda` and log in once, or set `OODA_ACCESS_TOKEN` + `OODA_ORG_ID`.
- **"No build output found"** → run the project's build first, and run `ooda
  publish` from the project root (not the build folder).
- **"New site URLs get a random suffix" / `suffix_required`** → you're on an
  older CLI (< 0.1.32) publishing a new site with an explicit `--slug`. Update
  the CLI (`npm install -g @oodarun/cli`) — newer versions append the required
  suffix automatically.
- **"Slug … is taken by another organisation" / "… is reserved for ooda.run's
  own pages"** → shouldn't happen on CLI 0.1.35+, which suffixes past both and
  publishes. If you see it, update the CLI (`npm install -g @oodarun/cli`); as a
  one-off, publishing with any `--slug` variant (`<name>-app`) also works.
- **"ooda.run rejected 5 slugs …"** (older CLIs: "Couldn't find a free slug
  after 5 attempts") → the CLI tried five suffixed names and every one was
  rejected. On CLI 0.1.35+ this is rare and means bad luck on collisions, not a
  policy you can't satisfy: read the reason the message quotes, then publish
  with a different `--slug` base. On an older CLI it usually means the base name
  itself hits a rule the CLI can't suffix past — `ooda-` template names
  (`ooda-react-blog`) used to fail this way, which is fixed server-side.
- **"… contains a word that isn't allowed on ooda.run"** → the slug, title,
  description, or tags tripped the banned-word filter. A suffix won't help —
  choose a different name for the flagged field. Don't retry the same value.
- **The short `{site}.{org}.ooda.run` address fails TLS right after publishing**
  → the organisation's subdomain is new and its certificate is still being
  issued. It takes a minute or two. Share the plain `{slug}.ooda.run` address
  meanwhile — it works immediately, and it redirects to the short one once the
  certificate lands.
- **Visiting the URL loops or shows "you don't have access"** → the site is
  `login`-gated and you're signed in to an account that isn't in the site's org.
  Make it public (`ooda sites access <slug> --mode public`) or sign in with an
  account in that org.
- **Styles / fonts / images missing after publishing** → make sure those files
  are in the build output. The site is served at the root of its own subdomain,
  so absolute paths like `/assets/...` and `/_next/...` work.
