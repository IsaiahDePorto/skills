---
name: coach-catalog-intelligence
description: Comprehensive technical reference and developer specification for decoding Coach style numbers, constructing Adobe Scene7 Dynamic Media image URLs, probing camera views, reverse-engineering Salesforce Commerce Cloud (SFCC) product pages, querying Tapestry in-store POS/scan APIs, and automating session token minting for live store pricing and clearance (Last Chance) detection.
metadata:
  version: "2.1.0"
  author: "Isaiah & Gemini"
---

# Coach Catalog Intelligence & Asset Architecture

This skill provides an end-to-end technical reference and developer specification for decoding Coach product identifiers, constructing high-resolution media assets via Adobe Scene7, reverse-engineering Salesforce Commerce Cloud (SFCC) storefront URLs, integrating with Tapestry's Point of Sale (POS) and in-store scanner APIs, and programmatically detecting store-level clearance ("Last Chance!") statuses and promotional pricing.

---

## 1. System Overview & Architectural Topology

Coach and parent company Tapestry operate across three distinct technical ecosystems, each with unique identifier indexing, security boundaries, and data payloads:

```
+-----------------------------------------------------------------------------------+
|                               TAPESTRY ECOSYSTEM                                  |
+--------------------------+------------------------------+-------------------------+
|   Adobe Scene7 CDN       |   Salesforce Commerce Cloud  |  In-Store POS / Scan    |
|   (images.coach.com)     |   (coach.com / outlet)       |  (app.scan.coach.com)   |
+--------------------------+------------------------------+-------------------------+
| Index: Style + Color     | Index: Parent Style Number   | Index: 12-Digit UPC     |
| Assets: Media, Crops,    | Data: MSRP, Measurements,    | Data: POS Name, Dept,   |
|         XMP EXIF Lineage |       Materials, Strap Drops |       Promos, Clearance |
| Auth: None (Open CORS)   | Auth: Akamai Bot WAF (403)   | Auth: Anonymous Session |
+--------------------------+------------------------------+-------------------------+
```

---

## 2. Identifier Parsing & Classification

### A. Style Number Classification
Coach assigns style numbers based on production tier and era:
1. **Modern Unified Alphanumeric (2020–Present):** Consists of 1 to 3 leading letters followed by 2 to 4 digits (e.g., `C1555`, `CAQ25`, `CCX04`, `CU068`, `CY201`, `CEN85`, `CEF29`). This format is shared across retail boutiques, specialty collaborations, and modern factory-exclusive production.
2. **Historical Factory / Outlet (MFF):** Typically begins with an `F` prefix followed by 5 digits (e.g., `F58292`).
3. **Boutique Deletes (Coach Reserve):** Retain their original boutique style number (e.g., `CCX04`), but are reassigned to outlet inventory when transferred.

### B. Hardware Finish Codes (Prefix - 2 Characters)
Coach compound color codes prepend a 2-character hardware plating finish to the material color code:
* **`IM`** = Imitation Gold (High-gloss polished brass; standard on factory/outlet items)
* **`SV`** = Silver (Polished nickel/chrome plating)
* **`LH`** = Light Gold (Champagne gold; softer hue used in boutiques and elevated outlet pieces)
* **`B4` / `BP`** = 1941 Heritage Brass / Antique Brass (Burnished, vintage gold-tone common on 1941 and boutique Tabby/Rogue lines)
* **`QB`** = Gunmetal / Quality Brass (Smoky dark chrome or blackened antique brass)
* **`GD`** = True Yellow Gold (Vibrant high-karat gold finish)
* **`V5`** = Vintage Brass / Pewter
* **`DK`** = Dark Gunmetal / Dark Pewter

