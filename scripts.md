# Umiña Achala: Chain Scripts and Their Options

How to run the scripts in `umina-achala-chain`, what each command-line option does, and which Hedera feature each script uses. Every script prints the same information with `--help`.

> **Status:** written for [chain #29](https://github.com/umina-achala/umina-achala-chain/issues/29) (refactor steps 1–4, branch `refactor`, 2026-11-09). Anything that writes to Hedera is run by the project owner; dry-run every step first.

## Table of Contents

- [Common to Every Script](#common-to-every-script)
- [Run Order on Testnet](#run-order-on-testnet)
- [Tokens](#tokens): `createToken.js`, `adminToken.js`
- [Topics](#topics): `createTopic.js`
- [Minting](#minting): `mintNFT.js`
- [Sales](#sales): `sellOwnItem.js`, `recordEvent.js`
- [Consignment](#consignment): `associateToken.js`, `consignToMerchant.js`, `sellConsigned.js`, `endConsignment.js`
- [Hedera Documentation](#hedera-documentation)

## Common to Every Script

| Option | What it does |
|---|---|
| `--dry-run` | Checks the arguments, the environment and (where the script reads it) the provenance topic and the ledger, prints what would happen, and **sends nothing**. Use it before every real run. |
| `-h`, `--help` | Prints the usage and options and stops. It reads no env file and touches no network. |
| `NETWORK=<name>` | Not an option but a shell variable, set before the command: `NETWORK=testnet node src/...`. It chooses the Hedera network, its env file and its mirror node. `testnet` when unset. `mainnet` is listed but **not enabled** until its keys and accounts are decided ([chain #6](https://github.com/umina-achala/umina-achala-chain/issues/6)). |

**Env file.** Each script reads the network's own file, `.env.testnet` on testnet. `createToken.js` and `createTopic.js` also take a file as an argument; its name must contain the network's name, so a mainnet file is never used on testnet by mistake.

**Mistyped options are refused.** A script stops with "Problems found" when it gets an option it does not take (for example `--dryrun` instead of `--dry-run`), an option without its value, or too many arguments. Nothing is read from the env file and nothing is sent.

**Problems found.** Before sending anything, each script lists every problem it finds: a variable that is not set, an ID that does not look like `0.0.1234`, a key that is not an ECDSA private key, a bad serial or price. Fix them all and run again.

**Serials and prices.** A serial is the NFT's serial number within its token (a positive whole number). A price is in HBAR, with at most 8 decimals.

**`--token umina|achala`.** Chooses the token class: `umina` (a stone, `UMINA_TOKEN_ID`, the default) or `achala` (a piece of jewelry, `ACHALA_TOKEN_ID`). An Achala always moves with every stone mounted in it.

## Run Order on Testnet

The full checklist is [chain #22](https://github.com/umina-achala/umina-achala-chain/issues/22). In short:

1. `createToken.js`, then `createToken.js --token achala`; save the token IDs.
2. `createTopic.js provenance`; save `HCS_TOPIC_ID`.
3. `mintNFT.js` for stones, then for an Achala made of them.
4. `associateToken.js buyer` (and `merchant`), then the sales and consignment scripts.
5. `adminToken.js` for pause and burn.

## Tokens

### `createToken.js`

Creates an NFT token class: its name, symbol and memo, the 5% royalty with the fallback fee, and the role keys. No metadata key, so NFT metadata can never change.

```
node src/token/createToken.js [env-file] [--token umina|achala] [--dry-run]
```

| Argument / option | Meaning |
|---|---|
| `[env-file]` | The env file to read. Default: the network's own (`.env.testnet`). Its name must contain the network's name. |
| `--token umina\|achala` | The class to create (default `umina`). |
| `--dry-run` | Prints the token settings, without keys, and sends nothing. |

Needs: `OPERATOR_ID`, `TREASURY_ID`, `FALLBACK_FEE_HBAR` and the keys `OPERATOR_KEY`, `ADMIN_KEY`, `SUPPLY_KEY`, `FEE_SCHEDULE_KEY`, `PAUSE_KEY`, `WIPE_KEY` (and `TREASURY_KEY` when the treasury is not the operator). Prints the new token ID; save it as `UMINA_TOKEN_ID` or `ACHALA_TOKEN_ID`.

Hedera: [Create a token](https://docs.hedera.com/native/tokens/define), [Custom token fees](https://docs.hedera.com/native/tokens/custom-fees).

### `adminToken.js`

Uses the token's admin keys. Each command is signed by its own key.

```
node src/token/adminToken.js <command> [serial] [options] [--token umina|achala] [--dry-run]
```

| Command | Meaning | Signed by |
|---|---|---|
| `fees --fallback-hbar <hbar> [--royalty <percent>]` | Replaces the custom fees: the royalty (default 5%) and the fallback fee in HBAR. Use it when the HBAR price has moved the fallback fee away from about USD 3. | `FEE_SCHEDULE_KEY` |
| `pause` / `unpause` | Stops, or allows again, every transfer of every serial (a compliance hold). | `PAUSE_KEY` |
| `wipe <serial> --account <id> --reason <text>` | Removes a serial from the account that holds it; the serial is burned. | `WIPE_KEY` |
| `burn <serial> --reason <text>` | Burns a serial that is still in the Treasury. | `SUPPLY_KEY` |

| Option | Meaning |
|---|---|
| `--account <id>` | `wipe` only: the account that holds the serial. |
| `--reason <text>` | `wipe` and `burn`: why, at most 200 characters. It is recorded in the `BURN` event, which is public and permanent: no personal data. |
| `--fallback-hbar <hbar>`, `--royalty <percent>` | `fees` only. |
| `--token umina\|achala` | The token class (default `umina`). Burning an Achala frees its stones. |

A stone mounted in an Achala is refused by `wipe` and `burn`: burn the Achala instead (`--token achala`).

Hedera: [Update token custom fees](https://docs.hedera.com/native/tokens/update-fee-schedule), [Pause a token](https://docs.hedera.com/native/tokens/pause), [Unpause a token](https://docs.hedera.com/native/tokens/unpause), [Wipe a token](https://docs.hedera.com/native/tokens/wipe), [Burn a token](https://docs.hedera.com/native/tokens/burn).

## Topics

### `createTopic.js`

Creates an HCS topic with a submit key (only the project can post) and an admin key (the topic can be updated or deleted later).

```
node src/topic/createTopic.js <provenance|display> [env-file] [--force] [--dry-run]
```

| Argument / option | Meaning |
|---|---|
| `<provenance\|display>` | Which topic. Required, so a topic is never created without saying which. |
| `[env-file]` | As for `createToken.js`. |
| `--force` | Create a new topic even when the variable (`HCS_TOPIC_ID` or `HCS_DISPLAY_TOPIC_ID`) already names one that exists. |
| `--dry-run` | Prints the topic settings, without keys, and sends nothing. |

Needs: `OPERATOR_ID`, `OPERATOR_KEY`, `HCS_SUBMIT_KEY`, `HCS_ADMIN_KEY`.

Hedera: [Create a topic](https://docs.hedera.com/native/consensus/create-topic), [Submit a message](https://docs.hedera.com/native/consensus/submit-message).

## Minting

### `mintNFT.js`

Validates the metadata, makes the image versions, pins the images and the metadata to IPFS, records `COMPLIANCE`, mints, then records `MINT`.

```
node src/mint/mintNFT.js <metadata-file | --form post-file> [--token umina|achala] [--out dir] [--force] [--dry-run | --images-only]
```

| Argument / option | Meaning |
|---|---|
| `<metadata-file>` | A metadata JSON file to mint. |
| `--form <post-file>` | A mint form post. The metadata file is generated from it (from the class's template) and then minted. Give either this or a metadata file. |
| `--out <dir>` | Where `--form` writes the generated metadata file (default `generated/`). |
| `--token umina\|achala` | The token class (default `umina`). An Achala's stones must be minted and held by the Treasury. |
| `--force` | Mint even when the `stone_id` already has a `MINT` on the topic. |
| `--images-only` | Makes the image versions and stops: no network, no pinning. |
| `--dry-run` | Checks the environment and the metadata and sends nothing; it still writes the generated metadata file. |

Needs: `OPERATOR_ID`, `OPERATOR_KEY`, `SUPPLY_KEY`, the token ID, `HCS_TOPIC_ID`, `PINATA_JWT` (an Achala also `UMINA_TOKEN_ID` and `TREASURY_ID`). See [mint-nft-token.md](mint-nft-token.md).

Hedera: [Mint a token](https://docs.hedera.com/native/tokens/mint), [Submit a message](https://docs.hedera.com/native/consensus/submit-message).

## Sales

### `sellOwnItem.js`

Sells an item UA holds itself, straight from the Treasury, in one transfer: the buyer pays UA and the NFT goes to the buyer (an Achala with its stones). Records `SALE`.

```
node src/sale/sellOwnItem.js <serial> <price-hbar> [--token umina|achala] [--dry-run]
```

| Argument / option | Meaning |
|---|---|
| `<serial>` | The NFT serial, held by the Treasury. |
| `<price-hbar>` | The price in HBAR. |
| `--token umina\|achala` | The token class (default `umina`). A stone mounted in an Achala is refused. |

Needs: `OPERATOR_ID`, `OPERATOR_KEY`, `TREASURY_ID`, the token ID, `HCS_TOPIC_ID`, `BUYER_ID`, `BUYER_KEY`.

Hedera: [Transfer tokens](https://docs.hedera.com/native/tokens/transfer), [Get NFT info](https://docs.hedera.com/native/tokens/get-nft-info).

### `recordEvent.js`

Records a `SALE` or `DELIVERED` event for something that already happened, after checking it against the ledger. It never moves an NFT or HBAR.

```
node src/record/recordEvent.js sale <serial> --seller <id> --buyer <id>
  --nft-tx <tx-id> --payment <on_chain|off_chain> [--payment-tx <tx-id>]
  [--sold-for <amount> [--currency <code>]] [--stone-id <id>]
  [--token umina|achala] [--dry-run]

node src/record/recordEvent.js delivered <serial> --buyer <id>
  [--method in_person] [--delivered-at <date-time>] [--proof <file>]
  [--stone-id <id>] [--token umina|achala] [--dry-run]
```

| Option | Meaning |
|---|---|
| `--seller <id>`, `--buyer <id>` | The accounts. |
| `--nft-tx <tx-id>` | `sale`: the transaction that moved the NFT from the seller to the buyer. |
| `--payment on_chain\|off_chain` | `sale`: paid on Hedera, or in cash, bank or card. |
| `--payment-tx <tx-id>` | `sale`, `on_chain`: the payment, when it was a separate transaction. |
| `--sold-for <amount>`, `--currency <code>` | `sale`: the price. Checked against the payment when on chain; recorded as stated when off chain. |
| `--method in_person` | `delivered`: how it was delivered. |
| `--delivered-at <date-time>` | `delivered`: when. |
| `--proof <file>` | `delivered`: a JSON file with the delivery message signed by the buyer's key (optional). |
| `--stone-id <id>` | The item's `stone_id`, when it is not taken from its `MINT`. |

## Consignment

See [consignment.md](consignment.md) for the flow and the 70 / 30 split.

### `associateToken.js`

Associates a token with the merchant or buyer test account, so it can receive NFTs. On testnet only; in production the account does this in its own wallet.

```
node src/consignment/associateToken.js <merchant|buyer> [--token umina|achala] [--dry-run]
```

| Argument / option | Meaning |
|---|---|
| `<merchant\|buyer>` | The account: `MERCHANT_ID`/`MERCHANT_KEY` or `BUYER_ID`/`BUYER_KEY`. Its key signs; UA pays. |
| `--token umina\|achala` | The token to associate (default `umina`). |

Hedera: [Associate tokens to an account](https://docs.hedera.com/native/tokens/associate).

### `consignToMerchant.js`

Consigns an Umiña to the merchant: the merchant associates the token, the Treasury sends the NFT, the merchant gives UA an NFT allowance, and `ISSUED_TO_OWNER` is recorded. A stone mounted in an Achala is refused.

```
node src/consignment/consignToMerchant.js <serial> [--dry-run]
```

| Argument / option | Meaning |
|---|---|
| `<serial>` | The Umiña NFT serial, held by the Treasury. |

Hedera: [Approve an allowance](https://docs.hedera.com/native/accounts/approve-allowance), [Transfer tokens](https://docs.hedera.com/native/tokens/transfer).

### `sellConsigned.js`

Sells a consigned Umiña in one transfer: the buyer pays, the merchant ends with 70% and UA with 30% after the royalty, and UA moves the NFT with the merchant's allowance. Records `SALE`.

```
node src/consignment/sellConsigned.js <serial> <price-hbar> [--dry-run]
```

| Argument / option | Meaning |
|---|---|
| `<serial>` | The Umiña NFT serial, held by the merchant. |
| `<price-hbar>` | The price in HBAR. `--dry-run` shows what each side is paid and ends with. |

Hedera: [Transfer tokens](https://docs.hedera.com/native/tokens/transfer), [Custom token fees](https://docs.hedera.com/native/tokens/custom-fees).

### `endConsignment.js`

Ends a consignment: the merchant revokes UA's allowance and returns the NFT to the Treasury, recorded as `TRANSFER`.

```
node src/consignment/endConsignment.js <serial> [--keep] [--dry-run]
```

| Argument / option | Meaning |
|---|---|
| `<serial>` | The Umiña NFT serial, held by the merchant. |
| `--keep` | Only revoke the allowance: the merchant keeps the NFT and UA can no longer sell it. No event is recorded. |

Hedera: [Delete an allowance](https://docs.hedera.com/native/accounts/adjust-allowance), [Transfer tokens](https://docs.hedera.com/native/tokens/transfer).

## Hedera Documentation

The Hedera pages behind the scripts, from the [Hedera documentation](https://docs.hedera.com) (Native SDKs):

| Topic | Page |
|---|---|
| Token classes, keys, supply | [Create a token](https://docs.hedera.com/native/tokens/define), [Token types](https://docs.hedera.com/native/tokens/token-types), [Token ID](https://docs.hedera.com/native/tokens/), [NFT ID](https://docs.hedera.com/native/tokens/nft-id), [Token Properties](https://docs.hedera.com/learn/core-concepts/tokens/properties) |
| Royalty and fallback fee | [Custom token fees](https://docs.hedera.com/native/tokens/custom-fees), [Custom Fee Schedule](https://docs.hedera.com/learn/core-concepts/tokens/custom-fees), [Update token custom fees](https://docs.hedera.com/native/tokens/update-fee-schedule) |
| Minting and admin | [Mint](https://docs.hedera.com/native/tokens/mint), [Burn](https://docs.hedera.com/native/tokens/burn), [Wipe](https://docs.hedera.com/native/tokens/wipe), [Pause](https://docs.hedera.com/native/tokens/pause), [Unpause](https://docs.hedera.com/native/tokens/unpause), [Delete a token](https://docs.hedera.com/native/tokens/delete), [Update a token](https://docs.hedera.com/native/tokens/update) |
| Accounts and transfers | [Associate tokens](https://docs.hedera.com/native/tokens/associate), [Transfer tokens](https://docs.hedera.com/native/tokens/transfer), [Approve an allowance](https://docs.hedera.com/native/accounts/approve-allowance), [Delete an allowance](https://docs.hedera.com/native/accounts/adjust-allowance), [Get NFT info](https://docs.hedera.com/native/tokens/get-nft-info) |
| Consensus Service | [Create a topic](https://docs.hedera.com/native/consensus/create-topic), [Submit a message](https://docs.hedera.com/native/consensus/submit-message) |
| Network fees | [Fees](https://docs.hedera.com/networks/fees) |
