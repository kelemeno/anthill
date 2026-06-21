# Deploying the read-only demo (anthilldao.dev)

A browse-only showcase. **Vercel** serves the frontend, **Render** runs the
backend in snapshot mode (a frozen graph, no chain, no keys), custom domain
**anthilldao.dev**. No contract is deployed. Both auto-deploy from `main`.

```
Vercel (anthilldao.dev) ──► frontend  (read-only static build via .env.production:
        │  REST + WS                    wallet + actions hidden)
        ▼
Render ──► backend  (snapshot mode via render.yaml: serves anthillSnapshot.json,
                     no chain)
```

## 1. Snapshot data (already committed)

`anthill-backend/anthillSnapshot.json` is the frozen graph + history the demo
serves. Regenerate any time (pure JS, no chain/foundry):

```bash
cd anthill-backend && npx tsx scripts/makeSnapshot.ts
git add anthillSnapshot.json && git commit -m "Refresh demo snapshot" && git push
```

## 2. Backend → Render (auto-deploys from `main`)

`anthill-backend/render.yaml` defines a free Node web service with
`SNAPSHOT_MODE=true` baked in — **no config from you.**

- Render dashboard → **New → Blueprint** → connect the `anthill-backend` repo
  (branch `main`). It reads `render.yaml` and deploys.
- Note the service URL, e.g. `https://anthill-backend.onrender.com`.
- Sanity check: `…/rootId` → `{"id":"0x…0002"}`.

⚠️ Free Render services **spin down after ~15 min idle** — the first hit after
that takes ~30–60s to wake. Fine for a demo; upgrade the instance to keep it warm.

## 3. Frontend → Vercel (auto-deploys from `main`)

`.env.production` already sets `VITE_READ_ONLY=true` and points at the Render URL,
and `vercel.json` adds the SPA rewrite.

- Vercel → **Add New → Project** → import the `anthill-frontend` repo. It detects
  Vite and builds (output `dist/`).
- **Only if** your Render URL differs from `anthill-backend.onrender.com`: set
  `VITE_BACKEND_URL` (`https://<svc>.onrender.com/`) and `VITE_WS_URL`
  (`wss://<svc>.onrender.com/`) in the Vercel project's Environment Variables
  (these override `.env.production`), then redeploy.
- Test at the Vercel preview URL before DNS: should load with "Satoshi → Noor, Lin".

## 4. Domain → Vercel + DNS

- Vercel project → **Settings → Domains → Add `anthilldao.dev`**.
- Add the record Vercel shows (an **A record** to `76.76.21.21`, or a CNAME) at
  your DNS host. `.dev` is HTTPS-only; Vercel auto-provisions the certificate.

## Flags / toggles

- `VITE_READ_ONLY` (frontend build): hides wallet + all on-chain actions.
- `VITE_BACKEND_URL` / `VITE_WS_URL` (frontend build): backend host overrides.
- `SNAPSHOT_MODE` (backend env, set in render.yaml): serve the frozen snapshot, no chain.
- `CAPTURE_SNAPSHOT` (backend env): one-off live load → write the snapshot (needs a chain).
