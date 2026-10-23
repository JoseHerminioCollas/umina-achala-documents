# Umiña Specimens Catalog

This file consolidates gemstones, minerals, ornamental stones, and their HS Codes for the Umiña Achala project. It serves as both a compliance reference and a cultural catalog.

## ℹ️ What are HS Codes?

HS Codes (Harmonized System Codes) are internationally standardized numerical identifiers used in global trade. Each code classifies goods for customs purposes, ensuring that shipments are properly documented, taxed, and regulated. The system is maintained by the World Customs Organization (WCO) and is recognized by nearly every country worldwide.

- **Structure**: HS Codes are typically 6–10 digits long. The first digits define the broad category (e.g., ores, stones, jewelry), while later digits specify the exact product type.
- **Purpose**: They allow customs authorities to determine tariffs, track trade statistics, and enforce import/export restrictions.
- **Compliance**: Exporters and importers must declare the correct HS Code for each item to avoid fines, delays, or seizure of goods.

## 📑 HS Codes in the Umiña Achala Project

In the Umiña Achala project, HS Codes are used to:

- **Authenticate provenance**: Each gemstone, mineral, or ornamental stone is mapped to its official HS Code, ensuring it is recognized under Peruvian and international trade law.
- **Automate export documentation**: By embedding HS Codes into metadata, Umiña tokens can generate customs-ready paperwork (e.g., VUCE forms, export declarations).
- **Enable compliance checks**: HS Codes act as a bridge between cultural cataloging and legal compliance, making sure stones are not misclassified or exported illegally.
- **Support tokenization**: When an Umiña specimen is minted as a digital token, its HS Code is paired with provenance identifiers (Concession ID, REINFO ID, Vendor RUC). This creates a verifiable link between the physical stone and its digital representation.

---

## 🧩 Example

- **Amethyst** → HS Code `7103.99.00.00` (“Worked Amethyst”)  
- **Gold** → HS Code `7108.12.00.00` (“Gold, Non-monetary, Unwrought”)  
- **Copper** → HS Code `2603.00.00.00` (“Copper Ores and Concentrates”)  

These codes ensure that when an Umiña token is traded or exported
## Gemstones & Minerals of Peru

### 💍 Precious Stones

|                |                |                |
|----------------|----------------|----------------|
| Diamond        | Ruby           | Sapphire       |
| Emerald        |                |                |

### 💎 Gemstones (Semi-Precious)

|                |                |                |
|----------------|----------------|----------------|
| Amethyst       | Amazonite      | Agate          |
| Aragonite      | Angelite       | Aquamarine     |
| Black Jade     | Blue Chalcedony| Beryl          |
| Chrysoberyl    | Chrysocolla    | Dumortierite   |
| Opal           | Garnet         | Rhodonite      |
| Leopard Jasper | Rosophia       | Turquoise      |
| Obsidian       | Serpentine     | Sodalite       |
| Topaz          | Tourmaline     | Quartz         |

### 🪨 Ornamental Stones

|                |                |                |
|----------------|----------------|----------------|
| Obsidian       | Rhodonite      | Serpentine     |
| Sodalite       | Leopard Jasper | Rosophia       |

### ⚒️ Minerals

|                |                |                |
|----------------|----------------|----------------|
| Copper         | Gold           | Silver         |
| Lead           | Zinc           | Iron Ore       |
| Pyrite         |                |                |


## 📦 HS Codes for the Umiña Achala Project

### 💍 Precious Stones

| Subcategory                | HS Code        | Description                                                                 |
|----------------------------|----------------|-----------------------------------------------------------------------------|
| Diamonds (Unworked)        | 7102.10.00.00  | Natural diamonds, unworked or simply sawn                                   |
| Diamonds (Worked, Cut)     | 7102.21.00.00  | Natural diamonds, worked, cut but not mounted                               |
| Diamonds (Worked, Mounted) | 7102.31.00.00  | Natural diamonds, worked, mounted or set                                    |
| Diamonds (Synthetic)       | 7104.90.00.00  | Synthetic or reconstructed diamonds                                         |
| Other Precious Stones (Raw)| 7103.10.00.00  | Other precious stones, unworked or simply sawn or roughly shaped            |
| Other Precious Stones (Cut)| 7103.21.00.00  | Other precious stones, worked, cut but not mounted                          |
| Other Precious Stones (Mounted)| 7103.91.00.00 | Other precious stones, worked, mounted or set                               |

