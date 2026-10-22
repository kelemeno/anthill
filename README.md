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

The backend holds the tree itself and people sign in with passkeys, so no chain or wallet is needed. `anthill-backend/src/engine.ts` is a line-by-line port of `Anthill.sol`, and an append-only log in `anthill-backend/data/` is the source of truth. A fresh data folder is seeded from `anthillSnapshot.json`; set `SEED=empty` to start with an empty tree.

```
npm run start:dev --prefix anthill-backend   # :5001
npm start --prefix anthill-frontend          # :5173
```

What people can do besides the tree actions:

- **Invites**: "Copy invite link" on your own node (or Invite someone in the account menu) makes a one-time link; whoever opens it joins under you, or the nearest free spot below you. `INVITE_ONLY=true` on the backend makes an invite required to join; until then anyone can still join at any free spot. `POST /admin/invites` (with `ADMIN_TOKEN`) makes an invite under any node, for bootstrapping.
- **Polls** weighted by reputation: an option's weight is the total reputation of its voters; closing a poll freezes the result.
- **Profiles**: a short bio and a link, shown on your node.
- **Passkeys and recovery**: you can add passkeys for other devices; each account has a recovery code (shown once, replaced when used) that registers a new passkey if all are lost.
- Greyed-out buttons say why an action isn't possible (the backend dry-runs it).

Backups: `anthill-backend/scripts/backupLog.sh` downloads the log via `GET /admin/log` and keeps 30 daily copies in `~/anthill-backups`; put the backend's `ADMIN_TOKEN` in `~/.anthill-admin-token`.

To try the signed-in UI locally without a passkey prompt, `npx tsx scripts/devSession.ts "Name"` (in `anthill-backend`) creates a test account on the local backend and prints a session to put in the browser's localStorage as `anthill.session`.

Passkeys only work on localhost or https. For a deployment set `RP_ID` (the frontend's domain, e.g. `anthilldao.dev`) and `ORIGINS` (e.g. `https://anthilldao.dev`) on the backend, and give it a persistent disk for `DATA_DIR`.

Checks:

- `npx tsx scripts/smokeOffchain.ts` (in `anthill-backend`, against a backend started with a fresh `DATA_DIR` on port 5098): registers a software passkey, signs in, joins, votes, renames.
- `npx tsx scripts/diffEngine.ts` (needs `anvil --port 8546 --disable-code-size-limit` and `forge build src/Anthill.sol`): runs random actions against both the contract and the engine and compares every revert and the full storage.

`moveTreeVote(voter, voter)` used to succeed and detach the voter into a self-loop; it now reverts with `A mTV 5`, in both the contract and the engine. Contracts deployed before that fix still have the bug.

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