### C. Material & Pattern Color Codes (Suffix - 2 to 5 Characters)
Appended directly to the hardware prefix (either joined directly or separated by a slash `/` on retail tags):
* **`BK` / `BLK`** = Black
* **`MPL`** = Maple (Deep rich brown)
* **`OPI`** = Optic White
* **`UPZ`** = Natural Raffia / Straw
* **`HA` / `CHK`** = Chalk / Off-White
* **`SAD` / `SADDL`** = Saddle Brown
* **`MID`** = Midnight Navy
* **`DTV`** = Granite (Dark Slate Grey)
* **`DE`** = Denim
* **`BIC`** = Beige
* **`POP`** = Pop Orange / Coral
* **`ND7`** = Natural / Tan
* **`DQC`** = Light Khaki / Chalk (Signature Coated Canvas)
* **Seasonal Codes** = Alphanumeric combinations (e.g., `F8Q`, `OU9`, `Z1J`, `MS`) representing limited-run seasonal palettes.

### D. Formatting Rules for System Inputs
* **For Adobe Scene7 Image URLs:** Strip all slashes (`/`), remove whitespace, and convert the entire string to lowercase:
  * Style `CCX04`, Color `B4/MPL` $\rightarrow$ `ccx04_b4mpl`.
* **For Salesforce Commerce Cloud (coach.com):** Preserve case (uppercase), convert spaces to `+`, and URL-encode the forward slash as `%2F`:
  * Slashed: `CCX04` + `B4/BK` $\rightarrow$ `?frp=CCX04+B4%2FBK`.
  * Unslashed: `CU068` + `B4MPL` $\rightarrow$ `?frp=CU068+B4MPL`.

---

## 3. Adobe Scene7 Dynamic Media Engine (`images.coach.com` & `coach.scene7.com`)

### A. The Master Image URL Template
```
https://images.coach.com/is/image/Coach/[style-number]_[color-code]_[angle-suffix]
```
*(Both `images.coach.com` and `coach.scene7.com` point to the same Akamai-fronted Adobe Dynamic Media origin. `images.coach.com` serves permissive CORS headers: `Access-Control-Allow-Origin: *`.)*

### B. Complete Camera Angle Taxonomy

#### 1. Handbags, Totes, Backpacks & Hobos
* **`_a0` (Hero Shot - DEFAULT):** Clean-cropped front silhouette on neutral studio backdrop (`#F0F0F0`). Always default to this view.
* **`_a1` (Three-Quarter Profile):** Displays side gusset depth, strap attachment rings, and structure.
* **`_a2` (Rear View):** Highlights back slip pockets, zip compartments, or uninterrupted exterior leather.
* **`_a3` (Interior / Open View):** Open top showing internal compartments, lining fabric, and storypatch/creed.
* **`_a4` (Base / Underneath):** Shows base width, protective metal feet, and structural bottom seams.
* **`_a5` (On-Figure / Human Scale):** Primary model shot illustrating proportion against the human body.
* **`_a6` / `_a61` (Secondary Lifestyle / Editorial):** Alternate editorial styling or secondary model framing.
* **`_a92` (Macro Detail Shot):** Tight macro crop of hardware clasps, leather grain, or signature 'C' closures.
* **`_swatch`:** Dedicated square texture/color swatch tile used in PDP variant selectors.

#### 2. Footwear & Shoes
*(Note: Footwear deviates completely from handbag numbering. Profile, sole, and collar views utilize non-sequential codes.)*
* **`_a0`:** Primary hero shot (exterior side profile of single shoe).
* **`_a3`:** Top-down view into collar, insole branding, and footbed.
* **`_a8`:** Lateral side view / profile.
* **`_a9`:** Rear heel counter and pull tab.
* **`_a10`:** Outsole / tread pattern shot.
* **`_a91`:** Close-up macro of toe box, stitching, or lace aglets.
* **`_a99`:** Alternate pair arrangement (e.g., both shoes angled together).
* **`_swatch`:** Material/leather color swatch.

