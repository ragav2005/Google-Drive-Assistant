# AI-Powered WhatsApp Google Drive Assistant

Turn your WhatsApp chat into a command console for Google Drive. This n8n workflow lets you LIST, DELETE, MOVE, RENAME, UPLOAD files, and even get AI SUMMARIES of documents in a folder using Google Gemini — all by sending structured WhatsApp messages.

---

## 🚀 What This Is
An event-driven automation (n8n workflow) that:
1. Listens for incoming WhatsApp messages (Cloud API)
2. Parses commands from message text (custom JavaScript)
3. Routes logic via a Switch node
4. Performs Google Drive operations (search, move, delete, upload, download)
5. Optionally runs AI document summarization with Google Gemini
6. Sends a formatted WhatsApp reply back to the user

Use it to remotely manage files hands‑free while on the go, without opening the Google Drive UI.

---

## ✨ Features
- Chat-Based File Management (LIST / DELETE / MOVE / RENAME / UPLOAD)
- AI-Powered Folder Summaries (concise bullet points for each document)
- Multi-step parsing & validation of user input
- Graceful fallback messaging when no results or bad syntax
- Modular response formatter nodes per command
- Batch processing for summaries (avoids overloading AI API)
- Designed for extensibility (e.g. SHARE, COPY, ZIP, OCR additions)

---

## 🧠 Supported Commands
Arguments are separated by `/`. Folded examples show expected syntax.

| Command | Syntax | Example | Description |
|---------|--------|---------|-------------|
| LIST | `LIST /<folder>` | `LIST /Invoices` | Lists file names in a folder |
| DELETE | `DELETE /<folder>/<file>` | `DELETE /Drafts/old-report.docx` | Soft deletes a file (not permanently) |
| MOVE | `MOVE /<source_folder>/<file>/<dest_folder>` | `MOVE /Drafts/report.pdf/Final` | Moves a file between folders |
| RENAME | `RENAME /<folder>/<old_file_name> <new_file_name>` | `RENAME /Images/pic1.jpg Family Photo.jpg` | Renames a file (space separates old & new) |
| UPLOAD | `UPLOAD /<folder>/<new_file_name>` (caption of the uploaded file) | Send PDF with caption: `UPLOAD /Receipts/oct-bill.pdf` | Uploads the attached document into the folder |
| SUMMARY | `SUMMARY /<folder>` | `SUMMARY /Meeting Notes` | Downloads each file and returns 30–40 word bullet summaries |

Notes:
- Folder lookup is performed relative to Drive root (`My Drive`).
- File matching is by exact name (as provided) inside the resolved folder.
- For RENAME the separator between old and new names is the first whitespace after the old file name token.

---

## ⚙️ Workflow Architecture

High-level flow (simplified):

```
WhatsApp Trigger
  → Parse (Code)  →  Switch (command router)
		LIST   → Resolve Folder → List Items → Format → Reply
		DELETE → Resolve Folder → Find File → Delete → Format → Reply
		MOVE   → Resolve Source Folder → Find File → Resolve Dest Folder → Move → Format → Reply
		RENAME → Resolve Folder → Find File → Update Name → Format → Reply
		UPLOAD → Resolve Folder → Download Media → HTTP Fetch Binary → Upload → Format → Reply
		SUMMARY→ Resolve Folder → List Files → (Loop) Download Each → Gemini Summarize → Aggregate → Reply
```

Key node groups (refer to `workflow.json`):
- `Code in JavaScript`: Extracts `command`, `arg1`, `arg2`, `arg3`
- `Switch`: Routes to the appropriate branch
- Drive folder/file ID resolvers (`ID of Folder L`, `ID of File M`, etc.)
- Action nodes (`LIST`, `DELETE`, `MOVE`, `RENAME`, `Upload file`)
- AI summarization (`Download file` → `Analyze document` → `Response Formatter SUMMARY`)
- Response formatters (one per command for consistent messaging)
- Final messaging (`Send message`)

---

## 🛡️ Security & Hardening
| Concern | Current State | Recommendation |
|---------|---------------|----------------|
| Hardcoded recipient number | Present in Send message node | Replace with expression: `={{ $('WhatsApp Trigger').item.json.messages[0].from }}` |
| Hardcoded Bearer token in HTTP Request (UPLOAD path) | Present | Switch to credential-based auth (Header Auth / Generic Token credential) |
| Command injection risk | Low (explicit whitelist) | Continue validating additions |
| Accidental permanent deletion | Disabled (`deletePermanently=false`) | Optionally add CONFIRM flow for destructive ops |
| Unauthorized usage | Anyone hitting the webhook could trigger | Add sender allowlist check in parser code |
| Rate limiting | None | Add a throttling Function/Code node if abuse is possible |
| Large file summarization | Potential cost/time | Add size/type filter before AI call |

Additional suggestions:
- Add logging (e.g. a simple Webhook Log or Slack node) for audit trails.
- Implement a HELP command returning usage instructions.

---

