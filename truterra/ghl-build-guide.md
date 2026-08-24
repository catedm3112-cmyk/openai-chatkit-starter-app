# TruTerra GHL Build Guide

**Location ID:** `EHl75N7YlN7nOMP30CYm`  
**Branch:** `claude/build-truterra-ghl-forms-xmPsS`  
**Date:** 2026-08-24

> **Security Override — Permanent:** All workflows use **internal GHL in-app notification only**. No SMS or outbound text messages to any phone number. Hard requirement.

---

## Blockers

| # | Issue | Root Cause | Fix |
|---|-------|-----------|-----|
| 1 | Forms cannot be created via API | GHL IAM blocks `POST /forms/` for Private Integration keys | Build all 4 forms manually in GHL UI: Sites → Forms → New Form |
| 2 | Workflows cannot be created via API | `ghl_create_workflow` requires `GHL_REFRESH_TOKEN` (v2 JWT) — currently absent from skill env | Extract token from GHL browser session, add to `.env`, restart MCP |

---

## Credential Unlock (Workflow Builder)

File to edit: `/Users/jakeshore/.clawdbot/workspace/skills/ghl-workflow-builder/.env`

Add:
```
GHL_REFRESH_TOKEN=<paste refreshJwt from GHL browser session>
```

### How to get `refreshJwt`
1. Open Chrome → GHL (logged in to TruTerra sub-account)
2. DevTools (F12) → Network tab → filter by `auth/refresh`
3. Reload or navigate in GHL
4. Click the POST to `services.leadconnectorhq.com/auth/refresh`
5. Response tab → find `refreshJwt` → copy the full value
6. Paste into `.env` as shown above → restart MCP server

---

## Pipeline Reference

| Pipeline | ID | New Lead Stage ID |
|----------|----|-----------------|
| Buyer Pipeline | `q1LsHftvluT7thkSyZrH` | `8788325f-d60e-4475-ac9a-b9c14884cb2f` |
| Seller Pipeline | `zwaDzNE5FxCZfPRvNS4l` | `f6963f0b-00c3-42c9-8c28-9b9b4aee1727` |

| Form | Pipeline |
|------|----------|
| Contact Us | Buyer |
| Seller | Seller |
| Investor | Buyer |
| Partnership | Buyer |

---

## Forms — Build Specs

### Form 1: Contact Us (`/contact`)

**GHL Name:** `TruTerra - Contact Us`  
**Submit button:** "Send Message"  
**Workflow trigger:** `contact-inquiry` tag → Buyer pipeline

| Field | Type | Required |
|-------|------|----------|
| First Name | Text (First Name) | Yes |
| Last Name | Text (Last Name) | Yes |
| Email | Email | Yes |
| Phone | Phone | Yes |
| Message | Textarea | No |

---

### Form 2: Seller (`/seller`)

**GHL Name:** `TruTerra - Seller: Land Evaluation Request`  
**Submit button:** "Request Evaluation"  
**Workflow trigger:** `seller-lead` tag → Seller pipeline

| Field | Type | Required | Custom Field Key |
|-------|------|----------|------------------|
| First Name | Text | Yes | — |
| Last Name | Text | Yes | — |
| Email | Email | Yes | — |
| Phone | Phone | Yes | — |
| Property Address or APN | Text | Yes | `property_address_or_apn` |
| Acreage (approx.) | Text/Number | No | `acreage` |
| County / State | Text | No | `county_state` |
| Asking Price (if known) | Text | No | `asking_price` |
| Additional Notes | Textarea | No | — |

---

### Form 3: Investor (`/investor`)

**GHL Name:** `TruTerra - Investor: Opportunity Access`  
**Submit button:** "Request Access"  
**Workflow trigger:** `investor-lead` tag → Buyer pipeline

| Field | Type | Required | Custom Field Key / Options |
|-------|------|----------|--------------------------|
| First Name | Text | Yes | — |
| Last Name | Text | Yes | — |
| Email | Email | Yes | — |
| Phone | Phone | Yes | — |
| Anticipated Budget to Invest | Dropdown | Yes | `anticipated_budget_to_invest` — Under $100K / $100K–$500K / $500K–$1M / $1M+ |
| Target Asset Type | Dropdown | Yes | `target_asset` — Farmland / Timberland / Rural Residential / Mixed-Use / Other |
| Target Geography | Text | No | State(s) or region |
| Additional Notes | Textarea | No | — |

---

### Form 4: Partnership (`/partnership`)

**GHL Name:** `TruTerra - Partnership: Request Access`  
**Submit button:** "Submit Request"  
**Workflow trigger:** `partner-lead` tag → Buyer pipeline

