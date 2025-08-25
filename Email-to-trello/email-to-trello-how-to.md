Email → Trello via n8n — Step-by-Step Guide

This doc shows how to watch an inbox in n8n, create Trello cards from emails, and auto-apply labels based on subject keywords (Support Request, Support, Help Desk, Issue, Bug).

1) Prerequisites

n8n (Cloud or self-hosted)

Email inbox

IMAP (fastest to set up) or

Gmail API OAuth (more secure; needs Google Cloud OAuth)

Trello account with a board and list ready

2) Create Credentials in n8n
Email

IMAP: Credentials → New → IMAP / Email

Host, Port, Username (full email), Password (or App Password)

Gmail API: Credentials → New → Gmail OAuth2

Use your Google Cloud Client ID/Secret and n8n’s redirect URI

Trello

Credentials → New → Trello

API Key & Token (from Trello developer page)

3) Build the Workflow

Create a workflow with three nodes:

Email Trigger (IMAP or Gmail)

IF (subject filter)

Trello (Create Card)

Connect: Email Trigger → IF → (true) Trello. The false branch ends (do nothing).

4) Configure Each Node
A) Email Trigger

IMAP

Mailbox: INBOX

Action: Mark as Read (or Nothing)

Format: Simple (provides subject, text, html)

Gmail Trigger

Choose your Gmail credential

Watch new messages (you’ll get Subject and snippet)

Tip: Press Test step and send yourself an email to see the available fields in the node’s Output.

B) IF — allow only relevant emails

Condition:

Left value (Expression):

{{ ($json.Subject || $json.subject || '').toLowerCase() }}


Operator: matches regex

Right value:

support request|support|help desk|issue|bug


This routes matching emails to true → Trello. Others go to false (stop).

C) Trello — Create Card

Resource: Card

Operation: Create

List: select your list

Name (Title):

{{ $json.Subject || $json.subject || 'No subject' }}


Description (prefer html/text/snippet as available):

{{ $json.html || $json.text || $json.snippet || '' }}

Labels (automatic)

Trello’s API wants label IDs (array). Add Additional Fields → Label IDs and paste this expression (replace the two IDs once for your board):

{{
  (() => {
    // Put your real label IDs here (one-time setup)
    const LABEL_SUPPORT = '605c72efb111111111111111'; // e.g. “Support”
    const LABEL_ISSUE   = '60aa12cd222222222222222'; // e.g. “Issue/Bug”

    const s = ($json.Subject || $json.subject || '').toLowerCase();
    if (/(support request|help desk|support)/i.test(s)) return [LABEL_SUPPORT];
    if (/(issue|bug)/i.test(s)) return [LABEL_ISSUE];
    return []; // no label if no keyword
  })()
}}


If you prefer label names instead of IDs, add a Trello → Label → Get Many node earlier, cache the id for each name, and map dynamically.

5) Getting Your Trello Label IDs (one-time)

Option A — From Trello UI

Board menu → More → Labels → click edit (pencil) on a label

Look at the browser URL: it ends with label=<LONG_ID> → copy that value

Option B — From n8n

Add node: Trello → Resource: Label → Operation: Get Many

Choose your Board

Test step → you’ll see all labels with id and name

6) Activate & Test

Activate the workflow (top-right)

Send an email with subject like “Support Request: Login issue”

A Trello card should appear with:

Title = email subject

Description = email body

Label = “Support” (or “Issue/Bug” depending on subject)

7) Optional Enhancements

Attachments: In Email Trigger enable “Download Attachments”, then add Trello → Add Attachment to Card after card creation.

Prevent duplicates: Before creating a card, add Trello → Card → Get Many and check if a card with the same subject exists.

Route to different lists: Use a Switch on subject keywords and send to different Trello lists.

8) Troubleshooting

Bad request (Trello): Ensure List selected, and Label IDs is an array of valid IDs.

IF node never true: Check the exact subject value in the Email Trigger output; adjust regex/keywords.

Gmail OAuth errors: Redirect URI mismatch or expired secrets—recreate the Gmail credential and ensure URIs match exactly.

No emails picked up (IMAP): Confirm IMAP is enabled in your mail settings and credentials/host/port are correct.

9) Example Output (what gets created)

Card Name: Issue: Payment failing on checkout

Description: full email body (HTML or text)

Labels: Issue

10) One-liner Daily Update (optional)

Email → Trello automation live: filters by subject keywords, creates cards with subject/body, and auto-labels as Support or Issue/Bug.

Quick Reference — Expressions

Subject (title)

{{ $json.Subject || $json.subject || 'No subject' }}


Body (description)

{{ $json.html || $json.text || $json.snippet || '' }}


Label IDs (by keyword)

{{
  (() => {
    const LABEL_SUPPORT = '605c72efb111111111111111';
    const LABEL_ISSUE   = '60aa12cd222222222222222';
    const s = ($json.Subject || $json.subject || '').toLowerCase();
    if (/(support request|help desk|support)/i.test(s)) return [LABEL_SUPPORT];
    if (/(issue|bug)/i.test(s)) return [LABEL_ISSUE];
    return [];
  })()
}}


IF condition (regex)

Left (Expression): {{ ($json.Subject || $json.subject || '').toLowerCase() }}
Operator: matches regex
Right: support request|support|help desk|issue|bug


Done! You now have a clean, production-ready Email → Trello automation with subject-based filtering and automatic labels.