## 🧩 Parser Logic Details
The parser normalizes to uppercase and scans a fixed array:
```
ALL_COMMANDS = ['LIST', 'DELETE', 'MOVE', 'SUMMARY', 'RENAME', 'UPLOAD']
```
Extracted arguments mapping:
- LIST, DELETE, MOVE, SUMMARY: `/<arg1>/<arg2>/<arg3>` style (missing segments become `null`)
- UPLOAD: `UPLOAD /<folder>/<file_name>` (file binary from WhatsApp media message)
- RENAME: `RENAME /<folder>/<old_file_name> <new file name with spaces>`

If parsing fails, the workflow returns a null-command structure and halts downstream.

---

## 🔧 Prerequisites
Before importing and activating:
- An operational n8n instance (self-hosted or cloud)
- WhatsApp Cloud API (Meta for Developers – number + token + webhook validation)
- Google Cloud Project with:
  - Google Drive API enabled
  - Vertex AI / Gemini access (model: `models/gemini-2.5-flash` or update accordingly)
- OAuth2 credentials for Google Drive (redirect URI configured for n8n)

Optional: A dedicated service account + delegated domain-wide access (advanced setup).

---

## 🔐 Required n8n Credentials
Create these under Credentials in n8n:
1. WhatsApp OAuth (Trigger) – receives inbound messages
2. WhatsApp API (Send Message & Media) – sends replies & downloads attachments
3. Google Drive OAuth2 – used by all Drive nodes
4. Google Gemini / PaLM API – used by `Analyze document`

---

## 🛠️ Setup Steps
1. Import `workflow.json` into n8n
2. Open each credential-using node and map to your stored credentials
3. Update the WhatsApp webhook configuration (Meta dashboard) to point to n8n's public webhook URL for the Trigger node
4. Replace hardcoded phone number:
	- In `Send message` node set Recipient: `={{ $('WhatsApp Trigger').item.json.messages[0].from }}`
5. Replace hardcoded Bearer token in HTTP Request node with a credential reference
6. (Optional) Add a sender allowlist in the parser:
	```js
	const allowed = ['<your_msisdn_including_country_code>'];
	const from = $input.first().json.messages?.[0]?.from;
	if (!allowed.includes(from)) { return [{ json: { command: null }}]; }
	```
7. Activate the workflow
8. Send a test message: `LIST /SomeFolder`

---

## 🧪 Testing Tips
- Start with `LIST /NonExisting` to confirm graceful empty response
- Upload test: Send a small PDF with caption `UPLOAD /TestFolder/test.pdf`
- Rename test: `RENAME /TestFolder/test.pdf Test Renamed.pdf`
- Summary test: Place 2–3 small text/PDF docs → `SUMMARY /TestFolder`

---

## 🧵 Batching & Summaries
The `splitInBatches` node limits how many documents are processed per loop iteration to avoid long executions or token overuse. Adjust `batchSize` depending on model latency and file sizes.

Enhancement ideas:
- Cache summaries + checksum of file to skip unchanged docs
- Parallelize downloads before AI calls (with concurrency safeguards)

---

## 📦 Extensibility Ideas
| Future Command | Purpose | Implementation Sketch |
|----------------|---------|------------------------|
| SHARE | Return shareable link | Drive `permissions.create` + link reply |
| COPY | Duplicate a file | Drive `copy` operation |
| ZIP | Compress folder | Use external service + upload result |
| HELP | List commands | Static response formatter |
| SEARCH | Fuzzy find by name | Drive query with `name contains` filter |
| OCR | Extract text from images | Vision API + Gemini summarization |

---

## 🧯 Troubleshooting
| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| No WhatsApp trigger | Webhook not verified / invalid URL | Re-copy webhook URL from Trigger node & re-subscribe in Meta portal |
| Drive nodes return empty | Folder name mismatch | Confirm exact spelling / case (Drive names) |
| Upload fails | HTTP Request auth invalid | Migrate to credential and re-authorize |
| Gemini errors | Model name changed / quota exceeded | Verify model ID & quota in GCP |
| MOVE does nothing | File not found in source folder | Add debug Code node to log resolved IDs |
| SUMMARY slow | Large PDFs or many docs | Reduce batch size / add early size filter |

---

## 📁 Repository Contents
| File | Purpose |
|------|---------|
| `workflow.json` | Full n8n workflow definition |
| `README.md` | Project documentation |

---

## ⚖️ License
Choose a license (e.g. MIT). Example:
```
MIT License – see LICENSE (add this file if distributing)
```

---

## ✅ Quick Reference Cheat Sheet
```
LIST /Folder
DELETE /Folder/file.ext
MOVE /SrcFolder/file.ext/DestFolder
RENAME /Folder/old.ext New Name.ext
UPLOAD /Folder/newfile.ext   (send with file attached)
SUMMARY /Folder
```

---

## 🙌 Contributions
PRs welcome: add commands, improve parsing, integrate caching, or add automated tests with the n8n CLI.

---

## 📝 Disclaimer
This workflow manipulates live Google Drive data. Test with a sandbox Drive or dedicated subfolders before enabling in production.

---

Happy automating! If you build enhancements, consider sharing a pull request.
