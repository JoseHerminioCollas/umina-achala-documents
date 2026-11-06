# Umiña NFT Transfers and Sales
- [Umiña NFT Transfers and Sales](#umiña-nft-transfers-and-sales)
  - [Overview](#overview)
  - [What Defines an Umiña NFT](#what-defines-an-umiña-nft)
    - [Token Creation](#token-creation)
    - [Minting: How Much Can Vary](#minting-how-much-can-vary)
    - [What UA Can Still Change After a Sale](#what-ua-can-still-change-after-a-sale)
  - [Transferring an NFT](#transferring-an-nft)
    - [Coin Transfer Compared With NFT Transfer](#coin-transfer-compared-with-nft-transfer)
    - [Wallet to Wallet (No Code)](#wallet-to-wallet-no-code)
    - [In UA's Code](#in-uas-code)
  - [After the NFT Leaves UA](#after-the-nft-leaves-ua)
    - [History on the Ledger](#history-on-the-ledger)
    - [Is the Price on the Ledger?](#is-the-price-on-the-ledger)
    - [What HCS Adds](#what-hcs-adds)
  - [When the Royalty Is Paid](#when-the-royalty-is-paid)
  - [UA Sales: What Can Happen in a Wallet](#ua-sales-what-can-happen-in-a-wallet)
    - [Why a Wallet Alone Cannot Make UA's Sale](#why-a-wallet-alone-cannot-make-uas-sale)
    - [Sale Options](#sale-options)
    - [UA Selling Its Own Items Wallet to Wallet](#ua-selling-its-own-items-wallet-to-wallet)
  - [Website: Showing Sold Stones](#website-showing-sold-stones)
    - [Current State](#current-state)
    - [Mirror Node Ownership Check](#mirror-node-ownership-check)
    - [Where It Goes in the Code](#where-it-goes-in-the-code)
    - [Failure Behavior](#failure-behavior)
  - [Open Decisions](#open-decisions)
  - [Code and Issue Reference](#code-and-issue-reference)

## Overview

> **Status: Phase 2 (testnet), in progress.** The code referred to here is in `umina-achala-chain` (on the fork's `main`, not yet upstream) and `umina-achala-web`, as of 2026-11-05. Code is referred to by file and function, not line number. No token or HCS topic exists yet: they are created once all Phase 2A code is written. Work that is not built is tracked in the issues listed in [Code and Issue Reference](#code-and-issue-reference).

This document explains how an Umiña NFT moves between accounts once it exists: plain wallet-to-wallet transfers, sales by UA, and what happens after the NFT is held by someone other than UA. It also covers what the ledger records, when the royalty is paid, and how the website can tell which stones have sold.

The consignment flow (merchant holds the NFT, UA sells through an allowance, 70 / 30 split) is described in [consignment.md](consignment.md). Token creation is in [create-nft-token.md](create-nft-token.md) and minting in [mint-nft-token.md](mint-nft-token.md).

## What Defines an Umiña NFT

An Umiña NFT is defined by two steps: **token creation** and **minting**. Once a serial is held by someone other than UA, it is an ordinary HTS NFT. It follows the general Hedera rules for NFTs and the rules set on the Umiña token.

### Token Creation

Token creation (`umina-achala-chain/src/token/lib.js`) sets the rules that apply to **every** serial: the royalty, the fee collector, the treasury and the keys.

Some properties can never change after creation:

- Token type (non-fungible), supply type (infinite) and decimals.
- Any key that was **not** set at creation. As far as we know, HTS does not allow adding one later (see [To Verify on Testnet](consignment.md#to-verify-on-testnet)). The current token has no KYC key and no freeze key.

The token is built in `buildTokenTransaction` (`umina-achala-chain/src/token/lib.js`) and created by `src/token/createToken.js`. There are two token classes with the same rules: Umiña for stones and Achala for jewelry (`--token achala`, [chain #15](https://github.com/umina-achala/umina-achala-chain/issues/15)). Decisions on them are in [chain #1](https://github.com/umina-achala/umina-achala-chain/issues/1) and [chain #6](https://github.com/umina-achala/umina-achala-chain/issues/6).

Everything else can be changed later by the matching key, as long as UA keeps that key. A key can also be removed permanently, which locks that property for good.

### Minting: How Much Can Vary

Different mint scripts can vary **only the serial's metadata**: at most 100 bytes, in UA's case an IPFS URI that points to the metadata JSON (`TokenMintTransaction().setMetadata(...)` in `umina-achala-chain/src/mint/mintNFT.js`, which mints both classes). The JSON can differ from stone to stone (gem type, certificates, images, schema version).

Everything else is token-wide and the same for every serial: royalty, fallback fee, fee collector and keys. A mint script cannot give one serial a different royalty. A class of items that needs different on-chain rules needs a **separate token**, as done for Achala jewelry ([chain #15](https://github.com/umina-achala/umina-achala-chain/issues/15)).

### What UA Can Still Change After a Sale

The current token sets six keys. While UA holds them, UA keeps these powers over **every** serial, including serials held by buyers:

| Key | What UA can do |
|---|---|
| Admin | Update the token's name, symbol, memo and keys, or delete the token |
| Fee schedule | Change the royalty, add or change a fallback fee, change the fee collector |
| Pause | Stop all transfers of every serial |
| Wipe | Remove a serial from any holder's account |
| Metadata | **Not set** ([chain #1](https://github.com/umina-achala/umina-achala-chain/issues/1)): NFT metadata is immutable, and this key can never be added |
| Supply | Mint more serials, or burn serials held by the Treasury |

`src/token/adminToken.js` uses the fee schedule, pause, wipe and burn keys ([chain #21](https://github.com/umina-achala/umina-achala-chain/issues/21)); wipe and burn record a `BURN` event with a reason.

**Trust trade-off.** Marketplaces such as SentX and explorers such as HashScan show these keys. Buyers can see a wipe, pause or metadata key as a risk: UA could take back or change the NFT they bought. Decided and in the code ([chain #1](https://github.com/umina-achala/umina-achala-chain/issues/1)): the wipe key is **kept**, there is **no** metadata key, and testnet uses one key for all roles. Separate keys or multi-signature for mainnet are still open ([chain #6](https://github.com/umina-achala/umina-achala-chain/issues/6)).

## Transferring an NFT

### Coin Transfer Compared With NFT Transfer

On Hedera both are a `TransferTransaction`. The differences:

| | Coin (HBAR) | NFT (HTS) |
|---|---|---|
| What moves | An amount | One serial, addressed by token ID and serial number |
| Receiver setup | None | The receiver must **associate** the token first, unless the account has a free auto-association slot |
| Who signs | Sender | The current holder, or a spender the holder gave an NFT allowance to |
| Custom fees | None | The token's royalty and fallback fee can apply (see [When the Royalty Is Paid](#when-the-royalty-is-paid)) |

### Wallet to Wallet (No Code)

A holder sending an Umiña NFT to another account needs no code. Everything happens in the two wallets (HashPack, Kabila, or the wallet used with SentX):

1. The receiver associates the Umiña token in their wallet, usually with an "Associate token" button. This step is skipped if the account has a free auto-association slot, which many newer wallet accounts have by default.
2. The sender opens the NFT, chooses **Send**, enters the receiver's account ID (`0.0.xxxx`) and approves. The sender pays a small network fee in HBAR.

A transfer with no HBAR attached pays no royalty; the receiver pays the fallback fee instead (about USD 3 in HBAR, see [When the Royalty Is Paid](#when-the-royalty-is-paid)). Nothing is recorded in UA's HCS topic unless someone records it with `recordEvent.js` (see [UA Selling Its Own Items Wallet to Wallet](#ua-selling-its-own-items-wallet-to-wallet)).

Listing an NFT for sale on SentX is different from sending it: the holder gives SentX an NFT allowance, the same mechanism UA uses for consignment.

### In UA's Code

The same transfer, as UA's scripts do it (see [consignment.md](consignment.md#code-samples)):

- `associateToken(...)` in `src/consignment/lib.js`: the receiver associates the token.
- `TransferTransaction().addNftTransfer(nftId, from, to)` in `src/consignment/consignToMerchant.js` and `src/sale/sellOwnItem.js`: a plain transfer, signed by the holder.
- `addApprovedNftTransfer(nftId, from, to)` in `src/consignment/sellConsigned.js`: the same transfer made by a spender through an allowance.

## After the NFT Leaves UA

### History on the Ledger

When UA no longer holds the NFT, UA no longer writes its history to HCS. The ledger still records every transfer. Anyone can see the full ownership chain of a serial on a mirror node or on HashScan (`https://hashscan.io/mainnet/token/<tokenId>/nft/<serial>`).

UA can keep posting stone-related events (re-certification, for example) to HCS for any serial, whoever holds it.

### Is the Price on the Ledger?

The ledger records each transaction in full, including every HBAR and token movement in it. Whether the price is visible depends on how the sale was paid:

| How the sale was paid | What the ledger shows |
|---|---|
| Atomic: payment and NFT in one transaction (UA's `sellConsigned.js`, marketplace sales) | The price. One transaction ID shows the buyer's HBAR debit, the seller's credit, the royalty to the fee collector, and the NFT moving |
| HBAR in a separate transaction | Both transactions, but nothing links them. They look like an unrelated HBAR transfer and NFT transfer |
| Off-chain (fiat, cash) | Only the NFT moving. No price |
| Through a marketplace smart contract | The transfers, as part of the contract call. Visible, but harder to read than a plain transfer |

How SentX settles a sale has not been checked. Open one SentX sale of an Umiña serial on HashScan to see it.

### What HCS Adds

For an atomic sale, the price is on the ledger even without HCS. UA's HCS `SALE` message adds meaning and convenience: it labels the event as a sale, names the merchant and buyer, states the 70 / 30 split explicitly and links the sale transaction ID, in one record UA controls. Someone reading the raw ledger would have to work that out themselves.

For a sale paid in separate transactions, the HCS record is the **only** thing that links the payment to the NFT transfer.

## When the Royalty Is Paid

UA does **not** receive a royalty on every transfer. HTS charges the royalty only when fungible value (HBAR or a fungible token) moves in the same transaction as the NFT.

| Transfer | Royalty (5%, fallback fee about USD 3 in HBAR) |
|---|---|
| Sale on SentX or another marketplace that settles on-chain | Yes, 5% |
| Consigned sale through `sellConsigned.js` | Yes, 5%, taken from the merchant's share (see [consignment.md](consignment.md#step-4-primary-sale-70--30-split)) |
| Gift, or sale paid in a separate transaction or off-chain | No royalty; the **receiver pays the fallback fee** |
| UA's own sale from the Treasury | Expected none: the Treasury is the fee collector. To verify on testnet |

**Fallback fee.** A fallback fee is a fixed fee charged to the **receiver** when no fungible value moves with the NFT. It reduces the leak, with limits:

- It is charged on genuine gifts too.
- It is a fixed amount, not 5% of the real price, so it cannot capture a price paid off-chain.
- **Decided:** a fallback fee of about USD 3, set in HBAR on the day the token is created ([chain #1](https://github.com/umina-achala/umina-achala-chain/issues/1), [chain #6](https://github.com/umina-achala/umina-achala-chain/issues/6)). **In the code:** `buildRoyaltyFee` (`src/token/lib.js`) adds it from `FALLBACK_FEE_HBAR` (38 HBAR on testnet).
- The USD value drifts with the HBAR price. `node src/token/adminToken.js fees --fallback-hbar <hbar>` updates it with the Fee Schedule Key ([chain #21](https://github.com/umina-achala/umina-achala-chain/issues/21)).

## UA Sales: What Can Happen in a Wallet

### Why a Wallet Alone Cannot Make UA's Sale

A consigned sale (`sellConsigned.js`) is one atomic transaction that:

- takes the price in HBAR from the buyer,
- splits it 70% to the merchant and 30% to UA,
- moves the NFT from the merchant to the buyer, using the allowance the merchant gave UA.

Hedera requires the payer of an allowance transfer to be the spender, so UA must pay for and sign the transaction, and the buyer signs too. Wallet interfaces only offer a plain Send. No wallet screen can build this transaction, so something has to build it and hand it to the wallets to sign.

### Sale Options

| Option | How it works | Trade-off |
|---|---|---|
| **1. Buyer pays from their wallet on UA's site** | UA's site and backend build the sale transaction and sign it with UA's key. The buyer's wallet opens through WalletConnect, shows the transaction, and the buyer approves | Feels like a wallet sale to the buyer. Atomic, keeps the 70 / 30 split and the HCS record. Needs wallet connection on the site and a backend endpoint |
| **2. List on SentX** | The NFT is listed and bought on SentX. No UA code | SentX pays the seller, so UA's 30% does not happen automatically. UA receives only the royalty. No HCS record |
| **3. Manual wallet transfers** | The buyer sends HBAR, then the holder sends the NFT | No code, but not atomic: one side trusts the other. The split is not enforced. No HCS record |

**Recommended:** Option 1. **Decision not made yet.**

What Option 1 needs:

- Wallet connection (`@hashgraph/hedera-wallet-connect`) in `umina-achala-web`. The site has none yet ([web #58](https://github.com/umina-achala/umina-achala-web/issues/58)).
- A small backend endpoint that holds UA's key and builds and signs the sale transaction. UA's key must not be in the browser. The site is currently static, so this is new infrastructure ([web #116](https://github.com/umina-achala/umina-achala-web/issues/116)).
- The sale transaction itself: `sellConsigned.js` for consigned items, and a Treasury sale for UA's own items ([chain #19](https://github.com/umina-achala/umina-achala-chain/issues/19)).
- The buyer's account must be associated with the token, which the wallet can do.

### UA Selling Its Own Items Wallet to Wallet

For NFTs UA holds itself in the Treasury, UA can sell wallet to wallet with no code except the HCS record:

1. The buyer associates the Umiña token, or has a free auto-association slot.
2. The buyer sends the HBAR to UA.
3. UA confirms the payment on HashScan, then sends the NFT from the Treasury wallet.

What this leaves out:

| Gap | What is needed |
|---|---|
| **Trust and order** | Two separate transactions, not one atomic one. Payment first protects UA |
| **HCS record** | `node src/record/recordEvent.js sale <serial> --seller <Treasury> --buyer <id> --nft-tx <tx> --payment on_chain --payment-tx <tx> [--sold-for <hbar>]` checks both transactions on the mirror node (the NFT moved, the buyer paid, the amount matches) and posts `SALE`, refusing a duplicate ([chain #8](https://github.com/umina-achala/umina-achala-chain/issues/8)). The atomic alternative is `src/sale/sellOwnItem.js` ([chain #19](https://github.com/umina-achala/umina-achala-chain/issues/19)) |
| **Website listing** | The site does not know the stone sold (see [Website: Showing Sold Stones](#website-showing-sold-stones), [web #115](https://github.com/umina-achala/umina-achala-web/issues/115)) |
| **Other listings** | If the stone is also listed on SentX, delist it, or confirm SentX drops the listing once the NFT moves |
| **Treasury key in a wallet** | Selling from a wallet means the Treasury key is in HashPack or similar on a device. That key also collects royalties and holds unsold stock. A hardware wallet is safer |
| **Royalty** | None, but Treasury sales are expected to be exempt anyway |

This does **not** work for consigned NFTs. They are in the merchant's account, and UA only has an allowance, which wallets cannot spend. Either the merchant sends the NFT, or UA uses `sellConsigned.js` or Option 1. A manual consigned sale also means paying the merchant's 70% in a separate transfer, which the HCS record should link too.

## Website: Showing Sold Stones

### Current State

- The marketplace reads its listings from `frontend/src/data/uminas.json`, bundled into the build. It has no sale status and nothing on the site checks the chain. After any sale (wallet to wallet, SentX or `sellConsigned.js`), the stone stays listed until the JSON is edited and the site is redeployed.
- None of the 30 stones in `uminas.json` has a `tokenId` or `serialNumber`, and the `Umina` type (`frontend/src/types/umina.ts`) does not define them. `StoneModal.tsx` already reads them for its HashScan link, so that link never shows.
- `StoneModal.tsx` hardcodes `mainnet` in the HashScan link.

Two ways to mark sold stones:

1. **By hand:** add a status to each stone in `uminas.json`, set it after each sale, and redeploy. Simple, but easy to forget, and it lags behind sales.
2. **Ask the chain:** the site checks each serial's current holder on the mirror node. **Recommended:** it catches every sale wherever it happened, with no step after selling.

### Mirror Node Ownership Check

Tracked in [web #115](https://github.com/umina-achala/umina-achala-web/issues/115).

**Prerequisite:** each minted stone in `uminas.json` gets its `tokenId` and `serialNumber`, and the `Umina` type defines them ([web #89](https://github.com/umina-achala/umina-achala-web/issues/89)). Stones that are not minted have nothing to check.

1. **One request for the whole collection.** When the marketplace loads, the site fetches the token's NFTs from the public mirror node:

   ```
   GET https://mainnet.mirrornode.hedera.com/api/v1/tokens/<tokenId>/nfts?limit=100
   ```

   Each entry has `serial_number`, the current holder `account_id`, and `deleted`. Follow `links.next` while there are more pages. No per-serial requests are needed.

2. **Decide the status.** A config list of seller accounts holds the Treasury and any merchants with consigned stones.

   | Condition | Status |
   |---|---|
   | Held by a seller account | Available |
   | Held by any other account | Sold |
   | `deleted` is true (burned or wiped) | Unavailable |
   | No serial in `uminas.json` | Not minted |

3. **Show it.** Cards get a "Sold" badge, and a filter shows only available stones. Sold stones stay visible as part of the gallery and provenance, and keep their HashScan link. The stone detail hides any buy or contact action for a sold stone.

### Where It Goes in the Code

| File | Change |
|---|---|
| `frontend/src/config.ts` | Network (mainnet or testnet), mirror node URL, token ID, seller account list. Also used by the HashScan link in `StoneModal.tsx:66` instead of the hardcoded `mainnet` ([web #75](https://github.com/umina-achala/umina-achala-web/issues/75)) |
| `frontend/src/types/umina.ts`, `data/uminas.json` | `tokenId` and `serialNumber` per minted stone |
| New hook, `useOwnership` | Fetches once per page load and returns serial → status, with loading and error state |
| `data/UminaFacade.ts` | Unchanged. It stays the static data source, and the hook adds status on top, so existing tests keep working |
| `Marketplace.tsx`, `Card.tsx`, `Filters.tsx`, `StoneModal.tsx` | Sold badge, "available only" filter, no buy action on sold stones |
| `locales/en.json`, `locales/es.json` | Text for "Sold" and the filter |

The public mirror node needs no API key and should allow requests from the browser, so this works on the current static site with no backend. Confirm this on the first test request.

### Failure Behavior

If the mirror node is slow or down, the site shows the listing exactly as it does today, with no badges. It does not block the page or mark everything sold. A sale shows up once the mirror node has caught up, normally within seconds.

**What this does not do:** record the sale (that is still the HCS script), or keep the seller list up to date when merchants join or leave consignment.

**Relation to the display topic.** The data model ([chain #10](https://github.com/umina-achala/umina-achala-chain/issues/10)) plans an HCS display topic (`LISTED`, `UNLISTED`) for what UA puts up for sale. The ownership check also catches sales that happened outside UA. Whether the site uses both is open in [web #115](https://github.com/umina-achala/umina-achala-web/issues/115).

**Status:** not built.

## Open Decisions

**Decided and in the code** ([chain #1](https://github.com/umina-achala/umina-achala-chain/issues/1), [chain #6](https://github.com/umina-achala/umina-achala-chain/issues/6)):

- No metadata key: NFT metadata is immutable.
- A fallback fee of about USD 3 in HBAR, set from `FALLBACK_FEE_HBAR` when the token is created and changeable later with `adminToken.js fees`.
- The wipe key is kept. Testnet uses one key for all roles.
- Tokens and HCS topics are created by the project owner, once all Phase 2A code is written (decided 2026-11-05). None exist yet: `.env.testnet` holds placeholder IDs.

**Decision not made yet:**

- Separate keys or multi-signature for mainnet ([chain #6](https://github.com/umina-achala/umina-achala-chain/issues/6)).
- How UA sells: site plus wallet (Option 1), SentX, or manual wallet transfers. See [Sale Options](#sale-options).
- Where the Treasury key lives if UA sells from a wallet.
- Whether the website uses the ownership check, the display topic, or both ([web #115](https://github.com/umina-achala/umina-achala-web/issues/115)).

## Code and Issue Reference

Chain code is in `umina-achala-chain`. "On fork" means on the `main` branch of the fork (origin), not yet on upstream `umina-achala/umina-achala-chain`, whose `main` is still the initial import; an issue closes only when its work reaches upstream. State as of 2026-11-06; "On branch" means on the fork's `achala-token-mint` branch, not merged yet.

| Topic | Code | Status | Issue |
|---|---|---|---|
| Token classes, keys, royalty and fallback fee | `src/token/lib.js` (`TOKEN_CLASSES`, `buildTokenTransaction`, `buildRoyaltyFee`), `src/token/createToken.js [--token umina\|achala]` | On fork | [chain #1](https://github.com/umina-achala/umina-achala-chain/issues/1), [#15](https://github.com/umina-achala/umina-achala-chain/issues/15) (Achala: on branch) |
| Fee update, pause, wipe, burn | `src/token/adminToken.js` | On fork | [chain #21](https://github.com/umina-achala/umina-achala-chain/issues/21) |
| HCS topics with submit and admin keys | `src/topic/createTopic.js`, `src/topic/lib.js` | On fork (renamed from `createUminaTopic.js` on branch) | [chain #9](https://github.com/umina-achala/umina-achala-chain/issues/9) |
| Event envelope, submit-key signing, receipt check | `src/hcs.js` (`buildEvent`, `submitEvent`) | On fork | [chain #17](https://github.com/umina-achala/umina-achala-chain/issues/17), [#10](https://github.com/umina-achala/umina-achala-chain/issues/10) |
| Mirror node reads (topic events, transactions, NFT holder, account key) | `src/mirror.js` | On fork | [chain #8](https://github.com/umina-achala/umina-achala-chain/issues/8), [#20](https://github.com/umina-achala/umina-achala-chain/issues/20) |
| Mint: pin, mint, `COMPLIANCE` and `MINT` events, duplicate `stone_id` check | `src/mint/mintNFT.js`, `src/mint/lib.js` | On fork (as `mintUminaNFT.js`; renamed `mintNFT.js` on branch) | [chain #3](https://github.com/umina-achala/umina-achala-chain/issues/3), [#20](https://github.com/umina-achala/umina-achala-chain/issues/20) |
| Photos: master per photo, 1080 px display version for `image`, EXIF stripped | `src/mint/images.js` | On fork | [chain #7](https://github.com/umina-achala/umina-achala-chain/issues/7) |
| Stone types, certified-only rules, RUC / REINFO / concession checks | `schema/stone-types.json`, `src/mint/stoneTypes.js`, `src/mint/metadataValidator.js`, `src/mint/importIma.js` | On fork | [chain #14](https://github.com/umina-achala/umina-achala-chain/issues/14), [#5](https://github.com/umina-achala/umina-achala-chain/issues/5) |
| Association | `src/consignment/lib.js` (`associateToken`), `src/consignment/associateToken.js` | On fork | |
| Consignment: transfer, allowance, `ISSUED_TO_OWNER` event | `src/consignment/consignToMerchant.js` | On fork | [chain #18](https://github.com/umina-achala/umina-achala-chain/issues/18) |
| Ending a consignment: revoke, return, `TRANSFER` event | `src/consignment/endConsignment.js` | On fork | [chain #18](https://github.com/umina-achala/umina-achala-chain/issues/18) |
| Consigned 70 / 30 sale with `SALE` event | `src/consignment/sellConsigned.js` | On fork | |
| Atomic sale of UA's own item from the Treasury | `src/sale/sellOwnItem.js` | On fork | [chain #19](https://github.com/umina-achala/umina-achala-chain/issues/19) |
| Recording sales made elsewhere and deliveries (`SALE`, `DELIVERED`) | `src/record/recordEvent.js` | On fork | [chain #8](https://github.com/umina-achala/umina-achala-chain/issues/8) |
| Achala schema and mint: stone checks, `TRANSFORMATION` per stone | `schema/hip412-achala-schema.json`, `src/mint/achala.js`, `src/mint/mintNFT.js --token achala` | On branch | [chain #16](https://github.com/umina-achala/umina-achala-chain/issues/16) |
| Selling an Achala with its Umiña NFTs (N+1 in one transfer) | none | Not built | [chain #16](https://github.com/umina-achala/umina-achala-chain/issues/16) |
| Achala with many stones: accent stones without an NFT each | none | Not decided | [chain #23](https://github.com/umina-achala/umina-achala-chain/issues/23) |
| Seller model (consignment or not) | | Open | [web #90](https://github.com/umina-achala/umina-achala-web/issues/90) |
| Wallet connection and payment | none | Not built | [web #58](https://github.com/umina-achala/umina-achala-web/issues/58) |
| Backend that signs UA's side of a sale | none | Not built | [web #116](https://github.com/umina-achala/umina-achala-web/issues/116) |
| `tokenId` / `serialNumber` in the site data | `frontend/src/types/umina.ts`, `frontend/src/data/uminas.json` | Not built | [web #89](https://github.com/umina-achala/umina-achala-web/issues/89) |
| HashScan link hardcodes mainnet | `frontend/src/components/StoneModal.tsx` | Not fixed | [web #75](https://github.com/umina-achala/umina-achala-web/issues/75) |
| Sold stones on the marketplace (mirror node ownership check) | `frontend/src/data/UminaFacade.ts`, `components/Marketplace.tsx` | Not built | [web #115](https://github.com/umina-achala/umina-achala-web/issues/115) |
