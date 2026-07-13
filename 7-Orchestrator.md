# Part 7 — The Orchestrator

## Building the End-to-End Automation Pipeline

> **Goal:** Build the orchestration layer that coordinates the entire document generation workflow—from retrieving data to producing finished documents—using a modular, extensible pipeline.

**Milestone:** By the end of this chapter, Greymatter Docs will process document generation jobs through a centralized orchestrator, making it easy to automate individual or batch document creation.

---

# THEORY & ARCHITECTURE

Up to this point, `main.py` has been coordinating every step of the application:

* Connecting to the database
* Loading customer data
* Connecting to LibreOffice
* Opening templates
* Replacing placeholders
* Building tables
* Saving documents

As the application grows, `main.py` will become difficult to maintain.

Instead, we'll introduce an **Orchestrator** that manages the workflow while delegating specific tasks to dedicated services.

The updated architecture:

```text
                     main.py
                        │
                        ▼
             DocumentOrchestrator
                        │
        ┌───────────────┼────────────────┐
        ▼               ▼                ▼
 Repository Layer   Processor Layer   Service Layer
        │               │                │
        ▼               ▼                ▼
     SQLite       Placeholder/Table     UNO API
                        │
                        ▼
                 Generated Document
```

The orchestrator acts as the application's **conductor**, coordinating each component without embedding business logic.

---

# BUILD IT

## Step 1 — Update the Project Structure

```text
greymatter_docs/
│
├── config/
├── database/
├── exceptions/
├── processor/
├── services/
├── orchestrator/
│   ├── __init__.py
│   ├── document_orchestrator.py
│   └── job_context.py
│
├── templates/
├── output/
├── logs/
└── main.py
```

---

## Step 2 — Create a Job Context

A job context carries all information needed during a document generation job.

**orchestrator/job_context.py**

```python
from dataclasses import dataclass
from pathlib import Path


@dataclass
class JobContext:
    """Represents a document generation job."""

    customer_id: int
    invoice_id: int
    template_name: str
    output_file: Path
```

This makes it easy to pass job information through the pipeline without long parameter lists.

---

## Step 3 — Build the Document Orchestrator

**orchestrator/document_orchestrator.py**

```python
import logging

from database.repositories import CustomerRepository
from database.invoice_repository import InvoiceRepository
from processor.document_processor import DocumentProcessor
from processor.table_processor import TableProcessor
from processor.template_loader import TemplateLoader
from services.libreoffice_service import LibreOfficeService
from services.writer_service import WriterService

logger = logging.getLogger(__name__)


class DocumentOrchestrator:
    """Coordinates the document generation workflow."""

    def __init__(self):
        self.customer_repository = CustomerRepository()
        self.invoice_repository = InvoiceRepository()
        self.document_processor = DocumentProcessor()
        self.table_processor = TableProcessor()
        self.libreoffice = LibreOfficeService()

    def generate_document(self, job):
        """Generate a document for a single job."""

        logger.info("Starting job for invoice %s", job.invoice_id)

        self.libreoffice.connect()

        writer = WriterService(self.libreoffice.desktop)

        customer = self.customer_repository.get_customer_by_id(
            job.customer_id
        )

        invoice = self.invoice_repository.get_invoice(
            job.invoice_id
        )

        items = self.invoice_repository.get_invoice_items(
            job.invoice_id
        )

        values = {}

        values.update(dict(customer))
        values.update(dict(invoice))

        template_path = TemplateLoader.get_template_path(
            job.template_name
        )

        document = writer.open_document(template_path)

        self.document_processor.replace_placeholders(
            document,
            values
        )

        tables = document.getTextTables()

        if tables.getCount() > 0:
            table = tables.getByIndex(0)

            self.table_processor.populate_table(
                table,
                items
            )

        writer.save_document(
            document,
            job.output_file
        )

        writer.close_document(document)

        logger.info("Job completed successfully.")
```

---

## Step 4 — Build a Job Queue

Although we'll introduce full batch processing in Part 9, we can prepare by creating a simple queue.

**main.py**

