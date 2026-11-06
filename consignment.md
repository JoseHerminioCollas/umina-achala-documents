# Umiña Consignment Model
- [Umiña Consignment Model](#umiña-consignment-model)
  - [Overview](#overview)
  - [Consignment Flow](#consignment-flow)
  - [Flow on Hedera](#flow-on-hedera)
    - [Step 1: Mint to Treasury](#step-1-mint-to-treasury)
    - [Step 2: Transfer to Merchant](#step-2-transfer-to-merchant)
    - [Step 3: Listing on Umiña Achala](#step-3-listing-on-umiña-achala)
    - [Step 4: Primary Sale (70 / 30 Split)](#step-4-primary-sale-70--30-split)
    - [Step 5: Secondary Sales and Royalties](#step-5-secondary-sales-and-royalties)
    - [Step 6: Fallback Fee](#step-6-fallback-fee)
  - [UA's Own Items (No Consignment)](#uas-own-items-no-consignment)
  - [Third-Party Marketplaces (SentX)](#third-party-marketplaces-sentx)
    - [The Bypass Problem](#the-bypass-problem)
    - [Control Options](#control-options)
  - [Token Configuration Decisions](#token-configuration-decisions)
  - [Open Questions](#open-questions)
  - [Recommended Direction](#recommended-direction)
  - [To Verify on Testnet](#to-verify-on-testnet)
  - [Testnet Implementation](#testnet-implementation)
    - [Scripts](#scripts)
    - [Setup](#setup)
    - [Run Order](#run-order)
    - [Code Samples](#code-samples)
    - [Review: Gaps Before Mainnet](#review-gaps-before-mainnet)

## Overview

> **Status: Phase 2 (testnet), in progress. Decision not made yet** ([web #90](https://github.com/umina-achala/umina-achala-web/issues/90)). The testnet code is in `umina-achala-chain` (fork `main`, 2026-11-05). UA has not decided whether it will take merchant items on consignment at all. This document describes how consignment would work if UA does, so the option can be evaluated and tested on testnet.

If UA takes items on consignment, Umiña Achala (UA) sells specimens for merchants. Each specimen is represented by an NFT serial in the Umiña collection on the Hedera Token Service (HTS). The merchant holds the NFT in their own wallet. UA lists the item for sale and takes a commission when the item first sells. After that, UA earns a royalty every time the item is resold.

This document describes the intended flow and the Hedera mechanics behind each step. It also covers how merchants listing on third-party marketplaces such as SentX affect the model, and which decisions must be made before the mainnet token is created.

## Consignment Flow

1. UA mints an NFT for the merchant's specimen.
2. The merchant receives the NFT and sees it in their wallet.
3. The item is listed for sale on Umiña Achala.
4. The item sells on Umiña Achala. Payment is split **70% to the merchant, 30% to UA**.
5. The buyer resells the item. UA receives a **royalty** on each resale.
6. A **fallback fee** applies when the NFT changes hands without on-chain payment.

## Flow on Hedera

### Step 1: Mint to Treasury

The Supply Role mints the serial. As described in [create-nft-token.md](create-nft-token.md), every newly minted NFT goes to the Treasury Account first.

### Step 2: Transfer to Merchant

- The merchant's account must **associate** the Umiña token before it can receive a serial.
- The Treasury transfers the serial to the merchant, and it then appears in the merchant's wallet.
- Custom fees and the token treasury: confirm whether the fallback fee is charged to the merchant on this transfer (see [To Verify on Testnet](#to-verify-on-testnet)).

### Step 3: Listing on Umiña Achala

UA does not hold the NFT once it is in the merchant's wallet. To sell it, UA needs permission to move it:

- The merchant grants UA's marketplace account an **NFT allowance** (`AccountAllowanceApproveTransaction` / `approveTokenNftAllowance`) for the serial.
- When a sale happens, UA's account moves the NFT from the merchant to the buyer, in the same transaction as the payment.

An allowance does not give UA exclusivity. The merchant can grant allowances to other marketplaces on the same serial (see [Third-Party Marketplaces](#third-party-marketplaces-sentx)). UA must watch the mirror node for transfers of listed serials and remove a listing once the NFT has left the merchant's account.

### Step 4: Primary Sale (70 / 30 Split)

With HBAR payment, the sale settles as one atomic transfer:

| Movement | Amount (on a sale of 100) |
|---|---|
| Buyer → Merchant | 70 |
| Buyer → UA | 30 |
| Merchant → Buyer | NFT serial |

**Royalty interaction.** HTS calculates the royalty on the fungible value received by the **NFT sender**, which here is the merchant. With the current 5% royalty, the merchant's 70 becomes 70 − 3.5 = **66.5**, and the 3.5 goes to the Treasury. The split has to be designed with this in mind. Options:

- Treat the royalty as part of UA's 30%, and set the transfer amounts so that the merchant nets 70.
- Accept that the royalty is charged on top, and state it in the consignment agreement.

**Decision not made yet.** The testnet code (`sellConsigned.js`) currently charges the royalty on top: it pays the merchant 70 and HTS then takes 3.5 of it.

**Fiat payment.** If the buyer pays off-chain (card, bank transfer), no fungible value moves with the NFT. HTS then charges the **fallback fee** to the buyer as the receiver. The buyer needs HBAR in their account and must sign the transaction. That is friction for buyers who don't use crypto, and the checkout design has to account for it.

### Step 5: Secondary Sales and Royalties

When a buyer resells on any marketplace that settles on-chain, HTS enforces the royalty automatically and routes it to the fee collector (currently the Treasury Account). No marketplace cooperation is needed. The royalty is set in `ROYALTY` in `umina-achala-chain/src/token/lib.js` (currently 5%).

### Step 6: Fallback Fee

The fallback fee is a fixed fee attached to the royalty. It is charged to the **receiver** of the NFT when no fungible value is exchanged in the transfer. It covers:

- Private deals paid off-chain and settled as a plain wallet-to-wallet transfer.
- "Gifts" used to avoid the royalty.

The fallback fee does **not** affect sales on marketplaces such as SentX. Those sales are paid on-chain and pay the normal royalty.

The fallback fee is a fixed amount, not a percentage. It cannot capture a price paid off-chain, and it is also charged on genuine gifts.

A fallback fee of about USD 3 in HBAR is **decided and in the code** ([chain #1](https://github.com/umina-achala/umina-achala-chain/issues/1), [chain #6](https://github.com/umina-achala/umina-achala-chain/issues/6)): `buildRoyaltyFee` in `umina-achala-chain/src/token/lib.js` sets it from `FALLBACK_FEE_HBAR` when the token is created. `node src/token/adminToken.js fees --fallback-hbar <hbar>` changes it later with the Fee Schedule Key ([chain #21](https://github.com/umina-achala/umina-achala-chain/issues/21)). See [nft-transfers-and-sales.md](nft-transfers-and-sales.md#when-the-royalty-is-paid).

## UA's Own Items (No Consignment)

UA can also sell specimens it owns itself. This case does not depend on the consignment decision. There is no merchant, no allowance and no split: the NFT stays in the Treasury from minting until it sells.

**Flow**

1. UA mints the NFT (`src/mint/mintNFT.js`; an Achala with `--token achala`). The serial goes to the Treasury, as in [Step 1](#step-1-mint-to-treasury).
2. The item is listed on Umiña Achala. No allowance is needed, because UA holds the NFT.
3. The buyer associates the Umiña token.
4. The sale settles as one atomic transfer. The Treasury signs, because it sends the NFT; the buyer signs, because HBAR leaves their account.
5. The sale is recorded in HCS as `SALE`, the same as a consigned sale.

**Primary sale**

| Movement | Amount (on a sale of 100) |
|---|---|
| Buyer → UA | 100 |
| Treasury → Buyer | NFT serial |

**Compared with a consigned sale**

| | Consigned sale | UA's own item |
|---|---|---|
| NFT holder before the sale | Merchant | Treasury |
| NFT transfer | `addApprovedNftTransfer` (UA spends the merchant's allowance) | `addNftTransfer` (the Treasury signs) |
| Payment | Buyer −100, merchant +70, UA +30 | Buyer −100, UA +100 |
| Signed by | UA (payer and spender), buyer | Treasury, buyer |
| Allowance | Required | Not needed |
| Royalty on the primary sale | Taken from the merchant's 70 | Not yet known. The Treasury is the sender; see [To Verify on Testnet](#to-verify-on-testnet) |
| SentX bypass | Possible (the merchant holds the NFT) | Not possible, UA controls the NFT until it sells |

Secondary sales and the fallback fee work the same in both cases ([Step 5](#step-5-secondary-sales-and-royalties), [Step 6](#step-6-fallback-fee)).

**Code.** `src/sale/sellOwnItem.js <serial> <price-hbar>` ([chain #19](https://github.com/umina-achala/umina-achala-chain/issues/19)) refuses a stone mounted in an Achala ([chain #24](https://github.com/umina-achala/umina-achala-chain/issues/24)), checks that the Treasury still holds the serial, then makes one atomic transfer and records `SALE`:

```js
new TransferTransaction()
  .addHbarTransfer(buyerId, priceHbar.negated())   // -100
  .addHbarTransfer(uaId, priceHbar)                // +100
  .addNftTransfer(nftId, treasuryId, buyerId)      // NFT from the Treasury
  .freezeWith(client);                             // signed by the buyer, and the Treasury if not the operator
```

**Status.** Built and tested on the fork's `main`; not yet run on testnet. No web checkout sells UA's own items yet ([web #116](https://github.com/umina-achala/umina-achala-web/issues/116)).

## Third-Party Marketplaces (SentX)

### The Bypass Problem

SentX is a Hedera NFT marketplace. It lists NFTs by taking an NFT allowance from the holder, the same way UA does. Once the merchant holds the NFT, they can list it on SentX directly. If the first sale happens on SentX:

- UA receives only the royalty (5%), **not** the 30% commission.
- UA's buyer checklist, compliance and export status steps are skipped.
- UA may not learn that the item sold until it sees the transfer on the mirror node.

Secondary sales on SentX are fine. The royalty applies, and the item reaches a wider market.

### Control Options

| Option | How it works | Trade-off |
|---|---|---|
| **1. UA custody until first sale** | The NFT stays in the Treasury or an escrow account until it sells. The merchant sees the item in a UA dashboard, not in their wallet. | No way around UA for the first sale. The merchant doesn't see the NFT in their wallet. |
| **2. KYC key** | Only accounts granted KYC by UA can hold the token. A SentX sale to an unapproved buyer fails. | Resales are also limited to UA-approved accounts. This may be desirable for export compliance. |
| **3. Freeze key** | UA freezes the merchant's account for the token until the item sells. | Freezing works per account for the whole token, not per serial. It locks every Umiña NFT that merchant holds, including ones UA is selling. Not practical. |
| **4. Off-chain enforcement** | The consignment agreement gives UA an exclusive listing window, or makes the commission payable wherever the item first sells. | Enforced by contract, not on-chain. Strong if UA physically holds the specimen (see below). |

**Physical custody.** If UA holds the physical specimen during consignment, the NFT is a claim on a stone in UA's possession. A buyer who purchases on SentX still has to go through UA to receive the specimen. UA can then apply its terms, compliance checks and commission at redemption. This makes Option 4 much stronger.

## Token Configuration Decisions

The token definition (`umina-achala-chain/src/token/lib.js`, shared by the Umiña and Achala classes) sets these keys: admin, supply, fee schedule, pause and wipe. It does **not** set a metadata key, a KYC key or a freeze key.

As far as we know, HTS does not allow adding a key that the token was not created with. Options 2 and 3 above must therefore be decided **before the mainnet token is created**.

| Decision | Current state | Must decide before mainnet | Status |
|---|---|---|---|
| KYC key | Not set | Yes | Decision not made yet |
| Freeze key | Not set | Yes | Decision not made yet |
| Fallback fee | Set in the code from `FALLBACK_FEE_HBAR` | Recommended | **Decided and built:** about USD 3 in HBAR ([chain #1](https://github.com/umina-achala/umina-achala-chain/issues/1)) |
| Metadata key | Not set | Yes | **Decided and built:** none ([chain #1](https://github.com/umina-achala/umina-achala-chain/issues/1)) |
| Royalty percentage | 5% | Can be changed later with the Fee Schedule Key | Decision not made yet |
| Fee collector | Treasury Account | Can be changed later with the Fee Schedule Key | Decision not made yet |

## Open Questions

**Decision not made yet** on any of these.

- Will UA take merchant items on consignment at all? The questions below apply only if it does.
- Who physically holds the specimen during consignment: UA or the merchant?
- Should buyers be able to resell freely on SentX and other marketplaces? If yes, do not set a KYC key.
- Does UA's 30% include the 5% royalty, or is the royalty charged on top?
- How will fiat buyers pay the fallback fee, or will UA pay it on their behalf?
- What is the fallback fee amount?
- How long is the exclusive listing window, if the consignment agreement includes one?

## Recommended Direction

This is a proposal. **Decision not made yet.** The testnet implementation below follows it so it can be tried out.

- UA holds the physical specimen during consignment.
- The NFT goes to the merchant's wallet, and the merchant grants UA an NFT allowance to sell it.
- The consignment agreement covers sales made outside UA: an exclusive window, and commission due at physical redemption.
- A fallback fee is added to the royalty before the mainnet token is created.
- UA monitors the mirror node for transfers of listed serials and removes stale listings.

With this setup, merchants keep wallet visibility, SentX stays open for resales, and UA's leverage comes from physical redemption rather than on-chain locks.

## To Verify on Testnet

- Is the fallback fee charged when the Treasury transfers a serial to the merchant?
- The exact royalty amount deducted on a 70 / 30 primary sale settled in HBAR.
- Whether a royalty is charged when UA sells its own item straight from the Treasury (`sellOwnItem.js`).
- Whether the fallback fee is charged to the Treasury when `endConsignment.js` returns an NFT (expected: no, the Treasury is the fee collector).
- The fallback fee charged on a zero-value transfer between two non-treasury accounts.
- Whether a token update can add a KYC or freeze key after creation (expected: no).
- How a SentX listing behaves when the same serial is also approved for UA, and what happens to the stale listing after a sale on the other marketplace.

## Testnet Implementation

The flow above is implemented as testnet scripts in `umina-achala-chain/src`, to try out the option while the decision on consignment is not made yet. They follow the [Recommended Direction](#recommended-direction): the NFT goes to the merchant's wallet and UA sells it through an NFT allowance. Test accounts in `.env.testnet` stand in for the merchant and the buyer, and their keys sign in place of the wallets they would use in production.

### Scripts

| Script | Step | What it does | Hedera calls | Signed by |
|---|---|---|---|---|
| `mint/mintNFT.js` | 1 | Mints a new serial to the Treasury: records `COMPLIANCE`, pins the photos and metadata, mints, records `MINT` (see [mint-nft-token.md](mint-nft-token.md#current-implementation-phase-2-testnet)) | `TopicMessageSubmitTransaction`, `TokenMintTransaction` | Supply key |
| `consignment/associateToken.js` | 2, 4 | Associates a token (Umiña, or Achala with `--token achala`) with the merchant or buyer account | `TokenAssociateTransaction` | That account |
| `consignment/consignToMerchant.js` | 2, 3 | Associates the merchant (skipped if already done), transfers the serial from the Treasury to the merchant, has the merchant grant UA an NFT allowance, and records `ISSUED_TO_OWNER` ([chain #18](https://github.com/umina-achala/umina-achala-chain/issues/18)) | `TokenAssociateTransaction`, `TransferTransaction`, `AccountAllowanceApproveTransaction`, `TopicMessageSubmitTransaction` | Merchant, Treasury, merchant |
| `consignment/sellConsigned.js` | 4 | Checks the merchant still holds the serial, settles the 70 / 30 sale and the NFT transfer in one transaction, and records `SALE` | `TokenNftInfoQuery`, `TransferTransaction`, `TopicMessageSubmitTransaction` | UA (payer and allowance spender), buyer |
| `consignment/endConsignment.js` | | Ends a consignment: checks the merchant holds the serial, the merchant revokes UA's allowance and returns the NFT to the Treasury, and `TRANSFER` (reason `consignment_ended`) is recorded. `--keep` only revokes the allowance and records nothing ([chain #18](https://github.com/umina-achala/umina-achala-chain/issues/18)) | `TokenNftInfoQuery`, `AccountAllowanceDeleteTransaction`, `TransferTransaction`, `TopicMessageSubmitTransaction` | Merchant |
| `consignment/lib.js` | | Shared helpers: the 70 / 30 split, token association, and argument and environment checks | | |

Every event goes through `src/hcs.js`, which signs it with `HCS_SUBMIT_KEY` when set and waits for the receipt ([chain #17](https://github.com/umina-achala/umina-achala-chain/issues/17)). If an event fails after the NFT has already moved, the script says so and prints the transaction ID, so it can be recorded by hand.

Minting and consignment are separate steps. Consignment never mints: it works on a serial that already exists, given by its number.

UA is the operator account (`OPERATOR_ID`) in every script, so it pays all transaction fees.

### Setup

1. Create two testnet accounts with ECDSA keys, one for the merchant and one for the buyer.
2. Add them to `.env.testnet` (see `.env.example`):

   ```
   MERCHANT_ID=0.0.…
   MERCHANT_KEY=…
   BUYER_ID=0.0.…
   BUYER_KEY=…
   ```

3. Create the token and the provenance topic, and set `UMINA_TOKEN_ID` and `HCS_TOPIC_ID` to their real IDs (`node src/token/createToken.js`, `node src/topic/createTopic.js provenance`; see [create-hcs-topic.md](create-hcs-topic.md)). Make sure `TREASURY_ID`, `SUPPLY_KEY` and `HCS_SUBMIT_KEY` are set. Until the topic exists, minting is refused on purpose.

Run every command from `umina-achala-chain`. Each script takes `--dry-run`, which checks the arguments and environment and sends nothing.

### Run Order

```bash
# 1. Mint: prints the new serial, e.g. 7. The NFT is in the Treasury.
node src/mint/mintNFT.js --form json/sample-umina-form-post.json

# 2. Consign: serial 7 moves to the merchant, UA may sell it, ISSUED_TO_OWNER is recorded.
node src/consignment/consignToMerchant.js 7

# 3. The buyer associates the token (once per buyer account).
node src/consignment/associateToken.js buyer

# 4. Sell for 100 HBAR: buyer -100, merchant +70, UA +30, NFT to the buyer, SALE recorded.
node src/consignment/sellConsigned.js 7 100

# Or, if the item does not sell: revoke the allowance and return it, TRANSFER recorded.
node src/consignment/endConsignment.js 7
```

`node src/consignment/associateToken.js merchant` is also available, but `consignToMerchant.js` already associates the merchant.

After the sale, open the sale transaction ID on HashScan to see the royalty HTS deducted from the merchant. That answers the royalty item in [To Verify on Testnet](#to-verify-on-testnet).

### Code Samples

**Association.** An account must associate a token before it can receive one of its NFTs. The account signs, and UA (the client's operator) pays the fee:

```js
await new TokenAssociateTransaction()
  .setAccountId(accountId)
  .setTokenIds([tokenId])
  .freezeWith(client)
  .sign(accountKey);
```

**Treasury → merchant.** A plain NFT transfer. An NFT is addressed by token ID and serial number:

```js
const nftId = new NftId(TokenId.fromString(process.env.UMINA_TOKEN_ID), 7);

new TransferTransaction()
  .addNftTransfer(nftId, treasuryId, merchantId)
  .freezeWith(client);   // the Treasury key signs if it is not the operator
```

**Allowance.** The merchant lets UA move this one serial on their behalf:

```js
await new AccountAllowanceApproveTransaction()
  .approveTokenNftAllowance(nftId, merchantId, uaId)
  .freezeWith(client)
  .sign(merchantKey);
```

**70 / 30 split.** Integer math in tinybars (1 HBAR = 100,000,000 tinybars). The commission is in basis points (3000 = 30.00%). The merchant receives the rest, so the two shares always add up to the price:

```js
export const UA_SHARE_BPS = 3000n;

export function consignmentSplit(priceTinybars) {
  const ua = (priceTinybars * UA_SHARE_BPS) / 10000n;
  return { merchant: priceTinybars - ua, ua };
}
```

**The sale.** One atomic transaction. The HBAR amounts must net to zero: the negative amount is the buyer's debit, the positive amounts are credits. `addApprovedNftTransfer` uses the merchant's allowance, which the network checks against the transaction's payer, so UA must be the payer. The buyer signs because HBAR leaves their account. The merchant does not sign.

```js
const tinybars = (n) => Hbar.fromTinybars(n.toString());

await new TransferTransaction()
  .setTransactionId(TransactionId.generate(uaId))           // UA pays and spends the allowance
  .addHbarTransfer(buyerId, tinybars(-priceTinybars))       // -100
  .addHbarTransfer(merchantId, tinybars(split.merchant))    // +70
  .addHbarTransfer(uaId, tinybars(split.ua))                // +30
  .addApprovedNftTransfer(nftId, merchantId, buyerId)       // NFT to the buyer
  .freezeWith(client)
  .sign(buyerKey);                                          // UA signs on execute
```

HTS then applies the royalty on its own: with 5%, the merchant ends with 66.5 and the Treasury receives 3.5. The royalty does not appear in the code.

**Notarization.** Like `MINT`, the sale is recorded in HCS, with the event envelope from [chain #10](https://github.com/umina-achala/umina-achala-chain/issues/10) (built by `buildEvent` in `src/hcs.js`, see [create-hcs-topic.md](create-hcs-topic.md#event-messages)):

```js
buildEvent(
  { event: "SALE", tokenId, serials: [serial], actorId: uaId, ref: saleTxId },
  {
    seller_id, buyer_id, payment: "on_chain", sold_for, currency: "HBAR",
    price_tinybars, merchant_tinybars, ua_tinybars,
  },
);
// then submitEvent(client, HCS_TOPIC_ID, message, submitKey): signs and waits for the receipt
```

### Review: Gaps Before Mainnet

- **Royalty and the split.** `UA_SHARE_BPS` gives UA 30% and the royalty is charged on top, so the merchant nets 66.5. Decision not made yet (see [Open Questions](#open-questions)). If UA's 30% is to include the royalty, `consignmentSplit` is the one place to change.
- **Wallet signing.** The `.sign(merchantKey)` and `.sign(buyerKey)` calls stand in for wallets. In production UA builds and freezes the transaction, sends `tx.toBytes()` to the merchant's or buyer's wallet (for example HashPack through WalletConnect), and executes the signed result. Association also happens in the wallet.
- **Listing watch.** `sellConsigned.js` checks the holder only at sale time. A mirror-node watch that removes listings once a serial leaves the merchant's account is not built. A proposed website ownership check is in [nft-transfers-and-sales.md](nft-transfers-and-sales.md#website-showing-sold-stones) ([web #115](https://github.com/umina-achala/umina-achala-web/issues/115)).
- **Wallet-only sales.** No wallet screen can build the 70 / 30 sale, because UA must pay for and sign it as the allowance spender. See [nft-transfers-and-sales.md](nft-transfers-and-sales.md#ua-sales-what-can-happen-in-a-wallet); the backend that would sign UA's side is [web #116](https://github.com/umina-achala/umina-achala-web/issues/116).
- **Fiat sales and the fallback fee.** The fallback fee is in the code; how fiat buyers pay it is not decided.
- **Testnet only.** The scripts use `Client.forTestnet()` and `.env.testnet`. Mainnet support is not built.
- **Not run on testnet yet.** No token or topic exists yet: they are created once all Phase 2A code is written. The items in [To Verify on Testnet](#to-verify-on-testnet) need a real run of the steps above, including whether the Treasury, as fee collector, is exempt from the fallback fee when `endConsignment.js` returns an NFT.

Done since the last review (2026-11-05, on the fork's `main`): every event is signed with the submit key and checked by its receipt ([chain #17](https://github.com/umina-achala/umina-achala-chain/issues/17)); consignment is recorded as `ISSUED_TO_OWNER` and can be ended with `endConsignment.js`; `consignToMerchant.js` requires `TREASURY_KEY` when the Treasury is not the operator, with script tests ([chain #18](https://github.com/umina-achala/umina-achala-chain/issues/18)); UA's own items sell with `sellOwnItem.js` ([chain #19](https://github.com/umina-achala/umina-achala-chain/issues/19)).
