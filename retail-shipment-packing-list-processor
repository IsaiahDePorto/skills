---
name: retail-shipment-packing-list-processor
description: Provides complete structural specifications, parsing rules, schema definitions, and multi-page stitching logic for incoming shipment packing list PDFs (Kittery Store SWK Family of Brands). Use this skill whenever an agent needs to extract, structure, normalize, validate, or analyze incoming retail shipment documents and packing list tables without hallucinating carton IDs or dropping unlinked continuation pages.
metadata:
  version: "1.0.0"
  domain: "retail-logistics"
---

# Retail Shipment Packing List Processing Guide

## 1. System Overview & Context

This document provides the definitive structural specification for parsing, extracting, and processing incoming shipment packing list PDFs for the retail store (**Store Code: `000004`**, **Store Name: `KITTERY STORE SWK FAMILY OF BRANDS`**). 

The source documents consist of multi-page or single-page PDF batches representing physical cartons delivered to the store. Each PDF may contain one or multiple distinct packing lists corresponding to individual shipping cartons.

---

## 2. Document Architecture & Multi-Page Linking Logic

A batch PDF consists of two distinct page classifications: **First Pages** (carton anchors) and **Subsequent Continuation Pages**.

### 2.1. First Page Structure (List Initialization)
A First Page always initializes a new packing list / carton record and includes:
- **Top Right Store Block:** Store Name (`KITTERY STORE SWK FAMILY OF BRANDS`) and Store ID (`000004`).
- **Left Header:** Title string `PACKING LIST`.
- **Right Header:** Page indicator formatted as `Page 1 of N` (e.g., `Page 1 of 1`, `Page 1 of 2`, `Page 1 of 3`).
- **Shipment Metadata Block:** Key-value pairs containing `Ship Via`, `Carton No.`, `Order Date`, `Request Shipping Date`, `Cancel Date`, and `Backorders Accepted`.
- **Order Banner:** A shaded banner containing `Purchase Order No.` and `Sales Order No.`.
- **Table Column Headers:** `RPRO No`, `ITEM NO.`, `Quantity Shipped (Each, Case)`, `ITEM DESCRIPTION`.
- **Table Line Items:** Initial batch of inventory records.

### 2.2. Subsequent Continuation Pages (`Page X of N` where `X > 1`)
Subsequent pages represent the overflow of the line-item table belonging to the immediately preceding First Page:
- **Top Right Store Block:** Store Name (`KITTERY STORE SWK FAMILY OF BRANDS`) and Store ID (`000004`).
- **Header:** `PACKING LIST` (left) and `Page X of N` (right, e.g., `Page 2 of 3`, `Page 3 of 3`).
- **Omitted Elements:** The Shipment Metadata Block (`Carton No.`, `Ship Via`, dates) and Order Banners are entirely absent.
- **Table Body:** Continuation of the line-item rows from the previous page.

### 2.3. Page Stitching & State Management Rules
1. **Carton Context Inheritance:** When processing sequential pages in a PDF, the active `Carton No.` initialized on `Page 1 of N` remains active and binds to all subsequent pages until a new `Page 1 of M` is encountered.
2. **Page Count Integrity:** Verify that pages `1` through `N` are accounted for in order. A packing list is complete only when page `N of N` has been processed.

---

## 3. Field Taxonomy & Operational Classifications

Fields within the document are strictly divided into **Active Analytical Fields** (used for extraction, reporting, and operational logic) and **Passive Verbatim Fields** (captured verbatim for audit trails but excluded from business logic and analytics).

### 3.1. Active Analytical Fields
- **`Carton No.`:** The unique physical package/shipping container identifier (e.g., `LP0001004327`, `LP0001004399`). Located solely on Page 1.
- **`RPRO No`:** The Retail Pro software inventory SKU code. Typically a 2- to 5-digit number (e.g., `16118`, `34`, `42`, `32380`).
- **`ITEM NO.`:** The manufacturer item code, vendor SKU, or alphanumeric product identifier (e.g., `150815`, `SND101221`, `IMP100405`, `PKM00015`, `807332B`, `RMM00827`).
- **`Quantity Shipped: Each`:** The total count of individual discrete units shipped (e.g., `6`, `12`, `24`, `1000`).
- **`Quantity Shipped: Case`:** The number of physical cases/boxes containing the eaches. When multiplied by the units-per-case, it equals the `Each` count. Can be an integer or a dash (`-`).
- **`ITEM DESCRIPTION`:** The exact text name and size/variant description of the product (e.g., `Down East Tartar Sauce 7.5oz`, `Wild Maine Blueberry Jam 3.75oz`, `Tea Towel Red`).

