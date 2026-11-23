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
  - [Website: Showing What Is for Sale](#website-showing-what-is-for-sale)
    - [Current State](#current-state)
    - [The Catalog: Built From the Chain](#the-catalog-built-from-the-chain)
    - [Commands: List, Rebuild, Check](#commands-list-rebuild-check)
    - [Keeping the Catalog Current After a Sale](#keeping-the-catalog-current-after-a-sale)
    - [Failure Behavior](#failure-behavior)
  - [Open Decisions](#open-decisions)
  - [Code and Issue Reference](#code-and-issue-reference)

## Overview

> **Status: Phase 2 (testnet), in progress.** The code referred to here is in `umina-achala-chain` (on `main`, fork and upstream since 2026-11-09) and `umina-achala-web`, as of 2026-11-06. Code is referred to by file and function, not line number. The testnet tokens and topics were created on 2026-11-10 ([chain #22](https://github.com/umina-achala/umina-achala-chain/issues/22)); minting and sales are next. Work that is not built is tracked in the issues listed in [Code and Issue Reference](#code-and-issue-reference).

This document explains how an Umiña NFT moves between accounts once it exists: plain wallet-to-wallet transfers, sales by UA, and what happens after the NFT is held by someone other than UA. It also covers what the ledger records, when the royalty is paid, and how the website shows only what UA has for sale.

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

**Trust trade-off.** Marketplaces such as SentX and explorers such as HashScan show these keys. Buyers can see a wipe, pause or metadata key as a risk: UA could take back or change the NFT they bought. Decided and in the code ([chain #1](https://github.com/umina-achala/umina-achala-chain/issues/1)): the wipe key is **kept**, there is **no** metadata key, and testnet uses one key for all roles. Separate keys or multi-signature for mainnet are still open ([chain #32](https://github.com/umina-achala/umina-achala-chain/issues/32), Phase 3).

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

When UA no longer holds the NFT, UA no longer writes its history to HCS. The ledger still records every transfer. Anyone can see the full ownership chain of a serial on a mirror node or on HashScan (`https://hashscan.io/mainnet/token/<tokenId>/<serial>`).

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
| Consigned sale through `sellConsigned.js` | Yes, 5%, taken from the merchant's share; part of UA's 30% (decided 2026-11-09, see [consignment.md](consignment.md#step-4-primary-sale-70--30-split)) |
| Gift, or sale paid in a separate transaction or off-chain | No royalty; the **receiver pays the fallback fee** |
| UA's own sale from the Treasury | Expected none: the Treasury is the fee collector. To verify on testnet |

**Fallback fee.** A fallback fee is a fixed fee charged to the **receiver** when no fungible value moves with the NFT. It reduces the leak, with limits:

- It is charged on genuine gifts too.
- It is a fixed amount, not 5% of the real price, so it cannot capture a price paid off-chain.
- **Decided:** a fallback fee of about USD 3, set in HBAR on the day the token is created ([chain #1](https://github.com/umina-achala/umina-achala-chain/issues/1), [chain #6](https://github.com/umina-achala/umina-achala-chain/issues/6)). **In the code:** `buildRoyaltyFee` (`src/token/lib.js`) adds it from `FALLBACK_FEE_HBAR` (45 HBAR on testnet, from 2026-11-10).
- The USD value drifts with the HBAR price. `NETWORK=testnet node src/token/adminToken.js fees --fallback-hbar <hbar>` updates it with the Fee Schedule Key ([chain #21](https://github.com/umina-achala/umina-achala-chain/issues/21)).

## UA Sales: What Can Happen in a Wallet

### Why a Wallet Alone Cannot Make UA's Sale

A consigned sale (`sellConsigned.js`) is one atomic transaction that:

- takes the price in HBAR from the buyer,
- splits it so the merchant ends with 70% and UA with 30%, the royalty included (decided 2026-11-09, [chain #30](https://github.com/umina-achala/umina-achala-chain/issues/30)),
- moves the NFT from the merchant to the buyer, using the allowance the merchant gave UA.

Hedera requires the payer of an allowance transfer to be the spender, so UA must pay for and sign the transaction, and the buyer signs too. Wallet interfaces only offer a plain Send. No wallet screen can build this transaction, so something has to build it and hand it to the wallets to sign.

### Sale Options

| Option | How it works | Trade-off |
|---|---|---|
| **1. Buyer pays from their wallet on UA's site** | UA's site and backend build the sale transaction and sign it with UA's key. The buyer's wallet opens through WalletConnect, shows the transaction, and the buyer approves | Feels like a wallet sale to the buyer. Atomic, keeps the 70 / 30 split and the HCS record. Needs wallet connection on the site and a backend endpoint |
| **2. List on SentX** | The NFT is listed and bought on SentX. No UA code | SentX pays the seller, so UA's 30% does not happen automatically. UA receives only the royalty. No HCS record |
| **3. Manual wallet transfers** | The buyer sends HBAR, then the holder sends the NFT | No code, but not atomic: one side trusts the other. The split is not enforced. No HCS record |

**Decided (2026-11-09):** Option 1 ([web #90](https://github.com/umina-achala/umina-achala-web/issues/90)).

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
| **HCS record** | `NETWORK=testnet node src/record/recordEvent.js sale <serial> --seller <Treasury> --buyer <id> --nft-tx <tx> --payment on_chain --payment-tx <tx> [--sold-for <hbar>]` checks both transactions on the mirror node (the NFT moved, the buyer paid, the amount matches) and posts `SALE`, refusing a duplicate ([chain #8](https://github.com/umina-achala/umina-achala-chain/issues/8)). The atomic alternative is `src/sale/sellOwnItem.js` ([chain #19](https://github.com/umina-achala/umina-achala-chain/issues/19)) |
| **Website listing** | Post `UNLISTED` to the display topic and rebuild the catalog; the stone is gone on the next page load (see [Keeping the Catalog Current After a Sale](#keeping-the-catalog-current-after-a-sale)) |
| **Other listings** | If the stone is also listed on SentX, delist it, or confirm SentX drops the listing once the NFT moves |
| **Treasury key in a wallet** | Selling from a wallet means the Treasury key is in HashPack or similar on a device. That key also collects royalties and holds unsold stock. A hardware wallet is safer |
| **Royalty** | None, but Treasury sales are expected to be exempt anyway |

This does **not** work for consigned NFTs. They are in the merchant's account, and UA only has an allowance, which wallets cannot spend. Either the merchant sends the NFT, or UA uses `sellConsigned.js` or Option 1. A manual consigned sale also means paying the merchant's 70% in a separate transfer, which the HCS record should link too.

## Website: Showing What Is for Sale

**Decided 2026-11-19 ([web #15](https://github.com/umina-achala/umina-achala-web/issues/15)):** the site shows **only what UA has for sale**, and only testnet NFTs in Phase 2B. A sold item is removed; there are no "Sold" badges. The chain is the source of truth.

### Current State

The marketplace reads `catalog.json` at run time, built from the chain by the catalog builder: each Umiña and Achala UA has for sale, with its token ID, serial, metadata and price. It is only as current as its last build; see [Keeping the Catalog Current After a Sale](#keeping-the-catalog-current-after-a-sale).

### The Catalog: Built From the Chain

A script run by hand ([chain #44](https://github.com/umina-achala/umina-achala-chain/issues/44)) builds the site's `catalog.json` from the mirror node, IPFS and the two HCS topics; the site loads it at run time ([web #89](https://github.com/umina-achala/umina-achala-web/issues/89)). An NFT is in the catalog when all of these hold:

- it is not deleted (burned or wiped);
- a **seller account** holds it: the Treasury, or a consigning merchant;
- for a merchant's NFT, UA's **NFT allowance** on the serial still exists. A merchant who revoked it is selling elsewhere, and UA cannot sell it;
- its latest display event is `LISTED`, which carries the price ([chain #43](https://github.com/umina-achala/umina-achala-chain/issues/43)).

Anything else is left out: sold, transferred, burned, unlisted, or a stone mounted in an Achala (listed with its Achala).

### Commands: List, Rebuild, Check

The order is always **list → rebuild the catalog → publish it**. The site shows the change on the next page load, with no site build. The scripts are in the chain repository; `NETWORK` picks `testnet` or `mainnet`. In the examples, `<serial>` is the NFT's serial number and `<price>` is in HBAR.

**1. See what the Treasury holds and what is listed.** It reads the mirror node and sends nothing:

```
NETWORK=testnet node src/display/displayEvent.js status
```

For the jewelry token:

```
NETWORK=testnet node src/display/displayEvent.js status --token achala
```

It prints one row per NFT the Treasury holds, with its state: `not listed`, `listed, 45 HBAR`, or `in Achala <stone_id>` for a stone mounted in a piece of jewelry.

**2. List, reprice or unlist.** Each is one HCS message to the display topic, signed with the topic's submit key; no NFT or HBAR moves. Run with `--dry-run` first: it makes every check and prints the message without sending it.

List an item, first as a dry run:

```
NETWORK=testnet node src/display/displayEvent.js listed <serial> --price <price> --dry-run
```

Then for real:

```
NETWORK=testnet node src/display/displayEvent.js listed <serial> --price <price>
```

List a piece of jewelry; its mounted stones go with it, at one price:

```
NETWORK=testnet node src/display/displayEvent.js listed <serial> --price <price> --token achala
```

Change a price: list it again with the new price (a new `LISTED`):

```
NETWORK=testnet node src/display/displayEvent.js listed <serial> --price <new-price>
```

Take an item down; the reason is public:

```
NETWORK=testnet node src/display/displayEvent.js unlisted <serial> --reason "<reason>"
```

Several at once, as `serial:price` pairs (one command over several lines):

```
for pair in 9:35 10:40 11:90; do
  NETWORK=testnet node src/display/displayEvent.js listed "${pair%%:*}" --price "${pair##*:}"
done
```

The script refuses: an NFT the Treasury does not hold, a burned one, a stone mounted in a piece of jewelry that still exists, the same price as the current listing, and unlisting an item that is not listed.

**3. Rebuild the catalog.** It reads the mirror node and IPFS and needs no key; if anything fails, the old file stays. `<out-dir>` is the folder the site serves `data/catalog.json` from (in development, the site's `public/data/`). It also saves two smaller copies of each photo under `<out-dir>/media/<cid>/`, a thumbnail for the cards and a display copy for the detail view, and a copy of each 3D model and PDF document, so visitors never load a file from a public IPFS gateway. A copy already on disk is never downloaded again, since an IPFS file never changes; the first build of a full catalog can take minutes:

```
NETWORK=testnet node src/catalog/buildCatalog.js <out-dir>
```

Print what would be written, without writing it:

```
NETWORK=testnet node src/catalog/buildCatalog.js <out-dir> --dry-run
```

**4. Publish it.** Copy `catalog.json` and `media/` to the live site's `data/` folder. A visitor sees it on the next page load: the site fetches the catalog without the browser's cached copy, and the server should send it with `Cache-Control: no-cache`. The copies never change, so the server can cache `data/media/` for a year. Serve `.glb` files as `model/gltf-binary`: Nginx's default `mime.types` has no entry for them.

**5. Check on the chain.** Open the display topic on HashScan (`https://hashscan.io/testnet/topic/<display-topic-id>/messages`), or read it from the mirror node with the messages decoded:

```
curl -s "https://testnet.mirrornode.hedera.com/api/v1/topics/<display-topic-id>/messages?limit=50" | jq -r '.messages[] | "\(.sequence_number) " + (.message | @base64d)'
```

Example output for one stone that was listed, repriced, taken down and listed again:

```
1 {"event":"LISTED","stone_id":"UMINA-2026-AG-01","serials":["15"],"price":"50","currency":"HBAR",…}
2 {"event":"LISTED","stone_id":"UMINA-2026-AG-01","serials":["15"],"price":"45",…}
3 {"event":"UNLISTED","stone_id":"UMINA-2026-AG-01","serials":["15"],"reason":"Photo retake",…}
4 {"event":"LISTED","stone_id":"UMINA-2026-AG-01","serials":["15"],"price":"45",…}
```

An item's state is its latest event: this stone is listed at 45 HBAR.

### Keeping the Catalog Current After a Sale

The catalog is only as current as its last build. How the site catches up depends on whether every change goes through the display topic.

**When every change goes through the display topic** (one seller, the Treasury, holds everything; items are listed and unlisted by script; sales happen only on the site): rebuild the catalog after each sale.

1. The buyer signs and the sale transaction succeeds.
2. The sale backend posts `UNLISTED` (reason `sold`) to the display topic, and `SALE` to the provenance topic.
3. It waits until the mirror node shows the `UNLISTED` (a few seconds after consensus); a rebuild started earlier would still list the item.
4. It runs the catalog builder, which writes a new `catalog.json` where the site serves it. Only one rebuild runs at a time, including one started by hand.
5. The buyer's browser hides the item at once; everyone else sees it gone on the next page load.

The browser reads only `catalog.json`: no mirror node calls when a page opens, so each visit costs one small file. The catalog stays the builder's output from the chain; nothing edits it. The backend must run where it can write the site's `data/` folder. Listings added or removed by script still need a rebuild by hand (or a scheduled one).

**When sellers can sell outside the site** (consignment: a merchant holds the NFT and UA has an allowance), the chain can change without a display event. Then the site checks the mirror node on each page load:

| What happened | What the mirror node shows | Site |
|---|---|---|
| Sold on UA's site | New holder | Hidden at once in the buyer's browser; for everyone on the next page load |
| Merchant sold on SentX, or sent the NFT wallet to wallet | New holder (UA's allowance is cleared by the transfer) | Hidden on the next page load |
| Merchant revoked UA's allowance | Same holder, no allowance to UA | Hidden on the next page load |
| Burned or wiped | `deleted` | Hidden |
| Merchant sold the physical piece and has not moved the NFT | Nothing changes | Still listed: the chain cannot know. The consignment agreement requires the merchant to tell UA, and UA posts `UNLISTED` |

Requests: `GET /api/v1/tokens/<tokenId>/nfts?limit=100` for each token (follow `links.next`), and `GET /api/v1/accounts/<merchant>/allowances/nfts?token.id=<tokenId>` for each merchant with listed items. The public mirror node needs no API key; confirm that it allows browser requests on the first test. Changes show within seconds of consensus. If the mirror node is slow or down, the site shows the catalog as built and does not block the page.

### Failure Behavior

The sale backend checks the holder (and, for a consigned item, the allowance) again before building a sale, so a stale listing cannot be sold. If the rebuild after a sale fails, the old catalog stays, and the next rebuild (after the next sale, or by hand) catches up.

**What this does not do:** keep the seller list up to date when merchants join or leave consignment.

## Open Decisions

**Decided and in the code** ([chain #1](https://github.com/umina-achala/umina-achala-chain/issues/1), [chain #6](https://github.com/umina-achala/umina-achala-chain/issues/6)):

- No metadata key: NFT metadata is immutable.
- A fallback fee of about USD 3 in HBAR, set from `FALLBACK_FEE_HBAR` when the token is created and changeable later with `adminToken.js fees`.
- The wipe key is kept. Testnet uses one key for all roles.
- Tokens and HCS topics are created by the project owner. On testnet, created 2026-11-10 ([chain #22](https://github.com/umina-achala/umina-achala-chain/issues/22)): Umiña [`0.0.11400486`](https://hashscan.io/testnet/token/0.0.11400486), Achala [`0.0.11400589`](https://hashscan.io/testnet/token/0.0.11400589), provenance topic [`0.0.11400670`](https://hashscan.io/testnet/topic/0.0.11400670), display topic [`0.0.11400772`](https://hashscan.io/testnet/topic/0.0.11400772).

**Decided (2026-11-09, [web #90](https://github.com/umina-achala/umina-achala-web/issues/90)):** the merchant owns a consigned item and holds its NFT and the physical item, and may sell it anywhere. UA's 30% (royalty included) applies to sales on UA's site; other on-chain sales pay the royalty; an off-chain sale ends with the merchant sending the NFT to the buyer, who pays the fallback fee. The merchant pays the artisan separately for labor, not in the sale. See [consignment.md](consignment.md#decisions-2026-11-09).

**Decided (2026-11-19, [web #15](https://github.com/umina-achala/umina-achala-web/issues/15)):** the website uses both. The display topic (`LISTED`) says what UA has for sale and at what price; the mirror node holder and allowance remove anything sold or no longer sellable. Only items for sale are shown, with no "Sold" badge. See [Website: Showing What Is for Sale](#website-showing-what-is-for-sale).

**Decision not made yet:**

- Separate keys or multi-signature for mainnet ([chain #32](https://github.com/umina-achala/umina-achala-chain/issues/32), Phase 3).
- Where the Treasury key lives if UA sells from a wallet.

## Code and Issue Reference

Chain code is in `umina-achala-chain`. "On `main`" means on the `main` branch of both the fork (origin) and upstream `umina-achala/umina-achala-chain`, which are the same since 2026-11-09. State as of 2026-11-11.

| Topic | Code | Status | Issue |
|---|---|---|---|
| Token classes, keys, royalty and fallback fee | `src/token/lib.js` (`TOKEN_CLASSES`, `buildTokenTransaction`, `buildRoyaltyFee`), `src/token/createToken.js [--token umina\|achala]` | On `main` | [chain #1](https://github.com/umina-achala/umina-achala-chain/issues/1), [#15](https://github.com/umina-achala/umina-achala-chain/issues/15) |
| Fee update, pause, wipe, burn | `src/token/adminToken.js` | On `main` | [chain #21](https://github.com/umina-achala/umina-achala-chain/issues/21) |
| HCS topics with submit and admin keys | `src/topic/createTopic.js`, `src/topic/lib.js` | On `main` (renamed from `createUminaTopic.js`) | [chain #9](https://github.com/umina-achala/umina-achala-chain/issues/9) |
| Event envelope, submit-key signing, receipt check | `src/hcs.js` (`buildEvent`, `submitEvent`) | On `main` | [chain #17](https://github.com/umina-achala/umina-achala-chain/issues/17), [#10](https://github.com/umina-achala/umina-achala-chain/issues/10) |
| Mirror node reads (topic events, transactions, NFT holder, account key) | `src/mirror.js` | On `main` | [chain #8](https://github.com/umina-achala/umina-achala-chain/issues/8), [#20](https://github.com/umina-achala/umina-achala-chain/issues/20) |
| Mint: pin, mint, `COMPLIANCE` and `MINT` events, duplicate `stone_id` check | `src/mint/mintNFT.js`, `src/mint/lib.js` | On `main` (renamed from `mintUminaNFT.js`) | [chain #3](https://github.com/umina-achala/umina-achala-chain/issues/3), [#20](https://github.com/umina-achala/umina-achala-chain/issues/20) |
| Photos: master per photo, 1080 px display version for `image`, EXIF stripped | `src/mint/images.js` | On `main` | [chain #7](https://github.com/umina-achala/umina-achala-chain/issues/7) |
| Stone types, certified-only rules, RUC / REINFO / concession checks | `schema/stone-types.json`, `src/mint/stoneTypes.js`, `src/mint/metadataValidator.js`, `src/mint/importIma.js` | On `main` | [chain #14](https://github.com/umina-achala/umina-achala-chain/issues/14), [#5](https://github.com/umina-achala/umina-achala-chain/issues/5) |
| Association | `src/consignment/lib.js` (`associateToken`), `src/consignment/associateToken.js` | On `main` | |
| Consignment: transfer, allowance, `ISSUED_TO_OWNER` event | `src/consignment/consignToMerchant.js` | On `main` | [chain #18](https://github.com/umina-achala/umina-achala-chain/issues/18) |
| Ending a consignment: revoke, return, `TRANSFER` event | `src/consignment/endConsignment.js` | On `main` | [chain #18](https://github.com/umina-achala/umina-achala-chain/issues/18) |
| Consigned 70 / 30 sale with `SALE` event | `src/consignment/sellConsigned.js` | On `main` | |
| Atomic sale of UA's own item from the Treasury | `src/sale/sellOwnItem.js` | On `main` | [chain #19](https://github.com/umina-achala/umina-achala-chain/issues/19) |
| Recording sales made elsewhere and deliveries (`SALE`, `DELIVERED`) | `src/record/recordEvent.js` | On `main` | [chain #8](https://github.com/umina-achala/umina-achala-chain/issues/8) |
| Achala schema and mint: stone checks, stones recorded in the Achala's `MINT` | `schema/hip412-achala-schema.json`, `src/mint/achala.js`, `src/mint/mintNFT.js --token achala` | On `main` | [chain #16](https://github.com/umina-achala/umina-achala-chain/issues/16) |
| Selling an Achala with its Umiña NFTs (N+1 in one transfer), and its `SALE` and `DELIVERED` | `src/sale/sellOwnItem.js --token achala`, `src/record/recordEvent.js --token achala`, `src/mint/achala.js` (`achalaSalePlan`) | On `main` | [chain #26](https://github.com/umina-achala/umina-achala-chain/issues/26) |
| Achala with many stones: accent stones without an NFT each | none | Not decided (Phase 2B) | [chain #23](https://github.com/umina-achala/umina-achala-chain/issues/23) |
| Consigning an Achala with its stones (consign, sell, end as one piece; `--token` on the three consignment scripts) | `src/consignment/lib.js` (`consignmentNfts`), the three consignment scripts | On the `refactor-v2` branch (2026-11-14), tested in the testnet run | [chain #28](https://github.com/umina-achala/umina-achala-chain/issues/28) |
| Reading the whole provenance topic on every run: cache or web indexer once the topic is large | `src/provenance.js` | After the testnet minting ([chain #22](https://github.com/umina-achala/umina-achala-chain/issues/22)) | [chain #37](https://github.com/umina-achala/umina-achala-chain/issues/37), from [#33](https://github.com/umina-achala/umina-achala-chain/issues/33) (closed) and [#29](https://github.com/umina-achala/umina-achala-chain/issues/29) (closed) |
| Refactor V2: UA's values (token classes, ID formats, royalty, commission, topics, images, networks) gathered into a top-level `config/` folder; `NETWORK` required, no default network; `mintNFT.js` split; test and sale-check speed. This repo stays UA only: another brand would start from a fork | `src/token/lib.js`, `src/consignment/lib.js`, `src/mint/` (values written in the code today) | In progress (Phase 2A): config with no change in behaviour, before the testnet mint ([chain #22](https://github.com/umina-achala/umina-achala-chain/issues/22)) | [chain #34](https://github.com/umina-achala/umina-achala-chain/issues/34), from [#29](https://github.com/umina-achala/umina-achala-chain/issues/29) (closed) |
| Refuse selling, consigning or burning a stone mounted in an Achala | `src/mint/achala.js` (`mountedProblem`), used by `sellOwnItem.js`, `consignToMerchant.js`, `sellConsigned.js`, `endConsignment.js` (#28), `adminToken.js` | On `main` | [chain #24](https://github.com/umina-achala/umina-achala-chain/issues/24) |
| `stone_id` in every HCS event | none | Not built (Phase 2B, before mainnet) | [chain #25](https://github.com/umina-achala/umina-achala-chain/issues/25) |
| Small HCS event fixes before mainnet | | Open (Phase 2B) | [chain #27](https://github.com/umina-achala/umina-achala-chain/issues/27) |
| Seller model (consignment or not) | | Open | [web #90](https://github.com/umina-achala/umina-achala-web/issues/90) |
| Wallet connection and payment | none | Not built | [web #58](https://github.com/umina-achala/umina-achala-web/issues/58) |
| Backend that signs UA's side of a sale | none | Not built | [web #116](https://github.com/umina-achala/umina-achala-web/issues/116) |
| Token ID and serial in the site data | `frontend/src/types/catalog.ts`, `catalog.json` | Built (branch `site-chain-data`) | [web #89](https://github.com/umina-achala/umina-achala-web/issues/89) |
| HashScan link hardcodes mainnet | `frontend/src/components/StoneModal.tsx` | Not fixed | [web #75](https://github.com/umina-achala/umina-achala-web/issues/75) |
| Sold stones on the marketplace (mirror node ownership check) | `frontend/src/data/CatalogFacade.ts`, `components/Marketplace.tsx` | Not built | [web #115](https://github.com/umina-achala/umina-achala-web/issues/115) |