#### 3. Small Leather Goods (SLGs - Wallets, Wristlets, Card Cases)
* **`_a0`:** Front face.
* **`_a1`:** Side edge / zipper profile.
* **`_a2`:** Reverse card slot / ID window.
* **`_a3`:** Interior billfold spread or accordion expansion.

#### 4. Charms, Keychains & Lanyards
* **`_a0`:** Front face.
* **`_a1`:** Angle tilt displaying ring, dog-leash clip, and hardware depth.

---

### C. Advanced Scene7 Protocol Commands (`req=`)

Appending `req=` query parameters exposes powerful origin-level Adobe Dynamic Media functions:

#### 1. Lightweight Angle Probing (`?req=exists`)
To eliminate broken image links and eliminate client-side `404 Not Found` network errors, probe an asset before rendering:
```
https://images.coach.com/is/image/Coach/ccx04_b4mpl_a92?req=exists
```
* **Payload:** Ultra-compact 57-byte plain text response:
  ```
  #S7Z OK
  #Sat Sep 05 18:23:06 UTC 2026
  catalogRecord.exists=1
  ```
* If the view exists: `catalogRecord.exists=1`.
* If the view does not exist: `catalogRecord.exists=0` (HTTP 200 is returned; parse the integer value).

#### 2. Production Lineage & Retouching Audit (`?req=xmp`)
```
https://images.coach.com/is/image/Coach/cfk02_immpl_a0?req=xmp
```
Returns a ~40 KB XML document detailing the master production asset:
* **Studio Hardware:** Identifies the physical lens (`aux:Lens="Canon EF 24-105mm f/4L IS II USM"`), camera body serial number (`aux:SerialNumber`), and lens serial number.
* **Studio Capture Date:** `photoshop:DateCreated` timestamp.
* **Post-Production Audit (`photoshop:History`):** Documents the exact local machine paths (e.g., Indian production facilities working in `+05:30` IST), every vector pen path and anchor point placed by the retoucher in Adobe Photoshop, and the exact timestamp when the file was converted from master 16-bit TIFF to production JPEG.

#### 3. Native Canvas Resolution (`?req=set`)
```
https://images.coach.com/is/image/Coach/cfk02_immpl_a0?req=set
```
Returns the XML image set declaration:
```xml
<set n="Coach/cfk02_immpl_a0" pv="1.0" type="img">
  <item dx="2400" dy="2400" iv="aZjsI1">
    <i n="Coach/cfk02_immpl_a0"/>
  </item>
</set>
```
Reveals that master studio shots are uploaded at **2400 × 2400 pixels**.

#### 4. Delivery Profile Properties (`?req=props`)
```
https://images.coach.com/is/image/Coach/cfk02_immpl_a0?req=props
```
Returns basic image processing configurations:
* Default served canvas size: `1000 × 1000`
* Default JPEG compression quality: `80`
* Background canvas fill: `0xf0f0f0ff` (hex color `#F0F0F0`)
* Color space: `sRGB IEC61966-2.1`

#### 5. Dynamic Rendering Overrides
* **Master High-Res Render:** Request maximum clarity up to native 2400px:
  `https://images.coach.com/is/image/Coach/[style]_[color]_a0?wid=2400&hei=2400&qlt=95`
* **Modern Formats:** Convert dynamically by supplying `fmt=webp` or `fmt=png-alpha`.
* **Preset Profiles:** Append responsive presets used by Coach storefronts:
  * `?$desktopProduct$` (High-res zoomable product crop)
  * `?$desktopThumbnail$` (Gallery view crop)
  * `?$desktopSwatchImage$` (Swatch thumbnail crop)

---

## 4. Salesforce Commerce Cloud (SFCC) Web Storefronts

### A. URL Structure & Slug Independence
Coach runs on Salesforce Commerce Cloud (`coach.com` and `coachoutlet.com`). While marketing links include descriptive product slugs, **the slug is 100% cosmetic for SEO and is completely ignored by the SFCC routing engine.**

