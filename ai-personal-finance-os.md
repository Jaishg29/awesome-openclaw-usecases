# AI Personal Finance OS

Log expenses, income, and track networth from WhatsApp. Capture receipts from Gmail automatically and maintain a live financial dashboard in Google Sheets.

## Pain Point

Every finance app becomes a chore. You stop opening it, stop categorizing, and quit. The problem isn't the app — it's the behavior change required.

This system works differently: you're already in WhatsApp all day, and your receipts already land in Gmail. OpenClaw sits between that existing behavior and a Google Sheet, capturing everything automatically.

Most bank-sync apps (Monarch, Copilot) only work with US banks via Plaid. This works with any bank, any currency, any country.

## What It Does

- **WhatsApp logging** — text or photo a receipt → logged instantly with category, currency conversion, duplicate detection  
- **Email capture** — financial emails auto-parsed and logged every few days  
- **Live dashboard** — month picker drives KPI cards, category breakdown, budget vs actual, net worth, savings trajectory  
- **Multi-person** — one WhatsApp group, each person tracked separately and together  
- **Any currency** — INR, CAD, GBP, EUR auto-converted to USD, original preserved in Notes  
- **Your data** — everything lives in a Google Sheet you own

## Skills Needed

- **gog** — required. `openclaw skills install gog`

## Choose Your Email Capture Route

Pick one before starting. The WhatsApp logging, AGENTS.md, and Google Sheet are identical across all three — only the email capture differs.

| Route | How | Best for |
| :---- | :---- | :---- |
| **A — Dedicated Gmail \+ forwarding** (recommended) | Create a new Gmail for your agent. Forward financial emails from your personal Gmail using filters. | Clean separation, any bank, any country |
| **B — Access your existing Gmail** | Authenticate gog with your personal Gmail directly. Agent reads from your Purchases label. | Less setup, comfortable sharing inbox access |
| **C — Plaid bank sync** | Replace the email script with a Plaid API script (free developer tier). | US banks, most automatic |

---

## Setup

