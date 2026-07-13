# Part 11 — Free Cloud Deployment with Hugging Face Spaces

## Deploying Greymatter Docs to the Cloud (Zero Cost)

> **Goal:** Deploy Greymatter Docs to the cloud using **Hugging Face Spaces** so users can generate documents through a web interface without installing Python or LibreOffice locally.

**Milestone:** By the end of this chapter, you'll have a publicly accessible Greymatter Docs application running on Hugging Face Spaces using Gradio.

> **Important Architecture Note**
>
> The LibreOffice UNO API requires a running LibreOffice process, which is difficult to manage reliably on the free Hugging Face Spaces environment. For cloud deployment, the recommended approach is to use **LibreOffice in headless CLI mode** (`soffice`) instead of the persistent UNO socket connection.
>
> This keeps the application lightweight, easier to deploy, and fully compatible with the free Hugging Face infrastructure.

---

# THEORY & ARCHITECTURE

Our local architecture looked like this:

```text
Browser
    │
    ▼
Python
    │
    ▼
SQLite
    │
    ▼
LibreOffice UNO
    │
    ▼
Generated Document
```

For Hugging Face we introduce a web layer.

```text
User Browser
      │
      ▼
Gradio Web UI
      │
      ▼
Greymatter Docs
      │
      ▼
SQLite
      │
      ▼
LibreOffice Headless
      │
      ▼
PDF / DOCX / ODT
```

Instead of running from the command line,

```bash
python main.py
```

users interact through a web browser.

---

# BUILD IT

## Step 1 — Create a Hugging Face Account

Create a free account at:

