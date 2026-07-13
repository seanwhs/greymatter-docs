# Appendix A — Greymatter Docs Architecture

## Understanding the Complete System Design

> **Objective:** This appendix provides a comprehensive overview of the Greymatter Docs architecture, explaining how each component interacts to transform data into professional business documents. It serves as a reference for developers who want to understand, maintain, or extend the platform.

---

# System Overview

Greymatter Docs follows a **layered architecture** that separates concerns into distinct, reusable components. Each layer has a single responsibility, making the application easier to understand, test, and maintain.

```text
                    User / Scheduler / API
                             │
                             ▼
                      Presentation Layer
                     (CLI / Gradio / REST)
                             │
                             ▼
                    Orchestration Layer
                  (Document Orchestrator)
                             │
      ┌──────────────────────┼──────────────────────┐
      ▼                      ▼                      ▼
 Repository Layer      Processing Layer      Service Layer
      │                      │                      │
      ▼                      ▼                      ▼
 SQLite Database     Placeholder Engine     LibreOffice
                     Table Engine           PDF Export
                     Template Engine        SMTP
      └──────────────────────┼──────────────────────┘
                             ▼
                     Generated Documents
```

---

# Architectural Principles

The architecture is based on several well-established software engineering principles.

| Principle                             | Purpose                                                                                         |
| ------------------------------------- | ----------------------------------------------------------------------------------------------- |
| Single Responsibility Principle (SRP) | Each class performs one task well.                                                              |
| Separation of Concerns                | Business logic, data access, and document processing remain independent.                        |
| Dependency Injection                  | Components receive dependencies rather than creating them internally (recommended enhancement). |
| Layered Architecture                  | Responsibilities are organized into logical layers.                                             |
| Repository Pattern                    | Encapsulates all database operations.                                                           |
| Service Pattern                       | Encapsulates external integrations such as LibreOffice and SMTP.                                |

---

# Project Structure

```text
greymatter_docs/
│
├── app.py                      # Gradio web application
├── main.py                     # CLI entry point
├── requirements.txt
├── packages.txt
├── .env
│
├── config/
│   ├── logger.py
│   ├── settings.py
│   └── validation.py
│
├── database/
│   ├── database.py
│   ├── schema.py
│   ├── seed.py
│   ├── customer_repository.py
│   ├── invoice_repository.py
│   └── delivery_repository.py
│
├── exceptions/
│
├── monitoring/
│
├── orchestrator/
│
├── processor/
│
├── services/
│
├── templates/
│
├── output/
│
├── logs/
│
└── backups/
```

---

# Layer 1 — Presentation Layer

## Responsibility

The Presentation Layer provides the entry point into the application.

Possible interfaces include:

* Command-line interface (CLI)
* Gradio web interface
* REST API
* Scheduled jobs (Cron or Task Scheduler)

### Responsibilities

* Receive user input
* Validate basic input
* Start document generation
* Return results

Example:

```text
User
 │
 ▼
Generate Invoice
 │
 ▼
Document Orchestrator
```

---

# Layer 2 — Orchestration Layer

The orchestrator coordinates the workflow without implementing business logic.

Responsibilities include:

* Loading data
* Opening templates
* Calling processors
* Saving documents
* Exporting PDFs
* Sending emails

Example workflow:

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
Export PDF
      │
      ▼
Email Customer
```

The orchestrator never performs SQL queries or directly manipulates LibreOffice objects.

---

# Layer 3 — Repository Layer

Repositories isolate all database operations.

```text
SQLite
   │
   ▼
DatabaseManager
   │
   ▼
Repositories
```

### CustomerRepository

```text
get_customer()

add_customer()

update_customer()

delete_customer()
```

### InvoiceRepository

```text
get_invoice()

get_invoice_items()

create_invoice()
```

### DeliveryRepository

```text
create_job()

update_status()
```

Benefits:

* Easier testing
* SQL contained in one location
* Database can be replaced later

---

# Layer 4 — Processing Layer

The processing layer transforms templates into finished documents.

Components include:

```text
Template Loader

↓

Placeholder Processor

↓

Table Processor

↓

