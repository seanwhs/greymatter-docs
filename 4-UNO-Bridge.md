# Part 4 — The UNO Bridge

## Connecting Python to LibreOffice and Working with the Document Object Model

> **Goal:** Replace the temporary text-based processor with a real LibreOffice UNO integration that opens Writer templates, modifies document content, replaces placeholders, and saves completed `.odt` documents.

**Milestone:** By the end of this chapter, Greymatter Docs will establish a live socket connection to LibreOffice, load a `.ott` template, replace placeholders using the UNO Document Object Model (DOM), and save a generated `.odt` document.

---

# THEORY & ARCHITECTURE

In Part 3, we validated our placeholder engine using plain text files.

That approach helped us develop the processing logic, but real LibreOffice documents (`.odt` and `.ott`) are ZIP archives containing XML—not plain text.

Instead of editing XML directly, we'll use the **LibreOffice UNO API**, which provides a rich object model for interacting with Writer documents.

The updated architecture looks like this:

```text
                 SQLite
                    │
                    ▼
          CustomerRepository
                    │
                    ▼
          Python Dictionary
                    │
                    ▼
         DocumentProcessor
                    │
                    ▼
        LibreOfficeService
                    │
                    ▼
              UNO API
                    │
                    ▼
         LibreOffice Writer
                    │
                    ▼
         Generated ODT Document
```

The key principle is **separation of responsibilities**:

| Component            | Responsibility                  |
| -------------------- | ------------------------------- |
| `CustomerRepository` | Retrieves business data         |
| `PlaceholderEngine`  | Maps placeholders to values     |
| `DocumentProcessor`  | Coordinates document generation |
| `LibreOfficeService` | Manages the UNO connection      |
| UNO API              | Manipulates Writer documents    |

---

# Understanding UNO in Plain English

Think of UNO as a remote control for LibreOffice.

Instead of clicking buttons in Writer, Python sends commands such as:

* Open this document
* Find this text
* Replace it
* Save the document
* Export as PDF
* Close the document

LibreOffice performs those actions exactly as if a user were doing them manually.

---

# Starting LibreOffice in Server Mode

UNO communicates through a socket connection.

Start LibreOffice using the following command.

## Windows

```bash
"C:\Program Files\LibreOffice\program\soffice.exe" ^
--accept="socket,host=localhost,port=2002;urp;" ^
--headless ^
--nologo ^
--nodefault ^
--nofirststartwizard
```

## Linux

```bash
libreoffice \
--accept="socket,host=localhost,port=2002;urp;" \
--headless \
--nologo \
--nodefault
```

### What these options mean

| Option                 | Purpose                       |
| ---------------------- | ----------------------------- |
| `--headless`           | Run without opening the GUI   |
| `--accept`             | Enable UNO socket connections |
| `--nologo`             | Skip the splash screen        |
| `--nodefault`          | Don't open an empty document  |
| `--nofirststartwizard` | Skip first-run setup          |

Leave LibreOffice running while developing.

---

# BUILD IT

## Step 1 — Install the UNO Python Module

If you're using LibreOffice's bundled Python, `uno` is already available.

If you're using a standalone Python installation, ensure it can import the LibreOffice UNO libraries.

Verify:

```python
import uno
```

If no exception is raised, you're ready to continue.

> **Platform Note:** Configuring the `uno` module can differ between operating systems. On some systems, you may need to add LibreOffice's `program` directory to your `PYTHONPATH`.

---

## Step 2 — Update the Project Structure

```text
greymatter_docs/
│
├── config/
├── database/
├── processor/
│
├── services/
│   ├── __init__.py
│   ├── libreoffice_service.py
│   └── writer_service.py
│
├── templates/
├── output/
├── logs/
└── main.py
```

We'll separate connection management from document manipulation.

---

## Step 3 — Build the LibreOffice Connection Service

**services/libreoffice_service.py**

