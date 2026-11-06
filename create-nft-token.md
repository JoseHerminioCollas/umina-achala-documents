# Umiña Achala: Create NFT Token

## Table of Contents

- [Umiña Achala: Create NFT Token](#umiña-achala-create-nft-token)
  - [Table of Contents](#table-of-contents)
  - [Purpose](#purpose)
  - [Token Parameters](#token-parameters)
    - [Roles \& Responsibilities](#roles--responsibilities)
    - [Metadata Policy (Immutability)](#metadata-policy-immutability)
    - [Royalty Policy](#royalty-policy)
  - [Token Creation (Hedera Token Service)](#token-creation-hedera-token-service)
  - [Provenance Registry (HCS)](#provenance-registry-hcs)
  - [Lifecycle Overview](#lifecycle-overview)
  - [Token Classes: Umiña and Achala](#token-classes-umiña-and-achala)
  - [Using the Keys After Creation](#using-the-keys-after-creation)
  - [Testnet Reference](#testnet-reference)


This document defines the **token creation process** for the Umiña Achala project, establishing the foundation for minting NFTs that represent Peruvian stones and minerals. It uses the **Hedera Token Service (HTS)** and integrates compliance safeguards through the **Hedera Consensus Service (HCS)**.

## Purpose

The Umiña token acts as the parent collection for all individual stones. Each NFT minted under this token represents a unique stone or mineral, anchored to regulatory and provenance data.

## Token Parameters

These parameters define the identity and configuration of the token at creation:

- **Name**  
  The human‑readable name of the token.  
  *Parameter:* `name`

- **Symbol**  
  The shorthand symbol used to represent the token in wallets and explorers.  
  *Parameter:* `symbol`

- **Type**  
  Defines whether the token is fungible or non‑fungible. For NFTs, this is set to `TokenType.NonFungibleUnique`.  
  *Parameter:* `tokenType`

- **Supply Type**  
  Determines whether the token supply is capped or unlimited. For NFTs, this is set to `TokenSupplyType.Infinite`, allowing ongoing minting of unique serials.  
  *Parameter:* `supplyType`

- **Initial Supply**  
  The number of tokens minted at creation. For NFTs, this is always `0` because minting occurs later.  
  *Parameter:* `initialSupply`

- **Decimals**  
  Defines divisibility of the token. For NFTs, this is always `0` since each token is unique and indivisible.  
  *Parameter:* `decimals`

- **Parameters associated with Roles**  
  See [Roles & Responsibilities](#roles--responsibilities)

- **Parameters associated with Royalties**  
  See [Royalty Policy](#royalty-policy)

### Roles & Responsibilities

The token lifecycle relies on several key roles defined at creation. Each role governs a specific aspect of token management and compliance.

- **Treasury Account**  
  Custodian of the token supply. All NFTs minted are first deposited into the Treasury before distribution. The Treasury acts as the vault, ensuring that provenance and compliance checks are centralized. It also collects fees and royalties.  
  *Key association:* `treasuryAccountId`

- **Admin Role**  
  Provides governance authority. This role can update global token properties, pause the token, or wipe assets if compliance fails. Best practice is to secure this role with multi‑signature controls.  
  *Key association:* `adminKey`

- **Supply Role**  
  Authorizes minting of new NFTs. The Supply Role holder is the “minter” in technical terms, but minted NFTs are always deposited into the Treasury Account first. This separation enforces governance and prevents uncontrolled minting.  
  *Key association:* `supplyKey`

- **Fee Schedule Role**  
  Manages royalty structures and custom fees. This role allows updates to fee percentages or collectors, ensuring artisans and miners are rewarded fairly.  
  *Key association:* `feeScheduleKey`

- **Pause Role**  
  Provides security control to temporarily halt token transfers.  
  This role is used in compliance scenarios where trading activity must be suspended  
  (e.g., pending regulatory review or suspected violation).  
  *Key association:* `pauseKey`

- **Wipe Role**  
  Provides security control to remove tokens from accounts if regulatory violations occur.  
  This role enforces compliance by allowing assets to be revoked when necessary,  
  ensuring that non‑compliant tokens cannot circulate.  
  *Key association:* `wipeKey`

- **Metadata Role**  
  Can rewrite the metadata of any serial, including serials already sold. **Not set**, by decision and in the code ([chain #1](https://github.com/umina-achala/umina-achala-chain/issues/1)); see [Metadata Policy](#metadata-policy-immutability).  
  *Key association:* `metadataKey`

Each of these keys keeps working on serials held by buyers. Pause, wipe and metadata keys are visible on marketplaces and explorers, and buyers can see them as a risk. A key that is not set at creation cannot be added later, and a key that is removed cannot be restored. See [nft-transfers-and-sales.md](nft-transfers-and-sales.md#what-ua-can-still-change-after-a-sale).

### Metadata Policy (Immutability)

In the Umiña Achala compliance model, metadata is treated as the immutable **birth certificate** of each NFT.

- **Frozen at Mint**  
  Metadata is set once during minting and never updated afterward. This ensures the NFT’s origin and compliance proofs remain permanent.

- **No Metadata Key Assigned**  
  Since updates are not permitted, no `metadataKey` is defined at token creation. This enforces immutability at the protocol level and avoids confusion for auditors or marketplaces.

  > **In the code** (2026-11-05, [chain #1](https://github.com/umina-achala/umina-achala-chain/issues/1)): `buildTokenTransaction` in `umina-achala-chain/src/token/lib.js` sets no metadata key, and the tests check that none is set even when an old `METADATA_KEY` is left in the env file. A key left off at creation can never be added, so no serial's metadata can be changed. See [nft-transfers-and-sales.md](nft-transfers-and-sales.md#what-ua-can-still-change-after-a-sale).

- **Compliance Integration**  
  Provenance attributes (e.g., `concession_id`, `vendor_ruc`, `hs_code`, `export_permit`) are embedded in the metadata at mint time. These values are immutable and form the NFT’s identity.

- **Lifecycle Logging in HCS**  
  Any subsequent events (cutting, artisan attribution, export) are recorded in the Hedera Consensus Service (`compliance_proof_hcs`) rather than metadata. This preserves the integrity of the original record while maintaining full traceability.

**Summary:** Metadata is immutable, lifecycle events are logged externally, and no `metadataKey` is assigned. This policy guarantees that each NFT’s origin record is permanent and auditable.

### Royalty Policy

Every resale of an NFT automatically generates a royalty fee to ensure artisans and miners are rewarded beyond the initial sale.

- **Royalty Percentage**  
  Defined by `.setNumerator(5)` and `.setDenominator(100)`, this enforces a **5% fee** on each secondary transfer of the NFT.

  The royalty is charged only when HBAR or a fungible token moves in the same transaction as the NFT. Gifts and sales paid off-chain or in a separate transaction pay nothing, unless a fallback fee is set. A fallback fee of about USD 3 in HBAR is **decided and in the code** ([chain #1](https://github.com/umina-achala/umina-achala-chain/issues/1), [chain #6](https://github.com/umina-achala/umina-achala-chain/issues/6)): `buildRoyaltyFee` in `umina-achala-chain/src/token/lib.js` adds it from `FALLBACK_FEE_HBAR` (38 HBAR on testnet). The HBAR amount drifts against the dollar; `node src/token/adminToken.js fees --fallback-hbar <hbar>` updates it ([chain #21](https://github.com/umina-achala/umina-achala-chain/issues/21)). See [nft-transfers-and-sales.md](nft-transfers-and-sales.md#when-the-royalty-is-paid).

- **Fee Collector**  
  Specified by `.setFeeCollectorAccountId(treasuryAccountId)`, the royalty is routed to the Treasury Account. The Treasury acts as the central vault, where funds can be redistributed to artisans, miners, or reinvested according to governance rules.

- **Governance Significance**  
  Royalties are enforced directly by Hedera’s token service through `.setCustomFees([...])`. This eliminates the need for external contracts, guarantees compliance, and centralizes control in the Treasury for auditability.

In practice, this policy ensures that the economic value of each NFT continues to support the community and compliance framework throughout its lifecycle.


## Token Creation (Hedera Token Service)

Execute a token creation transaction to establish the Umiña collection.

> **Simplified example.** The real transaction is built by `buildTokenTransaction` in `umina-achala-chain/src/token/lib.js`: it also sets the auto-renew account and adds the fallback fee to the royalty (`buildRoyaltyFee`), sets no metadata key, and takes the token class (Umiña or Achala).

**Example (JavaScript SDK):**
```javascript

const transaction = new TokenCreateTransaction()
    .setTokenName("Umiña")
    .setTokenSymbol("UMINA")
    .setTokenType(TokenType.NonFungibleUnique)
    .setTreasuryAccountId(treasuryAccountId)
    .setAdminKey(adminKey)
    .setSupplyKey(supplyKey)
    .setPauseKey(pauseKey)
    .setFeeScheduleKey(feeScheduleKey)
    .setCustomFees([
        new CustomRoyaltyFee()
            .setNumerator(5)
            .setDenominator(100)
            .setFeeCollectorAccountId(treasuryAccountId)
    ])
    .setWipeKey(wipeKey)
    .setSupplyType(TokenSupplyType.Infinite)
    .setInitialSupply(0)
    .setDecimals(0)
    .freezeWith(client);

```

## Provenance Registry (HCS)

> **Superseded.** The topics, their submit and admin keys, how they are created, and the current event format are in [create-hcs-topic.md](create-hcs-topic.md). The design changed from what this section describes: there are two topics (provenance and display), created separately from the token, each with a submit key; messages use the event envelope from [chain #10](https://github.com/umina-achala/umina-achala-chain/issues/10) (`event`, `schema_version`, `occurred_at`, `ref`, ...) rather than `event_type` and `data_hash`. This section is kept for its background until the end-of-session docs refresh.

Along with the token creation, a Master Hedera Consensus Service (HCS) topic is established to serve as the registry for all Umiña NFTs. This registry ensures that compliance and provenance data remain immutable and auditable throughout the lifecycle of each stone.

- **Topic ID**  
  A single HCS topic is created at the same time as the token and used to record lifecycle events for all NFTs in the Umiña collection.

- **Message Format**  
  Each message captures the lifecycle event of a stone and includes:  
  • `stone_id` → the unique identifier of the NFT  
  • `event_type` → the type of lifecycle event (such as mint, cutting, artisan assignment, export)  
  • `timestamp` → the Hedera consensus timestamp for the event  
  • `actor_id` → the account or entity responsible for the event  
  • `data_hash` → a hash of supporting documents or compliance proofs

**Example (JavaScript SDK):**
```javascript

// Example provenance messages for different lifecycle events

// Mint event: metadata birth certificate
const mintMessage = {
    stone_id: "UMINA-000124",
    event_type: "mint",
    timestamp: new Date().toISOString(),
    actor_id: "0.0.123456",
    // Best practice: SHA‑256 hash of the compliance document(s)
    data_hash: crypto.createHash("sha256")
                     .update(fs.readFileSync("compliance_docs/minting.pdf"))
                     .digest("hex")
};

// Export event: regulatory clearance
const exportMessage = {
    stone_id: "UMINA-000124",
    event_type: "export",
    timestamp: new Date().toISOString(),
    actor_id: "0.0.456789",
    // Again, SHA‑256 hash of export permit or customs docs
    data_hash: crypto.createHash("sha256")
                     .update(fs.readFileSync("compliance_docs/export_permit.pdf"))
                     .digest("hex")
};

// Submit to Hedera Consensus Service topic
await new ConsensusMessageSubmitTransaction()
    .setTopicId(provenanceTopicId)
    .setMessage(JSON.stringify(mintMessage))
    .execute(client);

await new ConsensusMessageSubmitTransaction()
    .setTopicId(provenanceTopicId)
    .setMessage(JSON.stringify(exportMessage))
    .execute(client);

```

This structure guarantees:  
- **Auditability**: every event is permanently recorded with consensus time.  
- **Traceability**: each NFT’s lifecycle can be reconstructed step by step.  
- **Compliance Proofs**: external documents are referenced via `data_hash`, ensuring integrity without storing sensitive data directly on‑chain.

## Lifecycle Overview

The Umiña NFT lifecycle is designed to ensure compliance, provenance, and community benefit at every stage:

1. **Token Creation**  
   The parent collection is established using the Hedera Token Service (HTS). Governance roles, royalty policy, and security keys are defined at this stage.

2. **Metadata Policy**  
   Each NFT minted under the token receives immutable metadata — its “birth certificate.” Compliance attributes are embedded at mint time and never altered afterward.

3. **Provenance Registry (HCS)**  
   Alongside token creation, a Hedera Consensus Service (HCS) topic is created to log lifecycle events. Cutting, artisan attribution, and export events are recorded here with consensus timestamps and document hashes.

4. **Minting**  
   Individual NFTs representing stones and minerals are minted under the parent token. Metadata anchors their identity, while provenance events are logged externally in HCS.

5. **Royalties & Distribution**  
   Secondary sales automatically enforce royalty payments routed to the Treasury Account, ensuring artisans and miners benefit beyond the initial sale.

**Summary:** The lifecycle flows from token creation → immutable metadata → provenance registry → minting → royalties.

## Token Classes: Umiña and Achala

Decided 2026-10-31 ([chain #15](https://github.com/umina-achala/umina-achala-chain/issues/15)): jewelry gets its own token class, separate from stones, with its own name and symbol in wallets and its own metadata schema.

| | Umiña | Achala |
|---|---|---|
| For | A single stone (a gold nugget is a stone) | A piece of jewelry made of one or more Umiña stones |
| Name, symbol | Umiña, `UMINA` | Achala, `ACHALA` |
| Memo | Umiña: Peruvian stones | Achala: Peruvian jewelry made of Umiña stones |
| Env variable for its ID | `UMINA_TOKEN_ID` | `ACHALA_TOKEN_ID` |
| Everything else | Shared: non-fungible, infinite supply, treasury, the five keys, **no metadata key**, 5% royalty with the HBAR fallback fee, auto-renew by the treasury | Same |

Both are built by the same `buildTokenTransaction` (`umina-achala-chain/src/token/lib.js`, `TOKEN_CLASSES`), so the two classes cannot drift apart. The Achala royalty may later differ (an artisan share); until decided it is the Umiña one, and `adminToken.js fees --token achala` can change it.

### Achala Pieces ([chain #16](https://github.com/umina-achala/umina-achala-chain/issues/16), decided 2026-11-05, updated 2026-11-06)

- **An Achala is one or more Umiña NFTs.** It lists the Umiña stones it is made of, and the Umiña NFTs are associated with it: its `MINT` event names them (`from_stone_ids`), which records them as mounted.
- **A stone can be gold.** A gold nugget is a stone, so it can be an Umiña. A gold piece is either an Umiña itself or an Achala made of a gold Umiña. The gold or silver of an Achala comes from one of its Umiña stones or from an invoice in its documents.
- **The buyer gets every NFT.** A sale moves the Achala and each Umiña mounted in it together (N+1 NFTs for N stones); the buyer is associated with both tokens. `sellOwnItem.js --token achala` does it ([chain #26](https://github.com/umina-achala/umina-achala-chain/issues/26), on branch `achala-sale-mounted-stones` of the chain fork, in review (2026-11-06)). Hedera allows at most 10 NFT transfers in one transaction, so an Achala has at most 9 stones (the schema maximum); the limit is to verify on testnet.
- **A stone in an Achala moves only with it.** Selling, consigning, burning or wiping a stone mounted in an Achala that still exists is refused; burning the Achala frees it ([chain #24](https://github.com/umina-achala/umina-achala-chain/issues/24)).
- **Taking a piece apart burns the Achala** (`adminToken.js burn` or `wipe`, `--token achala`), recorded as `BURN`. **The Umiña NFTs are kept**: once their Achala is burned, they can be mounted in a new Achala or sold on their own. The Achala's `BURN` unmounts its stones.
- **The stones get no events of their own** (decided 2026-11-06). Mounting and unmounting are recorded in the Achala's HCS events, its `MINT` and its `BURN`; there is no per-stone `TRANSFORMATION` or `UNMOUNTED` event.
- **One owner per piece.** Several owners of one piece (each holding a stone NFT of it) is not supported: that suits real estate, not jewelry.
- **Pieces with many stones:** an NFT for every stone does not scale; accent stones without an NFT each, or one Umiña per lot, are [chain #23](https://github.com/umina-achala/umina-achala-chain/issues/23), not decided yet.

The Achala metadata schema and the mint (`mintNFT.js --token achala`) are written, on the `main` branch of the `umina-achala-chain` fork (not yet upstream); see [mint-nft-token.md](mint-nft-token.md#minting-an-achala).

## Using the Keys After Creation

`umina-achala-chain/src/token/adminToken.js <command> [--token umina|achala]` uses the admin keys ([chain #21](https://github.com/umina-achala/umina-achala-chain/issues/21)), each command signed by its own key:

| Command | Key | Records |
|---|---|---|
| `fees --fallback-hbar <hbar> [--royalty <percent>]` | Fee Schedule | nothing (token-wide) |
| `pause` / `unpause` | Pause | nothing (token-wide) |
| `wipe <serial> --account <id> --reason <text>` | Wipe | `BURN` (`method: "wipe"`) |
| `burn <serial> --reason <text>` | Supply | `BURN` (`method: "burn"`) |

wipe and burn check the holder first; the reason is public and permanent, so no personal data.

## Testnet Reference

- **Creating a token:** `node src/token/createToken.js [env-file] [--token umina|achala] [--dry-run]` in `umina-achala-chain` (`--token=achala` works too; an unknown class or option is refused before anything is read). The project owner runs it; tokens and topics are created once all Phase 2A code is written (decided 2026-11-05), so **no current Umiña or Achala testnet token exists yet**.
- The old token `0.0.8130672` ("Rumi Peruvian Gemstones", created 2026-03-08 by an earlier script, with a metadata key and no fallback fee) does not follow the current decisions and will not be used; whether to delete it is open on [chain #1](https://github.com/umina-achala/umina-achala-chain/issues/1).

---

This document establishes the **foundation** of the Umiña NFT lifecycle. Next, see [NFT Minting](ca://s?q=Suggested_nft-minting.md) for the process of creating individual stone NFTs.