Assumes OpenClaw is installed and WhatsApp is connected. See [openclaw.dev](https://openclaw.dev) first.

### 1\. Create a dedicated Google account for your agent *(Route A only)*

Create a new Gmail — e.g. `myagent@gmail.com`. Financial emails from your personal Gmail will be forwarded here. Skip if using Route B.

### 2\. Set up Google Cloud and gog CLI

- Create a GCP project → enable **Sheets API** and **Gmail API**  
- Credentials → Create OAuth client ID (Desktop app) → download JSON  
- **Publish the OAuth app to Production** (OAuth consent screen → Publish app). Without this tokens expire every 7 days and logging breaks silently.  
- Install and authenticate gog:

gog auth credentials /path/to/client\_secret.json

gog auth add myagent@gmail.com \--services sheets,gmail \--remote

**Headless server:** Add `Environment="GOG_KEYRING_BACKEND=file"` and `Environment="GOG_KEYRING_PASSWORD=yourpassword"` to your OpenClaw systemd service or gog auth fails silently.

### 3\. Copy the Google Sheet template

Open → [**Clawcase Personal Finance OS Template**](https://docs.google.com/spreadsheets/d/1EkDabzXIrmRWb7tHNPjzRGgwWxgkaxNA0e1J_3hm-Ns/copy) → File → Make a copy

Note your sheet ID from the URL. Share with your agent Gmail (Editor access).

### 4\. Set up your WhatsApp expense group

If doing multi-people finance tracking, then create a group, add everyone. Your number is also your AI’s number. To activate your AI in the group, we need to first find the group JID:

openclaw sessions

\- Find your group name in the output — the unique JID might look like \`120363408297548645@g.us\`  
\- Configure OpenClaw to allow the group and disable requireMention:

openclaw config set channels.whatsapp.accounts.YOUR\_ACCOUNT.groupPolicy allowlist 

openclaw config set channels.whatsapp.accounts.YOUR\_ACCOUNT.groupAllowFrom '\["YOUR\_GROUP\_JID"\]' 

openclaw config set channels.whatsapp.accounts.YOUR\_ACCOUNT.groups.YOUR\_GROUP\_JID.requireMention false openclaw gateway restart

**Known bug:** A `default` WhatsApp account reappears after every restart with open groupPolicy. Remove it: `openclaw config unset channels.whatsapp.accounts.default`

### 5\. Add instructions to AGENTS.md

Open AGENTS.md in the OpenClaw web UI. Add the Dumb Money sections from [agents.md](https://github.com/Jaishg29/awesome-openclaw-usecases/blob/main/agents.md#agentsmd---your-bots-name-operating-rules). Replace:

- `YOUR_SHEET_ID` — sheet ID from Step 3  
- `YOUR_GROUP_JID` — group JID from Step 4  
- `YOUR_NUMBER` / `PARTNER_NUMBER` — phone numbers in international format on your whatsapp group  
- `myagent@gmail.com` — your agent Gmail if using separate mailbox with gmail forwarding or your own email if using that

WhatsApp group sessions only load AGENTS.md — not memory files. Keep the Dumb Money instructions in AGENTS.md or they won't work in the group.

### 6\. Set up Gmail forwarding *(Route A)*

In your personal Gmail → Settings → Forwarding and POP/IMAP → Add a forwarding address → enter your agent Gmail → verify the confirmation email.

Create **two separate filters** using the Subject field. You can change filters if you notice different keywords show up for you in your mails)

**Filter 1 — Order confirmations:**

subject:"ordered" OR subject:"order confirmation" OR subject:"thank you for your order" OR subject:"thanks for your order" OR subject:"thanks for your purchase" OR subject:"thanks for your tip" \-from:quora \-from:googlemaps

**Filter 2 — Receipts and refunds:**

subject:receipt OR subject:debited OR subject:credited OR subject:refund OR subject:"refund request" OR subject:"return processed" OR subject:"return request" \-from:quora \-from:googlemaps

For each filter: click Create filter → check **Apply label** → create a new label called `Personal-Finance-os` → also check **Forward to** your agent Gmail → check **Also apply filter to matching conversations** → Create filter.

The label helps you debug what's being captured. The "Also apply to matching conversations" option retroactively labels past emails — but it does NOT retroactively forward them. For historical emails, you need to manually select them in your Purchases folder and forward to your agent Gmail one by one.

Repeat both filters for any partner's Gmail.

*(Route B: skip. Route C: skip and use Plaid script instead of dm\_email\_sync.py)*

### 7\. Set up the email sync script and cron

Verify Python 3: `python3 --version` (install with `sudo apt install python3` if missing)

Download [dm\_email\_sync.py](https://github.com/Jaishg29/awesome-openclaw-usecases/blob/main/dm_email_sync_py.md) → place in `~/.openclaw/workspace/` → update `ACCOUNT` and `SHEET` at the top.

Create Gmail labels for state tracking:

gog gmail labels create "dumb-money-logged" \--account myagent@gmail.com \--no-input

gog gmail labels create "dumb-money-review" \--account myagent@gmail.com \--no-input

Test manually: `python3 ~/.openclaw/workspace/dm_email_sync.py`

Then set up the cron:

openclaw cron add \--name "Email Expense Sync" \--cron "0 20 \*/3 \* \*" \--tz "America/Chicago" \\

  \--session isolated \\

  \--message "Process emails in myagent@gmail.com. Log financial transactions to the Dumb Money sheet. Reply with a summary." \\

  \--deliver \--channel whatsapp \--to "YOUR\_GROUP\_JID"

### 8\. Test

Send in your WhatsApp group: `$45 HEB groceries`

Expected reply: `✅ Logged: 45 USD at HEB (Groceries) → Transactions!A2:M2`

The row reference (`Transactions!A2:M2`) must appear — if the agent says "logged" without it, the gog command didn't actually run. Debug: `openclaw logs --follow --max-bytes 1000000`

---

## AGENTS.md Instructions

See [agents.md](https://github.com/Jaishg29/awesome-openclaw-usecases/blob/main/agents.md#agentsmd---your-bots-name-operating-rules) for the full AGENTS.md text to copy into your workspace.

---

## Customization

- **Solo use** — remove partner phone mapping, hardcode WHO to your name  
- **Agents.md** \- go through carefully and edit/add relevant info to the template to make it yours.  
- **Dm\_email\_sync.py \-** go through carefully and edit/add relevant info to the template to make it yours. Might contain data relevant to my spending, merchants and patterns. You can ask your claude to train itself and update this over time as it sees different merchants and transactions, mine was trained and added on historical data in under 2 hrs  
- **More people** — add more phone → name mappings in Who Is Speaking  
- **Different channel** — works with Telegram or Discord, change the channel in cron and AGENTS.md trigger  
- **Bank sync** — replace dm\_email\_sync.py with a Plaid API script, everything else stays the same  
- **Investment tracking** — add an Investments tab, log trades via WhatsApp the same way  
- **Net worth projection** — add a projection tab with compound interest formulas referencing your Income and Transactions data

## Related Links

- [gog CLI](https://gogcli.sh)  
- [Google Cloud Console](https://console.cloud.google.com)  
- [Plaid API docs](https://plaid.com/docs/) *(Route C)*  
- [OpenClaw docs](https://openclaw.dev)

## The Key Insight

Finance apps fail because they ask you to change your behavior \- this doesn't. Some people are also always concerned about connecting with banks directly, this makes it secure, but a longer one time set up. You're already in WhatsApp. Your receipts and transactions already land in Gmail. The agent captures what's already happening — nothing new to remember. You can get proactive alerts on whatsapp or chat with your AI about your entire financial life

*Built and verified by Jaishree Garg — April 2026*  
