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

This is a superproject with the contracts, backend and frontend as submodules. First initialise them:

git submodule update --init --recursive

The working backend and frontend are on the `develop` branch of their respective submodules.

### contracts

Build with forge (the optimizer must be enabled in `foundry.toml`):

forge build

For local development with the backend and frontend, start an anvil node and deploy + seed the contract:

anvil --chain-id 1337

forge script script/Anthill.s.sol:SmallScript --broadcast --rpc-url http://localhost:8545

This deploys and seeds the contract at `0x5FbDB2315678afecb367f032d93F642f64180aa3`.

If using metamask you have to clear metamask activity between different anvil sessions, as nonce and other things might change. Do this in setting -> advanced -> reset account.

### backend

The backend runs on port 5001. For local anvil development set `testing=true`, then launch:

npm run start:dev

### frontend

The frontend runs as a Vite dev server on port 5173. For local anvil development set `testing=true`, then launch:

npm start

### deployment

The smart contract needs to be deployed, and the deployed address has to be set in the frontend and backend.
heroku builds backend based on github's main branch.
firebase frontend does not build on github yet, I should do that. But currently npm run build and npx firebase deploy deploys the frontend.