```python
import logging

import uno
from com.sun.star.connection import NoConnectException

logger = logging.getLogger(__name__)


class LibreOfficeService:
    """Manages the UNO connection."""

    def __init__(self):
        self.local_context = None
        self.resolver = None
        self.context = None
        self.desktop = None

    def connect(self):
        """Connect to a running LibreOffice instance."""

        try:
            self.local_context = uno.getComponentContext()

            self.resolver = (
                self.local_context.ServiceManager.createInstanceWithContext(
                    "com.sun.star.bridge.UnoUrlResolver",
                    self.local_context
                )
            )

            self.context = self.resolver.resolve(
                "uno:socket,host=localhost,port=2002;urp;StarOffice.ComponentContext"
            )

            self.desktop = (
                self.context.ServiceManager.createInstanceWithContext(
                    "com.sun.star.frame.Desktop",
                    self.context
                )
            )

            logger.info("Connected to LibreOffice.")

        except NoConnectException as error:
            logger.exception("LibreOffice is not running.")
            raise RuntimeError(
                "Unable to connect to LibreOffice."
            ) from error

        except Exception as error:
            logger.exception("UNO connection failed.")
            raise RuntimeError(
                "UNO initialization failed."
            ) from error
```

This class creates a reusable connection that other services can share.

---

## Step 4 — Build the Writer Service

**services/writer_service.py**

```python
import logging
from pathlib import Path

import uno


logger = logging.getLogger(__name__)


class WriterService:
    """Provides Writer document operations."""

    def __init__(self, desktop):
        self.desktop = desktop

    @staticmethod
    def _to_file_url(path: Path) -> str:
        return uno.systemPathToFileUrl(str(path.resolve()))

    def open_document(self, template_path: Path):
        """Open a Writer template."""

        try:
            document = self.desktop.loadComponentFromURL(
                self._to_file_url(template_path),
                "_blank",
                0,
                ()
            )

            logger.info("Template opened: %s", template_path)

            return document

        except Exception as error:
            logger.exception("Unable to open template.")
            raise RuntimeError(
                "Template could not be opened."
            ) from error

    def save_document(self, document, output_path: Path):
        """Save the generated document."""

        try:
            document.storeAsURL(
                self._to_file_url(output_path),
                ()
            )

            logger.info(
                "Document saved: %s",
                output_path
            )

        except Exception as error:
            logger.exception("Unable to save document.")
            raise RuntimeError(
                "Document save failed."
            ) from error

    @staticmethod
    def close_document(document):
        """Close the document."""

        document.close(True)
```

---

## Step 5 — Replace Placeholders in Writer

We'll search every paragraph and replace placeholders.

**processor/document_processor.py**

```python
import logging

from processor.placeholder_engine import PlaceholderEngine

logger = logging.getLogger(__name__)


class DocumentProcessor:
    """Processes Writer documents."""

    def replace_placeholders(
        self,
        document,
        values
    ):
        """Replace placeholders in the document."""

        try:
            text = document.Text

            cursor = text.createTextCursor()

            search = document.createSearchDescriptor()

            for key, value in values.items():

                search.setSearchString(
                    "{{" + key + "}}"
                )

                found = document.findFirst(search)

                while found:

                    found.setString(str(value))

                    found = document.findNext(
                        found.End,
                        search
                    )

            logger.info(
                "Placeholder replacement complete."
            )

        except Exception as error:
            logger.exception(
                "Replacement failed."
            )

            raise RuntimeError(
                "Unable to replace placeholders."
            ) from error
```

Notice that we use Writer's built-in search engine instead of manually walking XML nodes. This keeps the code simple and leverages the UNO API.

---

## Step 6 — Update `main.py`

