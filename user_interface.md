# Umiña Achala Application UI

The Umiña Achala application blends compliance transparency with cultural storytelling. Its interface highlights provenance, artisan craftsmanship, and gemological details such as stone cuts.

## 🖥️ Main UI Components

### 1. Landing Page
- **Header:** Project name Umiña Achala with tagline “Automating compliance, illuminating provenance.”
- **Hero Section:** Featured stone image with provenance details (origin, artisan, cut, certification).
- **Quick Actions:**
  - Browse Stones
  - Mint New NFT
  - Compliance Dashboard
- **Additional Features:**
  - Language Toggle (EN/ES)
  - Quick Compliance Check widget showing latest verified stone IDs or permits.

### 2. Stone Marketplace
- **Grid/List View:** Stones displayed with thumbnail images.
- **Filters:**
  - Type (Amethyst, Quartz, etc.)
  - Region (Cusco, Ayacucho, etc.)
  - Mounted/Unmounted
  - Cut (Brilliant, Cabochon, Emerald, Princess, Custom)
  - Certification (Fair Trade, Sustainability)
  - Compliance Status (Verified, Pending, Not Applicable)
- **Stone Card Preview:**
  - Image of stone/mount
  - Origin region + mine
  - Artisan name (if mounted)
  - Cut type displayed prominently
  - Price in UMA + USDC equivalent
  - Verified Exporter badge (if linked to SUNAT/VUCE)
  - Optional price history chart

### 3. Stone Detail Page
- **High‑resolution image / 3D scan viewer**
- **Metadata Panel:**
  - Origin: mine, region, export permit ID
  - Stone details: type, weight, grading
  - Cut: Brilliant, Cabochon, Emerald, etc.
  - Mount details: artisan, technique, certification
  - Compliance: law reference, customs clearance status
- **Buy Button:** Pay in UMA (with USDC conversion shown).
- **Provenance Timeline:** Animated visual flow showing extraction → cutting → certification → mounting → tokenization.
- **Additional Features:**
  - QR code linking to Hedera audit trail
  - Downloadable compliance certificate (PDF)
  - Separate “Cultural Storytelling” tab for artisan narratives

### 4. Compliance Dashboard
- **Role‑Based Views:**
  - Regulators: export permits, audit logs, restricted item alerts
  - Buyers: simplified compliance summaries
- **Search Options:**
  - Stone ID
  - Permit number
  - Cut type
  - Hedera Transaction ID
- **Visuals:** Timeline or audit trail showing each compliance step, including cutting and mounting.

### 5. Wallet & Transactions
- **Wallet Integration:** Show UMA balance, USDC balance, PEN/USD equivalents, and recent swaps.
- **Transaction History:** Purchases, sales, swaps, compliance logs.
- **Compliance Receipts:** Downloadable proof of each purchase/export.
- **Liquidity Pool Access:** Simple interface to swap UMA ↔ USDC.
- **Future Expansion:** Staking/rewards tab for artisans/miners.

## 🎨 Design Style
- **Colors:** Earth tones (stone gray, silver, gold, deep amethyst purple).
- **Typography:** Clean sans‑serif for compliance data, elegant serif for cultural storytelling.
- **Icons:**
  - 🪨 Stone icon for provenance
  - ✂️ Cut icon for stone cut
  - 🪙 Coin icon for UMA
  - 📜 Scroll icon for compliance
- **Dual Themes:**
  - Compliance Mode: minimalist, regulator‑friendly
  - Storytelling Mode: rich visuals, artisan focus

## ⚡ Example User Flow
1. Buyer lands on homepage → clicks Browse Stones.
2. Filters for Amethyst, Cusco, Brilliant Cut, Mounted.
3. Selects a stone → sees metadata including cut + artisan details.
4. QR code + compliance certificate available for verification.
5. Clicks Buy with UMA → wallet pops up with USDC conversion.
6. Transaction completes → buyer sees compliance receipt + NFT in wallet.
