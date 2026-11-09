# Umiña NFT Minting Guide
- [Umiña NFT Minting Guide](#umiña-nft-minting-guide)
  - [Overview](#overview)
  - [Current Implementation (Phase 2 testnet)](#current-implementation-phase-2-testnet)
    - [Steps](#steps)
    - [Photos](#photos)
    - [Metadata Rules](#metadata-rules)
    - [Minting an Achala](#minting-an-achala)
  - [Individual Stone Lifecycle](#individual-stone-lifecycle)
    - [🛠️ Umiña Achala Minting Workflow (Summary)](#️-umiña-achala-minting-workflow-summary)
      - [Step 1: Prepare Compliance Payload (metadata)](#step-1-prepare-compliance-payload-metadata)
      - [Step 2: HCS (Hedera Consensus Service ) Birth Certificate](#step-2-hcs-hedera-consensus-service--birth-certificate)
      - [Step 3: Metadata Anchoring](#step-3-metadata-anchoring)
      - [Step 4: HTS Minting](#step-4-hts-minting)
      - [Step 5: Verification](#step-5-verification)
    - [Transformation (Events)](#transformation-events)
    - [Sales \& Export](#sales--export)
  - [Minting \& Notarization](#minting--notarization)
    - [HTS Minting](#hts-minting)
    - [Metadata Anchoring](#metadata-anchoring)
    - [HCS Birth Certificate](#hcs-birth-certificate)
  - [Transformation (Events)](#transformation-events-1)
    - [Event Types](#event-types)
    - [Immutable Metadata](#immutable-metadata)
    - [Hedera Consensus Service (HCS) Logging](#hedera-consensus-service-hcs-logging)
    - [Purpose](#purpose)
  - [Sales \& Economic Management](#sales--economic-management)
    - [Sales \& Royalties](#sales--royalties)
    - [Export Integration](#export-integration)
    - [Export Automation Workflow](#export-automation-workflow)
    - [Artisan \& Marketplace Onboarding](#artisan--marketplace-onboarding)
    - [Economic Mechanisms](#economic-mechanisms)
    - [Purpose](#purpose-1)
  - [📱 dApp, Wallet, and Mobile App Integration](#-dapp-wallet-and-mobile-app-integration)
    - [🔍 Metadata Lookup](#-metadata-lookup)
    - [🖼️ Wallet Display](#️-wallet-display)
    - [📲 Mobile App Workflow](#-mobile-app-workflow)
    - [🛠️ Developer Notes](#️-developer-notes)
  - [Umiña Achala Website](#umiña-achala-website)
    - [Visibility \& Promotion](#visibility--promotion)
    - [Wallet \& Transaction Layer](#wallet--transaction-layer)
    - [Compliance \& Provenance](#compliance--provenance)
    - [Artisan \& Vendor Onboarding](#artisan--vendor-onboarding)
    - [Purpose](#purpose-2)
  - [Conclusion](#conclusion)

## Overview
This guide defines the minting process for Umiña NFTs, covering lifecycle events, compliance automation, metadata standards, and export integration. Each stone undergoes a (Minting → Transformation → Sales/Export) lifecycle, notarized on Hedera Consensus Service (HCS) and minted via Hedera Token Service (HTS).

## Current Implementation (Phase 2 testnet)

> As of 2026-11-06, on the `main` branch of the `umina-achala-chain` fork (not yet upstream). This section describes what the mint script actually does. The sections after it are the original design and are kept as background; where they differ (one combined "birth certificate" message, image handling, field names), **this section is current**.

`node src/mint/mintNFT.js <metadata-file | --form post-file> [--token umina|achala] [--out dir] [--force] [--dry-run | --images-only]`

One script mints both token classes, as `createToken.js` creates both: an Umiña (a stone) by default, an Achala (jewelry) with `--token achala`. Metadata of the other class is refused by its `stone_id`, with a hint to give `--token`.

### Steps

1. **Check, before anything leaves the machine:**
   - the environment (Pinata key with file and JSON pinning, token, topic, supply key; not needed with `--images-only`)
   - the photos (below)
   - the metadata against the schema and the [rules](#metadata-rules)
2. **Read the provenance topic** from the mirror node. Refuse if it does not exist, or if the `stone_id` already has a `MINT` event (unless `--force`) ([chain #20](https://github.com/umina-achala/umina-achala-chain/issues/20)).
3. **Record `COMPLIANCE`** (stone type, RUC, concession, REINFO, HS code). Its transaction ID becomes `compliance_proof_hcs` in the metadata, which can only point to a message that already exists ([chain #14](https://github.com/umina-achala/umina-achala-chain/issues/14)). If this fails, nothing is pinned.
4. **Make the image versions** in `generated/<stone_id>/` and **pin** them and the documents, then the metadata ([chain #7](https://github.com/umina-achala/umina-achala-chain/issues/7)).
5. **Mint** under `UMINA_TOKEN_ID` (an Achala: `ACHALA_TOKEN_ID`), signed by the supply key; the NFT's on-chain metadata is the metadata's `ipfs://` URI.
6. **Record `MINT`** with the mint transaction as `ref` ([chain #3](https://github.com/umina-achala/umina-achala-chain/issues/3), [#17](https://github.com/umina-achala/umina-achala-chain/issues/17)).

An Achala adds a stone check after step 2, and its `MINT` lists its stones; see [Minting an Achala](#minting-an-achala).

If something fails after pinning, the pinned CIDs are listed so they can be unpinned; if `MINT` fails after the mint, the serials and URI are printed so it can be recorded by hand. `--dry-run` runs every check and lists the files it would make, writing nothing; `--images-only` also writes the images and prints the metadata it would pin, without pinning, minting or credentials.

### Photos

| | Rule |
|---|---|
| Fields | `main_image` (required) and `images` (up to 5 more), each `{ file, description }`; a photo listed twice is refused |
| Accepted | JPEG, PNG or WebP, checked by the real file type; up to 25 MB and about 100 megapixels |
| Refused | GIF, HEIC (export iPhone photos as JPEG), RAW |
| Master, every photo | full size, the upload's format (JPEG and WebP at quality 95, PNG lossless); a `files[]` entry with its SHA-256 `checksum`; the main photo's master is the default file |
| Display, main photo only | 1080 px on the long side, JPEG quality 82, never enlarged; becomes `image`, which wallets and marketplaces show |
| Privacy | every version is turned upright and written **without EXIF, XMP or GPS**, so no location of a mine reaches IPFS |
| Thumbnails | none: the web site and the mobile app fetch the masters and resize them as they need (decided 2026-11-05) |

Receipts and permits go in `documents` (PDF or GLB) and are pinned as they are, as `files[]` entries after the masters.

### Metadata Rules

Checked by `src/mint/metadataValidator.js` against `schema/hip412-umina-schema.json`, which lives in `umina-achala-chain` (umina-achala-web keeps a copy; see [metadata.md](metadata.md#current-schema-2026-11-05)):

- **Certified stones only** ([chain #14](https://github.com/umina-achala/umina-achala-chain/issues/14)): every provenance field is required.
- **`stone_type`** must be an id from `schema/stone-types.json`: 40 curated types plus the 6,222 IMA-approved mineral species (imported from RRUFF by `src/mint/importIma.js`). There is no "other": a new type is added to the list first. Deprecated types are refused, and so are **fossils** (Ley 31204).
- **`materials`** (optional) lists a multi-material specimen, such as quartz with gold: exactly one host, which is the `stone_type`.
- **IDs** ([chain #5](https://github.com/umina-achala/umina-achala-chain/issues/5)):
  - `vendor_ruc`: 11 digits, prefix 10, 15, 17 or 20, with SUNAT's check digit verified
- **Seller** ([chain #31](https://github.com/umina-achala/umina-achala-chain/issues/31)): `vendor_name` is required, the seller's name next to its RUC. `creator` is the seller or the artisan, as the merchant chooses; the form's `creator` field sets it, and without one it stays "Umiña Achala". Both are public and permanent. They are not copied into the `COMPLIANCE` event.
  - `reinfo_id`: REINFO is tracked by the miner's RUC, so this is a RUC with the same checks
  - `mining_concession`: the INGEMMET Código Único, exactly 9 digits. Strict on purpose; the pattern is widened only after a different format is confirmed on the INGEMMET record.
- **The `Stone Type`, `Materials` and `Form` attributes** are written from the list, replacing any typed by hand.
- **Fields that change after the mint** (`reinfo_status`, `last_regulatory_check`, `sinceramiento_status`, `export_status`) record the state at the mint; later changes are HCS events. Custody (`current_location`) is no longer in the metadata.
- The required fields are to be confirmed by the legal review ([web #111](https://github.com/umina-achala/umina-achala-web/issues/111)) before the first real mint, since metadata is immutable.

### Minting an Achala

[chain #16](https://github.com/umina-achala/umina-achala-chain/issues/16). An Achala is a piece of jewelry made of one or more Umiña NFTs (see [Achala Pieces](create-nft-token.md#achala-pieces-chain-16-decided-2026-11-05-updated-2026-11-06)).

```bash
node src/mint/mintNFT.js --form json/sample-achala-form-post.json --token achala --dry-run
```

**Schema:** `schema/hip412-achala-schema.json` (template `json/achala-template.json`, example `json/achala-example.json`).

| Field | Rule |
|---|---|
| `stone_id` | `ACHALA-YYYY-XX-NN`, e.g. `ACHALA-2026-RI-01` (the item ID; called `stone_id` like the Umiña one, since the HCS events key every item by it) |
| `piece_type` | `ring`, `pendant`, `necklace`, `earrings`, `bracelet` or `brooch` |
| `umina_token_id` | The Umiña token every stone is in, named once for the piece. The mint fills it from `UMINA_TOKEN_ID`, also in the file generated from a form post, and refuses any other value |
| `stones` | 1 to 9 Umiña NFTs (an Achala is sold with its stones in one transfer, at most 10 NFTs), each `{ stone_id, serial }` (a stone has no `token_id` of its own), each listed once |
| `metals` | Optional. Each `{ stone_type, fineness, weight_g, source }`: `stone_type` must be a metal from the list; `source: "umina"` names one of the piece's stones in `stone_id` (e.g. a UA gold nugget), `source: "invoice"` needs the invoice in `documents` |
| `technique`, `artisan_rna` | Required (how it was made; the artisan's Registro Nacional del Artesano) |
| `hs_code`, `jurisdiction`, `legal_uri` | Required; jewelry of precious metal 7113.19 (silver 7113.11), of stones 7116.20, to confirm ([documents #1](https://github.com/umina-achala/umina-achala-documents/issues/1)) |
| `vendor_name`, `vendor_ruc` | Required; the RUC's check digit verified as for an Umiña |

The mining fields (concession, REINFO) are on the Umiña stones, not on the Achala.

**Extra steps for an Achala:**

- **Environment:** also `ACHALA_TOKEN_ID`, plus `UMINA_TOKEN_ID` and `TREASURY_ID` to check the stones. A `umina_token_id` other than `UMINA_TOKEN_ID` is refused before the mirror node is read, also on `--dry-run`.
- **Stone check, before anything is posted** (`--force` does not skip it): each stone is in the Umiña token, has a `MINT` event with that `stone_id` and serial, exists and is not burned (mirror node), is **held by the treasury**, and is not mounted in another Achala that still exists (listed in the `from_stone_ids` of an Achala `MINT` whose serial has no `BURN`).
- **`COMPLIANCE`:** `piece_type`, `artisan_rna`, `vendor_ruc`, `hs_code`, `from_stone_ids`.
- **`MINT`:** adds `from_stone_ids`, which records the stones as mounted. The stones get no events of their own: the Achala's `MINT` mounts them and its `BURN` (the piece taken apart) unmounts them. If the `MINT` event fails, the script says the stones are not recorded as mounted.
- **Attributes written from the data:** `Piece Type`, `Metal` (e.g. `Gold 18k`), `Stones` (each stone's type name, from its `COMPLIANCE` event), `Artisan`. Hand-written ones with these names are replaced.

The largest `MINT` (9 stones) fits in one 1 KB HCS message; a test checks it.

**Checking a file without minting:** `node src/mint/validateMetadata.js <file>` picks the Umiña or Achala schema from the `stone_id`. It counts `documents` as files, so a metal whose `source` is `invoice` passes when the invoice is attached, as in the mint.

**Pieces with many stones** (pavé, beads) would need an NFT for every stone. Simplifying this, with accent stones listed in the metadata without an NFT each, or one Umiña per lot of small stones, is [chain #23](https://github.com/umina-achala/umina-achala-chain/issues/23), not decided yet.

## Individual Stone Lifecycle
### 🛠️ Umiña Achala Minting Workflow (Summary)

#### Step 1: Prepare Compliance Payload (metadata)
- Collect Provenance Data
- Validate concession, miner, and vendor IDs using official registries.  
- Confirm HS Code classification and retrieve export certificate if applicable.
- Gather all validated fields into a structured payload.
- Generate and set `stone_id` automatically in the script by concatenating stone_type and the current date (e.g., opal-2026-06-12T17:36:12.123Z).
- Generate and set `timestamp` and set property

#### Step 2: HCS (Hedera Consensus Service ) Birth Certificate
- Submit a minting message to the Master Hedera Consensus Service (HCS) Topic.  
- Use metadata `stone_id` in message
- This notarizes the NFT creation with a consensus timestamp, serving as its “birth certificate.”
- Retain the HCS transaction ID to be used in metadata the `compliance_proof_hcs`.

#### Step 3: Metadata Anchoring
- Record the HCS transaction ID in the NFT metadata as `compliance_proof_hcs`.  
- Pin the HIP‑412 JSON metadata to IPFS.  
- Link the CID(Content Identifier,the hash generated by IPFS when metadata is pinned) to the NFT so provenance and compliance data remain immutable and accessible.

#### Step 4: HTS Minting
- Mint a new serial under the existing **Umiña Token ID** using Hedera Token Service.  
- This establishes the NFT’s unique digital identity.
- Link the CID(Content Identifier) generated by IPFS.

#### Step 5: Verification
- Customs officers or buyers query Hedera using the compliance proof reference.  
- Verify that the payload matches the NFT metadata.  
- Confirm provenance and artisan authenticity before trade or export.

### Transformation (Events)
- **Artisan Intervention**: The stone or mineral is processed by a registered artisan, with cutting, polishing, or setting work documented and the artisan’s official RNA (Registro Nacional del Artesano) ID recorded.  
- **Event Logging**: A TRANSFORMATION message is submitted to the Hedera Consensus Service (HCS) Master Topic, linking the finished piece back to its raw origin. The log includes the stone ID, event type, timestamp, actor ID, and a SHA‑256 hash of supporting off‑chain artifacts (such as scans or photographs).  

### Sales & Export
- Sales: Primary and secondary handled natively by HTS.
- Royalty Engine: 5% royalty routed to Umiña Achala.
- Fee Adjustment: Fee Schedule Key updates royalties across the collection.
- Export & Final Provenance:
  - Customs Verification: NFT acts as Digital Packing List for SUNAT.
  - HCS Final Notarization: Export Permit Number hashed to Master HCS Topic.

## Minting & Notarization

When an Umiña stone enters the blockchain lifecycle, three key actions occur:

### HTS Minting
- A new **serial number** is created under the "Umiña" Token ID using the **Hedera Token Service (HTS)**.
- This establishes the stone’s **unique digital identity**, ensuring it can be tracked and traded within the collection.
- The minting event is immutable, serving as the **foundation of provenance** for the stone.

### Metadata Anchoring
- A JSON file compliant with the **Hedera Improvement Proposal (HIP‑412)** standard is generated and **pinned to the InterPlanetary File System (IPFS)**.
- IPFS produces a **Content Identifier (CID)** — a cryptographic hash that uniquely represents the metadata file.
- Because metadata is frozen at minting by design, the CID ensures immutability: the NFT always points to the exact metadata file created at mint time.
- The metadata includes:
  - **Compliance Automation & Regulatory Database Integration**  
    - MINEM (REINFO): Verify concession is Vigente.  
    - SUNAT (RUC): Confirm shop is a legal entity.  
    - VUCE: Validate export permits and Certificates of Origin.  
    - Ley 32537 (2026): Mandatory georeferenced coordinates submission.  
    - Census Gate (Post‑June 2026): INEI Census Participation ID required.  
  - **HCS Proof of Compliance**  
    - Immutable timestamping of government queries.  
    - Cryptographic linkage via Topic ID embedded in metadata.  
    - Auditability for customs officers and buyers.  
  - **Legal Status Field**: Values such as `certified_origin`, `retail_acquisition`, or `legacy_collection`.  
  - **Identification**: Vendor RUC or Artisan RNA ID.  
  - **Compliance Fields**: Last regulatory check timestamp and REINFO status.  
  - **HS Code Mapping**:  
    - Semi‑Precious Stones: 7103.99.00.00  
    - Emeralds: 7103.91.00.00  
    - Gold: 7108.12.00.00  
    - Silver: 7106.91.00.00  
    - Pyrite: 2502.00.00.00  
  - **Physical‑to‑Digital Anchoring**  
    - Optical fingerprinting: macro‑photography of inclusions uploaded to IPFS.  

### HCS Birth Certificate
- A **Minting message** is submitted to the **Hedera Consensus Service (HCS)** Master Topic.
- This message records the stone’s ID, event type, timestamp, actor, and data hash.
- It acts as the stone’s **birth certificate**, notarizing its creation with a consensus timestamp and ensuring auditability.


## Transformation (Events)

Each stone or mineral, once minted and notarized, may go through multiple **Transformation events**. These events document the stone’s journey without altering the immutable NFT metadata. Instead, they are captured as notarized logs on the **Hedera Consensus Service (HCS)** and linked to off‑chain artifacts stored in the **InterPlanetary File System (IPFS)**.

### Event Types
- **Cutting & Artistry**: The raw stone or mineral is processed by a registered artisan, with the artisan’s official RNA (Registro Nacional del Artesano) ID recorded.  
- **Sale**: Records primary or secondary market transfer, including buyer and seller identifiers.  
- **Export**: Customs clearance and export permit details logged, linking the stone to international trade records.  
- **Gift**: Records voluntary transfer of ownership without payment, capturing donor and recipient identifiers.  
- **Stolen**: Event marking theft, with supporting police or regulatory documentation hashed and logged.  
- **Lost**: Event marking loss, with timestamp and contextual details recorded for provenance integrity.  
- **New Photos**: Digital Artifacts: New scans, macro‑photography, or optical fingerprints of the finished piece are generated and stored off‑chain (e.g., pinned to the InterPlanetary File System (IPFS) with their own Content Identifiers (CID)).  

### Immutable Metadata
- The NFT’s metadata created at minting is **never updated**.  
- Transformation events are external records, ensuring the original “birth certificate” metadata remains frozen while the stone’s lifecycle is fully auditable.

### Hedera Consensus Service (HCS) Logging
Each Transformation event generates a message submitted to the Hedera Consensus Service (HCS) Master Topic.  
The message includes:
- Stone ID  
- Event type (e.g., TRANSFORMATION, SALE, EXPORT, GIFT, STOLEN, LOST, NEW_PHOTOS)  
- Timestamp (ISO 8601 format)  
- Actor ID (artisan, vendor, customs officer, donor, recipient, etc.)  
- SHA‑256 hash of supporting off‑chain documents  
- InterPlanetary File System (IPFS) Content Identifier (CID) for new photos or scans  

### Purpose
- Transformation events ensure every stage of the stone’s journey is **documented, verifiable, and tamper‑proof**.  
- They provide a complete audit trail from mine to marketplace, including exceptional cases like theft, loss, or gifting.  
- By combining Hedera Consensus Service (HCS) notarization with InterPlanetary File System (IPFS) storage, the system guarantees both **immutability** and **transparency**.

## Sales & Economic Management

The Sales & Economic Management stage governs how Umiña NFTs are traded, monetized, and prepared for export. It ensures that every transaction is compliant, auditable, and linked to regulatory frameworks.

### Sales & Royalties
- **Primary Sales**: Initial transfers of ownership are handled natively by the Hedera Token Service (HTS).  
- **Secondary Sales**: Resales are also managed on‑chain, ensuring provenance continuity.  
- **Royalty Engine**: A fixed 5% royalty is automatically routed to Umiña Achala on every transaction, guaranteeing sustainable revenue for the ecosystem.  
- **Fee Adjustments**: The Fee Schedule Key allows updates to royalty percentages or transaction fees across the collection, ensuring adaptability to market conditions.  
- **Fractional Ownership**: Stones can be fractionally owned, with shares represented as fungible tokens paired with liquidity pools. This allows multiple buyers to co‑own a single stone and trade their shares independently.  
- **Marketplace Display**: All stones are listed with detailed provenance, artisan profiles, and compliance records, whether available for sale or already exported.

### Export Integration
Export is treated as the culmination of the sales process, where domestic trade transitions into international commerce.  
- **Customs Verification**: The NFT acts as a digital packing list for SUNAT, linking the stone to its compliance metadata.  
- **Final Notarization**: Export permit numbers are hashed and submitted to the Hedera Consensus Service (HCS) Master Topic.  
- **Legal Status Freeze**: The NFT’s metadata includes a legal status field updated to `Exported`, preventing further domestic transactions.  
- **QR Code Presentation**: A QR code linked to the NFT’s metadata and compliance records is generated. Customs agents can scan the code to instantly retrieve the stone’s HS code, Certificate of Origin, and notarized compliance proofs directly from the **Umiña Achala Website**.  

### Export Automation Workflow
1. **Selection**: Buyer chooses the stone via the Umiña Achala Website marketplace.  
2. **Lookup**: The decentralized application queries Hedera Improvement Proposal (HIP‑412) metadata for the Harmonized System (HS) code.  
3. **Certificate Generation**: The VUCE API issues a Digital Certificate of Origin.  
4. **Form Population**: DEF or VUCE forms are auto‑filled using metadata fields.  
5. **Compliance Check**: Chapter 97 (Patrimonio Cultural) is blocked to prevent export of protected cultural heritage.  
6. **Digital Signature Hashing**: XAdES‑BES signatures are hashed to the Hedera Consensus Service (HCS), ensuring cryptographic proof of export authorization.  

### Artisan & Marketplace Onboarding
- **Artisan Profiles**: Each artisan is assigned a decentralized identifier (DID), enabling verifiable digital identity.  
- **RNA Verification**: Artisan RNA IDs are cross‑referenced with the MINCETUR database to confirm authenticity.  
- **Marketplace Integration**: Onboarding workflows ensure that only verified artisans and compliant vendors can list stones on the Umiña Achala Website, maintaining ecosystem integrity.  

### Economic Mechanisms

| Mechanism              | Description                                                                 | Where Handled                          |
|------------------------|-----------------------------------------------------------------------------|----------------------------------------|
| **Primary Sales**      | Initial transfers of ownership managed natively by Hedera Token Service (HTS). | Sales & Royalties                      |
| **Secondary Sales**    | Resales tracked on‑chain to preserve provenance continuity.                  | Sales & Royalties                      |
| **Royalties**          | Fixed 5% royalty automatically routed to Umiña Achala on every transaction.          | Sales & Royalties                      |
| **Fee Adjustments**    | Fee Schedule Key allows updates to royalty percentages or transaction fees.  | Sales & Royalties                      |
| **Fractional Ownership** | Stones can be fractionally owned, represented as fungible token shares paired with liquidity pools. | Sales & Royalties / Umiña Achala Website       |
| **Export Integration** | Customs verification, export permits, and legal status freeze logged via Hedera Consensus Service (HCS). | Export Integration                     |
| **QR Code Presentation** | QR codes generated for customs agents to instantly verify HS codes, Certificates of Origin, and compliance proofs. | Export Integration / Umiña Achala Website      |


### Purpose
- This stage ensures that every sale, resale, and export is **transparent, compliant, and auditable**.  
- By combining HTS transaction logic, HIP‑412 metadata, QR code presentation, and HCS notarization, the **Umiña Achala Website** becomes the central hub for bridging domestic trade and international export.  

## 📱 dApp, Wallet, and Mobile App Integration

Umiña NFTs are designed to be portable across decentralized applications (dApps), wallets, and mobile interfaces. This section explains how HIP‑412 metadata is queried and displayed in user‑facing apps.

### 🔍 Metadata Lookup
- **dApp Query:** The application queries the HIP‑412 metadata pinned to IPFS.  
- **HS Code Retrieval:** The embedded `hs_code` field is used to auto‑populate customs/export forms.  
- **Compliance Anchors:** Fields like `reinfo_id`, `vendor_ruc`, and `vuce_cod` are parsed to confirm regulatory status before enabling trade or export actions.  
- **QR Code Generation:** A QR code is dynamically generated from the NFT’s `stone_id` and `compliance_proof_hcs`. This allows customs officers, buyers, or artisans to scan and instantly retrieve provenance and compliance proofs.

### 🖼️ Wallet Display
- **Visual Assets:** Wallets (e.g., HashPack, MetaMask with Hedera integration) render the `image` URI and extended `files` (3D scans, PDFs).  
- **Attributes Panel:** The `attributes` array is displayed as trait/value pairs, showing provenance, artisan details, and compliance status.  
- **Compliance Badge:** Wallets can highlight `Verification Level` and `Legal Status` as badges (e.g., “Full 2026 Compliance”).  
- **Audit Link:** The `hcs_compliance_topic` or `audit_log_uri` provides a clickable link to Hedera Consensus Service for proof verification.  
- **QR Code Display:** Wallets and mobile apps display a scannable QR code linked to the NFT’s compliance record. This enables offline verification and quick customs processing.

### 📲 Mobile App Workflow
1. **Asset Selection:** User taps an NFT in their wallet or marketplace app.  
2. **Metadata Fetch:** App retrieves HIP‑412 JSON from IPFS.  
3. **Compliance Check:** App runs a lightweight validation (REINFO status, Census ID, payroll flag).  
4. **UI Rendering:**  
   - Image thumbnail + high‑res file preview.  
   - Attributes displayed in a scrollable card.  
   - Export status (`is_exported`, `current_location`) shown with icons.  
   - QR code displayed for instant provenance verification.  
5. **Action Buttons:**  
   - **Verify Compliance:** Opens HCS audit trail.  
   - **Export Certificate:** Displays VUCE COD reference.  
   - **Artisan Profile:** Links to RNA verification and artisan details.  
   - **Scan QR:** Customs officers or buyers can scan the QR code to retrieve compliance proofs without navigating registries.

### 🛠️ Developer Notes
- **Standardization:** Use HIP‑412 schema validation before rendering.  
- **Caching:** Mobile apps should cache IPFS assets locally for offline provenance replay.  
- **Cross‑Platform:** Ensure consistent display across web dApps, desktop wallets, and mobile apps.  
- **Security:** Metadata updates must be anchored in HCS to prevent tampering; apps should reject unanchored updates.  
- **QR Integration:** QR codes should encode both `stone_id` and `compliance_proof_hcs` for redundancy, ensuring verification even if one field is missing.

This integration ensures that **buyers, customs officers, and artisans** can view provenance and compliance data directly inside their wallets or mobile apps, with QR codes providing a fast, offline‑friendly verification method.


## Umiña Achala Website

The Umiña Achala website serves as the central hub for showcasing tokenized stones and minerals, enabling buyers, artisans, and regulators to interact with the ecosystem in a transparent and compliant manner.

### Visibility & Promotion
- **Marketplace Display**: Stones and minerals are listed with detailed provenance, artisan profiles, and compliance records.  
- **Search & Filtering**: Buyers can filter by type (emerald, pyrite, silver, etc.), HS code, artisan, or legal status.  
- **Promotion Channels**: Integrated SEO, social media campaigns, and third‑party marketplace visibility ensure global reach.  
- **QR Code Integration**: Each listing generates a QR code that customs agents, buyers, or auditors can scan to instantly retrieve compliance metadata and Hedera Consensus Service (HCS) notarization records.  

### Wallet & Transaction Layer
- **Wallet Connection**: Buyers connect digital wallets to purchase Umiña NFTs directly.  
- **Supported Assets**: Transactions can be made using Umiña Achala Coin (UMA), USDC, or other supported cryptocurrencies.  
- **Fractional Ownership**: Stones can be fractionally owned, with shares represented as fungible tokens paired with liquidity pools.  

### Compliance & Provenance
- **HIP‑412 Metadata Display**: Each NFT’s immutable metadata is accessible, including HS codes, legal status, and compliance fields.  
- **Regulatory Integration**: Automated checks against MINEM (REINFO), SUNAT (RUC), VUCE, and Ley 32537 are displayed for transparency.  
- **Audit Trail**: HCS logs for minting, transformation, sale, export, or exceptional events (gift, stolen, lost) are accessible via the website interface.  
- **Regulator Access**: Regulators can log in to view compliance proofs and audit trails directly, ensuring oversight and enforcement capabilities are built into the platform.  

### Artisan & Vendor Onboarding
- **Profile Pages**: Artisans and vendors have verified profiles linked to RNA IDs and decentralized identifiers (DIDs).  
- **Portfolio Showcase**: Artisans can display their work, transformation events, and compliance history.  
- **Marketplace Access**: Only verified artisans and compliant vendors can list stones, ensuring integrity of the ecosystem.  

### Purpose
- The Umiña Achala website is designed to **increase visibility, promote trust, and streamline compliance**.  
- It bridges the gap between local artisans and global buyers, ensuring that every stone’s journey from mine to marketplace is transparent, auditable, and secure.


## Conclusion
This minting workflow ensures compliance, provenance, and auditability for every Umiña NFT. By integrating Peruvian regulatory databases, Hedera services, and HIP‑412 metadata, Umiña Achala achieves high‑integrity Real World Asset tokenization.

[Back to Top](#umiña-nft-minting-guide)