Document Processor
```

Responsibilities:

* Load templates
* Replace placeholders
* Populate dynamic tables
* Apply formatting
* Calculate totals

---

# Layer 5 — Service Layer

Services communicate with external systems.

Examples:

```text
LibreOffice Service

↓

PDF Service

↓

SMTP Service

↓

Health Service
```

Each service has one responsibility.

For example:

```text
PdfService

↓

Convert ODT

↓

PDF
```

---

# Database Architecture

```text
customers
      │
      │1
      │
      ├────────────∞
      │
invoice_headers
      │
      │1
      │
      ├────────────∞
      │
invoice_items

delivery_queue
```

Relationships

```text
Customer

↓

Invoice Header

↓

Invoice Items

↓

Generated PDF

↓

Delivery Queue
```

---

# Processing Pipeline

The complete processing workflow:

```text
Load Configuration

↓

Health Check

↓

Load Job

↓

Retrieve Database Records

↓

Open Template

↓

Replace Placeholders

↓

Populate Tables

↓

Save ODT

↓

Export PDF

↓

Queue Delivery

↓

Send Email

↓

Update Status

↓

Generate Report
```

---

# Exception Flow

```text
Application

↓

Repository Error

↓

DatabaseError

↓

Logger

↓

Main Pipeline

↓

User-Friendly Message
```

Every exception is:

* Logged
* Wrapped
* Reported

---

# Logging Architecture

```text
Application

↓

Logger

↓

Console

↓

Rotating Log File
```

Suggested logs

```text
application.log

error.log

audit.log

performance.log
```

---

# Configuration Flow

```text
.env

↓

settings.py

↓

Application Components
```

Configuration includes:

* Database
* SMTP
* Logging
* LibreOffice
* Output directories

---

# Deployment Options

```text
Development

↓

Windows

Linux

↓

Docker

↓

Hugging Face

↓

Cloud Server
```

Recommended progression:

| Stage       | Platform               |
| ----------- | ---------------------- |
| Development | Local Machine          |
| Testing     | Hugging Face Space     |
| Staging     | Docker                 |
| Production  | Cloud VPS / Kubernetes |

---

# Future Architecture

As the platform grows, consider evolving to a service-oriented architecture.

```text
                REST API
                    │
                    ▼
             API Gateway
                    │
      ┌─────────────┼─────────────┐
      ▼             ▼             ▼
 Document      Email Service   Report Service
 Service
      │
      ▼
SQLite / PostgreSQL
```

Benefits:

* Independent scaling
* Easier deployment
* Better fault isolation
* Team-based development

---

# Architectural Best Practices

## Keep Layers Independent

Presentation should never access SQLite directly.

Instead:

```text
Presentation

↓

Orchestrator

↓

Repository
```

---

## Avoid Business Logic in Templates

Incorrect:

```text
{{quantity * price}}
```

Correct:

Python calculates:

```python
line_total = quantity * unit_price
```

Template displays:

```text
{{line_total}}
```

---

## Prefer Composition

Instead of one large class handling every responsibility:

```text
Document Generator

↓

Placeholder Processor

↓

Table Processor

↓

Pdf Service

↓

SMTP Service
```

Each component remains focused and reusable.

---

## Log Meaningful Events

Log:

* Job started
* Document generated
* PDF exported
* Email sent
* Job completed

Avoid excessive logging that obscures important information.

---

## Validate Early

Before processing begins, verify:

* Database exists
* Template exists
* Output directory is writable
* LibreOffice is available
* SMTP settings are valid

Failing fast prevents wasted processing and produces clearer error messages.

---

# Architecture Summary

Throughout this tutorial series, Greymatter Docs evolved into a layered, modular, and production-oriented platform.

Key architectural strengths include:

* Clean separation of concerns.
* Repository and Service patterns.
* Centralized orchestration.
* Structured logging and exception handling.
* Extensible template processing.
* Support for local execution and cloud deployment.
* A clear path toward larger-scale deployments using Docker, REST APIs, or microservices.

This architecture provides a solid foundation for document automation projects ranging from simple internal tools to enterprise-grade document generation systems.