* **Full Marketing URL:**
  `https://www.coach.com/products/tabby-shoulder-bag-20/CY201.html?frp=CY201+B4%2FBK`
* **Minimal Functional URL (Programmatically Constructible):**
  `https://www.coach.com/products/CY201.html?frp=CY201+B4%2FBK`
  *(or simply `https://www.coach.com/products/CY201.html` to load the default hero variant)*

### B. Extracted PDP Metadata
Resolving the minimal URL yields:
1. **Full Retail Name & MSRP:** e.g., "Tabby Shoulder Bag 20", "$375".
2. **Physical Dimensions:** Length, height, width in inches and centimeters.
3. **Strap & Handle Drops:** Specific drop measurements for short handles vs. crossbody straps.
4. **Materials & Lining:** e.g., "Natural grain leather", "Fabric lining".
5. **Closure & Pocket Architecture:** Zipper, snap, magnetic closures, and interior/exterior slip configurations.
6. **Full Variant Sibling List:** Every available colorway code and swatch for that parent silhouette.

### C. Anti-Bot / WAF Boundaries
* `coach.com` and `coachoutlet.com` enforce strict Akamai Bot Manager policies. Standard HTTP requests via cURL, Python `requests`, or Node.js `fetch` receive `403 Forbidden`.
* Programmatic retrieval must run through a headless browser (Playwright, Puppeteer), a residential proxy gateway, or a dedicated reader/rendering service (such as Jina Reader).
* **Discontinued / Sold-Out Items:** On `coachoutlet.com`, older styles (e.g., `C9926`) that are fully delisted redirect to a generic 404 page ("Woops! We couldn't find that page"). In contrast, **Adobe Scene7 retains images indefinitely**, even after items are wiped from the public web catalog.

---

## 5. Tapestry In-Store POS & Scanner API (`scan.coach.com`)

Tapestry operates an internal web application for in-store associates and customers (`scan.coach.com`). The backend API gateway is hosted on:
```
https://app.scan.coach.com/api/
```
The catalog identifier for North America is `coach-us` (or `coach-ca` for Canada).

### A. The Item Details Endpoint
```
GET https://app.scan.coach.com/api/iteminfo/catalogs/coach-us/stores/{storeId}/items/{itemId}
```
* `{storeId}`: The 4-digit store code (e.g., `4501` for Coach Kittery).
* `{itemId}`: **MUST be a 12-digit UPC barcode.**

#### Critical Architecture Behavior:
* **Querying by Style + Color (e.g., `CY201`, `cfk02_immpl`):** The API returns HTTP `200 OK`, but **every field in the JSON payload is `null`**. The POS database is strictly indexed by physical inventory barcodes.
* **Querying by 12-Digit UPC (e.g., `196395712960`):** Returns the complete item record:
  ```json
  {
    "brand": null,
    "color": "Optic White",
    "colorCode": "OPI",
    "departmentNumber": "11",
    "imageURL": [
      "Coach/caq25_opi_a0",
      "Coach/caq25_opi_a99",
      "Coach/caq25_opi_a3",
      "Coach/caq25_opi_a8",
      "Coach/caq25_opi_a91",
      "Coach/caq25_opi_swatch",
      "Coach/caq25_opi_a10",
      "Coach/caq25_opi_a9"
    ],
    "itemClass": "305",
    "itemId": "196395712960",
    "productDescription": "<ul>\n\t<li>Leather upper</li>\n\t<li>Fabric lining and footbed</li>\n\t<li>EVA midsole</li>\n\t<li>Rubber outsole</li>\n\t<li>Lace-up closure</li>\n</ul><li>Style No. CAQ25</li>",
    "productId": "CAQ25 OPI  7   B",
    "productName": "SOHO SNKR-OPI-7   B",
    "size": "7",
    "storeNumber": "4501",
    "style": "CAQ25",
    "styleName": null,
    "styleNo": "CAQ25",
    "thumbimage": "Coach/caq25_opi_a0?$thumbnail$",
    "variationGroup": "CAQ25-OPI"
  }
  ```

