# Umiña Achala: Example Consignment Agreement

> **Example only. Not legal advice and not ready to sign.** This is a template for the agreement a merchant signs when they place an item with Umiña Achala (UA). It follows the seller model decided on 2026-11-09 ([web #90](https://github.com/umina-achala/umina-achala-web/issues/90), [consignment.md](consignment.md#decisions-2026-11-09)). A qualified person must review it before use ([web #105](https://github.com/umina-achala/umina-achala-web/issues/105)), and the version merchants sign should be in Spanish. Text in `[brackets]` is filled in per merchant. Items marked **(open)** depend on decisions not made yet.

---

## Consignment Agreement

**Date:** `[date]`
**Agreement number:** `[UA-CONS-0001]`

**Between**

1. **Umiña Achala** (`UA`), `[legal name]`, RUC `[number]`, address `[address]`, represented by `[name, role]`.
2. **The Merchant**, `[name or company name]`, RUC `[number]` (or DNI `[number]`), address `[address]`, Hedera account `[0.0.xxxx]`, email `[email]`.

### 1. Purpose

The Merchant asks UA to mint NFTs on the Hedera network for the items listed in Annex A, and to offer those items for sale on UA's website. Each item is a stone (an "Umiña") or a piece of jewelry (an "Achala") together with the Umiña stones mounted in it.

### 2. Ownership

2.1 The Merchant owns each item 100%, before and after minting, until it is sold. This agreement does not transfer ownership of any item to UA.

2.2 After minting, UA transfers each item's NFTs to the Merchant's Hedera account. For an Achala, these are the Achala NFT and the NFT of every stone mounted in it.

2.3 The Merchant keeps the physical item. UA does not hold it, unless the parties agree otherwise in writing.

### 3. Minting

3.1 Before minting, the Merchant gives UA the documents listed in Annex C for each item. UA mints only items that pass its checks; UA may refuse an item without giving a reason. **(open: the list of checks, #90 M1)**

3.2 The Merchant's Hedera account must be associated with the Umiña token and, for jewelry, the Achala token, before UA can send the NFTs.

3.3 NFT metadata is permanent once minted and cannot be corrected. It includes the Merchant's name and RUC. The Merchant chooses whether the NFT's creator is the Merchant or the artisan (`[Merchant / artisan name]`), and checks the item data in Annex A before minting.

3.4 `[Who pays the network fee or fallback fee, if one is charged when UA sends the NFTs to the Merchant: UA / the Merchant.]` **(open: to verify on testnet, chain #22)**

### 4. Listing on UA's Website

4.1 The Merchant sets the price of each item and may change it at any time by telling UA.

4.2 To list an item, the Merchant grants UA's account `[0.0.xxxx]` an NFT allowance for the item's NFTs. The allowance lets UA move the NFTs to a buyer only as part of a sale on UA's website, in the same transaction as the payment.

4.3 The Merchant may withdraw an item at any time by revoking the allowance or telling UA. UA removes a listing when the allowance is revoked or the NFT leaves the Merchant's account.

### 5. Sale on UA's Website: 70 / 30

5.1 When an item sells on UA's website, the Merchant receives **70%** of the sale price and UA receives **30%**. UA's 30% includes the 5% royalty that the Hedera token charges on the transfer.

5.2 The sale is one transaction on Hedera: the buyer's payment, the Merchant's share, UA's share and the NFTs move together, or nothing moves. The Merchant is paid in the coin the buyer used (`[HBAR / USDC / ...]`).

5.3 Example, on a sale of 1,000:

| | Merchant | UA |
|---|---|---|
| Paid in the sale transaction | 736.84 | 263.16 |
| Royalty, 5% of the Merchant's payment, taken by the token | −36.84 | +36.84 |
| **Result** | **700 (70%)** | **300 (30%)** |

Amounts are computed in the coin's smallest unit; any rounding remainder goes to the Merchant.

5.4 After the sale, the Merchant delivers the physical item to the buyer within `[n]` days, as described in Annex B. **(open: delivery terms, web #114)**

5.5 UA records the sale, and the delivery when it happens, on its public provenance record on Hedera.

### 6. Sales Outside UA's Website

6.1 The Merchant may sell an item anywhere: through a Hedera wallet, on another marketplace such as SentX, or in person. UA's 30% does not apply to those sales.

6.2 On every sale paid on the Hedera network, the token sends the 5% royalty to UA automatically.

6.3 If the Merchant sells an item off the network (cash, bank transfer, in person), the Merchant transfers the item's NFTs to the buyer's Hedera account. The buyer then pays the token's fixed fallback fee for each NFT received (currently about USD 3 in HBAR). The Merchant tells the buyer this before the sale.

6.4 The Merchant tells UA of any sale outside UA's website within `[n]` days, so UA can remove the listing and update its records.

### 7. Jewelry (Achala)

7.1 An Achala and the stones mounted in it are sold and transferred together. The Merchant does not transfer the NFT of a mounted stone on its own.

7.2 Only the Merchant may take apart a piece they own. Before doing so, the Merchant withdraws it from sale on UA's website and asks UA to burn the Achala NFT; the stones' NFTs are kept and can be sold on their own.

7.3 The Merchant pays the artisan who made the piece for the labor, separately from this agreement. The artisan receives no share of any sale, and UA has no obligation to the artisan.

### 8. Merchant Warranties

The Merchant confirms that, for each item:

- they own it and may sell it, and it is free of claims by others;
- it was obtained legally, and the origin documents given to UA are true and complete;
- it is not National Cultural Heritage, and is not a fossil or other item whose sale is restricted ([terms.md](terms.md), section 3);
- the description, weight, measurements and photos given to UA match the item.

If a warranty proves false, UA may remove the listing and record the problem on its provenance record, and the Merchant answers to the buyer.

### 9. Public Records and Personal Data

9.1 Hedera transactions and UA's provenance messages are public and permanent. They show account IDs, NFT serials and amounts, and cannot be deleted.

9.2 The Merchant's name and RUC are written into each item's NFT metadata, which is public and permanent; the Merchant agrees to this. UA does not put the Merchant's address, email or other personal data in those records.

9.3 UA handles the Merchant's personal data as described in its privacy policy.

### 10. Taxes

Each party is responsible for its own taxes on what it receives. UA issues `[a receipt / an invoice]` for its 30%. **(open: IGV on UA's commission, legal review, web #105)**

### 11. Term and Termination

11.1 This agreement starts on the date above and continues until either party ends it with `[n]` days' written notice.

11.2 On termination, UA removes the Merchant's listings, and the Merchant revokes UA's allowances. The NFTs stay in the Merchant's account; sales already made are not affected.

### 12. Liability

UA is not liable for network outages, wallet errors, changes in the price of HBAR or other coins, or for sales the Merchant makes outside UA's website. Neither party is liable for indirect losses.

### 13. Law and Disputes

This agreement is governed by the laws of Peru. The parties first try to settle any dispute by talking; if they cannot, it goes to the courts of `[Lima]`.

### Signatures

| Umiña Achala | Merchant |
|---|---|
| Name: | Name: |
| Signature: | Signature: |
| Date: | Date: |

---

## Annex A: Items

| # | Type | Stone ID | Description | Price | Token and serial (after minting) |
|---|---|---|---|---|---|
| 1 | Umiña | `[UMINA-...]` | `[e.g. quartz, 42 g]` | `[price]` | `[0.0.xxxx / n]` |
| 2 | Achala | `[ACHALA-...]` | `[e.g. silver ring with stones 3 and 4]` | `[price]` | `[0.0.xxxx / n]` |

For an Achala, list the mounted stones too; they are sold with it.

## Annex B: Delivery

`[How the physical item reaches the buyer after a sale on UA's website: in person in Lima, shipping in Peru, export. Who pays for shipping and insurance.]` **(open: web #114)**

## Annex C: Documents for Minting

`[The documents the Merchant gives UA for each item, e.g. RUC, REINFO registration, concession, purchase receipt, photos.]` **(open: #90 M1, legal review web #111)**
