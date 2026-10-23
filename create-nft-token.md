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

### Metadata Policy (Immutability)

In the Umiña Achala compliance model, metadata is treated as the immutable **birth certificate** of each NFT.

- **Frozen at Mint**  
  Metadata is set once during minting and never updated afterward. This ensures the NFT’s origin and compliance proofs remain permanent.

- **No Metadata Key Assigned**  
  Since updates are not permitted, no `metadataKey` is defined at token creation. This enforces immutability at the protocol level and avoids confusion for auditors or marketplaces.

- **Compliance Integration**  
  Provenance attributes (e.g., `concession_id`, `vendor_ruc`, `hs_code`, `export_permit`) are embedded in the metadata at mint time. These values are immutable and form the NFT’s identity.

- **Lifecycle Logging in HCS**  
  Any subsequent events (cutting, artisan attribution, export) are recorded in the Hedera Consensus Service (`compliance_proof_hcs`) rather than metadata. This preserves the integrity of the original record while maintaining full traceability.

**Summary:** Metadata is immutable, lifecycle events are logged externally, and no `metadataKey` is assigned. This policy guarantees that each NFT’s origin record is permanent and auditable.

### Royalty Policy

Every resale of an NFT automatically generates a royalty fee to ensure artisans and miners are rewarded beyond the initial sale.

- **Royalty Percentage**  
  Defined by `.setNumerator(5)` and `.setDenominator(100)`, this enforces a **5% fee** on each secondary transfer of the NFT.

- **Fee Collector**  
  Specified by `.setFeeCollectorAccountId(treasuryAccountId)`, the royalty is routed to the Treasury Account. The Treasury acts as the central vault, where funds can be redistributed to artisans, miners, or reinvested according to governance rules.

- **Governance Significance**  
  Royalties are enforced directly by Hedera’s token service through `.setCustomFees([...])`. This eliminates the need for external contracts, guarantees compliance, and centralizes control in the Treasury for auditability.

In practice, this policy ensures that the economic value of each NFT continues to support the community and compliance framework throughout its lifecycle.


## Token Creation (Hedera Token Service)

Execute a token creation transaction to establish the Umiña collection.

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

## Testnet Reference

- Current Testnet Token ID: `0.0.8130672`

---

This document establishes the **foundation** of the Umiña NFT lifecycle. Next, see [NFT Minting](ca://s?q=Suggested_nft-minting.md) for the process of creating individual stone NFTs.
