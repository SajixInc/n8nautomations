
# GitHub Issues → Trello Cards Sync in n8n (Step‑by‑Step, with exact node settings)

> This guide builds the workflow you showed in your screenshot: **GitHub Trigger → Edit Fields (Set) → Switch → Trello (Open/Reopen) / Trello (Closed)**.  
> You’ll end up with: new GitHub issues creating Trello cards, and state changes moving/updating the matching card.

---

## What this workflow does

- **Listens** for GitHub *Issues* events (`opened`, `reopened`, `edited`, `closed`).
- **Creates** a Trello card when an issue is **opened** or **reopened**.
- **Finds & updates/moves** the matching card when the issue is **edited** or **closed**.
- (Optional) Mirrors comments and attaches the GitHub link to the card.

---

## Prerequisites

- **GitHub**: Personal Access Token (repo read + webhook permissions) and repo admin access.
- **Trello**: API key + token, access to the board you’ll use.
- **n8n**: Cloud or self-hosted instance reachable via HTTPS.

---

## Final topology (visual)

```mermaid
flowchart LR
    A[GitHub Trigger<br/>Issues events] --> B[Edit Fields<br/>(Set)]
    B --> C[Switch on action]
    C -- opened / reopened --> D[Trello: Create Card]
    C -- edited --> E[Find Card] --> F[Trello: Update Card]
    C -- closed --> E2[Find Card] --> G[Trello: Update Card<br/>(move to Done)]
```

**Note:** “Find Card” can be done with **List → Get All Cards** (by list) and a filter that matches the issue number in the card name.

---

## Node‑by‑node build

### 1) GitHub Trigger
- **Node name:** `Github Trigger`
- **Type:** *GitHub Trigger* (not generic Webhook)
- **Events:** `Issues` (optionally also `Issue comment` if you’ll sync comments)
- **Owner/Repository:** your repo
- **Auth:** your GitHub credentials/token

> Fire a test by creating/editing an issue so you can see payloads in Executions.

---

### 2) Edit Fields (Set)
We’ll **normalize the payload** so later nodes use flat, easy fields.

- **Node name:** `Edit Fields`
- **Type:** *Set*
- **Mode:** *Manual*
- **Add the following fields** (use the right‑side “gear → Add Field → String/Number” and add expressions):

| New field (left) | Expression (right) |
|---|---|
| `action` | `{{ $json["body"]["action"] }}` |
| `issue_id` | `{{ $json["body"]["issue"]["id"] }}` |
| `issue_number` | `{{ $json["body"]["issue"]["number"] }}` |
| `title` | `{{ $json["body"]["issue"]["title"] }}` |
| `description` | `{{ $json["body"]["issue"]["body"] || "No description provided." }}` |
| `html_url` | `{{ $json["body"]["issue"]["html_url"] }}` |
| `state` | `{{ $json["body"]["issue"]["state"] }}` |
| `labels` | `{{ $json["body"]["issue"]["labels"] }}` |

> After running once, open the **Execution** and confirm these fields exist on the `Edit Fields` node output.

---

### 3) Switch (by action)
- **Node name:** `Switch`
- **Type:** *Switch (Rules)*
- **Property:** `{{ $json["action"] }}`
- **Rules:**
  - **Equals** → `opened`
  - **Equals** → `reopened`
  - **Equals** → `edited`
  - **Equals** → `closed`

You’ll now have 4 outputs from the Switch node.

---

## Branch A: opened / reopened → Create Trello card

### 4) Trello: Create Card
- **Node name:** `Opened / Reopened issues`
- **Type:** *Trello*
- **Resource:** `Card`
- **Operation:** `Create`
- **Board:** select your board
- **List:** select your intake list (e.g., **To Do**)
- **Name (use expression):**
  ```
  [#{{ $json["issue_number"] }}] {{ $json["title"] }}
  ```
- **Description (use expression):**
  ```
  {{ $json["description"] }}

  ---
  Source: {{ $json["html_url"] }}
  Issue ID: {{ $json["issue_id"] }}
  ```
- (Optional) **Attachment** — add another Trello node:
  - **Resource:** `Attachment`
  - **Operation:** `Create`
  - **Card:** use *‘From previous node’* → the card you just created
  - **URL:** `{{ $json["html_url"] }}`

> **Pro tip:** By prefixing the card name with `[#<issue_number>]`, you can easily find it later.

---

## Branch B: edited → Find & update the matching card

We need the card ID. There are two simple ways:

### Option 1 (simple): Get cards from the lists where it may be
If your team moves cards from **To Do → Doing → Done**, an *edited* issue likely sits in **To Do** or **Doing**. We’ll check **Doing** below; add other lists if needed.

#### 5) Trello: Get All Cards (Doing)
- **Node name:** `Get cards - Doing`
- **Type:** *Trello*
- **Resource:** `List`
- **Operation:** `Get All Cards`
- **List:** choose your **Doing** list (or **List ID** via expression)

This node outputs one item **per card** in Doing.

#### 6) IF: keep only the matching card
- **Node name:** `Match issue → card`
- **Type:** *IF*
- **Condition:**
  - **Value 1 (string):** `{{ $json["name"] }}`
  - **Operation:** `contains`
  - **Value 2 (string):** `[#{{ $node["Edit Fields"].json["issue_number"] }}]`

Cards that match go to the **true** branch.

