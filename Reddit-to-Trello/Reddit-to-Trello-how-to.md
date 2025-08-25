Reddit → Trello via n8n — Step‑by‑Step Guide
This doc shows how to connect Reddit to n8n, create Trello cards from selected Reddit posts or comments, and capture info like author, content, and post URL, with support for dynamic mapping, labels, and due dates.

1) Prerequisites
n8n (Cloud or self‑hosted)

Public HTTPS URL for n8n (Cloud, Tunnel, or your own domain)

Reddit account with API access (application created under https://www.reddit.com/prefs/apps)

Trello account with a board & list ready

2) Create API Credentials
Reddit
On Reddit, go to Preferences → Apps.

Create a new "script" application.

Note the Client ID, Client Secret.

Generate a refresh token (see Reddit API docs).

In n8n, go to Credentials → New → search Reddit OAuth2 API, and add the saved keys.

Trello
In Trello, create/get your API key and token.

In n8n, go to Credentials → New → search Trello, enter your key and token, and save.

3) Build the Workflow
A minimal workflow uses:

Trigger (Schedule/Cron for periodic, Webhook for on-demand, or Manual for testing)

Reddit (Fetch post(s) or comment(s))

Trello (Create Card)

Recommended:
Trigger → Reddit → (Optional: Filter/IF, Function) → Trello

4) Configure Each Node
A) Trigger
Use Schedule (Cron) or Manual Trigger for now.

Set up interval (e.g., every 10 min).

B) Reddit — Fetch Posts or Comments
Resource: Post or Comment as needed

Operation: "Get Many"

Subreddit: Set your subreddit name(s)

Filters: (Optional) e.g. hot/new/top, or add keywords using an IF/Function node afterward

Credentials: Select your Reddit credential

Test the node to see output like:

text
author, title, selftext/body, created_utc, permalink
For best results, append a deduplication check (see below).

C) (Optional) Filter/Function (for Keywords, Parsing, or Deduplication)
Add an IF or Function node to include/exclude posts/comments based on title/content or unique ID.

D) Trello — Create Card
Operation: Create

Board: Select your target board

List: Select your target list

Name (Title):

text
{{ $json["title"] || $json["body"] || "(No title)" }}
Description (rich details):

text
👤 Author: {{ $json["author"] }}
📝 Content: {{ $json["selftext"] || $json["body"] || "(No content)" }}
📅 Posted: {{ $moment($json["created_utc"] * 1000).format("YYYY-MM-DD HH:mm:ss") }}
🔗 [Reddit Post](https://www.reddit.com{{ $json["permalink"] }})
Labels (optional, must exist on board):

text
["Reddit", "Lead"]  // or insert dynamic mapping via Function node
Due (optional):

text
{{ $now.plus({ days: 2 }).toISO() }}
5) Activate & Test
Activate the workflow.

Allow the trigger (schedule or webhook) to execute.

Review your Trello board—new cards with Reddit details should appear.

6) Optional: Custom Business Logic
Filter for Keywords: Use an IF/Function node after Reddit to include/exclude only posts with keywords (e.g., "lead", "job", etc.).

Deduplication:
Use Trello’s Search Cards node (or add a hash as a label in the card) to prevent creation if the post already exists.

Attachments:
If Reddit posts contain images, extract the URL and attach using Trello node (Additional Fields → Attachments).

7) Example Trello Card Output
Card Title:
OG

Description:
👤 Author:  Specific_Speech_1671         
📝 Content: A dedicated and professional team member with strong problem-solving skills and a commitment to excellence. Adept in communication, leadership, and collaborative work environments. Dressed in formal attire, reflecting professionalism and readiness for corporate engagement.
📅 Posted: 2025-08-22 04:53:23
🔗 Reddit Post

Labels: Reddit, Lead
Due Date: 2 days from run

8) Troubleshooting
No cards in Trello:

Check Reddit permissions/credentials, try with "Manual Trigger" and watch output in n8n.

Make sure Trello credentials and board/list IDs are correct.

Check workflow logs for errors or missed mappings.

Duplicate cards:

Add a search step or include post URL in each card title for uniqueness.

Field missing in card:

Test Reddit node separately and confirm all required fields are present and mapped in Trello node.

9) One-liner Daily Update (optional)
Reddit → Trello automation live: Creates cards from new Reddit posts including author, content, time, and link—instantly organizing your opportunities in Trello.

10) Extensions & Good Practices
Map only posts from specific authors, or filter posts with attachments.

Use labels to categorize based on subreddit or post flair.

Schedule error notifications via email/Slack in case card creation fails.

Quick Reference — Top Expressions
What you want	Paste in Trello field
Card Title	`{{ $json["title"]
Author in Description	{{ $json["author"] }}
Content	`{{ $json["selftext"]
Timestamp	{{ $moment($json["created_utc"]*1000).format("YYYY-MM-DD HH:mm:ss") }}
Reddit Post URL	https://www.reddit.com{{ $json["permalink"] }}
Due Date	{{ $now.plus({ days: 2 }).toISO() }}
Done!
Now you can seamlessly capture new Reddit content as Trello cards, keeping your board in sync with Reddit leads or insights. Extend or adapt this workflow for advanced logic as needed!

