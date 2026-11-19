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

The backend holds the trees itself and people sign in with passkeys or Google, so no chain or wallet is needed. `anthill-backend/src/engine.ts` is a line-by-line port of `Anthill.sol`, and an append-only log in `anthill-backend/data/` is the source of truth. A fresh data folder is seeded from `anthillSnapshot.json`; that tree becomes the invite-only `demo` community, and the Commons starts empty (its founder starts its tree: the account `COMMONS_FOUNDER` names, by default the production one; when that account doesn't exist in the data folder, as in a fresh local one, the Commons has no founder and anyone can start it). Set `SEED=empty` for no demo.

```
npm run start:dev --prefix anthill-backend   # :5001
npm start --prefix anthill-frontend          # :5173
```

What people can do besides the tree actions:

- **Communities**: anyone signed in can start one (`POST /communities`) and becomes the root of its tree; each has its own reputation, invites and polls, while accounts and profiles are shared. Public communities are listed and visible to all; private ones answer 404 to anyone outside them and are joined by invite only. A public one is either open (anyone can take a free spot) or invite-only. Its founder can change the name, description, visibility and join policy. The `commons` community is the public one everyone can join; `INVITE_ONLY=true` makes it invite-only until its founder changes that. The original Anthill tree, which was the Commons at first, moved to `demo` (with everything done in it) and the Commons started over empty. Since that tree only ever grew, the backend plays a short scripted story in the demo once on startup (`anthill-backend/src/demoStory.ts`): supporters switching to a rising member until they climb above their parent, people changing who they endorse, someone leaving, someone moving, newcomers joining. The replay ("Watch it grow") shows it. Everything inside a community is under `/c/<id>/` (graph, history, action, check, invites, polls); the old unscoped paths answer for the Commons.
- **Sites**: each community shapes how it shows on Anthill (`anthill-backend/src/site.ts`): which of Explore, Polls, People and Agents it shows (You, its home, is always there; Agents only where the operator gave it an agent commons, and on by default there), an accent colour from a fixed palette (`indigo`, the look from before, `blue`, `teal`, `green`, `amber`, `orange`, `rose` or `violet`, by name: the colours, light and dark, are the frontend's) and an About in Markdown (up to 10,000 characters, counted after NFC normalisation as it's kept; control characters are dropped, line breaks kept; one with nothing visible, only whitespace or zero-width characters, is no About; the frontend renders it without any HTML or images). Pages are presentation: a page that's off is hidden, nothing in it is deleted and no rule changes. Whoever may change the community's settings (the founder, or the root of a founderless one) replaces it whole with `POST /c/<id>/site` `{pages, accent, about}`, where `pages` is `{explore, polls, people, agents}`, each `true` or `false` (`agents` may be left out where there's no commons; a list of the pages shown is taken too; checked strictly: known pages and accents, nothing else; 10 a minute per account; answers the community summary; saved changes to a community's settings and site together are limited to 50 a day per account and 50 a day per community, refused and unchanged saves not counted, as each is kept in the log for good; `POST /c/<id>/settings` is also limited to 10 a minute per account), and `GET /c/<id>/site` → `{site: {pages, accent, about}}` is for whoever can see the community. Community summaries carry `site: {pages, accent, hasAbout}`; an invite's preview of a private community shows the defaults. Answers have `pages` as that record. It's logged as its own `communitySite` entry with who saved it and the pages it could choose then, on or off, so a page it couldn't (Agents before it had a commons) is on; a save while it can't choose Agents (its commons taken away for a while) keeps the Agents choice it made before. It's in its saver's export, and stays when they delete their account: it's the community's.
- **Invites**: "Invite someone here" in the account menu (or "Copy invite link" on your overview) makes a one-time link (`/invite/<code>`) to the community you're in; whoever opens it joins under you, or the nearest free spot below you. `POST /admin/invites` (with `ADMIN_TOKEN`, optionally `community` and `under`) makes an invite under any node, for bootstrapping.
- **Polls** weighted by reputation: an option's weight is the total reputation of its voters; closing a poll freezes the result. Only current members' votes count (someone who leaves and comes back votes again). The creator closes a poll, or the founder once the creator has left. Each member can have 5 open polls in a community, a community 30 open and 500 in all.
- **Profiles**: a name, a short bio and a link, shown on your node in every community (renaming changes your name in all of them). An account has one name: joining a tree uses it whatever name the client sends, and `changeName` as an action renames the account everywhere, like `POST /name`. Names lose control, zero-width and bidi characters, and must be 1-40 characters.
- **Passkeys and recovery**: you can add passkeys for other devices, and see and remove them (`GET /auth/passkeys`, `POST /auth/passkeys/<id>/remove`; never the account's last way to sign in). Removing one also ends the sessions it made, except the caller's, so removing a lost device's passkey signs that device out. Each account made with a passkey has a recovery code (shown once, replaced when used) that registers a new passkey if all are lost. Recovering removes the account's other passkeys (the lost devices), unlinks its Google account (the answer's `googleUnlinked` names it, to link again if it was yours) and ends all its sessions: whatever someone holding a lost device could have added goes. Adding or removing a passkey, replacing the recovery code, linking or unlinking Google and deleting the account need a sign-in from the last 10 minutes (otherwise a 401 with `code: "reauth"`: sign in again with a passkey or, for an account without one, confirm with Google).
- **Google sign-in** (`anthill-backend/src/google.ts`): OpenID Connect's authorization code flow with PKCE, the backend as a confidential client; no Google script runs on the site. Signing in with a Google account that isn't linked to an account makes one (no recovery code: Google recovers it, and one can be made later). A signed-in account can link one Google account (`intent: "link"`) and unlink it if it has a passkey left; unlinking ends the sessions Google made, except the caller's. An account without passkeys confirms it's you with `intent: "confirm"`: only its own linked Google account is accepted, nothing is ever made for another one, and the answer is a fresh session that replaces the one that asked. The flow's state is sealed (AES-GCM, keyed from the session secret) rather than kept on the server, so starts, which need no sign-in, can't crowd out anyone's flow. The email is shown only to the account holder, never in a tree, profile or public answer. It's kept outside the append-only log, in `DATA_DIR/google-emails.json` (by account id, written atomically, mode 0600), updated to the email Google reports on each sign-in or link, and erased on unlink, on recovery (which unlinks Google) and on deletion; Google's account id (`sub`) stays in the log. The file isn't part of replay: without it, or without an account in it, the account shows "a Google account" until its next Google sign-in, and a start drops lines that no longer match a linked account. Setup: in Google Cloud, create an OAuth client of type "Web application" with the authorised redirect URIs `https://anthilldao.dev/auth/google` and `http://localhost:5173/auth/google` (each frontend origin + `/auth/google`), and set `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` on the backend (`sync: false` in `render.yaml`, so in Render's dashboard). Without them Google is off: `GET /auth/methods` says so and the site only offers passkeys.
- **Sessions**: every sign-in is a session in the log, with an id in its token, how it was made and a coarse device label like "Chrome on macOS" (no IP address). `GET /auth/sessions` lists the open ones, `POST /auth/sessions/<id>/end` ends one (no recent sign-in needed), `POST /auth/sign-out` ends the caller's, and `POST /auth/sign-out-everywhere` ends every session of the account, the caller's too. Tokens from before sessions were logged keep working until sign out everywhere.
- **Export and deletion**: `GET /auth/export` downloads everything Anthill holds about the account as JSON: the account and how it signs in (with the Google email, from its file outside the log), profile, open sessions, passkeys (not their keys), its place and reputation in each community, and every log entry it wrote or that is about it (each poll vote with its poll's question and options). It reads the whole log, so it's limited to 5 per account and 10 per IP an hour, and exports run one at a time (3 waiting at most, then a 429). `POST /auth/delete` with `confirm` (the account's name as it shows) leaves every community's tree with a normal leave (in each history), then removes the passkeys, Google link, recovery code, profile and every session, renames the account "Deleted account" and leaves the communities it founded without a founder: their tree's root changes their settings from then on, an empty public one can be started by anyone unless it's closed, and an empty private one is seen by nobody, so it stays empty. Nothing more is written for a deleted account, even by requests that were signed in before it. What deletion can't erase: the log is append-only, so every entry already in it stays there for good, readable by whoever runs the server and in its backups (`GET /admin/log`): past names, actions, votes, polls and invites (shown in a community's history), bios and links, the Google account id (`sub`), passkeys' public keys, recovery codes' hashes and sessions' device labels. The Google email was never in the log: it's kept outside it and erased.
- **Sign-up limits**: passkeys cost nothing to make in software, so nothing stops one person from making many accounts. The backend only slows that down (100 new accounts per IP and hour, passkey and Google sign-ups together, and 60 POSTs under `/auth/` per IP and minute); invite-only communities are the real guard against fake members inflating reputation and poll weight.
- Greyed-out actions say why they aren't possible (the backend dry-runs them).
- **Agent commons**: a community can have an agent commons ([agent-commons](https://github.com/kelemeno/agent-commons)) as its own site, a separate app with its own ledger. Anthill signs for it with its own ed25519 key (`ANTHILL_SIGNING_KEY`, or one made once in the data folder as `commons-signing-key`; keep it, since a linked commons records the public key): `GET /commons/key` is that key, `GET /c/<id>/commons/tree` the community's tree (members under their parents, the root under null, `version` its count of history steps), and `POST /c/<id>/commons/owner` (signed in, members only, body `{publicKey}`) Anthill's attestation that a commons key is the member's. See `anthill-backend/src/commonsLink.ts`; `npx tsx scripts/commonsLinkChecks.ts` checks the format against the commons'. Members can also pair their own agents (Claude Code, Codex, ...), running in their own sandbox, without an owner key: Anthill vouches for the agent's key and relays that to the commons' coordinator (`anthill-backend/src/commonsAgents.ts`). Which communities have a commons is operator config, `COMMONS_SITES` = `{"<community id>": {"coordinator": "https://...", "site": "https://...", "repo"?: "https://..."}}` (coordinator: the commons API; site: its playground; repo: the target repository agents clone); community summaries carry `commons: {coordinator, site, repo?}` or `null`, and the routes below answer 404 elsewhere. A member asks for a one-time code, `POST /c/<id>/commons/pairings` `{modelFamily, harness}` → `{id, code, expiresAt, prompt, loopPrompt}` (public communities only; 30 minutes; at most 3 open or claimed per account and community, a 4th ending the oldest unclaimed one; `GET /c/<id>/commons/pairings` lists them, without codes; kept in memory only: a restart drops them, nothing is logged); the agent claims it with `commons-agent pair`, `POST /c/<id>/commons/pairings/claim` `{code, agentKey, modelFamily, harness, proof}` (proof: the agent key's signature over `anthill-commons/pair/v1`) → `{status: "claimed", fingerprint, expiresAt}`, and polls `POST /c/<id>/commons/pairings/claim-status` `{code}`; the member's page polls `GET /c/<id>/commons/pairings/<pairing>` → `{id, status, expiresAt, claim?: {modelFamily, harness, fingerprint}}`, compares the fingerprint and calls `/approve` (Anthill signs `anthill-commons/agent/v1` and posts it to the coordinator's `/api/anthill/agent`; the coordinator's refusals are a 400 with its message and code, and one that can't be reached or doesn't answer is a 502 that can be retried, after which Anthill checks whether the approval landed before the pairing can end) → `{agent: {keyId, modelFamily, harness}}`, or `/reject`. `GET /c/<id>/commons/agents` lists the member's agents from the coordinator, and `POST /c/<id>/commons/agents/<keyId>/revoke` (optional `{reason}`) signs `anthill-commons/agent-revoke/v1` and relays it. `GET /c/<id>/commons/agent.md` is what agents read, rendered from `anthill-backend/templates/agent.md` with the community's URLs and the kit's pinned commit, `COMMONS_KIT_COMMIT` (default: `DEFAULT_KIT_COMMIT` in `anthill-backend/src/commonsAgents.ts`). Links agents are given use `PUBLIC_URL`, else Render's `RENDER_EXTERNAL_URL`. `npx tsx scripts/agentChecks.ts` checks all of it against a fake coordinator.

The website (`anthill-frontend`, see its README) lists your communities at `/`; inside one (`/c/<id>`) it's organised around people: You (your standing and next steps), Explore (the tree), Polls, People, Agents (where the community has a commons), About (when it has one), and a page per person; a community chooses which of Explore, Polls, People and Agents it shows.

Backups: `anthill-backend/scripts/backupLog.sh` downloads the log via `GET /admin/log` and keeps 30 daily copies in `~/anthill-backups`; put the backend's `ADMIN_TOKEN` in `~/.anthill-admin-token`.

To try the signed-in UI locally without a passkey prompt, `npx tsx scripts/devSession.ts "Name"` (in `anthill-backend`) creates a test account on the local backend and prints a session to put in the browser's localStorage as `anthill.session`.

Passkeys only work on localhost or https. For a deployment set `RP_ID` (the frontend's domain, e.g. `anthilldao.dev`) and `ORIGINS` (e.g. `https://anthilldao.dev`) on the backend, and give it a persistent disk for `DATA_DIR`. `ORIGINS` must list every origin the frontend is served from, Firebase preview channels too: only those can sign in (passkeys, and Google's redirect back), and only those get CORS for anything but public reads. Rate limits key on the client's IP: the right-most `X-Forwarded-For` entry, the one Render's proxy adds (`TRUSTED_PROXY_HOPS`, default 1, says how many proxies in front add one; 0 uses the socket address). `TRUST_CLOUDFLARE=true` uses `CF-Connecting-IP` instead, only for a deployment behind your own Cloudflare. The backend logs once how many `X-Forwarded-For` entries the first request had, to check this. Request bodies are limited to 64 KB.

Headers (`anthill-backend/src/headers.ts`): every answer has security headers (`X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, HSTS, `Cross-Origin-Opener-Policy: same-origin`, `Cross-Origin-Resource-Policy: cross-origin` since the frontend is another site), and answers under `/auth/` and `/admin/` are `Cache-Control: no-store`. CORS: an origin in `ORIGINS` gets itself echoed back with `Authorization` allowed; any other site gets `*` only on a GET without an `Authorization` header (public reads, e.g. a community's app reading its public tree), never a POST or anyone's session. Before this, every origin got `*` for everything, so a page elsewhere that posted to the backend no longer can. Server-to-server callers (agent-commons) send no `Origin` and are unaffected. The frontend's own headers and Content-Security-Policy are in its `firebase.json` (see its README).

The WebSocket (`/`) only sends: `update:<key>` after a change in a community, where the key is a public community's id or, for a private one, the opaque `updateKey` from its `/graph` (so outsiders can't tell which private community changed), and a bare `update` on startup. It allows 300 sockets per IP, closes sockets that send frames over 1 KB, and pings every 30 s, closing those that didn't answer.

Checks:

- `npx tsx scripts/smokeOffchain.ts` (in `anthill-backend`, against a backend started with a fresh `DATA_DIR` on port 5098): registers a software passkey, signs in, joins, votes, renames.
- `npx tsx scripts/startupChecks.ts` (in `anthill-backend`, no server needed): restarts in-process on throwaway data folders to check that requests wait for the log to load, that a torn last log line doesn't stop startup, that a failed log write changes nothing, and the one-time fixes.
- `npx tsx scripts/authChecks.ts` (in `anthill-backend`, no server needed): account changes need a recent sign-in, tokens from before session versions keep working until "sign out everywhere", and sign-outs and recoveries survive a restart.
- `npx tsx scripts/accountChecks.ts` (in `anthill-backend`, no server needed): Google sign-in against a fake Google (refusing id_tokens with a bad signature, audience, issuer, expiry, nonce or sub, and changed, expired or replayed states; no flood of starts crowding out a flow), linking, unlinking and confirming it's you with Google (never making an account), recovery unlinking Google, the Google email kept out of the log (in its own file, erased on unlink, recovery and deletion, kept across a restart, a missing, unreadable or stale file handled), sessions listed and ended, passkeys removed (and their sessions with them), the export (nothing private of others, its limits), deleting an account (leaving every tree, nothing written for it afterwards by requests still under way), an invite link's preview of a private community, and CORS, security headers and caching.
- `npx tsx scripts/siteChecks.ts` (in `anthill-backend`, no server needed): community sites: the defaults, strict validation (nothing logged for a refused or unchanged save), who may save (the founder, the root of a founderless community, nobody else), the limit, a private community's site hidden from outsiders (in an invite's preview too), the export, a deleted founder's site staying, replay after a restart and with the operator's commons changed, and older logs replaying as they did.
- `npx tsx scripts/treeRules.ts` (in `anthill-backend`, no server needed): the rules the server adds around the engine (nobody under the engine's scratch node, malformed args refused per item, who can start a founderless tree), reputation staying cheap on deep endorsement chains, and the rollback-safe form of closed communities in the log.
- `npx tsx scripts/diffEngine.ts` (needs `anvil --port 8546 --disable-code-size-limit` and `forge build src/Anthill.sol`): runs random actions against both the contract and the engine and compares every revert and the full storage.

`moveTreeVote(voter, voter)` used to succeed and detach the voter into a self-loop; it now reverts with `A mTV 5`, in both the contract and the engine. Contracts deployed before that fix still have the bug.

Like the contract, the engine lets anyone join or move under its scratch node `0x…99999999` (left with a parent after any move), a branch the root doesn't reach. The server refuses that and the engine's other sentinels as recipients, and treats anyone it finds there as outside the tree; startup logs them as "detached from the root".

Rolling the backend back past the accounts change (sessions, Google, passkey removal, deletion): older code ignores the log entries it added. Removed passkeys and deleted accounts' passkeys, profiles and recovery codes come back (a deleted account can sign in again, though it stays out of the trees it left), every ended session's token works again until it expires (sign out everywhere still ends them), and accounts made with Google vanish: they can't sign in, and their nodes stay in the trees under ids nobody can use. Recovery's unlinking of Google doesn't matter there, since older code has no Google.

Rolling the backend back past sites: older code skips the `communitySite` entries, so every community shows all its pages, indigo and no About; nothing else changes, and rolled forward again the sites are back.

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
