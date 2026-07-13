# Part 1 — The Foundations & Initial Setup

> **Goal:** Build and verify the development environment, create the Greymatter Docs project structure, configure logging, and confirm Python can communicate with LibreOffice.

**Milestone:** By the end of this chapter, you'll have a runnable project with a production-ready directory layout, centralized configuration, logging, and a successful connection test to a headless LibreOffice instance.

---

# THEORY & ARCHITECTURE

Before we can generate documents, we need four core components working together:

1. **Python** — Our application.
2. **LibreOffice** — The document engine.
3. **UNO API** — The bridge between Python and LibreOffice.
4. **Project Structure** — A clean, maintainable codebase.

The architecture at this stage is simple:

```text
             Python Application
                    │
                    ▼
             Configuration
                    │
                    ▼
              Logging System
                    │
                    ▼
             LibreOffice UNO
                    │
                    ▼
           Document Processing
```

Everything we build in later chapters depends on this foundation.

---

# BUILD IT

## Step 1 — Install Python

Install **Python 3.11 or later** from:

[https://python.org](https://python.org)

Verify the installation:

```bash
python --version
```

Expected output:

```text
Python 3.11.x
```

---

## Step 2 — Install LibreOffice

Download LibreOffice from:

[https://www.libreoffice.org](https://www.libreoffice.org)

Verify the installation:

### Windows

```bash
soffice --version
```

### Linux

```bash
libreoffice --version
```

Expected output:

```text
LibreOffice 25.x
```

(Your version may differ.)

---

## Step 3 — Create the Project

```text
greymatter_docs/
│
├── config/
│   ├── __init__.py
│   └── settings.py
│
├── database/
│   └── __init__.py
│
├── processor/
│   └── __init__.py
│
├── services/
│   └── __init__.py
│
├── templates/
│
├── output/
│
├── logs/
│
├── tests/
│
├── requirements.txt
│
└── main.py
```

Create the folders manually or with your preferred IDE.

---

## Step 4 — Create a Virtual Environment

Windows

```bash
python -m venv venv
```

Activate:

```bash
venv\Scripts\activate
```

Linux/macOS

```bash
python3 -m venv venv
```

Activate:

```bash
source venv/bin/activate
```

Upgrade pip:

```bash
python -m pip install --upgrade pip
```

---

## Step 5 — Create `requirements.txt`

```text
python-dotenv
```

We'll add more packages later as the project grows.

Install:

```bash
pip install -r requirements.txt
```

---

## Step 6 — Create Application Settings

`config/settings.py`

```python
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

OUTPUT_DIR = BASE_DIR / "output"
LOG_DIR = BASE_DIR / "logs"
TEMPLATE_DIR = BASE_DIR / "templates"

LOG_FILE = LOG_DIR / "greymatter.log"

APPLICATION_NAME = "Greymatter Docs"
APPLICATION_VERSION = "1.0.0"
```

Using `pathlib` makes file handling portable across operating systems.

---

## Step 7 — Configure Logging

Create:

`config/logger.py`

```python
import logging

from config.settings import LOG_FILE

LOG_FILE.parent.mkdir(exist_ok=True)

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)s | %(name)s | %(message)s",
    handlers=[
        logging.FileHandler(LOG_FILE),
        logging.StreamHandler()
    ]
)

logger = logging.getLogger("greymatter")
```

This configuration writes logs to both:

* Console
* Log file

---

## Step 8 — Create the UNO Service

Create:

`services/libreoffice_service.py`

```python
import logging

logger = logging.getLogger(__name__)


class LibreOfficeService:
    """
    Handles communication with LibreOffice.
    """

    def __init__(self):
        self.connected = False

    def connect(self):
        """
        Simulate a connection to LibreOffice.
        """
        try:
            logger.info("Attempting LibreOffice connection...")

            # Actual UNO connection comes in Part 4.

            self.connected = True

            logger.info("Connection successful.")

        except Exception as error:
            logger.exception("LibreOffice connection failed.")

            raise RuntimeError(
                "Unable to connect to LibreOffice."
            ) from error
```

For now, this is a stub. We'll replace it with a real UNO socket connection in Part 4 without changing the rest of the application.

---

## Step 9 — Create the Main Application

`main.py`

```python
from config.logger import logger
from config.settings import APPLICATION_NAME
from config.settings import APPLICATION_VERSION
from services.libreoffice_service import LibreOfficeService


def main():
    """
    Application entry point.
    """
    logger.info("=" * 60)
    logger.info("%s", APPLICATION_NAME)
    logger.info("Version %s", APPLICATION_VERSION)

    libreoffice = LibreOfficeService()

    try:
        libreoffice.connect()

        logger.info("Environment successfully initialized.")

    except RuntimeError as error:
        logger.error("%s", error)


if __name__ == "__main__":
    main()
```

Run the application:

```bash
python main.py
```

Expected output:

```text
INFO Greymatter Docs
INFO Attempting LibreOffice connection...
INFO Connection successful.
INFO Environment successfully initialized.
```

A log file named `greymatter.log` should also appear in the `logs/` directory.

---

# CODE WALKTHROUGH

### `settings.py`

Centralizes configuration values, avoiding hard-coded paths throughout the project.

### `logger.py`

Creates a single logging configuration that every module can reuse.

### `LibreOfficeService`

Acts as the application's gateway to LibreOffice. Using a dedicated service class keeps external dependencies isolated from business logic.

### `main.py`

Serves as the orchestrator. It initializes the application, starts logging, and verifies that the environment is ready. As the series progresses, this file will evolve into the main automation pipeline.

---

# CHALLENGE LAB

Complete the following tasks:

1. Change the application version to `1.1.0` and verify it appears in the log output.
2. Add a new configuration value:

```python
DATABASE_NAME = "greymatter.db"
```

3. Modify `LibreOfficeService.connect()` to log the message:

```text
LibreOffice service ready.
```

after a successful connection.

4. Create a new method:

```python
disconnect()
```

that logs:

```text
Disconnecting from LibreOffice...
```

and sets `connected` to `False`.

5. Update `main.py` to call `disconnect()` before exiting.

---

# TROUBLESHOOTING

| Problem                            | Cause                                             | Solution                                                                   |
| ---------------------------------- | ------------------------------------------------- | -------------------------------------------------------------------------- |
| `ModuleNotFoundError`              | Incorrect project structure or execution path     | Run `python main.py` from the project root.                                |
| No log file created                | `logs/` directory missing or no write permission  | Ensure the directory exists and is writable.                               |
| `PermissionError` writing logs     | File locked or insufficient permissions           | Close applications using the log file or run with appropriate permissions. |
| `python` command not found         | Python is not installed or not on the system PATH | Reinstall Python and enable the "Add Python to PATH" option.               |
| Virtual environment not activating | Incorrect activation command                      | Use the activation command appropriate for your operating system.          |

---

# Chapter Summary

In this chapter, you:

* Installed the core development tools.
* Created a modular project structure.
* Configured centralized application settings.
* Implemented production-ready logging.
* Built a stub `LibreOfficeService` with logging and exception handling.
* Created the application's entry point.
* Verified that the environment initializes successfully.

This foundation will remain largely unchanged throughout the series, allowing us to focus on adding new capabilities rather than reorganizing the project.

**Next:** **Part 2 — Data Strategy**, where we'll design the SQLite schema, implement a reusable data access layer, and build the first repository classes that power Greymatter Docs.
