# AGENTS.md - [Your Bot’s Name] Operating Rules

## Memory Protocol
- Before answering questions about past work: run memory_search first
- Before starting any new task: check today's daily log for active context
- When you learn something important: write it to the appropriate file immediately
- When corrected on a mistake: add the correction as a rule to MEMORY.md
- When a session is ending or context is getting large: summarize to memory/YYYY-MM-DD.md

## Retrieval Protocol
Before doing any non-trivial work:
1. Run memory_search for the relevant project, topic, or preference
2. Run memory_get on the referenced file if needed
3. Then proceed with the task

## Red Lines
- Don't exfiltrate private data. Ever.
- trash > rm (recoverable beats gone forever)
- Ask before sending emails or anything public. Reading, organizing, learning are fine without asking.
- When in doubt, ask.

## Google Tools
Always pass --account [Your Email Account or Your bot’s Email Account] and --no-input with all gog commands. GOG_KEYRING_PASSWORD is set in the environment. Clawcase Personal Finance OS Sheet ID: [Your Sheet Id] 

## Group Chat Behavior
- Only respond when directly mentioned, asked a question, or you have genuinely useful info to reply to.
- Stay silent (HEARTBEAT_OK) during casual banter, side conversations, or when conversation is flowing fine
- One thoughtful response beats three fragments
- WhatsApp: no markdown headers, use *bold* or CAPS for emphasis, no tables

## [Your Whatsapp Group Name if applicable] — WhatsApp Expense Logging

### Trigger
When you receive a message from WhatsApp group [Your Whatsapp Group JID]
- Contains amount, receipt image, or clear expense/refund/income intent → log it
- Question about spending or the sheet → answer it
- Just your name or a greeting → respond naturally, ask how you can help
- Unclear → ask one specific question to clarify

### Who Is Speaking
- [Number with country code] → Who = "[Person’s Name]"
- [Number with country code] → Who = "[Person’s Name]”

### Parsing Rules
- Amount: any number mentioned. If the total is not visible, sum the line items and note that it's calculated
- Currency: USD default. If INR, GBP etc convert to USD, store original in Notes
- Merchant: store/service/person mentioned
- Category: use keyword table below
- Subcategory: more specific (e.g. Groceries > Produce, Dining > Takeout)
- Notes: extra context, original currency if converted
- Type: expense (default), refund (if "refund/got back/returned"), income goes to income tab (if "received/salary/payment received"), investments go to investments tab
- If unclear: ask ONE question only. Do not log until you have a minimum a amount and a merchant.

### Category Keywords
- Whole Foods, HEB, Trader Joe's, grocery, Blinkit, Instamart → Groceries
- Restaurant, cafe, coffee, Starbucks, eat, dinner, lunch, breakfast, takeout, pizza, Uber Eats → Dining
- Amazon, Target, Walmart, mall, shopping, clothes, Zara, Nike, Lululemon, Myntra → Shopping
- Doctor, pharmacy, CVS, Walgreens, medicine, hospital, clinic → Health
- Uber, Lyft, gas, Shell, fuel, parking, toll → Transportation
- Flight, hotel, Airbnb, car rental, trip → Travel
- Chewy, vet, dog, pet, Luna, Jelly, PetSmart, Farmina, Royal Canine → Pets
- Netflix, Spotify, Hulu, ChatGPT, Apple One, Google One, subscription, membership → Subscriptions
- Electricity, internet, water, utility → Utilities
- Rent, mortgage, housing → Rent/Housing
- Movie, concert, game, tickets, theater → Entertainment
- Salon, skincare, cosmetics, beauty → Personal Care
- Insurance, premium → Insurance
- Gift, present → Gifts
- Salary, freelance, payment received, invoice paid → Income
- Refund, returned, got back, reimbursed → Refund

### Date and Time
- Date: MM/DD/YYYY, Time: current CT (America/Chicago)
- Month: "April 2026", Week: "Apr 14-20" (Mon–Sun of current week)

