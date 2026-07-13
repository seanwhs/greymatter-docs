# Part 6 — Production Mechanics

## Building Logging, Configuration, Custom Exceptions, and Resilient Error Handling

> **Goal:** Transform Greymatter Docs into a production-ready application by implementing structured logging, centralized configuration, custom exception classes, validation, and consistent error handling.

**Milestone:** By the end of this chapter, every module in Greymatter Docs will use a common logging framework, application configuration, and custom exception hierarchy, making the platform easier to debug, monitor, and maintain.

---

# THEORY & ARCHITECTURE

A document generation platform must be reliable. When something goes wrong—such as a missing template, a failed database query, or a lost LibreOffice connection—it should:

* Log the problem with enough detail for diagnosis.
* Raise a meaningful exception.
* Clean up any resources.
* Continue processing where appropriate.
* Avoid exposing internal errors to end users.

To achieve this, we'll introduce four production components:

1. **Centralized Configuration**
2. **Structured Logging**
3. **Custom Exceptions**
4. **Validation Utilities**

The updated architecture:

```text
                     main.py
                        │
                        ▼
                Document Pipeline
                        │
        ┌───────────────┼────────────────┐
        ▼               ▼                ▼
  Repository      UNO Services     Processor Engine
        │               │                │
        └───────────────┼────────────────┘
                        ▼
                 Custom Exceptions
                        │
                        ▼
                 Structured Logging
                        │
                        ▼
                   Log Files / Console
```

Every module follows the same error-handling pattern, ensuring consistent behavior across the application.

---

# BUILD IT

## Step 1 — Update the Project Structure

```text
greymatter_docs/
│
├── config/
│   ├── logger.py
│   ├── settings.py
│   └── validation.py
│
├── exceptions/
│   ├── __init__.py
│   ├── database_exceptions.py
│   ├── processor_exceptions.py
│   ├── service_exceptions.py
│   └── validation_exceptions.py
│
├── database/
├── processor/
├── services/
├── templates/
├── output/
├── logs/
└── main.py
```

---

## Step 2 — Improve Configuration Management

**config/settings.py**

```python
from pathlib import Path
import os

BASE_DIR = Path(__file__).resolve().parent.parent

OUTPUT_DIR = BASE_DIR / "output"
LOG_DIR = BASE_DIR / "logs"
TEMPLATE_DIR = BASE_DIR / "templates"

DATABASE_FILE = BASE_DIR / "greymatter.db"

LOG_FILE = LOG_DIR / "greymatter.log"

LOG_LEVEL = os.getenv("LOG_LEVEL", "INFO")

UNO_HOST = os.getenv("UNO_HOST", "localhost")
UNO_PORT = int(os.getenv("UNO_PORT", 2002))

APPLICATION_NAME = "Greymatter Docs"
APPLICATION_VERSION = "1.0.0"
```

Notice that configurable values are read from environment variables, making the application easier to deploy across environments.

---

## Step 3 — Create a Custom Exception Hierarchy

### `exceptions/database_exceptions.py`

```python
class DatabaseError(Exception):
    """Base exception for database operations."""


class DatabaseConnectionError(DatabaseError):
    """Raised when a database connection cannot be established."""


class QueryExecutionError(DatabaseError):
    """Raised when a SQL query fails."""
```

---

### `exceptions/service_exceptions.py`

```python
class LibreOfficeConnectionError(Exception):
    """Raised when LibreOffice cannot be reached."""


class DocumentOpenError(Exception):
    """Raised when a document cannot be opened."""


class DocumentSaveError(Exception):
    """Raised when a document cannot be saved."""
```

---

### `exceptions/processor_exceptions.py`

```python
class PlaceholderError(Exception):
    """Raised during placeholder processing."""


class TableProcessingError(Exception):
    """Raised when table generation fails."""
```

---

### `exceptions/validation_exceptions.py`

```python
class ValidationError(Exception):
    """Raised when input validation fails."""
```

---

## Step 4 — Build a Validation Utility

**config/validation.py**

```python
from pathlib import Path

from exceptions.validation_exceptions import ValidationError


def validate_file_exists(file_path: Path):
    """
    Ensure the supplied file exists.
    """

    if not file_path.exists():
        raise ValidationError(
            f"File not found: {file_path}"
        )


def validate_directory(directory: Path):
    """
    Create the directory if necessary.
    """

    directory.mkdir(
        parents=True,
        exist_ok=True
    )
```

These helper functions centralize common validation tasks.

---

## Step 5 — Improve Logging

**config/logger.py**

```python
import logging
from logging.handlers import RotatingFileHandler

from config.settings import (
    LOG_FILE,
    LOG_LEVEL,
)

LOG_FILE.parent.mkdir(
    parents=True,
    exist_ok=True
)

formatter = logging.Formatter(
    "%(asctime)s | %(levelname)s | "
    "%(name)s | %(funcName)s | %(message)s"
)

file_handler = RotatingFileHandler(
    LOG_FILE,
    maxBytes=5 * 1024 * 1024,
    backupCount=5,
)

file_handler.setFormatter(formatter)

console_handler = logging.StreamHandler()
console_handler.setFormatter(formatter)

logger = logging.getLogger("greymatter")
logger.setLevel(LOG_LEVEL)

logger.addHandler(file_handler)
logger.addHandler(console_handler)
```

