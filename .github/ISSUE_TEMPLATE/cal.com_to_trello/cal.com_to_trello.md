# n8n Workflow: Cal.com → Switch → Trello (Created / Rescheduled / Cancelled)

> **Workflow name:** Sai  
> **Status:** Active ✅  
> **Editor view:** Cal.com Trigger → Switch (Rules) → 3 Trello “create card” nodes (Created, Rescheduled, Cancelled)

---

## 1) What this workflow does

When a booking happens in **Cal.com**, the webhook payload is received by n8n.  
A **Switch** node checks the booking **event type** and routes the item to one of three Trello nodes:

- **Created** → Creates a card on a Trello list for new bookings.  
- **Rescheduled** → Creates a card on a Trello list for rescheduled bookings.  
- **Cancelled** → Creates a card on a Trello list for cancelled bookings.

This makes it easy to manage appointment lifecycle in Trello using three different lists or pipelines.

---

## 2) Prerequisites

- **Cal.com account** with Webhooks permitted.  
- **Trello board** with up to three lists (e.g., `Created`, `Rescheduled`, `Cancelled`).  
- n8n credentials configured for:
  - **Cal.com** (`Credentials > Cal.com (calApi)`)  
  - **Trello** (`Credentials > Trello API`)  

---

## 3) High‑level flow

```
Cal.com Trigger  ──▶  Switch (by event type)
                         ├──▶ Trello (Created)
                         ├──▶ Trello1 (Rescheduled)
                         └──▶ Trello2 (Cancelled)
```

---

## 4) Node‑by‑node setup (step by step)

### 4.1 Cal.com Trigger
- **Node type:** `n8n-nodes-base.calTrigger`  
- **Recommended Events:**  
  - `BOOKING_CREATED`  
  - `BOOKING_RESCHEDULED`  
  - `BOOKING_CANCELLED`  
- **Notes:** Your payload typically contains fields like:
  - `triggerEvent` (e.g., `BOOKING_CREATED`)  
  - `createdAt`, `startTime`, `endTime`  
  - `attendees[0].name`, `attendees[0].email`, `attendees[0].timeZone`  
  - `additionalNotes` / `description`

> After adding/activating the trigger, copy the webhook URL into Cal.com if you manage webhooks manually; otherwise, let n8n register it automatically when the workflow is active.

---

### 4.2 Switch (mode: Rules)
- **Node type:** `Switch`  
- **Value to compare:** set **Value 1** to the expression:
  ```
  {{ $json.triggerEvent }}
  ```
  (If your payload uses a different field, use that — e.g., `{{ $json.event }}` or `{{ $json.type }}`.)

- **Add three rules** (String → Equals):
  1) `BOOKING_CREATED`  
  2) `BOOKING_RESCHEDULED`  
  3) `BOOKING_CANCELLED`  

- **Connections:**  
  - **Output 1 (Created)** → `Trello` node  
  - **Output 2 (Rescheduled)** → `Trello1` node  
  - **Output 3 (Cancelled)** → `Trello2` node  

---

### 4.3 Trello (Created)
- **Node type:** `n8n-nodes-base.trello` → **Operation:** *Create Card*  
- **List ID:** Trello list for **Created** items (copy from Trello → “... More” on list → `Copy Link` to fetch list ID).  
- **Suggested Card Name template:**
  ```
  Name: {{ $json.attendees[0].name }}
  Booking Date: {{ $json.createdAt }}
  Start: {{ $json.startTime }}  End: {{ $json.endTime }}
  ```
- **Suggested Description template:**
  ```
  Attendee: {{ $json.attendees[0].name }} ({{ $json.attendees[0].email }})
  Time Zone: {{ $json.attendees[0].timeZone }}
  Notes: {{ $json.additionalNotes || $json.description }}
  Event: {{ $json.triggerEvent }}
  ```
- **Optional Additional Fields:**
  - Labels (e.g., “New”)  
  - Due date = `{{ $json.startTime }}`  
  - Members (assign a default member ID)

---