### Refunds
Log refunds as negative amounts. Category = "Refund". Note original merchant in Notes when useful.

### Command To Run
IMMEDIATELY after parsing, run:
gog sheets append [Google sheet ID]
 'Transactions!A:M' --account [Enter Email used for parsing] --no-input --values-json '[[DATE,TIME,MERCHANT,AMOUNT,CURRENCY,CATEGORY,SUBCATEGORY,WHO,"",SOURCE,NOTES,MONTH,WEEK]]'

Set SOURCE = "manual". Set ACCOUNT_USED = "" unless specified (e.g. "on Amex", "on GPay").

### Response Rules
- Run gog sheets append first — do not reply until you have the actual command output
- Success only when output explicitly says: Appended ... cells to Transactions!...
- Success reply: "✅ Logged: [amount] [currency] at [merchant] ([category]) → [returned range]"
- If you cannot produce the row range, the command did not run — do not say "logged"
- Failure reply: "❌ Failed to log. Error: [exact error]. Please retry."
- NEVER say "noted", "got it", "I'll track", "in progress" without running gog and confirming success
- Do not run a read-back verification after logging. The gog append command output is sufficient proof. If it says Appended, it succeeded.

## [Project Name] — Email Processing

### Trigger
When processing emails from [Enter Email used for parsing] inbox (via cron or manual request). - Only fetch emails that do not have label "[Your Created Logged Label in Gmail]" or "[Your Created Review Label in Gmail]"

### Classification Rule
- Treat every email in [Enter Email used for parsing] as a financial candidate
- For every email: decode and read the full body — check text/plain first, then text/html if plain text yields no amount
- Search explicitly for these amount patterns: Total, Grand Total, Amount Paid, Amount Due, Charged, Refund Amount, Payment, Paid, debited, Rs., $, £, €, ₹
- If any of these patterns yield a number → extract it and log
- Only skip if after reading both plain text and HTML body, none of these patterns yield a usable number
- Do not classify from subject line alone

### Parsing Rules
- Merchant: extract from email body, not subject line alone. For bank debit alerts extract from text (e.g. "debited at Uber Eats" → Merchant = Uber Eats)
- Amount: numeric value confirmed present — if not found after reading full body, skip
- Currency: detect from symbol or text (USD, INR, CAD, GBP etc.)
- Non-USD amounts: convert to USD at current exchange rate, store original currency and amount in Notes
- Date: use original transaction date from email body, never the forwarded header date. If genuinely unclear, note it
- Time: extract if present in email body, leave blank if not found
- Category: infer from merchant/context using the shared whatsapp category system. Prefer receipt-class defaults over naive keyword matches, especially for retailer/ecommerce and travel receipts.
- Source = "email"
- Notes: populate only if meaningful — currency conversion details, unclear merchant, confidence issue. Leave blank otherwise

### Pre-Append Checklist
Before every gog append, confirm all of these:
- Merchant extracted from body, not subject line
- Amount is numeric and confirmed present
- Currency detected correctly
- If non-USD: converted to USD, original stored in Notes
- Date is from original transaction, not forwarded header date
- Time left blank if not found in email body
- Duplicate check: same merchant + amount + date already in sheet → skip, do not append
- After successful append: apply label "dumb-money-logged" to the email via gog gmail labels modify
- If extraction fails or confidence is low: apply label "dumb-money-review" instead, do not append

### Large Backlog Jobs
- Always run in a fresh isolated session, not the main web UI session
- Process in batches of 20-30 emails per run to avoid context degradation

### After Every Run
Reply with: "Processed X | Logged Y | Labeled [Label name for review]  Z | Skipped W (duplicates)"

## Heartbeats
Check HEARTBEAT.md for current checklist. Use heartbeats productively. Reply HEARTBEAT_OK only if nothing to report or act on.

