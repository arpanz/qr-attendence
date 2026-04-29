# QR Attendance System

A QR-powered Digital ID and Entry Logging System built with **Google Apps Script**, **HTML/CSS/JS**, **Google Sheets**, and **Google Drive**. Replaces traditional biometric attendance with secure, real-time QR scanning via a web app.

---

## Submission Links

| | Link |
|---|---|
| 📄 Apps Script Project | [View Project](https://script.google.com/home/projects) |
| 🌐 Web App (Scanner) | [Open Scanner](https://script.google.com/macros/s/AKfycbxeq8tZuXUnAWNA7vMTBvLsrf-ZbvHoKCEPyM5T7eDD9I3a2IFPV5Fj0GWGosntqR8z/exec) |
| 📊 Google Sheet | [Open Sheet](https://docs.google.com/spreadsheets/d/1Kv2Ze97Iu8DpRnmdDiaV9tfL2mimzrpIwkYNNp_qfoM/edit) |
| 📁 Drive Folder (QR Codes) | [Open Folder](https://drive.google.com/drive/folders/13nwdPs214b4bkBLh4wFj1uJh_zLLmOv-) |

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

## Live Screenshots

### Google Drive — QR Code Storage

Each user gets a uniquely named QR image stored in Drive following the `QR_{UserID}_{Name}.png` naming convention.

![QR Codes in Google Drive](screenshots/drive-qr-codes.png)

### Google Sheets — Users Database

The Users sheet stores UserID, Name, Email, and a shareable Drive link to each user's QR code — auto-populated by `generateQRCodes()`.

![Users Sheet](screenshots/sheet-users.png)

### Google Sheets — Attendance Logs

Every scan is logged with UserID, Name, ISO timestamp, and IN/OUT status. The system auto-toggles: first scan = IN, next scan = OUT.

![Attendance Logs Sheet](screenshots/sheet-logs.png)

### Sample User QR Code

This is a real generated QR for **USR_001 — Riya Sharma**. It encodes an HMAC-signed JSON payload. Scan it with the web app to test IN/OUT logging live.

![Sample QR Code - Riya Sharma](screenshots/sample-qr.png)

---

## Testing the QR Code

You can test the system right now using the sample QR above:

### Step 1 — Open the Scanner
Go to: **[https://script.google.com/macros/s/AKfycbxeq8tZuXUnAWNA7vMTBvLsrf-ZbvHoKCEPyM5T7eDD9I3a2IFPV5Fj0GWGosntqR8z/exec](https://script.google.com/macros/s/AKfycbxeq8tZuXUnAWNA7vMTBvLsrf-ZbvHoKCEPyM5T7eDD9I3a2IFPV5Fj0GWGosntqR8z/exec)**

Open this on your **phone** (or any device with a camera).

### Step 2 — Start Scanning
Tap **"Start Scanner"** — a popup will appear requesting camera access. Allow it.

### Step 3 — Point at the QR
Hold your camera up to the **Sample User QR Code** image above (on another screen or printed out). The scanner will decode it automatically.

### Step 4 — Watch the Result
- **First scan** → green banner: `✅ Riya Sharma — CHECKED IN`
- **Scan again** → amber banner: `🔄 Riya Sharma — CHECKED OUT`
- The popup closes automatically after each successful scan

### Step 5 — Verify in Google Sheets
Open the **[Logs sheet](https://docs.google.com/spreadsheets/d/1Kv2Ze97Iu8DpRnmdDiaV9tfL2mimzrpIwkYNNp_qfoM/edit)** and check that a new row appeared with:
- UserID: `USR_001`
- Name: `Riya Sharma`
- Timestamp: current UTC time
- Status: `IN` (or `OUT` on second scan)

> **Note:** The QR encodes `{"id":"USR_001","sig":"..."}` with an HMAC-SHA256 signature. Trying to scan a plain text QR with just `USR_001` will fail validation — the system will show a pink error banner.

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
var SHEET_ID        = 'YOUR_GOOGLE_SHEET_ID';
var DRIVE_FOLDER_ID = 'YOUR_DRIVE_FOLDER_ID';
var HMAC_SECRET     = 'pick-any-long-random-string';
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
The signature is `HMAC-SHA256(userId, HMAC_SECRET)`. A QR cannot be faked — even if someone knows a user ID, they cannot produce a valid QR without the secret. QR images are saved to Google Drive with public view access and the link is written back to the Users sheet automatically.

### Scanner Workflow
1. User taps **Start Scanner** on the web app
2. A camera popup opens (bypasses Apps Script iframe camera restrictions)
3. `html5-qrcode` accesses the device camera (prefers back camera)
4. On first decode → camera stops immediately, payload sent via `postMessage` to parent, popup auto-closes after 1.2s
5. Parent receives the message, calls `logEntry()` via `google.script.run`
6. Feedback banner shows **IN** (green) / **OUT** (amber) / **Error** (pink)

> The popup closes immediately after the first successful decode — this prevents the same QR from being re-read and incorrectly toggling IN→OUT in a single session.

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
Determine action: last=OUT (or none) → IN  |  last=IN → OUT
    ↓
AppendRow to Logs sheet
    ↓
Release lock → return result
```

### Error Handling

The system handles all edge cases cleanly:

| Scenario | Behaviour |
|---|---|
| Invalid / unsigned QR | HMAC verification fails → error banner |
| User ID not in Users sheet | Lookup fails → "not found" error |
| Duplicate rapid scan (< 5s) | Blocked inside lock → 'duplicate' status |
| Camera access denied | Error shown inside scanner popup |
| Popup blocked by browser | Instructions shown on main page |
| Server / script failure | `withFailureHandler` catches and displays message |

---

## Concept Question

> **"If 500 students try scanning at the same time, how would you ensure your system handles concurrent requests without data conflicts or delays?"**

`LockService.getScriptLock()` serialises all Sheets write operations — only one execution holds the lock at a time. The cooldown check runs **inside** the lock (not before it), so two near-simultaneous scans cannot both pass the check before either writes. This eliminates the race condition entirely.

Apps Script handles approximately 30 concurrent executions before queuing additional requests, with a 10-second lock wait timeout. For true high-concurrency at 500+ simultaneous scans, the correct architecture would move to **Firestore with atomic transactions**, which handles concurrent writes natively without a script-level mutex and scales horizontally without execution quotas.

---

## File Structure

```
qr-attendence/
├── index.html        # Scanner frontend (HTML Service)
├── Code.gs           # Apps Script backend
├── screenshots/      # Submission evidence
└── README.md
```

> In Apps Script, copy both files into your project. The HTML file should be named `index` (without extension) in the Apps Script editor.

---

## Evaluation Checklist

- [x] Unique QR per user with HMAC-signed payload
- [x] Real camera scanning (html5-qrcode, no simulated input)
- [x] Validates scanned QR against Google Sheets database
- [x] Logs UserID, Name, Timestamp, Status (IN/OUT)
- [x] Auto IN/OUT toggle based on last scan state
- [x] Duplicate/rapid scan prevention (cooldown inside LockService)
- [x] LockService for concurrent request safety
- [x] Invalid QR + missing user + camera error handling
- [x] QR images stored in Google Drive with consistent naming (`QR_{ID}_{Name}.png`)
- [x] Drive links auto-written to Users sheet
- [x] No hardcoded user IDs
- [x] Clean, chronological log structure
