# SAP ABAP Study Notes — Video 2 of 11
> Based on: "FREE Video 2 of 11 - Learn SAP ABAP for Free for Freshers & Functional Consultants" by ZAP Yard
> Video: https://youtu.be/tOS0knSJ98g
> Cross-reference: SAP_ABAP_Video1_StudyNotes.md | ABAP_DDIC_Tutorial.md

---

## Table of Contents

1. [Quiz Recap — Video 1 Concepts](#1-quiz-recap)
2. [Agenda for Video 2](#2-agenda)
3. [SE38 — The ABAP Editor](#3-se38-the-abap-editor)
4. [Program Types — Executable Programs](#4-program-types)
5. [DATA Keyword — Variables & Types](#5-data-keyword--variables--types)
6. [WRITE Statement & Output Formatting](#6-write-statement--output-formatting)
7. [System Fields (sy-*)](#7-system-fields-sy-)
8. [Date Handling & Formatting](#8-date-handling--formatting)
9. [TYPES Keyword — Local Type Definitions](#9-types-keyword--local-type-definitions)
10. [Arithmetic Operators & Integer Division](#10-arithmetic-operators--integer-division)
11. [Local Structures in ABAP (BEGIN OF / END OF)](#11-local-structures-in-abap)
12. [Work Areas](#12-work-areas)
13. [CONSTANTS Keyword](#13-constants-keyword)
14. [SELECT SINGLE — Fetching One Record](#14-select-single)
15. [Selection Screens — Complete Guide](#15-selection-screens--complete-guide)
16. [Text Elements — Externalising Hardcoded Text](#16-text-elements)
17. [ABAP Program Events — Complete Lifecycle](#17-abap-program-events--complete-lifecycle)
18. [Subroutines — PERFORM / FORM / ENDFORM](#18-subroutines--perform--form--endform)
19. [Where-Used List](#19-where-used-list)
20. [Homework Assignments](#20-homework-assignments)
21. [Key Takeaways & Revision Questions](#21-key-takeaways--revision-questions)

---

## 1. Quiz Recap
*[Video: ~00:06]*

The session opens with a quiz on Video 1 material. Key concepts revisited:

- **DDIC objects**: Domain → Data Element → Table/Structure. Understanding what each layer adds (technical vs semantic)
- **Nested structures**: A structure can include another structure as a component. E.g., an `ADDRESS` structure embedded inside a `CUSTOMER` structure. You then access fields as `ls_customer-address-street`
- **Primary keys**: The set of fields that uniquely identify a row. In SAP, MANDT is always the first key field for client-dependent tables
- **Table indexing**: Primary key fields are automatically indexed by the DB. Secondary indexes can be created in SE11 under the Indexes tab for frequently queried non-key fields
- **Type groups**: An older mechanism (TYPE-POOLS) for sharing type definitions across programs. Less common now since DDIC types are preferred

These quiz points highlight what you should already be solid on before proceeding. If any of these feel uncertain, revisit Video 1 notes first.

---

## 2. Agenda
*[Video: ~09:22]*

Video 2 covers the practical ABAP programming fundamentals:

1. Creating ABAP reports in SE38
2. Data types and variable declaration
3. Designing selection screens
4. ABAP program events and their order
5. Strings and string operations
6. Internal tables (introduced conceptually)
7. Basic SELECT queries

This video is where you transition from DDIC concepts (metadata/schema) to **actual ABAP code** — writing programs that do something.

---

## 3. SE38 — The ABAP Editor
*[Video: ~18:07]*

**Transaction SE38** is the primary editor for writing ABAP programs. Think of it as a code editor with SAP awareness: syntax highlighting, where-used lookups, code activation, and execution all in one place.

> Modern SAP also offers **SE80** (Object Navigator) which is more like a project explorer/IDE. SE38 is the focused single-program editor. Both access the same programs; SE80 just gives a tree view of all objects in a package.

### Opening SE38

1. Type `SE38` in the command field → Enter
2. Enter your program name (e.g., `ZTEST_REPORT`) in the Program field
3. Click **Create** (or just press Enter if program exists to open it)

### Key Actions in SE38

| Button / Menu | Shortcut | Purpose |
|---|---|---|
| Activate | Ctrl+F3 | Compile and save the program permanently |
| Execute | F8 | Run the program |
| Check syntax | Ctrl+F2 | Check for syntax errors without activating |
| Where-used | Shift+F4 | Find all usages of a selected object |
| Pretty Printer | Shift+F1 | Auto-format and indent code |

### Naming Conventions
*[Video: ~19:04]*

- All **custom programs** must start with `Z` or `Y` — SAP reserves all other namespaces for standard objects
- Examples: `ZORDER_REPORT`, `YSALES_ANALYSIS`, `ZSD_OPEN_ORDERS`
- Use descriptive names with module prefix — `ZSD_OPEN_ORDERS` is better than `ZTEST1`
- Program names can be up to **40 characters** in modern SAP (only 8 in very old systems)

### Package Assignment & Transport Request

When you create a new program and click Save, SAP immediately asks two questions:

**"Which package?"**
- `$TMP` = local object, never transported. Fine for learning and throwaway tests.
- A real package (e.g., `ZSD_REPORTS`) = will be included in transport to QAS/PRD

**"Which Transport Request?"**
- Either create a new TR or add to an existing one
- Every change you save in a non-`$TMP` program is recorded in the TR

> This is SAP's built-in source control. Every saved change is tracked. Unlike git, you cannot have branches — there is one active version of each object in each system (DEV/QAS/PRD).

---

## 4. Program Types
*[Video: ~22:33]*

When creating a program in SE38, SAP asks for the **Program Type**. This determines how the program runs and what events are available to it.

| Type | Code | Description | When to use |
|---|---|---|---|
| **Executable Program** | `1` | Standalone program with selection screen, runs directly | Most reports and custom programs |
| **Include Program** | `I` | A code fragment — cannot run alone, included in another program | Splitting large programs into files |
| **Module Pool** | `M` | Screen-driven program (complex custom transaction screens) | Custom multi-screen transactions |
| **Function Group** | `F` | Container for Function Modules | Writing reusable Function Modules |
| **Class Pool** | `K` | Container for a global ABAP class | OOP development |
| **Interface Pool** | `J` | Container for a global ABAP interface | OOP interfaces |
| **Subroutine Pool** | `S` | Program with only FORMs, called from other programs | Legacy shared subroutine libraries |
| **Type Pool** | `T` | Program defining shared TYPES and CONSTANTS | Legacy shared type definitions |

**For this tutorial series, we focus on Executable Programs** — the most common type you will write as a developer.

An Executable Program:
- Has the header `REPORT <program_name>.`
- Has a selection screen (optional but typical)
- Is triggered by a user running it (SE38 F8, or via a T-Code linked to it, or via background job scheduler)
- Follows the event-driven lifecycle (covered in detail in Section 17)

---

## 5. DATA Keyword — Variables & Types
*[Video: ~25:22]*

### Declaring Variables

The `DATA` keyword declares variables. In ABAP, all variables must be declared before use — like C or Java, not like Python.

```abap
REPORT zmy_first_program.

" Single declaration per line
DATA lv_name    TYPE string.
DATA lv_age     TYPE i.
DATA lv_salary  TYPE p DECIMALS 2.
DATA lv_today   TYPE d.

" Chained declaration — comma-separated with colon after DATA
DATA: lv_city    TYPE string,
      lv_country TYPE string,
      lv_zip     TYPE numc5.
```

> **The colon-comma pattern**: `DATA: a TYPE i, b TYPE i.` is equivalent to `DATA a TYPE i. DATA b TYPE i.` The colon starts the chain; the comma continues it; the period ends it. This is ABAP's way of chaining the same keyword across multiple declarations.

### Core ABAP Data Types — Deep Dive

Understanding data types is fundamental. Each type has specific storage, range, and behavior implications.

**`C` — Character (Fixed-Length)**
- Fixed-length character string — you must specify length
- Declared as: `TYPE c LENGTH 10` or shorthand `TYPE c(10)`
- Padded with trailing spaces if value is shorter than defined length
- Default initial value: all spaces
- Use for: codes, IDs, fixed-format short text fields

```abap
DATA lv_code TYPE c LENGTH 4.
lv_code = 'AB'.
" Stored as 'AB  ' (padded with 2 spaces to fill the 4 chars)
" Comparisons ignore trailing spaces: lv_code = 'AB' is TRUE
```

**`STRING` — Variable-Length String**
- Dynamically sized — memory grows to fit the content
- More memory-efficient than `C` for varying-length text
- Default initial value: empty string (not spaces)
- Prefer `STRING` for user-facing messages, descriptions, dynamic text

```abap
DATA lv_message TYPE string.
lv_message = 'Hello, this can be any length without padding'.
```

**`N` — Numeric Characters (NUMC)**
- Stores digits only (0-9), but is NOT used for arithmetic
- Padded with **leading zeros**: `'42'` in a `NUMC5` field becomes `'00042'`
- Default initial value: all zeros ('00000')
- Use for: order numbers, document numbers, zip codes, item numbers — things that look like numbers but you never add or multiply them

```abap
DATA lv_order_no TYPE n LENGTH 10.
lv_order_no = 42.
" Stored as '0000000042'
WRITE lv_order_no.   " displays: 0000000042
```

**`I` — Integer**
- 4-byte signed integer
- Range: -2,147,483,648 to +2,147,483,647
- Use for: counters, loop variables, row counts, indices

```abap
DATA lv_count TYPE i.
lv_count = 100.
lv_count = lv_count + 1.   " 101
```

**`P` — Packed Decimal**
- Fixed-point number with exact decimal arithmetic
- Must specify decimal places: `TYPE p DECIMALS 2`
- Use for: **all financial and monetary calculations** — never use float for money
- Also used for quantities, rates, percentages
- Internally stored as packed BCD (Binary Coded Decimal) — very space-efficient

```abap
DATA lv_price   TYPE p DECIMALS 2.
DATA lv_tax     TYPE p DECIMALS 4.
lv_price = '199.99'.
lv_tax   = '0.0825'.   " 8.25% tax rate — exact, no floating point error
```

**`F` — Floating Point**
- 8-byte IEEE 754 double precision
- Has inherent rounding errors (like every language's float)
- Use for: scientific/engineering calculations only — NOT for money, NOT for business quantities
- In real SAP business development, you almost never use `F` type

**`D` — Date**
- Internally stored as 8 characters: `YYYYMMDD`
- But declared as type `D` — SAP treats it as a date with arithmetic support
- Default initial value: `'00000000'`
- Supports date arithmetic (addition/subtraction of days)

```abap
DATA lv_date  TYPE d.
lv_date = sy-datum.          " today, e.g. '20241215'
lv_date = lv_date + 30.      " 30 days later — ABAP handles month/year wrap
DATA lv_diff TYPE i.
lv_diff = sy-datum - '20240101'.   " number of days since Jan 1, 2024
```

**`T` — Time**
- Internally stored as 6 characters: `HHMMSS`
- Same arithmetic capability as dates

```abap
DATA lv_time TYPE t.
lv_time = sy-uzeit.   " current server time, e.g. '143022'
```

**`X` — Hexadecimal**
- Raw bytes
- Use for: binary data, checksums, BAPIs expecting raw data

**`ABAP_BOOL` — Boolean Convention**
- Not a native type — it is a DDIC type alias for `C LENGTH 1`
- Convention: `'X'` = true, `' '` (space) = false
- Use predefined constants `abap_true` and `abap_false` instead of literals

```abap
DATA lv_found TYPE abap_bool.
lv_found = abap_true.    " same as lv_found = 'X'
IF lv_found = abap_true.
  WRITE 'Found!'.
ENDIF.
```

### TYPE vs LIKE

Two ways to type a variable:

```abap
DATA lv_matnr  TYPE matnr.         " TYPE: reference a named DDIC type or element
DATA lv_matnr2 LIKE mara-matnr.    " LIKE: copy the type from an existing field or variable
```

`TYPE` is the modern and preferred approach. `LIKE` is legacy syntax from older ABAP but still extremely common in existing code you will maintain.

---

## 6. WRITE Statement & Output Formatting
*[Video: ~27:14]*

`WRITE` is the classic output statement for list-based reports. It outputs to SAP's scrollable list viewer — a text-based grid that users can scroll through and print.

### Basic WRITE Syntax

```abap
WRITE 'Hello World'.             " Output text on the current line
WRITE: / 'New line output'.      " / means: start a new line THEN write
WRITE: /5 'At column 5'.         " /5 means: new line, start at column 5
WRITE: 10 'At column 10'.        " No / means: same line, start at column 10

" Multiple values in one WRITE statement
WRITE: / 'Name:', lv_name, '  Age:', lv_age.

" You can chain WRITEs on one ABAP line
WRITE: / lv_carrid, lv_carrname, lv_currency.
```

### Formatting Options for WRITE

```abap
" Numbers
WRITE lv_amount DECIMALS 2.              " show exactly 2 decimal places
WRITE lv_price  CURRENCY lv_waers.       " locale-aware currency format (e.g. 1,234.56 USD)
WRITE lv_qty    UNIT lv_meins.           " format with unit of measure

" Dates
WRITE lv_date DD/MM/YYYY.                " explicit date format
WRITE lv_date USING EDIT MASK '__.__.____'.  " custom mask (__ = char placeholder)

" Alignment
WRITE lv_amount RIGHT-JUSTIFIED.         " right-align (default for numeric types)
WRITE lv_name   LEFT-JUSTIFIED.          " left-align (default for char types)
WRITE lv_name   CENTERED.                " center the value

" Colors for classic list output
WRITE: / lv_name COLOR COL_POSITIVE.     " green row
WRITE: / lv_name COLOR COL_NEGATIVE.     " red row
WRITE: / lv_name COLOR COL_HEADING.      " blue header color
WRITE: / lv_name COLOR COL_KEY.          " key column color
WRITE: / lv_name INTENSIFIED.            " bold/intensified display
```

### Page Structure Commands

```abap
ULINE.                " draw a full horizontal line across the page width
ULINE AT /5(40).      " draw a line starting at column 5, 40 characters wide
SKIP.                 " one blank line
SKIP 3.               " three blank lines
NEW-LINE.             " same as / — move to a new line
NEW-PAGE.             " force a page break (triggers END-OF-PAGE and TOP-OF-PAGE events)
```

> **Real-world note**: Classic `WRITE` reports are largely obsolete for new development. Modern SAP uses **ALV** (ABAP List Viewer) which provides interactive grids with built-in sort, filter, and Excel export. However, `WRITE` is still widely present in legacy systems and understanding it is important for maintenance work. It also helps you understand SAP's underlying rendering model.

---

## 7. System Fields (sy-*)
*[Video: ~30:45]*

System fields are global variables automatically populated by the SAP runtime. You always **read** them — you never write to them directly (they are set as side effects of operations).

They all live in a global structure named `SY` (alias `SYST` — both names work, `SY` is more common).

### Most Important System Fields

**`sy-subrc` — Return Code (THE most important field)**
- Set automatically after every database operation, function call, READ TABLE, OPEN DATASET, FIND, etc.
- `0` = success / found / OK
- Non-zero = error, not found, or condition not met
- **You must always check `sy-subrc` after any operation that can fail**

```abap
SELECT SINGLE * FROM kna1 INTO ls_customer WHERE kunnr = '1000'.
IF sy-subrc = 0.
  WRITE ls_customer-name1.
ELSEIF sy-subrc = 4.
  WRITE 'Customer not found'.
ENDIF.
```

Common `sy-subrc` values:

| Value | Meaning |
|---|---|
| `0` | Success / found |
| `4` | Not found / condition not met |
| `8` | Error (operation-specific) |

**`sy-datum` — System Date**
- Today's date in the SAP application server (server date, not client PC date)
- Type `D` (YYYYMMDD format internally)
- Use this rather than hardcoding dates

**`sy-uzeit` — System Time**
- Current server time
- Type `T` (HHMMSS format internally)

**`sy-uname` — Username**
- The SAP user ID of the currently logged-in user (e.g., `'JSMITH'`)
- Use for logging, audit trails, personalization

**`sy-mandt` — Current Client**
- The SAP client number (e.g., `'100'`, `'300'`)
- Never hardcode client numbers in programs — always reference `sy-mandt`

**`sy-index` — Loop Counter for DO/WHILE**
- Current iteration number inside `DO...ENDDO` or `WHILE...ENDWHILE` loops
- Starts at 1, increments by 1 each iteration

**`sy-tabix` — Internal Table Row Index**
- The row number of the current row being processed in a `LOOP AT...` statement
- Also set by `READ TABLE` — points to the row that was found
- Useful when you need to MODIFY or DELETE a specific row during a loop

**`sy-dbcnt` — Database Row Count**
- Number of rows affected by or returned from the last database operation
- After `SELECT * INTO TABLE lt_tab`, `sy-dbcnt` = number of rows fetched
- After `UPDATE table SET... WHERE...`, `sy-dbcnt` = rows updated

**`sy-repid` — Current Program Name**
- The internal name of the currently running program
- Useful for passing to Message statements and function modules that need the caller program name

**`sy-tcode` — Current Transaction Code**
- The T-Code used to launch the current program

**`sy-langu` — Language**
- The logon language of the current user (`'E'` = English, `'D'` = German, `'F'` = French)
- Used for language-dependent text lookups

**`sy-pagno` — Page Number**
- Current page number in list output (used in TOP-OF-PAGE / END-OF-PAGE events)

**`sy-msgid`, `sy-msgno`, `sy-msgv1-4` — Message Fields**
- Populated after a `MESSAGE` statement or a function module raises a message
- Useful for error handling: after calling a function, check sy-subrc and then read these fields to understand what error occurred

---

## 8. Date Handling & Formatting
*[Video: ~35:08]*

Date handling is a very frequent task in SAP business programming. Almost every report has date-range filters and date-formatted output.

### How Dates Are Stored

The `D` type stores dates as `YYYYMMDD` — 8 characters internally. So December 15, 2024 is stored as `'20241215'`. This means you can use ABAP's string offset/length operations to extract parts.

### Reading and Displaying Dates

```abap
DATA: lv_date    TYPE d,
      lv_day     TYPE c LENGTH 2,
      lv_month   TYPE c LENGTH 2,
      lv_year    TYPE c LENGTH 4,
      lv_display TYPE string.

lv_date = sy-datum.   " e.g., '20241215'

" Extract parts using ABAP string offset+length syntax: variable+offset(length)
lv_year  = lv_date(4).      " first 4 chars (offset 0, length 4): '2024'
lv_month = lv_date+4(2).    " 2 chars starting at position 4: '12'
lv_day   = lv_date+6(2).    " 2 chars starting at position 6: '15'

" Build display string manually
CONCATENATE lv_day '.' lv_month '.' lv_year INTO lv_display.
WRITE lv_display.   " outputs: 15.12.2024
```

### String Offset/Length Syntax — General Rule

This is one of the most useful and common ABAP patterns — accessing a substring of a character field:

```abap
DATA lv_text TYPE c LENGTH 10 VALUE 'ABCDEFGHIJ'.

WRITE lv_text(3).       " 'ABC'  — first 3 chars (implicit offset 0)
WRITE lv_text+3(3).     " 'DEF'  — 3 chars starting at position 3
WRITE lv_text+6(4).     " 'GHIJ' — 4 chars starting at position 6
WRITE lv_text+6.        " 'GHIJ' — from position 6 to end (no length = rest of string)
```

The syntax is: `variable+offset(length)` where offset is 0-based.

### Date Arithmetic

ABAP's `D` type supports direct arithmetic — and handles month/year boundaries correctly:

```abap
DATA: lv_today  TYPE d,
      lv_future TYPE d,
      lv_past   TYPE d,
      lv_diff   TYPE i.

lv_today  = sy-datum.
lv_future = lv_today + 90.           " 90 days from now — correct month/year wrap
lv_past   = lv_today - 30.           " 30 days ago
lv_diff   = lv_today - '20240101'.   " number of days since Jan 1, 2024
```

> ABAP handles calendar overflow automatically. Adding 90 days to March 15 gives June 13, not "March 105". Subtracting 30 days from January 10 gives December 11 of the previous year.

### WRITE Formatting for Dates

```abap
WRITE sy-datum.                               " uses user's SAP date format preference
WRITE sy-datum DD/MM/YYYY.                    " force DD/MM/YYYY regardless of user setting
WRITE sy-datum MM/DD/YYYY.                    " force MM/DD/YYYY
WRITE sy-datum USING EDIT MASK '__.__.____'.  " custom mask: DD.MM.YYYY
```

---

## 9. TYPES Keyword — Local Type Definitions
*[Video: ~44:36]*

### Why Define Local Types?

When a type is only needed within one program and does not need to be shared via DDIC, you define it locally with the `TYPES` keyword. This keeps the program self-contained and avoids polluting the DDIC with one-off types.

`TYPES` defines a **type template** — it describes a shape but allocates no memory. Memory is only allocated when you use `DATA` that references the type.

```abap
" TYPES defines a template — no memory yet, no variable exists
TYPES: ty_name    TYPE string,
       ty_amount  TYPE p DECIMALS 2,
       ty_counter TYPE i.

" DATA uses the template to create a variable — memory IS allocated
DATA: lv_product_name TYPE ty_name,
      lv_unit_price   TYPE ty_amount,
      lv_stock_count  TYPE ty_counter.
```

### Naming Convention for Types

By convention, local types are prefixed:

| Prefix | Meaning | Example |
|---|---|---|
| `ty_` | Generic local type | `ty_amount` |
| `ty_s_` | Local structure type | `ty_s_order_row` |
| `ty_t_` | Local table type | `ty_t_order_list` |

### The Critical Distinction: TYPES vs DATA

This confuses many beginners. The distinction is straightforward once you see it:

```abap
TYPES ty_age TYPE i.       " Defines a TYPE TEMPLATE named ty_age
                            " NO variable exists. NO memory allocated. Cannot be read or written.

DATA lv_age TYPE i.        " Defines a VARIABLE named lv_age
                            " Memory IS allocated. Initial value = 0. Can be read and written.

DATA lv_age2 TYPE ty_age.  " Defines a variable lv_age2 using the ty_age template
                            " Memory IS allocated. Same as declaring TYPE i directly.
```

Think of `TYPES` as equivalent to a class definition and `DATA` as equivalent to creating an instance. The `TYPES` definition is purely descriptive; `DATA` is the actual allocation.

---

## 10. Arithmetic Operators & Integer Division
*[Video: ~47:09 — 50:01]*

### Basic Arithmetic

```abap
DATA: lv_a      TYPE i VALUE 10,
      lv_b      TYPE i VALUE 3,
      lv_result TYPE i,
      lv_dec    TYPE p DECIMALS 4.

lv_result = lv_a + lv_b.    " Addition:      13
lv_result = lv_a - lv_b.    " Subtraction:   7
lv_result = lv_a * lv_b.    " Multiplication: 30
lv_result = lv_a / lv_b.    " Division:      see integer division section below
lv_result = lv_a MOD lv_b.  " Modulo/remainder: 1  (10 = 3*3 + 1)
lv_result = lv_a DIV lv_b.  " Integer division, explicitly truncated: 3
lv_result = lv_a ** 2.       " Exponentiation: 100

lv_dec    = lv_a / lv_b.    " With decimal result type: 3.3333
```

### Integer Division — The Floor vs Truncation Behavior
*[Video: ~50:01]*

This is one of the places where ABAP surprises programmers from other languages. When you divide and the **result variable is type `I` (integer)**, ABAP **truncates toward zero**:

```abap
DATA lv_int TYPE i.

lv_int = 10 / 3.    " = 3 (not 3.33) — fractional part discarded
lv_int = 7 / 2.     " = 3 (not 3.5)  — truncated
lv_int = -7 / 2.    " = -3 (not -4)  — truncated toward zero, not toward -infinity
```

Compare: floor(-7/2) = floor(-3.5) = -4, but ABAP integer division gives -3 because truncation goes toward zero.

When the **result variable is packed decimal**, you get full precision:

```abap
DATA lv_dec TYPE p DECIMALS 4.

lv_dec = 10 / 3.    " = 3.3333
lv_dec = 7 / 2.     " = 3.5000
lv_dec = -7 / 2.    " = -3.5000
```

**Practical rule**: Always match your result variable type to what you actually need. If you need fractional results for financial calculations, use `p DECIMALS n`. If you are counting discrete items (rows, iterations), `i` is fine and correct.

### The CEIL and FLOOR Distinction

The instructor distinguishes CEIL (round up) and FLOOR (round down) to explain what integer division does — it truncates, not rounds. Explicit rounding functions are available:

```abap
DATA lv_val TYPE f VALUE '3.7'.

WRITE: ceil( lv_val ).    " 4.0  — smallest integer >= value
WRITE: floor( lv_val ).   " 3.0  — largest integer <= value
WRITE: trunc( lv_val ).   " 3.0  — truncate fractional part (toward zero)
WRITE: round( lv_val ).   " 4.0  — standard rounding (half-up)
WRITE: frac( lv_val ).    " 0.7  — fractional part only
WRITE: abs( '-4.5' ).     " 4.5  — absolute value
WRITE: sign( '-4.5' ).    " -1   — sign: -1, 0, or 1
WRITE: sqrt( 16 ).        " 4.0  — square root
```

---

## 11. Local Structures in ABAP
*[Video: ~01:00:43]*

### What Is a Local Structure?

A structure is a **composite data container** — a group of named fields bundled together under one variable name. In other languages: a `struct` in C, a record in Pascal, a simple data class in Java.

A **local** structure is defined directly inside an ABAP program using the `TYPES...BEGIN OF...END OF` syntax. Unlike DDIC structures (defined in SE11), local structures exist only within that program. They cannot be referenced by other programs or DDIC objects.

### Defining a Local Structure with TYPES

```abap
TYPES: BEGIN OF ty_s_employee,
         emp_id     TYPE numc8,           " 8-digit employee number
         first_name TYPE c LENGTH 20,
         last_name  TYPE c LENGTH 30,
         salary     TYPE p DECIMALS 2,
         hire_date  TYPE d,
         department TYPE c LENGTH 10,
       END OF ty_s_employee.
```

The `BEGIN OF` and `END OF` delimit the structure. Every field between them is a **component** of the structure. The type name `ty_s_employee` refers to the entire record shape.

### Nested Structures

A structure can contain another structure as one of its components:

```abap
TYPES: BEGIN OF ty_s_address,
         street  TYPE c LENGTH 50,
         city    TYPE c LENGTH 30,
         country TYPE c LENGTH 3,
         zip     TYPE numc10,
       END OF ty_s_address.

TYPES: BEGIN OF ty_s_person,
         name    TYPE c LENGTH 40,
         age     TYPE i,
         address TYPE ty_s_address,     " embedded structure
       END OF ty_s_person.
```

Accessing nested fields uses the dash operator at each level:

```abap
DATA ls_person TYPE ty_s_person.
ls_person-name           = 'John Smith'.
ls_person-address-city    = 'Berlin'.
ls_person-address-country = 'DE'.
ls_person-address-zip     = '10115'.
```

### Using DDIC Table Types Directly

You do not always need to define your own structure. For standard SAP tables, you can directly reference the table's structure as a type:

```abap
DATA ls_carrier  TYPE scarr.    " single row shaped exactly like the SCARR table
DATA ls_customer TYPE kna1.     " single row shaped like the Customer Master table
DATA ls_material TYPE mara.     " single row shaped like the Material Master table
```

This is extremely common in real SAP code. The DDIC table definition doubles as a structure type, so there is no need to redefine fields you already know exist.

---

## 12. Work Areas
*[Video: ~01:03:52]*

### What Is a Work Area?

A **Work Area** (WA) is a variable that holds exactly **one row** of a structure. It acts as a row-level buffer — you load one row of data into it, work on its individual fields, and then either output it or move it into an internal table.

The work area is conceptually the "current record cursor" — one row at a time.

```abap
TYPES: BEGIN OF ty_s_flight,
         carrid   TYPE c LENGTH 3,
         connid   TYPE numc4,
         fldate   TYPE d,
         price    TYPE p DECIMALS 2,
         currency TYPE c LENGTH 5,
       END OF ty_s_flight.

DATA ls_flight TYPE ty_s_flight.     " Work Area — holds ONE row

" Assign values to individual fields via the dash (-) operator
ls_flight-carrid   = 'LH'.
ls_flight-connid   = '0400'.
ls_flight-fldate   = sy-datum.
ls_flight-price    = '450.00'.
ls_flight-currency = 'EUR'.

" Read individual fields back out
WRITE: / 'Carrier:', ls_flight-carrid.
WRITE: / 'Price:  ', ls_flight-price.
```

### Clearing a Work Area

Before reusing a work area (critical when processing rows in a loop), always clear it first to prevent stale field values from the previous iteration carrying over:

```abap
CLEAR ls_flight.
" Effect: all fields reset to initial values
" Strings → empty, numbers → 0, dates → '00000000', chars → spaces
```

Not clearing a work area between iterations is a very common source of bugs in ABAP, especially when some iterations set fewer fields than others.

---

## 13. CONSTANTS Keyword
*[Video: ~01:07:35]*

### Why Constants?

Constants prevent "magic literals" scattered through code. Instead of writing `'FERT'` in 10 places and having to remember what it means and hunt them all down if it changes, you write `lc_mattype_finished` once and reference it everywhere. The code becomes self-documenting and maintenance is trivial.

### Syntax

```abap
CONSTANTS: lc_tax_rate        TYPE p DECIMALS 4 VALUE '0.0825',   " 8.25% tax
           lc_max_retries     TYPE i              VALUE 3,
           lc_status_open     TYPE c LENGTH 1     VALUE 'O',
           lc_status_closed   TYPE c LENGTH 1     VALUE 'C',
           lc_status_hold     TYPE c LENGTH 1     VALUE 'H',
           lc_mattype_finish  TYPE mtart           VALUE 'FERT',
           lc_company_us      TYPE bukrs           VALUE 'US01'.
```

### Constants vs Variables

```abap
DATA      lv_counter TYPE i VALUE 0.   " Variable — initial value 0, but can be changed
CONSTANTS lc_limit   TYPE i VALUE 100. " Constant — fixed at 100 forever

lv_counter = lv_counter + 1.   " Fine — variables can change
lc_limit   = 200.              " SYNTAX ERROR at activation — constants are read-only
```

The `VALUE` keyword provides the initial/constant value. For `DATA`, it sets the initial value but the variable can change later. For `CONSTANTS`, the value is compile-time fixed.

### Naming Convention

Constants use the `lc_` prefix (local constant) or `gc_` (global constant) to distinguish them visually from variables at a glance.

---

## 14. SELECT SINGLE — Fetching One Record
*[Video: ~01:10:19]*

### The SCARR Training Table

`SCARR` is one of SAP's built-in demo/training tables that comes pre-populated with airline data. The instructor uses it because it is always available in any SAP system without needing real business data configured.

```
SCARR fields:
  MANDT    — Client
  CARRID   — Airline code (2-3 chars, e.g. 'LH' = Lufthansa, 'AA' = American Airlines)
  CARRNAME — Full airline name (e.g. 'Lufthansa')
  CURRCODE — Currency (e.g. 'EUR', 'USD')
  URL      — Airline website
```

Other SAP training tables you will encounter: `SPFLI` (flight schedule), `SFLIGHT` (individual flights), `SBOOK` (bookings). These form a mini flight-booking system used in all SAP training materials.

### SELECT SINGLE Syntax

`SELECT SINGLE` fetches exactly one row that matches the WHERE condition. It is roughly equivalent to `SELECT ... WHERE key = val LIMIT 1` in standard SQL.

```abap
DATA ls_carrier TYPE scarr.    " work area to receive the result row

SELECT SINGLE *
  FROM scarr
  INTO ls_carrier
  WHERE carrid = 'LH'.

IF sy-subrc = 0.
  " Row was found — ls_carrier is populated
  WRITE: / 'Airline:  ', ls_carrier-carrname.
  WRITE: / 'Currency: ', ls_carrier-currcode.
ELSE.
  " sy-subrc = 4 means no row matched the WHERE condition
  WRITE: / 'Airline not found'.
ENDIF.
```

### SELECT SINGLE vs SELECT (without SINGLE)

| Aspect | SELECT SINGLE | SELECT (multi-row) |
|---|---|---|
| Rows returned | Maximum 1 row | All matching rows |
| `INTO` target | Work area (one struct) | `INTO TABLE` (internal table) or work area inside ENDSELECT loop |
| `sy-subrc` after | 0 if found, 4 if not found | 0 if at least 1 row, 4 if 0 rows |
| When to use | You know the primary key, want exactly one record | You need a set of records |
| Performance | Very fast — DB uses key lookup directly | Depends on WHERE clause and indexes |

### Selecting Specific Fields — Best Practice

Rather than `SELECT *` (which fetches every column), list only the fields you actually need. This reduces network traffic and memory usage significantly, especially on wide tables (some SAP tables have 200+ columns).

```abap
SELECT SINGLE carrid carrname currcode
  FROM scarr
  INTO CORRESPONDING FIELDS OF ls_carrier   " maps columns to work area fields by name
  WHERE carrid = 'LH'.
```

`INTO CORRESPONDING FIELDS OF` maps the selected DB columns to the work area by matching field names. Fields in the work area that were not selected remain at their initial values.

---

## 15. Selection Screens — Complete Guide
*[Video: ~01:21:23 — 01:38:12]*

### What Is a Selection Screen?

A Selection Screen is SAP's **auto-generated input form** that appears before a report runs. It is the user interface layer between the operator and the program logic.

Conceptually: the user fills in a web form and clicks Submit. The form values become available as ABAP variables. The program then runs its logic using those values.

The critical insight is that the selection screen is **generated automatically by the ABAP compiler** based on your `PARAMETERS` and `SELECT-OPTIONS` declarations. You write simple ABAP declarations and SAP generates the HTML/screen equivalent for you. You do not manually build input fields.

### PARAMETERS — Single Value Input
*[Video: ~01:27:43]*

`PARAMETERS` creates a single labeled input field on the selection screen. One declaration = one field.

```abap
PARAMETERS: p_carrid  TYPE scarr-carrid,              " plain text input
            p_date    TYPE sy-datum,                   " date picker (calendar F4)
            p_max     TYPE i DEFAULT 100,              " pre-filled with 100
            p_active  TYPE abap_bool AS CHECKBOX,      " checkbox: value is 'X' or ' '
            p_code    TYPE c LENGTH 10 OBLIGATORY,     " required — red asterisk shown
            p_descr   TYPE string LOWER CASE.          " allows lowercase input
```

Each `PARAMETERS` declaration corresponds to one labeled row on the selection screen.

**Parameter modifier options:**

| Option | Effect |
|---|---|
| `DEFAULT value` | Pre-fills the field with a value |
| `OBLIGATORY` | Makes the field mandatory — cannot execute if empty |
| `AS CHECKBOX` | Renders as a checkbox (`'X'` = checked, `' '` = unchecked) |
| `RADIOBUTTON GROUP grp` | Groups into mutually exclusive radio buttons |
| `NO-DISPLAY` | Hidden field — for system-provided values |
| `LOWER CASE` | Allows lowercase input (SAP default converts to uppercase) |
| `VISIBLE LENGTH n` | Display width of the input box |
| `MEMORY ID xyz` | Links to SAP SET/GET parameter memory — auto-fills with user's last used value |

### Radio Buttons
*[Video: ~01:31:03]*

```abap
PARAMETERS: p_all    RADIOBUTTON GROUP grp1 DEFAULT 'X',  " selected by default
            p_open   RADIOBUTTON GROUP grp1,
            p_closed RADIOBUTTON GROUP grp1.
```

All parameters in the same GROUP name are mutually exclusive — selecting one deselects all others. `DEFAULT 'X'` marks which one starts selected.

Using radio button values in START-OF-SELECTION:

```abap
START-OF-SELECTION.
  IF p_all = 'X'.
    SELECT * FROM vbak INTO TABLE lt_orders.
  ELSEIF p_open = 'X'.
    SELECT * FROM vbak INTO TABLE lt_orders WHERE rfgsd = space.
  ELSEIF p_closed = 'X'.
    SELECT * FROM vbak INTO TABLE lt_orders WHERE rfgsd <> space.
  ENDIF.
```

### SELECT-OPTIONS — Range Input
*[Video: ~01:32:52]*

`SELECT-OPTIONS` is one of ABAP's most powerful UI features. It creates an input that accepts — from the user — any combination of:
- A single value (equals LH)
- A range (from LH to SQ)
- Multiple individual values (LH, AA, BA, SQ)
- Exclusions (NOT AA, NOT LH to SQ)
- Pattern matches (L* for anything starting with L)

```abap
SELECT-OPTIONS: s_carrid FOR scarr-carrid,    " carrier code range
                s_date   FOR sflight-fldate,  " date range — gets calendar F4 automatically
                s_price  FOR sflight-price.   " price range
```

The `FOR table-field` syntax serves two purposes: it copies the data type from that field, and it automatically attaches the appropriate F4 search help.

### What SELECT-OPTIONS Actually Creates in Memory

This is crucial to understand — a `SELECT-OPTIONS` declaration creates an **internal table** (a ranges table) with exactly 4 columns:

| Column | Type | Description |
|---|---|---|
| `SIGN` | C(1) | `'I'` = Include this range, `'E'` = Exclude this range |
| `OPTION` | C(2) | The comparison operator |
| `LOW` | Same type as the field | The lower bound or single value |
| `HIGH` | Same type as the field | The upper bound (used with BT = Between) |

**OPTION values:**

| Option | Meaning |
|---|---|
| `EQ` | Equals (single value) |
| `NE` | Not equals |
| `BT` | Between LOW and HIGH (inclusive) |
| `NB` | Not between |
| `GT` | Greater than |
| `GE` | Greater than or equal |
| `LT` | Less than |
| `LE` | Less than or equal |
| `CP` | Contains pattern (like SQL LIKE with * as wildcard) |
| `NP` | Does not contain pattern |

**Example**: If the user enters "LH to UA" with Include, the ranges table contains:
```
SIGN='I'  OPTION='BT'  LOW='LH'  HIGH='UA'
```

If the user enters individual values LH and AA:
```
SIGN='I'  OPTION='EQ'  LOW='LH'  HIGH=''
SIGN='I'  OPTION='EQ'  LOW='AA'  HIGH=''
```

### Using SELECT-OPTIONS in a WHERE Clause

```abap
SELECT * FROM scarr
  INTO TABLE lt_carriers
  WHERE carrid IN s_carrid.    " IN handles all ranges, includes, excludes, patterns
```

The `IN` keyword evaluates the entire ranges table automatically — all includes, excludes, patterns, and ranges. **This is why SELECT-OPTIONS is so powerful**: the user can specify complex multi-value filter logic (include these, exclude those, match this pattern) without any extra code on your part. The ranges table plus `IN` handles it all.

### Organizing the Screen with BLOCKS
*[Video: ~01:25:30]*

```abap
SELECTION-SCREEN BEGIN OF BLOCK b_filter WITH FRAME TITLE TEXT-b01.
  PARAMETERS:     p_carrid TYPE scarr-carrid OBLIGATORY.
  SELECT-OPTIONS: s_date   FOR sflight-fldate.
SELECTION-SCREEN END OF BLOCK b_filter.

SELECTION-SCREEN BEGIN OF BLOCK b_options WITH FRAME TITLE TEXT-b02.
  PARAMETERS: p_max     TYPE i DEFAULT 100,
              p_details TYPE abap_bool AS CHECKBOX DEFAULT 'X'.
SELECTION-SCREEN END OF BLOCK b_options.
```

`WITH FRAME` draws a visible box around the group. `TITLE TEXT-b01` displays a Text Element as the box label.

### Other Screen Organization Commands

```abap
SELECTION-SCREEN SKIP.                      " blank line
SELECTION-SCREEN ULINE.                     " horizontal separator line
SELECTION-SCREEN COMMENT 5(40) TEXT-001.    " static text at column 5, width 40

" Put two parameters on the same line:
SELECTION-SCREEN BEGIN OF LINE.
  PARAMETERS p_from TYPE d.
  SELECTION-SCREEN COMMENT (5) TEXT-002.    " ' To ' label between fields
  PARAMETERS p_to   TYPE d.
SELECTION-SCREEN END OF LINE.
```

---

## 16. Text Elements — Externalising Hardcoded Text
*[Video: ~01:38:12]*

### The Problem with Hardcoded Text

Hardcoding text strings directly in ABAP code is a significant problem in enterprise software because:

1. **SAP is multilingual**: The same program must display in German for German users, English for English users, Japanese for Japanese users — without code changes
2. **Translators cannot work with code**: Professional translators need a dedicated tool and a clean list of strings, not a programmer's codebase
3. **Maintenance is fragile**: Finding and changing all occurrences of a string spread through code is error-prone

Text Elements solve this by **externalising all user-visible text into a separate, translatable repository** attached to the program.

### Types of Text Elements

**1. Text Symbols**
- Identified by a 3-character alphanumeric ID (e.g., `001`, `hdr`, `e01`)
- Referenced in code as `TEXT-xyz`
- Used in: WRITE statements, MESSAGE statements, selection screen titles
- Stored and edited in SE38: Goto → Text Elements → Text Symbols

```abap
WRITE TEXT-001.           " outputs whatever text is stored for symbol 001
WRITE TEXT-hdr.           " can use letters too: 'hdr', 'b01', etc.

SELECTION-SCREEN BEGIN OF BLOCK b1 WITH FRAME TITLE TEXT-b01.
" The block title is pulled from Text Element b01
```

**2. Selection Texts**
- Labels displayed next to `PARAMETERS` and `SELECT-OPTIONS` fields on the selection screen
- Without these, SAP uses the parameter name itself as the label (e.g., `P_CARRID` instead of "Airline Code")
- Edited in SE38: Goto → Text Elements → Selection Texts

**3. List Headings**
- Page header and column heading text for classic list (`WRITE`) reports
- Edited in SE38: Goto → Text Elements → List Headings

### Creating and Using Text Symbols

In SE38 with your program open:
1. Menu: **Goto → Text Elements → Text Symbols**
2. Click New Entries
3. Enter the ID (`001`) and the text (`'Enter Selection Criteria'`)
4. Save and Activate the text elements

In your code:
```abap
SELECTION-SCREEN BEGIN OF BLOCK b1 WITH FRAME TITLE TEXT-001.
" Displays: "Enter Selection Criteria" as the block title
```

### Translation Workflow

When a project goes multilingual:
1. Developer creates all text elements in the development language (usually English)
2. Translator opens **SE63** (Translation Workbench)
3. Finds the program and sees all text symbols
4. Enters translations for each symbol in the target language
5. Users logging in with language `'D'` (German) automatically see German text; English users see English — same program, same code, different text at runtime

---

## 17. ABAP Program Events — Complete Lifecycle
*[Video: ~01:53:51]*

### The Fundamental Concept

ABAP reports are **NOT executed top-to-bottom like a Python script or main() function**. They are **event-driven** — the SAP runtime calls specific sections of your code at specific lifecycle points.

You declare participation in events by using the event keyword as a section header in your program. The ABAP runtime handles the sequencing, calling each section at the right moment.

This is similar conceptually to event handlers in GUI programming (onClick, onSubmit) but applied to report lifecycle events.

### The Complete Event Execution Order

```
User triggers program (F8, T-Code, or background job)
          │
          ▼
  LOAD-OF-PROGRAM
  ─────────────────────────────────────────────────────
  Fires once when program is first loaded into AS memory.
  Used for one-time initialization of class/function group globals.
          │
          ▼
  INITIALIZATION
  ─────────────────────────────────────────────────────
  Fires every time before the selection screen is shown.
  Set default values for parameters and ranges here.
          │
          ▼
  AT SELECTION-SCREEN OUTPUT
  ─────────────────────────────────────────────────────
  Fires just before the selection screen is rendered/refreshed.
  Dynamically show/hide/enable/disable fields here.
          │
    ┌─────────────────────┐
    │  Selection Screen   │  ← User sees and interacts with the screen
    │  is shown to user   │
    └────────┬────────────┘
             │  User fills fields and clicks Execute (F8)
             ▼
  AT SELECTION-SCREEN
  ─────────────────────────────────────────────────────
  Fires after Execute clicked, BEFORE main logic runs.
  Validate all inputs here. Raise 'E' messages to stop and show errors.
          │
  (if no errors — validation passed)
          │
          ▼
  START-OF-SELECTION
  ─────────────────────────────────────────────────────
  THE MAIN EVENT. Your core business logic runs here:
  fetch data from DB, process it, optionally output it.
          │
          ▼
  END-OF-SELECTION
  ─────────────────────────────────────────────────────
  Fires after START-OF-SELECTION finishes.
  Often used to trigger final display (e.g., ALV grid) after all data is collected.
          │
  (for each new page of list output)
          │
          ▼
  TOP-OF-PAGE               ← Print page header for each new page
          │
          ▼
  END-OF-PAGE               ← Print page footer for each page
```

### LOAD-OF-PROGRAM

Fires exactly **once** when the program is first loaded into the application server's memory session. It does NOT fire again if the program is run a second time in the same user session (because it's already in memory).

Rarely used in reports. More common in:
- Function Groups (to initialize the group's global data)
- Module Pools (for screen initialization)

```abap
LOAD-OF-PROGRAM.
  " One-time initialization code here
  " e.g., load configuration data into a global cache
```

### INITIALIZATION
*[Video: ~02:02:05]*

Fires **every time the program starts** (every execution), before the selection screen is rendered. This is the event for setting intelligent default values that the user will see pre-filled.

```abap
INITIALIZATION.
  p_carrid = 'LH'.            " default airline

  " Default date range: last 30 days to today
  s_date-sign   = 'I'.
  s_date-option = 'BT'.
  s_date-low    = sy-datum - 30.
  s_date-high   = sy-datum.
  APPEND s_date.              " add this row to the ranges table
```

Note the difference from `DEFAULT` in the PARAMETERS declaration:
- `PARAMETERS p_x TYPE t DEFAULT 'val'` is a static default — always the literal value
- `INITIALIZATION` allows **dynamic defaults** computed at runtime (like "30 days ago from today")

### AT SELECTION-SCREEN OUTPUT

Fires just before the selection screen is **displayed** (and every time the screen needs to be refreshed — e.g., after an error). This is the event for **dynamic screen modifications**: hiding fields based on checkbox state, making fields conditionally mandatory, changing field labels dynamically.

```abap
AT SELECTION-SCREEN OUTPUT.
  " Example: hide the max-records field when 'show all' checkbox is checked
  IF p_all = 'X'.
    LOOP AT SCREEN.
      IF screen-name = 'P_MAX'.
        screen-active = 0.   " 0 = invisible/disabled, 1 = visible/active
        MODIFY SCREEN.
      ENDIF.
    ENDLOOP.
  ENDIF.
```

The `SCREEN` system table contains one row per screen element. Modifying the `screen-active` flag dynamically hides or shows that element.

### AT SELECTION-SCREEN

Fires after the user clicks Execute, but **before** `START-OF-SELECTION`. This is the **input validation** event.

Use this to:
- Check that referenced objects exist in the database (company codes, material types, etc.)
- Perform cross-field validations (from-date must be before to-date)
- Reject nonsensical combinations of inputs

```abap
AT SELECTION-SCREEN.
  " Validate company code exists
  IF p_bukrs IS NOT INITIAL.
    SELECT SINGLE bukrs FROM t001 INTO @DATA(lv_check)
      WHERE bukrs = p_bukrs.
    IF sy-subrc <> 0.
      MESSAGE e001(00) WITH 'Company code' p_bukrs 'does not exist'.
      " Type 'E' stops execution, keeps selection screen open, highlights the error
    ENDIF.
  ENDIF.

  " Cross-field validation
  IF p_date_from > p_date_to.
    MESSAGE 'From-date must be earlier than To-date' TYPE 'E'.
  ENDIF.
```

When you raise a type `'E'` message inside `AT SELECTION-SCREEN`, execution stops, the selection screen remains open, the error is shown in the status bar, and the user can correct their input and re-submit.

**Sub-events of AT SELECTION-SCREEN:**

```abap
AT SELECTION-SCREEN ON p_bukrs.
  " Fires specifically when focus leaves the p_bukrs field
  " Good for instant field-level validation

AT SELECTION-SCREEN ON VALUE-REQUEST FOR p_carrid.
  " Fires when user presses F4 on p_carrid
  " Implement a custom popup selection list here

AT SELECTION-SCREEN ON HELP-REQUEST FOR p_carrid.
  " Fires when user presses F1 on p_carrid
  " Display custom help text or documentation
```

### AT SELECTION-SCREEN ON VALUE-REQUEST (F4 Help Implementation)

F4 help creates a popup selection list for a parameter field. This is how you give users a "pick from a list" experience:

```abap
AT SELECTION-SCREEN ON VALUE-REQUEST FOR p_carrid.
  " Build a list of valid choices
  SELECT carrid carrname FROM scarr
    INTO TABLE @DATA(lt_carriers).

  " Use the standard F4 helper function to display popup and return selection
  DATA: lt_return TYPE TABLE OF ddshretval.
  CALL FUNCTION 'F4IF_INT_TABLE_VALUE_REQUEST'
    EXPORTING
      retfield    = 'CARRID'
      value_org   = 'S'
    TABLES
      value_tab   = lt_carriers
      return_tab  = lt_return.

  IF sy-subrc = 0 AND lt_return IS NOT INITIAL.
    READ TABLE lt_return INTO DATA(ls_ret) INDEX 1.
    p_carrid = ls_ret-fieldval.
  ENDIF.
```

### START-OF-SELECTION

The **main event**. The core program logic lives here: fetch data from the database, process it, compute results, and output them (or store in global tables for `END-OF-SELECTION`).

```abap
START-OF-SELECTION.
  " Fetch matching records
  SELECT carrid carrname currcode
    FROM scarr
    INTO TABLE @DATA(lt_carriers)
    WHERE carrid IN s_carrid.

  " Check if anything was found
  IF lt_carriers IS INITIAL.
    MESSAGE 'No airlines found matching your selection' TYPE 'S'
            DISPLAY LIKE 'W'.
    RETURN.
  ENDIF.

  " Process: display each airline
  LOOP AT lt_carriers INTO DATA(ls_carrier).
    WRITE: / ls_carrier-carrid, ls_carrier-carrname, ls_carrier-currcode.
  ENDLOOP.
```

### END-OF-SELECTION

Fires after `START-OF-SELECTION` completes all its processing. The pattern of **fetching in START-OF-SELECTION and displaying in END-OF-SELECTION** is the standard structure for ALV reports:

```abap
" In global declarations:
DATA: gt_output TYPE TABLE OF ty_s_output.

START-OF-SELECTION.
  " Fetch data from DB into gt_output
  PERFORM fetch_data.
  " Additional processing
  PERFORM enrich_data.

END-OF-SELECTION.
  " Now gt_output has all data — display with ALV
  PERFORM display_alv USING gt_output.
```

This separation keeps concerns clean: data retrieval in START, presentation in END.

### TOP-OF-PAGE

Fires automatically whenever list output (`WRITE`) starts a new page. Use for page headers — report title, run date, column headings.

```abap
TOP-OF-PAGE.
  WRITE: / 'Airline Schedule Report'.
  WRITE: / 'Generated by:', sy-uname, 'on', sy-datum.
  ULINE.
  WRITE: / 'Code', 20 'Name', 50 'Currency'.
  ULINE.
```

### END-OF-PAGE

Fires at the bottom of each output page. Use for page footers and page numbers.

```abap
END-OF-PAGE.
  ULINE.
  WRITE: / 'Page', sy-pagno, '— Confidential'.
```

### Complete Working Event Example
*[Video: ~02:02:05]*

```abap
REPORT zflight_report.

" Global data — accessible across all events
DATA: gt_flights TYPE TABLE OF sflight,
      gs_flight  TYPE sflight.

PARAMETERS: p_carrid TYPE sflight-carrid OBLIGATORY,
            p_year   TYPE numc4 DEFAULT '2024'.

" ─────── EVENTS ────────────────────────────────────────────

INITIALIZATION.
  p_carrid = 'LH'.    " default to Lufthansa

AT SELECTION-SCREEN.
  " Validate year is reasonable
  IF p_year < 2000 OR p_year > 2099.
    MESSAGE 'Year must be between 2000 and 2099' TYPE 'E'.
  ENDIF.
  " Validate carrier exists
  SELECT SINGLE carrid FROM scarr INTO @DATA(lv_check)
    WHERE carrid = p_carrid.
  IF sy-subrc <> 0.
    MESSAGE e001(00) WITH 'Airline' p_carrid 'not found in system'.
  ENDIF.

START-OF-SELECTION.
  PERFORM fetch_flights.

END-OF-SELECTION.
  PERFORM display_results.

TOP-OF-PAGE.
  WRITE: / 'Flight Report —', p_carrid.
  ULINE.

" ─────── SUBROUTINES ────────────────────────────────────────

FORM fetch_flights.
  DATA lv_pattern TYPE c LENGTH 10.
  CONCATENATE p_year '%' INTO lv_pattern.    " e.g., '2024%'
  SELECT * FROM sflight INTO TABLE gt_flights
    WHERE carrid = p_carrid
    AND   fldate LIKE lv_pattern.
ENDFORM.

FORM display_results.
  IF gt_flights IS INITIAL.
    WRITE: / 'No flights found.'.
    RETURN.
  ENDIF.
  LOOP AT gt_flights INTO gs_flight.
    WRITE: / gs_flight-fldate, gs_flight-connid,
             gs_flight-price CURRENCY gs_flight-currency.
  ENDLOOP.
  ULINE.
  WRITE: / 'Total flights found:', sy-dbcnt.
ENDFORM.
```

---

## 18. Subroutines — PERFORM / FORM / ENDFORM
*[Video: ~02:06:00]*

### What Are Subroutines?

Subroutines are **named, reusable blocks of code** within a program. They are ABAP's classic (pre-OOP) mechanism for modularisation — equivalent to functions or procedures in other languages, but with important SAP-specific characteristics.

Key characteristics:
- They belong to a specific program and cannot normally be called from other programs (use Function Modules for that)
- They can access all global variables of the program directly without parameters
- They accept parameters via USING (input) and CHANGING (output/inout) when explicit passing is needed
- They are declared with `FORM` and closed with `ENDFORM`
- They are called with `PERFORM`

### Basic Syntax — No Parameters

```abap
REPORT zmyreport.

" Global data — accessible in all subroutines
DATA: gt_data  TYPE TABLE OF scarr,
      gv_count TYPE i.

START-OF-SELECTION.
  PERFORM fetch_data.      " call subroutine
  PERFORM process_data.
  PERFORM display_data.

" ─── SUBROUTINE DEFINITIONS (by convention at the bottom of the program) ───

FORM fetch_data.
  SELECT * FROM scarr INTO TABLE gt_data.
  gv_count = lines( gt_data ).
ENDFORM.

FORM process_data.
  " Sort by name
  SORT gt_data BY carrname.
ENDFORM.

FORM display_data.
  WRITE: / 'Airlines found:', gv_count.
  ULINE.
  LOOP AT gt_data INTO DATA(ls_row).
    WRITE: / ls_row-carrid, ls_row-carrname.
  ENDLOOP.
ENDFORM.
```

### Subroutines With Parameters

When you want to be explicit about what data a subroutine reads and modifies (better practice, avoids tight coupling to global state):

```abap
" USING    = pass by value — input parameters, read-only in subroutine
" CHANGING = pass by reference — input/output parameters, changes reflect in caller

FORM calculate_vat
  USING    iv_net_price  TYPE p
           iv_vat_rate   TYPE p
  CHANGING cv_vat_amount TYPE p
           cv_gross_price TYPE p.

  cv_vat_amount  = iv_net_price * iv_vat_rate.
  cv_gross_price = iv_net_price + cv_vat_amount.

ENDFORM.

" Calling with parameters:
DATA: lv_price     TYPE p DECIMALS 2 VALUE '100.00',
      lv_rate      TYPE p DECIMALS 4 VALUE '0.2000',  " 20% VAT
      lv_vat       TYPE p DECIMALS 2,
      lv_gross     TYPE p DECIMALS 2.

PERFORM calculate_vat
  USING    lv_price lv_rate
  CHANGING lv_vat lv_gross.

WRITE: / 'Net:', lv_price, 'VAT:', lv_vat, 'Gross:', lv_gross.
" Output: Net: 100.00  VAT: 20.00  Gross: 120.00
```

### Passing Internal Tables to Subroutines

```abap
FORM display_carriers
  USING it_carriers TYPE TABLE.    " generic table type — works but loses type info

FORM display_carriers_typed
  USING it_carriers TYPE ty_t_carriers.    " specific typed — preferred

FORM sort_and_dedup
  CHANGING ct_data TYPE ty_t_carriers.     " modifies the table in the caller
```

### Where to Place FORM Definitions

By convention (and for readability), all FORM definitions go **at the bottom of the program**, after all events (`START-OF-SELECTION`, `END-OF-SELECTION`, etc.). The ABAP compiler scans the entire program before execution so the physical order does not matter technically — but bottom placement is the universal SAP convention and makes programs readable.

### Subroutines vs. Function Modules vs. Methods

| | FORM/ENDFORM | Function Module | Method (Class) |
|---|---|---|---|
| Scope | Within one program only | Global — callable from any program | Within a class (can be public) |
| Can be called remotely (RFC) | No | Yes (if marked as RFC-enabled) | No (without special config) |
| Exception handling | No formal EXCEPTIONS | Formal `EXCEPTIONS` clause | `RAISING` with class-based exceptions |
| Testing | Hard to unit test | Testable via SE37 | Easy to unit test |
| Modern practice | Legacy — but everywhere in existing code | Still widely used, industry standard | Preferred for all new development |

> You will encounter `PERFORM/FORM` constantly when maintaining legacy SAP code — it is in virtually every ABAP program written before ~2010. You must be able to read and write it fluently. For new development, prefer methods in classes.

---

## 19. Where-Used List
*[Video: ~02:36:15]*

### What Is It?

The Where-Used List answers the question: **"If I change or delete this object, what will break?"** It shows every SAP object that references the object you are looking at.

### How to Use It

**In SE38** (for programs, types, variables, forms):
- Place cursor on an object name in your code
- Press **Shift+F4** or go to Utilities → Where-Used List

**In SE11** (for DDIC objects):
- Open any domain, data element, table, or structure
- Menu: Utilities → Where-Used List
- Select what to search (programs, tables, structures, function modules, etc.)

**In SE37** (for function modules):
- Open the function module
- Menu: Utilities → Where-Used List

### Practical Scenario

You have a custom domain `ZMAT_STATUS` of type `CHAR1` and a business user says "we need 3 characters for status codes now, not 1". Before you change the domain length:

1. Where-Used on the domain → finds 5 data elements using it
2. Where-Used on each data element → finds 12 database table fields and 3 structures
3. Where-Used on each affected table → finds 47 programs reading those fields
4. Impact assessment: you need to resize DB fields, check all 47 programs for field length assumptions, and test everything

This is why even "small" changes in SAP are complex — everything is interconnected through the DDIC.

### Learning with Where-Used

Where-Used is also a powerful learning tool when you're new to SAP:
1. You see an unfamiliar field on a SAP transaction screen
2. Press F1 on the field → get the technical field name and table
3. Open that table in SE11
4. Where-Used on the data element → find programs that use it
5. Open those programs in SE38 → see how experienced developers work with that field

This self-directed exploration is one of the fastest ways to learn real SAP development patterns.

---

## 20. Homework Assignments
*[Video: ~02:45:10]*

Practice exercises assigned by the instructor:

**Assignment 1 — Selection Screen**
Create a report (`ZASSIGN_SELSCREEN`) with a selection screen containing:
- A company code parameter (type `BUKRS`, obligatory)
- A date range (SELECT-OPTIONS for a date field)
- A checkbox "Include inactive records"
- Two radio buttons: "Summary View" and "Detail View"
- Organized into 2 titled, framed blocks: "Selection Criteria" and "Display Options"

**Assignment 2 — Input Validation**
In `AT SELECTION-SCREEN` of the above report:
- Validate the company code exists in table `T001`
- Show an error message highlighting the field if invalid
- Validate the date range (from-date ≤ to-date)

**Assignment 3 — SELECT SINGLE**
Create a report that:
- Takes a carrier ID as a parameter
- Uses SELECT SINGLE on SCARR to fetch the airline details
- Displays: Airline Code, Full Name, Currency, Website URL
- Handles the "not found" case gracefully

**Assignment 4 — Date Formatting**
Display today's date in these formats:
- `YYYYMMDD` (raw internal format)
- `DD.MM.YYYY` (European format)
- `MM/DD/YYYY` (US format)
- A custom "Day DD Month YYYY" format (e.g., "Wednesday 15 December 2024") — requires deriving the day name from the date

**Assignment 5 — Custom F4 Search Help**
Implement `AT SELECTION-SCREEN ON VALUE-REQUEST FOR` for the carrier parameter from Assignment 3. When the user presses F4, show a popup list of all airlines from SCARR for selection.

**Assignment 6 — Subroutine Refactoring**
Take any working report and refactor it into distinct subroutines:
- `FORM fetch_data` — all DB access
- `FORM process_data` — any calculations or transformations
- `FORM display_output` — all WRITE statements

---

## 21. Key Takeaways & Revision Questions

### Core Concepts Summary

**Data Types**: Always match the type to the use case. `P DECIMALS 2` for all financial values (never `F`). `D` for dates with arithmetic. `NUMC` for zero-padded IDs. `STRING` for variable-length text. `I` for counters and integers. `C` for fixed-length codes.

**`sy-subrc`**: Check it after EVERY operation that can fail. `0` means success. `4` means not found. Never skip this check — it is the root cause of many SAP bugs.

**Selection Screen Architecture**:
- `PARAMETERS` = one input field, one value
- `SELECT-OPTIONS` = a ranges table (SIGN/OPTION/LOW/HIGH), used with `IN` in WHERE clauses
- `SELECTION-SCREEN BLOCK` = visual grouping with optional frame and title
- Text elements = external, translatable text — never hardcode user-visible strings

**Event Order**: `INITIALIZATION` → `AT SELECTION-SCREEN OUTPUT` → (user input) → `AT SELECTION-SCREEN` → `START-OF-SELECTION` → `END-OF-SELECTION`

**FORM Subroutines**: Named code blocks called with `PERFORM`. Define at the bottom of the program. Use `USING` for input parameters (by value), `CHANGING` for output/inout parameters (by reference).

**Where-Used**: Use before modifying any shared or DDIC object to assess the impact.

### Revision Questions

1. What is the difference between `PARAMETERS` and `SELECT-OPTIONS`? What internal data structure does SELECT-OPTIONS create?
2. What are the 4 columns in a SELECT-OPTIONS ranges table? What does the SIGN column contain?
3. List the ABAP report event execution order from program start to list output. What is each event used for?
4. What is `sy-subrc`? What value means "not found" for a SELECT SINGLE? What should you always do after any DB operation?
5. Why is `TYPE P DECIMALS 2` required for financial calculations instead of `TYPE F`?
6. How does `SELECT SINGLE` differ from a regular `SELECT`? When would you use each?
7. Explain ABAP's string offset/length syntax with an example. How would you extract the year from a date field of type `D`?
8. What is the difference between `TYPES` and `DATA`? Does `TYPES` allocate memory?
9. What does `CLEAR ls_structure` do? Why is it important to call this inside a loop before building a work area?
10. What is the Where-Used List and what is the correct workflow before modifying a DDIC domain?
11. What happens in ABAP integer division when 10 is divided by 3 and the result is stored in type `I`? How is this different from storing it in `P DECIMALS 4`?
12. Why should you use Text Elements instead of hardcoded string literals in selection screen titles? What SAP tool do translators use to translate text elements?
13. What is the difference between `USING` and `CHANGING` when passing parameters to a FORM subroutine?
14. In what event should you validate user inputs and why? What message type stops execution and keeps the selection screen open?
15. What is the pattern for fetching data in `START-OF-SELECTION` and displaying in `END-OF-SELECTION`? Why is this useful?

---

*Notes based on ZAP Yard — SAP ABAP Video 2 of 11*
*Previous: SAP_ABAP_Video1_StudyNotes.md*
*Reference: ABAP_DDIC_Tutorial.md for additional code examples and syntax cheat sheet*
