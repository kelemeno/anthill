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

- ✅ **Mobile actions (touch popover)** — **done** — completes frontend **#6**
  On touch a tap now opens the node popover (info + join/vote/rename/leave/
  switch/move) via a real pointerup handler; tap empty space to dismiss, drill
  via the +N badge. So a phone can finally *act*, not just browse.

The next major direction is **Proof-of-Humanity + on-chain voting** (see the
dedicated sections below) — these are contract changes and need design sign-off
before any Solidity is written.

---

## Next — make it shippable

- **Public testnet deploy (Sepolia)** — **M** — incl. contracts **#6**
  Deploy + seed the contract on a public testnet, **verify it on the block
  explorer** (#6), and set the real address in the frontend/backend (currently a
  `0x000…0` placeholder), so there's a live, shareable demo instead of
  localhost-only.
  - `anthill-frontend/src/main.tsx` (`anthillContractAddress` TODO), backend config.

- **SSO / social login (no browser wallet)** — **M**
  Let people sign in with email or a social account (Google/Apple/etc.) instead
  of needing MetaMask or a browser wallet — the wallet-install step is the
  biggest onboarding drop-off for non-crypto users. We already use Reown AppKit,
  which supports **email + social login backed by an embedded wallet**, so this
  is mostly enabling/configuring AppKit's auth (set up a project with the social
  providers, turn on the embedded-wallet/email connectors in `createAppKit`)
  rather than new contract work — the embedded wallet signs transactions like any
  other account. Gas for those accounts still needs a story (faucet on testnet,
  or a paymaster / sponsored transactions later).
  - `anthill-frontend/src/main.tsx` (AppKit config), provider/project setup.

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

- **Proof-of-Humanity + on-chain voting** — **L** — contracts **#12**
  Now a first-class direction with its own sections below.

- **Scale / large-graph navigation** — **M/L** — frontend **#7**
  Stress-test large graphs (React Flow virtualises above ~300 nodes; there's a
  `/demo` generator, but layout / history replay / the backend's event-derived
  graph are untested at size). #7 also proposes scroll up/down controls and
  keeping the clickedNode away from the viewport edge for graphs too big to fit —
  partly covered now by pan/zoom + the 3-level collapse.

- **Graph interaction polish** — **M** — frontend
  Three follow-ups on the drag/layout/animation feel:
  - **Natural drag resistance.** The drag-wiggle is clamped to a small box, so a
    node hits the cap and stops abruptly. Replace the hard clamp with progressive
    resistance — the further you pull, the more it resists (asymptotic / rubber-
    band easing) — so it decelerates smoothly instead of slamming into a wall.
    (`AnthillNodeView` pointer handlers in `anthill-frontend/src/Graph/GraphSVG/GraphFlow.tsx`.)
  - **Keep the focus on a central vertical line.** The layout is fully fixed
    (nice + stable), but drilling deep side branches drifts far horizontally.
    Bias the layout so a node's two children open *towards the centre* (mirror
    the subtree around the focus's vertical axis), keeping the active path roughly
    vertical instead of wandering sideways. (Layout in `computeTreePositions` /
    the d3-dag sugiyama setup; may need a custom coord assignment.)
  - **Grow nodes out of their parent (big one).** When children appear (peek /
    expand), animate them *growing/sliding out of the parent node's position*
    rather than fading in where they land — and reverse on collapse. Likely a
    per-node mount animation that starts at the parent's screen position/scale
    and eases to its own (needs the parent position at mount; React Flow node
    mount + a transform transition, or a layout-animation lib).

