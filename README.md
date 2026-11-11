# anthill

## intro

Disclaimer: Demo app.

Anthill is a liquid democracy inspired reputation system, people are organised into a binary tree, and everyone is assigned a reputation (number).

The position does not directly affect the reputation. You get reputation by collecting secondary value votes. You can give anyone value votes above you in the tree, within a certain proximity. Similarly, you can receive value votes form anyone under you, in a certain proximity.

To emphasise: value votes, and not position determines your reputation.

Also you can leave the tree, or move to unoccupied spots in the tree.

Finally: if you have higher reputation than your parent in the binary tree, you can change positions with them, thus climbing the tree.

This is the repo for the smart contracts.

## development

### off-chain (default)

The backend holds the trees itself and people sign in with passkeys, so no chain or wallet is needed. `anthill-backend/src/engine.ts` is a line-by-line port of `Anthill.sol`, and an append-only log in `anthill-backend/data/` is the source of truth. A fresh data folder is seeded from `anthillSnapshot.json`; that tree becomes the invite-only `demo` community, and the Commons starts empty (its founder starts its tree: the account `COMMONS_FOUNDER` names, by default the production one; when that account doesn't exist in the data folder, as in a fresh local one, the Commons has no founder and anyone can start it). Set `SEED=empty` for no demo.

```
npm run start:dev --prefix anthill-backend   # :5001
npm start --prefix anthill-frontend          # :5173
```

What people can do besides the tree actions:

- **Communities**: anyone signed in can start one (`POST /communities`) and becomes the root of its tree; each has its own reputation, invites and polls, while accounts and profiles are shared. Public communities are listed and visible to all; private ones answer 404 to anyone outside them and are joined by invite only. A public one is either open (anyone can take a free spot) or invite-only. Its founder can change the name, description, visibility and join policy. The `commons` community is the public one everyone can join; `INVITE_ONLY=true` makes it invite-only until its founder changes that. The original Anthill tree, which was the Commons at first, moved to `demo` (with everything done in it) and the Commons started over empty. Since that tree only ever grew, the backend plays a short scripted story in the demo once on startup (`anthill-backend/src/demoStory.ts`): supporters switching to a rising member until they climb above their parent, people changing who they endorse, someone leaving, someone moving, newcomers joining. The replay ("Watch it grow") shows it. Everything inside a community is under `/c/<id>/` (graph, history, action, check, invites, polls); the old unscoped paths answer for the Commons.
- **Invites**: "Invite someone here" in the account menu (or "Copy invite link" on your overview) makes a one-time link (`/invite/<code>`) to the community you're in; whoever opens it joins under you, or the nearest free spot below you. `POST /admin/invites` (with `ADMIN_TOKEN`, optionally `community` and `under`) makes an invite under any node, for bootstrapping.
- **Polls** weighted by reputation: an option's weight is the total reputation of its voters; closing a poll freezes the result. Only current members' votes count (someone who leaves and comes back votes again). The creator closes a poll, or the founder once the creator has left. Each member can have 5 open polls in a community, a community 30 open and 500 in all.
- **Profiles**: a name, a short bio and a link, shown on your node in every community (renaming changes your name in all of them). An account has one name: joining a tree uses it whatever name the client sends, and `changeName` as an action renames the account everywhere, like `POST /name`. Names lose control, zero-width and bidi characters, and must be 1-40 characters.
- **Passkeys and recovery**: you can add passkeys for other devices; each account has a recovery code (shown once, replaced when used) that registers a new passkey if all are lost. Recovering removes the account's other passkeys (the lost devices) and ends all its sessions. Adding a passkey or replacing the recovery code needs a sign-in from the last 10 minutes (otherwise a 401 with `code: "reauth"`: sign in with a passkey again). `POST /auth/sign-out-everywhere` ends every session of the account, the caller's too.
- **Sign-up limits**: passkeys cost nothing to make in software, so nothing stops one person from making many accounts. The backend only slows that down (20 new accounts per IP and hour, 60 passkey ceremonies per IP and minute); invite-only communities are the real guard against fake members inflating reputation and poll weight.
- Greyed-out actions say why they aren't possible (the backend dry-runs them).

The website (`anthill-frontend`, see its README) lists your communities at `/`; inside one (`/c/<id>`) it's organised around people: You (your standing and next steps), Explore (the tree), Polls, People, and a page per person.

Backups: `anthill-backend/scripts/backupLog.sh` downloads the log via `GET /admin/log` and keeps 30 daily copies in `~/anthill-backups`; put the backend's `ADMIN_TOKEN` in `~/.anthill-admin-token`.

To try the signed-in UI locally without a passkey prompt, `npx tsx scripts/devSession.ts "Name"` (in `anthill-backend`) creates a test account on the local backend and prints a session to put in the browser's localStorage as `anthill.session`.

Passkeys only work on localhost or https. For a deployment set `RP_ID` (the frontend's domain, e.g. `anthilldao.dev`) and `ORIGINS` (e.g. `https://anthilldao.dev`) on the backend, and give it a persistent disk for `DATA_DIR`. Rate limits key on the client's IP: the right-most `X-Forwarded-For` entry, the one Render's proxy adds (`TRUSTED_PROXY_HOPS`, default 1, says how many proxies in front add one; 0 uses the socket address). `TRUST_CLOUDFLARE=true` uses `CF-Connecting-IP` instead, only for a deployment behind your own Cloudflare. The backend logs once how many `X-Forwarded-For` entries the first request had, to check this. Request bodies are limited to 64 KB.

The WebSocket (`/`) only sends: `update:<key>` after a change in a community, where the key is a public community's id or, for a private one, the opaque `updateKey` from its `/graph` (so outsiders can't tell which private community changed), and a bare `update` on startup. It allows 20 sockets per IP, closes sockets that send frames over 1 KB, and pings every 30 s, closing those that didn't answer.

Checks:

- `npx tsx scripts/smokeOffchain.ts` (in `anthill-backend`, against a backend started with a fresh `DATA_DIR` on port 5098): registers a software passkey, signs in, joins, votes, renames.
- `npx tsx scripts/startupChecks.ts` (in `anthill-backend`, no server needed): restarts in-process on throwaway data folders to check that requests wait for the log to load, that a torn last log line doesn't stop startup, that a failed log write changes nothing, and the one-time fixes.
- `npx tsx scripts/authChecks.ts` (in `anthill-backend`, no server needed): account changes need a recent sign-in, tokens from before session versions keep working until "sign out everywhere", and sign-outs and recoveries survive a restart.
- `npx tsx scripts/treeRules.ts` (in `anthill-backend`, no server needed): the rules the server adds around the engine (nobody under the engine's scratch node, malformed args refused per item, who can start a founderless tree), reputation staying cheap on deep endorsement chains, and the rollback-safe form of closed communities in the log.
- `npx tsx scripts/diffEngine.ts` (needs `anvil --port 8546 --disable-code-size-limit` and `forge build src/Anthill.sol`): runs random actions against both the contract and the engine and compares every revert and the full storage.

`moveTreeVote(voter, voter)` used to succeed and detach the voter into a self-loop; it now reverts with `A mTV 5`, in both the contract and the engine. Contracts deployed before that fix still have the bug.

Like the contract, the engine lets anyone join or move under its scratch node `0x…99999999` (left with a parent after any move), a branch the root doesn't reach. The server refuses that and the engine's other sentinels as recipients, and treats anyone it finds there as outside the tree; startup logs them as "detached from the root".

Rolling the backend back: code from before the `closed` join policy (backend `b3c2000` and older) reads any policy but `invite` as open. Closed communities are now logged as `invite` plus a `closed` flag (the `closed-rollback-safe` fix rewrites the demo's settings that way on its first start), so rolled back, they are invite only (members can invite again), not open. Until a build with that fix has started once, a rollback to `b3c2000` or older opens the demo to anyone.

### on-chain (CHAIN_MODE=true on the backend)

The frontend no longer connects wallets, so in this mode it can only browse the graph the backend reads from the contract.

To build and test use forge. For local development with backend and frontend deploy the smart contract on anvil with:

anvil --chain-id 1337

forge script script/Anthill.s.sol:SmallScript --broadcast --rpc-url http://localhost:8545

After this the backend can be launched with:

npm run start:dev

And the frontend can be launched with:

npm start

If using metamask you have to clear metamask activity between different anvil sessions, as nonce and other things might change. Do this is setting-> advanced -> reset account

### deployment

smart contract need to be deployed, and the address has to be set in the frontend, backend
heroku builds backend based on github's main branch
firebase frontend does not build on github yet, I should do that. But currently npm run build and npx firebase deploy deploys the frontend.

### zksync specific development

Warning! Adding to AnthillInner address to foundry.toml will make normal forge tests break.

https://github.com/matter-labs/foundry-zksync/tree/main

#### On dev updated 2024/05/10

Anthill original:

- install forge
- start zksync dockerized local setup (check that this works). If only testing contracts local node is enough.
- remove compiled foler, AnthillInner address from foundry.toml
- to detect missing libraries `../../zksync/fzksync/foundry-zksync/target/release/forge build  --zksync  --contracts-to-compile src/AnthillInner.sol --avoid-contracts "script/Anthill.s.sol:AnthillScript1" `
- To deploy missing libraries: `../../zksync/fzksync/foundry-zksync/target/release/forge create --zksync --deploy-missing-libraries --private-key 0x7726827caac94a7f9e1b160f7ea819f172f7b6f9d2a97f992c38edeab82d4110 --rpc-url http://localhost:3050 --chain 270 --verifier-url localhost:3010`
- To compile everything: `../../zksync/fzksync/foundry-zksync/target/release/forge build  --zksync `
- To run tests: `../../zksync/fzksync/foundry-zksync/target/release/forge test --zksync --rpc-url http://localhost:3050 --chain 270`
- To run scripts: `../../zksync/fzksync/foundry-zksync/target/release/forge script --zksync --slow script/Anthill.s.sol:SmallScript --broadcast --rpc-url http://localhost:3050 --chain 270`

Anthill:

To compile everything the old Anthill has to be compiled, i.e. follow the steps above.

- To run scripts: `../../zksync/fzksync/foundry-zksync/target/release/forge script --zksync --slow script/Anthill.s.sol:SmallScript --broadcast --rpc-url http://localhost:3050 --chain 270`

- Testnet deployment, if for first time:
  `../../zksync/fzksync/foundry-zksync/target/release/forge script --zksync --slow script/Anthill.s.sol:JustDeploy --broadcast --rpc-url https://sepolia.era.zksync.dev --chain 300 --private-key`

- To debug a single tx that happened on testnet/mainnet:

  - start era-test node before the tx
  - save data:
    `forge script  --zksync --slow script/Anthill.s.sol:ReadAndSave --rpc-url localhost:8011`
  -

- Testnet migration to new contract:

Run replay test: - launch era-test node:
`era_test_node fork https://sepolia.era.zksync.dev --fork-at 3074373 `

    - Testnet loadblock:
    `forge script --zksync --slow script/Anthill.s.sol:ReadAndSave --broadcast --rpc-url localhost:8011 --chain 300`

    - save all logs:
    `echo '{ "logs" :' > script-out/logs.json && cast logs -j --rpc-url https://sepolia.era.zksync.dev --address 0xe42923350EF3a534f84bb101453D9B442d42Bf0c --from-block 3074373 >> script-out/logs.json && echo '}' >> script-out/logs.json && yarn encodeLogs`

    - run replay test:
    `forge test --match-test  test_deployAndReexecute -vvvv --ffi`
