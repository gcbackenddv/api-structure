# API Document README

Complete reference of every API endpoint: **endpoint, request body, short description, and logic**.

- Base URL (local): `http://127.0.0.1:8000`
- Swagger UI: `/docs`
- Source: [app/main.py](app/main.py), [app/api/routes.py](app/api/routes.py), [app/api/editor_routes.py](app/api/editor_routes.py), [app/api/contact_routes.py](app/api/contact_routes.py)
- Note: `openapi.json` in the repo is out of date; this document follows the source code.

## Contents

1. [Common behaviour](#1-common-behaviour)
2. [General APIs](#2-general-apis)
3. [Async job APIs](#3-async-job-apis)
4. [Converter APIs `/convert/*`](#4-converter-apis-convert)
5. [Editor APIs `/editor/*`](#5-editor-apis-editor)
6. [Contact API](#6-contact-api)
7. [Error codes](#7-error-codes)

---

## 1. Common behaviour

### Request format
- File uploads use `multipart/form-data`. The file goes in the `file` field.
- Options are **query parameters** on `/convert/*` and **form fields** on `/editor/*`.
- The `create` editor endpoints and `/api/contact` take **JSON** (`application/json`).

### Standard conversion logic (all `/convert/*` file endpoints)
1. `save_upload()` stores the file in `UPLOAD_DIR` under a UUID name. An empty file returns `400`.
2. The converter runs in a worker thread. A semaphore limits concurrency (`CONVERSION_MAX_CONCURRENT`, default `2`).
3. For PDF outputs, the app logo is stamped on the file unless `skip_branding=true`.
4. The result is read fully into memory and returned as an attachment. Input and output files are then deleted.
5. The download filename is the original name with the new extension (e.g. `report.docx` becomes `report.pdf`).
6. Any failure returns `500` (or `400` for bad input) with `{"detail": "..."}`.

### Common query parameters

| Param | Type | Default | Meaning |
|---|---|---|---|
| `async` | bool | `false` | `true` returns **HTTP 202** and a job id immediately (see [section 3](#3-async-job-apis)). Supported on most `/convert/*` endpoints. |
| `skip_branding` | bool | `false` | Skip the logo stamp on the output PDF (office/text/image to PDF endpoints). |

---

## 2. General APIs

### `GET /api`
Returns API info.
- **Body:** none
- **Response:** `{"message":"Universal File Converter API running","version":"1.0.0","frontend":"/","office_preview":"/convert/office-to-pdf"}`

### `GET /health`
Health probe for hosting platforms.
- **Body:** none
- **Response:** `{"status":"ok"}`

### `GET /`
Serves the web frontend (`frontend/index.html`) if it exists.

### `POST /convert` (legacy, not working)
Generic queue entry. **The worker is not wired up, so do not use it.** Use `/convert/*`.
- **Body (JSON):** any object, `{}`

### `GET /jobs/{job_id}` (legacy, not working)
Status for the legacy queue above. Use `/convert/jobs/{job_id}` instead.

---

## 3. Async job APIs

Use these with any converter called with `?async=true`.

**Flow:** `POST /convert/<type>?async=true` returns `202 {job_id,...}`. Then poll `GET /convert/jobs/{job_id}` until `status` is `done`. Then call `GET /convert/jobs/{job_id}/file`.

### `GET /convert/jobs/{job_id}`
Get job status.
- **Body:** none
- **Response:**
```json
{
  "job_id": "abc123",
  "status": "queued | running | done | failed | cancelled",
  "download_filename": "report.pdf",
  "error": null,
  "poll_url": "/convert/jobs/abc123",
  "download_url": "/convert/jobs/abc123/file"
}
```
`download_url` is only set when `status == "done"`. Errors: `404` job not found, `410` job expired.

### `GET /convert/jobs/{job_id}/file`
Download the finished file. The job is **consumed** (deleted) after a successful download, so it can only be downloaded once.

### `DELETE /convert/jobs/{job_id}`
Cancel a job. Errors: `404` not found, `410` expired, `409` job can no longer be cancelled.

---

## 4. Converter APIs `/convert/*`

All use `POST` with `multipart/form-data`. "Query" parameters go in the URL, e.g. `?output_format=jpg`.

### 4.1 To PDF

| Endpoint | Body (form) | Query | Output | Logic |
|---|---|---|---|---|
| `POST /convert/office-to-pdf` | `file` (.docx/.xlsx/.pptx and other viewable Office types) | `async` | `application/pdf` | For in-app preview. The extension is checked against `viewable_office_extensions()` (`400` if missing or unsupported). The file is converted via LibreOffice. |
| `POST /convert/docx-to-pdf` | `file` (.docx) | `skip_branding`, `async` | PDF | Word to PDF. |
| `POST /convert/excel-to-pdf` | `file` (.xlsx) | `skip_branding`, `async` | PDF | Excel to PDF. |
| `POST /convert/powerpoint-to-pdf` | `file` (.pptx) | `skip_branding`, `async` | PDF | PowerPoint to PDF. |
| `POST /convert/image-to-pdf` | `file` and/or `files`: **one or many images** | `skip_branding`, `async` | PDF | Each image becomes a page. Repeat the `file` field for multi-select. Low-resolution and unsupported formats are rejected. |
| `POST /convert/heic-to-pdf` | `file` / `files`: HEIC images | `skip_branding`, `async` | PDF | Same as image-to-pdf, for iPhone HEIC photos. |
| `POST /convert/html-to-pdf` | **Exactly one of:** `file` (.html/.htm), `html` (raw HTML or a bare URL), `url` (live page). Optional `assets[]` (CSS, images, fonts for relative links). | `page_size` (`a4`\|`letter`\|`legal`, default `a4`), `wait_ms` (0-30000, default 2500, URL mode only), `async` | PDF | If `html` looks like a URL it is treated as `url`. Providing none or more than one source returns `400`. URL mode renders the page in a headless browser and waits `wait_ms` for JavaScript. |
| `POST /convert/text-to-pdf` | **One of:** `file` (.txt/.text/.md/.log) or `text` (raw string) | `page_size`, `skip_branding`, `async` | PDF | Renders plain text into PDF pages. Providing none or both returns `400`. |

### 4.2 From PDF

| Endpoint | Body (form) | Query | Output | Logic |
|---|---|---|---|---|
| `POST /convert/pdf-to-word` | `file` (PDF) | `async` | `.docx` | Extracts layout and text into Word. |
| `POST /convert/pdf-to-excel` | `file` (PDF) | `async` | `.xlsx` | Extracts tables into Excel. |
| `POST /convert/pdf-to-powerpoint` | `file` (PDF) | `async` | `.pptx` | One slide per page. |
| `POST /convert/pdf-to-images` | `file` (PDF) | `output_format` (`png`\|`jpg`\|`jpeg`, default `png`), `async` | **ZIP** (`*-pages.zip`) | Renders each page to an image and zips them. `400` if no pages were produced. |
| `POST /convert/pdf-to-heic` | `file` (PDF) | `quality` (0-100, default 90), `async` | **ZIP** (`*-heic.zip`) | One HEIC per page, zipped. |
| `POST /convert/pdf-to-text` | `file` (PDF) | `lang` (default `auto`) | **JSON** | Text extraction with OCR fallback. `auto` runs batched multilingual OCR. |
| `POST /convert/document-to-images` | `file` (PDF/Office document) | `output_format` (`png`\|`jpg`\|`jpeg`), `async` | ZIP, or streamed `multipart/mixed` (sync mode) | Document pages converted to images. |

`pdf-to-text` response:
```json
{ "success": true, "lang_requested": "auto", "lang_used": ["eng"], "ocr_languages": ["eng"], "text": "..." }
```

### 4.3 PDF tools

| Endpoint | Body (form) | Query | Output | Logic |
|---|---|---|---|---|
| `POST /convert/pdf-add-page-numbers` | `file` (PDF) | `page_mode` (`single`\|`facing`\|`facing_cover`), `position` (`bottom-center` default, `bottom-left`, `bottom-right`, `top-*`), `margin` (`small`\|`medium`\|`large`), `pages` (`all`, `2-`, `1-10`, `1,3,5`), `text_format` (default `{n}`; `{n}` = number, `{p}` = total), `start_number` (>=0, default 1), `font_size` (6-72, default 10), `font_style` (`regular`\|`bold`\|`italic`\|`bold_italic`), `color` (hex, default `#000000`), `async` | PDF | Stamps page numbers. `facing` puts odd pages on the right and even on the left. `facing_cover` leaves page 1 unnumbered. |
| `POST /convert/repair-pdf` | `file` (PDF) | `async` | PDF | Rebuilds a damaged PDF. |
| `POST /convert/pdf-to-pdfa` | `file` (PDF) | `level` (`1b`\|`2b`\|`3b`, default `2b`), `async` | PDF/A | Archival conversion (Ghostscript). |
| `POST /convert/pdf-ocr` | `file` (PDF) | `lang` (`auto`, or Tesseract codes like `eng`, `rus+eng`), `async` | **ZIP** (`*-ocr.zip`) | Makes a scanned PDF searchable. The ZIP contains `<name>.pdf` (searchable) and `<name>.txt` (extracted text). |
| `POST /convert/pdf-translate` | `file` (PDF) | `source_lang` (default `auto`), `target_lang` (default `bn`), `async` | PDF | Translates PDF text. Language codes: `en`, `bn`, `hi`, `zh-CN`, `ko`, `ja`, and so on. |

### 4.4 Image tools

| Endpoint | Body (form) | Query | Output | Logic |
|---|---|---|---|---|
| `POST /convert/image-to-word` | `file` / `files`: one or many images | `async` | `.docx` | OCR text goes into a Word document. |
| `POST /convert/image-to-excel` | `file` / `files` | `async` | `.xlsx` | OCR to a spreadsheet. |
| `POST /convert/image-to-powerpoint` | `file` / `files` | `async` | `.pptx` | One slide per image. |
| `POST /convert/image-to-text` | `file` (image) | `lang` (default `auto` = all installed Tesseract languages; or e.g. `rus+eng`, `ben`, `chi_sim`) | **JSON** | OCR. Same JSON shape as `pdf-to-text`. |
| `POST /convert/remove-background` | `file` (image) | `model` (`u2netp` default, `u2net`, `u2net_human_seg`, `u2net_cloth_seg`), `async` | PNG | AI background removal (rembg). `u2netp` is the fastest model. |

---

## 5. Editor APIs `/editor/*`

Editors work on **temporary** files. Each call uploads the file, edits it, and returns the edited file. Nothing is stored (except PDF sessions). Result filenames are `edited.docx`, `edited.xlsx`, `edited.pptx` and `edited.pdf`.

### 5.1 Word (.docx)

| Endpoint | Body | Response | Logic |
|---|---|---|---|
| `POST /editor/docx/create` | JSON `{"paragraphs": ["text1","text2"]}` | `.docx` file | Builds a new document, one paragraph per item. |
| `POST /editor/docx/read` | form: `file` | JSON: `paragraphs`, `content_blocks`, `table_rows`, `content_text`, `images`, `image_count`, `layout_blocks`, `page_count` | Parses the document so a client can render and edit it. |
| `POST /editor/docx/append` | form: `file`, `text` | `.docx` | Adds a new paragraph at the end. |
| `POST /editor/docx/update-paragraph` | form: `file`, `index` (int), `text` | `.docx` | Replaces paragraph number `index`. Bad index returns `400`. |
| `POST /editor/docx/replace-text` | form: `file`, `text`, optional `layout_blocks` (JSON string) | `.docx` | Replaces the whole document text. `layout_blocks` preserves formatting. |

### 5.2 Excel (.xlsx)

| Endpoint | Body | Response | Logic |
|---|---|---|---|
| `POST /editor/xlsx/create` | JSON `{"sheet_name":"Sheet1","rows":[["A","B"],[1,2]]}` | `.xlsx` | New workbook from a 2D array. |
| `POST /editor/xlsx/read` | form: `file`, optional `sheet_name` | JSON sheet data | Reads one sheet (the first by default). |
| `POST /editor/xlsx/update-cell` | form: `file`, `cell` (e.g. `B2`), `value`, optional `sheet_name` | `.xlsx` | Sets one cell. |
| `POST /editor/xlsx/append-row` | form: `file`, `row` (**JSON array string**, e.g. `["a",1]`), optional `sheet_name` | `.xlsx` | Appends a row. `400` if `row` is not a JSON array. |
| `POST /editor/xlsx/replace-grid` | form: `file`, `workbook` (**JSON object string**) | `.xlsx` | Replaces sheet contents with the supplied workbook grid. `400` if invalid. |

### 5.3 PowerPoint (.pptx)

| Endpoint | Body | Response | Logic |
|---|---|---|---|
| `POST /editor/pptx/create` | JSON `{"slides":[{"title":"T","body":"B"}]}` | `.pptx` | New presentation. |
| `POST /editor/pptx/read` | form: `file` | `{"slides":[...]}` | Reads every slide's content. |
| `POST /editor/pptx/update-slide` | form: `file`, `slide_index` (int), `title`, `body`, `table_rows_json` | `.pptx` | Updates one slide. Bad index returns `400`. |
| `POST /editor/pptx/update-all` | form: `file`, `slides_json` (**JSON array string**) | `.pptx` | Bulk update. Each item has `slide_index`, `title`, `body`, `title_inline_styles`, `body_inline_styles`, `title_is_bold/italic/underline`, `title_alignment`, `title_font_size`, `title_color_hex`, the same `body_*` fields, and `table_rows`. |

### 5.4 PDF editing

| Endpoint | Body (form) | Response | Logic |
|---|---|---|---|
| `POST /editor/pdf/extract-text` | `file`, `page_index` (default 0), `doc_index` (default 0) | JSON `{"success":true, ...blocks with normalized positions}` | Detects editable text blocks on a page (text, font, size, colour, position). |
| `POST /editor/pdf/apply-text-edits` | `file`, `edits_json` (JSON array; items hold `page_index`, `original_text`, `text`, `left/top/width/height`, `font_family`, `font_size_pt`, `color`, `deleted`) | PDF | In-place edit: the old text is redacted and the new text is written over it. |
| `POST /editor/pdf/assemble` | `documents[]` (one or more PDFs), `export_json` (JSON object with a `pages` array, optional `export_quality`) | PDF | Vector-preserving export. Rebuilds pages from the uploaded PDFs, handling page order, rotation, crop, text annotations and signatures without rasterizing. |
| `POST /editor/pdf/page-numbers` | `file`, `page_mode`, `position`, `margin`, `pages`, `text_format`, `start_number`, `font_size`, `font_style`, `color` (same meaning as the converter) | PDF | Adds page numbers. |
| `POST /editor/pdf/watermark` | `file`, `mode` (`text` default or `image`), `text`, `image` (file, for image mode), `image_size` (default 1.0), `position` (default `center`), `margin`, `pages` (default `all`), `font_size` (40), `font_family` (`arial`\|`impact`\|`comic_sans`\|`times_new_roman`), `font_style` (`bold`), `color` (`#808080`), `rotation` (0), `opacity` (0.15), `layer` (`over`\|`below`) | PDF | Stamps a text or image watermark. |
| `POST /editor/pdf/compress` | `file`, `compression_level` (`low`\|`medium` default\|`high`) | PDF | `low` cleans structure only. `medium` downsamples images (250 dpi threshold, JPEG 50). `high` downsamples harder (220 dpi, JPEG 38). |
| `POST /editor/pdf/redact` | `file`, `redactions_json` (JSON array, must not be empty; each item has `page_index` plus the region rectangle) | PDF (`redacted.pdf`) | Permanently removes content under the selected regions and covers them with black boxes. |
| `POST /editor/pdf/detect-forms` | `file` | JSON `{"success":true, ...fields}` | Lists the interactive AcroForm fields. |
| `POST /editor/pdf/fill-forms` | `file`, `fields_json` (JSON array of `{"id":"fieldId","value":"..."}`) | PDF (`filled.pdf`) | Fills the form fields. |

### 5.5 PDF sessions (hand-off to the Flutter/web editor)

A session stores a PDF temporarily and returns a token. The token lifetime is `EDITOR_SESSION_TTL_SEC` (default 3600 seconds).

| Endpoint | Body | Response | Logic |
|---|---|---|---|
| `POST /editor/sessions` | form: `file` (PDF), optional `name` | `{token, expires_in, file_name}` | Stores the PDF. Non-PDF files return `400`. The file header must be `%PDF`. |
| `GET /editor/sessions/{token}` | none | the PDF file | The editor page fetches the PDF with the token. Errors: `404` unknown, `410` expired. |
| `POST /editor/open/{tool}` | form: `file` (PDF), optional `name`, optional `editor_redirect_url` | `{token, expires_in, file_name, tool, redirect_url}` | For WordPress. It stores the PDF and returns `redirect_url` to send the user to the editor with the right tool open. |

`{tool}` values: `edit, crop, rotate, sign, forms, protect, unlock, watermark, page-numbers, compress, merge, split, organize, add-pages, delete-pages, ocr, translate, redact, pdfa`. Unknown tools return `400`.

The redirect base URL is taken from the form field `editor_redirect_url`, then from the env vars `EDITOR_APP_URL`, `FLUTTER_EDITOR_URL`, `FRONTEND_URL`. The result is `{base}/?token=...&tool=...&name=...`. If none is set, the API returns `500`.

---

## 6. Contact API

### `POST /api/contact`
Sends a contact-form email to the site owner.
- **Body (JSON):**
```json
{ "name": "John", "email": "john@example.com", "message": "Hello" }
```
- **Validation:** `name` 1-200 chars, `email` 3-254 chars, `message` 1-5000 chars. Values are trimmed, and the email must match `x@y.z`.
- **Response:** `{"ok": true, "message": "Your message was sent successfully."}`
- **Errors:** `422` invalid input or email, `503` email service failure.

---

## 7. Error codes

| Code | Meaning |
|---|---|
| 400 | Bad input: empty file, wrong type, invalid JSON field, missing or conflicting sources |
| 404 | Job or session not found |
| 409 | Job cannot be cancelled |
| 410 | Job or session expired |
| 422 | Validation error (missing or invalid field) |
| 500 | Conversion failed or server misconfigured |
| 503 | Email service unavailable |

All errors return `{"detail": "message"}`.

## Quick examples

```bash
# Convert Word to PDF
curl -X POST "http://127.0.0.1:8000/convert/docx-to-pdf" -F "file=@a.docx" -o a.pdf

# Async job
curl -X POST "http://127.0.0.1:8000/convert/pdf-to-word?async=true" -F "file=@a.pdf"
curl "http://127.0.0.1:8000/convert/jobs/<job_id>"
curl "http://127.0.0.1:8000/convert/jobs/<job_id>/file" -o a.docx

# Several images to one PDF
curl -X POST "http://127.0.0.1:8000/convert/image-to-pdf" -F "file=@1.png" -F "file=@2.png" -o out.pdf

# Compress a PDF
curl -X POST "http://127.0.0.1:8000/editor/pdf/compress" -F "file=@a.pdf" -F "compression_level=high" -o small.pdf

# Contact form
curl -X POST "http://127.0.0.1:8000/api/contact" -H "Content-Type: application/json" \
  -d '{"name":"John","email":"john@example.com","message":"Hi"}'
```
