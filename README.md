# Store PC — Print Agent Setup Guide...

> One-time setup. After this, the agent starts automatically every time the PC boots.

---

## Step 1 — Install Node.js

Download and install from: **https://nodejs.org/en/download** (choose LTS version)

Verify installation:

```powershell
node -v
npm -v
```

Both should show a version number. ✅

---

## Step 2 — Create Print Agent Folder

```powershell
mkdir C:\PrintAgent
cd C:\PrintAgent
```

---

## Step 3 — Create `print-agent.js`

```powershell
notepad C:\PrintAgent\print-agent.js
```

Paste the following code, then save and close:

```js
/**
 * print-agent.js — Store PC Local Print Agent
 * Setup: C:\PrintAgent\print-agent.js
 * Run with: pm2 start print-agent.js --name "print-agent"
 */



require("dotenv").config();
const Pusher = require("pusher-js");
const printer = require("pdf-to-printer");
const fs = require("fs");
const path = require("path");
const os = require("os");

// ─── Config from .env ──────────────────────────────────────────────────────
const PUSHER_KEY     = process.env.PUSHER_KEY;
const PUSHER_SECRET  = process.env.PUSHER_SECRET;
const PUSHER_APP_ID  = process.env.PUSHER_APP_ID;
const PUSHER_CLUSTER = process.env.PUSHER_CLUSTER || "ap2";
const BACKEND_URL    = process.env.BACKEND_URL;
const BRANCH_ID      = process.env.BRANCH_ID;
const PRINTER_NAME   = process.env.PRINTER_NAME || "";
const PAPER_SIZE     = process.env.PAPER_SIZE || "58mm";

// ─── Validate required config ──────────────────────────────────────────────
if (!PUSHER_KEY || !PUSHER_SECRET || !BACKEND_URL || !BRANCH_ID) {
  console.error("[PrintAgent] Missing required .env variables. Check PUSHER_KEY, PUSHER_SECRET, BACKEND_URL, BRANCH_ID.");
  process.exit(1);
}

// Ensure BACKEND_URL doesn't end with trailing slash for clean concatenation
const cleanBackendUrl = BACKEND_URL.replace(/\/+$/, "");

// ─── Temp folder for PDFs ──────────────────────────────────────────────────
const TEMP_DIR = path.join(os.tmpdir(), "print-agent-receipts");
if (!fs.existsSync(TEMP_DIR)) fs.mkdirSync(TEMP_DIR, { recursive: true });

// ─── Pusher Client ─────────────────────────────────────────────────────────
const pusher = new Pusher(PUSHER_KEY, {
  cluster: PUSHER_CLUSTER,
  forceTLS: true,
  authorizer: (channel) => ({
    authorize: (socketId, callback) => {
      const PusherServer = require("pusher");
      const serverPusher = new PusherServer({
        appId: PUSHER_APP_ID,
        key: PUSHER_KEY,
        secret: PUSHER_SECRET,
        cluster: PUSHER_CLUSTER,
        useTLS: true,
      });
      const authResponse = serverPusher.authorizeChannel(socketId, channel.name);
      callback(null, authResponse);
    },
  }),
});

// ─── Subscribe to branch print channel ────────────────────────────────────
const CHANNEL_NAME = `private-branch-${BRANCH_ID}-print`;
const channel = pusher.subscribe(CHANNEL_NAME);

channel.bind("pusher:subscription_succeeded", () => {
  console.log(`[PrintAgent] Connected & subscribed to: ${CHANNEL_NAME}`);
  console.log(`[PrintAgent] Printer: ${PRINTER_NAME || "Windows Default"} | Paper: ${PAPER_SIZE}`);
  console.log(`[PrintAgent] Waiting for print jobs...`);
});

channel.bind("pusher:subscription_error", (err) => {
  console.error("[PrintAgent] Subscription error:", err);
});

// ─── Handle print-job event ────────────────────────────────────────────────
channel.bind("print-job", async (data) => {
  const { orderId, orderNumber, paperSize, itemsFilter, printerName, pdfUrl: payloadPdfUrl } = data;

  const resolvedPaper   = paperSize   || PAPER_SIZE;
  const resolvedPrinter = printerName || PRINTER_NAME;
  const resolvedFilter  = itemsFilter || "all";

  console.log(`\n[PrintAgent] Print job received — Order: ${orderNumber} | Paper: ${resolvedPaper}`);

  const tempFileName = `receipt-${(orderNumber || orderId).replace(/[^a-zA-Z0-9]/g, "-")}-${Date.now()}.pdf`;
  const tempFilePath = path.join(TEMP_DIR, tempFileName);

  try {
    // 1. Determine PDF URL:
    //    - If payload has absolute http/https URL: use it directly
    //    - If payload has relative URL starting with /: append cleanBackendUrl + payloadPdfUrl
    //    - Fallback: construct default single-order URL from BACKEND_URL/orders/${orderId}/pdf
    let pdfUrl;
    if (payloadPdfUrl && payloadPdfUrl.startsWith("http")) {
      pdfUrl = payloadPdfUrl;
    } else if (payloadPdfUrl && payloadPdfUrl.startsWith("/")) {
      const cleanPath = payloadPdfUrl.startsWith("/api") && cleanBackendUrl.endsWith("/api")
        ? payloadPdfUrl.slice(4)
        : payloadPdfUrl;
      pdfUrl = `${cleanBackendUrl}${cleanPath}`;
    } else {
      pdfUrl = `${cleanBackendUrl}/orders/${orderId}/pdf?paperSize=${resolvedPaper}&itemsFilter=${resolvedFilter}`;
    }

    console.log(`[PrintAgent] Downloading PDF: ${pdfUrl}`);

    const response = await fetch(pdfUrl);
    if (!response.ok) throw new Error(`PDF download failed: HTTP ${response.status}`);

    // 2. Save to temp file
    const arrayBuffer = await response.arrayBuffer();
    fs.writeFileSync(tempFilePath, Buffer.from(arrayBuffer));
    console.log(`[PrintAgent] PDF saved: ${tempFilePath}`);

    // 3. Send to Windows printer via pdf-to-printer
    const printOptions = {};
    if (resolvedPrinter && resolvedPrinter.trim() !== "") {
      printOptions.printer = resolvedPrinter.trim();
    }

    await printer.print(tempFilePath, printOptions);
    console.log(`[PrintAgent] Printed successfully → ${printOptions.printer || "Default Printer"}`);

  } catch (err) {
    console.error(`[PrintAgent] Print failed for ${orderNumber}:`, err.message);
  } finally {
    // 4. Cleanup temp PDF after 15 seconds
    setTimeout(() => {
      if (fs.existsSync(tempFilePath)) {
        fs.unlinkSync(tempFilePath);
        console.log(`[PrintAgent] Temp file cleaned: ${tempFileName}`);
      }
    }, 15000);
  }
});

// ─── Graceful shutdown ─────────────────────────────────────────────────────
process.on("SIGINT",  () => { pusher.disconnect(); console.log("\n[PrintAgent] Disconnected."); process.exit(0); });
process.on("SIGTERM", () => { pusher.disconnect(); console.log("\n[PrintAgent] Disconnected."); process.exit(0); });

```