* [https://huggingface.co](https://huggingface.co)

Create a new **Space**.

Settings

```
SDK

Gradio
```

Hardware

```
CPU Basic (Free)
```

Visibility

```
Public
```

---

## Step 2 — Project Structure

Your project should now look like this.

```
greymatter_docs/

app.py

requirements.txt

packages.txt

README.md

config/

database/

processor/

services/

templates/

output/

greymatter.db
```

---

## Step 3 — Install LibreOffice

Hugging Face Spaces allows Debian packages through a file named

```
packages.txt
```

Create

```text
libreoffice

fonts-dejavu
```

This tells Hugging Face to install LibreOffice automatically during deployment.

---

## Step 4 — Create requirements.txt

```text
gradio

python-dotenv

pandas

sqlite-utils
```

Include any other Python packages used by your project.

---

## Step 5 — Replace UNO Socket Connections

Instead of

```python
LibreOfficeService.connect()
```

use the LibreOffice command line.

Example

```python
import subprocess

subprocess.run(
    [
        "soffice",
        "--headless",
        "--convert-to",
        "pdf",
        "invoice.odt",
        "--outdir",
        "output"
    ],
    check=True
)
```

This launches LibreOffice only when needed.

Benefits

* simpler deployment
* no socket management
* fewer background processes
* ideal for cloud hosting

---

## Step 6 — Build a Simple Web Interface

Create

```python
app.py
```

```python
import gradio as gr
from pathlib import Path

from orchestrator.document_orchestrator import (
    DocumentOrchestrator
)

OUTPUT = Path("output")


def generate_document(customer_id):

    orchestrator = DocumentOrchestrator()

    pdf = orchestrator.generate_customer_invoice(
        customer_id
    )

    return str(pdf)


demo = gr.Interface(
    fn=generate_document,
    inputs=gr.Number(
        label="Customer ID"
    ),
    outputs=gr.File(
        label="Generated PDF"
    ),
    title="Greymatter Docs",
    description="Automated Document Generation"
)

demo.launch()
```

Running locally

```bash
python app.py
```

opens

```
http://127.0.0.1:7860
```

---

## Step 7 — Push to GitHub

```
git init

git add .

git commit -m "Initial deployment"

git remote add origin ...

git push
```

---

## Step 8 — Connect Hugging Face

Inside the Space

```
Files

↓

Connect Repository

↓

GitHub
```

Select

```
Greymatter Docs
```

Deployment starts automatically.

---

## Step 9 — Automatic Build

During deployment Hugging Face performs

```
Install Debian packages

↓

Install Python packages

↓

Launch app.py

↓

Public URL available
```

Example

```
https://your-name-greymatter-docs.hf.space
```

---

# CODE WALKTHROUGH

## Why Gradio?

Gradio requires almost no web development knowledge.

A complete web interface can be built in only a few lines.

```python
gr.Interface(
    fn=generate_document,
    inputs=...,
    outputs=...
)
```

No HTML.

No JavaScript.

No CSS.

---

## Why Headless LibreOffice?

UNO works best when a long-running LibreOffice process is available.

Cloud platforms frequently restart containers.

Launching LibreOffice only when needed is more reliable.

```
Generate Document

↓

Start LibreOffice

↓

Convert

↓

Exit
```

---

## Stateless Design

Hugging Face containers may restart at any time.

Never assume

* uploaded files remain forever
* SQLite grows indefinitely
* generated PDFs stay on disk

Instead

Generate

↓

Download

↓

Delete temporary files

---

## Environment Variables

Inside Hugging Face

```
Settings

↓

Repository Secrets
```

Store

```
SMTP_USERNAME

SMTP_PASSWORD

API_KEYS
```

Never hardcode secrets.

---

# CHALLENGE LAB

Improve your deployment by implementing the following features.

### Challenge 1

Allow users to upload a CSV containing multiple customers.

Generate all invoices automatically.

---

### Challenge 2

Compress all generated PDFs into a ZIP archive.

Return

```
Invoices.zip
```

instead of individual files.

---

### Challenge 3

Add a progress bar.

```
Generating...

██████░░░░░ 60%
```

---

### Challenge 4

Allow users to upload their own

```
.ott
```

template.

---

### Challenge 5

Create an Admin page displaying

* Jobs processed
* PDFs generated
* Processing time
* Failed jobs

---

# TROUBLESHOOTING

| Problem                       | Cause                     | Solution                                                     |
| ----------------------------- | ------------------------- | ------------------------------------------------------------ |
| Build fails                   | Missing package           | Check `requirements.txt` and `packages.txt`.                 |
| `soffice` not found           | LibreOffice not installed | Add `libreoffice` to `packages.txt`.                         |
| SQLite is empty after restart | Ephemeral storage         | Bundle a seed database or recreate it at startup.            |
| Files disappear               | Container restarted       | Generate files on demand and let users download immediately. |
| Application times out         | Long-running jobs         | Split work into smaller batches and optimize processing.     |

---

# Production Recommendations

For a demonstration or small-scale application, **Hugging Face Spaces (Free)** is an excellent choice. As usage grows, consider migrating to a more robust deployment platform:

| Platform            | Free Tier          | Best Use                     |
| ------------------- | ------------------ | ---------------------------- |
| Hugging Face Spaces | ✅                  | Public demos and prototypes  |
| Render              | ✅                  | Small production web apps    |
| Railway             | ✅ (limited)        | Background workers and APIs  |
| Fly.io              | ✅ (limited)        | Global deployments           |
| Azure App Service   | Student/Trial      | Enterprise deployments       |
| AWS EC2 / Lightsail | Low cost           | Full control and scalability |
| Google Cloud Run    | Generous free tier | Containerized applications   |
| Docker + VPS        | Low cost           | Production self-hosting      |

---

# Chapter Summary

In this chapter, you:

* Adapted Greymatter Docs for cloud deployment.
* Switched from a persistent UNO socket connection to **headless LibreOffice** for better compatibility with free hosting.
* Built a simple **Gradio** web interface.
* Configured `requirements.txt` and `packages.txt`.
* Connected the project to **GitHub** and **Hugging Face Spaces**.
* Learned how to handle ephemeral storage, secrets, and deployment constraints.

At this point, **Greymatter Docs** is no longer just a desktop automation tool—it can be accessed through a web browser, allowing users to generate and download documents from anywhere, all on a free cloud-hosted platform.
