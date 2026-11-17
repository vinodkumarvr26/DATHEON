# JSON Extractor Tool

A Python web application and command-line tool that extracts structured
information from plain-text invoices and resumes. The FastAPI web interface
returns validated JSON for browser download; the separate CLI can save JSON
locally. It uses the Google Gen AI SDK with Gemini 3.5 Flash-Lite and validates
model responses with Pydantic v2.

> **Input limitation:** The application accepts plain text, including `.txt`
> uploads. It does not parse PDFs or Word documents or perform OCR.

## Problem statement

Invoices and resumes contain useful information in different layouts and
wording. Manually copying that information into a consistent structure takes
time and can lead to omissions or formatting mistakes. This project explores
using an LLM to organize document text into predictable JSON while validating
the result before saving it.

## Project objectives

- Extract common invoice details, line items, amounts, and payment terms.
- Extract resume contact details, education, work history, responsibilities,
  and skills.
- Preserve source meaning, represent missing information clearly, and avoid
  inventing details.
- Validate the result against typed Pydantic models before returning or saving
  output.
- Keep prompts, schemas, API configuration, extraction, and command-line
  execution in separate modules for clarity.

## Main features

- Invoice and resume prompt builders with parameterized document text.
- Few-shot instructional examples that demonstrate expected extraction behavior.
- JSON Schema structured output sent through `response_json_schema`.
- Pydantic validation of the returned JSON, including nested data and rejection
  of unexpected fields.
- A configured 60-second Gemini request timeout and a single attempt (no
  automatic retries).
- A same-origin FastAPI web interface for invoice and resume extraction.
- Browser-side JSON downloads. The web application does not save submitted
  documents or extraction results on its server.
- Command-line selection of invoice or resume, with optional custom input and
  output paths.
- Offline pytest coverage using mocked Gemini responses; tests do not call the
  Gemini API.

## Technology stack

- Python 3.14.7
- Google Gen AI Python SDK (`google-genai`)
- Gemini 3.5 Flash-Lite
- Pydantic v2
- `python-dotenv`
- `pytest`

## Architecture and extraction workflow

1. `run_sample_extraction.py` reads a UTF-8 plain-text document.
2. The selected extraction function in `extraction.py` rejects empty input,
   obtains the configured Gemini client and model, and builds the relevant
   prompt.
3. The extraction function generates a JSON Schema from `InvoiceSchema` or
   `ResumeSchema`. It recursively removes unsupported `default` metadata while
   retaining schema structure such as `$defs`, `$ref`, and
   `additionalProperties`.
4. The prompt and cleaned schema are sent to Gemini using JSON response mode
   (`response_mime_type="application/json"` and `response_json_schema`).
5. Pydantic parses and validates the returned JSON. The CLI writes only the
   validated model to the selected output file.

### Why Pydantic validation matters

Gemini output is treated as data that still needs validation. The invoice and
resume models declare the expected field names and types, including nested
line-item, education, and work-experience models. Pydantic checks the JSON
against those declarations, parses invoice dates, and rejects unexpected
fields. Invalid output is not saved as a successful extraction.

### Parameterized and few-shot prompts

The prompt functions accept document text as a parameter and insert it in a
clearly labeled untrusted-document section. This lets the same instructions be
reused for different files. The prompt warns the model not to follow
instructions found inside the document.

Few-shot prompting provides short fictional examples alongside the extraction
instructions. These examples demonstrate how to handle dates, numbers,
currency symbols, missing values, multiple items, and separate responsibilities.
The response schema is supplied separately through the SDK rather than copied
as a complete JSON template in the prompt.

## Project structure

```text
JSON-EXTRACTOR-TOOL/
├── .env.example
├── .gitignore
├── .python-version
├── README.md
├── app.py
├── extraction.py
├── gemini_client.py
├── prompts.py
├── requirements.txt
├── render.yaml
├── run_sample_extraction.py
├── schemas.py
├── samples/
│   ├── sample_invoice.txt
│   └── sample_resume.txt
├── static/
│   ├── fonts/
│   └── index.html
└── tests/
    ├── test_extraction.py
    ├── test_schemas.py
    └── test_web_app.py
```

The `.venv/` folder is created locally during setup and is excluded from version
control. Any other local environment folders, such as `.venv-1/`, are also
ignored.

## Windows setup

Run these commands in PowerShell from the project root.

### 1. Create and activate a virtual environment

```powershell
py -3.14 -m venv .venv
.\.venv\Scripts\Activate.ps1
```

If PowerShell prevents activation due to its execution policy, you can invoke
the environment's Python directly with `.\.venv\Scripts\python.exe` instead.

