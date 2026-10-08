# Lead Intake & Email Verification Automation (n8n)

Inbound leads arrive from different places (website forms, WhatsApp, referral partners) with different field names and missing values. This project cleans them up, scores them with an LLM, logs them to Google Sheets, verifies their emails and posts to Slack, and alerts the team whenever something breaks.

**Demo video:** <https://www.loom.com/share/167b9a9d3fc2455799fd9110141d87c2>

## Stack
n8n (self-hosted in Docker) · Groq LLM API · Google Sheets · Slack incoming webhooks · AbstractAPI · JavaScript (Code nodes)

## How it works

**Workflow 1: Intake & Classification** (`workflows/WFlow1-SparkLeadClassifyer.json`)
1. A webhook receives a lead from any source.
2. A Code node normalizes it to `{name, email, phone, source, message, timestamp}`, turning empty strings into `null`.
3. A validation gate requires a name plus an email or phone.
4. Groq classifies the lead as **HOT / WARM / COLD** with a one-sentence reason.
5. The output is parsed and checked before it is trusted.
6. The lead is appended to Google Sheets and summarized in Slack.

**Workflow 2: Email Verification** (`workflows/WFlow2-SparkEmailVerifyer.json`)
1. A Google Sheets trigger fires on each new row.
2. Rows with no email skip the API call and are marked `No Email`.
3. AbstractAPI checks the address, and the sheet's `Email Verified` column is updated (`Yes` / `No` / `No Email`).
4. Slack gets a confirmation for deliverable emails or a follow-up flag for the rest.

## Design decisions
- **HOT / WARM / COLD by purchase intent**, because it maps directly to follow-up priority. The reasoning is stored with each lead so the score can be audited.
- **LLM output is validated** (valid JSON, known label, reasoning present) before it reaches the sheet.
- **One shared alert node.** Each failure branch tags its step, so every alert reads the same way.
- **`Date Added` is the row key** for updates, since the sheet has no ID column.
- Model: `openai/gpt-oss-20b` (the original `llama-3.1-8b-instant` was retired by Groq during development).
- AbstractAPI's **Email Reputation** endpoint is used, and only its deliverability status drives the branching.

## Error handling
Missing fields, LLM failures, unparseable output and AbstractAPI failures each send a Slack alert to a separate alerts channel, containing the failed step, the reason, a timestamp and the raw lead. API calls retry up to 3 times before alerting.

Exercised during development: validation failure, LLM failure, parse failure, invalid API key and rate limiting (429). **Not tested:** exhausted free-tier quota.

## Setup
1. Import both JSON files into n8n.
2. Create these credentials: Groq (Header Auth, `Authorization: Bearer <key>`), Google Sheets OAuth2, Google Sheets Trigger OAuth2, and AbstractAPI (Query Auth, name `api_key`).
3. Replace each `https://hooks.slack.com/services/REPLACE_ME` with your own incoming webhook URL (one for new leads, one for alerts).
4. Create a sheet with the headers: `Name, Email, Phone, Message, Source, Lead Classification, AI Reasoning, Date Added, Email Verified`, then point both Google Sheets nodes at it.

## Known limitations and next steps
- AbstractAPI's free tier rate-limits bursts, so several new rows in one poll can return a 429. Batching the calls (one at a time, spaced out) would fix it.
- No de-duplication yet.
- A failure of the Sheets trigger itself happens before any alert branch can run. An n8n Error Workflow would cover it.
- Replace Google Sheets with a CRM API (e.g. GoHighLevel) and use a real lead ID instead of matching on timestamp.
- Use a Slack app with OAuth instead of webhook URLs.

## In plain English
Every new enquiry, from any source, is tidied up and checked for basic details. An AI then rates how ready the person is to buy, and the result lands in a spreadsheet and Slack within seconds. A second step checks the email is real and flags the ones that need a phone call. If anything breaks, the team gets an alert saying exactly what failed.
