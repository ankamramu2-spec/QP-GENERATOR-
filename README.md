# AI Question Paper Formatter — Complete Application (Phases 0–6)

Converts an uploaded question paper (PDF, scanned PDF, DOCX, TXT, JPG, PNG)
into a professionally formatted DOCX and PDF matching JMJ High School's
style, with a web dashboard to upload, process, search, and download.

## What's included and tested (built and verified in this environment)

- **Upload** — file type/size validation, safe filename handling, saved to disk + SQLite
- **Extraction** — PDF (pdfplumber), DOCX (python-docx), TXT, and OCR for
  scanned PDFs/images (Tesseract) — tested with a DOCX sample and an OCR'd image
- **AI Formatting** — rule-based parser detects school name, class, subject,
  time, marks, sections, and question types (MCQ, fill-in-blank, true/false,
  match-the-following, short/long answer) — tested end-to-end
- **DOCX/PDF Generation** — matches the JMJ style spec (Cambria fonts, 1"
  margins, 1.15 line spacing, page numbers) via python-docx + LibreOffice
  headless — tested, produces valid downloadable files
- **Dashboard** — upload form, live file list, search, stats widgets
  (total/completed/processing/failed), dark mode, structure preview modal,
  auto-refresh — tested via the API endpoints it calls

## What's included but NOT tested here (needs your own credentials/hosting)

- **Gmail integration** (`app/services/gmail_integration.py`) — fully
  written, but requires your own Google Cloud project and OAuth credentials
  to run. I can't complete Google's login flow on your behalf. Setup steps
  are in that file's docstring.
- **Live deployment to Render** — `Dockerfile` and `render.yaml` are
  provided and the Docker image logic mirrors what was tested locally, but
  I don't have access to deploy to your Render account. The SQLite database
  also has NOT yet been migrated to PostgreSQL in the code — for a single
  school's usage, SQLite is fine as long as your hosting plan gives it a
  persistent disk; if you outgrow that, the database layer (`app/database.py`)
  is small and isolated enough to swap out later.

## 1. Install

Requires Python 3.10+, and for full functionality: Tesseract OCR and
LibreOffice installed on the system (not just pip packages).

```bash
# System dependencies (Ubuntu/Debian, e.g. inside WSL or a Linux server)
sudo apt-get install tesseract-ocr libreoffice

# Python environment
python -m venv venv
source venv/bin/activate      # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

On Windows, install Tesseract from the official installer
(https://github.com/UB-Mannheim/tesseract/wiki) and LibreOffice from
libreoffice.org, then make sure both are on your PATH.

## 2. Run

```bash
uvicorn app.main:app --reload
```

Open **http://127.0.0.1:8000**.

## 3. Test it works
1. Upload a question paper (PDF/DOCX/TXT/JPG/PNG).
2. Click **Process** on its row — status changes to "processing" then "completed".
3. Click **Preview** to see the detected structure (school, class, sections, questions).
4. Click **DOCX** / **PDF** to download the formatted output.
5. Use the search box to filter by filename; check the stats widgets update.

## 4. Run with Docker (mirrors the deployment environment)
```bash
docker build -t qp-formatter .
docker run -p 8000:8000 qp-formatter
```

## 5. Deploy to Render (free tier)
1. Push this project to a GitHub repo.
2. In Render: New → Blueprint → connect the repo (it will read `render.yaml`).
   Or manually: New → Web Service → Environment: Docker.
3. First deploy will be slow (installing LibreOffice in the image). Subsequent
   deploys are faster due to layer caching.
4. Free tier sleeps after ~15 min of inactivity; first request after that
   takes 30-50 seconds to wake up.
5. The `uploads/` and `outputs/` folders are NOT persistent on Render's free
   tier (disk resets on redeploy/restart) — fine for a low-traffic tool where
   files are downloaded soon after processing; for guaranteed persistence,
   add a paid persistent disk or migrate file storage to Cloudflare R2.

## 6. Optional: Gmail automation
See the setup steps at the top of `app/services/gmail_integration.py`.
Requires: `pip install google-api-python-client google-auth-httplib2 google-auth-oauthlib`

## Project structure
```
qp_formatter/
├── app/
│   ├── main.py                  # FastAPI app, all routes wired here
│   ├── database.py               # SQLite schema + queries
│   ├── routes/
│   │   ├── upload.py             # POST /api/upload
│   │   ├── files.py              # GET /api/files, /api/stats
│   │   ├── process.py            # POST /api/process/{id}, GET /api/download/{id}/{fmt}
│   │   └── gmail.py               # POST /api/gmail/check-inbox (optional)
│   ├── services/
│   │   ├── extraction.py         # Phase 1: PDF/DOCX/TXT/OCR text extraction
│   │   ├── parsing.py            # Phase 2: rule-based question paper parser
│   │   ├── generation.py         # Phase 3: DOCX build + PDF conversion
│   │   └── gmail_integration.py  # Phase 5: optional Gmail polling
│   ├── static/{css,js}/
│   └── templates/index.html      # dashboard page
├── uploads/                      # raw uploaded files
├── outputs/                      # generated DOCX/PDF files
├── Dockerfile
├── render.yaml
└── requirements.txt
```

## Known limitations (honest notes, not hidden)
- The question-type parser is rule-based (regex/keyword matching), not a
  trained ML model — it handles common CBSE paper layouts well but unusual
  formatting may land in "unclassified_lines" for manual review, which is
  surfaced in the Preview modal rather than silently dropped.
- OCR accuracy depends on scan quality — low-resolution or skewed scans will
  extract text less reliably. This is a Tesseract limitation, not fixable
  by this app alone.
- No login/authentication — anyone with the URL can upload and download.
  Fine for a low-stakes internal tool; add auth before wider public exposure.
