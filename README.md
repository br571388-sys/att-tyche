---
title: Tyche Attendance Log
emoji: 📋
colorFrom: blue ok
colorTo: indigo
sdk: docker
app_port: 7860
pinned: false
---

# Tyche Attendance Log (MongoDB Atlas backend)

Node.js + Express app — Employee ID + 4-digit password login, daily attendance form (with holiday option), personal + team dashboard, admin edit/delete, and Word/PDF export. Data is stored in **MongoDB Atlas** (permanent — survives HF Space restarts). Optional push notifications remind employees to fill their attendance.

## Features
- **Login**: Employee ID + 4-digit PIN.
  - New Employee ID → asks Name + a 4-digit password (registers).
  - Existing Employee ID created before this update → asked to set a 4-digit password once, then logs straight in after that.
  - One attendance entry per employee per date (work day OR holiday, not both) — trying to submit a second entry for the same date is blocked with a clear error.
- **Form**: Date, Holiday toggle (with a reason field when Yes), Shift A/B/C, OT Yes/No + hours, Work, Machine Name/Number (optional), Supervisor Name.
- **Dashboard**: your own history by default, switch to any employee via the dropdown, plus a "Recent Updates" tab showing everyone's entries for today. Mobile-friendly scrollable table.
- **Download Attendance**: pick an employee (admin can pick anyone, regular employees are locked to their own ID), a date/range, and choose **Word (.docx) or PDF** at download time.
- **Change Password**: any logged-in user can change their own 4-digit PIN.
- **Admin / Owner account**: whoever logs in with the Employee ID set in the `OWNER_ID` secret gets admin powers —
  - Edit or delete any attendance entry (Dashboard → History tab).
  - Look up any employee's current 4-digit password (Change Password page → admin panel).
- **History retention**: set `HISTORY_DAYS` to auto-limit + auto-delete history older than N days (0 = keep everything forever).
- **Push notifications** (optional): reminds each employee every ~6 hours to fill today's attendance, and stops automatically once they've submitted that day's entry.
- Sessions are stored in MongoDB too (connect-mongo), so login state survives HF restarts.

## Step 1 — Get a MongoDB Atlas connection string
1. Create a free account/cluster at https://www.mongodb.com/cloud/atlas (if you don't have one already)
2. On your cluster, go to "Connect" -> "Drivers" and copy the connection string, it looks like:
   ```
   mongodb+srv://<username>:<password>@cluster0.xxxxx.mongodb.net/attendance?retryWrites=true&w=majority
   ```
3. Put your DB user's password in it, and add a database name (e.g. `attendance`)
4. In Atlas -> Network Access, allow `0.0.0.0/0` so Hugging Face can connect

## Step 2 — (Optional) Generate push notification keys
```
npm install
npm run generate-vapid
```
This prints a `VAPID_PUBLIC_KEY` and `VAPID_PRIVATE_KEY` pair — copy both. If you skip this, the app works fine, just without push reminders.

## Step 3 — Run locally (for testing)
```
npm install
export MONGODB_URI="mongodb+srv://..."
export OWNER_ID="9478"          # the Employee ID that should have admin powers
export HISTORY_DAYS=0           # 0 = keep all history forever, or e.g. 7 for a rolling 7-day window
export VAPID_PUBLIC_KEY="..."   # optional, from Step 2
export VAPID_PRIVATE_KEY="..."  # optional, from Step 2
npm start
```
The app runs at `http://localhost:7860`.

## Step 4 — Deploy on Hugging Face Spaces (Docker)
1. Create a new Space -> SDK = **Docker**
2. Upload/push all files (server.js, models.js, package.json, Dockerfile, generate-vapid.js, public/, README.md)
3. In Space Settings -> **Variables and secrets**, add:
   - `MONGODB_URI` = your Atlas connection string
   - `OWNER_ID` = the Employee ID that should become the admin/owner (that employee must log in at least once so the account exists)
   - `HISTORY_DAYS` = e.g. `7` to keep a rolling 7-day history, or `0` / leave unset to keep everything forever
   - (optional) `SESSION_SECRET` = any random secret string
   - (optional) `VAPID_PUBLIC_KEY` / `VAPID_PRIVATE_KEY` = for push notification reminders (see Step 2)
   - (optional) `VAPID_SUBJECT` = `mailto:you@example.com`
4. The Space will rebuild and run on port 7860

That's it — data is now stored permanently in MongoDB, no more file-reset issues.

## Notes
- **Passwords**: the 4-digit login PIN is stored as plain text in the database (not hashed), on purpose — this is what lets the `OWNER_ID` admin look up anyone's current PIN from the Change Password page. It's fine for a simple internal 4-digit attendance PIN, but employees shouldn't reuse this PIN anywhere sensitive like banking or email.
- **Cookies/sessions**: since Hugging Face Spaces often embeds apps in an iframe, session cookies are configured with `sameSite: 'none'` and `secure: true` (required for cross-site iframe cookies to work in modern browsers).
- **History retention**: the cleanup job runs once at startup and then daily at 00:30 server time. Until it runs, older entries may still briefly appear if `HISTORY_DAYS` was just changed — this resolves within a day.
