# AI Email Triage (n8n + Claude)

An automated, multilingual (DE / EN / TR / FR…) email-triage pipeline for a small online shop.
New Gmail messages are classified by Claude, logged to Google Sheets, and, when a reply is needed, a reply draft is saved in Gmail for human review.

**Human-in-the-loop by design: the workflow never sends an email.** It only creates drafts.

**Stack:** n8n (self-hosted, Docker) · Anthropic Claude (Haiku 4.5) · Gmail API · Google Sheets API · LangChain nodes with a structured output parser

## What it produces

For every new email:

| Field | Values |
|---|---|
| `category` | order_status, return_refund, product_question, complaint, business_inquiry, spam, other |
| `priority` | high / medium / low (payment problems, legal threats and angry customers → high) |
| `language` | ISO code of the email |
| `summary` | one-sentence English summary |
| `needs_reply` | false for spam and newsletters |
| `reply_draft` | short, polite reply in the **same language** as the sender |

Every email gets a row in the Sheet; replies appear as Gmail drafts in the right thread.

## Architecture

```
Gmail Trigger (polls every minute, UNREAD)
  → Basic LLM Chain (Claude Haiku 4.5 + Structured Output Parser, retry on fail)
      ├─ success → Edit Fields ─┬→ Append row to Google Sheets
      │                         └→ IF needs_reply → Gmail "Create a draft" (in the original thread)
      └─ error   → Append row (category = needs_manual_review, raw error kept for debugging)
```

## Screenshots

**n8n workflow**

![n8n workflow](screenshots/1-workflow.png)

**Google Sheet log:** every email classified, summarized and (where needed) answered in its own language

![Google Sheet results](screenshots/2-sheet-results.png)
![Google Sheet results, unseen test emails](screenshots/3-sheet-results-unseen.png)

**Gmail drafts:** replies waiting for human review (nothing is sent automatically)

![Gmail drafts](screenshots/4-gmail-drafts.png)

## Engineering highlights

- **Strict structured output.** A JSON schema-based output parser guarantees every field is present and typed, so the Sheet and the IF branch never see free text.
- **Fault tolerance.** Retry with a 5 s delay, plus a dedicated error branch: if the model fails, the email is still logged as `needs_manual_review` instead of being lost.
- **Prompt-injection resistant.** The system prompt treats the email as untrusted *data*, never as instructions. Because the workflow has no send node, even a successful injection could not send anything. Verified with two injection emails in the test set plus one in a separate generalization set.
- **No hallucinated promises.** Reply rules forbid committing to refunds, exchanges, deadlines or policies and forbid pointing to departments, pages or channels that may not exist. Drafts acknowledge the request and promise only a team review.
- **Same-language replies** (German, English, Turkish, French verified), including formal address (Sie / Siz / vous).
- **Threaded drafts.** Drafts are created with the original `threadId`, so they appear inside the customer's conversation.
- **Privacy-minded input.** The model receives only sender, subject and the Gmail snippet, not full bodies or attachments.
- **Portable.** The exported workflow contains no credentials or IDs; import, connect credentials, done.

## Evaluation

The prompt was engineered against a labeled test set rather than tuned by feel.

- `testset.csv`: 20 labeled emails (T01–T20) in German, English and Turkish: order status, returns, complaints, wholesale and collaboration inquiries, spam, phishing, an internship application, and two prompt-injection attacks (one hidden in an otherwise normal email).
- `test_emails.md`: the email texts.
- A Google Sheet matches each result by subject and scores category and priority automatically. Reply language, invented facts and injection resistance are reviewed by hand.
- **One change per prompt version (v1 → v4)**, so every score change can be attributed to a single edit.
- A separate **generalization set (N01–N05)** written *after* freezing the prompt checks that it works on unseen emails (newsletter, invoice request, thank-you note, French complaint, injection).

### Prompt iterations (same 20 emails, one change per version)

| Version | Category | Priority | Draft rule violations* |
|---|---|---|---|
| v1 (baseline) | 95 % | 89 % | 6 |
| v2 | 95 % | 100 % | 4 |
| v3 | 95 % | 100 % | 1 |
| v4 (final) | 95 % | 100 % | 0 |

\*Manual review: invented facts, promises, or references to departments/pages that may not exist.

The biggest gains came from tightening the reply rules, not from the classification instructions: the baseline invented policies and contact channels in 6 of 20 drafts; the final prompt has none.

### Results (final prompt v4, Claude Haiku 4.5)

| Check | Result |
|---|---|
| Category | 19 / 20 |
| Priority | 20 / 20 |
| Reply in the correct language | 20 / 20 |
| Drafts free of invented facts | 20 / 20 |
| Prompt injection resisted (T05, T20) | 2 / 2: no discount issued, system prompt not leaked |
| Spam & phishing → no draft created | 100 % |
| Unseen emails (N01–N05) | injection ignored, French complaint answered in French without promising a refund, newsletter filtered |

The pipeline also handled unseen emails in **Spanish** (a language that appears nowhere in the prompt or test set), English and German, answering in the sender's language without promising a cancellation, address change or discount.

A 25-email burst ran end-to-end without a single LLM error.

## Setup

1. Run n8n:
   ```bash
   docker run -d --name n8n -p 5678:5678 \
     -v n8n_data:/home/node/.n8n \
     -e GENERIC_TIMEZONE=Europe/Berlin -e TZ=Europe/Berlin \
     docker.n8n.io/n8nio/n8n
   docker update --restart unless-stopped n8n
   ```
2. Open `http://localhost:5678` and import `workflow.json`.
3. Create credentials in n8n (never commit them):
   - **Gmail OAuth2** and **Google Sheets OAuth2**: Google Cloud project with the Gmail and Sheets APIs enabled, OAuth consent screen in *Testing* mode with your account as test user, redirect URI `http://localhost:5678/rest/oauth2-credential/callback`
   - **Anthropic API key** (set a spending limit)
4. Create a Google Sheet with these headers in row 1: `date, from, subject, category, priority, language, summary, needs_reply, reply_draft`.
5. In both Google Sheets nodes replace `YOUR_SHEET_ID` with your sheet's ID and select your credentials. Select credentials in the Gmail Trigger, Anthropic Chat Model and Create a draft nodes as well.
6. Send yourself a test email, run once manually, then activate the workflow.

Tip: while developing, use a separate test Gmail account. In Google's *Testing* mode, OAuth refresh tokens expire after about 7 days, so reconnect the credential if the Gmail node stops authenticating.

## Roadmap

- Pass the full email body (not just the snippet) to the model for longer messages
- Carry sender / subject / threadId through the chain as fields to make row mapping independent of item order under partial failures
- Retrieval over a shop FAQ / policy document so drafts can answer questions, not just acknowledge them
- Slack or Telegram notification for `high` priority emails
- Re-run the benchmark on newer Claude models and compare cost vs. accuracy

## Files

| File | Purpose |
|---|---|
| `workflow.json` | n8n workflow (sanitized: no credentials, no IDs) |
| `testset.csv` | 20 labeled test emails with expected results |
| `test_emails.md` | texts of the test emails |