- **Product polish** — open-ended
  Clearer join/vote flows, surfacing reputation magnitude (e.g. node size by rep),
  richer history labels, etc. (frontend **#5** "make it pretty" — largely done.)

---

## Proof-of-Humanity (PoH) — contracts #12

**Goal:** an anti-sybil / governance layer so the tree reflects real, unique
people, and bad actors can be frozen or removed. Today the contract is purely
`onlyVoter` (you can only act on your own node) — there is no leader, admin, or
freeze concept, so this is all new mechanics in `anthill-contracts/src/Anthill.sol`.

**Proposed pieces**
- **Freeze a person** — a `frozen[address]` flag. A frozen node keeps its tree
  position but is blocked from acting (`addDagVote`, `moveTreeVote`,
  `switchPositionWithParent`, …) and/or excluded from reputation. Emits
  `Frozen` / `Unfrozen` events.
- **Remove by leader** — a leader can evict a node from the tree (reusing
  `handleLeavingVoterBranch` / `leaveTree` internals). Emits an event.
- **Access control** — define who a "leader" is (see open questions) and add an
  `onlyLeaderOf(voter)` modifier.

**Open questions (need answers before Solidity)**
1. **Who is a "leader"?** The node's tree parent? Any ancestor within the
   reputation proximity? The root? A separately-appointed admin/multisig?
2. **What does "freeze" disable** — just actions, or also exclude the node from
   reputation totals? Is there an unfreeze / appeal path?
3. **Removal effects** — does removing re-parent the children (pull-up, like
   `leaveTree`), or remove the whole subtree?
4. **Abuse guard** — what stops a leader freezing/removing arbitrarily (time
   lock, vote, reciprocity)?

**Sub-tasks (once design is set):** contract state + functions + modifier +
events; tests (`anthill-contracts/test`); backend to surface `frozen` and emit
the new events into the graph; frontend to show frozen state + leader actions in
the popover.

---

## On-chain voting — contracts #12

**Goal:** let the system make collective decisions on-chain, weighted by
reputation — and explore the "voting override" the issue mentions. The issue is
explicitly a feasibility question: *"is it possible to easily implement on-chain
voting?"*

**Proposed pieces**
- **Proposal + ballot model** — create a proposal, cast votes, tally. Vote
  weight = a voter's reputation (we already have `calculateReputation`).
- **What's votable?** Candidate first targets: freezing/removing a person
  (ties into PoH above), or protocol parameters (e.g. `MAX_REL_ROOT_DEPTH`).
- **"Override"** — clarify intent: override an individual's dag votes by
  collective decision, or a general governance override of an action?

**Open questions (need answers before Solidity)**
1. **What gets voted on first** — PoH actions (freeze/remove) or parameters?
2. **Weighting + quorum** — reputation-weighted; what threshold/quorum, and over
   what time window? (Reputation is expensive to compute on-chain — likely use
   the cached `calculatedReputationForEpoch`.)
3. **What does "override" mean** concretely?
4. **Cost** — on-chain tallies over many voters are gas-heavy; may need the same
   backend-orders-nodes trick as reputation recalculation (backend #6).

**Sub-tasks (once design is set):** proposal/vote structs + functions + events;
reputation-as-weight integration; tests; backend + frontend surfaces.

> These two are intertwined (voting is the natural governance mechanism for PoH
> freeze/remove). Recommend designing them together. **Next step: answer the open
> questions above, then I'll write the contract + tests.**

---

## Recently done (context)

These close several GitHub issues — listed so they can be closed out:

- Three-view toggle: **Tree** (structure) / **+ Votes** (focus's outgoing votes,
  green) / **Reputation** (focus's incoming votes, blue) — all on one tree layout.
- History scrubber scoped to the focused node **and** the active view, with
  prev/next/play controls; the overlay animates during playback.
  → closes backend **#4**, contracts **#11** (display how the graph changed).
- Mobile-responsive layout, LAN access for on-device testing, and tap-to-open
  node actions → closes frontend **#6**; three-levels-by-default → contracts
  **#7** (partial).
- In-app tutorial (6-step walkthrough) → closes frontend **#2**.
- Backend on Hono + WebSockets → closes backend **#3**, contracts **#4**.
- UI polish (header/footer, segmented toggle, weighted/coloured edges) →
  largely addresses frontend **#5**.
- Locked-layout graph navigation (hover-drill, click-to-focus that stays open),
  no-jump view switching.
- Playwright e2e + Vitest unit/integration regression suites.
- Toolchain modernised (Foundry-only, viem/Hono/tsx, Biome, pnpm, Vite 8).
