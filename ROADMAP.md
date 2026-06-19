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

- **Mobile actions (touch popover)** — **M** — finishes frontend **#6**
  The node popover (join / vote / rename / leave / switch / move) is driven by
  hover (`mouseenter`), which real touch devices never fire. So on a phone you
  can navigate and drill, but you **cannot act**. Wire the popover to open on a
  real tap (the mobile e2e test for it is currently skipped — un-skip it).
  - File: `anthill-frontend/src/Graph/GraphSVG/GraphSVG.tsx`, `GraphFlow.tsx`
  - Test: `anthill-frontend/e2e/mobile.spec.ts` (`test.skip` → enable)

---

## Next — make it shippable

- **Public testnet deploy (Sepolia)** — **M** — incl. contracts **#6**
  Deploy + seed the contract on a public testnet, **verify it on the block
  explorer** (#6), and set the real address in the frontend/backend (currently a
  `0x000…0` placeholder), so there's a live, shareable demo instead of
  localhost-only.
  - `anthill-frontend/src/main.tsx` (`anthillContractAddress` TODO), backend config.

- **CI & backend testing** — **M** — incl. backend **#2**, contracts **#11** (testing note)
  The e2e + on-chain integration tests need a running anvil + backend, so there's
  no hermetic CI yet. Add a workflow that boots anvil, deploys/seeds, runs the
  backend, then runs `vitest` + Playwright. Backend #2's idea: test the backend's
  graph-derivation by running the contract tests as a script and comparing the
  backend's read-old + apply-events against read-new. Also wire up frontend
  auto-build on GitHub (manual `firebase deploy` today; backend builds on Heroku
  from `main`).

- **Cleanup** — **S** — relates to contracts **#10**
  - Remove the now-unreachable 3-row reputation renderer (`dagLayout` /
    `SelectSentRecDagVotes`) — superseded by the on-tree Reputation view.
  - Reputation view: voters render at their deep tree positions, leaving
    blue-edge gaps; optionally pull them up beneath the focus for a tighter read.
  - Contract-interface / data-encoding cleanup (#10) — the issue notes this is
    only worth doing once others start contributing.

---

## Later — depth

- **Reputation recalculation** — **L** — backend **#6**, contracts TODOs
  There's no efficient on-chain algorithm: the plan (per #6) is for the backend to
  compute a valid node ordering and pass it to the contract, which then calculates
  reputation one node at a time (each node's predecessors are already done or
  don't exist). Also the script TODOs in `anthill-contracts/script/AnthillLegacy.s.sol`
  (single-node + all-nodes recalculation).

- **Proof-of-Humanity beta** — **L** — contracts **#12**
  Integrate freeze-person + remove-by-leader, and investigate an on-chain voting
  override (is easy on-chain voting feasible?). A governance/anti-sybil direction.

- **Scale / large-graph navigation** — **M/L** — frontend **#7**
  Stress-test large graphs (React Flow virtualises above ~300 nodes; there's a
  `/demo` generator, but layout / history replay / the backend's event-derived
  graph are untested at size). #7 also proposes scroll up/down controls and
  keeping the clickedNode away from the viewport edge for graphs too big to fit —
  partly covered now by pan/zoom + the 3-level collapse.

- **Product polish** — open-ended
  Clearer join/vote flows, surfacing reputation magnitude (e.g. node size by rep),
  richer history labels, etc. (frontend **#5** "make it pretty" — largely done.)

---

## Recently done (context)

These close several GitHub issues — listed so they can be closed out:

- Three-view toggle: **Tree** (structure) / **+ Votes** (focus's outgoing votes,
  green) / **Reputation** (focus's incoming votes, blue) — all on one tree layout.
- History scrubber scoped to the focused node **and** the active view, with
  prev/next/play controls; the overlay animates during playback.
  → closes backend **#4**, contracts **#11** (display how the graph changed).
- Mobile-responsive layout + LAN access for on-device testing → progresses
  frontend **#6**; three-levels-by-default → contracts **#7** (partial).
- In-app tutorial (6-step walkthrough) → closes frontend **#2**.
- Backend on Hono + WebSockets → closes backend **#3**, contracts **#4**.
- UI polish (header/footer, segmented toggle, weighted/coloured edges) →
  largely addresses frontend **#5**.
- Locked-layout graph navigation (hover-drill, click-to-focus that stays open),
  no-jump view switching.
- Playwright e2e + Vitest unit/integration regression suites.
- Toolchain modernised (Foundry-only, viem/Hono/tsx, Biome, pnpm, Vite 8).
