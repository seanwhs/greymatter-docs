# Part 8 — Output & Delivery

## PDF Export, Email Delivery, and Reliable Delivery Queues

> **Goal:** Extend Greymatter Docs beyond document generation by exporting Writer documents to PDF, delivering them via email, and introducing a delivery queue that tracks the status of every generated document.

**Milestone:** By the end of this chapter, Greymatter Docs will automatically generate an `.odt` document, export it as a PDF, queue it for delivery, send it via SMTP, and update the delivery status.

---

# THEORY & ARCHITECTURE

Generating a document is only half the workflow. Most business applications also need to deliver the final document.

Our output pipeline will:

1. Generate the document (`.odt`)
2. Export it to PDF
3. Create a delivery job
4. Send the document by email
5. Record the delivery result

Updated architecture:

```text
                    SQLite
                       │
                       ▼
              Document Orchestrator
                       │
          ┌────────────┴────────────┐
          ▼                         ▼
    LibreOffice Writer        Delivery Queue
          │                         │
          ▼                         ▼
      Export PDF              SMTP Service
          │                         │
          └────────────┬────────────┘
                       ▼
                 Customer Inbox
```

This separation allows document generation and delivery to evolve independently.

---

# BUILD IT

## Step 1 — Extend the Database

Add a table to track document delivery.

```sql
CREATE TABLE IF NOT EXISTS delivery_queue (

    id INTEGER PRIMARY KEY AUTOINCREMENT,

    invoice_id INTEGER NOT NULL,

    customer_id INTEGER NOT NULL,

    output_file TEXT NOT NULL,

    pdf_file TEXT NOT NULL,

    recipient_email TEXT NOT NULL,

    status TEXT NOT NULL,

    error_message TEXT,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    sent_at TIMESTAMP
);
```

### Status Values

| Status  | Meaning                |
| ------- | ---------------------- |
| PENDING | Waiting for delivery   |
| SENT    | Successfully delivered |
| FAILED  | Delivery failed        |

---

## Step 2 — Create the PDF Export Service

Project structure:

```text
services/
│
├── libreoffice_service.py
├── writer_service.py
├── pdf_service.py
└── smtp_service.py
```

---

### `services/pdf_service.py`

```python
import logging
from pathlib import Path

import uno

logger = logging.getLogger(__name__)


class PdfService:
    """Exports Writer documents to PDF."""

    @staticmethod
    def export(document, output_file: Path):
        """Export a Writer document as PDF."""

        try:
            pdf_path = output_file.with_suffix(".pdf")

            properties = (
                uno.createUnoStruct(
                    "com.sun.star.beans.PropertyValue"
                ),
            )

            properties[0].Name = "FilterName"
            properties[0].Value = "writer_pdf_Export"

            document.storeToURL(
                uno.systemPathToFileUrl(
                    str(pdf_path.resolve())
                ),
                properties,
            )

            logger.info(
                "PDF exported: %s",
                pdf_path
            )

            return pdf_path

        except Exception as error:
            logger.exception(error)

            raise RuntimeError(
                "Unable to export PDF."
            ) from error
```

---

## Step 3 — Configure SMTP Settings

Update `config/settings.py`

```python
SMTP_SERVER = "smtp.example.com"
SMTP_PORT = 587

SMTP_USERNAME = "username"

SMTP_PASSWORD = "password"

SMTP_SENDER = "noreply@example.com"
```

> **Production Tip:** Never hardcode credentials. Use environment variables or a secrets manager.

---

## Step 4 — Build the SMTP Service

**services/smtp_service.py**

```python
import logging
import smtplib

from email.message import EmailMessage

from config.settings import (
    SMTP_SERVER,
    SMTP_PORT,
    SMTP_USERNAME,
    SMTP_PASSWORD,
    SMTP_SENDER,
)

logger = logging.getLogger(__name__)


class SmtpService:

    def send_email(
        self,
        recipient,
        subject,
        body,
        attachment
    ):

        try:

            message = EmailMessage()

            message["Subject"] = subject
            message["From"] = SMTP_SENDER
            message["To"] = recipient

            message.set_content(body)

            with open(
                attachment,
                "rb"
            ) as file:

                message.add_attachment(
                    file.read(),
                    maintype="application",
                    subtype="pdf",
                    filename=attachment.name,
                )

            with smtplib.SMTP(
                SMTP_SERVER,
                SMTP_PORT
            ) as smtp:

                smtp.starttls()

                smtp.login(
                    SMTP_USERNAME,
                    SMTP_PASSWORD
                )

                smtp.send_message(message)

            logger.info(
                "Email delivered to %s",
                recipient
            )

        except Exception as error:

            logger.exception(error)

            raise RuntimeError(
                "Unable to send email."
            ) from error
```

