---
name: coach-style-image-retriever
description: Decodes Coach style numbers, hardware finishes, and color codes to construct high-accuracy product image URLs hosted on Adobe Scene7 (images.coach.com). Automatically determines standard camera angle availability based on product type, defaulting to the master hero image (_a0) and selecting alternate views (_a1 through _a5) dynamically to prevent 404 errors.
metadata:
  version: "1.0.0"
  author: "Isaiah & Gemini"
---

# Coach Style Decoder and Image URL Constructor

This skill provides a systematic framework for decoding Coach retail (boutique) and outlet style numbers, parsing compound hardware-color codes, and accurately reconstructing their corresponding Adobe Dynamic Media (Scene7) image URLs.

## 1. Core Objectives
- Parse and sanitize Coach style numbers (MFF, Boutique, and Boutique Deletes).
- Decode complex hardware-material color codes (e.g., converting `IM/BLK` to `IMBLK` and mapping its components).
- Construct valid image retrieval URLs using the exact naming sequence template.
- Implement heuristic filtering to select correct camera angle suffixes (`_a0` to `_a5`) based on the product category to avoid server-side `404 Not Found` errors.

---

## 2. Structural Parsing Guidelines

### A. Style Number Classification
Verify the category of the style number to understand its origin:
1. **Historical Outlet / Factory (MFF):** Typically begins with an `F` prefix followed by 5 digits (e.g., `F58292`).
2. **Modern Style Numbers (2020–Present):** Consists of a single-letter prefix (e.g., `C`, `CA`, `CB`, `CF`, `CJ`, `CP`), followed by a 3- or 4-digit sequence (e.g., `C1555`, `CJ123`). Both outlet and retail boutiques share this system.
3. **Boutique/Full-Price Deletes (Coach Reserve):** Keeps its original boutique alphanumeric code but is routed to outlet inventory.

### B. Hardware & Color Suffix Decoding
Decode the compound alphanumeric suffix (usually 4 to 5 characters) appended to the style number to determine the aesthetic trim. The pattern follows: `[Hardware Finish Prefix (2 chars)][Color Suffix]`.

#### Hardware Finish Prefix Reference:
- **`IM`** = Imitation Gold (Highly polished, typical on MFF outlet items)
- **`SV`** = Silver (Polished nickel/chrome)
- **`LH`** = Light Gold (Softer champagne gold, common in boutique and premium outlet items)
- **`QB`** = Gunmetal / Quality Brass (Smoky dark metal or burnished antique brass)
- **`BP` / `B4`** = Classic Brass / Heritage 1941 Brass
- **`GD`** = True Yellow Gold
- **`V5`** = Vintage Brass / Pewter
- **`DK`** = Dark Pewter / Gunmetal

#### Material Color/Pattern Suffix Reference:
- **`BLK` / `BK`** = Black
- **`DTV`** = Granite (Dark Grey)
- **`HA` / `CHK`** = Chalk / Off-White
- **`SADDL` / `SAD`** = Saddle Brown
- **`MID`** = Midnight Navy
- **`DQC`** = Light Khaki/Chalk (Signature coated canvas print)
- **Seasonal Codes** = Custom 3-character codes (e.g., `F8Q`, `OU9`) denoting short-run seasonal colors.

---

## 3. Image URL Construction Protocol

### A. The Master URL Template
Construct the image asset URL using the following format:
```
https://images.coach.com/is/image/Coach/[style-number]_[color-code-without-slash]_[angle-suffix]
```
*Note: Ensure any slashes (`/`) present in the color code are stripped (e.g., `IM/BLK` -> `IMBLK`).*

### B. Camera Angle Taxonomy
The suffix at the end of the URL maps to the designated camera angle of the product asset:
*   **`_a0` (Hero Shot - DEFAULT):** Front-facing, clean-cropped, primary retail image. **Always default to `_a0` when fetching a single image for an item.**
*   **`_a1` (Profile View):** Three-quarter angle showing depth and side gussets.
*   **`_a2` (Rear View):** Showcases back pockets, slips, or reverse styling.
*   **`_a3` (Interior View):** Open view displaying pockets, lining fabric, and creed patch.
*   **`_a4` (Bottom / Detail View):** Macro shot of hardware accents, leather textures, or bottom protective feet.
*   **`_a5` (On-Figure / Model View):** Showcases the scale of the bag styled on a human body silhouette.

---

## 4. Operational Logic & Error Prevention

To prevent broken image links (`404` errors), apply this heuristic filter before constructing alternate angle URLs:

| Product Category | Allowed Angle Suffixes | Default Suffix | Rationale |
| :--- | :--- | :--- | :--- |
| **Handbags / Totes / Backpacks** | `_a0`, `_a1`, `_a2`, `_a3`, `_a4`, `_a5` | `_a0` | Handbags require comprehensive scaling, interior, and bottom shots. |
| **Small Leather Goods (SLGs)** *(Wallets, Wristlets, Card Cases)* | `_a0`, `_a1`, `_a2`, `_a3` | `_a0` | Wallets do not have complex bottoms (`_a4`) or on-figure model shoots (`_a5`). |
| **Keychains / Charms / Lanyards** | `_a0`, `_a1` | `_a0` | Flat detail accessories only have front and occasionally a back/tilt view. |
| **Shoes / Boots** | `_a0`, `_a1`, `_a2`, `_a3` | `_a0` | Shoes showcase profile, back, and top/inside angles but omit body scale (`_a5`) or bottom soles unless highly textured. |
| **Apparel** | `_a0`, `_a1`, `_a2`, `_a5` | `_a0` | Focuses on front, back, detail close-ups, and model fit. |

### Fallback Routine
If a constructed URL fails or if multiple angles are desired but unverified, fall back strictly to the **`_a0`** hero shot, as it is guaranteed to be generated for every active product SKU.