### Why a Rotating Log?

Without rotation, log files grow indefinitely.

Rotating logs automatically archive older logs while keeping the most recent logs available.

---

## Step 6 — Improve the Database Manager

**database/database.py**

```python
import logging
import sqlite3

from config.settings import DATABASE_FILE

from exceptions.database_exceptions import (
    DatabaseConnectionError,
)

logger = logging.getLogger(__name__)


class DatabaseManager:

    def get_connection(self):

        try:

            connection = sqlite3.connect(
                DATABASE_FILE
            )

            connection.row_factory = sqlite3.Row

            logger.info(
                "Connected to SQLite."
            )

            return connection

        except sqlite3.Error as error:

            logger.exception(error)

            raise DatabaseConnectionError(
                "Unable to connect to SQLite."
            ) from error
```

The application now raises a meaningful exception instead of a generic `RuntimeError`.

---

## Step 7 — Improve the LibreOffice Service

Replace generic exceptions with domain-specific exceptions.

```python
from exceptions.service_exceptions import (
    LibreOfficeConnectionError,
)
```

Then:

```python
except Exception as error:

    logger.exception(error)

    raise LibreOfficeConnectionError(
        "Unable to establish UNO connection."
    ) from error
```

This makes it much easier to determine the source of a failure.

---

## Step 8 — Centralize Error Handling in `main.py`

```python
from config.logger import logger

from exceptions.database_exceptions import (
    DatabaseError,
)

from exceptions.service_exceptions import (
    LibreOfficeConnectionError,
)

from exceptions.processor_exceptions import (
    PlaceholderError,
    TableProcessingError,
)

from exceptions.validation_exceptions import (
    ValidationError,
)
```

```python
try:

    # Execute document generation

except ValidationError as error:

    logger.error(error)

except DatabaseError as error:

    logger.error(error)

except LibreOfficeConnectionError as error:

    logger.error(error)

except PlaceholderError as error:

    logger.error(error)

except TableProcessingError as error:

    logger.error(error)

except Exception as error:

    logger.exception(
        "Unexpected application error."
    )
```

The main pipeline becomes the single location for handling top-level failures.

---

# CODE WALKTHROUGH

## Why Custom Exceptions?

Consider two different failures:

```text
Database connection failed.
```

and

```text
Unable to save generated document.
```

Both might otherwise appear as generic `RuntimeError` exceptions, making diagnosis difficult.

Custom exceptions provide context and improve maintainability.

---

## Why Validate Early?

Instead of allowing the application to fail halfway through processing, validate prerequisites before starting.

For example:

```python
validate_file_exists(template_path)
```

If the template is missing, the application fails fast with a clear error message.

---

## Logging Levels

Use logging levels consistently:

| Level      | Purpose                                    |
| ---------- | ------------------------------------------ |
| `DEBUG`    | Detailed development information           |
| `INFO`     | Normal application events                  |
| `WARNING`  | Recoverable issues                         |
| `ERROR`    | Operation failed but application continues |
| `CRITICAL` | Application cannot continue                |

This allows log filtering based on deployment needs.

---

## Exception Chaining

Notice the use of:

```python
raise DatabaseConnectionError(...) from error
```

This preserves the original exception and stack trace while exposing a more meaningful application-level error.

---

# CHALLENGE LAB

Enhance the production mechanics with the following tasks:

1. Add a `ConfigurationError` exception for invalid application settings.
2. Validate that the output directory is writable before generating documents.
3. Add a log entry recording the start and end time of each document generation job.
4. Implement a decorator that logs the execution time of repository methods.
5. Configure separate log files for:

   * Application events
   * Errors only

---

# TROUBLESHOOTING

| Problem                                     | Cause                                        | Solution                                                           |
| ------------------------------------------- | -------------------------------------------- | ------------------------------------------------------------------ |
| Log file grows too large                    | Log rotation not configured                  | Use `RotatingFileHandler` with appropriate size and backup limits. |
| Generic exceptions make debugging difficult | Application raises `RuntimeError` everywhere | Introduce domain-specific exception classes.                       |
| Missing templates cause unexpected crashes  | Validation performed too late                | Validate files before processing begins.                           |
| Important errors are missing from logs      | Incorrect logging level                      | Ensure the logger level includes the desired severity.             |
| Stack trace lost                            | Exception re-raised without chaining         | Use `raise ... from error` to preserve the original cause.         |

---

# Chapter Summary

In this chapter, you:

* Centralized application configuration.
* Introduced a custom exception hierarchy.
* Enhanced logging with rotating log files.
* Added reusable validation utilities.
* Improved the database and UNO services with domain-specific exceptions.
* Consolidated top-level error handling in the application pipeline.

Greymatter Docs now has the operational foundations expected of a production-ready application. Failures are easier to diagnose, logs are more informative, and the codebase is better prepared for future growth.

**Next:** **Part 7 – The Orchestrator**, where we'll build the end-to-end automation pipeline, coordinate repositories, processors, and services, introduce a job workflow, and enable fully automated document generation from a single entry point.
