# Part 5 — The Processor Engine

## Building the Core Document Generation Engine with Dynamic Tables and Repeating Data

> **Goal:** Transform Greymatter Docs from a simple placeholder replacement tool into a full document generation engine capable of processing multiple data sources, generating dynamic tables, and handling repeating data.

**Milestone:** By the end of this chapter, Greymatter Docs will generate professional documents containing customer information and dynamically populated line-item tables using live data from SQLite.

---

# THEORY & ARCHITECTURE

So far, Greymatter Docs can replace individual placeholders such as:

```text
{{first_name}}
{{company}}
{{email}}
```

This works well for single values, but most business documents also contain **repeating data**, such as:

* Invoice line items
* Purchase order items
* Employee lists
* Attendance registers
* Product catalogs
* Expense reports

Instead of manually creating rows, our processor will generate them automatically from database records.

The updated architecture introduces a dedicated processor engine.

```text
                 SQLite
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
 CustomerRepository      InvoiceRepository
        │                       │
        └───────────┬───────────┘
                    ▼
           Processor Engine
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
 Placeholder Engine     Table Engine
          │                   │
          └─────────┬─────────┘
                    ▼
            LibreOffice Writer
                    │
                    ▼
          Generated Document
```

Instead of one processor doing everything, each component has a single responsibility.

---

# BUILD IT

## Step 1 — Expand the Database

We'll add two new tables:

```text
customers
```

```text
invoice_headers
```

```text
invoice_items
```

Relationship:

```text
customers
    │
    │ 1
    │
    ├───────────────∞
            invoice_headers
                    │
                    │ 1
                    │
                    ├──────────────∞
                     invoice_items
```

---

## Step 2 — Update the Database Schema

**database/schema.py**

```sql
CREATE TABLE IF NOT EXISTS invoice_headers (

    id INTEGER PRIMARY KEY AUTOINCREMENT,

    customer_id INTEGER NOT NULL,

    invoice_number TEXT NOT NULL,

    invoice_date TEXT,

    FOREIGN KEY(customer_id)
        REFERENCES customers(id)

);
```

---

```sql
CREATE TABLE IF NOT EXISTS invoice_items (

    id INTEGER PRIMARY KEY AUTOINCREMENT,

    invoice_id INTEGER NOT NULL,

    description TEXT,

    quantity INTEGER,

    unit_price REAL,

    FOREIGN KEY(invoice_id)
        REFERENCES invoice_headers(id)

);
```

---

## Step 3 — Seed Invoice Data

Example:

```python
invoice = (
    1,
    "INV-2025-001",
    "2025-07-01"
)
```

Invoice items:

```python
items = [

    (
        1,
        "Python Development",
        25,
        150.00
    ),

    (
        1,
        "LibreOffice Integration",
        10,
        175.00
    ),

    (
        1,
        "Testing",
        12,
        95.00
    )

]
```

---

## Step 4 — Create the Invoice Repository

**database/invoice_repository.py**

```python
import logging

from database.database import DatabaseManager

logger = logging.getLogger(__name__)


class InvoiceRepository:

    def __init__(self):
        self.database = DatabaseManager()

    def get_invoice(self, invoice_id):

        sql = """
        SELECT *

        FROM invoice_headers

        WHERE id = ?
        """

        try:

            with self.database.get_connection() as connection:

                cursor = connection.cursor()

                cursor.execute(
                    sql,
                    (invoice_id,)
                )

                return cursor.fetchone()

        except Exception as error:

            logger.exception(error)

            raise RuntimeError(
                "Unable to retrieve invoice."
            ) from error

    def get_invoice_items(self, invoice_id):

        sql = """
        SELECT *

        FROM invoice_items

        WHERE invoice_id = ?

        ORDER BY id
        """

        try:

            with self.database.get_connection() as connection:

                cursor = connection.cursor()

                cursor.execute(
                    sql,
                    (invoice_id,)
                )

                return cursor.fetchall()

        except Exception as error:

            logger.exception(error)

            raise RuntimeError(
                "Unable to retrieve invoice items."
            ) from error
```

---

## Step 5 — Design the Invoice Template

Create:

```text
templates/invoice.ott
```

The template should contain placeholders such as:

```text
Invoice Number

{{invoice_number}}

Invoice Date

{{invoice_date}}

Customer

{{first_name}} {{last_name}}

Company

{{company}}
```

Create an empty table with one sample row.

| Description     | Quantity     | Unit Price     | Total          |
| --------------- | ------------ | -------------- | -------------- |
| {{description}} | {{quantity}} | {{unit_price}} | {{line_total}} |

This single row will become our template row.

---

## Step 6 — Build the Table Processor

Create:

**processor/table_processor.py**

