# ABAP Internal Table DML & Application Layer Processing
> Study Reference: Data Manipulation, In-Memory SQL, and Execution Architecture
> Cross-reference: SAP_ABAP_Video2_StudyNotes.md | SAP_ABAP_Video3_StudyNotes.md

---

## Table of Contents

1. [Conceptual Foundation — Why Internal Tables Have DML](#1-conceptual-foundation)
2. [INSERT and APPEND](#2-insert-and-append)
3. [MODIFY — Update Rows](#3-modify--update-rows)
4. [DELETE — Remove Rows](#4-delete--remove-rows)
5. [READ TABLE WITH KEY — Single Record Lookup](#5-read-table-with-key--single-record-lookup)
6. [LOOP AT WHERE — Filtered Iteration](#6-loop-at-where--filtered-iteration)
7. [FILTER — Functional Table Filtering (Modern ABAP)](#7-filter--functional-table-filtering-modern-abap)
8. [SORT — Ordering Rows](#8-sort--ordering-rows)
9. [COLLECT — Aggregation Without Loops](#9-collect--aggregation-without-loops)
10. [DELETE ADJACENT DUPLICATES — Deduplication](#10-delete-adjacent-duplicates--deduplication)
11. [In-Memory SQL — SELECT on Internal Tables](#11-in-memory-sql--select-on-internal-tables)
12. [In-Memory JOINs Between Internal Tables](#12-in-memory-joins-between-internal-tables)
13. [Application Layer Processing — Execution Architecture](#13-application-layer-processing--execution-architecture)
14. [Work Processes, User Context, and Roll Area](#14-work-processes-user-context-and-roll-area)
15. [Quick Syntax Reference Card](#15-quick-syntax-reference-card)

---

## 1. Conceptual Foundation
*Why do internal tables have SQL-like operations built into the language?*

In most languages you'd loop through a list to manipulate it:

```python
# Python — you write the loop manually
materials = [...]
for mat in materials:
    if mat['type'] == 'FERT':
        mat['status'] = 'ACTIVE'
```

ABAP takes a different philosophy: because **tabular data manipulation is the core task of almost every business program**, the language provides **set-based operations** that operate on entire tables at once — no manual loops required.

This mirrors SQL's philosophy: instead of "for each row, do X", you say "update all rows where condition Y". The runtime handles iteration internally, optimally.

This is why ABAP is called a **4th-generation language** — the set operations are built into the language syntax, not a library you import.

### The Two Worlds of ABAP Data

```
DATABASE LAYER                    APPLICATION LAYER (your program's memory)
─────────────────                 ─────────────────────────────────────────
Real DB tables                    Internal Tables
  ↕ Open SQL                        ↕ ABAP DML statements
  SELECT / INSERT                   APPEND / MODIFY / DELETE
  UPDATE / DELETE                   READ TABLE / LOOP AT
                                    SORT / COLLECT / FILTER
```

Everything in this document operates on **internal tables** — data already loaded into the application server's memory. None of these statements touch the database directly.

---

## 2. INSERT and APPEND
*Adding rows to an internal table*

Both statements add rows, but they behave differently depending on table type and position.

### APPEND — Add to the End

`APPEND` always adds the row to the **end** of the table. It is the most common way to add rows to a standard table.

```abap
" Setup for both examples
TYPES: BEGIN OF ty_s_product,
         prod_id   TYPE numc6,
         category  TYPE c LENGTH 4,
         name      TYPE c LENGTH 30,
         price     TYPE p DECIMALS 2,
         stock_qty TYPE i,
       END OF ty_s_product.

DATA: lt_products TYPE TABLE OF ty_s_product,
      ls_product  TYPE ty_s_product.
```

**Example 1 — APPEND using work area (classic style)**

```abap
" Example 1: Build a product catalog row by row
" Use case: you're constructing output rows by combining data from multiple sources

CLEAR ls_product.
ls_product-prod_id   = '000001'.
ls_product-category  = 'ELEC'.
ls_product-name      = 'USB-C Charging Cable'.
ls_product-price     = '12.99'.
ls_product-stock_qty = 500.
APPEND ls_product TO lt_products.

CLEAR ls_product.                    " CRITICAL: always CLEAR before reuse
ls_product-prod_id   = '000002'.
ls_product-category  = 'ELEC'.
ls_product-name      = 'Wireless Mouse'.
ls_product-price     = '34.99'.
ls_product-stock_qty = 200.
APPEND ls_product TO lt_products.

CLEAR ls_product.
ls_product-prod_id   = '000003'.
ls_product-category  = 'FURN'.
ls_product-name      = 'Ergonomic Chair'.
ls_product-price     = '299.00'.
ls_product-stock_qty = 45.
APPEND ls_product TO lt_products.

WRITE: / 'Products loaded:', lines( lt_products ).
" Output: Products loaded:         3
```

> **Why CLEAR is critical**: `ls_product` is a reused buffer. Without `CLEAR`, fields from the previous row bleed into the next. E.g., if row 2 has no `category` assignment, it would inherit `'ELEC'` from row 1.

**Example 2 — APPEND using VALUE inline constructor (modern ABAP)**

```abap
" Example 2: Modern inline syntax — no work area variable needed
" Use case: when values are known at write time, cleaner and less error-prone

APPEND VALUE #( prod_id   = '000004'
                category  = 'BOOK'
                name      = 'Clean Code'
                price     = '39.95'
                stock_qty = 120 ) TO lt_products.

APPEND VALUE #( prod_id   = '000005'
                category  = 'BOOK'
                name      = 'The Pragmatic Programmer'
                price     = '44.95'
                stock_qty = 88 ) TO lt_products.

" After both examples, lt_products has 5 rows
WRITE: / 'Total products:', lines( lt_products ).
" Output: Total products:         5
```

### INSERT — Add at a Specific Position

`INSERT` adds a row at a **specified index** position, pushing existing rows down.

**Example 1 — INSERT at a specific index**

```abap
" Example 1: Insert a PRIORITY row at position 1 (top of list)
" Use case: prepend a header/summary row, or insert at a sorted position

DATA ls_priority TYPE ty_s_product.
ls_priority-prod_id   = '000000'.
ls_priority-category  = 'FEAT'.
ls_priority-name      = '*** FEATURED PRODUCT ***'.
ls_priority-price     = '0.00'.
ls_priority-stock_qty = 0.

INSERT ls_priority INTO lt_products INDEX 1.
" lt_products now has this row at position 1, all others shifted down

" Verify: read the first row
READ TABLE lt_products INTO ls_product INDEX 1.
WRITE: / 'First row:', ls_product-name.
" Output: First row: *** FEATURED PRODUCT ***
```

**Example 2 — INSERT INTO TABLE (for sorted/hashed tables)**

For sorted and hashed tables, you cannot use APPEND (which would violate sort order or hash integrity). Use `INSERT INTO TABLE` instead — the runtime places the row in the correct position automatically.

```abap
" Sorted table — rows are automatically kept in key order
DATA lt_sorted TYPE SORTED TABLE OF ty_s_product
     WITH UNIQUE KEY prod_id.

" INSERT INTO TABLE places rows in the correct sorted position automatically
INSERT VALUE #( prod_id = '000010' category = 'ELEC'
                name = 'Keyboard' price = '79.99' stock_qty = 150 )
       INTO TABLE lt_sorted.

INSERT VALUE #( prod_id = '000005' category = 'BOOK'
                name = 'Refactoring' price = '49.99' stock_qty = 60 )
       INTO TABLE lt_sorted.

INSERT VALUE #( prod_id = '000001' category = 'FURN'
                name = 'Desk Lamp' price = '24.99' stock_qty = 300 )
       INTO TABLE lt_sorted.

" Even though inserted in order 10→5→1, the sorted table maintains order 1→5→10
LOOP AT lt_sorted INTO ls_product.
  WRITE: / ls_product-prod_id, ls_product-name.
ENDLOOP.
" Output:
" 000001  Desk Lamp
" 000005  Refactoring
" 000010  Keyboard
```

### APPEND vs INSERT — When to Use Which

| Scenario | Statement |
|---|---|
| Adding to end of a standard table | `APPEND` |
| Adding at a specific position | `INSERT ... INDEX n` |
| Adding to a sorted or hashed table | `INSERT INTO TABLE` |
| Building result table in a loop | `APPEND` |
| Inserting with guaranteed uniqueness | `INSERT INTO TABLE` on a UNIQUE key table |

---

## 3. MODIFY — Update Rows
*Changing existing rows in an internal table*

`MODIFY` updates an existing row. The behavior differs depending on whether you specify an index or a key.

### MODIFY by Index — Update a Specific Row

**Example 1 — MODIFY a row at a known index**

```abap
" Example 1: Update the price of the first product in the table
" Use case: apply a global price change, update a specific position after reading it

READ TABLE lt_products INTO ls_product INDEX 1.
IF sy-subrc = 0.
  ls_product-price = ls_product-price * '0.90'.   " 10% discount
  MODIFY lt_products FROM ls_product INDEX 1.       " write back at same position
  WRITE: / 'Updated price:', ls_product-price.
ENDIF.
```

**Example 2 — MODIFY inside a LOOP (most common pattern)**

```abap
" Example 2: Apply a category-based price discount to ALL matching rows
" Use case: bulk update — raise all BOOK prices by 5%, all ELEC by 2%

LOOP AT lt_products INTO ls_product.
  CASE ls_product-category.
    WHEN 'BOOK'.
      ls_product-price = ls_product-price * '1.05'.   " +5%
    WHEN 'ELEC'.
      ls_product-price = ls_product-price * '1.02'.   " +2%
  ENDCASE.
  MODIFY lt_products FROM ls_product.   " no INDEX needed inside LOOP — uses sy-tabix
ENDLOOP.

" Verify results
LOOP AT lt_products INTO ls_product WHERE category = 'BOOK'.
  WRITE: / ls_product-name, ls_product-price.
ENDLOOP.
```

> Inside a `LOOP AT`, `MODIFY lt_products FROM ls_product` without an `INDEX` clause automatically uses `sy-tabix` — the current row index. This is the standard idiom.

> **Modern alternative**: Use `FIELD-SYMBOLS` to modify in place without MODIFY:
> ```abap
> FIELD-SYMBOLS <ls_p> TYPE ty_s_product.
> LOOP AT lt_products ASSIGNING <ls_p>.
>   IF <ls_p>-category = 'BOOK'.
>     <ls_p>-price = <ls_p>-price * '1.05'.   " direct modification — no MODIFY needed
>   ENDIF.
> ENDLOOP.
> ```

---

## 4. DELETE — Remove Rows
*Removing rows from an internal table*

### DELETE by Condition

**Example 1 — DELETE WHERE (bulk conditional delete)**

```abap
" Example 1: Remove all out-of-stock products from the display table
" Use case: clean up a result set before output — remove irrelevant rows

WRITE: / 'Before delete:', lines( lt_products ), 'rows'.

DELETE lt_products WHERE stock_qty = 0.

WRITE: / 'After delete: ', lines( lt_products ), 'rows'.

" More complex condition
DELETE lt_products WHERE category = 'FEAT'    " remove featured markers
                    OR  price < '5.00'.        " remove items below minimum price
```

**Example 2 — DELETE inside a LOOP (conditional, with complex logic)**

```abap
" Example 2: Delete rows where price-to-stock ratio is too low
" Use case: when deletion condition requires computed values or multi-step logic
" that cannot be expressed in a simple WHERE clause

DATA lv_ratio TYPE p DECIMALS 4.

LOOP AT lt_products INTO ls_product.
  IF ls_product-stock_qty > 0.
    lv_ratio = ls_product-price / ls_product-stock_qty.
    IF lv_ratio < '0.1000'.    " price per unit of stock too low — remove
      DELETE lt_products.      " deletes the CURRENT row (sy-tabix) — no INDEX needed
    ENDIF.
  ENDIF.
ENDLOOP.

WRITE: / 'After ratio filter:', lines( lt_products ), 'rows'.
```

> **Rule**: When using `DELETE lt_products` (without WHERE) inside a LOOP, it deletes the current row (`sy-tabix`). The loop cursor automatically adjusts — you do NOT skip a row.

---

## 5. READ TABLE WITH KEY — Single Record Lookup
*Finding one specific row instantly*

`READ TABLE ... WITH KEY` is the internal table equivalent of `SELECT SINGLE` on a DB table. It finds the first row matching the key condition and returns it (or sets `sy-subrc = 4` if not found).

### Linear Search vs Binary Search

By default, READ TABLE does a **linear scan** — it checks rows one by one from the top. For large tables this is O(n). Use `BINARY SEARCH` after a `SORT` to get O(log n) performance.

**Example 1 — Simple key lookup (linear search)**

```abap
" Example 1: Find a product by its ID
" Use case: look up a single record you know exists (small tables, or unique key lookup)

DATA ls_found TYPE ty_s_product.

READ TABLE lt_products INTO ls_found
  WITH KEY prod_id = '000003'.

IF sy-subrc = 0.
  WRITE: / 'Found product:', ls_found-name.
  WRITE: / 'Price:        ', ls_found-price.
  WRITE: / 'Stock:        ', ls_found-stock_qty.
ELSE.
  WRITE: / 'Product 000003 not found in table'.
ENDIF.
```

**Example 2 — Binary search on a sorted table (fast lookup for large tables)**

```abap
" Example 2: High-performance lookup on a large table
" Use case: you have 50,000+ rows and need to look up specific records repeatedly
" Linear search on 50,000 rows = up to 50,000 comparisons
" Binary search on 50,000 rows = max 16 comparisons (log2 50000 ≈ 15.6)

" Step 1: Sort the table on the key field you will search by
SORT lt_products BY prod_id ASCENDING.

" Step 2: Use BINARY SEARCH — only valid after SORT on the key fields
READ TABLE lt_products INTO ls_found
  WITH KEY prod_id = '000002'
  BINARY SEARCH.

IF sy-subrc = 0.
  WRITE: / 'Fast lookup found:', ls_found-name.
  " sy-tabix points to the row index — useful if you need to MODIFY it
  WRITE: / 'Row index was:', sy-tabix.
ELSE.
  WRITE: / 'Not found'.
ENDIF.
```

> **Important constraint**: `BINARY SEARCH` only works correctly if the table is sorted by the exact fields you are searching. If you sort by `prod_id` and then binary search by `category`, you get wrong results. Sort and search must use the same fields in the same order.

### READ TABLE with Field Symbol (No Copy — Efficient)

```abap
" Reference the row directly instead of copying it — useful for large structures
FIELD-SYMBOLS <ls_prod> TYPE ty_s_product.

READ TABLE lt_products ASSIGNING <ls_prod>
  WITH KEY prod_id = '000001'.

IF sy-subrc = 0.
  WRITE: / <ls_prod>-name, <ls_prod>-price.
  <ls_prod>-price = '9.99'.    " modifies the table row directly — no MODIFY needed
ENDIF.
```

---

## 6. LOOP AT WHERE — Filtered Iteration
*Iterating only over rows matching a condition*

Instead of looping all rows and using `IF` to skip unwanted ones, `LOOP AT ... WHERE` is both more readable and potentially more efficient — the runtime can skip non-matching rows without entering your loop body.

**Example 1 — Simple single-condition filter**

```abap
" Example 1: Display only electronics
" Use case: process a subset of a table without splitting it into multiple tables

WRITE: / '--- Electronics Catalog ---'.
LOOP AT lt_products INTO ls_product
  WHERE category = 'ELEC'.
  WRITE: / ls_product-prod_id, ls_product-name, ls_product-price.
ENDLOOP.

" Count electronics without looping manually
DATA lv_elec_count TYPE i.
LOOP AT lt_products INTO ls_product WHERE category = 'ELEC'.
  lv_elec_count = lv_elec_count + 1.
ENDLOOP.
WRITE: / 'Electronics count:', lv_elec_count.
```

**Example 2 — Multi-condition filter with computed threshold**

```abap
" Example 2: Find high-value, low-stock items that need reorder
" Use case: business logic that operates on a subset — report on items needing attention

DATA lv_reorder_threshold TYPE i VALUE 50.
DATA lv_min_price         TYPE p DECIMALS 2 VALUE '20.00'.

WRITE: / '--- Reorder Alert ---'.
WRITE: / 'Items with stock <', lv_reorder_threshold, 'and price >', lv_min_price.
ULINE.

LOOP AT lt_products INTO ls_product
  WHERE stock_qty < lv_reorder_threshold
  AND   price     > lv_min_price.

  WRITE: / ls_product-prod_id,
           ls_product-name,
    30     ls_product-stock_qty,
    40     ls_product-price.
ENDLOOP.

" Nested LOOP WHERE — process categories in sequence
DATA: lt_categories TYPE TABLE OF c LENGTH 4,
      lv_cat        TYPE c LENGTH 4.

APPEND 'ELEC' TO lt_categories.
APPEND 'BOOK' TO lt_categories.
APPEND 'FURN' TO lt_categories.

LOOP AT lt_categories INTO lv_cat.
  WRITE: / '--- Category:', lv_cat, '---'.
  LOOP AT lt_products INTO ls_product WHERE category = lv_cat.
    WRITE: / ls_product-name, ls_product-price.
  ENDLOOP.
ENDLOOP.
```

---

## 7. FILTER — Functional Table Filtering (Modern ABAP)
*Returns a new table containing only matching rows — no loop, no mutation*

`FILTER` is a modern ABAP (7.4+) operator that creates a **new internal table** containing only rows that match a condition. Unlike `LOOP AT WHERE` (which iterates a table), `FILTER` is a **functional expression** — it returns a value (a new table) without side effects.

> **Key distinction**: `LOOP AT WHERE` iterates the original table selectively. `FILTER` creates an entirely new table. The original table is unchanged.

**Example 1 — FILTER into a new typed table**

```abap
" Example 1: Create a sub-table of just electronics
" Use case: pass a filtered subset to a subroutine/method without modifying the original

DATA lt_electronics TYPE TABLE OF ty_s_product.

lt_electronics = FILTER #( lt_products USING WHERE category = 'ELEC' ).

WRITE: / 'All products:  ', lines( lt_products ).
WRITE: / 'Electronics:   ', lines( lt_electronics ).
" lt_products is UNCHANGED — FILTER is non-destructive

LOOP AT lt_electronics INTO ls_product.
  WRITE: / ls_product-name, ls_product-price.
ENDLOOP.
```

**Example 2 — FILTER with a key table (FILTER IN)**

For the most performant filtering, FILTER can match against another table of keys:

```abap
" Example 2: Filter products to only those in an approved category list
" Use case: you have a separate 'approved categories' table and want the intersection

" The table being filtered MUST be a SORTED TABLE or HASHED TABLE with the filter key
DATA lt_sorted_products TYPE SORTED TABLE OF ty_s_product
     WITH NON-UNIQUE KEY category prod_id.

" Populate the sorted table from lt_products
lt_sorted_products = CORRESPONDING #( lt_products ).

" Build the approved categories table (must be HASHED or SORTED with UNIQUE KEY)
TYPES: BEGIN OF ty_s_cat,
         category TYPE c LENGTH 4,
       END OF ty_s_cat.
DATA lt_approved TYPE HASHED TABLE OF ty_s_cat
     WITH UNIQUE KEY category.

INSERT VALUE #( category = 'ELEC' ) INTO TABLE lt_approved.
INSERT VALUE #( category = 'BOOK' ) INTO TABLE lt_approved.

" FILTER IN — keep only rows whose category appears in lt_approved
DATA lt_filtered TYPE TABLE OF ty_s_product.
lt_filtered = FILTER #( lt_sorted_products IN lt_approved
                        WHERE category = category ).

WRITE: / 'Approved products:', lines( lt_filtered ).
LOOP AT lt_filtered INTO ls_product.
  WRITE: / ls_product-category, ls_product-name.
ENDLOOP.
```

---

## 8. SORT — Ordering Rows
*Sorting an internal table by one or more fields*

`SORT` reorders the rows of an internal table in place. It is equivalent to `ORDER BY` in SQL, but applied to data already in memory.

**Example 1 — Single field sort, both directions**

```abap
" Example 1: Sort products by price for a "best value" display
" Use case: ordered output for reports, or pre-sorting before BINARY SEARCH

" Sort cheapest first
SORT lt_products BY price ASCENDING.

WRITE: / '--- Products by Price (Cheapest First) ---'.
LOOP AT lt_products INTO ls_product.
  WRITE: / ls_product-price, ls_product-name.
ENDLOOP.

" Sort most expensive first
SORT lt_products BY price DESCENDING.

WRITE: / '--- Products by Price (Most Expensive First) ---'.
LOOP AT lt_products INTO ls_product.
  WRITE: / ls_product-price, ls_product-name.
ENDLOOP.
```

**Example 2 — Multi-field sort (primary + secondary key)**

```abap
" Example 2: Sort by category first, then by price descending within each category
" Use case: grouped display — show each category's most expensive items first

SORT lt_products BY category ASCENDING
                   price     DESCENDING.

DATA lv_prev_cat TYPE c LENGTH 4.

WRITE: / '--- Products by Category, Expensive First ---'.
LOOP AT lt_products INTO ls_product.
  " Print a category header when the category changes
  IF ls_product-category <> lv_prev_cat.
    SKIP.
    WRITE: / '[ Category:', ls_product-category, ']'.
    lv_prev_cat = ls_product-category.
  ENDIF.
  WRITE: / '  ', ls_product-name, ls_product-price.
ENDLOOP.
```

---

## 9. COLLECT — Aggregation Without Loops
*Sum numeric fields grouped by key fields automatically*

`COLLECT` is one of ABAP's most powerful and unique statements. It works like a grouped SUM in SQL (`SELECT key, SUM(amount) GROUP BY key`), but applied to an internal table being built up row by row.

**How it works**: When you `COLLECT ls_row TO lt_table`, ABAP checks if a row with the same **non-numeric key fields** already exists in the table.
- If it **does NOT exist**: the row is inserted normally (like APPEND)
- If it **DOES exist**: the **numeric fields** of the existing row are increased by the values in `ls_row`

No duplicates accumulate. The table always contains one aggregated row per unique key combination.

**Example 1 — Sales totals by product category**

```abap
" Example 1: Sum total revenue per product category
" Use case: aggregated reporting — total sales per category, per region, etc.

TYPES: BEGIN OF ty_s_cat_total,
         category    TYPE c LENGTH 4,    " KEY field (non-numeric)
         total_value TYPE p DECIMALS 2,  " NUMERIC — will be summed
         total_stock TYPE i,             " NUMERIC — will be summed
       END OF ty_s_cat_total.

DATA: lt_totals  TYPE TABLE OF ty_s_cat_total,
      ls_total   TYPE ty_s_cat_total.

" Process each product and aggregate into category totals
LOOP AT lt_products INTO ls_product.
  CLEAR ls_total.
  ls_total-category    = ls_product-category.
  " Calculate stock value for this product
  ls_total-total_value = ls_product-price * ls_product-stock_qty.
  ls_total-total_stock = ls_product-stock_qty.

  COLLECT ls_total INTO lt_totals.
  " If category already exists: total_value and total_stock are ADDED to existing row
  " If category is new: row is created fresh
ENDLOOP.

WRITE: / '--- Category Totals ---'.
LOOP AT lt_totals INTO ls_total.
  WRITE: / ls_total-category,
    15    ls_total-total_stock, 'units',
    30    ls_total-total_value, 'value'.
ENDLOOP.
" Output (example):
" BOOK         208 units     16,655.36 value
" ELEC         700 units     23,996.00 value
" FURN          45 units     13,455.00 value
```

**Example 2 — Monthly order totals (real-world billing scenario)**

```abap
" Example 2: Summarize billing data by customer and month
" Use case: invoice summary report — one row per customer per month

TYPES: BEGIN OF ty_s_order,
         customer  TYPE kunnr,
         order_mon TYPE numc6,       " YYYYMM format — KEY
         net_value TYPE p DECIMALS 2, " NUMERIC — summed
         tax_value TYPE p DECIMALS 2, " NUMERIC — summed
         item_count TYPE i,           " NUMERIC — summed (count of items)
       END OF ty_s_order.

DATA: lt_raw_orders   TYPE TABLE OF ty_s_order,
      lt_monthly_totals TYPE TABLE OF ty_s_order,
      ls_order        TYPE ty_s_order.

" Simulate raw order data (in reality, fetched from VBAK/VBAP)
ls_order-customer = 'CUST001'. ls_order-order_mon = '202401'.
ls_order-net_value = '500.00'. ls_order-tax_value = '95.00'. ls_order-item_count = 3.
APPEND ls_order TO lt_raw_orders.

ls_order-customer = 'CUST001'. ls_order-order_mon = '202401'.
ls_order-net_value = '250.00'. ls_order-tax_value = '47.50'. ls_order-item_count = 1.
APPEND ls_order TO lt_raw_orders.    " Same customer, same month → will be COLLECTED

ls_order-customer = 'CUST001'. ls_order-order_mon = '202402'.
ls_order-net_value = '800.00'. ls_order-tax_value = '152.00'. ls_order-item_count = 5.
APPEND ls_order TO lt_raw_orders.    " Same customer, different month → new row

ls_order-customer = 'CUST002'. ls_order-order_mon = '202401'.
ls_order-net_value = '1200.00'. ls_order-tax_value = '228.00'. ls_order-item_count = 8.
APPEND ls_order TO lt_raw_orders.    " Different customer → new row

" COLLECT aggregates raw orders into monthly summaries
LOOP AT lt_raw_orders INTO ls_order.
  COLLECT ls_order INTO lt_monthly_totals.
ENDLOOP.

WRITE: / '--- Monthly Billing Summary ---'.
WRITE: / 'Customer    Month   Net Value  Tax Value  Items'.
ULINE.
LOOP AT lt_monthly_totals INTO ls_order.
  WRITE: / ls_order-customer, ls_order-order_mon,
           ls_order-net_value, ls_order-tax_value, ls_order-item_count.
ENDLOOP.
" Expected output (4 raw orders → 3 aggregated rows):
" CUST001   202401   750.00   142.50    4   ← row 1+2 merged
" CUST001   202402   800.00   152.00    5
" CUST002   202401  1200.00   228.00    8
```

---

## 10. DELETE ADJACENT DUPLICATES — Deduplication
*Remove consecutive duplicate rows after sorting*

`DELETE ADJACENT DUPLICATES` removes duplicate rows from an internal table, but only consecutive ones. This means you **must SORT first** to bring duplicates together. It is ABAP's equivalent of SQL's `DISTINCT`.

**Example 1 — Remove exact duplicate rows**

```abap
" Example 1: Remove duplicate material numbers from a collected key list
" Use case: you've built a list of material numbers from multiple sources
"           and want a unique list for a FOR ALL ENTRIES query

TYPES: BEGIN OF ty_s_matnr,
         matnr TYPE mara-matnr,
       END OF ty_s_matnr.

DATA: lt_matnr_list TYPE TABLE OF ty_s_matnr,
      ls_matnr      TYPE ty_s_matnr.

" Simulate building a list with duplicates from two data sources
ls_matnr-matnr = '000000000000001234'. APPEND ls_matnr TO lt_matnr_list.
ls_matnr-matnr = '000000000000005678'. APPEND ls_matnr TO lt_matnr_list.
ls_matnr-matnr = '000000000000001234'. APPEND ls_matnr TO lt_matnr_list. " duplicate
ls_matnr-matnr = '000000000000009999'. APPEND ls_matnr TO lt_matnr_list.
ls_matnr-matnr = '000000000000005678'. APPEND ls_matnr TO lt_matnr_list. " duplicate

WRITE: / 'Before dedup:', lines( lt_matnr_list ), 'rows'.  " 5

" Step 1: SORT — brings duplicates together (required before DELETE ADJACENT)
SORT lt_matnr_list BY matnr.

" Step 2: DELETE ADJACENT DUPLICATES — removes consecutive identical rows
DELETE ADJACENT DUPLICATES FROM lt_matnr_list COMPARING matnr.

WRITE: / 'After dedup: ', lines( lt_matnr_list ), 'rows'.  " 3
```

**Example 2 — Deduplication on a specific subset of fields**

```abap
" Example 2: Remove duplicate category+customer combinations
" Use case: build a unique list of customer-category pairs for a report header

TYPES: BEGIN OF ty_s_cust_cat,
         customer TYPE kunnr,
         category TYPE c LENGTH 4,
         amount   TYPE p DECIMALS 2,   " we don't care about uniqueness here
       END OF ty_s_cust_cat.

DATA: lt_cust_cat TYPE TABLE OF ty_s_cust_cat,
      ls_cc       TYPE ty_s_cust_cat.

" Simulate data with duplicate customer+category combos
ls_cc-customer = 'CUST001'. ls_cc-category = 'ELEC'. ls_cc-amount = '100.00'. APPEND ls_cc TO lt_cust_cat.
ls_cc-customer = 'CUST001'. ls_cc-category = 'BOOK'. ls_cc-amount = '50.00'.  APPEND ls_cc TO lt_cust_cat.
ls_cc-customer = 'CUST001'. ls_cc-category = 'ELEC'. ls_cc-amount = '200.00'. APPEND ls_cc TO lt_cust_cat. " dup pair
ls_cc-customer = 'CUST002'. ls_cc-category = 'ELEC'. ls_cc-amount = '75.00'.  APPEND ls_cc TO lt_cust_cat.
ls_cc-customer = 'CUST001'. ls_cc-category = 'BOOK'. ls_cc-amount = '25.00'.  APPEND ls_cc TO lt_cust_cat. " dup pair

WRITE: / 'Before:', lines( lt_cust_cat ), 'rows'.  " 5

" Sort on ONLY the fields you want to deduplicate
SORT lt_cust_cat BY customer category.

" Compare ONLY customer and category — amount differences are ignored
DELETE ADJACENT DUPLICATES FROM lt_cust_cat COMPARING customer category.

WRITE: / 'After: ', lines( lt_cust_cat ), 'rows'.  " 3 unique customer+category pairs

" Standard pattern: SORT + DELETE ADJACENT DUPLICATES COMPARING ALL FIELDS
" (when you want truly identical rows removed)
DELETE ADJACENT DUPLICATES FROM lt_cust_cat COMPARING ALL FIELDS.
```

> **The SORT-first rule is absolute.** Without it, only consecutive duplicates are removed — if two identical rows are separated by any different row, both survive. Always SORT on the same fields you use in COMPARING.

---

## 11. In-Memory SQL — SELECT on Internal Tables
*Treating internal tables as queryable data sources*

Modern ABAP (7.4+) allows you to write a `SELECT` statement where the **source is an internal table** instead of a database table. The ABAP SQL engine processes the query entirely in memory — no database round-trip.

Syntax: prefix the table variable with `@` to denote it as an ABAP host variable (data in memory, not a DB object).

This is powerful because: you can write complex aggregations, filters, and projections using familiar SQL syntax on data already in memory — without manually writing loops and accumulators.

**Example 1 — SELECT with GROUP BY and aggregation on an internal table**

```abap
" Example 1: Compute average price and total stock per category
" Use case: when you have a large in-memory dataset and need aggregated statistics
"           without writing a COLLECT loop manually

" We already have lt_products in memory — no DB needed
" Now run SQL-style aggregation directly on it

TYPES: BEGIN OF ty_s_cat_stat,
         category  TYPE c LENGTH 4,
         avg_price TYPE p DECIMALS 2,
         max_price TYPE p DECIMALS 2,
         min_price TYPE p DECIMALS 2,
         tot_stock TYPE i,
         prod_count TYPE i,
       END OF ty_s_cat_stat.

DATA lt_cat_stats TYPE TABLE OF ty_s_cat_stat.

SELECT category,
       AVG( price )     AS avg_price,
       MAX( price )     AS max_price,
       MIN( price )     AS min_price,
       SUM( stock_qty ) AS tot_stock,
       COUNT( * )       AS prod_count
  FROM @lt_products AS lp           " @ prefix = in-memory internal table
  GROUP BY category
  INTO TABLE @lt_cat_stats.

WRITE: / '--- Category Statistics ---'.
WRITE: / 'Category  Avg Price  Max Price  Min Price  Stock  Count'.
ULINE.
LOOP AT lt_cat_stats INTO DATA(ls_stat).
  WRITE: / ls_stat-category,
    12    ls_stat-avg_price,
    24    ls_stat-max_price,
    36    ls_stat-min_price,
    48    ls_stat-tot_stock,
    55    ls_stat-prod_count.
ENDLOOP.
```

**Example 2 — SELECT with WHERE and ORDER BY on an internal table**

```abap
" Example 2: Filter and sort an in-memory table using SQL syntax
" Use case: complex filtering with multiple conditions more readable as SQL than as LOOP WHERE

TYPES: BEGIN OF ty_s_premium,
         prod_id  TYPE numc6,
         name     TYPE c LENGTH 30,
         price    TYPE p DECIMALS 2,
         category TYPE c LENGTH 4,
       END OF ty_s_premium.

DATA lt_premium TYPE TABLE OF ty_s_premium.

" Find top 3 most expensive products that are in stock
SELECT prod_id, name, price, category
  FROM @lt_products AS lp
  WHERE stock_qty > 0
  AND   price > '20.00'
  ORDER BY price DESCENDING
  INTO TABLE @lt_premium
  UP TO 3 ROWS.        " TOP 3 — ABAP's equivalent of SQL LIMIT/TOP

WRITE: / '--- Top 3 Premium In-Stock Products ---'.
LOOP AT lt_premium INTO DATA(ls_p).
  WRITE: / ls_p-name, ls_p-price, ls_p-category.
ENDLOOP.
```

---

## 12. In-Memory JOINs Between Internal Tables
*Joining two internal tables like SQL JOIN — no database call*

You can JOIN two internal tables using ABAP SQL. Both source tables are prefixed with `@`. The join logic is identical to SQL JOINs — INNER JOIN returns only matched rows, LEFT OUTER JOIN returns all rows from the left table.

**Example 1 — INNER JOIN two internal tables to combine data**

```abap
" Example 1: Join products with a separate discounts table
" Use case: enrichment — combine product data with a separately loaded discount rules table

TYPES: BEGIN OF ty_s_discount,
         category   TYPE c LENGTH 4,
         disc_pct   TYPE p DECIMALS 2,   " discount percentage
         valid_from TYPE d,
       END OF ty_s_discount.

DATA: lt_discounts TYPE TABLE OF ty_s_discount,
      ls_disc      TYPE ty_s_discount.

" Build discount table (in reality, fetched from a customizing table)
ls_disc-category = 'ELEC'. ls_disc-disc_pct = '10.00'. ls_disc-valid_from = '20240101'. APPEND ls_disc TO lt_discounts.
ls_disc-category = 'BOOK'. ls_disc-disc_pct = '15.00'. ls_disc-valid_from = '20240101'. APPEND ls_disc TO lt_discounts.
" Note: FURN has no discount — will be excluded by INNER JOIN

TYPES: BEGIN OF ty_s_discounted_product,
         prod_id    TYPE numc6,
         name       TYPE c LENGTH 30,
         orig_price TYPE p DECIMALS 2,
         disc_pct   TYPE p DECIMALS 2,
         sale_price TYPE p DECIMALS 2,
       END OF ty_s_discounted_product.

DATA lt_sale_items TYPE TABLE OF ty_s_discounted_product.

" INNER JOIN: only products whose category has a discount are returned
SELECT lp~prod_id,
       lp~name,
       lp~price      AS orig_price,
       ld~disc_pct,
       lp~price * ( 1 - ld~disc_pct / 100 ) AS sale_price
  FROM @lt_products  AS lp
  INNER JOIN @lt_discounts AS ld
    ON ld~category = lp~category
  INTO TABLE @lt_sale_items.

WRITE: / '--- Sale Items (INNER JOIN result — FURN excluded) ---'.
LOOP AT lt_sale_items INTO DATA(ls_sale).
  WRITE: / ls_sale-name,
    30    ls_sale-orig_price, '→',
    45    ls_sale-sale_price.
ENDLOOP.
```

**Example 2 — LEFT OUTER JOIN to keep all products, show NULL for missing discount**

```abap
" Example 2: LEFT OUTER JOIN — show ALL products, with discount where available
" Use case: full product list with optional discount info — don't exclude products without discounts

DATA lt_all_with_disc TYPE TABLE OF ty_s_discounted_product.

SELECT lp~prod_id,
       lp~name,
       lp~price               AS orig_price,
       COALESCE( ld~disc_pct, '0.00' ) AS disc_pct,     " 0% if no discount
       lp~price * ( 1 - COALESCE( ld~disc_pct, '0.00' ) / 100 ) AS sale_price
  FROM @lt_products  AS lp
  LEFT OUTER JOIN @lt_discounts AS ld
    ON ld~category = lp~category
  ORDER BY lp~category, lp~price DESCENDING
  INTO TABLE @lt_all_with_disc.

WRITE: / '--- All Products with Discount Info (LEFT OUTER JOIN) ---'.
LOOP AT lt_all_with_disc INTO DATA(ls_item).
  IF ls_item-disc_pct = '0.00'.
    WRITE: / ls_item-name, ls_item-orig_price, '(no discount)'.
  ELSE.
    WRITE: / ls_item-name, ls_item-orig_price, '->', ls_item-sale_price.
  ENDIF.
ENDLOOP.
```

---

## 13. Application Layer Processing — Execution Architecture

Understanding WHERE your ABAP code actually runs is critical for understanding:
- Why internal tables are temporary (and what "temporary" means exactly)
- Why too much data in memory causes performance problems
- How to reason about concurrency and shared state

### The Three-Tier Reminder

```
Presentation Layer   →  SAP GUI / Fiori browser
Application Layer    →  ABAP Application Server (AS ABAP)
Database Layer       →  Oracle / HANA / MSSQL
```

Your ABAP program lives and runs entirely in the **Application Layer**. It reads from and writes to the **Database Layer** via Open SQL. The user sees results via the **Presentation Layer**.

### What Happens When You Run a Program (F8)

```
User presses F8 in SAP GUI
         │
         ▼ (DIAG protocol over network)
Application Server receives request
         │
         ▼
Dispatcher (the AS router) finds a free DIALOG WORK PROCESS
         │
         ▼
Your program is loaded into that Work Process
User context (session data) is loaded from the roll area into the work process memory
         │
         ▼
Program executes: ABAP statements run, internal tables live in the work process's memory
Database calls go out to the DB layer and results come back
         │
         ▼
Program finishes → results sent to GUI → work process is freed for next user
User context (including any SET PARAMETER values) saved back to roll area
```

---

## 14. Work Processes, User Context, and Roll Area

### Work Processes — The Execution Units

A **Work Process** (WP) is a single-threaded OS process on the application server that runs ABAP code. The AS ABAP has a **fixed pool** of work processes (configured by the system administrator). Multiple users share this pool.

Types of work processes:

| Type | Code | Purpose |
|---|---|---|
| **Dialog** | DIA | Interactive user requests (what your F8 program uses) |
| **Background** | BGD | Scheduled batch jobs (long-running, no GUI) |
| **Update** | UPD | Asynchronous database updates (V1 and V2 updates) |
| **Spool** | SPO | Print output and spool management |
| **Enqueue** | ENQ | Lock management — serializes concurrent access |
| **Message Server** | MSG | Load balancing between application servers |

**Key constraint**: If all Dialog WPs are busy (e.g., 100 users all running heavy reports simultaneously), the next user must **wait** until a WP is freed. This is why long-running reports should be scheduled as Background jobs — they use Background WPs and don't block Dialog WPs used by interactive users.

### The Roll Area — User Context Persistence

A Work Process is stateless — it serves one user request, then is handed to the next user. But the user's "state" (which program they're in, what parameters they set, what SAP Memory PIDs contain) must persist between requests.

This is the **Roll Area** (also called the User Context):

```
APPLICATION SERVER MEMORY
─────────────────────────────────────────────────────────
Work Process Pool (e.g., 40 Dialog WPs)
  ├── WP-01: [currently executing User A's report]
  ├── WP-02: [currently executing User B's query]
  ├── WP-03: [free]
  └── ...

Roll Area (one per logged-in user, stored in shared memory/paging file)
  ├── User A's Roll Area: { ABAP Memory, program state, stack frame }
  ├── User B's Roll Area: { ABAP Memory, program state, stack frame }
  └── User C's Roll Area: { ABAP Memory, program state, stack frame }
─────────────────────────────────────────────────────────
```

When a Work Process picks up User A's request:
1. **Roll-In**: User A's Roll Area is loaded into the Work Process memory
2. The program executes (internal tables exist in WP memory here)
3. **Roll-Out**: User A's Roll Area is saved back, WP is freed

This is where **ABAP Memory lives** — in the Roll Area. It persists between your program's execution steps because it's saved and restored with the roll area.

### Internal Tables Live in Application Server Memory

When your program runs `SELECT * FROM mara INTO TABLE lt_materials`, the returned rows are stored in:

1. **The Work Process's allocated heap memory** — during program execution
2. **Via the Roll Area** — if the data needs to persist across LUW (Logical Unit of Work) boundaries

This means:
- Internal tables are **per-user, per-session** — User A's `lt_materials` is completely separate from User B's even if they run the same program simultaneously
- They are **temporary** — when the program ends and the Work Process is freed, the memory is released. Nothing persists to disk unless you explicitly write to the database or ABAP Memory
- They are **bounded by memory** — a Work Process has a maximum memory size (configurable, typically 1-4 GB). Selecting 50 million rows into an internal table will crash the Work Process with a memory overflow dump (short dump)

### Practical Implications for Developers

**Why you must use WHERE clauses**: Without filtering, you load entire DB tables into the WP memory. The MARA table in a production SAP system might have 2 million rows. Loading all of them (`SELECT * FROM mara INTO TABLE lt_mara`) wastes memory and potentially causes a dump.

**Why background jobs exist**: A report that processes a month's worth of orders might take 30 minutes. Running this in Dialog WP blocks that WP for 30 minutes (and the SAP GUI times out after 600 seconds anyway). Background jobs run in BGD WPs with no timeout, no GUI dependency, and don't consume Dialog WPs.

**Why ABAP Memory is session-scoped**: The Roll Area belongs to one user session. This is why ABAP Memory only works within one session — it's literally stored in that session's Roll Area.

### Memory Limits and ABAP Short Dumps

If your program tries to allocate more memory than the WP allows, the ABAP runtime throws a **short dump** (also called an ABAP dump or ABAP exception dump):

- Transaction `ST22` — Short Dump Analysis — shows all recent dumps with full stack traces, variable contents at the time of the dump, and the exact line that caused it
- Common memory-related short dumps: `STORAGE_PARAMETERS_WRONG_SET`, `SYSTEM_NO_ROLL`, `DYNPRO_AREA_TOO_LARGE`

---

## 15. Quick Syntax Reference Card

```
╔══════════════════════════════════════════════════════════════════════════════╗
║              ABAP INTERNAL TABLE DML — QUICK REFERENCE                      ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  ADDING ROWS                                                                 ║
║  APPEND ls_row TO lt_tab.                  " add to end                      ║
║  APPEND VALUE #( f1 = v1 f2 = v2 ) TO lt. " inline add                      ║
║  INSERT ls_row INTO lt_tab INDEX n.        " add at position n               ║
║  INSERT VALUE #( ... ) INTO TABLE lt_hash. " for sorted/hashed               ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  READING ROWS                                                                ║
║  READ TABLE lt INTO ls WITH KEY f = v.     " find one row (linear)           ║
║  READ TABLE lt INTO ls WITH KEY f = v                                        ║
║    BINARY SEARCH.                          " find one row (fast, needs SORT) ║
║  READ TABLE lt ASSIGNING <fs> WITH KEY f=v " read by reference (no copy)    ║
║  READ TABLE lt INDEX n INTO ls.            " read by position                ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  LOOPING                                                                     ║
║  LOOP AT lt INTO ls.                       " all rows                        ║
║  LOOP AT lt INTO ls WHERE f = v.           " filtered rows only              ║
║  LOOP AT lt ASSIGNING <fs>.                " by reference (efficient)        ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  MODIFYING ROWS                                                              ║
║  MODIFY lt FROM ls.                        " update current row (in loop)    ║
║  MODIFY lt FROM ls INDEX n.                " update row at position n        ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  DELETING ROWS                                                               ║
║  DELETE lt WHERE f = v.                    " bulk delete by condition        ║
║  DELETE lt.                                " delete current row (in loop)    ║
║  DELETE lt INDEX n.                        " delete row at position n        ║
║  CLEAR lt.                                 " empty the entire table          ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  SORTING & DEDUPLICATION                                                     ║
║  SORT lt BY f1 ASCENDING f2 DESCENDING.    " multi-field sort                ║
║  DELETE ADJACENT DUPLICATES FROM lt                                          ║
║    COMPARING f1 f2.                        " dedup on specific fields        ║
║  DELETE ADJACENT DUPLICATES FROM lt                                          ║
║    COMPARING ALL FIELDS.                   " dedup on all fields             ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  AGGREGATION                                                                 ║
║  COLLECT ls INTO lt.                       " sum numerics, group by chars    ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  FUNCTIONAL / MODERN                                                         ║
║  lt2 = FILTER #( lt1 USING WHERE f = v ).  " new filtered table             ║
║  lines( lt )                               " row count function              ║
║  DESCRIBE TABLE lt LINES lv_n.             " row count (classic)             ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  IN-MEMORY SQL (Modern ABAP 7.4+)                                            ║
║  SELECT f1 f2 FROM @lt1 AS a               " query internal table directly  ║
║    WHERE f1 = v                                                              ║
║    GROUP BY f1 INTO TABLE @lt2.                                              ║
║  SELECT a~f1 b~f2                          " JOIN two internal tables        ║
║    FROM @lt1 AS a INNER JOIN @lt2 AS b                                       ║
║    ON b~key = a~key                                                          ║
║    INTO TABLE @lt3.                                                          ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  MEMORY                                                                      ║
║  EXPORT var TO MEMORY ID 'KEY'.            " write to ABAP memory           ║
║  IMPORT var FROM MEMORY ID 'KEY'.          " read from ABAP memory          ║
║  FREE MEMORY ID 'KEY'.                     " release ABAP memory            ║
║  SET PARAMETER ID 'PID' FIELD var.         " write to SAP memory            ║
║  GET PARAMETER ID 'PID' FIELD var.         " read from SAP memory           ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

*Study notes on ABAP DML, Internal Table Operations, and Application Layer Architecture*
*Cross-reference: SAP_ABAP_Video3_StudyNotes.md (SELECT patterns, joins, debugging)*
*Cross-reference: ABAP_DDIC_Tutorial.md (data types, internal table types, syntax basics)*