```python
from pathlib import Path

from config.logger import logger
from config.settings import OUTPUT_DIR
from orchestrator.document_orchestrator import (
    DocumentOrchestrator,
)
from orchestrator.job_context import JobContext

OUTPUT_DIR.mkdir(
    parents=True,
    exist_ok=True
)

jobs = [
    JobContext(
        customer_id=1,
        invoice_id=1,
        template_name="invoice.ott",
        output_file=OUTPUT_DIR / "invoice_001.odt",
    ),
    JobContext(
        customer_id=2,
        invoice_id=2,
        template_name="invoice.ott",
        output_file=OUTPUT_DIR / "invoice_002.odt",
    ),
]
```

---

## Step 5 — Execute the Queue

```python
def main():
    logger.info("=" * 60)
    logger.info("Greymatter Docs Starting")

    orchestrator = DocumentOrchestrator()

    for job in jobs:
        try:
            orchestrator.generate_document(job)
        except Exception:
            logger.exception(
                "Job failed for invoice %s",
                job.invoice_id
            )


if __name__ == "__main__":
    main()
```

Each job is processed independently. If one fails, the remaining jobs continue.

---

## Step 6 — Add Pipeline Hooks

Add logging hooks inside `generate_document()`:

```python
logger.info("Loading customer...")
logger.info("Loading invoice...")
logger.info("Opening template...")
logger.info("Replacing placeholders...")
logger.info("Populating tables...")
logger.info("Saving document...")
```

These checkpoints make it easy to trace execution in production logs.

---

# CODE WALKTHROUGH

## Why an Orchestrator?

Without an orchestrator:

* Business logic becomes scattered.
* `main.py` grows uncontrollably.
* Testing becomes more difficult.

With an orchestrator:

* Each service has one responsibility.
* Workflow changes happen in one place.
* The application becomes easier to extend.

---

## The Job Context

Instead of passing multiple arguments:

```python
generate_document(
    customer_id,
    invoice_id,
    template_name,
    output_path
)
```

we pass a single object:

```python
generate_document(job)
```

This improves readability and makes future enhancements—such as priority levels or output formats—straightforward.

---

## Fault Isolation

Notice that each job is wrapped in its own `try/except` block.

This ensures:

* One failed job does not stop the queue.
* Every failure is logged.
* Successful jobs continue processing.

This is essential for batch document generation.

---

## Pipeline Stages

Each document flows through the same sequence:

```text
Receive Job
      │
      ▼
Load Customer
      │
      ▼
Load Invoice
      │
      ▼
Open Template
      │
      ▼
Replace Placeholders
      │
      ▼
Populate Tables
      │
      ▼
Save Document
      │
      ▼
Close Document
```

This predictable pipeline makes debugging and maintenance much simpler.

---

# CHALLENGE LAB

Extend the orchestrator with the following features:

1. Add a unique `job_id` to `JobContext`.
2. Record the start and end time of each job.
3. Log the total processing time for each document.
4. Skip document generation if the output file already exists (unless an `overwrite` flag is set).
5. Generate a summary report showing:

   * Total jobs processed
   * Successful jobs
   * Failed jobs
   * Total execution time

---

# TROUBLESHOOTING

| Problem                                   | Cause                                 | Solution                                                              |
| ----------------------------------------- | ------------------------------------- | --------------------------------------------------------------------- |
| Pipeline stops after one failure          | Exceptions are not isolated           | Wrap each job in its own `try/except` block.                          |
| Wrong customer data appears in a document | Incorrect job configuration           | Verify the `customer_id` and `invoice_id` values in the `JobContext`. |
| Output file overwritten unintentionally   | No overwrite check                    | Verify whether the output file exists before saving.                  |
| Logs are difficult to follow              | Insufficient pipeline logging         | Add clear log messages before and after each major processing step.   |
| `AttributeError` on `JobContext`          | Missing or incorrect dataclass fields | Ensure the dataclass matches the required job attributes.             |

---

# Chapter Summary

In this chapter, you:

* Introduced a dedicated orchestration layer.
* Created a `JobContext` dataclass to encapsulate document generation requests.
* Built a `DocumentOrchestrator` to coordinate repositories, processors, and services.
* Executed multiple document generation jobs through a reusable pipeline.
* Improved resilience by isolating failures and enhancing logging.

Greymatter Docs now has a scalable automation workflow capable of processing multiple document generation jobs in a consistent and maintainable manner.

**Next:** **Part 8 – Output & Delivery**, where we'll extend the platform to export documents as PDF, deliver them via SMTP email, and introduce a simple delivery queue for reliable document distribution.
