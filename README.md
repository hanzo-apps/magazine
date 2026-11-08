# Press — Digital Magazine Issues

<!-- hanzo-badges -->
<p align="left">
  <a href="https://platform.hanzo.ai/new?repo=https%3A%2F%2Fgithub.com%2Fhanzo-apps%2Fmagazine&template=magazine"><img src="https://gallery.hanzo.ai/deploy.svg" alt="Deploy to Hanzo" height="32" /></a>
  <a href="https://hanzo.ai/chat?template=magazine"><img src="https://gallery.hanzo.ai/badges/edit-in-hanzo.svg" alt="Edit in Hanzo.ai" height="32" /></a>
  <a href="https://gallery.hanzo.ai/templates/magazine"><img src="https://gallery.hanzo.ai/badges/live-preview.svg" alt="Live Preview" height="32" /></a>
  <a href="https://gallery.hanzo.ai/llms/magazine/LLM.md"><img src="https://gallery.hanzo.ai/badges/llm-md.svg" alt="LLM.md Playbook" height="32" /></a>
</p>
<!-- /hanzo-badges -->

A magazine that ships in issues — a real, buildable Hanzo app you fork on
[hanzo.app](https://hanzo.app) and deploy live on Hanzo Cloud.

Editors curate numbered issues; each has a cover story and department sections
across features, dispatch, interview and essay. Readers open an issue and move
between its pieces.

- **UI** — [`@hanzo/gui`](https://www.npmjs.com/package/@hanzo/gui) (the Hanzo
  design system) under Vite + React 19. Bold magazine language: a masthead bar,
  oversized display type, a black/white/one-hot-red palette, two-column feature
  layout with a hanging pull-quote. No Tailwind, no second kit.
- **Auth** — [`@hanzo/iam`](https://www.npmjs.com/package/@hanzo/iam), OAuth2
  **PKCE** against [hanzo.id](https://hanzo.id). No local passwords.
- **Data** — [`@hanzo/base`](https://www.npmjs.com/package/@hanzo/base), the
  IAM-native, org-scoped data plane. Issues and articles are real Base
  collections, provisioned from [`schema.sql`](./schema.sql).

## The three views

| View | What it is |
| --- | --- |
| **Newsstand** | Every issue as a cover, newest first. Compose a new numbered issue; publish or keep it a draft. |
| **Issue contents** | The cover story up top, then department sections. File new pieces into the issue. |
| **Article read** | The feature layout: kicker, oversized headline, standfirst, a hanging pull-quote, and the body set in two columns. |

## Stack (pinned)

| Package | Version |
| --- | --- |
| `react` / `react-dom` | `^19.2.4` |
| `@hanzo/gui` + `@hanzogui/config` | `7.3.0` |
| `@hanzo/iam` | `^0.13.1` |
| `@hanzo/base` | `^0.2.1` |
| `vite` | `^6` (`@vitejs/plugin-react`) |
| `typescript` | `5.9.3` |

## Run it

```sh
npm install
npm run dev        # http://localhost:5173
npm run build      # tsc --noEmit && vite build  ->  dist/
npm run preview    # serve the production build (SPA fallback on)
```

Out of the box it runs against **live** Hanzo (hanzo.id + api.hanzo.ai) — no
config needed to see the cover and sign-in flow. Copy `.env.example` to `.env`
to point at a different environment.

## Environment contract

Only `VITE_`-prefixed vars reach the browser (this is a static SPA — there is no
server). Defaults in parentheses.

| Var | Purpose |
| --- | --- |
| `VITE_HANZO_IAM_URL` (`https://hanzo.id`) | OIDC issuer. |
| `VITE_IAM_CLIENT_ID` (`hanzo-app`) | IAM application (`<org>-<app>`). Per-app clients are provisioned at deploy; `hanzo-app` is the shared fallback. Its redirect-URI list must allow this deploy's `/auth/callback`. |
| `VITE_HANZO_REDIRECT_URI` (`${origin}/auth/callback`) | PKCE redirect. |
| `VITE_HANZO_BASE_URL` (`https://api.hanzo.ai`) | Browser-reachable Hanzo Base data plane. Deploy injects the provisioned URL. |

## How auth works — ambient IAM

`login()` starts an OAuth2 **PKCE S256** redirect to hanzo.id; hanzo.id returns
to `/auth/callback`, where `handleCallback()` exchanges the code for tokens
(stored in `localStorage`, refresh-aware via `offline_access`). Every deployed
app is a static site at `<slug>.hanzo.app`; there is **no server token** — the
SPA authenticates the user in the browser and carries the resulting IAM JWT to
Base.

The one deploy requirement: the IAM client (`VITE_IAM_CLIENT_ID`) must list this
origin's `/auth/callback` as an allowed redirect URI. Register a
`https://*.hanzo.app/auth/callback` wildcard on the shared client so every
forked app works, or register a dedicated `hanzo-<app>` client per template.

## How data works — Base from `schema.sql`

[`schema.sql`](./schema.sql) is the app's `databaseSchema` (SQL DDL). On publish,
Hanzo Cloud translates each `CREATE TABLE` into a Hanzo Base collection
(`provisionBaseFromDDL`, additive + idempotent). Base manages
`id`/`created`/`updated`/`owner`/`org`, stamps `owner`+`org` from the verified
IAM principal, and scopes every row to the caller's org via
`@request.auth.org_id = org` — a teammate in your org sees the row; other orgs
cannot. At runtime the views read/write `issues` and `articles` through
`@hanzo/base/react` (`useQuery`/`useMutation`) carrying the IAM token.

## Deploy — Hanzo Cloud

[`hanzo.yml`](./hanzo.yml) declares a static build (`npm run build` → `dist/`,
served at `<slug>.hanzo.app`) plus the Base schema to provision and the env to
inject. Do **not** build a container image locally — Hanzo Cloud owns builds and
deploys. CI here only proves the template compiles green.

## Layout

```
src/
  main.tsx          entry
  providers.tsx     GuiProvider -> IamProvider -> BaseProvider(client=IAM-token)
  app.tsx           route (/auth/callback) + auth gate
  press.tsx         the magazine design language (palette + primitives)
  nav.ts            Route/Go + the Issue/Article row types
  gui.config.ts     createGui(defaultConfig from @hanzogui/config/v5)
  iam.config.ts     IAM PKCE config
  env.ts            the VITE_ env contract, one place
  lib/base.ts       BaseClient carrying the IAM bearer token
  auth/callback.tsx PKCE return leg
  views/
    signed-out.tsx  the cover (signed-out landing)
    shell.tsx       signed-in route state
    newsstand.tsx   issue covers grid + compose
    issue.tsx       cover story + department sections + file a piece
    article.tsx     feature read (two-column body + pull-quote)
schema.sql          databaseSchema -> Base collections on publish
hanzo.yml           Hanzo Cloud build/deploy manifest
```
