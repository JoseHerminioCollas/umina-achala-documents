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

> **Status: Phase 2 (testnet), in progress. Seller model decided 2026-11-09** ([web #90](https://github.com/umina-achala/umina-achala-web/issues/90)): the merchant owns the item, holds its NFT and the physical item, and may sell it anywhere; UA takes 30% only when it sells on UA's site. See [Decisions (2026-11-09)](#decisions-2026-11-09). The testnet code is in `umina-achala-chain` (fork `main`).

Umiña Achala (UA) sells specimens for merchants. Each specimen is represented by an NFT serial in the Umiña collection on the Hedera Token Service (HTS). The merchant holds the NFT in their own wallet. UA lists the item for sale and takes a commission when the item first sells. After that, UA earns a royalty every time the item is resold.

This document describes the intended flow and the Hedera mechanics behind each step. It also covers how merchants listing on third-party marketplaces such as SentX affect the model, and which decisions must be made before the mainnet token is created.

## Consignment Flow

1. UA mints an NFT for the merchant's specimen.
2. The merchant receives the NFT and sees it in their wallet.
3. The item is listed for sale on Umiña Achala.
4. The item sells on Umiña Achala. The merchant ends with **70%** and UA with **30%**, the royalty included.
5. The buyer resells the item. UA receives a **royalty** on each resale.
6. A **fallback fee** applies when the NFT changes hands without on-chain payment.

## Decisions (2026-11-09)

From [web #90](https://github.com/umina-achala/umina-achala-web/issues/90). They apply to a merchant's Umiña and to a merchant's Achala (a piece of jewelry) with its stones.

- **Ownership.** The merchant owns the item 100%. UA mints it; the NFT (for an Achala, the Achala and every stone in it) goes to the merchant's wallet, and the merchant keeps the physical item.
- **Artisan.** The merchant pays the artisan separately for the labor of making a piece. The artisan is not a party to the sale and gets no share in the sale transfer.
- **Where the merchant may sell.** Anywhere, by default: on UA's site, through a wallet or SentX, or off chain.
- **UA sells its own items and merchants' items.** UA holds its own items (the NFT stays in the Treasury until sold); a merchant holds theirs.
- **How UA sells.** On its site: the buyer pays from their wallet, and UA's backend builds the transaction and signs UA's side ([nft-transfers-and-sales.md](nft-transfers-and-sales.md#sale-options), Option 1).
- **Seller in the metadata.** The merchant's name and RUC go in the NFT metadata, which is public and permanent. `creator` names the seller or the artisan, as the merchant chooses.
- **Taking a piece apart.** Only the merchant who owns an Achala may take it apart. UA then burns the Achala NFT at the merchant's request; the stones' NFTs are kept.
- **No KYC key** on the token, so buyers can resell freely on SentX and other marketplaces (2026-11-09; confirm before the mainnet token is created).

| Where it sells | What UA receives |
|---|---|
| UA's site | 30% of the price, the 5% royalty included: the merchant ends with exactly 70% ([Step 4](#step-4-primary-sale-70--30-split)) |
| Wallet transfer or SentX, paid on chain | The 5% royalty, automatically |
| Off chain (cash, in person) | The merchant sends the NFT to the buyer with no payment, so the buyer pays the fallback fee (about USD 3 in HBAR) for each NFT moved |

The merchant signs an agreement with these terms when an item is placed with UA: see the example in [consignment-agreement.md](consignment-agreement.md).

An Achala sold with its stones pays the royalty **twice**, once for each token (the Achala token and the Umiña token), each on the merchant's whole leg; the split is sized for that, see below ([chain #22](https://github.com/umina-achala/umina-achala-chain/issues/22), [chain #28](https://github.com/umina-achala/umina-achala-chain/issues/28)). Still open: how the fallback fee applies to an Achala moved with its stones without payment.

## Flow on Hedera

### Step 1: Mint to Treasury

The Supply Role mints the serial. As described in [create-nft-token.md](create-nft-token.md), every newly minted NFT goes to the Treasury Account first.

### Step 2: Transfer to Merchant

Hedera docs: [Associate tokens to an account](https://docs.hedera.com/native/tokens/associate), [Transfer tokens](https://docs.hedera.com/native/tokens/transfer).

- The merchant's account must **associate** the Umiña token before it can receive a serial.
- The Treasury transfers the serial to the merchant, and it then appears in the merchant's wallet.
- Custom fees and the token treasury: confirm whether the fallback fee is charged to the merchant on this transfer (see [To Verify on Testnet](#to-verify-on-testnet)).

### Step 3: Listing on Umiña Achala

Hedera docs: [Approve an allowance](https://docs.hedera.com/native/accounts/approve-allowance), [Delete an allowance](https://docs.hedera.com/native/accounts/adjust-allowance). The consignment scripts' options are in [scripts.md](scripts.md#consignment).

UA does not hold the NFT once it is in the merchant's wallet. To sell it, UA needs permission to move it:

- The merchant grants UA's marketplace account an **NFT allowance** (`AccountAllowanceApproveTransaction` / `approveTokenNftAllowance`) for the serial.
- When a sale happens, UA's account moves the NFT from the merchant to the buyer, in the same transaction as the payment.

An allowance does not give UA exclusivity. The merchant can grant allowances to other marketplaces on the same serial (see [Third-Party Marketplaces](#third-party-marketplaces-sentx)). UA must watch the mirror node for transfers of listed serials and remove a listing once the NFT has left the merchant's account.

### Step 4: Primary Sale (70 / 30 Split)

With HBAR payment, the sale settles as one atomic transfer:

| Movement | Amount (on a sale of 100) |
|---|---|
| Buyer → Merchant | 73.68 |
| Buyer → UA | 26.32 |
| Merchant → Buyer | NFT serial |
| Royalty, 5% of the merchant's 73.68 → Treasury (HTS) | 3.68 |
| **Result** | **Merchant 70, UA 30** |

**Royalty interaction.** HTS calculates the royalty on the fungible value received by the **NFT sender**, which here is the merchant. **Decided (2026-11-09): UA's 30% includes the royalty.** The transfer pays the merchant `70 / 0.95` of the price and UA the rest; HTS then moves 5% of the merchant's leg to the Treasury, so the merchant ends with exactly 70 and UA with 30.

**Achala with its stones (k = 2).** HTS charges the royalty once per token among the NFTs that move (answered on testnet 2026-11-18, see Testnet answers). So for an Achala the transfer pays the merchant `70 / 0.90` of the price (77.78 on 100) and UA the rest (22.22); two royalties of 3.89 then leave 70 and 30. Stones of the same token share one royalty, so the count is 2 however many stones the piece has.

**Code.** `consignmentSplit` in `src/consignment/lib.js` (used by `sellConsigned.js`) sets the legs this way, taking the rate from `ROYALTY`; the merchant's leg is rounded up, so any rounding remainder goes to the merchant ([chain #30](https://github.com/umina-achala/umina-achala-chain/issues/30), on `main`). `royaltyCount` gives the number of royalties (one per token among the NFTs that move) and `consignmentSplit` sizes the merchant's leg for it ([chain #28](https://github.com/umina-achala/umina-achala-chain/issues/28), on the `achala-split-k2` branch). HTS rounds each royalty down (checked on testnet), so the merchant may end a tinybar or two over 70%.

**Fiat payment.** If the buyer pays off-chain (card, bank transfer), no fungible value moves with the NFT. HTS then charges the **fallback fee** to the buyer as the receiver. The buyer needs HBAR in their account and must sign the transaction. That is friction for buyers who don't use crypto, and the checkout design has to account for it.

### Step 5: Secondary Sales and Royalties

When a buyer resells on any marketplace that settles on-chain, HTS enforces the royalty automatically and routes it to the fee collector (currently the Treasury Account). No marketplace cooperation is needed. The royalty is set in `ROYALTY` in `umina-achala-chain/src/token/lib.js` (currently 5%).

### Step 6: Fallback Fee

The fallback fee is a fixed fee attached to the royalty. It is charged to the **receiver** of the NFT when no fungible value is exchanged in the transfer. It covers:

- Private deals paid off-chain and settled as a plain wallet-to-wallet transfer.
- "Gifts" used to avoid the royalty.

The fallback fee does **not** affect sales on marketplaces such as SentX. Those sales are paid on-chain and pay the normal royalty.

The fallback fee is a fixed amount, not a percentage. It cannot capture a price paid off-chain, and it is also charged on genuine gifts.

A fallback fee of about USD 3 in HBAR is **decided and in the code** ([chain #1](https://github.com/umina-achala/umina-achala-chain/issues/1), [chain #6](https://github.com/umina-achala/umina-achala-chain/issues/6)): `buildRoyaltyFee` in `umina-achala-chain/src/token/lib.js` sets it from `FALLBACK_FEE_HBAR` when the token is created. `NETWORK=testnet node src/token/adminToken.js fees --fallback-hbar <hbar>` changes it later with the Fee Schedule Key ([chain #21](https://github.com/umina-achala/umina-achala-chain/issues/21)). See [nft-transfers-and-sales.md](nft-transfers-and-sales.md#when-the-royalty-is-paid).

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
| Payment | Buyer −100, merchant +73.68, UA +26.32 (70 / 30 after the royalty) | Buyer −100, UA +100 |
| Signed by | UA (payer and spender), buyer | Treasury, buyer |
| Allowance | Required | Not needed |
| Royalty on the primary sale | Taken from the merchant's leg, part of UA's 30% | Not yet known. The Treasury is the sender; see [To Verify on Testnet](#to-verify-on-testnet) |
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

**Status.** Built and tested, on `main`; not yet run on testnet. No web checkout sells UA's own items yet ([web #116](https://github.com/umina-achala/umina-achala-web/issues/116)).

## Third-Party Marketplaces (SentX)

### The Bypass Problem

SentX is a Hedera NFT marketplace. It lists NFTs by taking an NFT allowance from the holder, the same way UA does. Once the merchant holds the NFT, they can list it on SentX directly. If the first sale happens on SentX:

- UA receives only the royalty (5%), **not** the 30% commission.
- UA's buyer checklist, compliance and export status steps are skipped.
- UA may not learn that the item sold until it sees the transfer on the mirror node.

Secondary sales on SentX are fine. The royalty applies, and the item reaches a wider market.

**Decided (2026-11-09): this is accepted.** The merchant owns the item and may sell it anywhere; UA's 30% applies only to sales on UA's site. The control options below are kept for reference; none is planned.

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
| KYC key | Not set | Yes | **Decided (2026-11-09):** none, so resales are free; confirm before mainnet |
| Freeze key | Not set | Yes | Decision not made yet |
| Fallback fee | Set in the code from `FALLBACK_FEE_HBAR` | Recommended | **Decided and built:** about USD 3 in HBAR ([chain #1](https://github.com/umina-achala/umina-achala-chain/issues/1)) |
| Metadata key | Not set | Yes | **Decided and built:** none ([chain #1](https://github.com/umina-achala/umina-achala-chain/issues/1)) |
| Royalty percentage | 5% | Can be changed later with the Fee Schedule Key | Decision not made yet |
| Fee collector | Treasury Account | Can be changed later with the Fee Schedule Key | Decision not made yet |

## Open Questions

**Decided (2026-11-09)**, see [Decisions](#decisions-2026-11-09):
- UA takes merchant items: the merchant owns them and holds the NFT and the physical item.
- UA's 30% includes the 5% royalty.
- No exclusive listing window: the merchant may sell anywhere.
- The fallback fee is about USD 3 in HBAR ([chain #1](https://github.com/umina-achala/umina-achala-chain/issues/1)).
- Buyers may resell freely: no KYC key.
- Only the merchant may take their Achala apart.

**Decision not made yet:**
- How will fiat buyers on UA's site pay the fallback fee, or will UA pay it on their behalf?

## Direction

- The merchant holds the physical item and the NFT, and grants UA an NFT allowance to sell it on UA's site.
- Sales made outside UA pay UA the royalty, or the fallback fee when no payment moves with the NFT.
- UA monitors the mirror node for transfers of listed serials and removes stale listings.

With this setup, merchants keep wallet visibility, SentX stays open for resales, and UA's leverage comes from physical redemption rather than on-chain locks.

## To Verify on Testnet

- Is the fallback fee charged when the Treasury transfers a serial to the merchant? **Answered 2026-11-18: no.** Amethyst A (Umiña serial 4) to the merchant with `consignToMerchant.js` (`0.0.4631334@1795020823.004533545`): only the 0.035 HBAR network fee, no custom fee assessed. The allowance that lets UA sell it (`…@1795020824.278605208`) cost 0.878 HBAR, paid by the Treasury as payer ([chain #22](https://github.com/umina-achala/umina-achala-chain/issues/22)).
- The exact royalty amount deducted on a 70 / 30 primary sale settled in HBAR. **Answered 2026-11-18: 5% of the merchant's share, as planned.** Amethyst A (Umiña serial 4) sold for 100 HBAR with `sellConsigned.js` (`0.0.4631334@1795021018.906965675`): the buyer paid 100, the merchant was paid 73.68421053 and HTS took a royalty of 3.68421052 from it for the Treasury, so the merchant ended with 70.00000001 and UA with 29.99999999 (1 tinybar of rounding). No fallback fee; network fee 0.039 HBAR, paid by the Treasury. The merchant's allowance to UA is gone after the sale ([chain #22](https://github.com/umina-achala/umina-achala-chain/issues/22)).
- Whether a royalty is charged when UA sells its own item straight from the Treasury (`sellOwnItem.js`). **Answered 2026-11-18: no.** Agate A (Umiña `0.0.11400486` serial 3) sold to the buyer for 20 HBAR (`0.0.4631334@1795020554.893398518`): the buyer paid 20, the Treasury received 19.087 (20 less the 0.913 HBAR network fee, which the Treasury paid as payer), no custom fee assessed. The Treasury is the fee collector, so neither the royalty nor the fallback fee applies ([chain #22](https://github.com/umina-achala/umina-achala-chain/issues/22)).
- Whether the fallback fee is charged to the Treasury when `endConsignment.js` returns an NFT (expected: no, the Treasury is the fee collector). **Answered 2026-11-18: no.** Fluorite A (Umiña serial 9) returned from the merchant to the Treasury (`0.0.4631334@1795021555.395312341`): no custom fee assessed, only the 0.037 HBAR network fee, paid by the Treasury. The merchant's allowance to UA was revoked first ([chain #22](https://github.com/umina-achala/umina-achala-chain/issues/22)).
- How the royalty applies to an Achala sold with its stones (N+1 NFTs, one payment). **Answered 2026-11-18: once per token, each on the merchant's whole payment.** Achala D (Achala serial 3) with quartz D (Umiña serial 7), consigned, sold for 200 HBAR with `sellConsigned.js --token achala` (`0.0.4631334@1795028095.982253922`): the merchant was paid 147.368 and HTS took **two** royalties of 7.368 (5% of 147.368 each, one for each token) for the Treasury. The merchant ended with **132.632 (66.3%)**, not 140, and UA with 67.368 (33.7%). Several NFTs of the **same** token share one royalty: agate A and amethyst A (Umiña serials 3 and 4) moved together for 10 HBAR (`0.0.11389082@1795030325.265127024`) paid one royalty of 0.5 HBAR, not two. So an Achala sold with any number of stones pays **two** royalties (k = 2: the Achala token and the Umiña token), 10% of the merchant's share, and `sellConsigned.js` must size the merchant's payment for that to keep 70 / 30 ([chain #28](https://github.com/umina-achala/umina-achala-chain/issues/28), [chain #22](https://github.com/umina-achala/umina-achala-chain/issues/22)). **Fixed and confirmed the same day:** with the split sized for k = 2, Achala A (Achala serial 2) with chrysocolla A (Umiña serial 1), consigned, sold for 100 HBAR (`0.0.4631334@1795034963.612049269`): the merchant was paid 77.778, HTS took two royalties of 3.889, and the merchant ended with 70.00000002 and UA with 29.99999998.
- The fallback fee charged on a zero-value transfer between two non-treasury accounts. **Answered 2026-11-18: the full 45 HBAR, paid by the receiver, and the receiver must sign.** Malachite A (Umiña serial 5), merchant → buyer with no HBAR: a HashPack **Send** from the merchant failed with `INVALID_SIGNATURE` (`0.0.11386627@1795023361.312784246`, nothing moved), because the fallback fee is charged to the receiver. Signed by both (a one-off script, now `src/testnet/transferBetweenWallets.js`, [chain #40](https://github.com/umina-achala/umina-achala-chain/issues/40)), it succeeded (`0.0.11386627@1795027030.375637933`): 45 HBAR from the buyer to the Treasury, network fee 0.037 HBAR paid by the merchant, UA's allowance on the serial cleared. So an owner cannot move an Umiña NFT to another wallet, even their own, without the receiver's signature and the 45 HBAR fee, unless one side is the Treasury or HBAR is paid in the same transaction ([chain #22](https://github.com/umina-achala/umina-achala-chain/issues/22)). **Rerun 2026-11-19 with the repo tool** (`transferBetweenWallets.js 5 --from buyer`, [chain #40](https://github.com/umina-achala/umina-achala-chain/issues/40)): malachite A, buyer → merchant, no HBAR (`0.0.11389082@1795095906.871563330`): 45 HBAR fallback fee from the merchant (the receiver) to the Treasury, network fee 0.0375 HBAR paid by the buyer (the sender).
- **The fallback fee on an Achala moved with its stones: once per token. Answered 2026-11-19.** Achala A (Achala serial 2) with chrysocolla A (Umiña serial 1), buyer → merchant with no HBAR, both signing (`transferBetweenWallets.js 2 --token achala --from buyer`, `0.0.11389082@1795096213.337475487`): **two** fallback fees of 45 HBAR, one per token, both from the merchant (the receiver), 90 HBAR in all. As with the royalty, k = 2 for an Achala with any number of stones. A wallet-to-wallet move of an Achala with no HBAR costs the receiver 2 × the fallback fee ([chain #40](https://github.com/umina-achala/umina-achala-chain/issues/40), [chain #22](https://github.com/umina-achala/umina-achala-chain/issues/22)) **Accepted for now (2026-11-19):** no change on chain; the seller terms should say so ([web #90](https://github.com/umina-achala/umina-achala-web/issues/90)). The options are on [chain #41](https://github.com/umina-achala/umina-achala-chain/issues/41), to decide with the 1-tinybar gap.
- **Found 2026-11-18: a token payment avoids both the royalty and the fallback fee.** Agate A (Umiña serial 3), merchant → buyer with the buyer paying **1 tinybar** in the same transaction, both signing (`0.0.11386627@1795030704.997769979`): the royalty was assessed at **0** (5% of 1 tinybar rounds down) and no fallback fee was charged, because some HBAR was exchanged. So two owners who cooperate can move an Umiña NFT for the network fee only; the royalty and the fallback fee only bind a transfer whose price is really paid on-chain. A decision for mainnet: [chain #41](https://github.com/umina-achala/umina-achala-chain/issues/41) ([chain #22](https://github.com/umina-achala/umina-achala-chain/issues/22)). **Confirmed 2026-11-19 with the repo tool** (`transferBetweenWallets.js 5 --from merchant --hbar 0.00000001`, [chain #40](https://github.com/umina-achala/umina-achala-chain/issues/40)): malachite A, merchant → buyer for 1 tinybar (`0.0.11386627@1795096096.835972935`): royalty assessed at 0, no fallback fee, network fee 0.0375 HBAR. The same-token rule held too: agate A and amethyst A (Umiña serials 3 and 4), buyer → merchant for 10 HBAR (`0.0.11389082@1795096173.839695073`), one royalty of 0.5 HBAR, paid by the buyer (the sender).
- Whether a token update can add a KYC or freeze key after creation (expected: no).
- How a SentX listing behaves when the same serial is also approved for UA, and what happens to the stale listing after a sale on the other marketplace.

## Testnet Implementation

The flow above is implemented as testnet scripts in `umina-achala-chain/src`, to try out the option while the decision on consignment is not made yet. They follow the [Recommended Direction](#recommended-direction): the NFT goes to the merchant's wallet and UA sells it through an NFT allowance. Test accounts in `.env.testnet` stand in for the merchant and the buyer, and their keys sign in place of the wallets they would use in production.

### Scripts

| Script | Step | What it does | Hedera calls | Signed by |
|---|---|---|---|---|
| `mint/mintNFT.js` | 1 | Mints a new serial to the Treasury: records `COMPLIANCE`, pins the photos and metadata, mints, records `MINT` (see [mint-nft-token.md](mint-nft-token.md#current-implementation-phase-2-testnet)) | `TopicMessageSubmitTransaction`, `TokenMintTransaction` | Supply key |
| `consignment/associateToken.js` | 2, 4 | Associates the Umiña token, or with `--token achala` both tokens, with the merchant or buyer account | `TokenAssociateTransaction` | That account |
| `consignment/consignToMerchant.js` | 2, 3 | Associates the merchant (skipped if already done), transfers the serial from the Treasury to the merchant, has the merchant grant UA an NFT allowance, and records `ISSUED_TO_OWNER` ([chain #18](https://github.com/umina-achala/umina-achala-chain/issues/18)). `--token achala`: the Achala and its stones, one transfer, one event ([chain #28](https://github.com/umina-achala/umina-achala-chain/issues/28)) | `TokenAssociateTransaction`, `TransferTransaction`, `AccountAllowanceApproveTransaction`, `TopicMessageSubmitTransaction` | Merchant, Treasury, merchant |
| `consignment/sellConsigned.js` | 4 | Checks the merchant still holds the serial (mirror node), settles the 70 / 30 sale and the NFT transfer in one transaction, and records `SALE`. `--token achala`: the Achala and its stones for one payment ([chain #28](https://github.com/umina-achala/umina-achala-chain/issues/28)) | `TransferTransaction`, `TopicMessageSubmitTransaction` | UA (payer and allowance spender), buyer |
| `consignment/endConsignment.js` | | Ends a consignment: checks the merchant holds the serial (mirror node), the merchant revokes UA's allowance and returns the NFT to the Treasury, and `TRANSFER` (reason `consignment_ended`) is recorded. `--keep` only revokes the allowance and records nothing ([chain #18](https://github.com/umina-achala/umina-achala-chain/issues/18)). `--token achala`: the Achala and its stones ([chain #28](https://github.com/umina-achala/umina-achala-chain/issues/28)) | `AccountAllowanceDeleteTransaction`, `TransferTransaction`, `TopicMessageSubmitTransaction` | Merchant |
| `consignment/lib.js` | | Shared helpers: the 70 / 30 split, token association, and argument and environment checks | | |

Every event goes through `src/hcs.js`, which signs it with `HCS_SUBMIT_KEY` when set and waits for the receipt ([chain #17](https://github.com/umina-achala/umina-achala-chain/issues/17)). If an event fails after the NFT has already moved, the script says so and prints the transaction ID, so it can be recorded by hand.

Minting and consignment are separate steps. Consignment never mints: it works on a serial that already exists, given by its number.

UA is the operator account (`OPERATOR_ID`) in every script, so it pays all transaction fees.

### Setup

1. Create two testnet accounts with ECDSA keys, one for the merchant and one for the buyer, in the Hedera portal ([scripts.md, Test accounts](scripts.md#test-accounts)).
2. Add them to `.env.testnet` (see `.env.example`):

   ```
   MERCHANT_ID=0.0.…
   MERCHANT_KEY=…
   BUYER_ID=0.0.…
   BUYER_KEY=…
   ```

3. Create the token and the provenance topic, and set `UMINA_TOKEN_ID` and `HCS_TOPIC_ID` to their real IDs (`NETWORK=testnet node src/token/createToken.js`, `NETWORK=testnet node src/topic/createTopic.js provenance`; see [create-hcs-topic.md](create-hcs-topic.md)). Make sure `TREASURY_ID`, `SUPPLY_KEY` and `HCS_SUBMIT_KEY` are set. Until the topic exists, minting is refused on purpose.

Run every command from `umina-achala-chain`. Each script takes `--dry-run`, which checks the arguments and environment and sends nothing.

### Run Order

```bash
# 1. Mint: prints the new serial, e.g. 7. The NFT is in the Treasury.
NETWORK=testnet node src/mint/mintNFT.js --form testnet/chrysocolla_a.json

# 2. Consign: serial 7 moves to the merchant, UA may sell it, ISSUED_TO_OWNER is recorded.
NETWORK=testnet node src/consignment/consignToMerchant.js 7

# 3. The buyer associates the token (once per buyer account).
NETWORK=testnet node src/consignment/associateToken.js buyer

# 4. Sell for 100 HBAR: buyer -100, merchant +70, UA +30, NFT to the buyer, SALE recorded.
NETWORK=testnet node src/consignment/sellConsigned.js 7 100

# Or, if the item does not sell: revoke the allowance and return it, TRANSFER recorded.
NETWORK=testnet node src/consignment/endConsignment.js 7
```

`NETWORK=testnet node src/consignment/associateToken.js merchant` is also available, but `consignToMerchant.js` already associates the merchant.

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

**70 / 30 split.** Integer math in tinybars (1 HBAR = 100,000,000 tinybars). The commission is in basis points (3000 = 30.00%, `commissionBps` in `config/umina-achala.js`). The merchant receives the rest, so the two shares always add up to the price:

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

- **Royalty and the split.** Decided (2026-11-09): UA's 30% includes the royalty. `consignmentSplit` sets the legs so the merchant ends with 70% ([chain #30](https://github.com/umina-achala/umina-achala-chain/issues/30), on `main`).
- **Wallet signing.** The `.sign(merchantKey)` and `.sign(buyerKey)` calls stand in for wallets. In production UA builds and freezes the transaction, sends `tx.toBytes()` to the merchant's or buyer's wallet (for example HashPack through WalletConnect), and executes the signed result. Association also happens in the wallet.
- **Listing watch.** `sellConsigned.js` checks the holder only at sale time. A mirror-node watch that removes listings once a serial leaves the merchant's account is not built. A proposed website ownership check is in [nft-transfers-and-sales.md](nft-transfers-and-sales.md#website-showing-sold-stones) ([web #115](https://github.com/umina-achala/umina-achala-web/issues/115)).
- **Wallet-only sales.** No wallet screen can build the 70 / 30 sale, because UA must pay for and sign it as the allowance spender. See [nft-transfers-and-sales.md](nft-transfers-and-sales.md#ua-sales-what-can-happen-in-a-wallet); the backend that would sign UA's side is [web #116](https://github.com/umina-achala/umina-achala-web/issues/116).
- **Fiat sales and the fallback fee.** The fallback fee is in the code; how fiat buyers pay it is not decided.
- **Network.** The scripts run on the network named by `NETWORK`, with its env file and mirror node (`config/testnet.js`, `src/network.js`, [chain #29](https://github.com/umina-achala/umina-achala-chain/issues/29)). `NETWORK` has no default: a script refuses to run without `NETWORK=testnet` or `NETWORK=mainnet` ([chain #34](https://github.com/umina-achala/umina-achala-chain/issues/34), on the `refactor-v2` branch). Mainnet is listed but not enabled until its keys and accounts are decided ([chain #32](https://github.com/umina-achala/umina-achala-chain/issues/32)).
- **Achala consignment.** Built on the `refactor-v2` branch ([chain #28](https://github.com/umina-achala/umina-achala-chain/issues/28), 2026-11-14): `--token achala` on the three consignment scripts moves the piece with its stones; a stone of it is refused on its own. The testnet run showed two royalties on an Achala sale, one per token ([chain #22](https://github.com/umina-achala/umina-achala-chain/issues/22)); `sellConsigned.js` sizes the merchant's leg for them (`achala-split-k2` branch, [chain #28](https://github.com/umina-achala/umina-achala-chain/issues/28)).
- **Run on testnet 2026-11-18 and 2026-11-19** ([chain #22](https://github.com/umina-achala/umina-achala-chain/issues/22)): consignment, consigned sales, the Achala split and the wallet-to-wallet transfers; the results are in [To Verify on Testnet](#to-verify-on-testnet). The wallet-to-wallet test tool is `src/testnet/transferBetweenWallets.js` ([chain #40](https://github.com/umina-achala/umina-achala-chain/issues/40)).

Done since the last review (2026-11-05, now on `main`): every event is signed with the submit key and checked by its receipt ([chain #17](https://github.com/umina-achala/umina-achala-chain/issues/17)); consignment is recorded as `ISSUED_TO_OWNER` and can be ended with `endConsignment.js`; `consignToMerchant.js` requires `TREASURY_KEY` when the Treasury is not the operator, with script tests ([chain #18](https://github.com/umina-achala/umina-achala-chain/issues/18)); UA's own items sell with `sellOwnItem.js` ([chain #19](https://github.com/umina-achala/umina-achala-chain/issues/19)).
