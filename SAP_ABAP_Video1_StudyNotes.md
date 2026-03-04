# SAP ABAP Study Notes — Video 1 of 11
> Based on: "FREE Video 1 of 11 - Learn SAP ABAP for Free for Freshers & Functional Consultants" by ZAP Yard
> Video: https://www.youtube.com/watch?v=PkOdOd6t54w

---

## Table of Contents

1. [ERP & SAP Overview](#1-erp--sap-overview)
2. [SAP 3-Tier Architecture](#2-sap-3-tier-architecture)
3. [SAP GUI & Transaction Codes](#3-sap-gui--transaction-codes)
4. [System Landscape: Dev → QA → Prod](#4-system-landscape)
5. [SAP Implementation Phases](#5-sap-implementation-phases)
6. [ABAP Introduction](#6-abap-introduction)
7. [Data Dictionary (SE11) — Core Concepts](#7-data-dictionary-se11)
8. [Domains](#8-domains)
9. [Data Elements](#9-data-elements)
10. [Structures](#10-structures)
11. [Database Tables](#11-database-tables)
12. [Table Maintenance (SM30)](#12-table-maintenance-sm30)
13. [Packages & Transport Requests](#13-packages--transport-requests)
14. [Key Takeaways & Revision Points](#14-key-takeaways--revision-points)

---

## 1. ERP & SAP Overview
*[Video: ~02:13]*

**ERP (Enterprise Resource Planning)** = software that integrates all core business functions into one unified system with a **single shared database**.

Without ERP:
- Each department (HR, Finance, Warehouse, Sales) has its own isolated system
- Data is duplicated, out of sync, requires manual reconciliation

With SAP ERP:
- One database, all departments read/write the same data in real time
- A goods receipt in Warehouse automatically updates Inventory and posts a Finance document — no manual steps

**SAP** is the world's leading ERP vendor. Their product manages:

| Module | Domain |
|---|---|
| MM (Materials Management) | Procurement, Inventory |
| SD (Sales & Distribution) | Orders, Billing, Delivery |
| FI (Financial Accounting) | GL, AP, AR |
| CO (Controlling) | Cost Centers, Profit Centers |
| HR/HCM | Payroll, Personnel |
| PP (Production Planning) | Manufacturing |
| WM/EWM | Warehouse Management |

> **Mental model**: SAP is a massive pre-built application covering ~100,000 business processes. Your job as an ABAP developer is to **extend and customize** it — not rebuild it.

---

## 2. SAP 3-Tier Architecture
*[Video: ~12:47]*

```
┌─────────────────────────────────────┐
│         PRESENTATION LAYER          │  ← SAP GUI (saplogon.exe)
│    User's PC / SAP Fiori Browser    │     or web browser (Fiori)
└──────────────────┬──────────────────┘
                   │ Requests (DIAG protocol / HTTP)
┌──────────────────▼──────────────────┐
│         APPLICATION LAYER           │  ← SAP Application Server
│   ABAP Programs, Business Logic,    │     (runs on Linux/Windows)
│   Work Processes, Message Server    │
└──────────────────┬──────────────────┘
                   │ SQL (via Open SQL abstraction)
┌──────────────────▼──────────────────┐
│           DATABASE LAYER            │  ← Oracle / SAP HANA /
│     All SAP Tables & Data live here │     MS SQL Server / DB2
└─────────────────────────────────────┘
```

### Layer Responsibilities

**Presentation Layer**
- Just renders screens — no business logic here
- SAP GUI (thick client) communicates using SAP's proprietary **DIAG protocol**
- Modern SAP uses **Fiori** (browser-based, communicates via OData/REST)

**Application Layer**
- Where ABAP programs run
- Multiple **Work Processes** handle parallel requests (Dialog WP, Background WP, Update WP, Spool WP, Enqueue WP)
- **Message Server** handles load balancing between multiple application servers
- **ICM** (Internet Communication Manager) handles HTTP/S traffic

**Database Layer**
- All SAP data physically stored here
- Application layer never talks directly to DB with native SQL — always via **Open SQL** (SAP's abstraction layer)
- This is why the same ABAP code runs on Oracle, HANA, or MSSQL unchanged

---

## 3. SAP GUI & Transaction Codes
*[Video: ~18:01 — 19:28]*

### Checking System Status
- Menu: **System → Status** — shows:
  - System ID (SID), Client, Server name
  - SAP release version
  - Current logged-in user details

### Transaction Codes (T-Codes)
T-Codes are shortcuts to directly launch SAP applications — like URL routes in a web app.

**How to use:**
- Type in the **Command Field** (top-left box) and press Enter
- `/n` prefix = open in **same** session (e.g., `/nSE38`)
- `/o` prefix = open in **new** session (e.g., `/oSE11`)

**Essential T-Codes for ABAP Developers:**

| T-Code | Purpose |
|---|---|
| `SE11` | ABAP Data Dictionary (DDIC) |
| `SE38` | ABAP Editor — write & run programs |
| `SE80` | Object Navigator — IDE-like browser |
| `SE37` | Function Module editor |
| `SE24` | Class Builder (OOP) |
| `SM30` | Table Maintenance |
| `SE16` / `SE16N` | Data Browser — view table data |
| `ST05` | SQL Trace — performance analysis |
| `SM50` | Work Process Monitor |
| `SU01` | User Administration |
| `STMS` | Transport Management System |
| `SE09` / `SE10` | Transport Organizer |

---

## 4. System Landscape
*[Video: ~38:16 — 40:14]*

SAP projects always use at least **3 separate systems** (servers):

```
  DEVELOPER writes code
         │
         ▼
┌─────────────────┐
│   DEVELOPMENT   │  Unit testing by developer
│   (DEV / D00)   │  All custom objects created here
└────────┬────────┘
         │  Transport Request (TR) moved via STMS
         ▼
┌─────────────────┐
│     QUALITY     │  Integration & User Acceptance Testing (UAT)
│   (QAS / Q00)   │  Functional consultants & key users test here
└────────┬────────┘
         │  Transport Request approved & moved
         ▼
┌─────────────────┐
│   PRODUCTION    │  Live business data — NEVER develop here
│   (PRD / P00)   │  Changes only via approved transports
└─────────────────┘
```

### Why this matters for developers

- You **always** develop in DEV. Never directly in PRD.
- Every change is tracked in a **Transport Request** (like a git commit + PR)
- Moving to PRD requires approval — a change management process
- A bug you introduce in DEV can be caught in QAS before it hits real business operations

> Think of it like: DEV = feature branch, QAS = staging, PRD = production

---

## 5. SAP Implementation Phases
*[Video: ~46:34]*

Standard SAP projects follow the **ASAP (Accelerated SAP)** methodology:

| Phase | Activity |
|---|---|
| **Project Preparation** | Scope, team, infrastructure setup |
| **Business Blueprinting** | Document AS-IS business processes, design TO-BE in SAP |
| **Realization** | Configure SAP, write custom ABAP, build reports |
| **Final Preparation** | Data migration, end-user training, cutover planning |
| **Go-Live & Support** | Production launch, hypercare support |

As an ABAP developer, you spend most time in **Realization** (building) and **Support** (fixing production issues).

---

## 6. ABAP Introduction
*[Video: ~01:07:40]*

**ABAP** = Advanced Business Application Programming

- SAP's proprietary language, created in the **1980s** (originally called ABAP/4 — the /4 stood for 4th generation)
- Classified as a **4th-generation language (4GL)** — higher abstraction than C/Java, with built-in DB access and report list processing as native language features
- Runs inside the SAP Application Server — compiled to SAP's own intermediate bytecode, executed by the ABAP runtime (not native machine code)
- Syntax is **English-like**, fully case-insensitive, and statement-terminated by `.` (period)
- Code written in 1990 still runs on modern S/4HANA — full backwards compatibility

### Why SAP Created Its Own Language

SAP needed a language that could:
1. **Seamlessly integrate with database access** — SQL is first-class syntax, not a library call
2. **Generate UI screens automatically** — selection screens and lists are declared, not manually built
3. **Handle multilingual output** — text element translations built into the language
4. **Enforce SAP-specific concepts** — clients, transport requests, authorizations

No existing language in the 1980s had all of these, so SAP built one.

### ABAP vs. Other Languages — Quick Orientation

| Concept | Python/Java | ABAP |
|---|---|---|
| Case sensitive | Yes | **No** — `DATA`, `data`, `Data` are identical |
| Statement end | `;` / newline | `.` period — forgetting it is the #1 syntax error |
| String delimiter | `"` or `'` | `'` single quotes ONLY — `"` is a comment character |
| Comment | `#` / `//` | `"` double quote — rest of line is a comment |
| Null/None | `null` / `None` | **Initial value** — no null pointer exceptions |
| Array/List | `[]` / `List<T>` | **Internal Table** — with SQL-like ops built in |
| Dict/Map | `{}` / `HashMap` | **Hashed Internal Table** with UNIQUE KEY |
| Struct/Record | `struct` / POJO | **Structure / Work Area** |
| SQL | ORM / JDBC | **Open SQL** (built into language syntax, not a library) |
| Boolean | `true`/`false` | `'X'` = true, `' '` (space) = false — no native bool |

### The No-Null Guarantee

ABAP has no null references. Every variable always has a valid initial value:
- Strings → empty string
- Numbers → `0`
- Characters → spaces `' '`
- Dates → `'00000000'`

This eliminates entire classes of NullPointerException bugs common in Java/C# code.

---

## 7. Data Dictionary (SE11)
*[Video: ~01:11:13]*

The **DDIC** is SAP's centralized metadata repository — a schema registry, type system, and UI label library all in one. It defines:
- Database table structures and field definitions
- Data types (Domains, Data Elements), constraints
- Search helps (F4 dropdowns), views, lock objects
- Reusable type definitions used across all programs

**Why a separate metadata layer?**
Imagine SAP has 10,000 programs, 500 screens, and 200 reports that all show a "Material Number" field. If you need to change the label from "Material Number" to "Product Code" — or extend the field length — you change it once in the Data Element in DDIC and every one of those programs, screens, and reports reflects the change automatically. Without DDIC, you would need to find and edit each of those 10,700 places individually.

This centralisation is what makes SAP enterprise-maintainable at scale.

### The DDIC Type Hierarchy

```
DOMAIN  ──────────────────────►  Technical: what type? what length? what values allowed?
   │
   └──► DATA ELEMENT  ────────►  Semantic: what does it mean? what labels to show on screen?
              │
              └──► TABLE FIELD / STRUCTURE FIELD / PROGRAM VARIABLE
```

**Example chain:**
```
Domain: CHAR18  (character, length 18)
   └──► Data Element: MATNR  (label: "Material Number", F4 search help, documentation)
              └──► Table MARA,  field MATNR
              └──► Table MARC,  field MATNR
              └──► Table EKPO,  field MATNR
              └──► Structure: BAPI_MATERIAL, field MATERIAL
              └──► ABAP variable: DATA lv_matnr TYPE matnr.
```

One Domain serves many Data Elements. One Data Element serves many fields. Change the Domain → all fields resize. Change the Data Element label → all screens update automatically.

### Object Types in SE11

| Object Type | What it is |
|---|---|
| Database Table | Physical table in the DB (transparent table) |
| View | Virtual table joining multiple tables |
| Data Type | Domain, Data Element, Structure, or Table Type |
| Type Group | Legacy shared type/constant pools |
| Domain | Technical type definition |
| Search Help | F4 popup value selection |
| Lock Object | Concurrency control for DB records |

---

## 8. Domains
*[Video: ~01:13:26]*

A **Domain** defines the **technical data type** of a field.

**What a Domain specifies:**
- Data type (CHAR, NUMC, DATS, CURR, INT, etc.)
- Length
- Decimal places (for numeric types)
- Allowed values (value table or fixed values list)
- Case sensitivity, sign allowed, etc.

**Creating a Domain in SE11:**
1. SE11 → select "Domain" → enter name (e.g., `ZSTATUS`) → Create
2. Set: Data type = `CHAR`, Length = `1`
3. Go to **Value Range** tab → add Fixed Values:
   - `O` = Open
   - `C` = Closed
   - `H` = On Hold
4. Activate (Ctrl+F3)

> After activation, any field using this domain will automatically have a dropdown showing only O, C, H as valid inputs.

**Common SAP Data Types in Domains:**

| Type | Description | Example |
|---|---|---|
| `CHAR` | Character string | Names, codes |
| `NUMC` | Numeric characters (no arithmetic) | Order numbers, zip codes |
| `DATS` | Date (stored as YYYYMMDD) | `20240315` |
| `TIMS` | Time (stored as HHMMSS) | `143022` |
| `DEC` | Decimal number | Quantities |
| `CURR` | Currency amount (links to currency key) | Prices |
| `CUKY` | Currency key | `USD`, `EUR` |
| `QUAN` | Quantity (links to unit of measure) | Stock levels |
| `UNIT` | Unit of measure | `KG`, `EA` |
| `INT4` | 4-byte integer | Counters |
| `CLNT` | Client field (always 3 chars) | `100`, `300` |

---

## 9. Data Elements
*[Video: ~01:16:00]*

A **Data Element** adds **business meaning** on top of a Domain.

**What a Data Element adds:**
- Field labels (short, medium, long, heading) — shown on screens automatically
- Documentation / help text (F1 help)
- Search help assignment
- Parameter ID (for passing values between transactions)

**Creating a Data Element in SE11:**
1. SE11 → "Data type" → enter name (e.g., `ZORDER_STATUS`) → Create → Data Element
2. **Type** tab: assign the Domain (`ZSTATUS`)
3. **Field Label** tab: fill in
   - Short: `Sts`
   - Medium: `Status`
   - Long: `Order Status`
   - Heading: `Order Status`
4. Activate

**Key distinction:**

| Domain | Data Element |
|---|---|
| *Technical* — how is data stored? | *Semantic* — what does it mean? |
| `CHAR 1` | "Order Status" with label "Status" |
| Reused across many data elements | Reused across many table fields |

One Domain can power many Data Elements. Example: Domain `CHAR1` is used by data elements for Status, Gender, Flag, Priority, etc. — each with different labels and documentation.

---

## 10. Structures
*[Video: ~01:28:03]*

A **Structure** is a DDIC object that groups multiple fields together — but has **no physical database table** behind it.

**Uses of Structures:**
- Define the shape of a work area / internal table row in ABAP
- Group related fields for passing between programs, function modules, BAPIs
- Screen field groups

**Creating a Structure in SE11:**
1. SE11 → "Data type" → enter name (e.g., `ZORDER_HEADER`) → Create → Structure
2. Add fields using Data Elements:

| Component | Typing Method | Associated Type |
|---|---|---|
| ORDER_ID | Data element | ZORDER_ID |
| CUSTOMER | Data element | KUNNR |
| ORDER_DATE | Data element | DATS |
| AMOUNT | Data element | WERTV8 |
| STATUS | Data element | ZORDER_STATUS |

3. Activate

**Using a structure in ABAP:**
```abap
DATA: ls_header  TYPE zorder_header.      " single row
DATA: lt_headers TYPE TABLE OF zorder_header.  " multiple rows

ls_header-order_id = 'ORD001'.
ls_header-customer = '1000'.
APPEND ls_header TO lt_headers.
```

> A Structure is like a class with only public fields and no methods. It's purely a data container definition.

---

## 11. Database Tables
*[Video: ~02:01:16]*

A **Transparent Table** in DDIC = a real physical table in the database, 1:1 mapped.

When you activate a transparent table in DDIC, SAP executes `CREATE TABLE` on the underlying database. The DDIC definition IS the schema.

### Client-Dependent vs. Client-Independent
*[Video: ~02:04:10]*

**Client-Dependent** (most tables):
- Has `MANDT` (Client) as the **first key field**
- Data is isolated per client — client 100's data is invisible to client 200
- Example: `KNA1` (Customer Master) — each client has its own customers

**Client-Independent**:
- No `MANDT` field
- Data is **shared across all clients** in the system
- Used for pure configuration / customizing data (e.g., currency codes `TCURC`)

### Creating a Custom Table in SE11
*[Video: ~02:01:16 — 02:10:12]*

**Step 1: Basic settings**
1. SE11 → "Database Table" → enter `ZORDERS` → Create
2. Short description: "Custom Orders Table"
3. **Delivery Class**: `A` (Application data — business data, client-specific)
4. **Data Browser/Table View**: "Display/Maintenance Allowed"

**Delivery Classes explained:**

| Class | Meaning |
|---|---|
| `A` | Application data (your custom business data) |
| `C` | Customizing data (config, client-specific) |
| `G` | Customizing, client-independent |
| `E` | System table (SAP internal) |
| `S` | System table (client-independent) |
| `W` | System table (transport required) |

**Step 2: Fields**

| Field | Key | Data Element | Description |
|---|---|---|---|
| MANDT | ✓ | MANDT | Client |
| ORDER_ID | ✓ | CHAR10 | Order ID (part of primary key) |
| CUSTOMER | | KUNNR | Customer Number |
| ORDER_DATE | | DATS | Order Date |
| AMOUNT | | WERTV8 | Net Amount |
| CURRENCY | | WAERS | Currency Key |
| STATUS | | ZORDER_STATUS | Custom status field |

> The primary key = MANDT + ORDER_ID. Together they uniquely identify every row.

**Step 3: Technical Settings**
- Data Class: `APPL0` (Master data, small — for custom tables)
- Size Category: `0` (0–800 entries expected)
- These affect how the DB allocates physical storage.

**Step 4: Activate** — SAP runs DDL on the database.

---

## 12. Table Maintenance (SM30)
*[Video: ~02:19:22]*

**SM30** provides a generated UI to add/edit/delete rows in a custom table — like a basic admin CRUD interface, auto-generated by SAP.

### Enabling Table Maintenance

In SE11 on your table:
1. Go to **Utilities → Table Maintenance Generator**
2. Set: Authorization Group, Function Group (e.g., `ZORD`), Maintenance Type: One step
3. Click Generate → creates two Function Modules (`ZORD_*`) and screens

### Using SM30

1. T-Code `SM30` → enter table name `ZORDERS` → Maintain
2. You get a generated list screen showing all rows
3. You can: New Entry, Copy, Delete, Save
4. Changes are saved directly to the table

> SM30 is mainly used by **functional consultants** to manage configuration/customizing tables. Developers use SE16/SE16N to browse data.

---

## 13. Packages & Transport Requests
*[Video: ~02:44:14]*

### Packages

Every ABAP development object (table, program, function, class) must be assigned to a **Package**.

- Packages are like **namespaces + deployment units**
- They group related objects together
- They determine whether objects are **local** (never transported) or part of a **transport layer**

| Package type | Use |
|---|---|
| `$TMP` | Local, never transported — use only for throwaway tests |
| Custom package (e.g., `ZORDERS`) | Will be transported to QAS and PRD |

> **Never build real deliverables in `$TMP`** — they stay only in DEV forever.

### Transport Requests

When you create or change a DDIC object or program, SAP prompts you to assign it to a **Transport Request (TR)**.

A Transport Request is like a **bundle of changes** (think: a git commit or a release package) that travels through the landscape.

```
Transport Request TR: DEV-K900123
  ├── ZORDERS (Database Table)
  ├── ZORDER_STATUS (Domain)
  ├── ZORDER_STATUS_DE (Data Element)
  └── ZORDER_HEADER (Structure)
```

**TR workflow:**
```
Developer creates/edits objects
       │ assigned to TR
       ▼
TR is "Released" in DEV (SE09/SE10)
       │
       ▼
Basis team imports TR into QAS (STMS)
       │
       ▼
Testing passes → TR imported into PRD (STMS)
```

**T-Codes for transports:**

| T-Code | Purpose |
|---|---|
| `SE09` | View/release your own transport requests |
| `SE10` | View all transport requests |
| `STMS` | Transport Management System (Basis team uses this to import) |

> **Rule**: One TR per feature/task, ideally. Don't mix unrelated changes in one TR — if one change fails QAS review, the whole TR is blocked.

---

## 14. Key Takeaways & Revision Points

### Architecture
- SAP = 3 tiers: Presentation (GUI) → Application (ABAP runtime) → Database
- Open SQL abstracts the DB so ABAP code is DB-vendor agnostic
- Always: DEV → QAS → PRD. Never develop directly in production.

### DDIC Hierarchy
```
Domain (technical type)
  → Data Element (business label + meaning)
    → Table Field / Structure Component
      → ABAP Variable (TYPE data_element_name)
```

### The MANDT Field
- Client field — first key field in almost every table
- Makes all data client-isolated
- SAP Open SQL **auto-filters by MANDT** — you never add it to WHERE clauses

### Creating Custom Objects — the Order
When building from scratch, always follow this order:
1. Create **Domain** (if no suitable standard domain exists)
2. Create **Data Element** referencing the Domain
3. Create **Structure** (if you need a reusable row shape)
4. Create **Database Table** using the Data Elements
5. Generate **Table Maintenance** (SM30) if config data
6. Assign all to a **Package** (not `$TMP`) and a **Transport Request**

### Quick Revision Questions
1. What are the 3 tiers of SAP architecture and what runs in each?
2. What is the difference between a Domain and a Data Element? Give a concrete example of each.
3. What does activating a Transparent Table in DDIC do at the DB level?
4. Why is `MANDT` always the first key field in SAP tables? What does "client-dependent" mean?
5. What is a Transport Request and why is assigning objects to `$TMP` a problem for real projects?
6. What is the difference between client-dependent and client-independent tables? Give one example of each.
7. What T-Code do you use to: browse table data / write and edit ABAP programs / create DDIC objects / manage transports?
8. In the SAP landscape, what must happen between DEV and PRD — and who is responsible for each step?
9. Why did SAP create ABAP instead of using an existing language?
10. What is the no-null guarantee in ABAP and what problem does it eliminate?
11. What is the Where-Used impact of changing a Domain that is referenced by 50 Data Elements?
12. When creating a custom table, in what order do you create the DDIC objects (Domain → Data Element → Table)?

---

*Notes based on ZAP Yard — SAP ABAP Video 1 of 11*
*Next: SAP_ABAP_Video2_StudyNotes.md*
*Reference: ABAP_DDIC_Tutorial.md for code examples, syntax patterns, and full cheat sheet*
