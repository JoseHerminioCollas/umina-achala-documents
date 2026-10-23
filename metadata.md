# 📊 Umiña Metadata Standard (HIP-412)

This document defines the JSON metadata schema for Umiña NFTs, ensuring compliance with the [Hedera HIP-412 standard](https://hips.hedera.com) and Peruvian trade regulations (Ley Nº 32537, 2026).

---

## Metadata Field Descriptions

| Key | Type | Description |
| :--- | :--- | :--- |
| `name` | String | Public identifier (e.g., "Umiña Stone #003 - Chrysocolla"). |
| `creator` | String | Project or mining/artisanal entity. |
| `description` | String | Narrative of provenance, geology, and compliance. |
| `image` | URI | Immutable IPFS link to the primary visual asset. |
| `type` | String | MIME type of the media (e.g., `image/jpg`). |
| `format` | String | Metadata version, fixed at `HIP412@2.0.0`. |
| `metadata_version` | String | Internal schema version (e.g., `1.0.0`). |
| `properties` | Object | Compliance and regulatory anchors. |
| `files` | Array | Extended assets (3D scans, permits, receipts). |
| `attributes` | Array | Traits and compliance status displayed in wallets. |

---

## 🏛️ Properties: Regulatory Anchors

The `properties` object contains compliance‑critical fields:

- **`stone_id`** → Unique identifier (`UMINA-2026-CH-03`).  
- **`hs_code`** → Harmonized System Code (`7103.99.00.00`).  
- **`jurisdiction`** → Country of origin (`Peru`).  
- **`legal_uri`** → Project legal reference (`https://umina-achala.pe/`).  
- **`vendor_ruc`** → SUNAT tax ID (`20999887766`).  
- **`artisan_rna`** → MINCETUR RNA number (`RNA-112233`).  
- **`mining_concession`** → Concession ID (`Arequipa-Chrys-03`).  
- **`reinfo_id`** → Miner’s REINFO ID (`PERU-MIN-445566`).  
- **`reinfo_status`** → Current REINFO status (`Vigente`).  
- **`cod_reference`** → VUCE Certificate of Origin reference (`VUCE-ORIGIN-2026-003`).  
- **`standard_compliance`** → Governing framework (`DS Nº 014-92-EM`).  
- **`compliance_proof_hcs`** → Hedera HCS Topic ID (`0.0.987654`).  
- **`audit_log_uri`** → HCS audit trail URI (`hcs://0.0.987654`).  
- **`last_regulatory_check`** → ISO 8601 timestamp (`2026-05-10T12:00:00Z`).  
- **`vuce_xml_hash`** → SHA‑256 hash of VUCE XML signature.  
- **`sunafil_payroll_verified`** → Boolean flag (`true`).  
- **`census_status`** → Census verification (`Finalized`).  
- **`sinceramiento_timestamp`** → Georeferenced filing timestamp (`2026-05-20T09:45:00Z`).  
- **`export_status`** → Nested object:  
  - `is_exported` → Boolean (`false`)  
  - `export_timestamp` → Null until export  
  - `export_permit_id` → VUCE permit (`VUCE-2026-EXP-100`)  
  - `current_location` → Current status (`Peru - Pending Export`)  
- **`store_receipt_hash`** → SHA‑256 hash of retail receipt.  
- **`sinceramiento_status`** → Filing status (`Verified (Georeferenced)`).  
- **`inei_census_id`** → Census ID (`INEI-2026-MAPE-8823`).  
- **`hcs_compliance_topic`** → HCS compliance topic reference (`0.0.987654`).  

---

## 🔗 Attributes: On‑Chain Provenance

The `attributes` array uses `trait_type` and `value` pairs:

- **`Stone Type`** → Chrysocolla  
- **`HS Code`** → 7103.99.00.00  
- **`Weight`** → 8.7 carats  
- **`Grading`** → Collector Grade  
- **`Date Mined`** → 2026‑02‑14  
- **`Stone Cut`** → Cabochon  
- **`Artisan`** → María López  
- **`Artisan Technique`** → Wire Wrapping  
- **`Track`** → Track C: Mine‑to‑Artisan  
- **`Mining Concession`** → Arequipa‑Chrys‑03  
- **`Legal Status`** → Certified Origin & Artisan Transformation  
- **`Verification Level`** → Full 2026 Compliance (Mine‑to‑Artisan)  
- **`Current Location`** → Peru  

---

## 📌 Example Metadata (Chrysocolla)

```json
{
  "name": "Umiña Stone #003 - Chrysocolla",
  "creator": "Umiña Achala",
  "description": "Certified Mine-to-Market Chrysocolla. Extracted from Arequipa Region with verified ethical provenance and full regulatory compliance.",
  "image": "ipfs://QmChrysocollaThumbnailHash",
  "type": "image/jpg",
  "format": "HIP412@2.0.0",
  "metadata_version": "1.0.0",
  "properties": { ... },
  "files": [ ... ],
  "attributes": [ ... ]
}
