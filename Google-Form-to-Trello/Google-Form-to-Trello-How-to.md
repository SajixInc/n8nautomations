# 🔗 End-to-End: Google Form → Google Sheet → n8n → Trello (with Labels)


## Quick Links (Pin These)

- **Trello API Key** → https://trello.com/app-key  
- **Generate Trello Token** (replace `YOUR_API_KEY`) → https://trello.com/1/authorize?expiration=never&name=MyApp&scope=read,write&response_type=token&key=YOUR_API_KEY  
- **Trello API Docs** → https://developer.atlassian.com/cloud/trello/  

- **Google Cloud Console** → https://console.cloud.google.com/  
- **Enable Sheets API** → https://console.cloud.google.com/apis/library/sheets.googleapis.com  
- **Enable Drive API** → https://console.cloud.google.com/apis/library/drive.googleapis.com  
- **Credentials (create Client ID/Secret)** → https://console.cloud.google.com/apis/credentials  
- **OAuth Consent Screen** → https://console.cloud.google.com/apis/credentials/consent  

- **n8n OAuth2 Docs** → https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.oauth2/  
- **n8n Google Sheets Node** → https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/  
- **n8n Trello Node** → https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.trello/  

---

## 0) What You’re Building

- User submits a **Google Form** → data lands in a **Google Sheet**.  
- **n8n** watches the sheet for new rows.  
- For each new row, n8n creates a **Trello card** in a chosen list and applies a **label**.  
- Label is taken from the sheet’s **`LabelName`** column and matched to Trello labels by name (**case-insensitive**).

**Pipeline:**  
Google Form → Google Sheet → **n8n (Google Sheets Trigger)** → **Set** (normalize fields) → **Trello (Get All Labels)** → **Code** (LabelName → LabelId) → **Trello (Create Card)**

---

## 1) Google Form + Google Sheet

1. Create your Google Form → https://docs.google.com/forms  
   Add fields **exactly** (avoid trailing spaces):  
   `First Name, Second Name, Gmail, Phone Number, Date of Birth, Gender, Address, Company, Role, Experiences, LabelName`

2. Link to a Sheet: **Form → Responses → Google Sheets icon → “Create new spreadsheet”.**  
3. Verify headers (Row 1 **must match exactly**):  
   `First Name | Second Name | Gmail | Phone Number | Date of Birth | Gender | Address | Company | Role | Experiences | LabelName`  
4. Submit one test response (creates one data row).

---

## 2) Google Cloud Setup (OAuth for Sheets access)

> ⚠ Without this, n8n cannot read your Google Sheet.

A) Open **Google Cloud Console** → https://console.cloud.google.com/ → **Create Project**.  

B) **Enable APIs**:  
- Sheets API → https://console.cloud.google.com/apis/library/sheets.googleapis.com  
- Drive API → https://console.cloud.google.com/apis/library/drive.googleapis.com  

C) **OAuth consent screen** → https://console.cloud.google.com/apis/credentials/consent  
- User Type: **External**  
- Fill **App name**, **support email**, **developer contact email**  
- **Scopes** (at least):
  - `.../auth/spreadsheets`
  - `.../auth/drive.readonly`
- Add your Google account under **Test users** → Save

D) **Create OAuth Client** → https://console.cloud.google.com/apis/credentials  
- **Application type**:  
  - **Web application** (for n8n Cloud/self-hosted) **or**  
  - **Desktop** (for n8n Desktop)  
- If **Web**: add Authorized Redirect URI **exactly**:  
  `https://YOUR-N8N-DOMAIN/rest/oauth2-credential/callback`  
- If **Desktop** (local n8n default):  
  `http://localhost:5678/rest/oauth2-credential/callback`  
- Copy **Client ID** and **Client Secret**.

E) In **n8n** → **Credentials → Google (OAuth2 / Google Sheets)** → paste **Client ID** & **Client Secret** → **Connect** → sign in → **Allow**.  
Docs: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.oauth2/

---

## 3) Trello Setup (API Key, Token, IDs, Labels)

1) Open **Trello API Key** → https://trello.com/app-key → copy **Key**.  
2) **Generate Token** (requires the key):  
   https://trello.com/1/authorize?expiration=never&name=MyApp&scope=read,write&response_type=token&key=YOUR_API_KEY  
   Replace `YOUR_API_KEY`, authorize, then copy the **Token**.  
