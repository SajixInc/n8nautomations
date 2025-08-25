# n8n Workflow: Cal.com → Switch → Trello (Created / Rescheduled / Cancelled)

**Workflow name:** Sai  
**Status:** Active ✅  

---

## 1. Overview

This workflow automates the flow of booking events from **Cal.com** into **Trello**.  
Depending on whether a booking is **created**, **rescheduled**, or **cancelled**, a card is added to the corresponding Trello list.  

This setup makes it easy to manage the appointment lifecycle directly within Trello.  

---

## 2. Prerequisites

- **Cal.com account** with webhook support enabled.  
- **Trello board** with three lists (e.g., *Created*, *Rescheduled*, *Cancelled*).  
- **n8n credentials** configured for:  
  - Cal.com (API/Webhook integration)  
  - Trello API (API Key + Token)  

---

## 3. Workflow Structure

```
Cal.com Trigger  ──▶  Switch (by event type)
                         ├──▶ Trello (Created)
                         ├──▶ Trello1 (Rescheduled)
                         └──▶ Trello2 (Cancelled)
```

---

## 4. Node Configuration

### 4.1 Cal.com Trigger
- **Node type:** `calTrigger`  
- **Subscribed events:**  
  - `BOOKING_CREATED`  
  - `BOOKING_RESCHEDULED`  
  - `BOOKING_CANCELLED`  
- **Payload fields commonly available:**  
  - `triggerEvent`  
  - `createdAt`, `startTime`, `endTime`  
  - `attendees[0].name`, `attendees[0].email`, `attendees[0].timeZone`  
  - `additionalNotes` / `description`  

---

### 4.2 Switch Node
- **Node type:** `Switch`  
- **Comparison Value:**  
  ```n8n
  {{ $json.triggerEvent }}
  ```  
- **Rules (String Equals):**  
  - `BOOKING_CREATED`  
  - `BOOKING_RESCHEDULED`  
  - `BOOKING_CANCELLED`  
- **Connections:**  
  - **Output 1 → Trello (Created)**  
  - **Output 2 → Trello1 (Rescheduled)**  
  - **Output 3 → Trello2 (Cancelled)**  

---

### 4.3 Trello (Created)
- **Operation:** Create Card  
- **List ID:** List for new bookings.  

**Card Name Example:**  
```n8n
Name: {{ $json.attendees[0].name }}
Booking Date: {{ $json.createdAt }}
Start: {{ $json.startTime }}  End: {{ $json.endTime }}
```  

**Card Description Example:**  
```n8n
Attendee: {{ $json.attendees[0].name }} ({{ $json.attendees[0].email }})
Time Zone: {{ $json.attendees[0].timeZone }}
Notes: {{ $json.additionalNotes || $json.description }}
Event: {{ $json.triggerEvent }}
```  

---

### 4.4 Trello1 (Rescheduled)
- **Operation:** Create Card  
- **List ID:** List for rescheduled bookings.  

**Card Name Example:**  
```n8n
RESCHEDULED: {{ $json.attendees[0].name }}
New Start: {{ $json.startTime }}  New End: {{ $json.endTime }}
```  

**Card Description Example:**  
```n8n
Attendee: {{ $json.attendees[0].name }} ({{ $json.attendees[0].email }})
Time Zone: {{ $json.attendees[0].timeZone }}
Notes: {{ $json.additionalNotes || $json.description }}
Event: {{ $json.triggerEvent }}
```  

---

### 4.5 Trello2 (Cancelled)
- **Operation:** Create Card  
- **List ID:** List for cancelled bookings.  

**Card Name Example:**  
```n8n
CANCELLED: {{ $json.attendees[0].name }}
Originally: {{ $json.startTime }} - {{ $json.endTime }}
```  

**Card Description Example:**  
```n8n
Attendee: {{ $json.attendees[0].name }} ({{ $json.attendees[0].email }})
Time Zone: {{ $json.attendees[0].timeZone }}
Reason/Notes: {{ $json.additionalNotes || $json.description }}
Event: {{ $json.triggerEvent }}
```  

---

## 5. Testing
1. Activate the workflow in n8n.  
2. Create, reschedule, and cancel a booking in Cal.com.  
3. Verify that the **Switch node** correctly routes to the expected branch.  
4. Confirm new Trello cards are created in the right lists.  

---

## 6. Field Mapping Quick Reference

| Purpose      | Expression |
|--------------|------------|
| Event        | `{{ $json.triggerEvent }}` |
| Created At   | `{{ $json.createdAt }}` |
| Start Time   | `{{ $json.startTime }}` |
| End Time     | `{{ $json.endTime }}` |
| Name         | `{{ $json.attendees[0].name }}` |
| Email        | `{{ $json.attendees[0].email }}` |
| Time Zone    | `{{ $json.attendees[0].timeZone }}` |
| Notes        | `{{ $json.additionalNotes || $json.description }}` |

---

## 7. Enhancements (Optional)
- **Auto-archiving:** Archive cards after the meeting ends.  
- **Notifications:** Send Slack/Email alerts on each booking event.  
- **Deduplication:** Use booking UID to prevent duplicate Trello cards.  
- **Booking link:** Add booking URL from payload if available.  

---

## 8. Troubleshooting
- ❌ No cards created? → Check n8n Executions for webhook payload.  
- ❌ Wrong branch triggered? → Confirm `{{ $json.triggerEvent }}` matches the Switch rule values.  
- ❌ Timezone mismatch? → Normalize to UTC or use a Code node for formatting.  
- ❌ Trello auth issues? → Refresh Trello credentials and verify List IDs.  

---

## 9. Maintenance
- Keep this file in your repo for documentation.  
- Update list IDs or payload field names if Cal.com schema changes.  
- Use **n8n workflow versioning** for rollback and history.  

---


