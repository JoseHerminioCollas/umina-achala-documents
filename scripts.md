# Umiña Achala: Chain Scripts and Their Options

How to run the scripts in `umina-achala-chain`, what each command-line option does, and which Hedera feature each script uses. Every script prints the same information with `--help`.

> **Status:** written for [chain #29](https://github.com/umina-achala/umina-achala-chain/issues/29) (refactor steps 1–4, on `main`, fork and upstream, 2026-11-09; the issue closed 2026-11-11). Its remaining items moved to [chain #28](https://github.com/umina-achala/umina-achala-chain/issues/28) (`--token` on the consignment scripts), [chain #33](https://github.com/umina-achala/umina-achala-chain/issues/33) (code changes after the testnet minting: splitting `mintNFT.js`, speed) and [chain #34](https://github.com/umina-achala/umina-achala-chain/issues/34) (white label: a config file for another token). Token creation, topic creation, minting and the HCS events described in detail for the first testnet run at chain tag `v0.2.0-testnet.1` (2026-11-10). Anything that writes to Hedera is run by the project owner; dry-run every step first.

## Table of Contents

- [Common to Every Script](#common-to-every-script)
- [Run Order on Testnet](#run-order-on-testnet)
- [Working Examples](#working-examples): a full testnet run, command by command
- [Tokens](#tokens): `createToken.js`, `adminToken.js`
- [Topics](#topics): `createTopic.js`
- [Minting](#minting): `mintNFT.js`
- [HCS Events](#hcs-events): the envelope, each event, signing, reading the topic
- [Sales](#sales): `sellOwnItem.js`, `recordEvent.js`
- [Consignment](#consignment): `associateToken.js`, `consignToMerchant.js`, `sellConsigned.js`, `endConsignment.js`
- [Hedera Documentation](#hedera-documentation)

## Common to Every Script

| Option | What it does |
|---|---|
| `--dry-run` | Checks the arguments, the environment and (where the script reads it) the provenance topic and the ledger, prints what would happen, and **sends nothing**. Use it before every real run. |
| `-h`, `--help` | Prints the usage and options and stops. It reads no env file and touches no network. |
| `NETWORK=<name>` | Not an option but a shell variable, set before the command: `NETWORK=testnet node src/...`. It chooses the Hedera network, its env file and its mirror node. `testnet` when unset. `mainnet` is listed but **not enabled** until its keys and accounts are decided ([chain #32](https://github.com/umina-achala/umina-achala-chain/issues/32)). |

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

## Working Examples

A full testnet run, in order, run from the root of `umina-achala-chain`. Run each command with `--dry-run` first, then again without it. The output shown is from real dry runs on 2026-11-10 at tag `v0.2.0-testnet.1`; the IDs will differ. Steps 1 and 2 were run for real on 2026-11-10 at `v0.2.0-testnet.2` ([chain #22](https://github.com/umina-achala/umina-achala-chain/issues/22)): Umiña [`0.0.11400486`](https://hashscan.io/testnet/token/0.0.11400486), Achala [`0.0.11400589`](https://hashscan.io/testnet/token/0.0.11400589), provenance topic [`0.0.11400670`](https://hashscan.io/testnet/topic/0.0.11400670), display topic [`0.0.11400772`](https://hashscan.io/testnet/topic/0.0.11400772).

The examples use these items. Serials count from 1 in each new token, so on a fresh token they are:

| Item | `stone_id` | Token, serial | Used for |
|---|---|---|---|
| Stone A | `UMINA-2026-CH-99` | Umiña #1 | Mounted in the Achala |
| Stone B | `UMINA-2026-CH-98` | Umiña #2 | Sold loose from the Treasury |
| Stone C | `UMINA-2026-CH-97` | Umiña #3 | Consigned and sold |
| Stone D | `UMINA-2026-CH-96` | Umiña #4 | Consigned, then returned |
| Pendant | `ACHALA-2026-PE-99` | Achala #1 | Sold with Stone A |

`<...>` marks a value to replace: an ID that a previous step printed.

### 0. Before you start

```bash
cd ~/projects/umina-achala/umina-achala-chain
git describe --tags          # v0.2.0-testnet.2
npm install && npm test
ls testnet/                  # achala_a.json  chrysocolla_a.json  chrysocolla_a1.jpg  chrysocolla_a1.png  …
```

In `.env.testnet`, besides the operator, treasury and role keys:

- **`HCS_SUBMIT_KEY` and `HCS_ADMIN_KEY`**: set both. This run uses a separate key for each, generated with OpenSSL, to practise mainnet ([create-hcs-topic.md](create-hcs-topic.md), "Getting one", option 3). They are keys, not accounts, so they need no HBAR. On testnet the operator key is also allowed.
- **`UMINA_TOKEN_ID`, `ACHALA_TOKEN_ID`, `HCS_TOPIC_ID`**: leave them empty. Steps 1 and 2 fill them.
- **`MERCHANT_ID`, `MERCHANT_KEY`, `BUYER_ID`, `BUYER_KEY`**: two testnet accounts with ECDSA keys, each with a little HBAR. The buyer pays for the sales in step 6. See [Test accounts](#test-accounts) below.
- **`PINATA_JWT`**: a key allowed to use both `pinFileToIPFS` and `pinJSONToIPFS`.

#### Test accounts

The merchant and the buyer are two extra testnet accounts from the [Hedera portal](https://portal.hedera.com), held by the same portal login as the operator account. They stand in for real people: on mainnet the merchant and the buyer sign in their own wallets, and these variables are not used.

1. In the portal, on **Testnet**, use **Create an account with ECDSA key**. Do it twice. Each new account is listed under **Extra Accounts**, marked `ECDSA`, with its account ID (`0.0.…`).
2. Expand each account and copy into `.env.testnet`:
   - its account ID → `MERCHANT_ID` or `BUYER_ID`;
   - its **private** key → `MERCHANT_KEY` or `BUYER_KEY`. Hex or DER both work. Not the public key, and not the EVM address (`0x` and 40 hex characters).
3. Each new account starts with 1000 testnet HBAR, more than the sales in the examples need (the buyer pays 25, 120 and 50 HBAR). The portal shows when each account can be refilled.

The two accounts must differ from each other and from the operator/Treasury account. Check them: this confirms each ID and key are well formed, then asks the mirror node whether the key belongs to that account. It prints no private keys.

```bash
node --input-type=module -e '
import dotenv from "dotenv"; import { envProblems } from "./src/env.js"; import { parseEcdsaKey } from "./src/keys.js"; import { accountKey } from "./src/mirror.js";
dotenv.config({ path: ".env.testnet" });
for (const r of ["MERCHANT", "BUYER"]) {
  const p = envProblems(process.env, [r+"_ID", r+"_KEY"]); if (p.length) { console.log(r, p); continue; }
  const k = await accountKey(process.env[r+"_ID"]);
  const mine = parseEcdsaKey(process.env[r+"_KEY"]).publicKey.toStringRaw();
  console.log(r, process.env[r+"_ID"], k?._type, k?.key?.endsWith(mine) ? "key matches" : "KEY DOES NOT MATCH");
}'
```

Each line should end in `ECDSA_SECP256K1 key matches`.

Any script prints its usage with `--help`:

```bash
node src/mint/mintNFT.js --help
```

### 1. Create the two tokens

```bash
node src/token/createToken.js --dry-run
```
```
🚀 Creating Umiña NFT Class...
Dry run, nothing sent:
{
  name: 'Umiña',
  symbol: 'UMINA',
  saveIdAs: 'UMINA_TOKEN_ID',
  type: 'NonFungibleUnique',
  supplyType: 'Infinite',
  treasury: '0.0.4631334',
  autoRenewAccount: '0.0.4631334',
  royalty: '5/100 to 0.0.4631334',
  fallbackFee: '45 HBAR',
  memo: 'Umiña: Peruvian stones',
  roles: [ 'admin', 'supply', 'feeSchedule', 'pause', 'wipe' ],
  metadataKey: 'none (NFT metadata is immutable)',
  maxTransactionFee: '50 HBAR (a ceiling; the network charges about USD 2 in HBAR)',
  treasurySignsSeparately: false
}
```
```bash
node src/token/createToken.js
```
```
✅ Success! Umiña Token ID: 0.0.<n>
SAVE THIS ID IN .env.testnet AS: UMINA_TOKEN_ID
```
Copy the ID into `.env.testnet`, then do the same for the Achala class:

```bash
node src/token/createToken.js --token achala --dry-run
node src/token/createToken.js --token achala          # save as ACHALA_TOKEN_ID
```

### 2. Create the provenance topic

```bash
node src/topic/createTopic.js provenance --dry-run
node src/topic/createTopic.js provenance              # save as HCS_TOPIC_ID
```

Without `HCS_SUBMIT_KEY` and `HCS_ADMIN_KEY`, the dry run stops with:
```
❌ Problems in .env.testnet:
   - HCS_SUBMIT_KEY is not set
   - HCS_ADMIN_KEY is not set
```

The display topic is optional; no script posts to it yet:
```bash
node src/topic/createTopic.js display --dry-run
node src/topic/createTopic.js display                 # save as HCS_DISPLAY_TOPIC_ID
```

### 3. Mint the stones

**Optional: look at the images and metadata first.** This needs no credentials and touches no network:
```bash
node src/mint/mintNFT.js --form testnet/chrysocolla_a.json --images-only
```
```
✅ Metadata file generated from testnet/chrysocolla_a.json: generated/UMINA-2026-CH-99.json
✅ Metadata for UMINA-2026-CH-99 are valid.
✅ Made generated/UMINA-2026-CH-99/01-master.jpg (image/jpeg, 1080×1080)
✅ Made generated/UMINA-2026-CH-99/01-display.jpg (image/jpeg, 1080×1080)
✅ Made generated/UMINA-2026-CH-99/02-master.jpg (image/jpeg, 300×300)
Would pin generated/UMINA-2026-CH-99/01-master.jpg (image/jpeg)
...
Would pin this metadata (CIDs pending):
{ ... }
```

**Stone A** (`UMINA-2026-CH-99`, the sample post as it is):
```bash
node src/mint/mintNFT.js --form testnet/chrysocolla_a.json --dry-run
node src/mint/mintNFT.js --form testnet/chrysocolla_a.json
```
```
✅ COMPLIANCE recorded for UMINA-2026-CH-99: 0.0.<operator>@<time>
✅ Pinned 01-master.jpg: ipfs://<cid>
...
✅ Metadata pinned to IPFS: ipfs://<metadata-cid>
✅ Minted Umiña NFT serials: 1 (0.0.<operator>@<time>)
```
followed by a table with `stone_id`, `event: MINT`, `ipfs_cid` and `hcs_tx_id`.

**Stones B, C and D.** Copy the sample post into `generated/` with a new `stone_id`. `generated/` is git-ignored. The post names its files relative to `testnet/`, so the copy points them at `../testnet/`:
```bash
for n in 98 97 96; do
  sed -e "s/UMINA-2026-CH-99/UMINA-2026-CH-$n/" -e 's#"file": "#"file": "../testnet/#' testnet/chrysocolla_a.json > generated/post-CH-$n.json
done
node src/mint/mintNFT.js --form generated/post-CH-98.json --dry-run
node src/mint/mintNFT.js --form generated/post-CH-98.json     # serial 2
node src/mint/mintNFT.js --form generated/post-CH-97.json     # serial 3
node src/mint/mintNFT.js --form generated/post-CH-96.json     # serial 4
```
Only `stone_id` changes; the name still says "#99". Minting a `stone_id` again is refused:
```
❌ UMINA-2026-CH-99 is already minted (MINT event #2, serials 1). Use --force to mint it again.
```

### 4. Mint the Achala

The sample Achala post names its stone as serial `3`. Point it at Stone A, serial 1:
```bash
sed -e 's/"serial": 3/"serial": 1/' -e 's#"file": "#"file": "../testnet/#' testnet/achala_a.json > generated/post-PE-99.json
node src/mint/mintNFT.js --form generated/post-PE-99.json --token achala --dry-run
node src/mint/mintNFT.js --form generated/post-PE-99.json --token achala
```
Before posting anything, a real run checks the stone on the ledger:
```
✅ Stones checked: UMINA-2026-CH-99
✅ COMPLIANCE recorded for ACHALA-2026-PE-99: ...
...
✅ Minted Achala NFT serials: 1 (...)
```
Stone A is now mounted. Without `--token achala` the post is refused, with a hint to add it.

### 5. Associate the tokens with the buyer

The buyer receives an Achala together with its stones, so the buyer associates both tokens. `consignToMerchant.js` associates the merchant by itself, so the merchant line is optional. An account that is already associated is reported and skipped.
```bash
node src/consignment/associateToken.js buyer --dry-run
node src/consignment/associateToken.js buyer
node src/consignment/associateToken.js buyer --token achala
node src/consignment/associateToken.js merchant
```

### 6. Sell from the Treasury

**Stone B, loose, for 25 HBAR:**
```bash
node src/sale/sellOwnItem.js 2 25 --dry-run
node src/sale/sellOwnItem.js 2 25
```
```
✅ Sold 0.0.<umina>/2 to 0.0.<buyer>: SUCCESS (0.0.<operator>@<time>)
```

**Stone A on its own is refused**, also on `--dry-run`, because it is mounted:
```bash
node src/sale/sellOwnItem.js 1 25 --dry-run
```
```
❌ ... UMINA-2026-CH-99 (0.0.<umina>/1) is mounted in Achala ACHALA-2026-PE-99 ...: it moves only with its Achala, and is freed when the Achala is burned
```

**The pendant with its stone, for 120 HBAR:**
```bash
node src/sale/sellOwnItem.js 1 120 --token achala --dry-run
node src/sale/sellOwnItem.js 1 120 --token achala
```
This moves Achala #1 and Umiña #1 to the buyer in one transfer. The `SALE` event lists `from_stone_ids` and `umina_serials`.

### 7. Consignment

**Stone C: consign it, then sell it for 50 HBAR.** The merchant ends with 70% and UA with 30%, royalty included:
```bash
node src/consignment/consignToMerchant.js 3 --dry-run
node src/consignment/consignToMerchant.js 3
node src/consignment/sellConsigned.js 3 50 --dry-run      # shows what each side is paid and ends with
node src/consignment/sellConsigned.js 3 50
```

**Stone D: consign it, then end the consignment:**
```bash
node src/consignment/consignToMerchant.js 4
node src/consignment/endConsignment.js 4 --dry-run
node src/consignment/endConsignment.js 4                  # back to the Treasury, recorded as TRANSFER
```
`endConsignment.js 4 --keep` would instead only revoke UA's allowance and leave the NFT with the merchant.

### 8. Record a sale or a delivery after the fact

A sale made wallet to wallet or paid off chain is recorded with `recordEvent.js`, which checks it against the ledger first. For example, the delivery of Stone B to the buyer:
```bash
node src/record/recordEvent.js delivered 2 --buyer 0.0.<buyer> --method in_person --dry-run
node src/record/recordEvent.js delivered 2 --buyer 0.0.<buyer> --method in_person
```
And a sale paid in cash, where `<nft-tx>` is the transaction that moved the NFT:
```bash
node src/record/recordEvent.js sale 2 --seller 0.0.<seller> --buyer 0.0.<buyer> \
  --nft-tx <nft-tx> --payment off_chain --sold-for 300 --currency PEN --dry-run
```

### 9. Admin

```bash
node src/token/adminToken.js pause --dry-run
node src/token/adminToken.js pause                         # every transfer of every serial stops
node src/token/adminToken.js unpause

node src/token/adminToken.js fees --fallback-hbar 40 --dry-run
node src/token/adminToken.js fees --fallback-hbar 40       # royalty stays 5%

node src/token/adminToken.js burn 4 --reason "testnet check" --dry-run
node src/token/adminToken.js burn 4 --reason "testnet check"   # Stone D, back in the Treasury
```
`burn` and `wipe` require `--reason`. It is public and permanent, so it must contain no personal data:
```
❌ Problems found:
   - --reason is required: it is recorded in the BURN event
```
`wipe` removes a serial from the account that holds it. For example, Stone C from the buyer:
```bash
node src/token/adminToken.js wipe 3 --account 0.0.<buyer> --reason "testnet check" --dry-run
```

### 10. Check the results

```bash
# the topic's events, decoded
curl -s "https://testnet.mirrornode.hedera.com/api/v1/topics/<topic-id>/messages?limit=100" \
  | jq -r '.messages[].message | @base64d'

# who holds each serial
curl -s "https://testnet.mirrornode.hedera.com/api/v1/tokens/<umina-token-id>/nfts" \
  | jq '.nfts[] | {serial_number, account_id, deleted}'
```
Or on HashScan: `https://hashscan.io/testnet/topic/<topic-id>` and `https://hashscan.io/testnet/token/<token-id>`.

### Mistakes the scripts catch

A mistyped option sends nothing:
```bash
node src/token/createToken.js --dryrun
```
```
❌ Problems found:
   - Unknown option '--dryrun'. ...
```
A missing variable is named before anything is sent:
```bash
node src/sale/sellOwnItem.js 1 25 --dry-run      # with BUYER_ID and BUYER_KEY not set yet
```
```
❌ Problems found:
   - BUYER_ID is not set
   - BUYER_KEY is not set
```

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

#### The two classes

Both classes have the same settings except these:

| | Umiña (`--token umina`, default) | Achala (`--token achala`) |
|---|---|---|
| Name | `Umiña` | `Achala` |
| Symbol | `UMINA` | `ACHALA` |
| Memo | `Umiña: Peruvian stones` | `Achala: Peruvian jewelry made of Umiña stones` |
| Save the ID as | `UMINA_TOKEN_ID` | `ACHALA_TOKEN_ID` |

#### What the token is created with

One `TokenCreateTransaction` sets all of the following. Only the public halves of the keys go on the token.

| Setting | Value | Why |
|---|---|---|
| Type | `NonFungibleUnique` | Each serial is one item: one stone or one piece of jewelry. |
| Supply type | `Infinite`, initial supply 0, decimals 0 | Serials exist only when `mintNFT.js` mints them; there is no cap. |
| Treasury | `TREASURY_ID` | New serials land here, and the royalty is paid here. |
| Auto-renew account | `TREASURY_ID` | Pays the token's renewal so it does not expire. The renewal period is the SDK default, 90 days: HashScan shows an expiry 90 days ahead, which moves forward at each renewal. Keep the Treasury funded, or the token can expire. |
| Admin key | public key of `ADMIN_KEY` | Can update or delete the token later. It must also sign the creation. |
| Supply key | public key of `SUPPLY_KEY` | Signs every mint and every `burn`. |
| Fee schedule key | public key of `FEE_SCHEDULE_KEY` | Signs `adminToken.js fees`. |
| Pause key | public key of `PAUSE_KEY` | Signs `adminToken.js pause` / `unpause`. |
| Wipe key | public key of `WIPE_KEY` | Signs `adminToken.js wipe`. |
| Metadata key | **none** | NFT metadata can never change. A key left off at creation can never be added (chain #1). |
| Freeze and KYC keys | none | Not used. |
| Custom fee | one royalty fee, see below | |
| Max transaction fee | 50 HBAR | The most the operator will pay for the creation. A token with custom fees costs about USD 2, charged in HBAR at the hour's rate. The SDK's default cap, 30 HBAR, is too low once HBAR is under about 6.7 cents: on 2026-11-10, at 6.54 cents, the network refused the creation with `INSUFFICIENT_TX_FEE` (fixed after tag `v0.2.0-testnet.1`). 50 HBAR covers USD 2 down to 4 cents. It is a ceiling: the network charges the actual fee. |

**The royalty.** 5/100 of the value exchanged for the NFT, paid to `TREASURY_ID` on each transfer that carries HBAR or a fungible token with the NFT. When nothing of value moves with the NFT (a gift, or a sale paid off chain), Hedera charges the **fallback fee** instead: `FALLBACK_FEE_HBAR` HBAR, paid by the receiver. The fallback fee should be about USD 3, converted on the day the token is created. Record the amount and the rate on chain #6, and replace it later with `adminToken.js fees` if the HBAR price moves. `FALLBACK_FEE_HBAR` must be a positive amount with at most 8 decimals, for example `38` or `37.5`.

#### Step by step

1. **Read the command line.** An unknown option, more than one env file, or a `--token` that is not `umina` or `achala` is refused here, before the env file is read.
2. **Choose the env file.** It is the network's own file (`.env.testnet`) or the one given. A file whose name does not contain the network's name is refused, so mainnet credentials are never sent to testnet or the other way round.
3. **Check the environment.** Every problem is listed at once:
   - `OPERATOR_ID` and `TREASURY_ID` must look like `0.0.1234`.
   - `FALLBACK_FEE_HBAR` must be set and positive.
   - `OPERATOR_KEY`, `ADMIN_KEY`, `SUPPLY_KEY`, `FEE_SCHEDULE_KEY`, `PAUSE_KEY` and `WIPE_KEY` must be ECDSA (secp256k1) private keys, as 64 hex characters (optionally with `0x`) or as DER. An ED25519 key is refused.
   - When `TREASURY_ID` is not `OPERATOR_ID`, `TREASURY_KEY` is required too, because the treasury must sign.
4. **Build the transaction** with the settings above.
5. **`--dry-run` stops here.** It prints the name, symbol, the variable to save the ID as, type, supply type, treasury, auto-renew account, royalty, fallback fee, memo, the role names, `metadataKey: none`, the max transaction fee and whether the treasury signs separately. It prints no keys and makes no network call.
6. **Connect** to the network as the operator (`OPERATOR_ID`, `OPERATOR_KEY`). The operator pays the fee.
7. **Sign and send.** The transaction is frozen and signed with `ADMIN_KEY` (and with `TREASURY_KEY` when the treasury is a separate account). The operator signs as payer when it executes.
8. **Wait for the receipt** and print the new token ID with `SAVE THIS ID IN .env.testnet AS: UMINA_TOKEN_ID` (or `ACHALA_TOKEN_ID`).

The script does not edit the env file: copy the ID into it yourself. Running it again creates **another** token; it does not check whether the variable already names one.

**If it fails,** it prints the error with the operator and treasury IDs. A failure before the receipt means no token was created. Common causes: the operator account has too little HBAR; a key in the env file does not belong to the account it is used for; or `INSUFFICIENT_TX_FEE`, when USD 2 in HBAR is more than the 50 HBAR cap (HBAR under 4 cents). The receipt in the error shows the network's rate (`exchangeRateInCents`), and the mirror node shows what was charged: a refused creation costs only the node fee, about 0.002 HBAR.

**Check it** on HashScan (`https://hashscan.io/testnet/token/<token-id>`) or the mirror node (`https://testnet.mirrornode.hedera.com/api/v1/tokens/<token-id>`): the custom fees, the keys and `metadata_key: null`.

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

#### The two topics

| Kind | Memo | Save the ID as | Holds |
|---|---|---|---|
| `provenance` | `Umiña Achala provenance` | `HCS_TOPIC_ID` | Each item's history: `COMPLIANCE`, `MINT`, `ISSUED_TO_OWNER`, `SALE`, `TRANSFER`, `DELIVERED`, `BURN` (see [HCS Events](#hcs-events)). Every script that records or checks an event uses this topic. |
| `display` | `Umiña Achala display` | `HCS_DISPLAY_TOPIC_ID` | What the site shows and what is for sale (chain #10). No script posts to it yet. |

#### What the topic is created with

| Setting | Value | Why |
|---|---|---|
| Submit key | public key of `HCS_SUBMIT_KEY` | Only messages signed with this key are accepted, so nobody else can write to the history. |
| Admin key | public key of `HCS_ADMIN_KEY` | Can update or delete the topic later. It must also sign the creation. |
| Auto-renew account | `OPERATOR_ID` | Pays the topic's renewal so it does not expire. |

On mainnet each is a separate key. This testnet run uses separate keys too, generated with OpenSSL, to practise mainnet; see [create-hcs-topic.md](create-hcs-topic.md) for how to generate them in a format the scripts read. On testnet the operator key is also allowed (one key for every role). `--dry-run` shows whether each key is the same as the operator key.

#### Step by step

1. **Read the command line.** The kind (`provenance` or `display`) is required. An unknown option or more than two arguments is refused.
2. **Choose the env file**, as for `createToken.js`: the network's own, or a file whose name contains the network's name.
3. **Check the environment.** `OPERATOR_ID` must look like `0.0.1234`. `OPERATOR_KEY`, `HCS_SUBMIT_KEY` and `HCS_ADMIN_KEY` must be ECDSA private keys. Both topic keys are required, so a topic is never created that anyone could post to or that could not be managed later.
4. **Build the transaction** with the memo and the keys above.
5. **`--dry-run` stops here.** It prints the kind, memo, the variable to save the ID as, the auto-renew account, and whether each key is the operator key. No keys, no network.
6. **Refuse to replace a topic** (skipped with `--force`). It reads the mirror node: if `HCS_TOPIC_ID` (or `HCS_DISPLAY_TOPIC_ID`) already names a topic that exists and is not deleted, it stops and names that topic and its memo. A placeholder ID that does not exist is fine.
7. **Connect** as the operator, who pays.
8. **Sign and send.** The transaction is signed with `HCS_ADMIN_KEY`; the submit key does not sign the creation.
9. **Wait for the receipt** and print the new topic ID with `SAVE THIS ID IN .env.testnet AS: HCS_TOPIC_ID` (or `HCS_DISPLAY_TOPIC_ID`).

Copy the ID into the env file yourself. Keep `HCS_SUBMIT_KEY` in the env file afterwards: every script that records an event signs with it (see [Signing](#signing)).

A new provenance topic starts empty. Items minted on an older topic are not on it, so `mintNFT.js`, the sales scripts and `recordEvent.js` treat them as never minted. Start a new testnet run with a new token **and** a new topic, so the two match.

**Check it** on HashScan (`https://hashscan.io/testnet/topic/<topic-id>`) or the mirror node (`https://testnet.mirrornode.hedera.com/api/v1/topics/<topic-id>`): the memo, `submit_key` and `admin_key`.

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

Needs: `OPERATOR_ID`, `OPERATOR_KEY`, `SUPPLY_KEY`, the token ID, `HCS_TOPIC_ID`, `PINATA_JWT` (an Achala also `UMINA_TOKEN_ID` and `TREASURY_ID`). `HCS_SUBMIT_KEY` is needed unless the topic's submit key is the operator key (see [Signing](#signing)). `--images-only` needs none of them. See [mint-nft-token.md](mint-nft-token.md).

#### The order: record, pin, mint, record

A mint writes to three places: HCS, IPFS (through Pinata) and the token. They happen in this order, so a failure never leaves a `MINT` event for an NFT that does not exist:

| # | Step | Writes to | If it fails |
|---|---|---|---|
| 1 | `COMPLIANCE` event | HCS | Nothing was pinned or minted. |
| 2 | Pin each image version and document | IPFS | The CIDs pinned so far are printed: unpin them in Pinata. The `COMPLIANCE` event stays on the topic. |
| 3 | Pin the metadata JSON | IPFS | As step 2. |
| 4 | Mint one serial whose metadata is `ipfs://<metadata CID>` | Token | As step 2. |
| 5 | `MINT` event | HCS | The NFT exists but has no `MINT`. The script prints the serial and the `ipfs://` URI so it can be recorded by hand. For an Achala, its stones are then not recorded as mounted. |

`COMPLIANCE` comes first because the metadata is immutable once pinned: its `compliance_proof_hcs` can only point to a message that already exists.

#### The input: a metadata file or a form post

- **Metadata file** (`node src/mint/mintNFT.js <file>`): a HIP-412 JSON file with no `image` or `files` of its own. It names its photos and documents in input-only fields, each with a `file` (a path relative to the metadata file) and a `description`:
  - `main_image` (required): its display version becomes `image`, and its master the default file.
  - `images` (optional): up to 5 more photos.
  - `documents` (optional): receipts, permits and 3D scans, `.pdf` or `.glb`.
- **Form post** (`--form <post-file>`): a JSON file standing in for the mint form. The script copies the class's template (`json/umina-template.json` or `json/achala-template.json`), fills it from the post, and writes it to `<out>/<stone_id>.json` (default `generated/`):
  - `name`, `description` and `creator` go to the top level (`creator` stays the template's, Umiña Achala, unless the post sets it to the seller or artisan).
  - `attributes` (a map) becomes `attributes` pairs.
  - Every other field goes under `properties`.
  - Upload paths are made absolute, and empty fields (`null` or `""`) are removed, so a missing compliance field fails validation instead of passing as an empty string.

  The generated file is written even on `--dry-run`, so it can be read before minting.

`properties.stone_id` decides the class and must match `--token`: `UMINA-YYYY-XX-NN` (e.g. `UMINA-2026-CH-03`) for an Umiña, `ACHALA-YYYY-XX-NN` (e.g. `ACHALA-2026-RI-01`) for an Achala. Metadata of the other class is refused, with a hint to give `--token`.

#### The photos

Each photo is checked from its file header before anything is written:

- JPEG, PNG or WebP only, by the real type and not the extension. GIF, HEIC (iPhone) and RAW are refused; export iPhone photos as JPEG.
- At most 25 MB and about 100 megapixels.
- Each photo listed once.

Then they are re-encoded with sharp into `generated/<stone_id>/`. Each is turned upright from its EXIF orientation and written **without any metadata**, so no EXIF or GPS location of a mine reaches IPFS:

| File | Made for | Format |
|---|---|---|
| `01-master.<ext>`, `02-master.<ext>`, ... | every photo (01 is `main_image`) | Full size, the upload's format: JPEG and WebP at quality 95, PNG lossless. |
| `01-display.jpg` | `main_image` only | 1080 px on the long side (never enlarged), JPEG quality 82. |

The SHA-256 of each master is computed. Running again overwrites the files.

#### The pinned metadata

What is pinned is the metadata with:

- `main_image`, `images` and `documents` removed.
- `image` = `ipfs://<CID of 01-display.jpg>`, and `type` = `image/jpeg`. Wallets and marketplaces show `image`.
- `files[]`: one entry per master (`uri`, `type`, `checksum` (its SHA-256), `metadata.description`; the main one has `is_default_file: true`), then one per document. The web site and the app fetch the masters and resize them, so no thumbnails are made.
- `properties.compliance_proof_hcs` = the `COMPLIANCE` transaction ID, and `properties.hcs_compliance_topic` = `HCS_TOPIC_ID`.
- The attributes wallets show, written from the data, replacing any hand-written ones with the same `trait_type`:
  - Umiña: `Stone Type`, `Materials` and `Form`, from `schema/stone-types.json`.
  - Achala: `Piece Type`, `Metal`, `Stones` (each stone's type name, from its `COMPLIANCE` event) and `Artisan`.
- An Achala's `properties.umina_token_id` filled from `UMINA_TOKEN_ID`. Any other value is refused.

Each file is pinned to Pinata (`pinFileToIPFS`) under the name `<stone_id>-<file name>`, and the same file listed twice is pinned once. The metadata is pinned with `pinJSONToIPFS` under the name `<stone_id>`. `PINATA_JWT` needs both permissions.

#### Step by step

1. **Read the command line.** One of a metadata file or `--form` is required, not both. `--dry-run` and `--images-only` cannot be combined.
2. **Check the environment** (skipped by `--images-only`). `OPERATOR_ID`, the token ID (`UMINA_TOKEN_ID` or `ACHALA_TOKEN_ID`) and `HCS_TOPIC_ID` must look like `0.0.1234`. `OPERATOR_KEY` and `SUPPLY_KEY` must be ECDSA keys. `PINATA_JWT` must be set. `HCS_SUBMIT_KEY`, if set, must be an ECDSA key. An Achala also needs `UMINA_TOKEN_ID` and `TREASURY_ID`.
3. **Generate the metadata file** from `--form`, if given.
4. **Check the metadata**, listing every problem at once:
   - the photo and document entries;
   - that every file exists (and that `testnet/`, which is git-ignored, exists when files point into it);
   - `stone_id` against the class;
   - each photo's header;
   - the schema (`schema/hip412-umina-schema.json` or `schema/hip412-achala-schema.json`), with placeholder `ipfs://pending` URIs since nothing has a CID yet;
   - what a schema cannot check: stone types against the list (not deprecated, not a fossil, exactly one `host` in `materials`), and the check digits of `vendor_ruc` and `reinfo_id`.
5. **`--dry-run` stops here.** It lists the image files it would make and the documents it would pin. Nothing is written except the generated metadata file, and nothing is sent.
6. **Make the image versions** into `generated/<stone_id>/`.
7. **`--images-only` stops here.** It lists what it would pin, prints the metadata it would pin (CIDs pending), and names the token and topic it would use. It needs no credentials and touches no network.
8. **Read the provenance topic** from the mirror node. It stops if the topic does not exist or is deleted. It stops if the `stone_id` already has a `MINT` event, naming that event's sequence number and serials; `--force` mints it again anyway.
9. **An Achala: check its stones.** For each of `properties.stones` (`stone_id` and `serial`), it reads the NFT from the mirror node and checks that:
   - there is a `MINT` for that `stone_id` in `UMINA_TOKEN_ID` with that serial;
   - the serial exists and is not burned;
   - `TREASURY_ID` holds it;
   - it is not mounted in another Achala that still exists (an Achala `MINT` that lists it, with no `BURN` of that Achala).

   `--force` does **not** skip these checks.
10. **Connect** as the operator.
11. **Record `COMPLIANCE`** in the provenance topic and wait for its receipt.
12. **Pin** each master, the display version and each document, then the metadata JSON.
13. **Mint.** A `TokenMintTransaction` on the class's token, with one metadata entry, the bytes of `ipfs://<metadata CID>`, signed with `SUPPLY_KEY`. The operator pays. The receipt gives the new serial: serials count up from 1 in each token, and the token's treasury holds the new NFT.
14. **Record `MINT`** with the serial, `ref` = the mint transaction ID, and `ipfs_cid` = the metadata CID. An Achala's `MINT` adds `from_stone_ids`, which records its stones as mounted.
15. **Print a table:** `stone_id`, `event`, `actor_id`, `ipfs_cid`, `hcs_tx_id` and the time.

#### Minting on testnet with the sample posts

- `testnet/chrysocolla_a.json` mints `UMINA-2026-CH-99` from `chrysocolla_a1.jpg` (main image) and `chrysocolla_a1.png`–`a3.png` (three views, transparent), with `chrysocolla_a_store-receipt.pdf` and `chrysocolla_a_mining-permit.pdf`, all beside the post. `testnet/` is a symlink to the `umina-achala-testnet` repo, which holds every testnet post, image and document; it is git-ignored here, and its README has the naming convention.
- To mint a second stone, copy the post and change its `stone_id` (e.g. `UMINA-2026-CH-98`). The same `stone_id` again is refused without `--force`, and minting it again with `--force` gives one `stone_id` two serials. An Achala sale refuses a stone minted twice, because its serial would be a guess.
- `testnet/achala_a.json` mints `ACHALA-2026-PE-99` with `"stones": [{ "serial": 3, "stone_id": "UMINA-2026-CH-99" }]`. On a new token, change the serial to the one the Umiña mint printed (the first is 1).
- A stone in an Achala can only be sold, consigned or burned with its Achala. To test the sale of a loose stone, mint one more stone and leave it out of every Achala.

**Check it** on HashScan (`https://hashscan.io/testnet/token/<token-id>/<serial>`), on the mirror node (`/api/v1/tokens/<token-id>/nfts/<serial>`: the `metadata` field is the base64 of `ipfs://<CID>`), and on any IPFS gateway (`https://gateway.pinata.cloud/ipfs/<CID>`).

Hedera: [Mint a token](https://docs.hedera.com/native/tokens/mint), [Submit a message](https://docs.hedera.com/native/consensus/submit-message).

## HCS Events

Every record the scripts make is one JSON message on the provenance topic (`HCS_TOPIC_ID`). The topic is the item's history: the scripts read it back from the mirror node before they act, for example to find a stone's serial, to refuse a second `MINT`, or to see whether a stone is mounted.

### The envelope

Every event starts with the same fields (chain #10), followed by the event's own:

| Field | Meaning |
|---|---|
| `event` | The event type, e.g. `MINT`. |
| `schema_version` | `1`. Bumped when an event's shape changes. |
| `stone_id` | The item's `stone_id` (`UMINA-...` or `ACHALA-...`), when there is one. |
| `token_id` | The token, `0.0.1234`. |
| `serials` | The serials, as strings. Empty for `COMPLIANCE`, which comes before the mint. |
| `actor_id` | The account that recorded it: the operator. |
| `occurred_at` | When it happened, UTC to the second, e.g. `2026-11-10T14:00:00Z`. |
| `ref` | The related Hedera transaction ID, when there is one (the mint, the transfer). |

A message must fit in one HCS message, **1024 bytes** of JSON. A larger event is refused before anything is sent.

### The events

| Event | Recorded by | Its own fields |
|---|---|---|
| `COMPLIANCE` | `mintNFT.js`, before pinning | Umiña: `stone_type`, `vendor_ruc`, `mining_concession`, `reinfo_id`, `hs_code`. Achala: `piece_type`, `artisan_rna`, `vendor_ruc`, `hs_code`, `from_stone_ids`. |
| `MINT` | `mintNFT.js`, after the mint | `ipfs_cid` (the metadata). An Achala adds `from_stone_ids`. `ref` is the mint transaction. |
| `ISSUED_TO_OWNER` | `consignToMerchant.js` | `to_id` (the merchant) and `allowance_ref` (the allowance transaction). `ref` is the transfer to the merchant. |
| `SALE` | `sellOwnItem.js`, `sellConsigned.js`, `recordEvent.js sale` | `seller_id`, `buyer_id`, `payment` (`on_chain` or `off_chain`) and the price (`sold_for`). An Achala adds `from_stone_ids`, `umina_token_id` and `umina_serials`. `ref` is the transfer. |
| `TRANSFER` | `endConsignment.js` | `from_id` (the merchant), `to_id` (the Treasury), `reason: "consignment_ended"`. `ref` is the return transfer. |
| `DELIVERED` | `recordEvent.js delivered` | The method, the time, and the proof (`signed` or `none`). `ref` is the sale. |
| `BURN` | `adminToken.js wipe` and `burn` | `reason`, `method` (`wipe` or `burn`), and for wipe `from_id`. `ref` is the wipe or burn. Burning an Achala unmounts its stones. |

How the scripts read them:

- **Minted:** a `MINT` with that `stone_id` (and for a serial, that token and serial).
- **Mounted:** a stone is mounted while an Achala `MINT` lists it in `from_stone_ids` and no `BURN` names that Achala serial. The stones get no events of their own. `from_stone_ids` in `COMPLIANCE` does not mount anything; only the `MINT` does.
- **Stone type of a stone in an Achala:** its latest `COMPLIANCE`.

### Signing

The topic only accepts messages signed with its submit key. Each message is a `TopicMessageSubmitTransaction`, paid for and signed by the operator. When `HCS_SUBMIT_KEY` is set, it also signs.

- When the topic's submit key **is** the operator key, the operator's signature is enough and `HCS_SUBMIT_KEY` may be left empty.
- When it is a **different** key, `HCS_SUBMIT_KEY` must be set, or every event is rejected with `INVALID_SIGNATURE`. In `mintNFT.js` that happens at `COMPLIANCE`, before anything is pinned or minted.

Simplest on testnet: set `HCS_SUBMIT_KEY` to the same value as when the topic was created and leave it there.

Each submission waits for its receipt, so a message rejected at consensus (a wrong signature, a deleted or expired topic) is reported as a failure, not as a record. The script prints the transaction ID of each message it records.

### Reading the topic

- The mirror node lists the messages, oldest first, base64 encoded: `https://testnet.mirrornode.hedera.com/api/v1/topics/<topic-id>/messages`. The scripts decode them and skip anything that is not a JSON object.
- HashScan shows them decoded: `https://hashscan.io/testnet/topic/<topic-id>`.

The mirror node can be a few seconds behind consensus. A script run straight after another may not see the event just recorded yet; wait a few seconds and run it again.

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

The three consignment scripts take an Umiña serial only; `associateToken.js` already takes `--token`. Consigning an Achala with its stones (`--token achala` on `consignToMerchant.js`, `sellConsigned.js` and `endConsignment.js`) is [chain #28](https://github.com/umina-achala/umina-achala-chain/issues/28), after the testnet minting ([chain #33](https://github.com/umina-achala/umina-achala-chain/issues/33)).

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