---

## Step 5 — Create the Delivery Repository

**database/delivery_repository.py**

```python
import logging

from database.database import DatabaseManager

logger = logging.getLogger(__name__)


class DeliveryRepository:

    def __init__(self):
        self.database = DatabaseManager()

    def create_job(
        self,
        customer_id,
        invoice_id,
        output_file,
        pdf_file,
        recipient,
    ):

        sql = """
        INSERT INTO delivery_queue
        (
            customer_id,
            invoice_id,
            output_file,
            pdf_file,
            recipient_email,
            status
        )
        VALUES (?, ?, ?, ?, ?, 'PENDING')
        """

        with self.database.get_connection() as connection:

            cursor = connection.cursor()

            cursor.execute(
                sql,
                (
                    customer_id,
                    invoice_id,
                    str(output_file),
                    str(pdf_file),
                    recipient,
                ),
            )

            connection.commit()

            return cursor.lastrowid
```

---

## Step 6 — Update the Orchestrator

After saving the Writer document:

```python
pdf_service = PdfService()

pdf_file = pdf_service.export(
    document,
    job.output_file
)
```

Create the delivery job.

```python
delivery_id = delivery_repository.create_job(
    job.customer_id,
    job.invoice_id,
    job.output_file,
    pdf_file,
    customer["email"]
)
```

Then send the email.

```python
smtp_service.send_email(
    recipient=customer["email"],
    subject="Your Invoice",
    body="Please find your invoice attached.",
    attachment=pdf_file,
)
```

Finally, update the delivery status to `SENT`.

If an exception occurs, update the status to `FAILED` and record the error message.

---

# CODE WALKTHROUGH

## PDF Export

LibreOffice supports many export formats through filters.

For Writer documents:

```python
writer_pdf_Export
```

Other examples include:

| Filter              | Purpose |
| ------------------- | ------- |
| `writer_pdf_Export` | PDF     |
| `writer8`           | ODT     |
| `MS Word 2007 XML`  | DOCX    |

This makes Greymatter Docs extensible without changing the processing pipeline.

---

## Delivery Queue

Rather than sending emails immediately after document generation, we first create a delivery record.

Benefits:

* Failed deliveries can be retried.
* Delivery history is preserved.
* Administrators can monitor queue status.
* Future enhancements can support asynchronous workers.

---

## SMTP Service

The `SmtpService` has a single responsibility:

* Build the email.
* Attach the PDF.
* Send it.

Keeping email logic separate from the orchestrator simplifies testing and future integrations with services such as Microsoft 365, Gmail, or Amazon SES.

---

## Workflow

```text
Generate ODT
      │
      ▼
Export PDF
      │
      ▼
Create Delivery Job
      │
      ▼
Send Email
      │
      ▼
Update Delivery Status
```

Each stage is independent, making it easier to recover from failures.

---

# CHALLENGE LAB

Enhance the output and delivery pipeline with the following tasks:

1. Add support for multiple attachments.
2. Send both the `.odt` and `.pdf` versions of the document.
3. Implement a retry mechanism that attempts to resend failed emails up to three times.
4. Add email templates with placeholders such as:

```text
{{first_name}}
{{invoice_number}}
```

5. Record the delivery duration in the database.

---

# TROUBLESHOOTING

| Problem                           | Cause                               | Solution                                                        |
| --------------------------------- | ----------------------------------- | --------------------------------------------------------------- |
| PDF export fails                  | Incorrect LibreOffice export filter | Use `writer_pdf_Export` for Writer documents.                   |
| Email not delivered               | SMTP configuration is incorrect     | Verify server, port, credentials, and TLS settings.             |
| Attachment missing                | Incorrect file path                 | Ensure the PDF exists before sending the email.                 |
| Delivery status remains `PENDING` | Status update not executed          | Update the queue after successful or failed delivery.           |
| SMTP authentication fails         | Invalid username or password        | Confirm credentials and avoid hardcoding secrets in production. |

---

# Chapter Summary

In this chapter, you:

* Exported Writer documents to PDF using the LibreOffice UNO API.
* Built a dedicated `PdfService` for document conversion.
* Implemented an `SmtpService` for email delivery with PDF attachments.
* Introduced a `delivery_queue` table to track delivery jobs.
* Updated the orchestrator to create delivery records, send emails, and record delivery outcomes.

Greymatter Docs now supports a complete document delivery workflow, transforming generated documents into deliverable business assets.

**Next:** **Part 9 – Professional Polish**, where we'll add batch processing, progress reporting, performance optimizations, configuration profiles, and operational enhancements that prepare Greymatter Docs for high-volume document generation.
