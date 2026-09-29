# Field Receipts to J: Drive

Field guy snaps a receipt, types the BuildOps PO number, vendor and total, and it lands in `J:\Receipts\<Person>\<YYYY-MM>\` within about 5 minutes, named by PO so it's easy to attach to the PO in BuildOps. No receipt emails; you only get an alert if the office PC can't save a file.

## How it works

```
Phone (receipts.html on GitHub Pages)
   │  photo(s) → one PDF, coded, queued on phone if no signal
   ▼
Supabase: "receipts" edge function → private "receipts" bucket + rcpt_receipts table
   ▲
   │  outbound HTTPS every 5 min (no firewall changes)
Office PC scheduled task (Receipt-Sync.ps1) → writes to \\server\share\Receipts (J:)
   │
   └─► edge function marks it synced → n8n "Receipt Notify" → Outlook email to accounting
```

## Files

| File | What it is |
|---|---|
| `sql/01_Receipts_Schema.sql` | Tables, private bucket, RLS deny-all, `rcpt_add_person` / `rcpt_reset_code` helpers |
| `sql/02_Receipts_Seed.sql` | Add the field guy, jobs, cost codes |
| `supabase/functions/receipts/index.ts` | Edge function: phone actions + office agent actions |
| `receipts.html` | Phone capture page (GitHub Pages) |
| `office-agent/Receipt-Sync.ps1` | Sync script that runs on the office PC |
| `office-agent/Install-ReceiptSync.ps1` | One-time installer for the scheduled task |
| n8n: **Receipt Notify** | Already created in your n8n (unpublished) |

## Setup, in order

| # | Step | Where |
|---|---|---|
| 1 | Run `01_Receipts_Schema.sql` | Supabase SQL Editor, new tab **01 Receipts Schema** |
| 2 | Edit and run `02_Receipts_Seed.sql` (person, code, jobs, cost codes) | New tab **02 Receipts Seed** |
| 3 | Deploy edge function `receipts` with **Verify JWT OFF** | Dashboard > Edge Functions, or `supabase functions deploy receipts --no-verify-jwt` |
| 4 | Add the edge function secrets (table below) | Edge Functions > Secrets |
| 5 | Publish **Receipt Notify** (sends from Liam Outlook; swap credential if accounting should see a different sender) | n8n |
| 6 | Set `FN_URL` at the top of `receipts.html`, commit to a repo, e.g. `HAGGERTYbuilds/Field-Receipts` | GitHub Pages |
| 7 | On the always-on office PC, run `Install-ReceiptSync.ps1` as admin | Office PC |
| 8 | Text the field guy the page link + his code. On iPhone: Share > Add to Home Screen | |
| 9 | Test: send a receipt, confirm file on J:, CSV row, accounting email | |

### Edge function secrets

| Secret | Value |
|---|---|
| `RECEIPT_AGENT_SECRET` | Generate one: PowerShell `[Convert]::ToBase64String((1..32 \| % {Get-Random -Max 256}))`. Same value goes into the installer. |
| `RECEIPT_ALLOWED_ORIGINS` | `https://haggertybuilds.github.io` |
| `RECEIPT_NOTIFY_URL` | `https://haggertybuilds.app.n8n.cloud/webhook/receipt-notify` |
| `RECEIPT_NOTIFY_SECRET` | `Wr8FGjK8ME9KWJjR9-U7ky1bwp0mhDLx` (matches the check in the n8n webhook) |
| `RECEIPT_NOTIFY_TO` | Leave unset (no per-receipt emails) |
| `RECEIPT_ADMIN_TO` | Your email, for sync failure alerts |
| `RECEIPT_DISPLAY_ROOT` | `J:\Receipts` |

### Office PC requirements

- Stays on, and doesn't sleep.
- A Windows/domain account (e.g. `HAGGERTY\svc-receipts`) with **write** access to the Receipts folder. It needs "Log on as a batch job" (Windows usually grants it when the task is registered).
- The **UNC path** behind J:. Run `net use` on any office PC and read the J: line (e.g. `\\HCI-FS01\Shared`). Scheduled tasks can't see drive letters.
- Outbound HTTPS to `*.supabase.co`. Ask IT to allow that if the network has a web filter.

## What lands on J:

```
J:\Receipts\Firstname L\2026-09\PO-10482_2026-09-29_Home Depot_142.50.pdf
J:\Receipts\Firstname L\2026-09\PO-10517_2026-09-29_Chevron_68.20.pdf
J:\Receipts\Firstname L\2026-09\_Receipts-2026-09.csv
```

- Month folder follows the upload date.
- PO is cleaned up automatically: `PO# 10482`, `po-10482` and `10482` all become `PO-10482`.
- Duplicate names get ` (2)`, ` (3)`.
- The CSV is a running index accounting can open in Excel. If someone has it open while a receipt syncs, the file still saves and the CSV row is skipped (logged).

## Field guy experience

- Signs in once with his code. The phone remembers it.
- **Take photo** (add more pages for long receipts) or **Upload file** (an emailed PDF receipt).
- PO number, vendor, total, optional note.
- No signal? The receipt stays saved on the phone and sends when service comes back or the page is reopened.
- **Recent** shows each receipt as On phone / Uploaded / On J: / Problem, plus whether office sync is running.

## Day-to-day admin (SQL Editor)

| Task | SQL |
|---|---|
| Add a person | `select rcpt_add_person('Jane D.', 'Jane D', 'CODE-4821', 'jane@haggertybuilds.com');` |
| Reset a code | `select rcpt_reset_code('Jane D', 'NEW-CODE-99');` |
| Turn someone off | `update rcpt_people set active=false where folder_name='Jane D';` |
| Add a job | `insert into rcpt_jobs (job_number, job_name) values ('2415','Grant Street Apartments');` |
| Close a job | `update rcpt_jobs set active=false where job_number='2415';` |
| Anything stuck? | `select rel_path, status, sync_attempts, sync_error, created_at from rcpt_receipts where status<>'synced' order by created_at;` |
| Retry a failed one | `update rcpt_receipts set status='pending', sync_attempts=0 where id='...';` |
| Office PC last check-in | `select value from rcpt_settings where key='agent_last_seen';` |

## Security notes

- All tables have RLS on with no policies, and anon/authenticated have no grants. Only the edge function (service role) reads or writes.
- Bucket is private. The office PC gets 10-minute signed download links, never a Supabase key.
- Access codes are stored as SHA-256 hashes. 10 wrong codes from one IP locks that IP out for 15 minutes.
- Office PC config (holding the agent secret) is readable only by the task account and admins.
- Folder names can't end in a period (Windows drops it), so use `Firstname L` for the folder and `Firstname L.` for the display name.
- Receipts stay in Supabase after syncing as a backup. If you want them deleted after 90 days, that's a small add.
