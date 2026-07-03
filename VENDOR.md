# Vendor: open-lovable

- **Fork (ours):** https://github.com/codehornets/open-lovable
- **Upstream parent:** [firecrawl/open-lovable](https://github.com/firecrawl/open-lovable)
- **Stack:** Next.js 15 (App Router) + React 19 + TypeScript. Uses **Firecrawl**
  (`@mendable/firecrawl-js`) to scrape a target URL, the **Vercel AI SDK** (`ai` +
  `@ai-sdk/{anthropic,openai,google,groq}`) to regenerate it as a React app, and a
  **code-execution sandbox** (Vercel Sandbox *or* **E2B**) to install/run the
  generated project and stream a live preview. Tailwind + Radix UI frontend.

## Layer & role

**Landing-page cloning / regeneration engine.** Point it at any public URL → it
scrapes the page + brand styles and rebuilds it as an editable React project. This
is the open-source, self-hosted analogue of SmartCloner (which only targets
GoHighLevel). For outreach360 it slots next to `generateLandingPage → R2`: clone a
competitor/reference page into editable code we host on the Cloudflare worker + R2
bucket, instead of hand-authoring from scratch.

## Deploy classification: HTTP SERVICE (deploy-ready)

Serves a Next.js web app + `/api/*` REST routes (`scrape-url-enhanced`,
`scrape-website`, `extract-brand-styles`, `generate-ai-code-stream`,
`create-ai-sandbox`, …). No thin wrapper needed.

- **Entrypoint:** `pnpm start` (`next start`) after `pnpm build`. See
  `Dockerfile.railway`.
- **PORT:** `next start` binds Railway's injected `$PORT` automatically (falls back
  to `3000`). No fixed-port shim needed — unlike local-deep-research.
- **Health route:** `GET /` (the home page renders without any API key) — used as
  `healthcheckPath`.

### Scaffolding added (this dir, additive only)

- `railway.json` — DOCKERFILE builder → `Dockerfile.railway`, healthcheck `/`.
- `Dockerfile.railway` — two-stage `pnpm install --frozen-lockfile` → `pnpm build`
  → `pnpm start`. Deliberately does **not** switch the app to Next `standalone`
  output, so the vendored `next.config.ts` is left untouched (additive-only, like
  the other vendored dirs). Trade-off: fatter image, upstream-clean source.
- `.env.local` — local dev env (gitignored via `.env*`); placeholders only, no
  secrets. Defaults `SANDBOX_PROVIDER=e2b` (self-host-friendly; Vercel Sandbox
  needs a linked Vercel project / OIDC).

## Key env vars

- `FIRECRAWL_API_KEY` — **required**; the scraper. https://firecrawl.dev
- **One** LLM provider key — `ANTHROPIC_API_KEY` (default/best), or `OPENAI_API_KEY`
  / `GEMINI_API_KEY` / `GROQ_API_KEY`. (Or `AI_GATEWAY_API_KEY` to fan out via
  Vercel AI Gateway.)
- **Sandbox** — `SANDBOX_PROVIDER=e2b` + `E2B_API_KEY` (https://e2b.dev), *or*
  `SANDBOX_PROVIDER=vercel` + a linked Vercel project (`VERCEL_OIDC_TOKEN`, or
  `VERCEL_TOKEN`/`VERCEL_TEAM_ID`/`VERCEL_PROJECT_ID`).
- `MORPH_API_KEY` — optional; faster incremental edits ("fast apply").

## How outreach360 calls it

A Base44 function POSTs a target URL to `OPEN_LOVABLE_URL` (this service's
`/api/*`), authenticated with a shared bearer token, mirroring the
`GMAPS_SCRAPER_URL` / `LOCAL_DEEP_RESEARCH_URL` pattern. Set `OPEN_LOVABLE_URL`
(+ token) in Base44 env once the Railway service is live. Use it to seed editable
landing-page code from a reference URL; the generated HTML/React then feeds the
existing R2 site-hosting path.
