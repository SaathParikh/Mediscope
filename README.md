# Bill Audit (hospital-bill-checker)

Upload an itemized hospital bill (a PDF or a photo), see which charges look duplicated, bundled or too high, and get questions and a letter to send to the billing office. Create a free account to save your scans and come back to them later.
It is a learning aid, not legal, medical or financial advice. A flag means "worth asking about", not "wrong".

## Run it

Requires Python 3.9+.

```bash
python -m venv .venv
source .venv/bin/activate            # Windows: .venv\Scripts\activate
pip install -r server/requirements.txt
cp .env.example .env                 # optional: add keys
python server/app.py
```

Open http://localhost:8000. The site must be opened through the server (not by double-clicking the HTML files) because the pages call `/api/...` and load `/css`, `/js` and `/assets`.

**After you change any file in `server/`, stop the server (Ctrl+C) and start it again.** It does not reload by itself. Forgetting this is the most common reason a new feature seems to do nothing.

### Accounts and saved scans

- Click **Sign in** in the header, then **Create account** (email and a password of at least 8 characters).
- While signed in, every scan saves the original file, the results, your answers and your letter selections. Open them any time from **My scans**, change them, make a new letter, or start a new scan.
- Scanning without an account still works. Nothing is saved on the server in that case.
- Everything is stored in `data/mediscope.db` (created automatically, along with `data/secret.key`). Add `data/` to your `.gitignore`, and back it up if you care about the scans.
- Passwords are stored hashed, never as plain text. Saved bill files are **not encrypted** on disk, so only run this on a computer you trust.
- You can delete a scan, or your whole account, from the My scans page.
- If you serve the site over HTTPS, set `COOKIE_SECURE=1` in `.env`.

### Camera

**Take photo** opens a live camera view in the browser. Browsers only allow this on `http://localhost` or HTTPS. On any other address the button falls back to your phone's own camera picker. Photos are read with OCR (see below).

### Photos and scanned PDFs (optional)

PDFs with selectable text work with no extras. Photos and scanned PDFs need OCR:

- Install Tesseract: macOS `brew install tesseract`, Ubuntu `sudo apt install tesseract-ocr`, Windows: the UB Mannheim installer, then set `TESSERACT_CMD` in `.env`.
- For scanned PDFs also install Poppler: macOS `brew install poppler`, Ubuntu `sudo apt install poppler-utils`.

Without these, the scan page shows a message asking for a text PDF.

### AI overview and chat (optional)

Set `ANTHROPIC_API_KEY` in `.env`. The results page then offers an AI overview of the flagged charges and a free-text box in the "did this happen?" chat. Only the flagged line items (code, description, date, amount, quantity, flag type) and what the user types are sent. The full bill is not. Without a key, the overview is written by rules and the chat uses Yes / No / Not sure buttons only.

## Structure

```
html/index.html  scan.html  results.html  dispute.html  learn.html  aboutpg.html
html/login.html  scans.html
css/style.css
javascript/main.js       shared helpers, storage, sign-in state, header and footer
javascript/scanner.js    upload, camera, progress, calls /api/scan
javascript/detector.js   duplicate, upcoding, unbundling and quantity rules; confidence; overview
javascript/results.js    flagged charges, feedback, overview, chat
javascript/dispute.js    questions, letter tones, email and report actions
assets/logo.svg
server/                  app.py (routes, accounts, saved scans), billparser.py, ocr.py, ai.py
data/                    created at first run: mediscope.db, secret.key (do not commit)
.env.example
```

## How it works

1. `/api/scan` extracts text (PDF text layer, else OCR) and `billparser.py` pulls out date, code, description, quantity and amount per line. If you are signed in, the original file is also saved to your account.
2. `detector.js` runs the rules in the browser and returns flagged items with a starting confidence.
3. Confidence adapts: each "Good catch" / "Not an issue" click is stored in `localStorage` and counted as a vote against the rule's starting score for that flag type. The patient's Yes / No answers then shift it.
4. The dispute page builds questions and a letter (polite, firm or formal) from the charges the user selected, and the profile fields they fill in.
5. "Send to hospital" and "Email me a summary" open the user's email app. "Report to CMS / HHS" copies the summary and opens the HHS-OIG fraud report page. Nothing is sent automatically.

## Privacy

Without an account: the server reads each upload while it works and does not store it. Scan results live in `sessionStorage`. Name, hospital, emails, feedback and the running total saved live in `localStorage`.

With an account: the original file, scan results, your answers, your email, a hashed password and the name, hospital and email details you type for the letter are stored in `data/mediscope.db`. Feedback and the running total saved stay in `localStorage` on the device.

## Troubleshooting

- **"The server doesn't have the account feature yet"** on the sign-in page: `server/app.py` is the old version, or the server wasn't restarted after you replaced it.
- **"Can't reach the server"**: the server isn't running, or you opened the page from a different address than the one it prints.
- **"Server error"**: the real error is printed in the terminal window running `python server/app.py`.

## Known limits

- The line parser expects one charge per line with a CPT/HCPCS code and a trailing amount. Unusual layouts may be missed.
- The rules are a small hand-written set, not the federal code-pair tables. Savings figures are rough estimates.
- The server is meant for local use. Before putting it on the internet add HTTPS (and set `COOKIE_SECURE=1`), stronger rate limiting, encryption of saved files, and request logging that excludes bill content.