```python
from config.logger import logger
from config.settings import OUTPUT_DIR

from database.repositories import CustomerRepository
from database.schema import create_schema
from database.seed import seed_database

from processor.document_processor import DocumentProcessor

from services.libreoffice_service import LibreOfficeService
from services.writer_service import WriterService

from processor.template_loader import TemplateLoader


def main():

    logger.info("=" * 60)

    try:

        create_schema()
        seed_database()

        repository = CustomerRepository()

        customer = repository.get_all_customers()[0]

        values = dict(customer)

        libreoffice = LibreOfficeService()

        libreoffice.connect()

        writer = WriterService(
            libreoffice.desktop
        )

        template = TemplateLoader.get_template_path(
            "customer_letter.ott"
        )

        document = writer.open_document(template)

        processor = DocumentProcessor()

        processor.replace_placeholders(
            document,
            values
        )

        OUTPUT_DIR.mkdir(exist_ok=True)

        writer.save_document(
            document,
            OUTPUT_DIR / "customer_letter.odt"
        )

        writer.close_document(document)

        logger.info(
            "Document generated successfully."
        )

    except Exception as error:
        logger.exception(error)


if __name__ == "__main__":
    main()
```

Run:

```bash
python main.py
```

Expected result:

```text
INFO Connected to LibreOffice.
INFO Template opened.
INFO Placeholder replacement complete.
INFO Document saved.
INFO Document generated successfully.
```

A completed `.odt` document will appear in:

```text
output/customer_letter.odt
```

---

# CODE WALKTHROUGH

## UNO Connection

The `LibreOfficeService` establishes a socket connection to the running LibreOffice process and exposes the `Desktop` object, which serves as the entry point for opening and creating documents.

---

## `WriterService`

This service encapsulates document lifecycle operations:

* Open a template
* Save a document
* Close a document

Keeping these operations in one class simplifies testing and future enhancements, such as PDF export.

---

## `DocumentProcessor`

Rather than editing document XML directly, it uses Writer's search functionality to locate placeholders and replace them with data.

This approach is:

* Easier to understand
* Less error-prone
* Independent of the document's XML structure

---

## File URLs

UNO expects **file URLs**, not operating system paths.

This conversion:

```python
uno.systemPathToFileUrl(...)
```

ensures compatibility across Windows, macOS, and Linux.

---

# CHALLENGE LAB

Extend the UNO integration with the following tasks:

1. Replace placeholders in:

   * Headers
   * Footers
   * Text boxes

2. Add support for:

```text
{{generated_date}}
```

using today's date.

3. Log every placeholder that is replaced.

4. Add a method:

```python
document_is_modified()
```

that checks whether the document contains unsaved changes.

5. Generate three customer documents in a single execution.

---

# TROUBLESHOOTING

| Problem                                          | Cause                                           | Solution                                                                                       |
| ------------------------------------------------ | ----------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `ImportError: No module named uno`               | Python cannot locate the UNO libraries          | Configure your environment to use LibreOffice's UNO Python bindings or update `PYTHONPATH`.    |
| `NoConnectException`                             | LibreOffice is not running in server mode       | Start LibreOffice with the `--accept` and `--headless` options before running the application. |
| Template opens but placeholders are not replaced | Placeholder names do not match the data keys    | Ensure template placeholders exactly match the dictionary keys (e.g., `{{first_name}}`).       |
| `IllegalArgumentException` when saving           | Invalid output path or file URL                 | Verify the output directory exists and use `uno.systemPathToFileUrl()`.                        |
| Generated document is empty                      | Incorrect template path or failed document load | Confirm the template exists and `loadComponentFromURL()` returns a valid document.             |

---

# Chapter Summary

In this chapter, you:

* Established a live UNO socket connection to LibreOffice.
* Built a reusable `LibreOfficeService` for connection management.
* Implemented a `WriterService` to open, save, and close Writer documents.
* Replaced placeholders directly within `.ott` templates using the UNO Document Object Model.
* Generated your first real `.odt` document from database data.

Greymatter Docs has now moved beyond prototype code and into true document automation.

**Next:** **Part 5 — The Processor Engine**, where we'll expand the document processor to support repeating sections, dynamic tables, conditional content, and multi-record document generation. This is where Greymatter Docs evolves from simple mail merges into a full document generation engine.
