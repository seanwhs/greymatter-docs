# Part 3 — Template Architecture

## Designing Professional LibreOffice Templates and Building the Placeholder Engine

> **Goal:** Design reusable LibreOffice Writer templates (`.ott`), define a placeholder convention, and build the first template processor that discovers and replaces placeholders using regular expressions.

**Milestone:** By the end of this chapter, Greymatter Docs will be able to load a template, identify placeholders, replace them with database values, and save a completed document.

---

# THEORY & ARCHITECTURE

A document template separates **presentation** from **data**.

Instead of hardcoding text into Python, we create a professional LibreOffice template with placeholders that are replaced at runtime.

For example:

```text
Customer Name : {{first_name}} {{last_name}}

Company       : {{company}}

Email         : {{email}}

Country       : {{country}}
```

When processed, the document becomes:

```text
Customer Name : John Smith

Company       : Greymatter Consulting

Email         : john@example.com

Country       : Singapore
```

This separation allows non-programmers to update document layouts without changing the application code.

---

## Template Processing Architecture

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
          Template Processor
                   │
                   ▼
        Placeholder Engine (Regex)
                   │
                   ▼
          LibreOffice Template
                   │
                   ▼
          Completed Document
```

Notice that the processor doesn't know where the data came from—it only receives a dictionary of values. This loose coupling makes the engine reusable.

---

# Placeholder Convention

We'll adopt a simple and consistent syntax:

```text
{{field_name}}
```

Examples:

```text
{{first_name}}

{{last_name}}

{{company}}

{{email}}

{{invoice_number}}

{{date}}

{{total_amount}}
```

**Rules:**

* Always use double curly braces.
* Placeholder names should match database field names where possible.
* Use lowercase and underscores.
* Avoid spaces or special characters.

This convention will remain consistent throughout the series.

---

# BUILD IT

## Step 1 — Update the Project Structure

```text
greymatter_docs/
│
├── config/
├── database/
├── processor/
│   ├── __init__.py
│   ├── placeholder_engine.py
│   ├── template_processor.py
│   └── template_loader.py
│
├── services/
├── templates/
│   ├── customer_letter.odt
│   └── customer_letter.ott
│
├── output/
├── logs/
└── main.py
```

> **Note:** For now, create the template as an `.odt` while designing it in LibreOffice. Once finalized, save it as an `.ott` (Document Template).

---

## Step 2 — Create a Sample Template

Open LibreOffice Writer and create a document like this:

```text
------------------------------------------------

Customer Information

Customer Name:
{{first_name}} {{last_name}}

Company:
{{company}}

Email:
{{email}}

Phone:
{{phone}}

Address:
{{address}}

City:
{{city}}

Country:
{{country}}

------------------------------------------------

Thank you for choosing our services.

Greymatter Docs

------------------------------------------------
```

Save it as:

```
templates/customer_letter.ott
```

---

## Step 3 — Create the Template Loader

**processor/template_loader.py**

```python
from pathlib import Path

from config.settings import TEMPLATE_DIR


class TemplateLoader:
    """Loads template files from disk."""

    @staticmethod
    def get_template_path(template_name: str) -> Path:
        return TEMPLATE_DIR / template_name
```

Keeping file path logic in one place makes it easier to support multiple template locations later.

---

## Step 4 — Build the Placeholder Engine

**processor/placeholder_engine.py**

```python
import logging
import re
from typing import Dict, List

logger = logging.getLogger(__name__)


class PlaceholderEngine:
    """Finds and replaces placeholders."""

    PLACEHOLDER_PATTERN = re.compile(r"\{\{([a-zA-Z0-9_]+)\}\}")

    @classmethod
    def find_placeholders(cls, text: str) -> List[str]:
        """Return every placeholder found in text."""
        placeholders = cls.PLACEHOLDER_PATTERN.findall(text)

        logger.info(
            "Found %d placeholders.",
            len(placeholders)
        )

        return placeholders

    @classmethod
    def replace_placeholders(
        cls,
        text: str,
        values: Dict[str, str]
    ) -> str:
        """Replace placeholders using a dictionary."""

        def replacement(match):
            key = match.group(1)

            return str(values.get(key, match.group(0)))

        return cls.PLACEHOLDER_PATTERN.sub(
            replacement,
            text
        )
```

The regular expression captures only the placeholder name, making replacement straightforward.

---

## Step 5 — Create the Template Processor

For this chapter, we'll process plain text to validate our placeholder engine. In **Part 4**, we'll replace this with the LibreOffice UNO document model.

**processor/template_processor.py**

```python
import logging
from pathlib import Path

from processor.placeholder_engine import PlaceholderEngine

logger = logging.getLogger(__name__)


class TemplateProcessor:
    """Processes text-based templates."""

    def process_template(
        self,
        template_path: Path,
        output_path: Path,
        values: dict
    ):
        """Read, replace placeholders, and save output."""

        try:
            with open(
                template_path,
                "r",
                encoding="utf-8"
            ) as template_file:

                content = template_file.read()

            updated_content = (
                PlaceholderEngine.replace_placeholders(
                    content,
                    values
                )
            )

            with open(
                output_path,
                "w",
                encoding="utf-8"
            ) as output_file:

                output_file.write(updated_content)

            logger.info(
                "Generated document: %s",
                output_path
            )

        except Exception as error:
            logger.exception(
                "Template processing failed."
            )

            raise RuntimeError(
                "Unable to process template."
            ) from error