### 💎 Semi-Precious & Ornamental Stones

| Subcategory                | HS Code        | Description                                                                 |
|----------------------------|----------------|-----------------------------------------------------------------------------|
| Semi-Precious Stones       | 7103.99.00.00  | Semi-precious stones, worked or unworked                                    |
| Ornamental Stones          | 7103.99.00.00  | Ornamental stones (e.g., obsidian, serpentine, sodalite), worked or unworked|

### 📿 Jewelry

| Subcategory                | HS Code        | Description                                                                 |
|----------------------------|----------------|-----------------------------------------------------------------------------|
| Jewelry (Precious Metal)   | 7113.19.00.00  | Articles of jewelry and parts thereof, of precious metal (e.g., gold, silver) |
| Jewelry (Pearls/Stones)    | 7116.20.00.00  | Articles of natural/cultured pearls, precious or semi-precious stones       |

### ⚒️ Minerals (Peru)

| Subcategory                | HS Code        | Description                          |
|----------------------------|----------------|--------------------------------------|
| Copper (Base Metal Ore)    | 2603.00.00.00  | Copper ores and concentrates         |
| Lead (Base Metal Ore)      | 2607.00.00.00  | Lead ores and concentrates           |
| Zinc (Base Metal Ore)      | 2608.00.00.00  | Zinc ores and concentrates           |
| Iron Ore (Base Metal Ore)  | 2601.11.00.00  | Non-agglomerated iron ores           |
| Pyrite (Mineral)           | 2502.00.00.00  | Unroasted Iron Pyrites               |
| Gold (Precious Metal)      | 7108.12.00.00  | Gold, non-monetary, unwrought        |
| Silver (Precious Metal)    | 7106.91.00.00  | Silver, unwrought                    |

## JSON Specimen Mapping