#### 7) Trello: Update Card
- **Node name:** `Update card (edited)`
- **Type:** *Trello*
- **Resource:** `Card`
- **Operation:** `Update`
- **Card:** **Use expression** → `{{ $json["id"] }}`  (this is the matched card’s id)
- **Fields to update:** (click **Add Field**)
  - **Name:** `[#{{ $node["Edit Fields"].json["issue_number"] }}] {{ $node["Edit Fields"].json["title"] }}`
  - **Description:** 
    ```
    {{ $node["Edit Fields"].json["description"] }}

    ---
    Source: {{ $node["Edit Fields"].json["html_url"] }}
    Issue ID: {{ $node["Edit Fields"].json["issue_id"] }}
    ```

> If your card might be in **To Do** as well, duplicate steps **5–7** pointed at the **To Do** list and run the same filter/update (or first try Doing, if not found, try To Do).

### Option 2 (cleaner at scale): Store a mapping (recommended)
- After **Create Card**, add **n8n Data Store** → *Save `{ issue_id → card_id }`*.
- On *edited/closed*, read the Data Store by `issue_id` and you instantly have the `card_id`.  
This avoids fetching all cards.

---

## Branch C: closed → Move the card to “Done” (and optionally mark complete)

Follow the **same find steps** as Branch B to get the card, then update the list.

#### 5′) Trello: Get All Cards (Doing / To Do / Anywhere you expect it)
- Repeat **Get All Cards** + **IF filter** until you find the card.

#### 6′) Trello: Update Card (move it)
- **Node name:** `Move to Done`
- **Type:** *Trello*
- **Resource:** `Card`
- **Operation:** `Update`
- **Card:** `{{ $json["id"] }}`
- **Add Field → List:** set to your **Done** list *(use dropdown or List ID expression)*

> You can also add a **Comment** like “Issue closed on GitHub” using **Trello → Comment → Create**.

---

## Optional: sync comments from GitHub to Trello
- **Switch** branch: `action == "created"` on the **issue_comment** event
- **Find card** (same method as above)
- **Trello → Comment → Create**:  
  ```
  💬 GitHub comment by {{ $json["body"]["comment"]["user"]["login"] }}:
  {{ $json["body"]["comment"]["body"] }}

  Link: {{ $json["body"]["comment"]["html_url"] }}
  ```

---

## Getting List IDs quickly

If you need IDs instead of dropdowns:

1. **Trello → Board → Get Lists** (pick the board)  
2. Copy the `id` of **To Do**, **Doing**, **Done**.

Or directly from Trello URLs using a browser extension/DevTools, but the n8n node is easiest.

---

## JSON field cheat‑sheet (from your payload)

From the GitHub webhook payload (as you pasted), n8n shows it under `body`. Useful paths:

- `action` → `$json["body"]["action"]`
- `issue.number` → `$json["body"]["issue"]["number"]`
- `issue.title` → `$json["body"]["issue"]["title"]`
- `issue.body` → `$json["body"]["issue"]["body"]`
- `issue.html_url` → `$json["body"]["issue"]["html_url"]`
- `issue.state` → `$json["body"]["issue"]["state"]`

We copied these into flat fields in the **Edit Fields** node.

---

## Minimal working rule set (copy/paste)

**Card name (Create / Update):**
```
[#{{ $json["issue_number"] }}] {{ $json["title"] }}
```

**Card description (Create / Update):**
```
{{ $json["description"] }}

---
Source: {{ $json["html_url"] }}
Issue ID: {{ $json["issue_id"] }}
```

**IF filter to match a card to this issue (on items from “Get All Cards”):**
- Value 1: `{{ $json["name"] }}`
- Operation: `contains`
- Value 2: `[#{{ $node["Edit Fields"].json["issue_number"] }}]`

**Move to Done (Update Card → List):** choose your **Done** list (or ID).

---

## Troubleshooting

- **Switch never matches?** Check the **Edit Fields** output; confirm `action` is set. It should be one of `opened`, `reopened`, `edited`, `closed`.
- **Update Card asks to “Select a Card”?** Use **Use expression → `{{ $json["id"] }}`** from the card you found in the **Get All Cards** + **IF** step.
- **Didn’t find the card?** Ensure card names always include the `[#<issue_number>]` prefix on creation. If cards can live in multiple lists, search those lists too **or** use the **Data Store mapping** approach.
- **Labels not syncing?** Trello labels require **label IDs**. Create a small **Set** node with a mapping `{ "bug": "<trelloLabelId>" }` then pass IDs to the Trello node.
- **Rate limits:** If you get 429s, add a short **Wait** before Trello calls or a **Rate Limit** node.

---

## Suggested node names (so your canvas matches this doc)

- `Github Trigger`
- `Edit Fields`
- `Switch` (mode: Rules)
- `Opened / Reopened issues` (Trello: Create Card)
- `Get cards - Doing` (Trello: List → Get All Cards)
- `Match issue → card` (IF)
- `Update card (edited)` (Trello: Update)
- `Move to Done` (Trello: Update, set List = Done)

---

## Done ✅

You now have a clean one‑way sync:
- **Open/Reopen** → Create card
- **Edit** → Update card
- **Close** → Move card to **Done**

If you want a two‑way sync (moving a Trello card to **Done** closes the GitHub issue), create a second workflow with **Trello Trigger → GitHub (HTTP Request) → Close Issue** using the issue number extracted from the card name.
