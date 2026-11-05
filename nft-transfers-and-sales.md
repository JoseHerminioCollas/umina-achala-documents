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

## Overview

This document explains how an Umiña NFT moves between accounts once it exists: plain wallet-to-wallet transfers, sales by UA, and what happens after the NFT is held by someone other than UA. It also covers what the ledger records, when the royalty is paid, and how the website can tell which stones have sold.

The consignment flow (merchant holds the NFT, UA sells through an allowance, 70 / 30 split) is described in [consignment.md](consignment.md). Token creation is in [create-nft-token.md](create-nft-token.md) and minting in [mint-nft-token.md](mint-nft-token.md).

## What Defines an Umiña NFT

An Umiña NFT is defined by two steps: **token creation** and **minting**. Once a serial is held by someone other than UA, it is an ordinary HTS NFT. It follows the general Hedera rules for NFTs and the rules set on the Umiña token.

### Token Creation

Token creation (`umina-achala-chain/src/uminaToken.js`) sets the rules that apply to **every** serial: the royalty, the fee collector, the treasury and the keys.

Some properties can never change after creation:

- Token type (non-fungible), supply type (infinite) and decimals.
- Any key that was **not** set at creation. As far as we know, HTS does not allow adding one later (see [To Verify on Testnet](consignment.md#to-verify-on-testnet)). The current token has no KYC key and no freeze key.

Everything else can be changed later by the matching key, as long as UA keeps that key. A key can also be removed permanently, which locks that property for good.

### Minting: How Much Can Vary

Different mint scripts can vary **only the serial's metadata**: at most 100 bytes, in UA's case an IPFS URI that points to the metadata JSON. The JSON can differ from stone to stone (gem type, certificates, images, schema version).

Everything else is token-wide and the same for every serial: royalty, fallback fee, fee collector and keys. A mint script cannot give one serial a different royalty. A class of stones that needs different on-chain rules needs a **separate token**.

### What UA Can Still Change After a Sale

The current token sets six keys. While UA holds them, UA keeps these powers over **every** serial, including serials held by buyers:

| Key | What UA can do |
|---|---|
| Admin | Update the token's name, symbol, memo and keys, or delete the token |
| Fee schedule | Change the royalty, add or change a fallback fee, change the fee collector |
| Pause | Stop all transfers of every serial |
| Wipe | Remove a serial from any holder's account |
| Metadata | Rewrite a serial's metadata after it is sold (HIP-657) |
| Supply | Mint more serials, or burn serials held by the Treasury |

**Trust trade-off.** Marketplaces such as SentX and explorers such as HashScan show these keys. Buyers can see a wipe, pause or metadata key as a risk: UA could take back or change the NFT they bought. Whether to keep, move to multi-signature, or remove each key is a decision for before mainnet (see [Open Decisions](#open-decisions)).

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

With the current token (no fallback fee), a transfer with no HBAR attached pays no royalty, and nothing is recorded in UA's HCS topic.

Listing an NFT for sale on SentX is different from sending it: the holder gives SentX an NFT allowance, the same mechanism UA uses for consignment.

### In UA's Code

The same transfer, as UA's scripts do it (see [consignment.md](consignment.md#code-samples)):

- `associateToken(...)` in `consignment.js`: the receiver associates the token.
- `TransferTransaction().addNftTransfer(nftId, from, to)` in `consignToMerchant.js`: a plain transfer, signed by the holder.
- `addApprovedNftTransfer(nftId, from, to)` in `sellConsigned.js`: the same transfer made by a spender through an allowance.

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

| Transfer | Royalty (current token: 5%, no fallback fee) |
|---|---|
| Sale on SentX or another marketplace that settles on-chain | Yes, 5% |
| Consigned sale through `sellConsigned.js` | Yes, 5%, taken from the merchant's share (see [consignment.md](consignment.md#step-4-primary-sale-70--30-split)) |
| Gift, or sale paid in a separate transaction or off-chain | **None** |
| UA's own sale from the Treasury | Expected none: the Treasury is the fee collector. To verify on testnet |

**Fallback fee.** A fallback fee is a fixed fee charged to the **receiver** when no fungible value moves with the NFT. It reduces the leak, with limits:

- It is charged on genuine gifts too.
- It is a fixed amount, not 5% of the real price, so it cannot capture a price paid off-chain.
- The current token has none. The Fee Schedule Key can add one later, as long as UA keeps that key.

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

- Wallet connection (`@hashgraph/hedera-wallet-connect`) in `umina-achala-web`. The site has none yet.
- A small backend endpoint that holds UA's key and builds and signs the sale transaction. UA's key must not be in the browser. The site is currently static (GitHub Pages), so this is new infrastructure.
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
| **HCS record** | A script that takes both transaction IDs, checks them on the mirror node (amount, buyer, the NFT actually moved), then posts the `SALE` message. Not built |
| **Website listing** | The site does not know the stone sold (see [Website: Showing Sold Stones](#website-showing-sold-stones)) |
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

**Prerequisite:** each minted stone in `uminas.json` gets its `tokenId` and `serialNumber`, and the `Umina` type defines them. Stones that are not minted have nothing to check.

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
| `frontend/src/config.ts` | Network (mainnet or testnet), mirror node URL, token ID, seller account list. Also used by the HashScan link in `StoneModal.tsx` instead of the hardcoded `mainnet` |
| `frontend/src/types/umina.ts`, `data/uminas.json` | `tokenId` and `serialNumber` per minted stone |
| New hook, `useOwnership` | Fetches once per page load and returns serial → status, with loading and error state |
| `data/UminaFacade.ts` | Unchanged. It stays the static data source, and the hook adds status on top, so existing tests keep working |
| `Marketplace.tsx`, `Card.tsx`, `Filters.tsx`, `StoneModal.tsx` | Sold badge, "available only" filter, no buy action on sold stones |
| `locales/en.json`, `locales/es.json` | Text for "Sold" and the filter |

The public mirror node needs no API key and should allow requests from the browser, so this works on the current static site with no backend. Confirm this on the first test request.

### Failure Behavior

If the mirror node is slow or down, the site shows the listing exactly as it does today, with no badges. It does not block the page or mark everything sold. A sale shows up once the mirror node has caught up, normally within seconds.

**What this does not do:** record the sale (that is still the HCS script), or keep the seller list up to date when merchants join or leave consignment.

**Status:** not built.

## Open Decisions

**Decision not made yet** on any of these.

- Which token keys to keep, move to multi-signature, or remove before mainnet (admin, fee schedule, pause, wipe, metadata, supply). See [What UA Can Still Change After a Sale](#what-ua-can-still-change-after-a-sale).
- Whether to add a fallback fee, and its amount. See [When the Royalty Is Paid](#when-the-royalty-is-paid).
- How UA sells on mainnet: site plus wallet (Option 1), SentX, or manual wallet transfers. See [Sale Options](#sale-options).
- Where the Treasury key lives if UA sells from a wallet.
- Whether to build the mirror node ownership check on the website.
