# QR Attendance System

A QR-powered Digital ID and Entry Logging System built with **Google Apps Script**, **HTML/CSS/JS**, **Google Sheets**, and **Google Drive**.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  Google Apps Script                     │
│                                                         │
│  ┌─────────────┐     doGet()     ┌──────────────────┐  │
│  │  index.html │ ◄─────────────── │    Code.gs       │  │
│  │  (Scanner   │                 │                  │  │
│  │   Frontend) │  logEntry(id)   │  - logEntry()    │  │
│  │             │ ───────────────► │  - generateQR()  │  │
│  │  html5-qrcode│  {status,name, │  - setupSheets() │  │
│  │  camera scan│   action, time} │  - computeHmac() │  │
│  └─────────────┘ ◄─────────────── └────────┬─────────┘  │
│                                            │            │
└────────────────────────────────────────────┼────────────┘
                                             │
              ┌──────────────────────────────┼──────────┐
              │         Google Sheets        │          │
              │                             ▼          │
              │  ┌──────────┐    ┌──────────────────┐  │
              │  │  Users   │    │      Logs        │  │
              │  │ UserID   │    │ UserID Timestamp │  │
              │  │ Name     │    │ Name   Status    │  │
              │  │ Email    │    │ (IN / OUT)       │  │
              │  │ QR_Link  │    └──────────────────┘  │
              │  └────┬─────┘                          │
              └───────┼────────────────────────────────┘
                      │
              ┌───────▼──────────┐
              │   Google Drive   │
              │  QR_USR_001.png  │
              │  QR_USR_002.png  │
              │       ...        │
              └──────────────────┘
```

---

## Setup (Step-by-Step)

### 1. Create the Google Sheet
1. Go to [sheets.new](https://sheets.new) and create a new spreadsheet
2. Copy the **Sheet ID** from the URL:
   `https://docs.google.com/spreadsheets/d/**SHEET_ID_HERE**/edit`

### 2. Create a Google Drive Folder
1. Go to [drive.google.com](https://drive.google.com) and create a folder called `QR Codes`
2. Open it and copy the **Folder ID** from the URL:
   `https://drive.google.com/drive/folders/**FOLDER_ID_HERE**`

### 3. Open Apps Script
1. In your Google Sheet: **Extensions → Apps Script**
2. Delete any existing code
3. Create two files:
   - `Code.gs` — paste the contents of `Code.gs` from this repo
   - `index.html` — paste the contents of `index.html` from this repo

### 4. Configure `Code.gs`
At the top of `Code.gs`, set your values:
```javascript
var SHEET_ID       = 'YOUR_GOOGLE_SHEET_ID';
var DRIVE_FOLDER_ID = 'YOUR_DRIVE_FOLDER_ID';
var HMAC_SECRET    = 'pick-any-long-random-string';
```

### 5. Run Setup Functions (in order)
From the Apps Script editor toolbar, run each function once:
```
1. setupSheets()     → creates Users and Logs tabs with headers
2. addTestUsers()    → adds 5 sample users to test with
3. generateQRCodes() → generates QR PNGs, saves to Drive, writes links to sheet
```
> You will be prompted for permissions on the first run — click **Allow**.

### 6. Deploy as Web App
1. Click **Deploy → New Deployment**
2. Type: **Web App**
3. Execute as: **Me**
4. Who has access: **Anyone**
5. Click **Deploy** and copy the Web App URL

---

## How It Works

### QR Generation Logic
Each QR code encodes a **signed JSON payload**:
```json
{ "id": "USR_001", "sig": "a3f8c2..." }
```
The signature is `HMAC-SHA256(userId, HMAC_SECRET)`. This means a QR code cannot be faked — even if someone knows a user ID, they cannot produce a valid QR without the secret.

### Scanner Workflow
1. Page loads → user taps **Start Scanner**
2. `html5-qrcode` accesses the device camera (prefers back camera)
3. On decode → payload is parsed and sent to `logEntry()` via `google.script.run`
4. Feedback banner shows IN (green) / OUT (amber) / Error (pink)
5. Scanner auto-resumes after 3 seconds

### Validation Logic (`logEntry`)
```
Receive payload
    ↓
Parse JSON → extract userId + sig
    ↓
Verify HMAC signature
    ↓
Acquire LockService.getScriptLock()   ← prevents race conditions
    ↓
Lookup userId in Users sheet
    ↓
Find last log entry for this user
    ↓
Cooldown check: last scan < 5s ago? → return 'duplicate'
    ↓
Determine action: last=OUT (or none) → IN | last=IN → OUT
    ↓
AppendRow to Logs sheet
    ↓
Release lock → return result
```

### Concurrent Scan Handling
`LockService.getScriptLock()` serialises all write operations. Only one execution can hold the lock at a time — if 500 students scan simultaneously, each request waits its turn (up to 10s timeout). The cooldown check happens **inside** the lock to prevent race conditions where two requests both pass the check before either writes.

> **Honest scale note:** Apps Script handles ~30 concurrent executions before queuing. For true high-concurrency (500+ simultaneous), the architecture would move to Firestore with atomic transactions. LockService is the correct Apps Script-native answer.

---

## File Structure

```
qr-attendence/
├── index.html   # Scanner frontend (HTML Service)
├── Code.gs      # Apps Script backend
└── README.md
```

> In Apps Script, copy both files into your project. The HTML file should be named `index` (without extension) in the Apps Script editor.

---

## Evaluation Checklist

- [x] Unique QR per user with HMAC-signed payload
- [x] Real camera scanning (html5-qrcode, no simulated input)
- [x] Validates against Google Sheets database
- [x] Logs UserID, Name, Timestamp, Status (IN/OUT)
- [x] Auto IN/OUT toggle based on last scan
- [x] Duplicate/rapid scan prevention (cooldown + lock)
- [x] LockService for concurrent request safety
- [x] Invalid QR + camera error handling
- [x] QR images stored in Google Drive with share links
- [x] No hardcoded user IDs