### 3.2. Passive Verbatim Fields (Capture Verbatim, Do Not Analyze)
The following fields must be extracted and preserved verbatim in output data structures for record-keeping, but **must never be used in operational calculations, stock filtering, grouping, or analytical comparisons**:
- `Purchase Order No.` (e.g., `KITADD082526`, `KIT0824091725555`)
- `Sales Order No.` (e.g., `A4564566-001`, `A4559749-001`)
- `Ship Via` (e.g., `HANDDELIVR`)
- `Order Date` (e.g., `8/24/2026 12:00:00 AM`)
- `Request Shipping Date` (e.g., `8/25/2026 12:00:00 AM`)
- `Cancel Date` (often blank/null)
- `Backorders Accepted` (e.g., `NO`)

---

## 4. Critical Edge Cases & Parsing Invariants

### Invariant 1: The Store Code `000004` Is NEVER a Carton Number
The number `000004` is printed at the top-right header of every single page (both first pages and subsequent pages) directly below the store name. 
- **Rule:** `000004` is exclusively the Store Location ID. Never assign, parse, or fallback to `000004` as a `Carton No.`.

### Invariant 2: Missing / Blank `RPRO No`
Certain items (e.g., store supplies like packing tape `PKM00015`, display boxes `PKS00144`, tissue paper `SGM00119`, or pre-assembled gift baskets like `191311`) do not possess a Retail Pro number on the packing slip.
- **Rule:** When the `RPRO No` column is absent, preserve the value as empty/blank (or null). Do not reject or fail the row; continue extracting the `ITEM NO.`, quantities, and description.

### Invariant 3: Non-Numeric / Dash (`-`) Case Quantities
Items shipped as loose units, single supplies, unpackaged displays, or un-cased items will have a dash (`-`) or empty space in the `Case` column.
- **Rule:** Preserve the dash (`"-"`) or empty string verbatim. Do not coerce it to `0` or force integer conversion unless explicitly requested by a specific downstream schema.

### Invariant 4: Blank Subsequent Continuation Pages
Occasionally, a multi-page document will contain a trailing continuation page (e.g., `Page 2 of 2` or `Page 3 of 3`) where the header is printed normally, but the table body below is completely blank with zero line items.
- **Rule:** Safely ignore and skip blank continuation pages without failing validation. The packing list remains complete and valid.

### Invariant 5: Initial Capture of Duplicate Cartons
PDF batches frequently contain repeated identical pages or duplicate carton manifests (e.g., identical `Carton No.` entries repeated across different pages in the same file or across separate files).
- **Rule:** During initial ingestion and extraction, capture every carton occurrence verbatim. Downstream de-duplication must only be executed as an explicit secondary pipeline step.

---

## 5. Standardized Data Schema & Extraction Flow

When generating structured datasets (JSON, CSV, relational models, or programmatic dictionaries) from these documents, follow this standard entity hierarchy:

### 5.1. Packing List Level (Carton Record)
- `carton_number`: String (from Page 1 metadata)
- `store_id`: String (constant `"000004"`)
- `store_name`: String (constant `"KITTERY STORE SWK FAMILY OF BRANDS"`)
- `page_count_total`: Integer (extracted from `Page 1 of N`)
- `passive_metadata`: Object containing:
  - `purchase_order_no`: String
  - `sales_order_no`: String
  - `ship_via`: String
  - `order_date`: String
  - `request_shipping_date`: String
  - `cancel_date`: String
  - `backorders_accepted`: String
- `line_items`: Array of Line Item records (aggregated across all pages `1` through `N`).

### 5.2. Line Item Level
- `rpro_no`: String (empty string if blank on list)
- `item_no`: String (mandatory SKU/UPC)
- `quantity_each`: Integer (total units)
- `quantity_case`: String or Integer (integer count or `"-"`)
- `item_description`: String (verbatim item title)

---

## 6. Execution & Quality Checklist for Agents

Before completing any extraction or analysis task involving these packing lists, verify:
- [ ] Has every `Carton No.` been extracted strictly from the metadata block on Page 1 (and never from `000004`)?
- [ ] Were multi-page line items correctly combined under their parent `Carton No.` based on `Page X of N`?
- [ ] Were blank continuation pages gracefully skipped without dropping the parent list?
- [ ] Did rows with blank `RPRO No` values retain their `ITEM NO.` and quantities without throwing errors?
- [ ] Were dash (`-`) case entries preserved without corrupting numeric columns?
- [ ] Were passive metadata fields preserved verbatim without contaminating operational item counts?
- [ ] Were duplicate carton pages captured during initial ingestion prior to secondary de-duplication?