```json
{
  "project": "Umiña Achala",
  "version": "1.0.6",
  "base_chapter": "71",
  "specimens": {
    "gemstones": {
      "amethyst": { "hs_code": "7103.99.00.00", "description": "Worked Amethyst", "origin": "semi-precious" },
      "amazonite": { "hs_code": "7103.99.00.00", "description": "Worked Amazonite", "origin": "semi-precious" },
      "agate": { "hs_code": "7103.99.00.00", "description": "Worked Agate", "origin": "semi-precious" },
      "aragonite": { "hs_code": "7103.99.00.00", "description": "Worked Aragonite", "origin": "semi-precious" },
      "angelite": { "hs_code": "7103.99.00.00", "description": "Worked Angelite", "origin": "semi-precious" },
      "aquamarine": { "hs_code": "7103.99.00.00", "description": "Worked Aquamarine", "origin": "semi-precious" },
      "black_jade": { "hs_code": "7103.99.00.00", "description": "Worked Black Jade", "origin": "semi-precious" },
      "blue_chalcedony": { "hs_code": "7103.99.00.00", "description": "Worked Blue Chalcedony", "origin": "semi-precious" },
      "beryl": { "hs_code": "7103.99.00.00", "description": "Worked Beryl", "origin": "semi-precious" },
      "chrysoberyl": { "hs_code": "7103.99.00.00", "description": "Worked Chrysoberyl", "origin": "semi-precious" },
      "chrysocolla": { "hs_code": "7103.99.00.00", "description": "Worked Chrysocolla", "origin": "semi-precious" },
      "dumortierite": { "hs_code": "7103.99.00.00", "description": "Worked Dumortierite", "origin": "semi-precious" },
      "ruby": { "hs_code": "7103.91.00.00", "description": "Worked Rubies", "origin": "precious" },
      "sapphire": { "hs_code": "7103.91.00.00", "description": "Worked Sapphires", "origin": "precious" },
      "opal": { "hs_code": "7103.99.00.00", "description": "Worked Opal", "origin": "semi-precious" },
      "garnet": { "hs_code": "7103.99.00.00", "description": "Worked Garnet", "origin": "semi-precious" },
      "rhodonite": { "hs_code": "7103.99.00.00", "description": "Worked Rhodonite", "origin": "semi-precious" },
      "leopard_jasper": { "hs_code": "7103.99.00.00", "description": "Worked Leopard Jasper", "origin": "semi-precious" },
      "rosophia": { "hs_code": "7103.99.00.00", "description": "Worked Rosophia", "origin": "semi-precious" },
      "turquoise": { "hs_code": "7103.99.00.00", "description": "Worked Turquoise", "origin": "semi-precious" },
      "obsidian": { "hs_code": "7103.99.00.00", "description": "Worked Obsidian", "origin": "semi-precious" },
      "serpentine": { "hs_code": "7103.99.00.00", "description": "Worked Serpentine", "origin": "semi-precious" },
      "sodalite": { "hs_code": "7103.99.00.00", "description": "Worked Sodalite", "origin": "semi-precious" },
      "topaz": { "hs_code": "7103.99.00.00", "description": "Worked Topaz", "origin": "semi-precious" },
      "tourmaline": { "hs_code": "7103.99.00.00", "description": "Worked Tourmaline", "origin": "semi-precious" },
      "quartz": { "hs_code": "7103.99.00.00", "description": "Worked Quartz", "origin": "semi-precious" },
      "emerald": { "hs_code": "7103.91.00.00", "description": "Worked Emeralds", "origin": "precious" },
      "diamond_natural": { "hs_code": "7102.10.00.00", "description": "Natural diamonds, unworked or simply sawn", "origin": "precious", "note": "Peru has no natural deposits; included for completeness only" },
      "diamond_worked_cut": { "hs_code": "7102.21.00.00", "description": "Natural diamonds, worked, cut but not mounted", "origin": "precious" },
      "diamond_worked_mounted": { "hs_code": "7102.31.00.00", "description": "Natural diamonds, worked, mounted or set", "origin": "precious" },
      "diamond_synthetic": { "hs_code": "7104.90.00.00", "description": "Synthetic or reconstructed diamond", "origin": "synthetic" }
    },
    "minerals": { 
      "gold": { "hs_code": "7108.12.00.00", "description": "Gold, Non-monetary, Unwrought", "origin": "precious" }, 
      "silver": { "hs_code": "7106.91.00.00", "description": "Silver, Unwrought", "origin": "precious" }, 
      "copper": { "hs_code": "2603.00.00.00", "description": "Copper Ores and Concentrates", "origin": "base-metal" }, 
      "lead": { "hs_code": "2607.00.00.00", "description": "Lead Ores and Concentrates", "origin": "base-metal" }, 
      "zinc": { "hs_code": "2608.00.00.00", "description": "Zinc Ores and Concentrates", "origin": "base-metal" }, 
      "iron_ore": { "hs_code": "2601.11.00.00", "description": "Non-agglomerated Iron Ores", "origin": "base-metal" }, 
      "pyrite": { "hs_code": "2502.00.00.00", "description": "Unroasted Iron Pyrites", "origin": "industrial" } 
    },
    "ornamental_stones": {
      "obsidian": { "hs_code": "7103.99.00.00", "description": "Worked Obsidian", "origin": "ornamental" },
      "rhodonite": { "hs_code": "7103.99.00.00", "description": "Worked Rhodonite", "origin": "ornamental" },
      "serpentine": { "hs_code": "7103.99.00.00", "description": "Worked Serpentine", "origin": "ornamental" },
      "sodalite": { "hs_code": "7103.99.00.00", "description": "Worked Sodalite", "origin": "ornamental" },
      "leopard_jasper": { "hs_code": "7103.99.00.00", "description": "Worked Leopard Jasper", "origin": "ornamental" },
      "rosophia": { "hs_code": "7103.99.00.00", "description": "Worked Rosophia", "origin": "ornamental" }
    }
  }
}
```