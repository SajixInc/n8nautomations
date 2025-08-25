# Telegram → Trello via n8n — Step‑by‑Step Guide

> This doc shows how to connect a Telegram bot to n8n, create Trello cards from messages, and reply back in Telegram with the card details — including labels and due dates.

---

## 1) Prerequisites
- **n8n** (Cloud or self‑hosted)
- **Public HTTPS URL** for n8n (Cloud, Tunnel, or your own domain)
- **Telegram account**
- **Trello account** with a board & list ready

---

## 2) Create your Telegram Bot (BotFather)
1. In Telegram, search **@BotFather** → tap **Start**.
2. Send **/newbot** → give a name and a unique username ending in `bot` (e.g., `SahuTraineeBot`).
3. Copy the **Bot Token** (looks like `123456789:AA...`).
4. (Optional for groups) Using BotFather:
   - **/setprivacy** → choose bot → **Disable** (to read non‑command messages in groups).
   - **/setjoingroups** → **Enable** if you’ll add the bot to groups.

---

## 3) Make n8n reachable (Webhook)
Telegram delivers updates to a **webhook**, so n8n needs a **public HTTPS URL**.

Choose one:
- **n8n Cloud**: you already have a public base URL (e.g., `https://<subdomain>.n8n.cloud/`).
- **Self‑hosted (domain)**: set `WEBHOOK_URL` to your domain with valid HTTPS.
- **Local dev**: use **Tunnel** (built‑in: `n8n tunnel`) or tools like **ngrok/Cloudflare Tunnel**.

> If you were using `localhost`, replace it everywhere with your new public **HTTPS** URL.

---

## 4) Create Credentials in n8n
### Telegram
1. **Credentials** → **New** → search **Telegram** → **Telegram Bot API**.
2. Paste your **Bot Token** → **Save**.

### Trello
1. **Credentials** → **New** → search **Trello**.
2. Provide your Trello **API key & token** → **Save**.

> Tip: If you don’t have Trello key/token yet, Trello’s developer page provides them once you sign in.

---

## 5) Build the Workflow
Create a new workflow with **three nodes**:

1. **Telegram Trigger** (start node)
2. **Trello** (Create Card)
3. **Telegram** (Send Message)

Connect: **Telegram Trigger → Trello → Telegram (Send Message)**

---

## 6) Configure Each Node

### A) Telegram Trigger (Message)
- **Update Types**: check **Message** (and others if needed).
- **Credentials**: select your Telegram credential.
- No expression needed here.

### B) Trello — Create Card
- **Operation**: `Create`
- **Board**: choose your board
- **List**: choose your list
- **Name (Title)**
```n8n
{{ $json["message"]["text"] }}
```
- **Description** (optional but recommended)
```n8n
Created by Telegram user: {{ $json["message"]["from"]["first_name"] || "Unknown" }}
Message ID: {{ $json["message"]["message_id"] }}
Chat Type: {{ $json["message"]["chat"]["type"] }}
Raw Text: {{ $json["message"]["text"] }}
```
- **Additional Fields → Labels** (optional)
  - If your labels already exist on the board, you can hardcode names:
```n8n
["Bug", "High Priority"]
```
  - Or pass label IDs as an array of IDs.
- **Additional Fields → Due** (optional)
  - Set due date **2 days from now**:
```n8n
{{ $now.plus({ days: 2 }).toISO() }}
```

> You can change `2` to any number of days. If you prefer a fixed date, use ISO format like `2025-08-31T17:00:00.000Z`.

### C) Telegram — Send Message (Reply to user)
- **Chat ID**
```n8n
{{ $("Telegram Trigger").item.json.message.chat.id }}
```
- **Text** (clean summary back to the user)
```n8n
✅ Trello card created!

📌 Title: {{ $json["name"] }}
📝 Description: {{ $json["desc"] || "(No description)" }}
🏷️ Labels: {{ ($json["labels"] || []).map(l => l.name).join(", ") || "None" }}
📅 Due: {{ $json["due"] || "None" }}
🔗 Link: {{ $json["shortUrl"] }}
```

> The fields above read from the **output of the Trello node**, so keep the connection `Trello → Telegram (Send Message)`.

---

## 7) Activate & Test
1. **Activate** the workflow (top‑right toggle). n8n will register the Telegram webhook.
2. In Telegram, open your bot and send a message (e.g., `Fix login bug`).
3. Expect:
   - A new Trello card created in your chosen list.
   - A Telegram reply showing Title, Description, Labels, Due, and Card Link.

---

## 8) Optional: Dynamic Labels from Message Text
Add a **Function** (or **IF/Switch**) node between **Telegram Trigger** and **Trello**:

**Function node code (example mapping):**
```n8n
// Map keywords in the message to label names
const text = $json.message.text?.toLowerCase() || "";
const labels = [];
if (text.includes("bug")) labels.push("Bug");
if (text.includes("urgent") || text.includes("high")) labels.push("High Priority");

return [{
  ...$json,
  derived: { labels }
}];
```

Then, in **Trello → Labels** use:
```n8n
{{ $json.derived.labels }}
```

> Ensure those labels exist on your Trello board. If you prefer label **IDs**, map to IDs instead of names.

---

## 9) Troubleshooting
- **Bad Request: chat not found** → Your **Chat ID** is wrong. Use:
```n8n
{{ $("Telegram Trigger").item.json.message.chat.id }}
```
- **409: Conflict – webhook already set** → Clear it, then re‑activate:
```
https://api.telegram.org/bot<YOUR_TOKEN>/deleteWebhook?drop_pending_updates=true
```
- **No executions in n8n** → Your n8n URL isn’t public/HTTPS or the webhook points to an old base URL.
- **Forbidden: bot was blocked by the user** → User must unblock the bot.
- **Labels not applied** → Make sure labels exist on the Trello board (or use correct label IDs).

---

## 10) Example Telegram Reply (What users see)
```
✅ Trello card created!

📌 Title: Fix login bug
📝 Description: Created by Telegram user: Sahu\nMessage ID: 42\nChat Type: private\nRaw Text: Fix login bug
🏷️ Labels: Bug, High Priority
📅 Due: 2025-08-27T10:00:00.000Z
🔗 Link: https://trello.com/c/abcd1234
```

---

## 11) One‑liner Daily Update (optional)
> **Telegram → Trello automation live:** creates cards with title, description, labels & due date, and replies in Telegram with full card details and link.

---

## 12) Notes & Good Practices
- Keep titles short; push details into **Description**.
- Use **commands** like `/todo Buy milk` with a **Switch** node to handle different intents.
- Sanitize/validate inputs before creating cards.
- Prefer **Production** webhooks for always‑on flows (n8n shows Test vs Production URLs in trigger nodes).

---

### Quick Reference — Expressions
- **Card title from Telegram text**
```n8n
{{ $json["message"]["text"] }}
```
- **Due date +2 days**
```n8n
{{ $now.plus({ days: 2 }).toISO() }}
```
- **Chat ID (reply to same chat)**
```n8n
{{ $("Telegram Trigger").item.json.message.chat.id }}
```
- **Labels list joined**
```n8n
{{ ($json["labels"] || []).map(l => l.name).join(", ") || "None" }}
```

---

**Done!** You can now create Trello tasks from Telegram and send professional confirmations back to users. If you’d like, we can extend this to support commands (`/new`, `/status`, `/assign`) or attach files/images to cards.

