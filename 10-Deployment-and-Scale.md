# Part 10 — Deployment & Scale

## Scheduling, Environment Configuration, System Hardening, and Production Deployment

> **Goal:** Prepare Greymatter Docs for production by implementing environment-based configuration, scheduled execution, graceful shutdown, health checks, deployment automation, and operational hardening.

**Milestone:** By the end of this chapter, Greymatter Docs will be ready to run as an unattended production service capable of processing scheduled document generation jobs reliably and securely.

---

# THEORY & ARCHITECTURE

By now, Greymatter Docs can:

* Read data from SQLite
* Process LibreOffice templates
* Generate `.odt` documents
* Export PDF files
* Deliver documents by email
* Process batch jobs
* Monitor execution
* Generate operational reports

The final step is making the application suitable for continuous operation.

The production architecture:

```text
                    Scheduler
               (Cron / Task Scheduler)
                       │
                       ▼
                Greymatter Docs
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
   SQLite         LibreOffice      SMTP Server
        │              │              │
        └──────────────┼──────────────┘
                       ▼
               Generated Documents
                       │
                       ▼
                 Log Files / Reports
```

Instead of being started manually, the platform will execute automatically based on a schedule.

---

# BUILD IT

## Step 1 — Environment-Based Configuration

Create a `.env` file.

```text
APPLICATION_NAME=Greymatter Docs

LOG_LEVEL=INFO

UNO_HOST=localhost
UNO_PORT=2002

DATABASE_FILE=greymatter.db

SMTP_SERVER=smtp.example.com
SMTP_PORT=587

SMTP_USERNAME=username
SMTP_PASSWORD=password

OUTPUT_DIRECTORY=output
```

> **Security Note:** Never commit `.env` files containing secrets to version control. Add `.env` to your `.gitignore` file.

---

## Step 2 — Load Environment Variables

Update **config/settings.py**

```python
from pathlib import Path
import os

from dotenv import load_dotenv

load_dotenv()

BASE_DIR = Path(__file__).resolve().parent.parent

APPLICATION_NAME = os.getenv(
    "APPLICATION_NAME",
    "Greymatter Docs"
)

LOG_LEVEL = os.getenv(
    "LOG_LEVEL",
    "INFO"
)

UNO_HOST = os.getenv(
    "UNO_HOST",
    "localhost"
)

UNO_PORT = int(
    os.getenv("UNO_PORT", 2002)
)

DATABASE_FILE = (
    BASE_DIR /
    os.getenv(
        "DATABASE_FILE",
        "greymatter.db"
    )
)

OUTPUT_DIR = (
    BASE_DIR /
    os.getenv(
        "OUTPUT_DIRECTORY",
        "output"
    )
)
```

This allows the same codebase to run in development, testing, and production with different configurations.

---

## Step 3 — Add a Health Check Service

Create:

`services/health_service.py`

```python
import logging
from pathlib import Path

from config.settings import (
    DATABASE_FILE,
    OUTPUT_DIR,
)

logger = logging.getLogger(__name__)


class HealthService:
    """Performs startup health checks."""

    def check_database(self):
        if not DATABASE_FILE.exists():
            raise FileNotFoundError(
                DATABASE_FILE
            )

        logger.info(
            "Database OK"
        )

    def check_output_directory(self):
        OUTPUT_DIR.mkdir(
            parents=True,
            exist_ok=True
        )

        logger.info(
            "Output directory OK"
        )

    def run(self):
        self.check_database()
        self.check_output_directory()

        logger.info(
            "Health check passed."
        )
```

Before processing jobs, verify that critical resources are available.

---

## Step 4 — Graceful Shutdown

Update `main.py`

```python
import signal
import sys

from config.logger import logger


def shutdown_handler(
    signal_number,
    frame
):
    logger.info(
        "Shutdown requested."
    )

    sys.exit(0)


signal.signal(
    signal.SIGINT,
    shutdown_handler
)

signal.signal(
    signal.SIGTERM,
    shutdown_handler
)
```

This ensures the application exits cleanly when interrupted.

---

## Step 5 — Create a Deployment Script

### Windows

`deploy.bat`

```batch
@echo off

call venv\Scripts\activate

python main.py

pause
```

---

### Linux

`deploy.sh`

```bash
#!/bin/bash

source venv/bin/activate

python main.py
```

Make executable:

```bash
chmod +x deploy.sh
```

---

## Step 6 — Schedule Execution

### Windows Task Scheduler

Create a scheduled task:

| Setting   | Value             |
| --------- | ----------------- |
| Trigger   | Daily / Hourly    |
| Program   | `python.exe`      |
| Arguments | `main.py`         |
| Start In  | Project directory |

---

### Linux Cron

Edit the crontab:

```bash
crontab -e
```

Example:

```bash
0 * * * * /home/user/greymatter_docs/deploy.sh
```

This runs the application at the start of every hour.

---

## Step 7 — Production Logging

Store logs separately:

```text
logs/

application.log

error.log

audit.log

execution.log
```

