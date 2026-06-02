# 📦 WISMO Order Tracking Automation — n8n Workflow

> **WISMO** = "Where Is My Order?" — A proactive order tracking bot that automatically notifies customers when their shipment status changes, reducing support volume and improving customer experience.

---

## 📋 Table of Contents

- [What This Does](#what-this-does)
- [Workflow Architecture](#workflow-architecture)
- [Prerequisites](#prerequisites)
- [Setup Guide](#setup-guide)
- [Google Sheet Structure](#google-sheet-structure)
- [Node-by-Node Reference](#node-by-node-reference)
- [Channel Setup](#channel-setup)
  - [✅ Email — Ready to Use](#-email--ready-to-use)
  - [⚠️ SMS — DLT Registration Required (India)](#️-sms--dlt-registration-required-india)
  - [⚠️ WhatsApp — Meta Business API Required](#️-whatsapp--meta-business-api-required)
- [Known Limitations](#known-limitations)
- [Security Notes](#security-notes)
- [Status Values Reference](#status-values-reference)

---

## What This Does

Every minute, the workflow:

1. Reads all orders from a Google Sheet
2. Fetches the latest tracking status from TrackingMore API
3. Checks if the status has changed since last notification
4. Routes to the customer's preferred channel — Email, WhatsApp, or SMS
5. Sends a proactive notification to the customer
6. Escalates exceptions to a human via email alert
7. Updates the sheet with new status, timestamp, and notification flag

---

## Workflow Architecture

```
1. Store Event Trigger  (every 1 min)
↓
1b. Read Orders Sheet
↓
2. Get Order Details
↓
3a. Create Tracking     (registers with TrackingMore — safe to re-run, errors ignored)
↓
3b. Get Tracking Status
↓
4. API Success Check
├── FALSE → 9. Escalate To Human  (API failure alert)
└── TRUE
    ↓
    5. Build Notification + Channel Object
    ↓
    5b. Exception Check
    ├── TRUE  → 9. Escalate To Human  (shipment exception)
    └── FALSE
        ↓
        5c. Status Changed?
        ├── FALSE → End  (status unchanged, no duplicate notification sent)
        └── TRUE
            ↓
            6. Channel Router
            ├── email    → 7a. Send Email Notification    → 8. Update Sheet Status
            ├── whatsapp → 7b. Send WhatsApp Notification → 8. Update Sheet Status
            └── sms      → 7c. Send SMS Notification      → 8. Update Sheet Status
```

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| n8n instance | Self-hosted or n8n Cloud |
| Google account | For Sheets (orders) + Gmail (sending + alerts) |
| TrackingMore account | Free tier at trackingmore.com — get API key |
| Email | Gmail OAuth2 credential in n8n — works immediately |
| SMS | 2factor.in account + DLT registration — **7–30 day process** |
| WhatsApp | Meta Developer account + Business verification — **3–7 day process** |

---

## Setup Guide

### Step 1 — Import Workflow

1. Open your n8n instance
2. Go to **Workflows** → Click **+** → **Import from file**
3. Upload `wismo-workflow.json`

### Step 2 — Create Credentials in n8n

Go to **n8n → Credentials → Add credential** and create:

- **Google Sheets OAuth2** — connect your Google account
- **Gmail OAuth2** — connect your Gmail account
- TrackingMore API key goes directly in node 3a and 3b as an HTTP header (not a credential)

### Step 3 — Replace All Placeholders

Open the imported workflow and replace every placeholder below:

| Placeholder | Where | Replace With |
|-------------|-------|-------------|
| `YOUR_GOOGLE_SHEET_ID` | Nodes 1b, 8 | Sheet ID from URL e.g. `1FeB47Tg...` |
| `YOUR_GOOGLE_SHEETS_CREDENTIAL_ID` | Nodes 1b, 8 | Auto-filled after connecting Google Sheets credential |
| `YOUR_TRACKINGMORE_API_KEY` | Nodes 3a, 3b | From trackingmore.com → Settings → API Keys |
| `YOUR_GMAIL_CREDENTIAL_ID` | Nodes 7a, 9 | Auto-filled after connecting Gmail credential |
| `YOUR_ALERT_EMAIL@gmail.com` | Node 9 | Your email address for exception alerts |
| `YOUR_2FACTOR_API_KEY` | Node 7c | From 2factor.in dashboard — needs DLT first |
| `YOUR_DLT_SENDER_ID` | Node 7c | Your approved 6-char DLT sender ID e.g. `DRIZRA` |
| `YOUR_BRAND` | Node 7c | Your business name in the SMS footer |
| `YOUR_META_ACCESS_TOKEN` | Node 7b | From Meta Business → System Users |
| `YOUR_WHATSAPP_PHONE_NUMBER_ID` | Node 7b | From Meta Developer Console → WhatsApp → API Setup |
| `YOUR_APPROVED_TEMPLATE_NAME` | Node 7b | Your approved Meta message template name |

### Step 4 — Manually Set Filter on Node 1b (Important)

> ⚠️ This step is required. The filter cannot be auto-imported — n8n does not support it in JSON for this node version.

In n8n, open **1b. Read Orders Sheet** → go to **Options** → add **Filters**:

- Filter 1: `NOTIFICATION SENT` **does not equal** `Yes`
- Filter 2: `LAST STATUS` **does not equal** `delivered`

This prevents delivered and already-notified orders from hitting the TrackingMore API every minute.

### Step 5 — Configure Google Sheet

See [Google Sheet Structure](#google-sheet-structure) below.

### Step 6 — Activate

Toggle the workflow to **Active**. It runs every minute automatically.

---

## Google Sheet Structure

Create a sheet tab named exactly **Orders** with these column headers (case-sensitive, no extra spaces):

| Column | Type | Notes |
|--------|------|-------|
| `ORDER ID` | Text | Unique ID, e.g. `ORD-1001` |
| `CUSTOMER NAME` | Text | Full name |
| `EMAIL` | Text | Customer email |
| `PHONE` | Text | 10-digit number, no country code, e.g. `9876543210` |
| `TRACKING NUMBER` | Text | Carrier tracking number |
| `CARRIER` | Text | TrackingMore carrier code e.g. `ekart`, `delhivery` |
| `PREFERRED CHANNEL` | Text | Must be exactly `email`, `sms`, or `whatsapp` |
| `LAST STATUS` | Text | Auto-updated by workflow — leave blank on first entry |
| `LAST NOTIFIED AT` | Text | Auto-updated by workflow — leave blank |
| `NOTIFICATION SENT` | Text | Auto-updated to `Yes` after first notification |

**Sample rows:**

| ORDER ID | CUSTOMER NAME | EMAIL | PHONE | TRACKING NUMBER | CARRIER | PREFERRED CHANNEL | LAST STATUS |
|----------|--------------|-------|-------|----------------|---------|-------------------|-------------|
| ORD-1001 | Jagadeesh S | jagadeesh@example.com | 9705104793 | FMPP4043145125 | ekart | email | |
| ORD-1002 | Bhagya T | bhagya@example.com | 9505571733 | 369699442745 | ekart | email | |

> **Tip:** Use `email` as the preferred channel for all rows until SMS and WhatsApp setup is complete.

---

## Node-by-Node Reference

### 1. Store Event Trigger
- **Type:** Schedule Trigger
- **Interval:** Every 1 minute
- **Note:** Increase to 5 or 15 minutes if you have many orders, to stay within TrackingMore API limits

### 1b. Read Orders Sheet
- **Type:** Google Sheets — Get Rows
- **Filter (must be set manually in UI):** Skip `NOTIFICATION SENT = Yes` AND `LAST STATUS = delivered`
- **Why:** Avoids re-calling the API for completed orders on every run

### 2. Get Order Details
- **Type:** Set node
- **What it does:** Maps raw sheet column names to clean variable names used across all downstream nodes
- **Fields created:** `orderId`, `customerName`, `email`, `phone`, `trackingNumber`, `carrier`, `preferredChannel`, `rowNumber`, `lastStatus`, `notificationSent`

### 3a. Create Tracking
- **Type:** HTTP Request — POST
- **Endpoint:** `https://api.trackingmore.com/v4/trackings/create`
- **Purpose:** Registers the tracking number with TrackingMore on first run
- **On duplicate:** Returns an error but `onError: continueRegularOutput` means workflow continues safely — not a problem

### 3b. Get Tracking Status
- **Type:** HTTP Request — GET
- **Endpoint:** `https://api.trackingmore.com/v4/trackings/get?tracking_numbers=...`
- **Returns:** `delivery_status`, tracking checkpoints, carrier info
- **On network error:** `onError: continueErrorOutput` — error output is intentionally unconnected, so failed rows are silently skipped

### 4. API Success Check
- **Type:** IF node
- **Condition:** `meta.code === 200`
- **TRUE →** Proceed to build notification
- **FALSE →** Escalate to human with "API Failure" alert

### 5. Build Notification + Channel Object
- **Type:** Set node (raw JSON mode)
- **Builds these fields for all downstream nodes:**

| Field | Value |
|-------|-------|
| `status` | `delivery_status` from TrackingMore |
| `trackingNumber` | Tracking number from API response |
| `customMessage` | Human-readable status message (see table below) |
| `channel` | From `preferredChannel` in sheet |
| `destination` | Email address if channel=email, phone number otherwise |
| `customerName` | From sheet |
| `orderId` | From sheet |

**Status messages built:**

| API Status | Message Sent |
|-----------|-------------|
| `pending` | Your order has been received and is awaiting carrier updates. |
| `transit` | Your order is on its way! We will keep you updated. |
| `delivered` | Great news! Your order has been delivered. |
| anything else | Your shipment requires attention. |

### 5b. Exception Check
- **Type:** IF node
- **Condition:** `status === "exception"`
- **TRUE →** Escalate to human (shipment problem)
- **FALSE →** Check if status has changed

### 5c. Status Changed?
- **Type:** IF node
- **Condition:** `$json.status` is not equal to `$node["2. Get Order Details"].json["lastStatus"]`
- **TRUE →** Status is new — send notification
- **FALSE →** Status is same as last time — end silently, no duplicate sent
- **Example:** Sheet has `pending`, API returns `transit` → TRUE, notify customer

### 6. Channel Router
- **Type:** Switch node
- **Routes on:** `$json.channel` value
- **Output 0 (email):** → 7a. Send Email Notification
- **Output 1 (whatsapp):** → 7b. Send WhatsApp Notification
- **Output 2 (sms):** → 7c. Send SMS Notification

### 7a. Send Email Notification
- **Type:** Gmail
- **To:** `$json.destination` (customer email)
- **Subject:** `Order {orderId} Update - {status}`
- **Status:** ✅ Working — connects to 8. Update Sheet Status

### 7b. Send WhatsApp Notification
- **Type:** HTTP Request — POST
- **Endpoint:** Meta Cloud API `graph.facebook.com/v19.0/{phone_number_id}/messages`
- **Uses:** Approved template with 4 variables: customerName, orderId, status, trackingNumber
- **Status:** ⚠️ Needs Meta setup — see WhatsApp section below
- **Connects to:** 8. Update Sheet Status on success

### 7c. Send SMS Notification
- **Type:** HTTP Request — POST
- **Endpoint:** `https://2factor.in/API/R1/Bulk/`
- **Status:** ⚠️ Needs DLT registration — see SMS section below
- **Connects to:** 8. Update Sheet Status on success

### 8. Update Sheet Status
- **Type:** Google Sheets — Update Row
- **Matches on:** `ORDER ID`
- **Updates three columns:**
  - `LAST STATUS` → current status from TrackingMore
  - `NOTIFICATION SENT` → `Yes`
  - `LAST NOTIFIED AT` → current timestamp

### 9. Escalate To Human
- **Type:** Gmail
- **To:** `YOUR_ALERT_EMAIL@gmail.com`
- **Triggered by two sources:**
  - Node 4 FALSE output — TrackingMore API returned non-200
  - Node 5b TRUE output — shipment status is `exception`
- **Email contains:** Order ID, Tracking Number, Issue type, Timestamp

---

## Channel Setup

### ✅ Email — Ready to Use

Email works immediately after connecting Gmail OAuth2.

**Steps:**
1. In n8n Credentials → Add → Gmail OAuth2
2. Authorize with your Google account
3. Replace `YOUR_GMAIL_CREDENTIAL_ID` and `YOUR_ALERT_EMAIL@gmail.com` in the workflow
4. Done — node 7a will send customer emails, node 9 will send alert emails

---

### ⚠️ SMS — DLT Registration Required (India)

> This workflow uses **2factor.in** as the SMS gateway. In India, all transactional SMS must go through a **DLT (Distributed Ledger Technology) registered sender** as mandated by TRAI. Without this, every SMS will be blocked by the mobile carrier — it is a government compliance requirement, not a platform restriction.

**Why SMS was not working during development:**
- The 2factor.in API key and sender ID were configured, but the DLT registration was not yet complete
- Without a registered Sender ID and approved Template, SMS is rejected at the carrier level — not at the API level
- 2factor.in accepts the API call but the message never reaches the recipient

**Full DLT registration process (7–30 days total):**

**Step 1 — Register on a DLT portal**

You only need to register on one. Choose any:

| Portal | URL |
|--------|-----|
| Airtel DLT | https://www.airtel.in/business/registration |
| Vodafone Idea DLT | https://www.vodafoneidea.com/dlt |
| BSNL DLT | https://www.ucc-bsnl.co.in |
| Jio DLT | https://trueconnect.jio.com |
| TATA DLT | https://telemarketer.tatateleservices.com |

**Step 2 — Register as a Business Entity**

Documents required:
- GST Certificate
- PAN Card
- Certificate of Incorporation or business registration proof

Approval time: 3–7 business days

**Step 3 — Register your Sender ID (Header)**

- Must be exactly 6 alphabetic characters
- Example: `DRIZRA`, `ORDBOT`, `SHOPUP`
- Must reflect your brand name
- Approval time: 3–7 business days

**Step 4 — Register your SMS Content Template**

- Every message format must be pre-approved
- Use `{#var#}` as placeholder for dynamic values
- The approved template for this workflow:
  ```
  Hello {#var#}, Order {#var#} status: {#var#}. Tracking: {#var#}. Thank you. - YOUR_BRAND
  ```
- Template must match your node 7c `smsText` exactly (same words, same order)
- Approval time: 3–7 business days

**Step 5 — Link DLT to 2factor.in**

- Sign up at https://2factor.in
- Go to Dashboard → DLT Settings
- Enter your DLT Entity ID and Sender ID
- This links your approved registration to the API

**Step 6 — Get 2factor.in API key**

- Go to 2factor.in Dashboard → API Keys → Copy key
- Replace `YOUR_2FACTOR_API_KEY` in node 7c

**Step 7 — Update node 7c with approved values**

| Field | Value |
|-------|-------|
| `apikey` | Your 2factor.in API key |
| `smsFrom` | Your approved DLT Sender ID |
| `smsText` | Must match your approved template exactly |

**Cost:**
- DLT registration: Free
- 2factor.in: ~₹0.15–₹0.20 per SMS (transactional)

---

### ⚠️ WhatsApp — Meta Business API Required

> WhatsApp does not allow sending messages to arbitrary numbers via automation. The **Meta Cloud API** requires business verification, a dedicated phone number, and pre-approved message templates. You cannot use the regular WhatsApp or WhatsApp Business mobile app for this.

**Why WhatsApp was not working during development:**
- The node URL was set to a placeholder (`hi`) and had no valid configuration
- Even with the correct URL, Meta requires business verification before messages can be sent
- All outbound messages must use a pre-approved template — free-form messages are not allowed for business-initiated conversations

**Full Meta WhatsApp API setup process (3–14 days total):**

**Step 1 — Create a Meta Developer account**

- Go to https://developers.facebook.com
- Sign in with a Facebook account
- Accept developer terms

**Step 2 — Create a Meta App**

- Click **My Apps** → **Create App**
- App type: **Business**
- Fill in app name and contact email

**Step 3 — Add WhatsApp to your App**

- In the App Dashboard → **Add Product** → **WhatsApp** → **Set Up**

**Step 4 — Create and Verify a Meta Business account**

- Go to https://business.facebook.com
- Create a Business Portfolio
- Go to **Business Settings** → **Business Info** → **Start Verification**
- Submit: Business registration document, phone number, website
- Verification time: **3–7 business days**

**Step 5 — Get your WhatsApp Phone Number ID**

- In Meta Developer Console → Your App → **WhatsApp** → **API Setup**
- For testing: use the free test number provided by Meta (can only message verified test numbers)
- For production: click **Add Phone Number** and follow the steps to add your business number
- Copy the **Phone Number ID** (a long numeric string)
- Replace `YOUR_WHATSAPP_PHONE_NUMBER_ID` in node 7b URL

**Step 6 — Generate a Permanent Access Token**

- Go to **Business Settings** → **System Users** → **Add**
- Give the system user a name, set role to **Admin**
- Click **Add Assets** → select your WhatsApp app → assign **Full Control**
- Click **Generate New Token** → select your app → select `whatsapp_business_messaging` permission
- Copy the token — it does not expire
- Replace `YOUR_META_ACCESS_TOKEN` in node 7b header

**Step 7 — Create and get a Message Template approved**

- Go to **WhatsApp Manager** (business.facebook.com/wa/manage) → **Message Templates** → **Create Template**
- Category: **Utility** (for order/shipping updates)
- Language: English
- Template body example:
  ```
  Hello {{1}}, your order {{2}} is now *{{3}}*. Tracking number: {{4}}. Thank you for shopping with us!
  ```
  Where:
  - `{{1}}` = customerName
  - `{{2}}` = orderId
  - `{{3}}` = status
  - `{{4}}` = trackingNumber
- Submit for review
- Approval time: a few minutes to 24 hours
- Copy the approved template name
- Replace `YOUR_APPROVED_TEMPLATE_NAME` in node 7b

**Step 8 — Customer Opt-In (Required by Meta Policy)**

- Customers must have explicitly opted in to receive WhatsApp messages from your business
- Add a checkbox to your order form: *"I agree to receive order updates via WhatsApp"*
- Sending to non-opted-in numbers can result in account suspension

**Cost:**
- Meta charges per conversation (24-hour window): ~$0.005–$0.08 depending on country
- India rate: approximately ₹0.40–₹0.80 per conversation

---

## Known Limitations

| Issue | Impact | Workaround |
|-------|--------|-----------|
| Filter on node 1b cannot be imported via JSON | All rows processed every minute until manually set | Set the filter manually in n8n UI after import (Step 4 in setup) |
| `5c. Status Changed?` FALSE branch is silent | No log when status is unchanged | Expected behavior — prevents duplicate notifications |
| TrackingMore free tier rate limits | May fail with 50+ orders | Increase trigger interval to 5–15 min or upgrade plan |
| SMS and WhatsApp need external registration | Cannot use these channels immediately | Set `PREFERRED CHANNEL = email` for all rows during setup period |
| `3b. Get Tracking Status` error output is unconnected | Network errors on TrackingMore silently skip the order | Order will retry on next trigger run |

---

## Security Notes

- **Never push real API keys to GitHub.** All values in this file are placeholders.
- Store credentials in **n8n Credentials**, not as hardcoded values in node parameters.
- The TrackingMore API key in nodes 3a and 3b is in plain JSON — treat it as sensitive, rotate it if accidentally exposed.
- The Meta Access Token in node 7b grants WhatsApp send access — keep it private and never log it.
- The Google Sheet ID is not secret, but restrict sheet access to your service account only.
- The 2factor.in API key controls SMS billing — secure it and monitor usage.

---

## Status Values Reference

| Status | Meaning | Action |
|--------|---------|--------|
| `pending` | Registered, awaiting carrier pickup | Notify customer |
| `inforeceived` | Carrier received shipment info | Notify customer |
| `transit` | Shipment in transit | Notify customer |
| `pickup` | Out for delivery | Notify customer |
| `delivered` | Successfully delivered | Notify customer, mark done |
| `exception` | Problem with shipment | Escalate to human |
| `expired` | Tracking data expired | Escalate to human |
| `undelivered` | Delivery attempt failed | Escalate to human |
| `notfound` | Tracking number not found | Escalate to human |

---

*Built with n8n · TrackingMore · Gmail · 2factor.in · Meta Cloud API*
