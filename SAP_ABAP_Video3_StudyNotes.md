# SAP ABAP Study Notes — Video 3 of 11
> Based on: "FREE Video 3 of 11 - Learn SAP ABAP for Free for Freshers & Functional Consultants" by ZAP Yard
> Video: https://www.youtube.com/watch?v=n2gQmeCV-Vk
> Cross-reference: SAP_ABAP_Video1_StudyNotes.md | SAP_ABAP_Video2_StudyNotes.md | ABAP_DDIC_Tutorial.md

---

## Table of Contents

1. [Data Declarations & Work Areas — Revisited with Depth](#1-data-declarations--work-areas)
2. [The MARA Table — SAP Material Master](#2-the-mara-table--sap-material-master)
3. [Fetching Data with SELECT — Patterns & Best Practices](#3-fetching-data-with-select)
4. [Filtering with WHERE Clauses](#4-filtering-with-where-clauses)
5. [Internal Tables — Architecture & Operations](#5-internal-tables--architecture--operations)
6. [DESCRIBE TABLE — Counting Rows](#6-describe-table--counting-rows)
7. [Joining Database Tables in ABAP](#7-joining-database-tables-in-abap)
8. [The MAKT Table — Material Descriptions](#8-the-makt-table--material-descriptions)
9. [Custom Internal Tables for JOIN Results](#9-custom-internal-tables-for-join-results)
10. [Modularizing Code with INCLUDE Programs](#10-modularizing-code-with-include-programs)
11. [The TOP Include Convention](#11-the-top-include-convention)
12. [Debugging ABAP Programs](#12-debugging-abap-programs)
13. [Breakpoint Types — Session vs External](#13-breakpoint-types--session-vs-external)
14. [The ABAP Debugger — Navigation & Inspection](#14-the-abap-debugger--navigation--inspection)
15. [Passing Data Between Programs — ABAP Memory](#15-passing-data-between-programs--abap-memory)
16. [Passing Data Across Sessions — SAP Memory](#16-passing-data-across-sessions--sap-memory)
17. [ABAP Memory vs SAP Memory — Side-by-Side Comparison](#17-abap-memory-vs-sap-memory--side-by-side-comparison)
18. [Key Takeaways & Revision Questions](#18-key-takeaways--revision-questions)

---

## 1. Data Declarations & Work Areas
*[Video: ~15:26 — 21:21]*

### Context: Where We Are in the Learning Journey

By Video 3, you know how to:
- Create DDIC objects (Domains, Data Elements, Tables, Structures)
- Write a basic ABAP program in SE38
- Declare variables with DATA
- Build a selection screen with PARAMETERS and SELECT-OPTIONS
- Understand the event lifecycle (INITIALIZATION → AT SELECTION-SCREEN → START-OF-SELECTION)

Video 3 goes deeper on **data retrieval patterns** — how to get data out of the database efficiently, how to hold it in memory, and how to manage that memory across programs. These are the core skills of writing any real ABAP report or data processing program.

### Work Area Declaration Recap — With Material Master Context

A **work area** (also called a structure variable or header area) holds exactly one row of data at a time. It is a variable whose type matches a database table's row structure.

```abap
" Option 1: Use the DDIC table as the type directly
DATA ls_material TYPE mara.

" Option 2: Use LIKE to copy type from a table row
DATA ls_material LIKE mara.

" Option 3: Define your own structure type, then declare the work area
TYPES: BEGIN OF ty_s_material,
         matnr TYPE mara-matnr,    " material number
         mtart TYPE mara-mtart,    " material type
         mbrsh TYPE mara-mbrsh,    " industry sector
         meins TYPE mara-meins,    " base unit of measure
       END OF ty_s_material.

DATA ls_mat TYPE ty_s_material.
```

**When to use each approach:**
- `TYPE mara` — when you need all fields of the table (use sparingly — wide tables waste memory)
- Custom structure type — when you only need specific fields (preferred for performance)
- `LIKE mara` — legacy style, equivalent to `TYPE mara`, but `TYPE` is the modern standard

### PARAMETERS Declarations for Material Reports

```abap
PARAMETERS: p_matnr TYPE mara-matnr,     " Material number input — gets MATNR type + search help automatically
            p_mtart TYPE mara-mtart,     " Material type filter
            p_werks TYPE marc-werks.     " Plant (used when joining to plant-level tables)
```

By typing a PARAMETER to a DDIC table field (`TYPE mara-matnr`), SAP automatically:
1. Applies the field's data type and length
2. Attaches the standard F4 search help for that field
3. Uses the Data Element's label as the field's screen label

This is why the DDIC hierarchy matters in practice — you inherit search helps and labels for free.

---

## 2. The MARA Table — SAP Material Master
*[Video: ~20:26]*

### What Is the Material Master?

The Material Master is one of the most important data objects in SAP. It contains all information about a material (a product, raw material, semi-finished good, finished good, etc.) that a company procures, produces, or sells.

Because different departments need different information about the same material, the Material Master is split across **multiple tables** — each table holds a "view" for a different department:

| Table | View | Contains |
|---|---|---|
| `MARA` | General Data | Material number, type, weight, dimensions, unit of measure |
| `MARC` | Plant Data | MRP settings, production data per plant |
| `MARD` | Storage Location | Stock quantities per plant/storage location |
| `MAKT` | Descriptions | Material descriptions in multiple languages |
| `MVKE` | Sales Data | Sales organization, distribution channel pricing |
| `MARM` | Units of Measure | Alternative units (pallet, carton, etc.) |
| `MBEW` | Accounting | Valuation data, standard price, moving average price |

**MARA is the central/header table** — every material has exactly one MARA record (per client). The key is `MANDT + MATNR`.

### Key MARA Fields

| Field | Description | Type | Example |
|---|---|---|---|
| `MATNR` | Material Number | CHAR 18 | `'000000000000001234'` (leading zero padded) |
| `MTART` | Material Type | CHAR 4 | `'FERT'` = Finished, `'ROH'` = Raw, `'HALB'` = Semi-finished |
| `MBRSH` | Industry Sector | CHAR 1 | `'M'` = Mechanical Engineering, `'A'` = Plant Engineering |
| `MEINS` | Base Unit of Measure | UNIT 3 | `'EA'` = Each, `'KG'` = Kilogram, `'L'` = Litre |
| `MATKL` | Material Group | CHAR 9 | Grouping code for procurement/reporting |
| `NTGEW` | Net Weight | QUAN | Weight without packaging |
| `GEWEI` | Weight Unit | UNIT 3 | `'KG'`, `'LB'` |
| `LVORM` | Deletion Flag | CHAR 1 | `'X'` = flagged for deletion |
| `ERSDA` | Creation Date | DATS | When the material was first created |

### Material Number Formatting

Material numbers in SAP are stored internally as 18-character, left-zero-padded strings. So material `'1234'` is stored as `'000000000000001234'`. This is why the `MATNR` field uses type `NUMC 18` in the domain (numeric characters, zero-padded).

When displaying material numbers to users, SAP uses a **conversion exit** (`MATNR`) to strip leading zeros: `'000000000000001234'` displays as `'1234'`. When reading from user input, the conversion exit re-pads it.

This matters when you write reports: if you filter `WHERE matnr = '1234'`, it won't find anything because the DB stores `'000000000000001234'`. You must convert first:

```abap
" Convert user-entered material number to internal format
CALL FUNCTION 'CONVERSION_EXIT_MATNR_INPUT'
  EXPORTING input  = p_matnr
  IMPORTING output = lv_matnr_internal.

SELECT SINGLE * FROM mara INTO ls_material
  WHERE matnr = lv_matnr_internal.
```

---

## 3. Fetching Data with SELECT — Patterns & Best Practices
*[Video: ~20:26]*

### The SELECT Statement — Full Anatomy

```abap
SELECT  <field list or *>
  FROM  <table name>
  INTO  <target> [TABLE <internal table> | CORRESPONDING FIELDS OF <work area>]
  [WHERE <conditions>]
  [AND <more conditions>]
  [GROUP BY <fields>]
  [HAVING <aggregate conditions>]
  [ORDER BY <fields> [ASCENDING|DESCENDING]].
```

Every part is optional except `SELECT`, `FROM`, and `INTO`.

### Pattern 1: SELECT SINGLE into a Work Area

Use when you know the primary key and expect exactly one row.

```abap
DATA ls_material TYPE mara.

SELECT SINGLE *
  FROM mara
  INTO ls_material
  WHERE matnr = lv_matnr.

IF sy-subrc = 0.
  WRITE: / 'Material Type:', ls_material-mtart.
  WRITE: / 'Base UoM:     ', ls_material-meins.
ELSE.
  MESSAGE 'Material not found' TYPE 'I'.
ENDIF.
```

After `SELECT SINGLE`:
- `sy-subrc = 0` → row found, `ls_material` is populated
- `sy-subrc = 4` → no row matched the WHERE condition, `ls_material` is at initial values

### Pattern 2: SELECT * INTO TABLE — Bulk Fetch

Use when you expect multiple rows. Fetches all matching rows into an internal table in one database round-trip.

```abap
DATA lt_materials TYPE TABLE OF mara.

SELECT *
  FROM mara
  INTO TABLE lt_materials
  WHERE mtart = 'FERT'.

WRITE: / sy-dbcnt, 'finished goods found'.
```

After execution:
- `lt_materials` contains all FERT materials as rows
- `sy-dbcnt` = number of rows fetched
- `sy-subrc = 0` if at least one row was found, `= 4` if no rows

### Pattern 3: SELECT with Field List — Selective Columns

This is the **preferred pattern** for performance. Never use `SELECT *` in production code for wide tables — you waste network bandwidth and memory fetching columns you never use.

```abap
" Define a structure with only the fields you need
TYPES: BEGIN OF ty_s_mat_info,
         matnr TYPE mara-matnr,
         mtart TYPE mara-mtart,
         meins TYPE mara-meins,
         matkl TYPE mara-matkl,
       END OF ty_s_mat_info.

DATA: lt_mat_info TYPE TABLE OF ty_s_mat_info,
      ls_mat_info TYPE ty_s_mat_info.

SELECT matnr mtart meins matkl
  FROM mara
  INTO CORRESPONDING FIELDS OF TABLE lt_mat_info
  WHERE mtart IN s_mtart
  AND   lvorm = space.     " exclude deletion-flagged materials
```

`INTO CORRESPONDING FIELDS OF TABLE` maps columns by field name. The internal table's structure can have more or fewer fields than you selected — unmatched fields stay at initial value.

### Pattern 4: SELECT with ORDER BY

```abap
SELECT matnr mtart meins
  FROM mara
  INTO CORRESPONDING FIELDS OF TABLE lt_mat_info
  WHERE mtart = 'FERT'
  ORDER BY matnr ASCENDING.
```

> **Performance note**: Prefer sorting in ABAP (`SORT lt_table BY field.`) over `ORDER BY` in SQL unless the table is indexed on the sort field. `ORDER BY` forces a sort on the DB side; ABAP `SORT` is done in the application server's memory and is often faster for large result sets already fetched.

### Pattern 5: SELECT ENDSELECT — Row-by-Row Processing (Legacy)

Older ABAP code often uses this pattern — an implicit cursor loop. The runtime fetches one row at a time.

```abap
SELECT * FROM mara INTO ls_material
  WHERE mtart = 'FERT'.

  " This body executes once per row
  WRITE: / ls_material-matnr, ls_material-meins.

ENDSELECT.

IF sy-subrc <> 0.
  WRITE 'No materials found'.
ENDIF.
```

> **Important**: `SELECT...ENDSELECT` is **discouraged in modern ABAP** because it keeps the database cursor open and sends one round-trip per row. Always prefer `SELECT INTO TABLE` to fetch all rows at once, then loop in ABAP memory. The only legitimate use of `SELECT...ENDSELECT` today is when processing billions of rows that cannot fit in memory.

---

## 4. Filtering with WHERE Clauses
*[Video: ~30:20]*

### WHERE Clause Operators

```abap
SELECT * FROM mara INTO TABLE lt_mats
  WHERE mtart  =  'FERT'            " equals
  AND   meins  <> 'EA'              " not equals (also: NE)
  AND   ntgew  >  '0.000'           " greater than
  AND   ntgew  <= '100.000'         " less than or equal
  AND   matnr  IN s_matnr           " in ranges table (SELECT-OPTIONS)
  AND   matkl  BETWEEN 'A' AND 'Z'  " between two values (inclusive)
  AND   mtart  IN ('FERT', 'HALB')  " in a list of values
  AND   lvorm  =  space             " not deletion-flagged (space = initial value)
  AND   matnr  LIKE '000000001%'.   " pattern match — % is wildcard
```

### The `space` Keyword

`space` is an ABAP constant equal to `' '` (a single space character). It is the initial value for character fields. Using `WHERE lvorm = space` means "where the deletion flag is not set" — it's equivalent to `WHERE lvorm = ' '` but reads more clearly.

```abap
" These are equivalent:
WHERE lvorm = space.
WHERE lvorm = ' '.
WHERE lvorm IS INITIAL.    " modern syntax — most readable
```

### Filtering on Material Type — MTART Values

Standard SAP material types you will see in WHERE clauses:

| MTART | Description |
|---|---|
| `FERT` | Finished Product |
| `ROH` | Raw Material |
| `HALB` | Semi-Finished Product |
| `HAWA` | Trading Goods (purchased for resale) |
| `DIEN` | Service |
| `NLAG` | Non-Stock Material |
| `VERP` | Packaging |
| `ERSA` | Spare Parts |

### WHERE Clause Performance — The Golden Rules

1. **Filter on the primary key fields first** — this enables a direct key access (fastest possible lookup)
2. **Filter on indexed fields** — secondary indexes exist for frequently queried fields; check SE11 → Indexes tab
3. **Use `IN s_range` for SELECT-OPTIONS variables** — ABAP handles complex multi-value logic automatically
4. **Avoid functions in WHERE on the left side** — `WHERE UPPER( matnr ) = 'ABC'` cannot use indexes
5. **Use `AND` not `OR` when possible** — `OR` conditions are harder to optimize and often cause full table scans

### Checking for Initial/Empty Values

```abap
" All equivalent ways to check if a field is empty/initial:
WHERE matnr  = space.         " for character fields (space = initial)
WHERE matnr  IS INITIAL.      " modern, type-agnostic, clearest intent
WHERE ntgew  = 0.             " for numeric fields
WHERE ersda  = '00000000'.    " for date fields (initial date)
```

---

## 5. Internal Tables — Architecture & Operations
*[Video: ~34:35]*

### The Core Concept — Why Internal Tables Exist

In most programming languages, you fetch data from a database and work with it using whatever collection type the language provides (Python list, Java ArrayList, etc.). In ABAP, the **internal table** is a first-class language construct — not a library class — specifically designed for working with tabular/relational data in memory.

This means:
- SQL-like operations (`READ TABLE ... WITH KEY`, `DELETE ... WHERE`, `SORT BY`) are native language statements
- No casting, no serialization, no ORM — ABAP structures map directly to DB table rows
- The runtime knows the table's structure and can optimize operations accordingly

### The Three-Part Internal Table Pattern

Working with internal tables always involves three things:

```
Internal Table Type Definition    →    Internal Table Variable    →    Work Area
     (the shape/schema)                 (the in-memory collection)    (one row buffer)

TYPES ty_s_mat...                      DATA lt_mats TYPE TABLE OF...    DATA ls_mat TYPE ty_s_mat.
```

```abap
" Step 1: Define the row structure (or use an existing DDIC structure)
TYPES: BEGIN OF ty_s_material,
         matnr TYPE mara-matnr,
         mtart TYPE mara-mtart,
         meins TYPE mara-meins,
         descr TYPE makt-maktx,    " we'll fetch this from a second table
       END OF ty_s_material.

" Step 2: Declare the internal table and work area
DATA: lt_materials TYPE TABLE OF ty_s_material,  " the collection
      ls_material  TYPE ty_s_material.            " one-row buffer

" Step 3: Populate from DB
SELECT matnr mtart meins
  FROM mara
  INTO CORRESPONDING FIELDS OF TABLE lt_materials
  WHERE mtart = 'FERT'.

" Step 4: Process rows in a loop
LOOP AT lt_materials INTO ls_material.
  WRITE: / ls_material-matnr, ls_material-mtart.
ENDLOOP.
```

### SELECT INTO TABLE vs APPEND

Two ways to populate an internal table:

**Method 1: SELECT directly INTO TABLE (bulk, preferred)**
```abap
SELECT matnr mtart FROM mara
  INTO CORRESPONDING FIELDS OF TABLE lt_materials
  WHERE mtart = 'FERT'.
" Fetches ALL matching rows in one DB call — most efficient
```

**Method 2: Build rows manually with APPEND**
```abap
CLEAR ls_material.
ls_material-matnr = '000000000000001234'.
ls_material-mtart = 'FERT'.
ls_material-meins = 'EA'.
APPEND ls_material TO lt_materials.
" Use this when building a result table by combining/transforming data from multiple sources
```

### LOOP AT — Processing Rows

```abap
" Classic loop — copies each row into work area (safe but a copy)
LOOP AT lt_materials INTO ls_material.
  " ls_material is a COPY of the current row
  " Changes to ls_material don't affect lt_materials unless you call MODIFY
  WRITE: / ls_material-matnr, ls_material-meins.
ENDLOOP.

" Loop with WHERE filter (ABAP-level filter, not DB)
LOOP AT lt_materials INTO ls_material
  WHERE mtart = 'FERT'.
  " only rows matching the condition are processed
ENDLOOP.

" Modern loop with inline declaration
LOOP AT lt_materials INTO DATA(ls_mat).
  WRITE: / ls_mat-matnr.
ENDLOOP.

" Loop with field symbol (reference — no copy, more efficient for large structures)
FIELD-SYMBOLS: <ls_mat> TYPE ty_s_material.
LOOP AT lt_materials ASSIGNING <ls_mat>.
  <ls_mat>-meins = 'KG'.    " modifies the table directly — no MODIFY needed
ENDLOOP.
```

### Modifying Rows During a Loop

```abap
LOOP AT lt_materials INTO ls_material.
  IF ls_material-meins = 'G'.          " if unit is grams
    ls_material-meins = 'KG'.           " change to kilograms
    MODIFY lt_materials FROM ls_material.   " write the work area back into the table
  ENDIF.
ENDLOOP.
```

Without the `MODIFY` statement, changes to `ls_material` inside a loop are lost — `ls_material` is just a copy.

### Deleting Rows

```abap
" Delete the current row inside a LOOP
LOOP AT lt_materials INTO ls_material.
  IF ls_material-lvorm = 'X'.
    DELETE lt_materials.    " deletes the row sy-tabix is pointing at
  ENDIF.
ENDLOOP.

" Delete rows matching a condition (no loop needed)
DELETE lt_materials WHERE mtart = 'DIEN'.   " remove all service materials

" Delete all rows (clear the table)
CLEAR lt_materials.       " removes all rows, table is empty
REFRESH lt_materials.     " same as CLEAR for standard tables (legacy keyword)
```

### READ TABLE — Find a Specific Row

```abap
" Search by field value
READ TABLE lt_materials INTO ls_material
  WITH KEY matnr = '000000000000001234'.

IF sy-subrc = 0.
  WRITE: / 'Found:', ls_material-mtart.
  " sy-tabix now points to the row's index in the table
ELSE.
  WRITE: / 'Not found'.
ENDIF.

" Binary search (only works on SORTED tables or after SORT)
SORT lt_materials BY matnr.
READ TABLE lt_materials INTO ls_material
  WITH KEY matnr = lv_search_key
  BINARY SEARCH.
" BINARY SEARCH is O(log n) — much faster than linear scan for large tables
```

### SORT

```abap
SORT lt_materials BY matnr ASCENDING.          " sort by one field
SORT lt_materials BY mtart matnr DESCENDING.   " multi-field sort, all descending
SORT lt_materials BY mtart ASCENDING matnr DESCENDING.   " mixed direction
```

---

## 6. DESCRIBE TABLE — Counting Rows
*[Video: ~39:22]*

### The DESCRIBE TABLE Statement

`DESCRIBE TABLE` interrogates an internal table's metadata at runtime.

```abap
DATA: lv_lines TYPE i,
      lv_kind  TYPE c LENGTH 1.

DESCRIBE TABLE lt_materials LINES lv_lines.
" lv_lines now contains the number of rows in lt_materials

DESCRIBE TABLE lt_materials KIND lv_kind.
" lv_kind = 'T' (standard), 'S' (sorted), 'H' (hashed)
```

### The Modern Equivalent

In modern ABAP (from release 7.0+), use the built-in function `lines()`:

```abap
DATA lv_count TYPE i.
lv_count = lines( lt_materials ).
WRITE: / lv_count, 'materials loaded'.

" Inline usage — directly in conditions
IF lines( lt_materials ) = 0.
  MESSAGE 'No data found' TYPE 'S' DISPLAY LIKE 'W'.
  RETURN.
ENDIF.
```

`lines()` is shorter, more readable, and the preferred modern style. You will see `DESCRIBE TABLE` in older code, and `lines()` in newer code.

### Checking If a Table Is Empty

```abap
" Three equivalent ways:
IF lt_materials IS INITIAL.        " preferred — most readable
IF lines( lt_materials ) = 0.      " also fine
IF lt_materials[] IS INITIAL.      " bracket notation — legacy but common

" Combined check after SELECT:
SELECT * FROM mara INTO TABLE lt_materials WHERE mtart = 'FERT'.
IF lt_materials IS INITIAL.
  MESSAGE 'No finished goods found' TYPE 'S' DISPLAY LIKE 'W'.
  RETURN.
ENDIF.
```

---

## 7. Joining Database Tables in ABAP
*[Video: ~43:57 — 51:29]*

### Why You Need Joins

Real-world SAP data is spread across multiple tables — by design, to keep tables normalized and module-independent. A simple "show me a list of materials with their descriptions" requires data from at minimum two tables:
- `MARA` — the material number, type, UoM
- `MAKT` — the material description text (language-dependent, separate table)

There are **three strategies** for combining data from multiple tables in ABAP:

### Strategy 1: SQL JOIN (Single DB Call)

Standard relational join — executes entirely in the database.

```abap
TYPES: BEGIN OF ty_s_mat_desc,
         matnr TYPE mara-matnr,
         mtart TYPE mara-mtart,
         meins TYPE mara-meins,
         maktx TYPE makt-maktx,    " description text from MAKT
       END OF ty_s_mat_desc.

DATA: lt_result TYPE TABLE OF ty_s_mat_desc.

SELECT a~matnr a~mtart a~meins b~maktx
  FROM mara AS a
  INNER JOIN makt AS b
    ON  b~matnr = a~matnr
    AND b~spras = sy-langu         " match current logon language
  INTO CORRESPONDING FIELDS OF TABLE lt_result
  WHERE a~mtart IN s_mtart
  AND   a~lvorm = space.
```

**Key syntax points:**
- Table aliases: `FROM mara AS a` assigns alias `a` to MARA
- Field qualification: `a~matnr` means field `matnr` from alias `a` (tilde `~` is the separator in ABAP joins, not dot)
- `INNER JOIN` only returns rows where a match exists in BOTH tables
- `LEFT OUTER JOIN` returns all rows from the left table, with nulls for unmatched right-table fields

```abap
" LEFT OUTER JOIN example — show all materials, even those without a description
SELECT a~matnr a~mtart b~maktx
  FROM mara AS a
  LEFT OUTER JOIN makt AS b
    ON  b~matnr = a~matnr
    AND b~spras = sy-langu
  INTO CORRESPONDING FIELDS OF TABLE lt_result.
" Materials without a MAKT entry will have maktx = '' (empty)
```

### Strategy 2: FOR ALL ENTRIES (SAP-Specific Pattern)

This is one of the most important and SAP-specific patterns you will use constantly. It solves the problem of "I have a list of keys, I want to fetch matching rows from another table."

It generates an optimized `WHERE key IN (val1, val2, val3, ...)` query on the database.

```abap
" Step 1: Fetch the primary data
SELECT matnr mtart
  FROM mara
  INTO TABLE lt_mara_data
  WHERE mtart IN s_mtart.

" CRITICAL CHECK: lt_mara_data must NOT be empty before using FOR ALL ENTRIES
" If the driving table is empty, FOR ALL ENTRIES fetches ALL rows from the target table!
IF lt_mara_data IS INITIAL.
  RETURN.
ENDIF.

" Step 2: Fetch descriptions for those specific materials only
SELECT matnr maktx
  FROM makt
  INTO TABLE lt_descriptions
  FOR ALL ENTRIES IN lt_mara_data       " lt_mara_data drives the key list
  WHERE matnr = lt_mara_data-matnr      " join condition
  AND   spras = sy-langu.               " language filter

" Step 3: Merge in ABAP
LOOP AT lt_mara_data INTO ls_mara.
  READ TABLE lt_descriptions INTO ls_desc
    WITH KEY matnr = ls_mara-matnr.
  IF sy-subrc = 0.
    ls_output-matnr = ls_mara-matnr.
    ls_output-mtart = ls_mara-mtart.
    ls_output-maktx = ls_desc-maktx.
    APPEND ls_output TO lt_output.
  ENDIF.
ENDLOOP.
```

**When to use FOR ALL ENTRIES vs SQL JOIN:**
- Use SQL JOIN when the join conditions are simple and both tables are medium-sized
- Use FOR ALL ENTRIES when you have already fetched the primary data and need to enrich it with lookups, or when the join would be too complex for the DB optimizer
- FOR ALL ENTRIES gives you more control over what gets fetched in each step

### Strategy 3: Nested SELECT (Anti-Pattern — DO NOT USE)

```abap
" ❌ NEVER DO THIS — N+1 problem: one DB call per material
LOOP AT lt_mara_data INTO ls_mara.
  SELECT SINGLE maktx FROM makt INTO lv_desc
    WHERE matnr = ls_mara-matnr
    AND   spras = sy-langu.
  ls_mara-maktx = lv_desc.
ENDLOOP.
```

If `lt_mara_data` has 10,000 rows, this makes 10,000 separate database calls. On a production SAP system with millions of materials, this would bring the system to its knees. Always use either a JOIN or FOR ALL ENTRIES instead.

---

## 8. The MAKT Table — Material Descriptions
*[Video: ~43:57]*

### What Is MAKT?

`MAKT` (Material Master Short Texts) stores **language-dependent** material descriptions. This is a separate table from `MARA` because:
- The same material needs different text in different languages
- German users see "Schraubenbolzen M8", English users see "Hexagonal Bolt M8"
- The master data (material number, type, weight) in MARA is language-independent; only the description changes

### MAKT Structure

| Field | Key | Type | Description |
|---|---|---|---|
| `MANDT` | ✓ | CLNT | Client |
| `MATNR` | ✓ | CHAR 18 | Material Number |
| `SPRAS` | ✓ | LANG | Language Key (e.g., `'E'` = English, `'D'` = German) |
| `MAKTX` | | CHAR 40 | Short Text / Description |
| `MAKTG` | | CHAR 40 | Short Text in upper case (for search) |

**Primary Key**: MANDT + MATNR + SPRAS — one description per material per language.

### Always Filter MAKT by Language

When joining or fetching from MAKT, **always add a language filter**, otherwise you get one row per language per material (multiplying your result set):

```abap
" The language key is in sy-langu — current user's logon language
WHERE matnr = lv_matnr
AND   spras = sy-langu.

" Or for a specific language:
WHERE matnr = lv_matnr
AND   spras = 'E'.   " always fetch English descriptions
```

A common bug in SAP ABAP reports is joining MAKT without the language filter, resulting in duplicate material rows (one per language the description exists in).

---

## 9. Custom Internal Tables for JOIN Results
*[Video: ~43:57 — 51:29]*

### The Pattern: Define a Custom Type for Combined Data

When joining multiple tables, the result doesn't map to any single DDIC table. You define a **custom structure** that combines fields from all participating tables:

```abap
" Combined result structure — fields from both MARA and MAKT
TYPES: BEGIN OF ty_s_mat_full,
         matnr TYPE mara-matnr,    " from MARA
         mtart TYPE mara-mtart,    " from MARA
         meins TYPE mara-meins,    " from MARA
         matkl TYPE mara-matkl,    " from MARA
         ntgew TYPE mara-ntgew,    " from MARA
         gewei TYPE mara-gewei,    " from MARA
         maktx TYPE makt-maktx,    " from MAKT — the description
         spras TYPE makt-spras,    " from MAKT — language
       END OF ty_s_mat_full.

DATA: lt_mat_full TYPE TABLE OF ty_s_mat_full,
      ls_mat_full TYPE ty_s_mat_full.
```

Then fetch directly into this custom table:

```abap
SELECT a~matnr a~mtart a~meins a~matkl a~ntgew a~gewei
       b~maktx b~spras
  FROM mara AS a
  INNER JOIN makt AS b
    ON  b~matnr = a~matnr
    AND b~spras = sy-langu
  INTO CORRESPONDING FIELDS OF TABLE lt_mat_full
  WHERE a~mtart IN s_mtart
  AND   a~lvorm = space.

WRITE: / 'Records fetched:', lines( lt_mat_full ).

LOOP AT lt_mat_full INTO ls_mat_full.
  WRITE: / ls_mat_full-matnr,
           ls_mat_full-maktx,
           ls_mat_full-mtart,
           ls_mat_full-meins.
ENDLOOP.
```

### Join Condition Detail — Why MARA~MATNR = MAKT~MATNR

The join condition `b~matnr = a~matnr` creates the relational link between the two tables — it tells the database: "for each MARA row, find the MAKT row where the material number matches." Without this, you would get a cartesian product (every MARA row paired with every MAKT row — catastrophic for large tables).

The full join condition should always be:
```abap
ON  b~matnr = a~matnr     " material number must match
AND b~spras = sy-langu     " AND language must match current user's language
```

Both conditions together ensure exactly one description row per material row (assuming one description per language per material, which is the case in MAKT).

---

## 10. Modularizing Code with INCLUDE Programs
*[Video: ~01:05:01 — 01:08:10]*

### The Problem: Growing Programs

A real-world SAP report might have:
- 50+ data declarations (global variables, types, internal tables)
- A complex selection screen with 20+ parameters
- Multiple FORM subroutines spanning hundreds of lines
- Total program size: 2,000–10,000+ lines

Maintaining all of this in one file becomes unwieldy. ABAP's solution is **INCLUDE programs**.

### What Is an INCLUDE Program?

An INCLUDE program is simply a **named fragment of ABAP code** stored as a separate object in SAP, but inserted verbatim into the host program at compile time.

Think of it exactly like a `#include` in C or an `import` that pastes code inline — there is no separate compilation unit, no scope boundary, no separate namespace. The included code is treated as if you had typed it directly in the main program.

```abap
" In main program ZMAT_REPORT:
REPORT zmat_report.

INCLUDE zmat_report_top.      " global data declarations
INCLUDE zmat_report_sel.      " selection screen declarations
INCLUDE zmat_report_f01.      " form subroutines
```

```abap
" In INCLUDE zmat_report_top (content):
" No REPORT statement — just declarations
DATA: gt_materials TYPE TABLE OF ty_s_mat_full,
      gs_material  TYPE ty_s_mat_full,
      gv_company   TYPE bukrs.

TYPES: BEGIN OF ty_s_mat_full,
         matnr TYPE mara-matnr,
         mtart TYPE mara-mtart,
         maktx TYPE makt-maktx,
       END OF ty_s_mat_full.
```

### Rules for INCLUDE Programs

1. **Cannot run independently** — if you press F8 on an INCLUDE, SAP gives an error. It has no `REPORT` or `PROGRAM` header. It is purely a fragment.
2. **No duplicate declarations** — since the INCLUDE is pasted inline, declaring the same variable in both the main program and the INCLUDE causes a duplicate declaration error
3. **Naming convention**: Include names must be unique in the SAP system. Common convention: `<main_program>_<purpose>` (e.g., `ZMAT_REPORT_TOP`, `ZMAT_REPORT_F01`)
4. **Cannot be nested circularly** — you cannot have INCLUDE A containing INCLUDE B which includes INCLUDE A
5. **Transport together** — the main program and all its INCLUDEs should be in the same transport request

### Standard INCLUDE Naming Convention

SAP's own programs follow this naming pattern, and custom programs adopt the same convention:

| Include Name Suffix | Purpose |
|---|---|
| `_TOP` | Global data declarations, type definitions |
| `_SEL` or `_SCR` | Selection screen declarations |
| `_F01`, `_F02`, ... | Form subroutines (one file per logical group) |
| `_I01`, `_I02`, ... | Module (PAI/PBO) implementations for screen programming |
| `_O01`, `_O02`, ... | Output modules for screen programming |

### When to Use INCLUDEs

Use INCLUDEs when:
- Your program exceeds ~500 lines and managing it in one file is difficult
- You want to logically separate data declarations from business logic from UI
- Your team has multiple developers working on different parts of the same program

Do NOT use INCLUDEs for code that should be shared across multiple programs — use Function Modules or Classes for that instead. INCLUDEs are tightly coupled to their host program.

---

## 11. The TOP Include Convention
*[Video: ~01:05:01]*

### What Is a TOP Include?

By strong convention in SAP ABAP development, the first INCLUDE in any executable program is the **TOP include** (short for "top-of-program"). It contains:

1. All global `DATA` and `TYPES` declarations
2. Global `CONSTANTS` declarations
3. Global structure and table definitions used throughout the program

```abap
" Main program: ZMAT_REPORT
REPORT zmat_report.

INCLUDE zmat_report_top.     " ← TOP include: all global declarations go here

SELECTION-SCREEN BEGIN OF BLOCK b1 WITH FRAME TITLE TEXT-001.
  SELECT-OPTIONS: s_mtart FOR mara-mtart,
                  s_matnr FOR mara-matnr.
SELECTION-SCREEN END OF BLOCK b1.

START-OF-SELECTION.
  PERFORM fetch_data.
  PERFORM display_results.

INCLUDE zmat_report_f01.     " FORM subroutines
```

```abap
" Content of ZMAT_REPORT_TOP:
" ─────────────────────────────────
TYPES: BEGIN OF ty_s_material,
         matnr TYPE mara-matnr,
         mtart TYPE mara-mtart,
         meins TYPE mara-meins,
         maktx TYPE makt-maktx,
       END OF ty_s_material.

DATA: gt_materials TYPE TABLE OF ty_s_material,
      gs_material  TYPE ty_s_material.

CONSTANTS: gc_lang_en TYPE spras VALUE 'E'.
```

### Why This Matters

When you maintain a large SAP program or do a code review, you immediately know where to find global variable declarations: the TOP include. This is a universal convention in SAP ABAP shops — even new developers can navigate unfamiliar programs because the structure is predictable.

When you use SE80 (Object Navigator) to browse a program's objects, all INCLUDEs are listed in the tree view under the main program, making navigation straightforward.

---

## 12. Debugging ABAP Programs
*[Video: ~01:16:31]*

### Why ABAP Debugging Is Different

In most development environments you debug with an IDE breakpoint (Eclipse, VS Code, IntelliJ). ABAP is different:
- SAP has its own integrated debugger, launched from within the SAP GUI
- The debugger runs **server-side** — you are debugging the ABAP runtime on the application server, not local code
- You can debug programs run by other users (with authorization) — critical for investigating production issues
- The debugger can inspect the live database state alongside the program variables

### Setting a BREAKPOINT in Code

The simplest way to enter the debugger is to place a `BREAK-POINT` statement directly in your code:

```abap
START-OF-SELECTION.
  PERFORM fetch_data.

  BREAK-POINT.     " ← execution pauses here and the debugger opens

  PERFORM display_results.
```

When the program reaches `BREAK-POINT`:
1. The current program execution is paused
2. The ABAP Debugger window opens automatically
3. You can inspect all variable values, step through code, and watch table contents

> **Important**: Always remove `BREAK-POINT` statements before transporting to QAS/PRD. They are development-time aids only. A breakpoint accidentally left in production code will pause execution for every user who runs the program.

### Setting a Session Breakpoint (Without Code Change)

You can set a breakpoint **without modifying the source code** using the ABAP Editor:
1. Open the program in SE38
2. Click on the line number where you want to pause
3. Press F9 (or menu: Utilities → Breakpoints → Set)
4. A red dot appears on the line

Session breakpoints are personal — they only apply to your own SAP user session. They are not saved in the source code. They disappear when you log out.

### The `/h` Debugger Entry Command

Type `/h` in the SAP command field and press Enter to **switch the current session into debug mode**. The next transaction or program you run will open in the debugger automatically, pausing at the first executable statement.

This is extremely useful when you want to debug a standard SAP transaction without modifying its source code.

---

## 13. Breakpoint Types — Session vs External
*[Video: ~01:18:21]*

### Why Multiple Breakpoint Types?

Not all ABAP programs run in your own dialog session. Some programs run as:
- Background jobs (scheduled batch programs — no interactive session)
- RFC calls initiated by an external system
- Programs triggered by another user's session
- Programs running under a service user account

For these cases, a standard session breakpoint (which only fires in your session) is useless. SAP provides different breakpoint types to handle these scenarios.

### Breakpoint Types Explained

**1. Session Breakpoints (standard)**
- Fires only when the program runs in **your own session**
- Set via SE38 (F9) or `/h` command
- Not saved in code, disappears when you log out
- Use for: debugging programs you run yourself

**2. Hardcoded BREAK-POINT (code-level)**
- Written directly in the ABAP source as `BREAK-POINT`
- Fires for **all users** who run the program
- Persists until you remove it from code
- Use for: quick testing during development; ALWAYS remove before transport

**3. User-Specific External Breakpoints (`BREAK username`)**
- Fires only when a **specific named user** runs the program
- Set with: `BREAK username.` in the code (e.g., `BREAK jsmith.`)
- More targeted than `BREAK-POINT` — won't affect other users running the same program
- Use for: debugging code that runs under another user's session, or testing user-specific behavior

```abap
BREAK-POINT.        " pauses for ALL users — dangerous in shared environments
BREAK jsmith.       " pauses only when user JSMITH runs this code
```

**4. Dynamic External Breakpoints (SE80 / Debugger)**
- Set from within the debugger itself for a specific user
- Used for debugging RFC calls and background jobs
- Most complex to set up — requires debugger authorization

### Practical Debugging Workflow

For a developer testing their own code in DEV:
1. Add `BREAK-POINT` at the point of interest
2. Run the program (F8)
3. Debugger opens automatically at that line
4. Inspect, step through, fix
5. Remove `BREAK-POINT` before activating the final version

For investigating a bug where the program runs under a different user (e.g., a background job):
1. Add `BREAK <your_username>.` in the code
2. Have the job/RFC triggered (or trigger it yourself logged in as that user)
3. The debugger opens when YOUR session intercepts it

---

## 14. The ABAP Debugger — Navigation & Inspection

### The Debugger Interface

When the debugger opens, you see:
- **Source code panel** — shows the current program, with the current execution line highlighted
- **Variable display panel** — shows variables, their types, and current values
- **Toolbar** — navigation controls

### Key Navigation Controls

| Key / Button | Action |
|---|---|
| **F5** | Step Into — execute one statement, entering called subroutines/methods |
| **F6** | Step Over — execute one statement, skipping over subroutine internals |
| **F7** | Step Out — run to the end of the current subroutine and return to the caller |
| **F8** | Continue — run until the next breakpoint |
| **F12** | Exit debugger (terminate the program) |

### Inspecting Variables

In the debugger, you can inspect any variable by:
- Double-clicking on a variable name in the source panel — it appears in the variable display
- Typing the variable name directly in the variable display field
- Right-clicking → "Display/Change Variable"

**Watching Internal Tables:**
- Click on an internal table variable in the debugger
- A separate "Table view" shows all rows with their field values
- You can scroll through thousands of rows, inspect individual cells, and even change values in the debugger for testing

### Changing Variable Values at Runtime

One of the most powerful debugging features in ABAP: you can **modify variable values while the program is paused** and then continue execution. This lets you test "what would happen if this variable had value X" without changing the code.

Double-click on a variable in the display → the value field becomes editable → change it → press Enter → continue.

### Watching the Database

The ABAP Debugger has a built-in SQL debugger (ST05 overlay) accessible from the debugger toolbar. You can see every SQL statement being executed against the database in real time, with execution times. This is invaluable for identifying performance issues.

---

## 15. Passing Data Between Programs — ABAP Memory
*[Video: ~01:48:11]*

### The Problem

SAP programs do not share variables automatically. Each ABAP program runs in its own context — variables declared in Program A are invisible to Program B. But sometimes you need one program to pass data to another.

Scenario: Your main report (`ZREPORT_A`) fetches 5,000 material records. It then calls a sub-report or another program (`ZREPORT_B`) to process or display those records. How does `ZREPORT_B` get the 5,000 rows without re-fetching them from the database?

Answer: **ABAP Memory**.

### What Is ABAP Memory?

ABAP Memory is a **temporary key-value store** that exists within a single internal SAP session (one user's logon session, one work process). It is:
- Shared between all programs running in the same internal session
- Cleared when the user logs out
- Not visible to other users or sessions
- Capable of storing any ABAP data object: variables, structures, internal tables, even complex nested objects

Think of it as a session-scoped dictionary/hashmap that any program in the same session can read and write.

### EXPORT to ABAP Memory

Writes data into ABAP Memory under a named key (the Memory ID).

```abap
" Program A — stores data in ABAP Memory
DATA: lt_materials TYPE TABLE OF ty_s_material,
      lv_count     TYPE i,
      ls_settings  TYPE ty_s_report_settings.

" ... populate lt_materials from DB ...

" Export to ABAP Memory under a named key
EXPORT lt_materials TO MEMORY ID 'ZMAT_DATA'.
EXPORT lv_count     TO MEMORY ID 'ZMAT_COUNT'.
EXPORT ls_settings  TO MEMORY ID 'ZMAT_SETTINGS'.

" Now call the second program
SUBMIT zreport_b AND RETURN.    " AND RETURN = control returns to Program A after B finishes
```

### IMPORT from ABAP Memory

Reads data from ABAP Memory using the same named key.

```abap
" Program B — retrieves data from ABAP Memory
DATA: lt_materials TYPE TABLE OF ty_s_material,
      lv_count     TYPE i.

IMPORT lt_materials FROM MEMORY ID 'ZMAT_DATA'.
IMPORT lv_count     FROM MEMORY ID 'ZMAT_COUNT'.

IF sy-subrc <> 0.
  MESSAGE 'Required data not in memory — run Program A first' TYPE 'E'.
  RETURN.
ENDIF.

" lt_materials now has all the data Program A put there
WRITE: / 'Processing', lv_count, 'materials'.
LOOP AT lt_materials INTO DATA(ls_mat).
  " process...
ENDLOOP.
```

### FREE MEMORY — Cleaning Up

After you are done with the data, free the memory to release resources:

```abap
FREE MEMORY ID 'ZMAT_DATA'.
FREE MEMORY ID 'ZMAT_COUNT'.
```

### Important Limitations of ABAP Memory

1. **Session-scoped only**: If Program B opens in a new SAP session (new window via `/o`), it cannot access ABAP Memory written by Program A in the original session
2. **Temporary**: Cleared when the user logs out, or when `FREE MEMORY` is called
3. **No type enforcement**: The key is just a string. If Program A exports a table of type X and Program B imports it expecting type Y, you get a type mismatch error at runtime
4. **Single-user**: Each user has their own ABAP Memory space — you cannot use it to pass data between users

---

## 16. Passing Data Across Sessions — SAP Memory
*[Video: ~01:56:52]*

### What Is SAP Memory?

SAP Memory is a **user-level persistent store** — it survives across sessions and transactions for the same user (until they log out from SAP entirely). It is different from ABAP Memory in that it persists beyond a single program execution.

More practically: SAP Memory is used to **pre-populate fields in SAP transactions**. When you look at a SAP screen and a field is already filled in with your last-used value, that's SAP Memory at work.

### Parameter IDs (PIDs)

SAP Memory uses **Parameter IDs** (PIDs) — short named identifiers linked to specific fields. Each Data Element in DDIC can be linked to a Parameter ID. When a program saves a value to SAP Memory using a PID, any screen field that uses a Data Element with that same PID will automatically pre-fill.

Examples of common PIDs:

| PID | Associated Field | Description |
|---|---|---|
| `BUK` | BUKRS | Company Code |
| `MAT` | MATNR | Material Number |
| `KUN` | KUNNR | Customer Number |
| `WRK` | WERKS | Plant |
| `CAR` | CARRID | Airline Code |

### SET PARAMETER — Write to SAP Memory

```abap
" Save the current material number to SAP Memory
SET PARAMETER ID 'MAT' FIELD ls_material-matnr.

" Save the company code
SET PARAMETER ID 'BUK' FIELD p_bukrs.

" Save multiple values
SET PARAMETER ID 'WRK' FIELD lv_plant.
SET PARAMETER ID 'KUN' FIELD ls_customer-kunnr.
```

### GET PARAMETER — Read from SAP Memory

```abap
" Read the material number from SAP Memory
DATA lv_matnr TYPE matnr.
GET PARAMETER ID 'MAT' FIELD lv_matnr.

IF sy-subrc = 0.
  WRITE: / 'Last used material:', lv_matnr.
ELSE.
  WRITE: / 'No material stored in memory yet'.
ENDIF.

" Read company code
DATA lv_bukrs TYPE bukrs.
GET PARAMETER ID 'BUK' FIELD lv_bukrs.
```

### Practical Use Case: Auto-Populating Transaction Fields

When a user opens transaction MM03 (Material Display) after running your report, if your report did `SET PARAMETER ID 'MAT' FIELD ls_material-matnr`, the material number field in MM03 will already be filled with the value from your report. This is how SAP creates the "smart" auto-fill behavior users experience.

```abap
" Your report — after processing material '000000000000001234':
SET PARAMETER ID 'MAT' FIELD ls_material-matnr.

" Then launch MM03 programmatically:
CALL TRANSACTION 'MM03' AND SKIP FIRST SCREEN.
" The first screen of MM03 is the material number entry screen
" AND SKIP FIRST SCREEN skips it because the field is already populated from SAP Memory
```

### How Parameter IDs Are Linked to Data Elements

In SE11, when you edit a Data Element:
- Go to the "Further Characteristics" tab
- "Parameter ID" field — link it to a PID like `MAT`
- "Set Parameter" checkbox — automatically saves field value to SAP Memory when user leaves the field
- "Get Parameter" checkbox — automatically reads from SAP Memory to pre-fill the field when the screen opens

This is how SAP provides the "sticky fields" UX without any custom code in transactions.

---

## 17. ABAP Memory vs SAP Memory — Side-by-Side Comparison

This comparison is frequently asked in SAP ABAP interviews and important to have completely clear.

| Aspect | ABAP Memory | SAP Memory |
|---|---|---|
| **Scope** | Within ONE internal session (one work process context) | Across ALL sessions of the SAME user |
| **Lifetime** | Duration of the internal session | Until user logs out of SAP entirely |
| **What it stores** | Any ABAP data: variables, structures, tables, complex objects | Simple scalar values (single field values) |
| **How to write** | `EXPORT varname TO MEMORY ID 'key'.` | `SET PARAMETER ID 'PID' FIELD variable.` |
| **How to read** | `IMPORT varname FROM MEMORY ID 'key'.` | `GET PARAMETER ID 'PID' FIELD variable.` |
| **How to clear** | `FREE MEMORY ID 'key'.` | Cannot be explicitly cleared (persists until logout) |
| **Cross-session?** | No — only current internal session | Yes — survives across transactions and sessions |
| **Use case** | Passing complex data structures between programs called in sequence | Pre-populating screen fields, passing simple IDs between transactions |
| **Key identifier** | Free-text string (your naming choice) | Standard PID linked to a Data Element in DDIC |
| **Performance** | Stores full data in work process memory — can be large | Small — stores only field values |

### Memory Architecture Diagram

```
USER LOGS INTO SAP
       │
       ▼
SAP Memory (user-level)
  ├── PID 'BUK' = '1000'          ← persists across all transactions in this logon
  ├── PID 'MAT' = '000000000001'
  └── PID 'WRK' = '0001'
       │
       ├── Session 1 (main window)
       │     │
       │     ├── ABAP Memory (session 1 only)
       │     │     ├── 'ZMAT_DATA' → lt_materials (5000 rows)
       │     │     └── 'ZMAT_COUNT' → 5000
       │     │
       │     ├── Program A running...
       │     └── Program B running... (can access session 1's ABAP Memory)
       │
       └── Session 2 (opened with /o)
             ├── ABAP Memory (session 2 only — DIFFERENT from session 1)
             └── Can access SAP Memory (shared user-level store)
```

---

## 18. Key Takeaways & Revision Questions

### Core Concepts Summary

**SELECT patterns**: Always use `SELECT INTO TABLE` for multi-row results (one DB call). Never use `SELECT...ENDSELECT` in modern code. Never put `SELECT` inside a `LOOP`. Use `FOR ALL ENTRIES IN` to look up related records in bulk.

**Internal table lifecycle**: Always check `IS INITIAL` after a SELECT, before using FOR ALL ENTRIES, and before displaying results. Always `CLEAR` the work area before each APPEND in a loop to avoid stale values.

**MARA and MAKT**: `MARA` is the core material table (one row per material). `MAKT` stores language-dependent descriptions (one row per material per language). Always join MAKT with an `AND b~spras = sy-langu` condition.

**INCLUDE programs**: Code fragments pasted inline at compile time. Cannot run alone. Always follow the TOP/F01 naming convention. Use for organizing large programs — not for shared code.

**Debugging**: Session breakpoints for your own testing. `BREAK-POINT` in code for temporary pausing. `BREAK username` for user-specific pausing. Always remove breakpoints before transporting.

**ABAP Memory**: Session-scoped. Stores complex data objects. Use `EXPORT/IMPORT TO/FROM MEMORY ID`. Free after use.

**SAP Memory**: User-scoped (cross-session). Stores scalar field values. Use `SET/GET PARAMETER ID`. Links to DDIC Data Elements via PIDs. Powers auto-fill behavior in screens.

### Revision Questions

1. What is the difference between `SELECT SINGLE` and `SELECT INTO TABLE`? When do you use each?
2. Why is `SELECT...ENDSELECT` discouraged in modern ABAP? What should you use instead?
3. What is the N+1 problem in ABAP and how does `FOR ALL ENTRIES IN` solve it?
4. What is the CRITICAL rule about the driving table when using `FOR ALL ENTRIES IN`? What happens if you break it?
5. What are the three strategies for combining data from multiple tables in ABAP? Compare their trade-offs.
6. Why is `MAKT` a separate table from `MARA`? What field must you always include in the WHERE clause when querying MAKT?
7. What does `DESCRIBE TABLE lt_tab LINES lv_count` do? What is the modern equivalent?
8. What is an INCLUDE program? Can it run independently? Why do programs use them?
9. What is the naming convention for the TOP include? What goes in it?
10. What is the difference between a session breakpoint and an external breakpoint? When would you use each?
11. How do you set a breakpoint for a specific user without affecting others running the same program?
12. What is ABAP Memory? What is its scope? What statements do you use to write and read it?
13. What is SAP Memory? How does it differ from ABAP Memory? What are Parameter IDs?
14. A screen field auto-populates when a user navigates to a transaction. Which memory mechanism is responsible?
15. You have Program A that fetches a 10,000-row internal table. Program B needs those rows. How do you pass them? (What mechanism and what syntax?)
16. What would happen if you did `EXPORT lt_big_table TO MEMORY ID 'DATA'.` and then the user opened a second session with `/o`? Would the second session's programs see the data?

---

*Notes based on ZAP Yard — SAP ABAP Video 3 of 11*
*Previous: SAP_ABAP_Video2_StudyNotes.md*
*Reference: ABAP_DDIC_Tutorial.md for syntax cheat sheet and additional code patterns*