### 2. Install dependencies

```powershell
python -m pip install -r requirements.txt
```

The project has been developed using Python 3.14.7. The dependency constraints
are recorded in `requirements.txt`.

### 3. Configure Gemini credentials

Create a local `.env` file from the safe template:

```powershell
Copy-Item .env.example .env
```

Open `.env` in your editor and set your own Gemini API key and model name:

```text
GEMINI_API_KEY=your_private_key
GEMINI_MODEL=gemini-3.5-flash-lite
```

Replace the key placeholder locally; never paste a real key into source code,
documentation, screenshots, issue reports, or commits. The `.env` file is
excluded by `.gitignore`; keep `.env.example` free of real credentials.

## Deploy the web application to Render

The repository includes a Render Blueprint in `render.yaml` and pins the
runtime to Python 3.14.7 in `.python-version`. Render reads
`GEMINI_API_KEY` and `GEMINI_MODEL` from the service environment; a committed
`.env` file is not required.

### 1. Publish the project to GitHub

Create an empty GitHub repository (do not initialize a second README), then
publish the reviewed project from PowerShell in the project directory. Before
staging, confirm `.env`, virtual environments, caches, and generated
`outputs/*.json` files are ignored. Explicitly stage only project source, tests,
samples, documentation, and deployment configuration:

```powershell
git add .env.example .gitignore .python-version README.md app.py extraction.py gemini_client.py prompts.py requirements.txt render.yaml run_sample_extraction.py schemas.py samples static tests
git status --short
git diff --cached --check
git diff --cached --name-only
```

Inspect the staged files, confirm that `.env`, output JSON files, virtual
environments, and caches are absent, then commit and push to the branch you
intend to deploy:

```powershell
git commit -m "Prepare JSON Extractor for Render"
git remote add origin https://github.com/<your-account>/<your-repository>.git
git push -u origin <your-branch>
```

If an `origin` remote already exists, verify it with `git remote -v` rather
than adding it again. Keep the GitHub repository's visibility appropriate for
the project and do not publish secrets.

### 2. Create the Render service

1. Sign in to Render and connect your GitHub account.
2. In the Render Dashboard, choose **New +** > **Blueprint** and select the
   GitHub repository containing `render.yaml`.
3. Review the `json-extractor` service and confirm the **Free** plan, then apply
   the Blueprint. The Blueprint configures:

   - Runtime: `python` (Python 3.14.7 from `.python-version`)
   - Build command: `pip install -r requirements.txt`
   - Start command: `uvicorn app:app --host 0.0.0.0 --port $PORT`
   - Health-check path: `/api/health`

### 3. Configure Gemini environment variables

In the service's **Environment** settings on Render, add these values
manually:

- `GEMINI_API_KEY`: your private Gemini API key
- `GEMINI_MODEL`: the Gemini model name to use, such as
  `gemini-3.5-flash-lite`

Save the settings and allow Render to redeploy. Do not put either value in
`render.yaml`, GitHub, or this README. Do not configure a `.env` file on the
deployed service.

### 4. Verify the deployment

Open the deployed service URL. The application page should load at `/`, and
`/api/health` should return a JSON status of `ok`. The health check confirms
the web process is responding; it does not make a Gemini request or verify the
Gemini credentials. After setting the environment values, a user can test an
extraction with non-sensitive sample text. The application and frontend use
the same origin.

### Free-tier and privacy limitations

- Free services may spin down after inactivity, so the first request after a
  quiet period can have a cold-start delay. Free-tier limits and behavior can
  change; check Render's current plan terms.
- The service uses ephemeral storage and has no persistent disk. The web
  application does not write uploaded documents or Gemini responses to files
  or a database. JSON downloads are created in the browser. The separate local
  CLI may write output files on the machine where it is run.
- Submitted text is sent to Gemini for processing. The application's
  no-server-storage behavior does not control data handling by Render, Google,
  or a user's browser; review their applicable terms before sending sensitive
  documents. The existing privacy notice in the UI is retained.
- The extraction limit is a conservative process-local maximum of 5 requests
  per minute for the entire service process because the application does not
  verify client IPs behind Render's proxy. It deliberately does not trust
  `X-Forwarded-For`. This in-memory limit resets when the process restarts,
  may be shared by all users of that process, and is not distributed
  protection; multiple service processes do not share counters.
- Extraction request bodies are limited to 350,000 bytes, which accommodates a
  valid 50,000-character document with JSON escaping overhead.

## Run extraction

Commands below make a live Gemini API request. Confirm your configuration and
review the input before running them.

### Extract the sample invoice