```

> **Why plain text?** An `.ott` or `.odt` file is a ZIP archive containing XML, not a plain text file. Until we introduce the UNO API in Part 4, we'll validate our placeholder logic using a text-based template.

---

## Step 6 — Create a Text Template for Testing

Create:

```
templates/customer_letter.txt
```

Contents:

```text
Customer Information

Customer Name:
{{first_name}} {{last_name}}

Company:
{{company}}

Email:
{{email}}

Phone:
{{phone}}

Address:
{{address}}

City:
{{city}}

Country:
{{country}}

Thank you for choosing Greymatter Docs.
```

---

## Step 7 — Update `main.py`

```python
from pathlib import Path

from config.logger import logger
from config.settings import OUTPUT_DIR
from database.repositories import CustomerRepository
from database.schema import create_schema
from database.seed import seed_database
from processor.template_loader import TemplateLoader
from processor.template_processor import TemplateProcessor


def main():
    logger.info("=" * 60)
    logger.info("Greymatter Docs Starting")

    try:
        create_schema()
        seed_database()

        repository = CustomerRepository()
        customer = repository.get_all_customers()[0]

        values = dict(customer)

        template_path = TemplateLoader.get_template_path(
            "customer_letter.txt"
        )

        output_path = (
            OUTPUT_DIR / "customer_letter_output.txt"
        )

        OUTPUT_DIR.mkdir(exist_ok=True)

        processor = TemplateProcessor()

        processor.process_template(
            template_path=template_path,
            output_path=output_path,
            values=values
        )

        logger.info("Document generation complete.")

    except Exception as error:
        logger.exception(error)


if __name__ == "__main__":
    main()
```

Run the application:

```bash
python main.py
```

Expected output file:

```
output/customer_letter_output.txt
```

Contents:

```text
Customer Information

Customer Name:
John Smith

Company:
Greymatter Consulting

Email:
john@example.com

Phone:
555-1234

Address:
100 Main Street

City:
Singapore

Country:
Singapore

Thank you for choosing Greymatter Docs.
```

---

# CODE WALKTHROUGH

## `TemplateLoader`

Encapsulates template path resolution. If templates move to another directory or storage service in the future, only this class needs to change.

---

## `PlaceholderEngine`

The heart of this chapter.

The regular expression:

```python
r"\{\{([a-zA-Z0-9_]+)\}\}"
```

matches placeholders like:

```text
{{first_name}}
{{email}}
{{postal_code}}
```

but ignores invalid formats such as:

```text
{{First Name}}
{{customer-email}}
```

Keeping placeholder names predictable simplifies both template design and data mapping.

---

## `replace_placeholders()`

The replacement function:

* Looks up the placeholder in the provided dictionary.
* Replaces it if found.
* Leaves the original placeholder unchanged if no matching value exists.

This behavior makes missing data easy to spot during testing.

---

## `TemplateProcessor`

For now, it performs three simple tasks:

1. Read the template.
2. Replace placeholders.
3. Save the generated document.

In Part 4, we'll replace file-based text processing with direct manipulation of LibreOffice documents through the UNO API.

---

# CHALLENGE LAB

Enhance the placeholder engine with the following features:

1. Add support for:

```text
{{current_date}}
```

that inserts today's date.

2. Log every placeholder that could not be resolved.

3. Add a method:

```python
validate_placeholders(
    template_text,
    values
)
```

that returns a list of unresolved placeholders before processing.

4. Create a second template named:

```
invoice_template.txt
```

using at least 10 placeholders.

5. Generate both templates from the same customer data.

---

# TROUBLESHOOTING

| Problem                            | Cause                                      | Solution                                                                              |
| ---------------------------------- | ------------------------------------------ | ------------------------------------------------------------------------------------- |
| Output still contains placeholders | Missing keys in the values dictionary      | Ensure placeholder names match dictionary keys exactly.                               |
| `FileNotFoundError`                | Template file not found                    | Verify the template exists in the `templates/` directory and the filename is correct. |
| `UnicodeDecodeError`               | Incorrect file encoding                    | Save templates as UTF-8.                                                              |
| Empty output file                  | Template read failed or file was empty     | Confirm the template contains content and is opened successfully.                     |
| `PermissionError` writing output   | Output file is open in another application | Close the file before regenerating it.                                                |

---

# Chapter Summary

In this chapter, you:

* Designed a reusable placeholder convention.
* Created a simple template architecture.
* Built a regex-powered placeholder engine.
* Implemented a template loader and processor.
* Generated the first document from live database data.
* Prepared the codebase for direct LibreOffice integration.

At this stage, Greymatter Docs can merge structured data into templates. In the next chapter, we'll replace the temporary text processor with the **LibreOffice UNO Bridge**, enabling the application to open `.ott` templates, manipulate Writer documents through the UNO Document Object Model, and save real `.odt` documents programmatically.
