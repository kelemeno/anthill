# Anthill — Roadmap

Anthill is a liquid-democracy–inspired reputation system (see [README](./README.md)):
people sit in a binary tree, and reputation comes from **value (dag) votes**, not
tree position. This is currently a **demo app** that runs locally against an
[anvil](https://book.getfoundry.sh/anvil/) node.

This roadmap is organised by **phase**, not by calendar date. Items carry a rough
effort estimate — **S** (hours), **M** (a day or two), **L** (multi-day). Effort
is a guess, not a commitment.

Repo layout: a superproject with three submodules — `anthill-contracts`
(Foundry), `anthill-backend` (Hono + viem, port 5001), `anthill-frontend`
(React + Vite + React Flow, port 5173). Working code lives on each submodule's
`develop` branch.

---

## Now — finish what's started

The visualisation is in good shape; the gap is making it usable end-to-end,
especially on a phone.

- **Mobile actions (touch popover)** — **M**
  The node popover (join / vote / rename / leave / switch / move) is driven by
  hover (`mouseenter`), which real touch devices never fire. So on a phone you
  can navigate and drill, but you **cannot act**. Wire the popover to open on a
  real tap (the mobile e2e test for it is currently skipped — un-skip it).
  - File: `anthill-frontend/src/Graph/GraphSVG/GraphSVG.tsx`, `GraphFlow.tsx`
  - Test: `anthill-frontend/e2e/mobile.spec.ts` (`test.skip` → enable)

---

## Next — make it shippable

- **Public testnet deploy (Sepolia)** — **M**
  Deploy + seed the contract on a public testnet and set the real address in the
  frontend/backend (currently a `0x000…0` placeholder), so there's a live,
  shareable demo instead of localhost-only.
  - `anthill-frontend/src/main.tsx` (`anthillContractAddress` TODO), backend config.

- **CI** — **M**
  The e2e + on-chain integration tests need a running anvil + backend, so there's
  no hermetic CI yet. Add a workflow that boots anvil, deploys/seeds, runs the
  backend, then runs `vitest` + Playwright. Also: the frontend doesn't auto-build
  on GitHub (manual `firebase deploy`); wire that up (backend already builds on
  Heroku from `main`).

- **Cleanup** — **S**
  - Remove the now-unreachable 3-row reputation renderer (`dagLayout` /
    `SelectSentRecDagVotes`) — superseded by the on-tree Reputation view.
  - Reputation view: voters render at their deep tree positions, leaving
    blue-edge gaps; optionally pull them up beneath the focus for a tighter read.

---

## Later — depth

- **Reputation recalculation** — **L**
  Contract-side TODOs (`anthill-contracts/script/AnthillLegacy.s.sol`): a method
  to recalculate reputation for a single node, and to trigger recalculation for
  all nodes. Needed for the reputation numbers to stay correct as votes change.

- **Scale** — **M/L**
  Stress-test large graphs. React Flow virtualisation kicks in above ~300 nodes
  and there's a `/demo` synthetic generator, but real large-graph performance
  (layout, history replay, the backend's event-derived graph) is untested at size.

- **Product polish** — open-ended
  Clearer join/vote flows, surfacing reputation magnitude (e.g. node size by rep),
  richer history labels, etc.

---

## Recently done (context)

- Three-view toggle: **Tree** (structure) / **+ Votes** (focus's outgoing votes,
  green) / **Reputation** (focus's incoming votes, blue) — all on one tree layout.
- History scrubber scoped to the focused node **and** the active view, with
  prev/next/play controls; the overlay animates during playback.
- Mobile-responsive layout; LAN access for on-device testing; in-app tutorial.
- Locked-layout graph navigation (hover-drill, click-to-focus that stays open),
  no-jump view switching, weighted/coloured vote edges.
- Playwright e2e + Vitest unit/integration regression suites.
- Toolchain modernised (Foundry-only, viem/Hono/tsx, Biome, pnpm, Vite 8).