```powershell
.\.venv\Scripts\python.exe run_sample_extraction.py --type invoice
```

By default, this reads `samples/sample_invoice.txt` and writes
`outputs/invoice_output.json`.

### Extract the sample resume

```powershell
.\.venv\Scripts\python.exe run_sample_extraction.py --type resume
```

By default, this reads `samples/sample_resume.txt` and writes
`outputs/resume_output.json`.

### Use custom input and output paths

```powershell
.\.venv\Scripts\python.exe run_sample_extraction.py --type invoice --input .\samples\my_invoice.txt --output .\outputs\my_invoice.json
```

```powershell
.\.venv\Scripts\python.exe run_sample_extraction.py --type resume --input .\samples\my_resume.txt --output .\outputs\my_resume.json
```

The runner also accepts `--document-type` as an alias for `--type`. Use
`--help` to display available options:

```powershell
.\.venv\Scripts\python.exe run_sample_extraction.py --help
```

## Run automated tests

The tests use mocked responses for extraction behavior and do not call Gemini:

```powershell
.\.venv\Scripts\python.exe -m pytest -v
```

Tests cover schema shape and cleanup, prompt validation, mocked successful
invoice and resume extraction, empty and invalid responses, and structured
request configuration.

## Sample JSON structures

These are abbreviated examples of the output shape. Actual fields may be `null`
or empty collections when the document does not provide corresponding data.

### Invoice

```json
{
  "invoice_number": "INV-2026-0142",
  "invoice_date": "2026-10-12",
  "due_date": "2026-11-11",
  "supplier_name": "Northstar Paper Goods",
  "supplier_address": "48 Cedar Lane\nMapleford, OR 97000",
  "supplier_email": "billing@example.invalid",
  "bill_to_name": "Juniper Studio",
  "bill_to_address": "215 Lantern Street\nMapleford, OR 97000",
  "line_items": [
    {
      "description": "Recycled notebooks",
      "quantity": 12.0,
      "unit_price": 8.5,
      "amount": 102.0
    }
  ],
  "subtotal": 114.0,
  "tax_amount": 9.12,
  "total_due": 123.12,
  "currency_symbol": "$",
  "payment_terms": "Net 30"
}
```

### Resume

```json
{
  "full_name": "Maya Example",
  "email": "maya.example@example.invalid",
  "phone": "(555) 010-2048",
  "location": "Mapleford, OR",
  "summary": "Organized junior data analyst with experience preparing reports.",
  "education": [
    {
      "degree": "Bachelor of Science in Information Systems",
      "institution": "Mapleford State College",
      "graduation_date": "May 2025"
    }
  ],
  "experience": [
    {
      "job_title": "Data Assistant",
      "company": "Pinecone Research Co.",
      "start_date": "June 2025",
      "end_date": "August 2026",
      "responsibilities": ["Prepared weekly spreadsheet summaries."]
    }
  ],
  "skills": ["Excel", "Python fundamentals", "data quality"]
}
```

## Error handling and troubleshooting

- **Missing model or API key:** Check the corresponding setting in `.env`.
  Error messages do not print the API key.
- **Gemini API error:** The CLI reports the API error type, HTTP status, API
  status, and safe message. Check network access, model availability, quota, and
  account configuration.
- **Empty response:** The extraction layer reports that no text was returned.
- **Validation error:** The CLI reports that the model output did not match the
  Pydantic schema; the invalid payload is not written as a successful result.
- **Missing input file:** The CLI displays the file path it could not find.
- **Other errors:** The CLI reports the exception type and general
  troubleshooting guidance without dumping exception details.
- **Slow or unavailable service:** Requests use the configured 60-second timeout
  and one attempt. A later manual invocation can be made if a transient service
  error occurs; there is no retry loop.
- **PowerShell activation issue:** Use the virtual environment's full Python
  path as shown in the commands above.

## Current limitations and future improvements

The application accepts plain-text documents only. Extraction quality depends
on the source text and model response. It does not currently include PDF or
Word parsing, OCR, a database, batch processing, or human review workflows.
The web service does not persist documents or extraction results; the local
CLI can save output files when requested.

Possible future work includes adding document-format conversion, larger test
coverage for CLI error paths, schema-versioned outputs, and a user review step
before saving extracted data. These are future ideas, not implemented features.

## GitHub submission recommendations

- Commit source code, tests, sample text, README, requirements, and
  `.env.example`.
- Do not commit `.env`, virtual environments, caches, or generated output JSON.
- Review `git status` and inspect staged files before committing.
- Never include API keys in commits, logs, screenshots, or issue descriptions.