Purpose:

| File              | Contents              |
| ----------------- | --------------------- |
| `application.log` | Normal operations     |
| `error.log`       | Errors and exceptions |
| `audit.log`       | Business events       |
| `execution.log`   | Batch summaries       |

Separating logs simplifies monitoring and troubleshooting.

---

## Step 8 — Backup Strategy

Recommended directories:

```text
backups/

database/

generated_documents/

logs/
```

Automate backups before:

* Database migrations
* Major releases
* Batch processing
* System upgrades

---

# CODE WALKTHROUGH

## Environment Configuration

Using `.env` files allows the application to adapt to different environments without changing code.

Typical environments include:

* Development
* Testing
* Staging
* Production

---

## Health Checks

Running validation at startup helps detect configuration issues before long-running jobs begin.

Checks might include:

* Database availability
* Template directory existence
* Output directory accessibility
* LibreOffice connectivity
* SMTP configuration

---

## Graceful Shutdown

Without signal handling, interrupted jobs may leave:

* Partially written documents
* Open database transactions
* Locked files

Handling termination signals ensures resources are released cleanly.

---

## Scheduling

Greymatter Docs is now designed to run automatically at regular intervals, making it suitable for:

* Nightly invoice generation
* Weekly reporting
* Monthly statements
* Automated HR document production

---

## Operational Readiness

A production deployment requires more than working code. Operational practices such as structured logging, backups, configuration management, and health checks improve reliability and simplify maintenance.

---

# CHALLENGE LAB

Complete the following enhancements:

1. Add a startup banner displaying:

   * Application version
   * Python version
   * Operating system
   * Current environment

2. Create a `HealthReport` summarizing all startup checks.

3. Add automatic cleanup for generated documents older than 30 days.

4. Rotate backup folders weekly and retain the last four backups.

5. Implement a command-line interface (CLI) supporting:

```bash
python main.py --generate

python main.py --health-check

python main.py --seed

python main.py --export-report
```

---

# TROUBLESHOOTING

| Problem                     | Cause                                      | Solution                                                              |
| --------------------------- | ------------------------------------------ | --------------------------------------------------------------------- |
| Scheduled task does not run | Incorrect working directory or permissions | Verify the scheduler's "Start In" directory and execution account.    |
| `.env` values not applied   | Environment file not loaded                | Ensure `load_dotenv()` is called before reading configuration values. |
| Health check fails          | Missing database or output directory       | Validate required files and create directories during startup.        |
| Application exits abruptly  | No signal handling                         | Register handlers for `SIGINT` and `SIGTERM`.                         |
| Logs grow indefinitely      | No log rotation or archival                | Configure rotating handlers and archive old log files.                |

---

# Chapter Summary

Congratulations—you have completed the **Greymatter Docs** series.

Across ten parts, you built a complete automated document generation platform using Python, SQLite, and the LibreOffice UNO API.

You implemented:

* A modular, production-ready project architecture.
* A SQLite-backed data access layer using the Repository pattern.
* Reusable LibreOffice Writer templates with placeholder processing.
* A UNO bridge for document automation.
* Dynamic table generation for repeating business data.
* Structured logging, validation, and custom exception handling.
* An orchestration layer for end-to-end automation.
* PDF export and email delivery with delivery tracking.
* Batch processing, performance monitoring, and execution reporting.
* Deployment tooling, scheduling, health checks, and operational hardening.

---

# Where to Go Next

The platform you've built is a strong foundation, but many real-world enhancements are possible:

| Enhancement                                              | Benefit                                                      |
| -------------------------------------------------------- | ------------------------------------------------------------ |
| **Template Versioning**                                  | Manage multiple revisions of business templates.             |
| **REST API**                                             | Trigger document generation from external applications.      |
| **Web Dashboard**                                        | Monitor jobs, queues, and delivery status through a browser. |
| **Authentication & Authorization**                       | Support multiple users and role-based access.                |
| **PostgreSQL or MySQL**                                  | Scale beyond a single-file SQLite database.                  |
| **Redis & Background Workers**                           | Process large queues asynchronously.                         |
| **Cloud Storage (S3, Azure Blob, Google Cloud Storage)** | Archive generated documents securely.                        |
| **Digital Signatures**                                   | Sign PDFs for legal and compliance workflows.                |
| **Docker & Containerization**                            | Simplify deployment and improve portability.                 |
| **Automated Testing (pytest)**                           | Build a comprehensive regression test suite.                 |

---

# Final Thoughts

Greymatter Docs demonstrates how a well-structured Python application can automate complex document workflows using open technologies. By combining modular architecture, clean coding practices, SQLite for data management, and the LibreOffice UNO API for document manipulation, you've built a platform that can be extended to support invoices, contracts, reports, certificates, HR forms, and countless other business documents.

The principles used throughout this series—single responsibility, separation of concerns, repository patterns, structured logging, and layered architecture—are applicable far beyond document automation. They provide a solid foundation for building maintainable, scalable business applications.
