# Logistics & Invoice Management System

> An enterprise-grade, full-stack web application for end-to-end management of international vehicle imports, container shipments, customs invoicing, and multi-currency financial operations.

---

## Table of Contents

- [Business Problem](#business-problem)
- [Tech Stack](#tech-stack)
- [Architecture Overview](#architecture-overview)
- [Key Features & Modules](#key-features--modules)
- [Database Design](#database-design)
- [API Structure](#api-structure)
- [Challenges Solved](#challenges-solved)
- [Project Scale](#project-scale)
- [Author](#author)

---

## Business Problem

International vehicle import/export operations involve a dense web of interdependent processes: sourcing vehicles from Japanese auctions, tracking chassis numbers through yards and containers, generating country-specific customs invoices, managing multi-party shipping logistics, reconciling currency conversions, and coordinating with dozens of consignees across multiple destination countries.

Existing off-the-shelf software cannot accommodate the domain-specific rules involved — e.g., invoice type name overrides per destination country, per consignee, and per vehicle type (hybrid/non-hybrid/bike), chassis exclusivity constraints across containers, or per-container battery data sheets for hybrid exports.

This system was built from scratch to replace a fragmented spreadsheet workflow and now serves as the operational backbone for the company's entire import/export pipeline.

---

## Tech Stack

| Layer | Technology | Why |
|---|---|---|
| **Backend Framework** | CodeIgniter 3 (PHP) | Lightweight MVC, stable, easy onboarding for domain team |
| **Database** | MySQL on AWS RDS | Managed, multi-schema, production-ready |
| **File Storage** | AWS S3 + CloudFront CDN | Scalable object storage for documents and vehicle images |
| **PDF Generation** | mPDF | Full Unicode support for multi-language customs invoice PDFs |
| **Excel Export** | PHPExcel | Complex multi-sheet, formula-capable export reports |
| **Email** | Custom Mailer Library | Integrated notification workflows |
| **Frontend** | jQuery, Bootstrap, DataTables | Responsive UI with server-side paginated data tables |
| **Charts** | NVD3, Morris, Rickshaw | Financial and operational dashboards |
| **Mobile API** | REST (CodeIgniter Services controller) | Device-registered token auth for mobile clients |
| **Backup Storage** | FTP via custom handler | Secondary file backup alongside S3 |

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────┐
│                     HTTP Requests                         │
│           (Web Browser / Mobile App / API Client)         │
└────────────────────────┬─────────────────────────────────┘
                         │
                ┌────────▼────────┐
                │   Controllers   │  52 controllers
                │  (JI_Controller │  Role/permission gating
                │   base class)   │  on every endpoint
                └────────┬────────┘
                         │
           ┌─────────────┼──────────────┐
           │             │              │
    ┌──────▼──────┐ ┌────▼─────┐ ┌─────▼──────┐
    │   Models    │ │Libraries │ │  Helpers   │
    │ (73+ files) │ │ Service  │ │ menu, kas, │
    │ Business    │ │ Mailer   │ │ common,    │
    │ Logic +     │ │ Aws_sdk  │ │ utilities  │
    │ Data Layer  │ │ FtpUpload│ └────────────┘
    └──────┬──────┘ └──────────┘
           │
    ┌──────▼────────────────┐
    │      Data Stores      │
    │  MySQL (3 schemas)    │
    │  AWS S3 (images/docs) │
    └───────────────────────┘
```

### Design Patterns Used

- **Base Controller / Base Model** — `JI_Controller` and `JI_Model` enforce session auth, timezone, and shared utilities app-wide across all 52 controllers.
- **Global State via `$GLOBALS`** — Authenticated user context (`user_id`, `role`, `country_ids`, `menus`) is set once at the controller level and consumed downstream, avoiding repeated session reads.
- **Autoloaded Model Injection** — 25+ models are globally autoloaded so all controllers share a consistent model surface without repetitive manual `load->model()` calls.
- **Permission-Gated Field Locking** — Granular RBAC (116+ permission types) locks form fields at both the server-render layer (PHP) and client fetch layer (JS) — two-part enforcement per sensitive field.
- **Dual Persistence** — Files are written to both S3 (primary) and FTP (backup), handled transparently by the upload layer.

---

## Key Features & Modules

### 1. Container Management
- Create and manage shipping containers with full packing lists.
- Assign vehicles, parts, tyres, and electronics to containers with per-row detail.
- Track `container_packing_other` rows with associated chassis numbers and embedded images.
- Custom autocomplete for chassis assignment with **exclusive assignment logic** — selecting a chassis in one slot auto-clears it from any other slot in real time.
- Per-country and per-consignee rules enforced at packing time.

### 2. Invoice Generation (PDF + Excel)
- Country-specific invoice templates with layered override logic:
  - Override by destination country
  - Override by consignee identity
  - Override by vehicle type (hybrid, bike, standard)
- Multi-format output: mPDF for print-ready customs invoices, PHPExcel for structured Excel exports.
- Battery data sheet — auto-generated per container for hybrid vehicle exports, embedding chassis numbers, battery IDs, and inline CDN images.
- Tanzania-specific row suppression logic (tyre rows hidden, invoice header overridden).
- Cambodia-specific type name cascade (consignee → bike check → default).
- Description prefix stripping applied consistently across 5 code paths in 3 model files.

### 3. Vehicle (Car / RORO) Tracking
- Full vehicle lifecycle: auction purchase → yard → container → shipping → destination.
- Chassis number tracing across all system events with a dedicated trace view.
- Support for RORO (Roll-on/Roll-off) vessel scheduling and invoicing alongside container shipments.
- Fuel type classification for hybrid detection, feeding into battery export rules.

### 4. Auction / TT Module
- Tracks vehicles sourced from Japanese auction markets.
- Treasure Truck (TT) statement generation and cash flow reporting.
- Monthly market targets, per-market purchase limit controls.
- Cross-country chassis code transfer queries.

### 5. Role-Based Access Control (RBAC)
- 116+ granular permissions defined in `User.php`.
- Permissions control not just page access but individual form fields, dropdown options, and data visibility.
- Country-scoped access — users see only data for their assigned destination countries.
- Session-based auth with 2-hour expiry; device token auth for mobile clients with per-device OS tracking.

### 6. Mobile REST API
- `ws/*` route namespace serves a companion mobile application.
- Device registration with OS tracking in `mapp_users` table.
- Authenticated via `app_user_id` + `device_id` pair.
- Endpoints cover containers, cars, invoices, auctions, exchange rates, and AR data.

### 7. Financial Reporting
- Invoice-level accounts receivable (AR) tracking.
- Costing module with expense head breakdown per shipment.
- Shipping company rate management with per-route charge structures.
- Port clearance tracking tied to individual invoices.
- Excel-formatted KAS reports and a flexible custom report builder.

### 8. SOC & WAPP Containers
- Shipper's Own Container (SOC) and WAPP (Japan) container types with dedicated invoice workflows.
- Separate management views and controller paths per container type to keep business rules isolated.

### 9. Spare Parts, Tyres & Electronics
- Parallel inventory modules for non-vehicle cargo types.
- Each has its own invoice type, packing logic, and Excel export format — sharing base patterns but keeping domain rules separate.

---

## Database Design

The system uses **three MySQL schemas** on a single AWS RDS instance:

| Schema | Purpose |
|---|---|
| `main_db` | Core operational data — invoices, cars, containers, users |
| `purchase_db` | Purchase and auction transaction records |
| `auxiliary_db` | Supplementary data (freezone and pre-freezone records) |

### Core Table Groups

```
Users & Auth
├── user_management          user accounts, roles, country assignments
├── user_role_management     RBAC permission assignments (116+ types)
└── mapp_users               mobile device registrations with OS tracking

Inventory
├── car_master               vehicle records (chassis, fuel type, specs)
├── yard_master              yard / storage locations
├── parts_master             spare parts catalog
├── tyre_master              tyre products
└── electronics_master       electronics items

Container & Shipping
├── container_master         container records
├── container_packing_cars   vehicles assigned to containers
├── container_packing_other  non-car packing items (chassis + image support)
├── shipping_company         shipping lines and rates
└── roro_master              RORO vessel schedules

Invoices & Finance
├── invoice_master           invoice headers (type, country, consignee)
├── invoice_detail           per-item invoice rows
├── merged_invoices          combined / merged invoice records
├── costing_master           cost breakdown entries per shipment
└── port_clearance           customs clearance tracking per invoice

Auction / Market
├── mukechi_master           auction lot records
├── tt_master                Treasure Truck entries
└── market_limits            per-market monthly purchase limits

Masters / Lookup
├── country_master           destination countries
├── port_master              shipping ports
├── consignee_master         importers / consignees
├── auction_company_master   auction houses
└── description_master       cargo description catalog (id 187 = hybrid battery)
```

> No ORM or migration framework. Schema is managed via raw SQL against AWS RDS — a deliberate choice to keep the database layer predictable and the team's SQL knowledge the primary interface, rather than adding an abstraction layer over a highly domain-specific schema.

---

## API Structure

The `ws/*` route namespace exposes a REST-style API consumed by the mobile application.

### Authentication
```
POST  ws/login_authanticate
      Body: { login_id, password, device_id, os }
      Returns: user profile + country access list + session token
```

### Key Endpoints

| Endpoint | Description |
|---|---|
| `ws/get_containers` | Container list with packing summary |
| `ws/get_cars` | Vehicle list with current status |
| `ws/get_mukechi_list` | Auction lot listing |
| `ws/get_auction_companies` | Auction house master list |
| `ws/invoices_ar` | AR invoice data |
| `ws/get_cars_ar` | Vehicle-level AR data |
| `ws/get_exchange_rate` | Live exchange rate lookup |

All endpoints enforce session or device token authentication. Country-scoped filtering is applied automatically based on the authenticated user's assigned countries.

### Web-facing AJAX Endpoints (selected)

| Endpoint | Purpose |
|---|---|
| `get_chassis_list` | Chassis autocomplete source for forms |
| `containers_get` | Container data fetch for add/edit form |
| `get_ajax_invoice_details` | Invoice detail fetch (AJAX) |
| `chassis_trace` | Full lifecycle trace for a given chassis |

---

## Challenges Solved

### 1. Country-Specific Invoice Customization at Scale
**Problem:** Different destination countries require different invoice type names, row suppression rules, and field labels — sometimes varying further by consignee or vehicle type within the same country.

**Solution:** A layered override system in both the PDF model and Excel model. Priority order: consignee override → vehicle-type override → country default. Each override is a conditional check on `to_country_id`, `consignee_id`, or `fuel_id`, applied at row-generation time so the core output logic remains clean and each rule is isolated.

---

### 2. Chassis Exclusive Assignment in Container Packing
**Problem:** A chassis number should appear in at most one "other item" slot in the container packing form. The form is fully dynamic (rows added/removed client-side), making server-side-only enforcement awkward.

**Solution:** A custom JS autocomplete component that:
- Re-reads all current main chassis inputs on each focus event (fresh scan, no stale state).
- Greys out already-assigned chassis values (still selectable for re-assignment).
- On selection, scans all other `child_chassis` fields and clears any duplicate — enforcing exclusivity at the UI layer before the form is submitted.

---

### 3. Multi-Schema Query Coordination
**Problem:** Operational data lives across three MySQL schemas. CodeIgniter's query builder is single-connection by default and doesn't natively support cross-schema joins.

**Solution:** Multiple DB connection objects are initialized in `database.php`. Cross-schema queries use raw SQL with schema-prefixed table names (e.g., `purchase_db.table_name`). A set of environment constants maps schema names so queries stay portable between dev and production without code changes.

---

### 4. PHPExcel Sheet Registration Order
**Problem:** PHPExcel throws errors when `getStyle()` is called on a sheet before it is registered with the workbook. This caused silent failures in the battery export when sheets were built in a loop across containers.

**Solution:** Enforced strict ordering — `addSheet()` → `setTitle()` → then any `getStyle()` or cell writes. Identified this constraint from PHPExcel internals and documented it so the ordering is preserved through future edits.

---

### 5. Binary Image Embedding in Excel from CDN
**Problem:** Battery data sheets require vehicle images embedded directly into Excel cells, fetched at generation time from CDN URLs that may serve images in any format.

**Solution:** Used `imagecreatefromstring(file_get_contents($img_url))` with `PHPExcel_Worksheet_MemoryDrawing` and `MIMETYPE_DEFAULT` rather than `MIMETYPE_JPEG` — the only combination that handles mixed image formats from CDN reliably without requiring known MIME types up front.

---

### 6. Field-Level Permission Enforcement Without Breaking Form Submission
**Problem:** Certain fields (e.g., destination country on a container) must be read-only for users without a specific permission. A standard `disabled` attribute on a `<select>` excludes the field from the POST body, breaking backend validation.

**Solution:** Two-part enforcement:
1. **On data fetch (JS):** Strip all non-selected options from the dropdown after data loads, leaving only the current value. Functionally read-only — no `disabled` needed.
2. **On server render (PHP):** Conditionally output only the matching `<option>` when rendering the edit form, so restricted users can never get extra options even via direct HTML inspection.

Both layers are necessary — JS alone can be bypassed; PHP alone doesn't cover dynamically fetched form data.

---

## Project Scale

| Metric | Value |
|---|---|
| Controllers | 52 |
| Models | 73+ |
| Views | 1,000+ |
| Defined Routes | 1,130+ |
| RBAC Permission Types | 116+ |
| Autoloaded Models | 25 |
| Largest Model File | ~818 KB (Excel export model) |
| Database Schemas | 3 (MySQL on AWS RDS) |
| REST API Endpoints (`ws/*`) | 15+ |
| View Modules / Subdirectories | 38 |

---

## Author

**Kashif Umar** — Backend Developer

Designed and built the core backend architecture, multi-module data layer, REST API, permission system, country-specific invoice logic, and Excel/PDF generation pipeline for this system.

---

*Built with CodeIgniter 3 · PHP · MySQL · AWS RDS · AWS S3 · PHPExcel · mPDF*