3) In **n8n** → **Credentials → Trello** → paste **API Key** & **Token** → Save.  
4) **Board ID** is in the board URL (example: `https://trello.com/b/1GQZ0MgM/my-board` → Board ID = `1GQZ0MgM`).  
5) **List ID**: pick from the node dropdown **or** fetch via API:  
   `https://api.trello.com/1/boards/BOARD_ID/lists?key=KEY&token=TOKEN`  
6) In your Trello board, **create labels** you plan to use (e.g., *Urgent, Candidate, Low*).  
Docs: https://developer.atlassian.com/cloud/trello/

---

## 4) Build the n8n Workflow (node-by-node)

### Node A — Google Sheets Trigger
- **Resource:** Spreadsheet  
- **Operation:** Row Added (or “watch for new rows”)  
- Choose Spreadsheet & Worksheet  
- **Credentials:** your Google credential

### Node B — Edit Fields (Set)
Add these fields (left = key, right = expression):

```
Name        = {{$json["First Name"]}} {{$json["Second Name"]}}
Email       = {{$json["Gmail"]}}
Phone       = {{$json["Phone Number"]}}
DOB         = {{$json["Date of Birth"]}}
Gender      = {{$json["Gender"]}}
Address     = {{$json["Address"]}}
Company     = {{$json["Company"]}}
Role        = {{$json["Role"]}}
Experiences = {{$json["Experiences"]}}
LabelName   = {{$json["LabelName"]}}
```

### Node C — Trello1 (Get All → Label)
- **Resource:** Label  
- **Operation:** Get All  
- **Board:** select your board  
- Output: array of labels → `[{{ id, name, color, ... }}]`

### Node D — Code (match LabelName → LabelId)

```javascript
// Input 0: item from Set (form row)
// Input 1: items from Trello1 (all labels)
const formItem = items[0].json;
const labels = $items(1).map(i => i.json);

const wanted = (formItem.LabelName || '').toLowerCase().trim();
const match = labels.find(l => (l.name || '').toLowerCase().trim() === wanted);

formItem.LabelId = match ? match.id : undefined;

// Optional default:
// const DEFAULT = 'YOUR_DEFAULT_LABEL_ID';
// formItem.LabelId ||= DEFAULT;

return [{ json: formItem }];
```

### Node E — Trello (Create → Card)
- **Board/List:** select the target list  
- **Name (title):**  
  `{{$json["Name"]}} — {{$json["Role"]}}`
- **Description** (example):
```
📧 Email: {{$json["Email"]}}
📞 Phone: {{$json["Phone"]}}
🎂 DOB: {{$json["DOB"]}}
⚧ Gender: {{$json["Gender"]}}
🏠 Address: {{$json["Address"]}}
🏢 Company: {{$json["Company"]}}
💼 Role: {{$json["Role"]}}
🧠 Experiences: {{$json["Experiences"]}}
```
- **Labels (IDs):** `{{$json["LabelId"]}}`

**Connections:**  
`Google Sheets Trigger → Set → Code (main)`  
`Trello1 (Get All) → Code (second input)`  
`Code → Trello (Create Card)`

---

## 5) Test the Pipeline
1) Submit a new entry via the **Google Form**.  
2) Confirm the new row appears in the **Google Sheet**.  
3) In **n8n → Executions**, the run should be green.  
4) Check **Trello**: new card in target list **with the label**.

---

## 6) Troubleshooting (deep fixes)

- **Error 400: redirect_uri_mismatch**  
  Ensure the Authorized redirect URI is **exact**:  
  - Cloud/self-hosted: `https://YOUR-N8N-DOMAIN/rest/oauth2-credential/callback`  
  - Desktop (default): `http://localhost:5678/rest/oauth2-credential/callback`  
  Re-save the OAuth client → reconnect the credential in n8n.

- **Insufficient permissions / can’t list files**  
  Enable Sheets + Drive APIs, add your Google account as a **Test user**, reconnect credential.

- **Trello 401/403**  
  Re-generate Token (read,write), confirm your account has access to the board.

- **No label on card**  
  `LabelName` in the sheet must match a Trello label’s name (case-insensitive).  
  Check the Code node output: `LabelId` should be set.

- **Undefined values**  
  In the Set node, expressions must match the column headers **exactly**. Prefer cleaning headers (no trailing spaces).

---

## 7) Hardening & Best Practices
- Keep all secrets inside **n8n Credentials** (never hard-code).  
- Add a **Processed** column in the sheet; after card creation, write back `Yes` to prevent duplicates.  
- Add an **error branch** to notify you via Email/Slack on failures.  
- Version-control your **n8n workflow JSON** in Git.  
- Optional validation in the Code node (e.g., ensure Email/Phone present).

 

 