### 4.4 Trello1 (Rescheduled)
- **Node type:** `n8n-nodes-base.trello` → **Operation:** *Create Card*  
- **List ID:** Trello list for **Rescheduled** items.  
- **Suggested Card Name template:**
  ```
  RESCHEDULED: {{ $json.attendees[0].name }}
  New Start: {{ $json.startTime }}  New End: {{ $json.endTime }}
  ```
- **Suggested Description template:**
  ```
  Attendee: {{ $json.attendees[0].name }} ({{ $json.attendees[0].email }})
  Time Zone: {{ $json.attendees[0].timeZone }}
  Notes: {{ $json.additionalNotes || $json.description }}
  Event: {{ $json.triggerEvent }}
  ```
- **Tips:** If your payload includes original times (e.g., `oldStartTime`, `oldEndTime`), add them to the description for context.

---

### 4.5 Trello2 (Cancelled)
- **Node type:** `n8n-nodes-base.trello` → **Operation:** *Create Card*  
- **List ID:** Trello list for **Cancelled** items.  
- **Suggested Card Name template:**
  ```
  CANCELLED: {{ $json.attendees[0].name }}
  Originally: {{ $json.startTime }} - {{ $json.endTime }}
  ```
- **Suggested Description template:**
  ```
  Attendee: {{ $json.attendees[0].name }} ({{ $json.attendees[0].email }})
  Time Zone: {{ $json.attendees[0].timeZone }}
  Reason/Notes: {{ $json.additionalNotes || $json.description }}
  Event: {{ $json.triggerEvent }}
  ```

---

## 5) Testing the workflow

1. **Activate** the workflow in n8n.  
2. In Cal.com, **create**, **reschedule**, and **cancel** a test booking.  
3. Confirm the **Switch** node routes to the expected output (Created/Rescheduled/Cancelled).  
4. Check your Trello board for new cards in the correct lists.  

> Use the **Executions** tab in n8n to inspect incoming payloads and debug mappings.

---

## 6) Field mapping quick reference

| Purpose        | Expression example |
|----------------|--------------------|
| Event          | `{{ $json.triggerEvent }}` |
| Created at     | `{{ $json.createdAt }}` |
| Start time     | `{{ $json.startTime }}` |
| End time       | `{{ $json.endTime }}` |
| Name           | `{{ $json.attendees[0].name }}` |
| Email          | `{{ $json.attendees[0].email }}` |
| Time zone      | `{{ $json.attendees[0].timeZone }}` |
| Notes/Desc     | `{{ $json.additionalNotes || $json.description }}` |

> Use whatever keys exist in **your** payload; the above are common from Cal.com.

---

## 7) Optional enhancements

- **Auto-archiving**: Add a Trello node to archive cards after the meeting end time.  
- **Notifications**: Add Email/Slack nodes to notify your team on each event type.  
- **Deduplication**: Use a **Set** or **IF** node with a key (e.g., booking `uid`) to avoid duplicate cards.  
- **Link back to booking**: If the payload includes a booking URL/UID, add it to the description.  

---

## 8) Troubleshooting

- **No cards created?** Confirm the webhook is hitting n8n (check Executions).  
- **Wrong branch?** Log `{{ $json.triggerEvent }}` to ensure the Switch rule matches exactly.  
- **Time zone issues?** Normalize to UTC or format via a **Code** node if needed.  
- **Trello auth errors?** Reconnect your Trello credential and verify the List ID.

---

## 9) Visual (from your editor)

> This is the actual layout captured from n8n.

![Workflow Screenshot](./881bc7e4-9399-4941-b44e-e3c3b863a15c.png)

---

## 10) Versioning & maintenance

- Keep a copy of this `.md` in your repo or knowledge base.  
- Note any changes to list IDs, labels, or payload fields when Cal.com updates schemas.  
- Use **workflow tags** and **version IDs** in n8n for traceability.

---

**Author:** Generated from your screenshot and common Cal.com payload fields.  
**Last updated:** (fill date/time when you store this file)