---

## Step 4 — Create `.env` File

```powershell
notepad C:\PrintAgent\.env
```

Paste and fill in your values:

```env
PUSHER_APP_ID=
PUSHER_KEY=
PUSHER_SECRET=
PUSHER_CLUSTER=

BACKEND_URL=

# MongoDB _id of this branch (get from DB or super-admin panel)
BRANCH_ID=FILL_THIS_IN

# Leave blank = Windows default printer
# Or set exact name from Settings > Printers & Scanners
PRINTER_NAME=

# 58mm for TVS MP-280 Lite | 80mm for TSP-100
PAPER_SIZE=58mm
```

> [!IMPORTANT]
> Fill in `BRANCH_ID` with the MongoDB `_id` of this store's branch.

---

## Step 5 — Install Dependencies

```powershell
cd C:\PrintAgent
npm init -y
npm install pusher-js pusher pdf-to-printer dotenv
```

---

## Step 6 — Install PM2 (Global)

```powershell
npm install pm2 -g
```

---

## Step 7 — Start the Agent

```powershell
pm2 start C:\PrintAgent\print-agent.js --name "print-agent"
```

Check it's running:

```powershell
pm2 list
```

Status should be **`online`** ✅

---

## Step 8 — Auto-Start on Windows Boot

Open **PowerShell as Administrator** (Right-click → Run as Administrator), then run:

```powershell
$action = New-ScheduledTaskAction -Execute "cmd.exe" -Argument '/c "pm2 resurrect"'
$trigger = New-ScheduledTaskTrigger -AtLogOn
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopOnIdleEnd
Register-ScheduledTask -TaskName "PM2 Print Agent" -Action $action -Trigger $trigger -Settings $settings -RunLevel Highest -Force
```

Then save the current process list:

```powershell
pm2 save
```

---

## Step 9 — Verify

```powershell
pm2 logs print-agent
```

You should see:

```
[PrintAgent] Connected & subscribed to: private-branch-xxx-print
[PrintAgent] Printer: Windows Default | Paper: 58mm
[PrintAgent] Waiting for print jobs...
```

**Test it** — click Print Invoice from POS → bill should print on TVS printer! ✅

---

## Daily Use — Nothing to Do!

PC turns ON → agent starts automatically → prints whenever POS triggers it.
Staff never needs to touch the terminal.

---

## Future: Switch Printer (TVS → TSP-100)

```powershell
notepad C:\PrintAgent\.env
```

Update these two lines:

```env
PAPER_SIZE=80mm
PRINTER_NAME=Star TSP100
```

Then:

```powershell
pm2 restart print-agent
```

**Zero code changes needed.** ✅

---

## Useful PM2 Commands

| Command | Purpose |
|---|---|
| `pm2 list` | Check agent status |
| `pm2 logs print-agent` | View live logs |
| `pm2 restart print-agent` | Restart agent |
| `pm2 stop print-agent` | Stop agent |
| `pm2 start print-agent` | Start agent |
