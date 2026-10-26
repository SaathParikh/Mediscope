# Bill Audit (hospital bill checker)

Upload an itemised hospital bill (PDF or picture). See which charges might be duplications, bundlings or overcharging. Get some questions to ask and a letter to send to the billing office. Create a free account to save your scans.
This is a learning tool. It does not provide legal, medical or financial advice. The flags don't indicate that something is wrong, just that it's worth asking about.

## How to run

Python 3.9+

```bash
python -m venv .venv
source .venv/bin/activate # Windows .venv\Scripts\activate
pip install -r server/requirements.txt
cp .env.example .env # optional: include keys
python server/app.py
```

Go to http://localhost:8000. The site needs to be run through the server, and not opened directly from the HTML files. This is because all the pages use the /api/ routes, as well as loading CSS, JS and assets.

If you make any changes to files inside the server folder, you will need to stop the server (Ctrl+C) and restart. This is the most common reason new features aren't working.

## Accounts and saving scans

Click Sign in in the header and create an account using an email address and 8+ character password.
While logged in your scans will be stored. Each scan will have the original document, results, answers and letter choices saved. You can view these at any point by going to My scans. You can edit these, create a new letter or start a new scan.
You can still scan without an account. However, nothing will be stored on the server.
All data is stored in data/mediscope.db (automatically created). You should add data/ to your .gitignore. If you want to keep your scans backed up you can do so. Passwords are stored hashed. The bill documents are not encrypted. Run this on a computer you trust.
You can delete individual scans or your entire account from My scans.
If you host this site with HTTPS set COOKIE_SECURE=1 in your .env.

## Camera

Opening the Take a photo option will start a live camera feed. This is only allowed to happen on http://localhost or HTTPS in browsers. If you try to open it on any other address it will default back to your phone's camera selector. The photos will be read using OCR (see below).
## Taking a photo / scanning a PDF (optional)

Text based PDFs will work without any additional setup. If you're scanning a photo or a scanned PDF you'll need to set up OCR.

Install Tesseract. Mac: brew install tesseract, Ubuntu: sudo apt install tesseract-ocr, Windows: UB Mannheim Installer. Then in your .env set TESSERACT_CMD.
If scanning a scanned PDF also install Poppler. Mac: brew install poppler, Ubuntu: sudo apt install poppler-utils.

Otherwise you will get a message on the scan page saying you need a text based PDF.

## AI summary + chat (optional)

Set ANTHROPIC_API_KEY in your .env. When viewing the results page you will see an AI summary of the flagged charges. There is also a free text chat box under "did this happen?". Only the flagged charges (code, description, date, amount, quantity and flag type) and any text you put in the chat box will be sent. Your entire bill will not be sent. If no API key is given the summary will be made using rules. The chat box will only have Yes/No/Not sure buttons.

## File structure
```

html/
index.html
scan.html
results.html
dispute.html
learn.html
aboutpg.html
login.html
scans.html
css/
style.css
javascript/
main.js # Shared functions, storage, logged in status, header and footer
scanner.js # Upload, camera, progress bar, sends to /api/scan
detector.js # Duplicate, upcoding, unbundling, quantity rules, confidence score
results.js # Flagged charges, feedback, summary, chat
dispute.js # Questions, letter tone, send email/report
assets/
logo.svg
server/
app.py # Routes, accounts, storing scans
billparser.py
ocr.py
ai.py
data/ # Automatically created on first run
mediscope.db
secret.key
.env.example
```

## How it works

1. The /api/scan endpoint extracts text from the document (text layer of PDF, otherwise uses OCR). This is parsed by billparser.py to extract the date, code, description, quantity and amount of each line. If you're logged in the original document will be saved to your account.
2. The detector.js runs the rules and returns the flagged charges along with an initial confidence.
3. Confidence is dynamic. Every time you click Good catch/Not an issue it will be added to localStorage. This will act as a vote against the initial confidence of the rule. The patient's answers (Yes/No) will affect their confidence.
4. The dispute page will generate questions and a letter (polite, firm or formal) based on the charges you've flagged. This will also use the profile information you've entered.
5. Send to hospital and "Email me a summary" will open your email app. Report to CMS / HHS will copy the summary and take you to the HHS-OIG fraud reporting page. Nothing will be automatically sent.

## Privacy
No account:

The server will temporarily read the uploaded file. It is not stored. The results are stored in sessionStorage. Your name, hospital, emails, feedback and running total are stored in localStorage.
Account:

The original document, scan results, answers, email address, hashed password and the details you type into the profile (name, hospital, email) are stored in data/mediscope.db. Feedback and running total are stored in localStorage.

## Troubleshooting

Sign in page says "The server does not currently support accounts": The server/app.py is old or you haven't restarted the server.
Can't connect to server: Server isn't running or you've opened the page on a different address.
Server error: Real error is shown in the terminal window where you ran python server/app.py.
## Limitations

Line parser assumes there is one charge per line, containing a CPT/HCPCS code and ending with an amount. May miss unusual formats.
Rules are a small number of manually created rules. Not using federal code pair tables. Savings are rough estimates.
The server is intended to be run locally. If you plan to host it online you should add HTTPS (set COOKIE_SECURE=1), more rate limiting, encrypted files and log requests but not bill information.
