# bbpp — Big Beautiful Peace Prize

A static site for awarding the Big Beautiful Peace Prize. Visitors fill in a recipient's name and achievement, preview the certificate, and send it directly to the recipient's inbox.

**Live site:** https://bigbeautifulpeaceprize.com

## Stack

- **Static site:** Plain HTML/CSS/JS (no build step)
- **Hosting:** Cloudflare Pages (free tier)
- **Certificate rendering & delivery:** [parchment](https://github.com/nopolabs/parchment) Cloudflare Worker
- **Bot protection:** Cloudflare Turnstile (managed challenge widget)
- **Source control:** GitHub (nopolabs/bbpp)

## Monthly cost

| Service | Cost |
|---|---|
| Cloudflare Pages | $0 |
| Domain | ~$1 amortized |
| **Total fixed cost** | **~$0/month** |

## Project structure

```
bbpp/
├── src/
│   ├── index.html            # Single-page app — form, preview, confirmation
│   ├── assets/
│   │   └── bbpp-seal.png     # Prize seal image used in the certificate
│   ├── _headers              # Cloudflare Pages HTTP response headers
│   └── favicon.*             # Favicons
└── functions/
    └── parchment/
        └── [[path]].ts       # Catch-all Pages Function: Turnstile guard + parchment proxy
```

## How it works

1. Visitor fills in recipient name, achievement (optional), and email
2. **Preview** fetches `GET /parchment/render?name=...` → proxied to the parchment worker → returns a certificate PNG
3. Visitor completes the Cloudflare Turnstile challenge (unlocks the **Award the Prize** button)
4. **Award the Prize** posts `POST /parchment/issue` with the Turnstile token
5. The `functions/parchment/[[path]].ts` Pages Function intercepts the POST:
   - Verifies the Turnstile token with Cloudflare's `v0/siteverify` API
   - On success, forwards the request to the parchment worker with an API key in the `Authorization` header
6. Parchment renders the certificate and emails it to the recipient via Resend

## Local development

No build step needed — open `src/index.html` directly in a browser for UI work.

To develop the Pages Function locally:

```bash
npx wrangler pages dev src --compatibility-date=2024-09-23
```

Create a `.dev.vars` file in the repo root for local secrets (never commit this):

```
PARCHMENT_BASE_URL=https://parchment-worker-bbpp.danrevel.workers.dev
PARCHMENT_API_KEY=...
TURNSTILE_SECRET_KEY=1x0000000000000000000000000000000AA
```

> The `TURNSTILE_SECRET_KEY` value above is Cloudflare's "always pass" test secret — use it locally so you don't need to solve a real challenge. Use the real secret in production.

## Deployment

Deployment is automatic — push to `main` on GitHub and Cloudflare Pages builds and deploys.

- **Build command:** *(none)*
- **Build output directory:** `src`

### Pages secrets

Set these once in Cloudflare (not in source):

```bash
npx wrangler pages secret put PARCHMENT_BASE_URL --project-name bbpp
npx wrangler pages secret put PARCHMENT_API_KEY --project-name bbpp
npx wrangler pages secret put TURNSTILE_SECRET_KEY --project-name bbpp
```

| Secret | Value |
|---|---|
| `PARCHMENT_BASE_URL` | `https://parchment-worker-bbpp.danrevel.workers.dev` |
| `PARCHMENT_API_KEY` | BBPP_KEY (must match `ISSUE_API_KEY` on parchment-worker-bbpp) |
| `TURNSTILE_SECRET_KEY` | From Cloudflare Turnstile dashboard for site key `0x4AAAAAADCnY8bWziMBe6XP` |

## Turnstile

The Turnstile site key `0x4AAAAAADCnY8bWziMBe6XP` is configured for `bigbeautifulpeaceprize.com` in the [Cloudflare Turnstile dashboard](https://dash.cloudflare.com/?to=/:account/turnstile).

The widget is embedded in `src/index.html` with `data-callback="onTurnstileSuccess"`, which enables the **Award the Prize** button when the challenge passes.

## Security

- The `/parchment/issue` endpoint is protected by two layers:
  1. **Turnstile** — the Pages Function rejects any POST without a valid Turnstile token (prevents bots)
  2. **API key** — the parchment worker rejects any request without a valid `Authorization: Bearer` header (prevents direct calls to the worker that bypass Turnstile)
- The parchment worker is only reachable at its `workers.dev` URL — no custom domain route — so all public traffic flows through the Pages Function
