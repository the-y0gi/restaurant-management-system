# 🖨️ Store PC — Print Agent Setup Guide

> **One-time setup only.** After this, the agent starts automatically every time the PC boots.  
> Staff never needs to touch the terminal again.

---

## 📋 Table of Contents

1. [Prerequisites](#prerequisites)
2. [Step 1 — Install Node.js](#step-1--install-nodejs)
3. [Step 2 — Create Print Agent Folder](#step-2--create-print-agent-folder)
4. [Step 3 — Create print-agent.js](#step-3--create-print-agentjs)
5. [Step 4 — Create .env File](#step-4--create-env-file)
6. [Step 5 — Install Dependencies](#step-5--install-dependencies)
7. [Step 6 — Install PM2](#step-6--install-pm2-global)
8. [Step 7 — Start the Agent](#step-7--start-the-agent)
9. [Step 8 — Auto-Start on Windows Boot](#step-8--auto-start-on-windows-boot)
10. [Step 9 — Verify](#step-9--verify)
11. [Daily Use](#daily-use)
12. [Switch Printer (Future)](#future-switch-printer-tvs--tsp-100)
13. [PM2 Commands Reference](#pm2-commands-reference)

---

## Prerequisites

| Requirement | Details |
|---|---|
| **OS** | Windows 10 / 11 |
| **Node.js** | v18+ (LTS recommended) |
| **PM2** | Process manager (installed in Step 6) |
| **Pusher Account** | Keys already in `.env.example` |
| **Backend** | Deployed on Vercel |

---

## Step 1 — Install Node.js

Download and install from: **https://nodejs.org/en/download** — choose the **LTS version**.

After installing, verify in PowerShell:

```powershell
node -v
npm -v
```

✅ Both should show a version number (e.g. `v20.x.x` and `10.x.x`).

---

## Step 2 — Create Print Agent Folder

Open PowerShell and run:

```powershell
mkdir C:\PrintAgent
```

> All agent files will live in `C:\PrintAgent\`.

---

## Step 3 — Create `print-agent.js`

The main script file is already present in this repo at:

```
scripts/print-agent.js
```

**Copy it to the target location:**

```powershell
Copy-Item ".\scripts\print-agent.js" "C:\PrintAgent\print-agent.js"
```

Or manually create it:

```powershell
notepad C:\PrintAgent\print-agent.js
```

Paste the contents of [`print-agent.js`](./print-agent.js), then save and close.

### How it works

```
POS (Browser)
    │
    │  triggers print
    ▼
Backend (Vercel)
    │
    │  broadcasts via Pusher
    ▼
Pusher Channel: private-branch-{BRANCH_ID}-print
    │
    │  receives event
    ▼
print-agent.js (Running on Store PC)
    │
    ├─ Downloads PDF from backend
    ├─ Saves to temp folder
    ├─ Sends to Windows printer via pdf-to-printer
    └─ Cleans up temp file after 15 seconds
```

### Key Features

| Feature | Description |
|---|---|
| **Private Pusher Channel** | Each branch gets its own `private-branch-{id}-print` channel |
| **Server-side Auth** | Uses Pusher secret directly — no HTTP auth endpoint needed |
| **Fallback Config** | Per-job `paperSize` / `printerName` override `.env` defaults |
| **Auto Cleanup** | Temp PDFs deleted 15 seconds after printing |
| **Graceful Shutdown** | Handles `SIGINT` / `SIGTERM` cleanly |

---

## Step 4 — Create `.env` File

An example config file is at `scripts/print-agent.env.example`.

```powershell
notepad C:\PrintAgent\.env
```

Paste and fill in your values:

```env
# ─── Pusher Keys (same as backend .env) ─────────────────────────────
PUSHER_APP_ID=
PUSHER_KEY=
PUSHER_SECRET=
PUSHER_CLUSTER=

# ─── Backend URL ─────────────────────────────────────────────────────
BACKEND_URL=

# ─── Branch Settings ─────────────────────────────────────────────────
# MongoDB _id of this branch (get from DB or super-admin panel)
BRANCH_ID=FILL_THIS_IN

# ─── Printer Settings ────────────────────────────────────────────────
# Leave blank = Windows default printer
# Or set exact name from: Settings > Printers & Scanners
PRINTER_NAME=

# 58mm for TVS MP-280 Lite | 80mm for TSP-100
PAPER_SIZE=58mm
```

> [!IMPORTANT]
> You **must** fill in `BRANCH_ID` with the MongoDB `_id` of this store's branch.  
> Get it from the database or the super-admin panel.

### Environment Variables Reference

| Variable | Required | Default | Description |
|---|---|---|---|
| `PUSHER_APP_ID` | ✅ Yes | — | Pusher application ID |
| `PUSHER_KEY` | ✅ Yes | — | Pusher app key |
| `PUSHER_SECRET` | ✅ Yes | — | Pusher secret (used for server-side auth) |
| `PUSHER_CLUSTER` | ❌ No | `ap2` | Pusher cluster region |
| `BACKEND_URL` | ✅ Yes | — | Full Vercel backend URL (no trailing slash) |
| `BRANCH_ID` | ✅ Yes | — | MongoDB `_id` of this branch |
| `PRINTER_NAME` | ❌ No | Windows Default | Exact printer name from Windows settings |
| `PAPER_SIZE` | ❌ No | `58mm` | `58mm` for TVS MP-280 Lite, `80mm` for TSP-100 |

---

## Step 5 — Install Dependencies

```powershell
cd C:\PrintAgent
npm init -y
npm install pusher-js pusher pdf-to-printer dotenv
```

### Dependencies Explained

| Package | Purpose |
|---|---|
| `pusher-js` | WebSocket client — listens to Pusher channels |
| `pusher` | Pusher server SDK — used for private channel auth |
| `pdf-to-printer` | Sends PDF to Windows print spooler via SumatraPDF |
| `dotenv` | Loads `.env` config into `process.env` |

---

## Step 6 — Install PM2 (Global)

PM2 is a process manager that keeps the agent alive and auto-restarts it on crash.

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

✅ Status should show **`online`**.

---

## Step 8 — Auto-Start on Windows Boot

This ensures the agent starts automatically every time the PC turns on.

### 1. Open PowerShell as Administrator

Right-click PowerShell → **Run as Administrator**

### 2. Register the Scheduled Task

```powershell
$action = New-ScheduledTaskAction -Execute "cmd.exe" -Argument '/c "pm2 resurrect"'
$trigger = New-ScheduledTaskTrigger -AtLogOn
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopOnIdleEnd
Register-ScheduledTask -TaskName "PM2 Print Agent" -Action $action -Trigger $trigger -Settings $settings -RunLevel Highest -Force
```

### 3. Save the Current PM2 Process List

```powershell
pm2 save
```

> [!NOTE]
> This saves the current list of PM2 processes. On next boot, Windows runs `pm2 resurrect` which restores them all automatically.

---

## Step 9 — Verify

```powershell
pm2 logs print-agent
```

You should see output like:

```
[PrintAgent] ✅ Connected & subscribed to: private-branch-xxx-print
[PrintAgent] 🖨️  Printer: Windows Default | Paper: 58mm
[PrintAgent] ⏳ Waiting for print jobs...
```

**Test it** — click **Print Invoice** from the POS → bill should print on the TVS printer! ✅

---

## Daily Use

```
PC turns ON
    │
    ▼
Windows Scheduled Task runs: pm2 resurrect
    │
    ▼
print-agent.js starts automatically
    │
    ▼
Connects to Pusher channel
    │
    ▼
Waits silently for print jobs
    │
    ▼
Prints whenever POS triggers it ✅
```

**Staff never needs to touch the terminal.**

---

## Future: Switch Printer (TVS → TSP-100)

When the Star TSP-100 arrives, just update the `.env`:

```powershell
notepad C:\PrintAgent\.env
```

Change these two lines:

```env
PAPER_SIZE=80mm
PRINTER_NAME=Star TSP100
```

Then restart the agent:

```powershell
pm2 restart print-agent
```

✅ **Zero code changes needed.**

---

## PM2 Commands Reference

| Command | Purpose |
|---|---|
| `pm2 list` | View all running processes and their status |
| `pm2 logs print-agent` | View live logs (Ctrl+C to exit) |
| `pm2 restart print-agent` | Restart the agent (e.g., after `.env` change) |
| `pm2 stop print-agent` | Stop the agent |
| `pm2 start print-agent` | Start the agent |
| `pm2 save` | Save process list (run after any changes) |
| `pm2 resurrect` | Restore saved process list |
| `pm2 delete print-agent` | Remove the process from PM2 entirely |

---

## Troubleshooting

### Agent not connecting

- Check `PUSHER_KEY`, `PUSHER_SECRET`, and `PUSHER_CLUSTER` in `.env`
- Ensure `BRANCH_ID` is the correct MongoDB `_id` for this branch
- Run `pm2 logs print-agent` to see the error

### PDF not printing

- Check that `PRINTER_NAME` matches exactly what's shown in **Settings → Printers & Scanners**
- Leave `PRINTER_NAME` blank to use the Windows default printer
- Ensure the printer is online and not in error state

### Agent crashes on startup

- Run `node C:\PrintAgent\print-agent.js` directly to see the raw error
- Check that all npm packages are installed: `cd C:\PrintAgent && npm install`
- Verify `.env` file exists at `C:\PrintAgent\.env`

### After PC restart, agent not running

- Verify the scheduled task exists: open **Task Scheduler** and look for `PM2 Print Agent`
- Ensure you ran `pm2 save` after starting the agent
- Try running `pm2 resurrect` manually and check logs

---

## File Structure

```
C:\PrintAgent\
├── print-agent.js    ← Main agent script
├── .env              ← Your config (never commit this!)
├── package.json      ← Node dependencies
└── node_modules\     ← Installed packages
```

---

> **Maintained by:** Cyber Nest Dev Team  
> **Backend:** https://rms-pizza-hut-backend.vercel.app  
> **Channel Pattern:** `private-branch-{BRANCH_ID}-print`