| Field | Type | Required | Custom Field Key / Options |
|-------|------|----------|--------------------------|
| First Name | Text | Yes | — |
| Last Name | Text | Yes | — |
| Email | Email | Yes | — |
| Phone | Phone | Yes | — |
| Company / Organization | Company Name | No | Standard GHL field |
| Partner Type | Dropdown | Yes | `partner_type` — Broker / Lender / Developer / Conservation Org / Other |
| Capital Available | Dropdown | No | `capital_available` — Under $500K / $500K–$2M / $2M–$10M / $10M+ |
| Partnership Description | Textarea | Yes | — |

---

## Workflows — Action Specs

Each workflow: **Tag → Opportunity → Internal In-App Notification**  
Set trigger = `form_submission` pointing to the form ID after building the form.

### Workflow 1: TruTerra - Contact Us Lead

```
Action 1: Add Contact Tag
  tags: ["contact-inquiry"]

Action 2: Create Opportunity
  pipelineId: q1LsHftvluT7thkSyZrH
  pipelineStageId: 8788325f-d60e-4475-ac9a-b9c14884cb2f
  opportunityName: {{contact.full_name}} - Contact Inquiry
  status: open

Action 3: Internal In-App Notification
  message: New Contact Us inquiry from {{contact.full_name}} ({{contact.email}} / {{contact.phone}}). Check GHL for details.
```

---

### Workflow 2: TruTerra - Seller: Land Evaluation Request

```
Action 1: Add Contact Tag
  tags: ["seller-lead"]

Action 2: Create Opportunity
  pipelineId: zwaDzNE5FxCZfPRvNS4l
  pipelineStageId: f6963f0b-00c3-42c9-8c28-9b9b4aee1727
  opportunityName: {{contact.full_name}} - Seller Lead
  status: open

Action 3: Internal In-App Notification
  message: New Seller lead from {{contact.full_name}} ({{contact.email}} / {{contact.phone}}). Property: {{contact.property_address_or_apn}}. Check GHL for full submission.
```

---

### Workflow 3: TruTerra - Investor: Opportunity Access

```
Action 1: Add Contact Tag
  tags: ["investor-lead"]

Action 2: Create Opportunity
  pipelineId: q1LsHftvluT7thkSyZrH
  pipelineStageId: 8788325f-d60e-4475-ac9a-b9c14884cb2f
  opportunityName: {{contact.full_name}} - Investor Lead
  status: open

Action 3: Internal In-App Notification
  message: New Investor inquiry from {{contact.full_name}} ({{contact.email}} / {{contact.phone}}). Budget: {{contact.anticipated_budget_to_invest}}. Target: {{contact.target_asset}}. Check GHL for details.
```

---

### Workflow 4: TruTerra - Partnership: Request Access

```
Action 1: Add Contact Tag
  tags: ["partner-lead"]

Action 2: Create Opportunity
  pipelineId: q1LsHftvluT7thkSyZrH
  pipelineStageId: 8788325f-d60e-4475-ac9a-b9c14884cb2f
  opportunityName: {{contact.full_name}} - Partnership Inquiry
  status: open

Action 3: Internal In-App Notification
  message: New Partnership inquiry from {{contact.full_name}} ({{contact.email}} / {{contact.phone}}). Partner type: {{contact.partner_type}}. Capital: {{contact.capital_available}}. Check GHL for details.
```

---

## Embed Code Template

After building each form in GHL (Sites → Forms → form → Integrate → Embed), the code looks like:

```html
<script src="https://link.msgsndr.com/js/form_embed.js"></script>
<iframe
  src="https://api.leadconnectorhq.com/widget/form/FORM_ID"
  style="width:100%;height:auto;border:none;"
  id="inline-FORM_ID"
  data-layout="{'id':'FORM_ID'}"
  data-trigger-type="alwaysShow"
  data-trigger-value=""
  data-activation-type="alwaysActivated"
  data-activation-value=""
  data-deactivation-type="neverDeactivate"
  data-deactivation-value=""
  data-form-name="TruTerra Form"
  data-height="600"
  data-layout-iframe-id="inline-FORM_ID"
  data-form-id="FORM_ID"
  title="TruTerra Form"
></iframe>
```

### Page → Form → Embed Mapping

| Page | Form Name in GHL | Form ID (fill after build) |
|------|-----------------|---------------------------|
| `/contact` | TruTerra - Contact Us | \[ paste here \] |
| `/seller` | TruTerra - Seller: Land Evaluation Request | \[ paste here \] |
| `/investor` | TruTerra - Investor: Opportunity Access | \[ paste here \] |
| `/partnership` | TruTerra - Partnership: Request Access | \[ paste here \] |

---

## Tags (auto-created on first use in GHL)

- `contact-inquiry`
- `seller-lead`
- `investor-lead`
- `partner-lead`