#### Extracted POS Fields:
* `productName`: Standard retail naming schema including style, color, size, and width (e.g., `SOHO SNKR-OPI-7   B`).
* `departmentNumber`: Internal merchandising department (e.g., `11` = Footwear, `01` / `02` = Handbags/Accessories).
* `itemClass`: POS merchandise subclass (e.g., `305`, `257`).
* `productDescription`: Unsanitized raw HTML containing official bullet-point materials, lining, sole composition, and construction features.
* `imageURL`: Exact array of all available Scene7 asset paths, explicitly revealing all active camera angles.
* `variationGroup`: The parent style-color family identifier (e.g., `CAQ25-OPI`).

---

### B. Sibling Size & Variant Matrix (`retrive-similar-items`)
```
GET https://app.scan.coach.com/api/iteminfo/catalogs/coach-us/stores/{storeId}/retrive-similar-items/{itemId}
```
*(Note: Tapestry's production URL literally uses the spelling `retrive-similar-items`.)*

By supplying a single valid UPC (`itemId`), this endpoint returns the **complete matrix of all child UPCs across all sizes and colorways** tied to that parent variation group:
* `sizeMap`: Maps every shoe/clothing size (e.g., `5`, `5.5`, `6`, ..., `11`) directly to its corresponding 12-digit UPC `itemId`.
* `colorNameMap`: Lists all alternate colorways available in the store catalog with their swatch URLs.

---

### C. In-Store Pricing & Session Token Minting

The pricing route requires an `Authorization: Bearer <token>` header:
```
POST https://app.scan.coach.com/api/priceinfo/catalogs/coach-us/stores/{storeId}/items/{itemId}
```

#### Anonymous Token Minting via Empty PIN:
In many stores (including store `4501`), store PIN validation is disabled (`pinActivatedInStore: false`). A valid session token can be minted anonymously:

1. **Mint Token:**
   * **URL:** `POST https://app.scan.coach.com/api/validatepin/catalogs/coach-us/stores/{storeId}`
   * **Payload:** `{"storeId": "{storeId}", "pin": ""}`
   * **Response:**
     ```json
     {
       "pinActivatedInStore": false,
       "pinNotExistsForDuration": false,
       "storeId": "4501",
       "token": "XzA83N6eoj3Kq5M1mF/B5iqmnQ/brh3vEsc5AGRWGNM=",
       "valid": true
     }
     ```
2. **Execute Pricing Query:**
   Pass the returned token in the `Authorization: Bearer {token}` header to `/api/priceinfo/...` with payload:
   ```json
   {
     "couponNum": [],
     "isLoggedIn": "false",
     "userAppliedCouponNum": "",
     "personalizedPromosList": []
   }
   ```

---

## 6. Clearance & "Last Chance!" Badge Detection Engine

### A. Reverse-Engineered Frontend Rendering Logic
Inside `scan.coach.com`'s core UI chunk (`9.[hash].chunk.js`), the clearance badge is governed by a strict array membership check on `itemGroups`:

```javascript
// e is the itemPriceInfo object returned by /api/priceinfo/...
const a = e.itemGroups && Array.isArray(e.itemGroups) && e.itemGroups.includes("2");

return i.a.createElement(
  "div",
  { className: "final-price-container " + (e.className || "") },
  // ... MSRP / Comparable Value ...
  // ... Current Sale Price ...
  // ... Percentage Off ...
  a ? i.a.createElement(
        "span",
        { className: "last-chance-badge", "aria-label": "Last Chance!" },
        "Last Chance!"
      ) : ""
);
```

### B. Tapestry POS Merchandise Hierarchy
* **`itemGroups.includes("2")`:** Identifies **Clearance / "Last Chance!"** merchandise. Triggers the red UI badge.
* **`itemGroups: ["5"]`:** Standard mainline outlet / non-clearance inventory.
* **Clearance Promo Array:** When an item is marked "Last Chance", the `appliedPromotions` array populates with promotional metadata (e.g., `20% OFF LAST CHANCE`), including discount amounts, discount percentages, and promotion IDs.

### C. Live Pricing Payload Reference (Clearance Item `198685143560`)
```json
{
  "appliedPromotions": [
    {
      "couponId": "",
      "discountAMT": 23.1,
      "discountPCT": 20.0,
      "discountType": "PCT_",
      "displayName": "20% OFF LAST CHANCE",
      "myoffer": false,
      "priceAfterPromo": 92.4,
      "promoNameTranslations": {
        "en-us": "20% Off Last Chance",
        "es": "20% OFF LAST CHANCE",
        "fr": "20% OFF LAST CHANCE",
        "ko": "20% OFF LAST CHANCE",
        "pt": "20% OFF LAST CHANCE",
        "zh": "20% OFF LAST CHANCE"
      },
      "promotionId": "200171",
      "promotionType": "TTHR"
    }
  ],
  "brand": "Coach",
  "itemGroups": [
    "2",
    "5"
  ],
  "itemId": "198685143560",
  "itemSalePrice": 115.5,
  "mfsrp": 165.0,
  "pdpsalePrice": 92.4,
  "savingsAmount": 72.6,
  "savingsPCT": 44.0,
  "storeNumber": "4501"
}
```

#### Field Mappings:
* `mfsrp`: Comparable Value / Original MSRP ($165.00)
* `itemSalePrice`: Base store sale price before extra promos ($115.50)
* `pdpsalePrice`: Final checkout price after promotions ($92.40)
* `savingsAmount`: Total dollar savings ($72.60)
* `savingsPCT`: Total percentage discount (44% OFF)
* `itemGroups`: Contains `"2"` $\rightarrow$ **Last Chance! Flag = TRUE**
* `appliedPromotions[0].displayName`: Promotion label ("20% OFF LAST CHANCE")

---

## 7. Unified Cross-System Resolution Matrix

| Target Information | Required Minimal Input | Target Endpoint | Bot/WAF / Auth Restrictions |
| :--- | :--- | :--- | :--- |
| **Product Images (Hero / Detail)** | Style + Color (`ccx04_b4mpl`) | `https://images.coach.com/is/image/Coach/[style]_[color]_[angle]` | **None.** Open CORS (`*`). Unauthenticated direct GET. |
| **Angle Existence Check** | Style + Color + Angle | `https://images.coach.com/is/image/Coach/[style]_[color]_[angle]?req=exists` | **None.** Returns 57-byte `catalogRecord.exists=1` or `0`. |
| **Master Resolution Render** | Style + Color (`2400×2400`) | `https://images.coach.com/is/image/Coach/[style]_[color]_a0?wid=2400&hei=2400&qlt=95` | **None.** Renders native 2400px master asset. |
| **Production Lineage & EXIF** | Style + Color | `https://images.coach.com/is/image/Coach/[style]_[color]_a0?req=xmp` | **None.** Returns 40KB XML (lens, serials, edit history). |
| **Official MSRP, Dimensions, Straps** | Style Number (`CY201`) + optional Color (`B4%2FBK`) | `https://www.coach.com/products/[STYLE].html?frp=[STYLE]+[COLOR]` | **Akamai WAF.** Requires headless browser or scraping gateway. **No slug needed.** |
| **POS Name, Class, Dept, Bullet Specs** | 12-Digit UPC (`196395712960`) + Store (`4501`) | `https://app.scan.coach.com/api/iteminfo/catalogs/coach-us/stores/[store]/items/[upc]` | **None.** Direct JSON GET. Style+Color combo yields `null`. |
| **Full Sibling Size Matrix & UPCs** | Single 12-Digit UPC (`196395712960`) + Store | `https://app.scan.coach.com/api/iteminfo/catalogs/coach-us/stores/[store]/retrive-similar-items/[upc]` | **None.** Direct JSON GET. Reconstructs all sibling sizes/UPCs. |
| **Store Pricing & "Last Chance" Status** | 12-Digit UPC + Store | `https://app.scan.coach.com/api/priceinfo/catalogs/coach-us/stores/[store]/items/[upc]` | **Auth Required.** POST request requiring token minted via `/validatepin` with empty PIN. |

---

## 8. Reference Implementation (Python Client)

```python
import urllib.request
import json

class CoachCatalogClient:
    def __init__(self, store_id: str = "4501", catalog: str = "coach-us"):
        self.store_id = store_id
        self.catalog = catalog
        self.base_url = "https://app.scan.coach.com/api"
        self._token = None

    def get_token(self) -> str:
        """Mints an anonymous session token using store empty PIN validation."""
        if self._token:
            return self._token
        url = f"{self.base_url}/validatepin/catalogs/{self.catalog}/stores/{self.store_id}"
        req = urllib.request.Request(
            url,
            data=json.dumps({"storeId": self.store_id, "pin": ""}).encode("utf-8"),
            headers={"Content-Type": "application/json"}
        )
        with urllib.request.urlopen(req) as resp:
            data = json.loads(resp.read().decode("utf-8"))
            self._token = data.get("token")
            return self._token

    def get_item_info(self, upc: str) -> dict:
        """Retrieves item metadata, class, department, and Scene7 asset paths."""
        url = f"{self.base_url}/iteminfo/catalogs/{self.catalog}/stores/{self.store_id}/items/{upc}"
        req = urllib.request.Request(url, headers={"Accept": "application/json"})
        with urllib.request.urlopen(req) as resp:
            return json.loads(resp.read().decode("utf-8"))

    def get_pricing_and_status(self, upc: str) -> dict:
        """Retrieves live pricing, savings %, promotions, and Last Chance clearance status."""
        token = self.get_token()
        url = f"{self.base_url}/priceinfo/catalogs/{self.catalog}/stores/{self.store_id}/items/{upc}"
        payload = {
            "couponNum": [],
            "isLoggedIn": "false",
            "userAppliedCouponNum": "",
            "personalizedPromosList": []
        }
        req = urllib.request.Request(
            url,
            data=json.dumps(payload).encode("utf-8"),
            headers={
                "Content-Type": "application/json",
                "Authorization": f"Bearer {token}"
            }
        )
        with urllib.request.urlopen(req) as resp:
            data = json.loads(resp.read().decode("utf-8"))

        item_groups = data.get("itemGroups") or []
        is_last_chance = "2" in item_groups
        promos = data.get("appliedPromotions") or []

        return {
            "upc": upc,
            "is_last_chance": is_last_chance,
            "comparable_value": data.get("mfsrp"),
            "sale_price": data.get("pdpsalePrice"),
            "savings_amount": data.get("savingsAmount"),
            "savings_pct": round(data.get("savingsPCT", 0)),
            "promotions": [p.get("displayName") for p in promos]
        }

    @staticmethod
    def check_scene7_angle_exists(style: str, color: str, angle: str = "a0") -> bool:
        """Probes Scene7 for angle existence using the 57-byte check."""
        clean_style = style.lower()
        clean_color = color.lower().replace("/", "")
        url = f"https://images.coach.com/is/image/Coach/{clean_style}_{clean_color}_{angle}?req=exists"
        req = urllib.request.Request(url, headers={"User-Agent": "Mozilla/5.0"})
        try:
            with urllib.request.urlopen(req, timeout=5) as resp:
                text = resp.read().decode("utf-8")
                return "catalogRecord.exists=1" in text
        except Exception:
            return False
