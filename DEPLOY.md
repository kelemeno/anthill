# Deploying the read-only demo (anthilldao.dev)

A browse-only showcase: **Firebase Hosting** serves the frontend, **Heroku** runs
the backend in snapshot mode (a frozen graph, no chain, no keys), custom domain
**anthilldao.dev**. No contract is deployed.

```
Firebase (anthilldao.dev) ──► frontend  (VITE_READ_ONLY=true: graph browsing,
        │  REST + WS                      wallet + actions hidden)
        ▼
Heroku ──► backend  (SNAPSHOT_MODE=true: serves anthillSnapshot.json, no chain)
```

## 1. Snapshot data (already committed)

`anthill-backend/anthillSnapshot.json` is the frozen graph + history the demo
serves. Regenerate it any time (pure JS, no anvil/foundry):

```bash
cd anthill-backend
npx tsx scripts/makeSnapshot.ts   # writes ./anthillSnapshot.json
git add anthillSnapshot.json && git commit -m "Refresh demo snapshot"
```

## 2. Backend → Heroku (auto-builds from `main`)

Snapshot mode is baked into the `Procfile` (`web: SNAPSHOT_MODE=true node
build/App.js`), so **no config var is needed**. Deploy = put the code on `main`:

```bash
cd anthill-backend && git checkout main && git merge develop && git push origin main
```

Sanity check: `curl https://<heroku-app>.herokuapp.com/rootId` → `{"id":"0x…0002"}`.
(To make the Heroku backend read a real chain later, revert the Procfile and set
`RPC_URL` / `CONTRACT_ADDRESS`.)

## 3. Frontend → Firebase (manual `firebase deploy`)

```bash
cd anthill-frontend
VITE_READ_ONLY=true npm run build          # default backend: anthill-db.herokuapp.com
# if the Heroku app differs, override the host:
# VITE_READ_ONLY=true VITE_BACKEND_URL=https://<app>.herokuapp.com/ \
#   VITE_WS_URL=wss://<app>.herokuapp.com/ npm run build
npx firebase deploy --only hosting          # project anthill-147b0
```

Before DNS propagates, test at the default URL: **https://anthill-147b0.web.app**.

## 4. Domain → Firebase + DNS

- Firebase console → **Hosting → Add custom domain → `anthilldao.dev`**.
- Add the **TXT** (verification) + **two A records** it shows, at your DNS host.
- `.dev` is HTTPS-only; Firebase auto-provisions the certificate.

## Flags / toggles

- `VITE_READ_ONLY` (frontend build): hides wallet + all on-chain actions.
- `VITE_BACKEND_URL` / `VITE_WS_URL` (frontend build): backend host overrides.
- `SNAPSHOT_MODE` (backend env): serve the frozen snapshot, no chain.
- `CAPTURE_SNAPSHOT` (backend env): one-off live load → write the snapshot (needs a chain).
