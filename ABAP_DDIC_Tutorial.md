# ABAP & DDIC Tutorial: A First-Principles Guide for Experienced Programmers

> You know how to code. This guide explains **why SAP works the way it does**, not just how to use it.

---

## Table of Contents

1. [The Mental Model — What is SAP/ERP?](#1-the-mental-model)
2. [ABAP: The Language](#2-abap-the-language)
3. [DDIC — The Data Dictionary](#3-ddic-the-data-dictionary)
4. [Classic Reports (ALV & Write)](#4-classic-reports)
5. [Internal Tables & Work Areas](#5-internal-tables--work-areas)
6. [SELECT & Open SQL](#6-select--open-sql)
7. [Modularization: Forms, Functions, Classes](#7-modularization)
8. [Program Flow: Events](#8-program-flow-events)
9. [Putting It All Together — Full Report Example](#9-full-example)
10. [Cheat Sheet](#10-cheat-sheet)

---

## 1. The Mental Model

### What is ERP? (First Principles)

ERP (Enterprise Resource Planning) is a **single shared database** for an entire company. Instead of:

```
HR system   ──┐
Finance sys ──┤  All separate DBs, all out of sync
Warehouse   ──┘
```

SAP gives you:

```
HR + Finance + Warehouse + Sales + Procurement
         ──────────────────────────
              ONE shared database
```

This means:
- When a sale is posted, inventory decrements and a financial entry posts **simultaneously**
- Every department sees the **same data** in real time
- No integration layers, no ETL jobs between systems

### What is SAP specifically?

SAP (Systems, Applications & Products) is the dominant ERP vendor. Their flagship product is **SAP ECC** (older) and **SAP S/4HANA** (modern, in-memory). Think of SAP as a massive, pre-built application with ~100,000 customizable business processes already coded.

### Where does ABAP fit?

```
Business Process  →  SAP Standard Program (written in ABAP)
                       ↕ reads/writes
                   DDIC (Data Dictionary)
                       ↕ maps to
                   Database Tables (Oracle, HANA, MSSQL, etc.)
```

**ABAP** = SAP's proprietary language, like a hybrid of COBOL + SQL + OOP, born in the 1980s and evolved continuously since.

You write ABAP to:
- **Report** on business data (the main use case in this tutorial)
- **Enhance** standard SAP transactions
- **Build** custom business logic (BAPIs, exits, enhancements)
- **Interface** with external systems (IDocs, RFCs)

---

## 2. ABAP: The Language

### Key Differences vs. Languages You Know

| Concept | Java/Python/C# | ABAP |
|---|---|---|
| Case sensitivity | Yes | **No** — `data` = `DATA` = `Data` |
| Statement terminator | `;` or newline | `.` (period) |
| String delimiter | `"` or `'` | `'` single quotes only |
| Comment | `//` or `#` | `"` double quote |
| Null | `null` / `None` | initial value (empty string, 0, etc.) |
| Arrays | `List<T>`, `[]` | **Internal Tables** |
| Struct/Record | `struct`, `class` | **Structure** or **Work Area** |
| DB query | ORM / raw SQL | **Open SQL** (SAP's SQL abstraction) |

### Basic Syntax

```abap
" This is a comment
REPORT zmy_first_report.          " Program declaration

DATA: lv_name    TYPE string,     " Variable declaration
      lv_counter TYPE i.          " i = integer

lv_name    = 'Hello SAP World'.
lv_counter = 42.

WRITE: / lv_name,                 " / means new line
       / lv_counter.
```

### Naming Conventions (Hungarian-style, SAP enforced by culture)

| Prefix | Meaning | Example |
|---|---|---|
| `lv_` | Local Variable | `lv_name` |
| `gv_` | Global Variable | `gv_company_code` |
| `lt_` | Local Table (internal table) | `lt_orders` |
| `gt_` | Global Table | `gt_materials` |
| `ls_` | Local Structure (single row) | `ls_order_line` |
| `gs_` | Global Structure | `gs_header` |
| `lc_` | Local Constant | `lc_max_rows` |
| `lo_` | Local Object (class instance) | `lo_alv` |

**Customer namespace**: All custom objects must start with `Y` or `Z` (e.g., `ZREPORT_SALES`, `ZCUSTOMER_TABLE`). SAP reserves everything else for standard objects.

### Data Types

```abap
DATA: lv_text    TYPE string,        " variable-length string
      lv_char    TYPE c LENGTH 10,   " fixed-length char
      lv_int     TYPE i,             " 4-byte integer
      lv_dec     TYPE p DECIMALS 2,  " packed decimal (financial)
      lv_float   TYPE f,             " floating point
      lv_date    TYPE d,             " date: YYYYMMDD
      lv_time    TYPE t,             " time: HHMMSS
      lv_bool    TYPE abap_bool.     " 'X' = true, ' ' = false
```

> **Why packed decimal for finance?** Floating point has rounding errors (0.1 + 0.2 ≠ 0.3). SAP uses `p` type for exact decimal arithmetic in financial calculations.

### Control Flow

```abap
" IF / ELSEIF / ELSE
IF lv_counter > 10.
  WRITE 'Big number'.
ELSEIF lv_counter = 0.
  WRITE 'Zero'.
ELSE.
  WRITE 'Small'.
ENDIF.

" CASE (switch)
CASE lv_status.
  WHEN 'A'. WRITE 'Active'.
  WHEN 'I'. WRITE 'Inactive'.
  WHEN OTHERS. WRITE 'Unknown'.
ENDCASE.

" DO loop (like for i in range(n))
DO 5 TIMES.
  WRITE sy-index.    " sy-index is the loop counter (system variable)
ENDDO.

" WHILE loop
WHILE lv_counter < 100.
  lv_counter = lv_counter + 1.
ENDWHILE.
```

### System Variables (sy-* fields)

SAP populates a global structure `SY` (also written `SYST`) automatically:

| Variable | Meaning |
|---|---|
| `sy-subrc` | Return code — **0 means success**, like errno |
| `sy-mandt` | Current client (SAP instance/tenant) |
| `sy-uname` | Logged-in username |
| `sy-datum` | Today's date |
| `sy-uzeit` | Current time |
| `sy-index` | Current loop iteration |
| `sy-tabix` | Current internal table row index |
| `sy-dbcnt` | Number of rows affected by last DB operation |

> `sy-subrc` is **the most important one**. After every DB call, file read, or function call, check it.

---

## 3. DDIC — The Data Dictionary

### What is the DDIC?

The DDIC is SAP's **metadata layer** — a centralized repository that defines:
- Database table structures
- Data types (Domains, Data Elements)
- Views, Search Helps, Lock Objects
- Type pools (shared type definitions)

Think of it as a **schema registry + type system** that everything in ABAP depends on.

### The DDIC Type Hierarchy

```
Domain
  └── Data Element
        └── Table Field / Structure Component / Program Variable
```

**Domain** = raw data type + value constraints
- e.g., "a 4-character field that can only contain values: 'EUR', 'USD', 'GBP'"

**Data Element** = semantic meaning layered on a Domain
- e.g., "Currency Code" — adds field labels, documentation, search help

**Table Field** = a column in a DB table, typed by a Data Element

This separation means:
- Change the label in the Data Element → all screens showing that field update automatically
- Change the domain length → all tables/structures using it resize together

### Transparent Tables (the main table type)

A **Transparent Table** in DDIC = a real physical DB table, 1:1 mapped.

```
DDIC Table: MARA (Material Master General Data)
  ├── MANDT  (Client)        TYPE MANDT
  ├── MATNR  (Material No.)  TYPE MATNR   ← Data Element
  ├── MTART  (Material Type) TYPE MTART
  ├── MBRSH  (Industry)      TYPE MBRSH
  └── ...hundreds more fields
```

Key standard SAP tables to know:

| Table | Description |
|---|---|
| `MARA` | Material Master (general) |
| `MARC` | Material Master (plant data) |
| `KNA1` | Customer Master |
| `LFA1` | Vendor Master |
| `VBAK` | Sales Order Header |
| `VBAP` | Sales Order Items |
| `EKKO` | Purchase Order Header |
| `EKPO` | Purchase Order Items |
| `BKPF` | Accounting Document Header |
| `BSEG` | Accounting Document Line Items |
| `T001` | Company Codes |

> **The MANDT field**: Almost every SAP table has `MANDT` (client) as the first key field. SAP is multi-tenant at the application layer — one database, multiple isolated "clients". ABAP's Open SQL automatically filters by the current client. **You never need to filter by MANDT in your WHERE clause** — SAP does it for you.

### Creating a Custom Table in DDIC (SE11)

Transaction **SE11** is where you work in the DDIC.

**Step-by-step: Create table `ZORDERS`**

1. SE11 → select "Database Table" → enter `ZORDERS` → Create
2. Short Description: "Custom Orders Table"
3. **Delivery Class**: `A` (Application table, stored in client)
4. **Data Browser/Table View Maintenance**: Display/Maintenance Allowed

**Fields tab:**

| Field | Key | Data Element | Type | Length | Description |
|---|---|---|---|---|---|
| MANDT | ✓ | MANDT | CLNT | 3 | Client |
| ORDER_ID | ✓ | CHAR10 | CHAR | 10 | Order ID |
| CUSTOMER | | KUNNR | CHAR | 10 | Customer No. |
| ORDER_DATE | | DATS | DATS | 8 | Order Date |
| AMOUNT | | WERTV8 | CURR | 13 | Amount |
| CURRENCY | | WAERS | CUKY | 5 | Currency |
| STATUS | | CHAR1 | CHAR | 1 | Status |

5. **Technical Settings**: Data Class `APPL0`, Size Category `0`
6. Activate (Ctrl+F3)

> When you activate, SAP runs `CREATE TABLE` on the actual database. DDIC activation = DDL execution.

### Structures vs. Tables

**Structure** = DDIC object that defines a row shape but has **no physical DB table**. Used for:
- Passing data between programs/functions
- Defining work areas for internal tables
- Screen field groups

```abap
" Reference a DDIC structure in ABAP:
DATA: ls_order TYPE zorders.          " Single row (work area)
DATA: lt_orders TYPE TABLE OF zorders." Multiple rows (internal table)
```

---

## 4. Classic Reports

### Two Styles of Reports

**1. Classic WRITE reports** — output formatted text to a scrollable list (SAP's equivalent of `printf` to a terminal)

**2. ALV Reports** (ABAP List Viewer) — output an interactive grid with sort, filter, export to Excel built in. **This is what you should use in practice.**

### The REPORT Statement

```abap
REPORT zmy_report
  LINE-SIZE 255        " page width
  LINE-COUNT 65        " lines per page
  NO STANDARD PAGE HEADING
  MESSAGE-ID zmessages.  " message class for error messages
```

### Selection Screen (the UI input form)

Before a report runs, SAP shows a **Selection Screen** — a generated form for user input. You declare it with:

```abap
" Single value input
PARAMETERS: p_bukrs TYPE bukrs,          " Company Code
            p_date  TYPE sy-datum.       " Date

" Range input (from/to + include/exclude)
SELECT-OPTIONS: s_matnr FOR mara-matnr,  " Material Number range
                s_mtart FOR mara-mtart.  " Material Type range
```

`SELECT-OPTIONS` creates a **ranges table** — a special internal table with columns:
- `SIGN`: `I` = Include, `E` = Exclude
- `OPTION`: `EQ`, `NE`, `BT` (between), `CP` (contains pattern), `GT`, `LT`, etc.
- `LOW`: lower value
- `HIGH`: upper value (for BT)

You use it in SQL like:
```abap
SELECT * FROM mara
  WHERE matnr IN s_matnr
  AND   mtart IN s_mtart.
```

### WRITE — Classic List Output

```abap
WRITE: / 'Hello World'.          " / = new line before output
WRITE: 'inline'.                 " no / = same line
WRITE: /5 'column 5'.            " /5 = new line, column 5
WRITE: lv_date USING EDIT MASK '__.__.____'.  " formatted date

ULINE.                           " draw a horizontal line
SKIP.                            " blank line
SKIP 3.                          " 3 blank lines

" Formatting
WRITE lv_amount CURRENCY lv_currency.   " locale-aware currency format
WRITE lv_date   DD/MM/YYYY.            " date format
```

---

## 5. Internal Tables & Work Areas

This is the **most important ABAP concept** for a programmer coming from other languages.

### The Core Idea

An **Internal Table** = an in-memory table (like a `List<struct>` in Java or `[]dict` in Python), but with SQL-like operations built into the language.

A **Work Area** = a single row of that table (like a pointer to one element).

```abap
" Define types
TYPES: BEGIN OF ty_order,
         order_id   TYPE char10,
         customer   TYPE kunnr,
         amount     TYPE wertv8,
         status     TYPE char1,
       END OF ty_order.

" Declare internal table and work area
DATA: lt_orders TYPE TABLE OF ty_order,  " the table
      ls_order  TYPE ty_order.           " one row
```

### Populating Internal Tables

```abap
" Method 1: Assign to work area, then append
ls_order-order_id  = 'ORD001'.
ls_order-customer  = 'CUST100'.
ls_order-amount    = 500.
ls_order-status    = 'O'.
APPEND ls_order TO lt_orders.

" Method 2: Inline value assignment (modern ABAP)
APPEND VALUE #( order_id = 'ORD002'
                customer = 'CUST200'
                amount   = 750 ) TO lt_orders.

" From DB select (most common — covered in next section)
SELECT * FROM vbak INTO TABLE lt_orders.
```

### Looping Over Internal Tables

```abap
" Classic loop with work area
LOOP AT lt_orders INTO ls_order.
  WRITE: / ls_order-order_id,
           ls_order-customer,
           ls_order-amount.
ENDLOOP.

" Loop with WHERE condition (filters in ABAP, not DB)
LOOP AT lt_orders INTO ls_order
  WHERE status = 'O'.
  " ... only open orders
ENDLOOP.

" Modern ABAP: loop with field symbol (reference, no copy)
FIELD-SYMBOLS: <ls_order> TYPE ty_order.
LOOP AT lt_orders ASSIGNING <ls_order>.
  <ls_order>-status = 'C'.    " modifies the table directly
ENDLOOP.
```

### Reading a Single Row

```abap
" Read by table key
READ TABLE lt_orders INTO ls_order
  WITH KEY order_id = 'ORD001'.

IF sy-subrc = 0.
  WRITE ls_order-amount.   " found
ELSE.
  WRITE 'Not found'.
ENDIF.
```

### Sorting

```abap
SORT lt_orders BY amount DESCENDING.
SORT lt_orders BY customer amount.    " multi-field sort
```

### Modifying & Deleting Rows

```abap
" Modify current row inside LOOP
LOOP AT lt_orders INTO ls_order.
  IF ls_order-amount > 1000.
    ls_order-status = 'H'.             " set to hold
    MODIFY lt_orders FROM ls_order.    " write back to table
  ENDIF.
ENDLOOP.

" Delete rows by condition
DELETE lt_orders WHERE status = 'C'.

" Delete duplicates (after sort)
SORT lt_orders BY customer.
DELETE ADJACENT DUPLICATES FROM lt_orders COMPARING customer.
```

### Table Types Deep Dive

| Type | Declaration | Access | Use when |
|---|---|---|---|
| **Standard** | `TYPE TABLE OF` | Linear scan or binary search | General purpose, append-heavy |
| **Sorted** | `TYPE SORTED TABLE OF` | Binary search always | Read-heavy, need ordered data |
| **Hashed** | `TYPE HASHED TABLE OF` | Hash lookup (O(1)) | Key-based lookup only |

```abap
" Hashed table — like a HashMap/dict
DATA: lt_lookup TYPE HASHED TABLE OF ty_order
      WITH UNIQUE KEY order_id.

" Sorted table
DATA: lt_sorted TYPE SORTED TABLE OF ty_order
      WITH NON-UNIQUE KEY customer.
```

---

## 6. SELECT & Open SQL

### Why Open SQL, not native SQL?

ABAP's Open SQL:
1. **Automatically adds MANDT** to every query
2. **Database-agnostic** — same code runs on Oracle, HANA, MSSQL, DB2
3. **Buffered** — SAP table buffers reduce DB round trips
4. Returns data into **ABAP types** directly

### Basic SELECT Patterns

```abap
" Select all rows into internal table
SELECT * FROM mara
  INTO TABLE lt_mara.

" Select specific fields (always prefer this — don't SELECT *)
SELECT matnr mtart mbrsh
  FROM mara
  INTO CORRESPONDING FIELDS OF TABLE lt_mara
  WHERE mtart = 'FERT'          " Finished goods
  AND   lvorm = space.          " Not flagged for deletion

" Select single row
SELECT SINGLE *
  FROM kna1
  INTO ls_customer
  WHERE kunnr = lv_customer_no.

IF sy-subrc <> 0.
  " customer not found
ENDIF.

" Select with range (from SELECT-OPTIONS)
SELECT matnr mtart
  FROM mara
  INTO TABLE lt_mara
  WHERE matnr IN s_matnr.

" Select with JOIN
SELECT v~vbeln v~erdat k~name1
  FROM vbak AS v
  INNER JOIN kna1 AS k ON k~kunnr = v~kunnr
  INTO TABLE lt_result
  WHERE v~auart = 'OR'.        " Standard Order type
```

### Performance Rules (Critical in SAP)

SAP runs on massive datasets. Bad SQL = slow reports that bring down production.

**Rule 1: Never SELECT inside a LOOP**

```abap
" ❌ BAD — N+1 problem, one DB call per order
LOOP AT lt_orders INTO ls_order.
  SELECT SINGLE * FROM kna1 INTO ls_customer
    WHERE kunnr = ls_order-customer.
ENDLOOP.

" ✅ GOOD — one DB call, then loop in memory
SELECT kunnr name1 FROM kna1
  INTO TABLE lt_customers
  FOR ALL ENTRIES IN lt_orders       " ← like SQL IN clause
  WHERE kunnr = lt_orders-customer.

LOOP AT lt_orders INTO ls_order.
  READ TABLE lt_customers INTO ls_customer
    WITH KEY kunnr = ls_order-customer.
ENDLOOP.
```

> `FOR ALL ENTRIES IN` generates an optimized IN-list query. **Caution**: the driving table (`lt_orders`) must not be empty — check first, else it selects ALL rows!

**Rule 2: Use WHERE clauses on indexed fields**

Every SAP table has a primary key (always includes MANDT). Other indexes exist too. Check SE11 → Indexes tab. Filter on indexed fields first.

**Rule 3: Never select more columns than you need**

`SELECT *` fetches all columns across the network. In wide tables (BSEG has 200+ fields), this is expensive.

**Rule 4: Aggregate in the database, not in ABAP**

```abap
" ✅ Let DB aggregate
SELECT matnr SUM( labst ) AS total_stock
  FROM mard
  INTO TABLE lt_stock
  GROUP BY matnr
  HAVING SUM( labst ) > 0.
```

---

## 7. Modularization

### FORM / ENDFORM (Subroutines — classic style)

```abap
REPORT zmy_report.

START-OF-SELECTION.
  PERFORM get_data.
  PERFORM display_data.

FORM get_data.
  SELECT * FROM mara INTO TABLE gt_mara
    WHERE mtart = 'FERT'.
ENDFORM.

FORM display_data.
  LOOP AT gt_mara INTO gs_mara.
    WRITE: / gs_mara-matnr.
  ENDLOOP.
ENDFORM.
```

Passing parameters to FORM:
```abap
FORM calculate_tax
  USING    iv_amount   TYPE p
           iv_rate     TYPE p
  CHANGING cv_tax      TYPE p.

  cv_tax = iv_amount * iv_rate / 100.

ENDFORM.

" Call it:
PERFORM calculate_tax
  USING    lv_price lv_tax_rate
  CHANGING lv_tax_amount.
```

### Function Modules (SAP's reusable library units)

Function Modules (FMs) live in **Function Groups** (like namespaced libraries). They're SAP's equivalent of a public API — most standard SAP functionality is exposed as FMs.

```abap
" Calling a standard SAP Function Module
CALL FUNCTION 'CONVERSION_EXIT_MATNR_INPUT'
  EXPORTING
    input         = lv_raw_matnr
  IMPORTING
    output        = lv_matnr
  EXCEPTIONS
    length_error  = 1
    OTHERS        = 2.

IF sy-subrc <> 0.
  " handle error
ENDIF.
```

Key standard FMs to know:

| Function Module | Purpose |
|---|---|
| `POPUP_TO_CONFIRM` | Show confirmation dialog |
| `REUSE_ALV_GRID_DISPLAY` | Display ALV grid (older method) |
| `CONVERSION_EXIT_*` | Format conversions (material no., etc.) |
| `HR_INFOTYPE_OPERATION` | HR data operations |
| `BAPI_*` | Business APIs (stable, remote-capable) |

### Classes & Methods (modern OOP ABAP)

```abap
CLASS zcl_order_processor DEFINITION.
  PUBLIC SECTION.
    METHODS: constructor
               IMPORTING iv_bukrs TYPE bukrs,
             process_orders
               RETURNING VALUE(rv_count) TYPE i.
  PRIVATE SECTION.
    DATA: mv_bukrs   TYPE bukrs,
          mt_orders  TYPE TABLE OF zorders.
ENDCLASS.

CLASS zcl_order_processor IMPLEMENTATION.
  METHOD constructor.
    mv_bukrs = iv_bukrs.
  ENDMETHOD.

  METHOD process_orders.
    SELECT * FROM zorders INTO TABLE mt_orders
      WHERE status = 'O'.
    rv_count = lines( mt_orders ).    " lines() = array length
  ENDMETHOD.
ENDCLASS.

" Usage:
DATA(lo_processor) = NEW zcl_order_processor( iv_bukrs = 'US01' ).
DATA(lv_count) = lo_processor->process_orders( ).
```

---

## 8. Program Flow: Events

ABAP reports are **event-driven**, not procedural top-to-bottom. The runtime calls your code at specific lifecycle points:

```
Program Load
     │
     ▼
INITIALIZATION          ← Set default values for selection screen
     │
     ▼
AT SELECTION-SCREEN     ← Validate input before execution
     │
     ▼
START-OF-SELECTION      ← Main execution (fetch data, process)
     │
     ▼
END-OF-SELECTION        ← Post-processing (usually for reports)
     │
     ▼
TOP-OF-PAGE             ← Runs before each new page in output
     │
     ▼
END-OF-PAGE             ← Runs at bottom of each page
```

```abap
REPORT zorder_report.

PARAMETERS: p_bukrs TYPE bukrs OBLIGATORY.  " Required field

INITIALIZATION.
  p_bukrs = '1000'.    " Default company code

AT SELECTION-SCREEN.
  " Validate: check company code exists
  SELECT SINGLE bukrs FROM t001 INTO @DATA(lv_bukrs)
    WHERE bukrs = p_bukrs.
  IF sy-subrc <> 0.
    MESSAGE 'Invalid Company Code' TYPE 'E'.  " E = Error, stops execution
  ENDIF.

START-OF-SELECTION.
  " Fetch and display data
  SELECT * FROM vbak INTO TABLE @DATA(lt_orders)
    WHERE bukrs = p_bukrs.

  LOOP AT lt_orders INTO DATA(ls_order).
    WRITE: / ls_order-vbeln, ls_order-erdat.
  ENDLOOP.
```

### Message Types

| Type | Effect |
|---|---|
| `'S'` | Success — status bar message, continues |
| `'I'` | Info — popup, user clicks OK, continues |
| `'W'` | Warning — status bar, continues |
| `'E'` | Error — status bar, **stops** at selection screen |
| `'A'` | Abort — popup, terminates program |
| `'X'` | Exit dump — short dump (like a crash), for development |

---

## 9. Full Example

### Complete ALV Report: Open Sales Orders by Company Code

```abap
*&---------------------------------------------------------------------*
*& Report: ZOPEN_ORDERS_REPORT
*& Description: Display open sales orders with customer name and value
*& Author: New ABAP Developer
*&---------------------------------------------------------------------*
REPORT zopen_orders_report.

*----------------------------------------------------------------------*
* Type Definitions
*----------------------------------------------------------------------*
TYPES: BEGIN OF ty_output,
         vbeln  TYPE vbeln_va,    " Sales Order Number
         erdat  TYPE erdat,       " Creation Date
         kunnr  TYPE kunnr,       " Customer Number
         name1  TYPE name1_gp,    " Customer Name
         netwr  TYPE netwr_ak,    " Net Value
         waerk  TYPE waerk,       " Currency
         auart  TYPE auart,       " Order Type
       END OF ty_output.

*----------------------------------------------------------------------*
* Global Data
*----------------------------------------------------------------------*
DATA: gt_output  TYPE TABLE OF ty_output,
      gs_output  TYPE ty_output.

*----------------------------------------------------------------------*
* Selection Screen
*----------------------------------------------------------------------*
SELECTION-SCREEN BEGIN OF BLOCK b1 WITH FRAME TITLE TEXT-001.
  PARAMETERS:     p_bukrs TYPE bukrs OBLIGATORY DEFAULT '1000'.
  SELECT-OPTIONS: s_erdat FOR vbak-erdat,           " Date range
                  s_kunnr FOR vbak-kunnr,           " Customer range
                  s_auart FOR vbak-auart.            " Order type range
SELECTION-SCREEN END OF BLOCK b1.

*----------------------------------------------------------------------*
* Initialization: Set smart defaults
*----------------------------------------------------------------------*
INITIALIZATION.
  " Default: last 30 days
  s_erdat-sign   = 'I'.
  s_erdat-option = 'BT'.
  s_erdat-low    = sy-datum - 30.
  s_erdat-high   = sy-datum.
  APPEND s_erdat.

*----------------------------------------------------------------------*
* Input Validation
*----------------------------------------------------------------------*
AT SELECTION-SCREEN.
  SELECT SINGLE bukrs FROM t001 INTO @DATA(lv_check)
    WHERE bukrs = p_bukrs.
  IF sy-subrc <> 0.
    MESSAGE e001(00) WITH 'Company code' p_bukrs 'does not exist'.
  ENDIF.

*----------------------------------------------------------------------*
* Main Processing
*----------------------------------------------------------------------*
START-OF-SELECTION.

  " Step 1: Get sales order headers
  SELECT vbeln erdat kunnr netwr waerk auart
    FROM vbak
    INTO TABLE @DATA(lt_orders)
    WHERE bukrs = p_bukrs
    AND   erdat IN s_erdat
    AND   kunnr IN s_kunnr
    AND   auart IN s_auart
    AND   rfgsd = space.          " Not yet billed (example filter)

  IF lt_orders IS INITIAL.
    MESSAGE 'No orders found for selection' TYPE 'S'
            DISPLAY LIKE 'W'.
    RETURN.
  ENDIF.

  " Step 2: Get customer names in one shot
  SELECT kunnr name1
    FROM kna1
    INTO TABLE @DATA(lt_customers)
    FOR ALL ENTRIES IN lt_orders
    WHERE kunnr = lt_orders-kunnr.

  " Step 3: Build output table
  LOOP AT lt_orders INTO DATA(ls_order).

    CLEAR gs_output.
    gs_output-vbeln = ls_order-vbeln.
    gs_output-erdat = ls_order-erdat.
    gs_output-kunnr = ls_order-kunnr.
    gs_output-netwr = ls_order-netwr.
    gs_output-waerk = ls_order-waerk.
    gs_output-auart = ls_order-auart.

    " Lookup customer name
    READ TABLE lt_customers INTO DATA(ls_cust)
      WITH KEY kunnr = ls_order-kunnr.
    IF sy-subrc = 0.
      gs_output-name1 = ls_cust-name1.
    ENDIF.

    APPEND gs_output TO gt_output.

  ENDLOOP.

  " Step 4: Sort by date descending
  SORT gt_output BY erdat DESCENDING.

*----------------------------------------------------------------------*
* Display with ALV
*----------------------------------------------------------------------*
END-OF-SELECTION.

  PERFORM display_alv.

*----------------------------------------------------------------------*
* Form: Display ALV Grid
*----------------------------------------------------------------------*
FORM display_alv.

  " Build field catalog (column definitions)
  DATA: lt_fcat TYPE slis_t_fieldcat_alv,
        ls_fcat TYPE slis_fieldcat_alv.

  " Helper macro-like pattern for field catalog
  DEFINE add_field.
    CLEAR ls_fcat.
    ls_fcat-fieldname  = &1.   " ABAP field name
    ls_fcat-seltext_m  = &2.   " Column header
    ls_fcat-outputlen  = &3.   " Column width
    APPEND ls_fcat TO lt_fcat.
  END-OF-DEFINITION.

  add_field 'VBELN' 'Order No.'    10.
  add_field 'ERDAT' 'Created'      10.
  add_field 'KUNNR' 'Customer'     10.
  add_field 'NAME1' 'Name'         30.
  add_field 'NETWR' 'Net Value'    13.
  add_field 'WAERK' 'Currency'      5.
  add_field 'AUART' 'Order Type'    4.

  " Layout options
  DATA: ls_layout TYPE slis_layout_alv.
  ls_layout-zebra        = 'X'.    " alternating row colors
  ls_layout-colwidth_optimize = 'X'.

  " Display
  CALL FUNCTION 'REUSE_ALV_GRID_DISPLAY'
    EXPORTING
      it_fieldcat = lt_fcat
      is_layout   = ls_layout
      i_save      = 'A'            " save layout variants
    TABLES
      t_outtab    = gt_output
    EXCEPTIONS
      OTHERS      = 1.

  IF sy-subrc <> 0.
    MESSAGE 'ALV display error' TYPE 'E'.
  ENDIF.

ENDFORM.
```

---

## 10. Cheat Sheet

### Quick Reference Card

```
PROGRAM STRUCTURE
─────────────────
REPORT zname.              Program header
PARAMETERS: p_x TYPE t.    Single input
SELECT-OPTIONS: s_x FOR table-field.  Range input
INITIALIZATION.            Set defaults
AT SELECTION-SCREEN.       Validate input
START-OF-SELECTION.        Main code
END-OF-SELECTION.          Post-processing


DATA DECLARATION
────────────────
DATA: lv_var  TYPE typename.
DATA: ls_row  TYPE ddic_structure.
DATA: lt_tab  TYPE TABLE OF typename.
DATA: lt_tab  TYPE TABLE OF ddic_table.
CONSTANTS: lc_x TYPE t VALUE 'val'.


INTERNAL TABLE OPERATIONS
─────────────────────────
APPEND ls_row TO lt_tab.
INSERT ls_row INTO TABLE lt_tab.  " for sorted/hashed
LOOP AT lt_tab INTO ls_row [WHERE field = val].
  MODIFY lt_tab FROM ls_row.
  DELETE lt_tab.                  " delete current row
ENDLOOP.
READ TABLE lt_tab INTO ls_row WITH KEY field = val.
SORT lt_tab BY field1 [DESCENDING] field2.
DELETE lt_tab WHERE field = val.
DELETE ADJACENT DUPLICATES FROM lt_tab COMPARING field.
DESCRIBE TABLE lt_tab LINES lv_count.
" OR:
lv_count = lines( lt_tab ).


OPEN SQL
────────
SELECT [SINGLE] [*|f1 f2] FROM table
  INTO [TABLE] target
  WHERE field = val
  AND   field IN range
  FOR ALL ENTRIES IN lt_driver WHERE field = lt_driver-field.
  GROUP BY f1 HAVING condition.
  ORDER BY f1 [DESCENDING].

INSERT zmy_table FROM ls_row.
UPDATE zmy_table SET field = val WHERE key = kval.
DELETE zmy_table WHERE key = kval.
MODIFY zmy_table FROM ls_row.     " insert or update


CONTROL FLOW
────────────
IF cond. ELSEIF cond. ELSE. ENDIF.
CASE var. WHEN val. WHEN OTHERS. ENDCASE.
DO n TIMES. ENDDO.
WHILE cond. ENDWHILE.
LOOP AT lt INTO ls. ENDLOOP.
CHECK condition.    " like continue if NOT condition
EXIT.               " break out of loop or end program
RETURN.             " end current subroutine/method


STRING OPERATIONS
─────────────────
CONCATENATE s1 s2 INTO lv_result SEPARATED BY ' '.
SPLIT lv_str AT ',' INTO s1 s2 s3.
CONDENSE lv_str [NO-GAPS].
TRANSLATE lv_str TO UPPER CASE.
REPLACE 'old' WITH 'new' INTO lv_str.
SEARCH lv_str FOR 'pattern'.
strlen( lv_str ).    " string length function


IMPORTANT TRANSACTIONS
──────────────────────
SE11   DDIC (tables, structures, domains)
SE38   ABAP Editor (write/run programs)
SE80   Object Navigator (IDE-like browser)
SE37   Function Module editor
SE24   Class editor
SM30   Table maintenance (view/edit data)
SE16   Data Browser (view table contents)
ST05   SQL Trace (performance analysis)
ST12   ABAP Trace
/nSE38 Force-navigate to SE38 (/ = new session)
```

---

## What's Next?

Having mastered these basics, the natural progression is:

1. **Enhancements** — BADIs (Business Add-Ins), User Exits, Implicit/Explicit Enhancements to modify standard SAP without changing SAP code
2. **BAPIs** — Business APIs for transactional data updates (create orders, post goods movements)
3. **IDocs** — SAP's EDI/messaging format for system-to-system integration
4. **Smartforms / Adobe Forms** — SAP's document printing (invoices, delivery notes)
5. **Web Dynpro / Fiori** — SAP's web UI frameworks
6. **CDS Views** — HANA-native views replacing classic DDIC views (the future of SAP data modeling)
7. **OData / REST APIs** — Exposing SAP data to external systems

---

*Tutorial version 1.0 — Written for experienced programmers new to SAP/ABAP*