```python
import logging

logger = logging.getLogger(__name__)


class TableProcessor:

    def populate_table(
        self,
        table,
        rows
    ):
        """
        Populate a Writer table.

        """

        try:

            for index, item in enumerate(rows):

                if index > 0:

                    table.Rows.insertByIndex(
                        index,
                        1
                    )

                table.getCellByName(
                    f"A{index+2}"
                ).setString(
                    item["description"]
                )

                table.getCellByName(
                    f"B{index+2}"
                ).setValue(
                    item["quantity"]
                )

                table.getCellByName(
                    f"C{index+2}"
                ).setValue(
                    item["unit_price"]
                )

                total = (
                    item["quantity"] *
                    item["unit_price"]
                )

                table.getCellByName(
                    f"D{index+2}"
                ).setValue(
                    total
                )

            logger.info(
                "Inserted %s invoice rows.",
                len(rows)
            )

        except Exception as error:

            logger.exception(error)

            raise RuntimeError(
                "Unable to populate table."
            ) from error
```

---

## Step 7 — Update the Document Processor

Add a method:

```python
def populate_invoice_table(
    self,
    document,
    items
):
```

Locate the first Writer table.

```python
tables = document.getTextTables()

table = tables.getByIndex(0)
```

Then call:

```python
processor = TableProcessor()

processor.populate_table(
    table,
    items
)
```

---

## Step 8 — Update `main.py`

```python
invoice_repository = InvoiceRepository()

invoice = invoice_repository.get_invoice(1)

items = invoice_repository.get_invoice_items(1)
```

Merge customer and invoice dictionaries.

```python
values = {}

values.update(dict(customer))

values.update(dict(invoice))
```

Replace placeholders.

Populate the invoice table.

Save the document.

---

# CODE WALKTHROUGH

## Why Separate the Table Processor?

Placeholder replacement and table generation solve different problems.

The placeholder engine works with individual text values.

The table processor works with collections of records.

Keeping them separate makes both components simpler and easier to test.

---

## Dynamic Row Creation

Instead of designing a template with 100 empty rows, we create only one.

The processor duplicates rows as needed.

Example:

Database contains:

```text
3 records
```

Generated document:

```text
Description
Python Development

LibreOffice Integration

Testing
```

Database contains:

```text
50 records
```

Generated document:

```text
50 table rows
```

The template never changes.

---

## Calculated Fields

Notice this calculation:

```python
total = (
    item["quantity"] *
    item["unit_price"]
)
```

This demonstrates an important principle:

Business calculations belong in Python, not in document templates.

Templates should focus on layout, while the application handles logic.

---

## Repository Collaboration

By this stage, the application coordinates data from multiple repositories:

```text
CustomerRepository
```

```text
InvoiceRepository
```

Later chapters will introduce additional repositories for templates, jobs, audit logs, and generated documents.

---

# CHALLENGE LAB

Extend the processor engine with the following features:

1. Add support for:

```text
{{invoice_total}}
```

2. Calculate:

* Subtotal
* GST/VAT
* Grand Total

3. Format currency with two decimal places.

Example:

```text
150.00
```

4. Automatically number each invoice line.

| No | Description        |
| -- | ------------------ |
| 1  | Python Development |
| 2  | Testing            |

5. Highlight rows where the quantity exceeds 20.

---

# TROUBLESHOOTING

| Problem                         | Cause                                         | Solution                                                                     |
| ------------------------------- | --------------------------------------------- | ---------------------------------------------------------------------------- |
| Table remains empty             | Wrong table index or template missing a table | Verify the document contains a Writer table and retrieve the correct table.  |
| `NoSuchElementException`        | Cell name is incorrect                        | Confirm cell names (e.g., `A2`, `B2`) match the table structure.             |
| Calculated totals are incorrect | Incorrect arithmetic or data types            | Ensure numeric values are converted to `int` or `float` before calculations. |
| Extra blank rows appear         | Template contains additional sample rows      | Leave only one template row and let the processor insert new rows.           |
| `NoneType` errors               | Repository returned no invoice or items       | Validate database contents before processing.                                |

---

# Chapter Summary

In this chapter, you:

* Expanded the database to support invoices and line items.
* Introduced the `InvoiceRepository` for retrieving invoice data.
* Designed an invoice template with a reusable table row.
* Built a `TableProcessor` to generate dynamic Writer tables.
* Combined placeholder replacement with repeating data generation.
* Established a clear separation between business logic and document layout.

Greymatter Docs can now generate realistic business documents containing both single-value fields and dynamic tabular data. This marks the transition from basic document automation to a true document generation platform.

**Next:** **Part 6 – Production Mechanics**, where we'll harden the application with structured logging, custom exception classes, configuration management, and resilient error handling suitable for production deployments.
