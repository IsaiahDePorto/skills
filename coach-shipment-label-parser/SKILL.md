---
name: coach-shipment-label-parser
description: Visual parser and extraction engine for Coach carton contents labels. Extracts, normalizes, and reconciles shipment data across Standard DC Carton manifests, Single-SKU Factory Master Packs, and Atypical OLPN labels. Integrates with Coach Scene7 image CDN, POS scanner UPC indexing, and retail/outlet channel classification. Use when transcribing, verifying, or analyzing Coach shipment carton label images.
license: MIT
metadata:
  version: "1.0.0"
  author: "Tapestry Store Operations & AI Engineering"
---
```

# Coach Shipment Carton Label Parser & Intelligence

This skill provides an end-to-end operational and technical specification for transcribing, normalizing, and analyzing inventory labels found on inbound shipping cartons at Coach stores. It formalizes parsing logic across three distinct label formats, handles multi-page carton reconciliation, and bridges extracted identifiers with Tapestry's Adobe Scene7 image pipeline and POS/Scan APIs.

---

## 1. Label Architecture & Format Taxonomy

Coach inbound shipments arrive under three distinct physical label formats:

```
+---------------------------------------------------------------------------------------------------+
|                                  COACH CARTON LABEL SPECIFICATIONS                                |
+-------------------------------+-----------------------------------+-------------------------------+
| Feature                       | 1. Standard Format                | 2. Single-SKU Master Pack     | 3. Atypical Format (OLPN)     |
+-------------------------------+-----------------------------------+-------------------------------+
| Primary Header Tag            | "Carton Contents"                 | (No title - PO/STO header)    | "OLPN Contents"               |
| Primary Purpose               | DC Outbound Manifest              | Factory Vendor Uniform Pack   | Fulfillment / Store Transfer  |
| SKU Capacity                  | Multi-SKU (1 to 100+ items)       | Strictly 1 SKU per box        | Multi-SKU (1 to 100+ items)   |
| Primary Identifier            | CARTON# (SSCC-18)                 | 12-Digit UPC Barcode          | DLPN# / Pkt#                  |
| Style & Color Encoding        | Two separate columns              | Single string: [STYLE] [COL]  | Single string: [STYLE] [COL]  |
| Quantity Representation       | Integer (e.g., 4)                 | Integer (e.g., QTY : 4)       | Float (e.g., 4.0)             |
| Footer Verification           | Total Pieces: [Count]             | Single QTY check              | Total Qty: [##.0] //CRTNDNTS  |
+-------------------------------+-----------------------------------+-------------------------------+
```

---

## 2. Ingestion & Extraction Protocols by Format

### Format 1: Standard Format ("Carton Contents")
The primary multi-SKU manifest generated at Tapestry Distribution Centers (e.g., Jacksonville DC).

#### Header Metadata Block
* **`Date & Time`:** Timestamp (`MM/DD/YY HH:MM:SS`, e.g., `9/22/26 18:49:57`).
* **`Page`:** Top-right `Page: X` indicator. If page count > 1, requires pagination tracking.
* **`Wave`:** 11-digit DC picking wave identifier (`YYYYMMDD###`, e.g., `20260922042`).
* **`Store` & `Cust Name`:** Destination store number (e.g., `4501`) and store name (`COACH 4501 KITTERY`).
* **`CARTON#`:** 20-digit SSCC-18 tracking string (e.g., `00000923033544394108`). Matches the sidebar barcode `(00) 0 0092303...`.
* **`Cust #` & `Cust PO`:** Store internal customer code (`B610`) and purchase order (`6311101000B610`).
* **`Pkt#` & `Ord`:** 10-digit packing slip ID (`8119815350`) and order number (`4522103634`).
* **`Ctn` (Box Spec):** Physical box type code `CTN/[##][Letter]` (e.g., `CTN/69R`, `CTN/63R`, `CTN/56R`, `CTN/50R`, `CTN/38R`). Defines carton cubic volume.
* **`Chute`:** Sortation slide address (e.g., `403-01`, `309-03`, `106-26`).
* **`Carrier`:** Shipping method code (e.g., `FDER` = FedEx Economy/Freight).

#### Line Items Table Columns
| Field | Name | Description & Extraction Rules |
| :--- | :--- | :--- |
| 1 | `Ln` | Sequential integer line number across the wave/carton (e.g., `16`, `17`, `18`). |
| 2 | `Location` | DC bin slotting coordinate. Blank for standard goods; populated (e.g., `FX2452C2`, `FX2535B4`) for footwear or specialized picking rows. |
| 3 | `Style` | Alphanumeric Coach style number (e.g., `CFP22`, `CW786`, `CW637`) or legacy 5-digit numeric (e.g., `67690`). |
| 4 | `Color` | Hardware finish + color code (e.g., `SV/PVH`, `IMXAQ`, `B4MPL`, `QB/BK`, `LHSLV`). Clean forward slashes (`/`) when bridging to Scene7. |
| 5 | `Size` | Blank for handbags and small leather goods. Populated for apparel (`S`, `M`, `L`, `XL`), footwear (`11 D`, `9 B`), or one-size (`ONE`). |
| 6 | `Description`| Uppercase/mixed product name, hard-truncated at ~21 characters (e.g., `Colby Small Flap Cros`, `Suede Brooklyn Shoul`). |
| 7 | `Qty` | Integer item quantity. |

#### Routing & Lifecycle Stamp
* Large bold stamp in the right column (e.g., `09.23 R`, `09.16 N`):
  * **Date Component:** Target delivery or sort date (`MM.DD`).
  * **Suffix `R`:** **Replenishment / Regular** stock (replenishing existing core store inventory).
  * **Suffix `N`:** **New / Floor Set / Launch** stock (new release merchandise or upcoming campaign floor sets).

---

### Format 2: Single-SKU Master Pack (Factory Vendor Label)
Affixed directly to overseas factory cartons containing exactly one product variant packed in uniform multiples.

#### Data Block
* **`PO`:** 10-digit manufacturing purchase order (e.g., `4517990698`).
* **`STO`:** 10-digit Stock Transfer Order (e.g., `4517990770`).
* **`SKU`:** Single combined string formatted as `[Style] [Color]` (e.g., `CEW04 IMBLK`, `CCD71 IMBLK`, `CR111 IMPOP`).
* **`QTY`:** Integer case quantity (typically `QTY : 4`).

#### Barcode & Identifier Architecture
* **Vertical 12-Digit UPC Barcode:** Scannable UPC printed vertically adjacent to the SKU block (e.g., `198685184716`, `196395997640`, `198685323977`).
  * **Direct POS Integration:** This 12-digit UPC is the exact key required by Tapestry's In-Store POS API (`https://app.scan.coach.com/api/iteminfo/catalogs/coach-us/stores/{storeId}/items/{upc}`). Enables zero-latency resolution of live pricing, bullet descriptions, and inventory without style-lookup steps.
* **Horizontal Master Tracking Barcode:** 10-digit package ID (e.g., `9004252575`) with GS1-128 / SSCC-18 representation.
* **Corner Production Batch Index:** Small corner numeric index (e.g., `327`, `2526`, `225`).

---

### Format 3: Atypical Format ("OLPN Contents")
Order License Plate Number manifest used for cross-docking, specialized distribution paths, and specific fulfillment orders.

#### Header Metadata
* **`Header Tag`:** Clearly titled `OLPN Contents`.
* **`CTR` Zone:** Sorting center/zone identifier (e.g., `CTR 56F`).
* **`DLPN#`:** 20-digit Delivery License Plate Number (e.g., `00000195031068258239`).
* **`Cust PO`:** 10-digit store customer PO (e.g., `4522030838`).
* **`Pkt#`:** Distinct format beginning with `D` and padded with zeros (e.g., `D0073120230000000021`).

#### Table Schema & Distinctions
* Columns: `LN` | `SKU` | `Item Description` | `Qty`
* **Combined SKU Column:** Style and color are merged directly (e.g., `CW778 IMNAV`, `CFX34 IMZBA`, `6303 IMBLK`).
* **Unabbreviated Item Descriptions:** Significantly longer strings that preserve detailed product names (e.g., `Medium Corner Zip Wallet in Suede`, `Nolita 19 in Crocodile-Embossed Leather`).
* **Decimal Quantities:** Quantities are explicitly rendered as floating-point numbers (`1.0`, `2.0`, `12.0`).
* **Integrity Footer:** Ends with `Total Qty: [##.0]` and the system footer delimiter `//CRTNDNTS`.

---

## 3. Cross-System Synthesis & Retail Channel Classification

By cross-referencing extracted style and color strings with `coach-catalog-intelligence`:

### Hardware Plating & Channel Verification Matrix
1. **`IM` Prefix (Imitation Gold):**
   * Plating: High-gloss yellow brass.
   * Channel: **100% Factory Outlet (Made For Factory - MFF)**.
2. **`B4` / `BP` Prefix (1941 Antique Brass):**
   * Plating: Burnished vintage antique brass.
   * Channel: **Retail Boutique (Full-Price)**.
   * **Boutique Delete Flag:** When `B4` styles (e.g., `CW637 B425D` Suede Brooklyn Shoulder, `CR990 B4/BK` Refined Calf Leather, `CR398 B4MPL` 3-in-1 Wallet) arrive at an outlet store (Store 4501), flag immediately as **Boutique Delete / Coach Reserve (Target for markdown pricing & clearance auditing)**.
3. **`LH` / `GLD` Prefix (Light Gold / High-Shine Gold):**
   * Plating: Soft champagne or luxury yellow gold.
   * Channel: **Predominantly Retail Boutique (~90%)**. Flag items like `I695 LHSLV` or `CT181 GLD ONE`.
4. **`SV` (Polished Silver) & `QB` (Gunmetal):**
   * Channel: **Dual-Channel / Ambiguous**. Used across both retail boutique lines and factory outlet production.

### Dynamic Scene7 Hero Asset URL Construction
Every parsed line item can be immediately transformed into its master hero image URL without making API calls:
```
Template: https://images.coach.com/is/image/Coach/{clean_style}_{clean_color}_a0

Examples:
- Standard: Style CFP22, Color SV/PVH  -> https://images.coach.com/is/image/Coach/cfp22_svpvh_a0
- Single-SKU: SKU CEW04 IMBLK         -> https://images.coach.com/is/image/Coach/cew04_imblk_a0
- OLPN: SKU CFX34 IMZBA               -> https://images.coach.com/is/image/Coach/cfx34_imzba_a0
```

---

## 4. Multi-Page & Multi-Image Reconciliation Rules

Carton manifests may span multiple physical labels or arrive across multiple photographic images:

1. **State Preservation:**
   * When `Page: 1` displays `Continued...` at the bottom, maintain an open carton object in memory indexed by `CARTON#` or `Pkt#`.
   * Subsequent images matching the same `CARTON#` or `Ord#` must be merged into the existing record rather than treated as a separate carton.
2. **Line Continuity Check:**
   * Verify that the initial `Ln` on Page 2 immediately succeeds the final `Ln` on Page 1 (e.g., Page 1 ends at Ln 27 $\rightarrow$ Page 2 begins at Ln 28).
3. **Piece Count Audit:**
   * Calculate: `Sum(line_items.qty)`.
   * Compare against `Total Pieces: [Count]` (Standard) or `Total Qty: [##.0]` (OLPN).
   * If `Sum != Total`, explicitly flag an **Inventory Discrepancy / Incomplete Page Warning**.

---

## 5. Normalized JSON Output Schema

When the agent extracts data from any Coach contents label, normalize the output into this canonical structure:

```json
{
  "carton_metadata": {
    "format": "standard | single_sku | atypical_olpn",
    "carton_number": "00000923033544394108",
    "store_number": "4501",
    "store_name": "COACH 4501 KITTERY",
    "order_number": "4522103634",
    "packet_number": "8119815350",
    "wave_id": "20260922042",
    "chute": "403-01",
    "carton_type": "CTN/69R",
    "carrier": "FDER",
    "routing_stamp": {
      "target_date": "09.23",
      "type_code": "R",
      "type_description": "Replenishment"
    },
    "timestamp": "2026-09-22T18:49:57",
    "pages_present": [1, 2],
    "is_complete": true
  },
  "line_items": [
    {
      "line_number": 29,
      "style": "CW637",
      "color": "B425D",
      "size": null,
      "description": "Suede Brooklyn Shoul",
      "quantity": 3,
      "location": null,
      "upc_12digit": null,
      "channel_classification": "Retail Boutique (Boutique Delete / Coach Reserve)",
      "hardware_finish": "1941 Antique Brass (B4)",
      "scene7_hero_url": "https://images.coach.com/is/image/Coach/cw637_b425d_a0"
    }
  ],
  "verification_audit": {
    "calculated_quantity": 3,
    "manifest_total_quantity": 3,
    "discrepancy_detected": false
  }
}
```

---

### Highlights of Operational Features Built In:

1. **Seamless Multi-Page Stitching**: Handles boxes where Page 1 and Page 2 are photographed separately or side-by-side, checking `Continued...`, sequential line numbering, and verifying `Sum(Qty) == Total Pieces`.
2. **Direct POS Acceleration**: On Single-SKU master packs, the agent extracts the raw 12-digit vertical UPC (`198685184716`, etc.) so it can immediately call `scan.coach.com` for pricing, clearance checks, and detailed specifications without intermediate style matching.
3. **Boutique Delete Flagging**: Automatically identifies retail boutique hardware finishes (`B4`, `BP`, `LH`, `GLD`) arriving at Store 4501 (Kittery Outlet) and classifies them as Coach Reserve / Boutique Deletes for special processing.
4. **Lifecycle Stamp Tracking**: Accurately maps the large bold `R` (Replenishment) vs. `N` (New / Floor Set / Launch) status alongside the date code.
