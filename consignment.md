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
  - [Third-Party Marketplaces (SentX)](#third-party-marketplaces-sentx)
    - [The Bypass Problem](#the-bypass-problem)
    - [Control Options](#control-options)
  - [Token Configuration Decisions](#token-configuration-decisions)
  - [Open Questions](#open-questions)
  - [Recommended Direction](#recommended-direction)
  - [To Verify on Testnet](#to-verify-on-testnet)

## Overview

Umiña Achala (UA) sells specimens on consignment for merchants. Each specimen is represented by an NFT serial in the Umiña collection on the Hedera Token Service (HTS). The merchant holds the NFT in their own wallet. UA lists the item for sale and takes a commission when the item first sells. After that, UA earns a royalty every time the item is resold.

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

**Fiat payment.** If the buyer pays off-chain (card, bank transfer), no fungible value moves with the NFT. HTS then charges the **fallback fee** to the buyer as the receiver. The buyer needs HBAR in their account and must sign the transaction. That is friction for buyers who don't use crypto, and the checkout design has to account for it.

### Step 5: Secondary Sales and Royalties

When a buyer resells on any marketplace that settles on-chain, HTS enforces the royalty automatically and routes it to the fee collector (currently the Treasury Account). No marketplace cooperation is needed. The royalty is set in `ROYALTY` in `umina-achala-chain/src/uminaToken.js` (currently 5%).

### Step 6: Fallback Fee

The fallback fee is a fixed fee attached to the royalty. It is charged to the **receiver** of the NFT when no fungible value is exchanged in the transfer. It covers:

- Private deals paid off-chain and settled as a plain wallet-to-wallet transfer.
- "Gifts" used to avoid the royalty.

The fallback fee does **not** affect sales on marketplaces such as SentX. Those sales are paid on-chain and pay the normal royalty.

The current token definition has **no fallback fee**. One must be added to the `CustomRoyaltyFee` before the mainnet token is created.

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

The current token definition (`umina-achala-chain/src/uminaToken.js`) sets these keys: admin, supply, fee schedule, pause, wipe and metadata. It does **not** set a KYC key or a freeze key.

As far as we know, HTS does not allow adding a key that the token was not created with. Options 2 and 3 above must therefore be decided **before the mainnet token is created**. The same applies to adding a fallback fee to the royalty, although the Fee Schedule Key can change custom fees later.

| Decision | Current state | Must decide before mainnet |
|---|---|---|
| KYC key | Not set | Yes |
| Freeze key | Not set | Yes |
| Fallback fee | Not set | Recommended |
| Royalty percentage | 5% | Can be changed later with the Fee Schedule Key |
| Fee collector | Treasury Account | Can be changed later with the Fee Schedule Key |

## Open Questions

- Who physically holds the specimen during consignment: UA or the merchant?
- Should buyers be able to resell freely on SentX and other marketplaces? If yes, do not set a KYC key.
- Does UA's 30% include the 5% royalty, or is the royalty charged on top?
- How will fiat buyers pay the fallback fee, or will UA pay it on their behalf?
- What is the fallback fee amount?
- How long is the exclusive listing window, if the consignment agreement includes one?

## Recommended Direction

- UA holds the physical specimen during consignment.
- The NFT goes to the merchant's wallet, and the merchant grants UA an NFT allowance to sell it.
- The consignment agreement covers sales made outside UA: an exclusive window, and commission due at physical redemption.
- A fallback fee is added to the royalty before the mainnet token is created.
- UA monitors the mirror node for transfers of listed serials and removes stale listings.

With this setup, merchants keep wallet visibility, SentX stays open for resales, and UA's leverage comes from physical redemption rather than on-chain locks.

## To Verify on Testnet

- Is the fallback fee charged when the Treasury transfers a serial to the merchant?
- The exact royalty amount deducted on a 70 / 30 primary sale settled in HBAR.
- The fallback fee charged on a zero-value transfer between two non-treasury accounts.
- Whether a token update can add a KYC or freeze key after creation (expected: no).
- How a SentX listing behaves when the same serial is also approved for UA, and what happens to the stale listing after a sale on the other marketplace.
