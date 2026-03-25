# PropFlow Bridge 🏠→🤖

> **Automated Real Estate Knowledge Base Sync**  
> Airtable → Make.com → Voiceflow | No-Code AI Agent Integration

![Status](https://img.shields.io/badge/Status-Production%20Ready-brightgreen)
![Stack](https://img.shields.io/badge/Stack-Airtable%20%7C%20Make.com%20%7C%20Voiceflow-blue)
![Trigger](https://img.shields.io/badge/Trigger-Every%2015%20min-orange)
![License](https://img.shields.io/badge/License-MIT-lightgrey)

---

## 🎯 What is PropFlow Bridge?

PropFlow Bridge eliminates the manual CSV upload workflow between a real estate property database and a Voiceflow AI agent. When a new property is added or updated in Airtable, Make.com automatically syncs it to Voiceflow's Knowledge Base — keeping the AI agent's responses accurate and up-to-date in real time.

**Before:** Agent gives outdated answers → manual CSV export → manual upload → 30+ min delay  
**After:** Agent answers with live data → fully automated → syncs within 15 minutes ✅

---

## 🏗️ Architecture

```
Airtable (Property DB)
        │
        │  Watch Records (Last Updated trigger)
        ▼
Make.com Scenario
        │
        │  Tools → Set Variable (KB_Text)
        │  HTTP → POST /v1/knowledge-base/docs/upload/table
        ▼
Voiceflow Knowledge Base
        │
        │  RAG Retrieval (vector search)
        ▼
AI Agent Response
("We have 3 available properties in Maple...")
```

---

## 🛠️ Tech Stack

| Tool | Role | Plan |
|------|------|------|
| **Airtable** | Property database (23 fields, 2 views) | Free/Pro |
| **Make.com** | Automation trigger + API orchestration | Free (1000 ops/mo) |
| **Voiceflow** | AI agent + Knowledge Base (RAG) | Free ($5 credits) |

---

## 📋 Airtable Schema

The `Properties` table contains 23 fields:

```
Property ID        │ Text          │ PROP-001 format
Property Name      │ Text          │ Human-readable name
Address            │ Text          │ Full street address
City               │ Single Select │ Vaughan, Toronto, etc.
Neighborhood       │ Single Select │ Maple, Thornhill, etc.
Province           │ Text          │ Ontario
Postal Code        │ Text          │ L6A 1A1
Property Type      │ Single Select │ Detached, Condo, Townhouse
Bedrooms           │ Number        │ 1–6
Bathrooms          │ Number        │ 1–5
Parking Spots      │ Number        │ 0–3
Square Footage     │ Number        │ sqft
Price (CAD)        │ Currency      │ CAD $
Status             │ Single Select │ Available / Sold / Rented / Under Offer
Available Date     │ Date          │ Move-in date
Description        │ Long Text     │ Property description
Amenities          │ Long Text     │ Features list
Agent Name         │ Text          │ Listing agent
Agent Phone        │ Text          │ Contact number
Agent Email        │ Text          │ Contact email
Last Updated       │ Date+Time     │ ⚡ Make.com trigger field
Voiceflow Doc ID   │ Text          │ Stores returned document ID
Internal Notes     │ Text          │ Team notes
```

**Views:**
- `All Properties` — Full database grid view
- `Available Only` — Filtered by Status = Available (used as Make.com trigger view)

---

## ⚙️ Make.com Scenario

### Modules (in order)

**1. Airtable — Watch Records**
```
Base:          PropFlow Bridge – Property DB
Table:         Properties
View:          Available Only
Trigger Field: Last Updated
Limit:         1 record per run
```

**2. Tools — Set Variable**
```
Variable Name: KB_Text
Variable Value: [concatenated property fields]
```

**3. HTTP — Make a Request**
```
URL:     https://api.voiceflow.com/v1/knowledge-base/docs/upload/table
Method:  POST
Headers:
  authorization: VF.DM.your_api_key   ← NO "Bearer" prefix!
  projectID:     your_project_id
Body:    application/json
```

### JSON Body Structure
```json
{
  "data": {
    "name": "{{1.`Property ID`}}",
    "searchableFields": ["property_name", "neighborhood", "status"],
    "items": [
      {
        "property_name": "{{1.`Property Name`}}",
        "neighborhood": "{{1.Neighborhood}}",
        "city": "{{1.City}}",
        "property_type": "{{1.`Property Type`}}",
        "bedrooms": "{{1.Bedrooms}}",
        "price": "{{1.`Price (CAD)`}}",
        "status": "{{1.Status}}"
      }
    ]
  }
}
```

---

## 🔑 Authentication

> ⚠️ **Critical:** Voiceflow Knowledge Base API does NOT use `Bearer` prefix.

```
# ❌ Wrong
Authorization: Bearer VF.DM.xxxxx

# ✅ Correct
authorization: VF.DM.xxxxx
```

Your API key is in **Voiceflow → Settings → API Keys** (starts with `VF.DM.`)

Your Project ID is in **Voiceflow → Settings → General → Metadata**

---

## 🧪 Test Results

| Test | Query | Result |
|------|-------|--------|
| Neighborhood filter | "Properties in Vaughan?" | ✅ Listed 3 properties with price & type |
| New record sync | Added PROP-019 to Airtable | ✅ Appeared in Voiceflow KB within 15 min |
| Dynamic data | Multiple records added | ✅ Each synced correctly |
| API auth | VF.DM key without Bearer | ✅ 201 Created |

### Sample Agent Response
**Query:** `"What properties do you have available in Maple?"`

**Response:** *"We have 3 available properties in Vaughan: 77 Allegranza Avenue (4-bed detached, $1,299,000), 550 Highway 7 E (1-bed condo, $529,000), and 420 Vellore Park Avenue (4-bed detached, $1,499,000)."*

---

## 🐛 Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `415 Unsupported Media Type` | Wrong endpoint or multipart body | Use `/upload/table` + JSON body |
| `401 Unauthorized` | `Bearer` prefix in auth header | Remove `Bearer`, use raw key |
| `400 Bad Request — searchableFields empty` | Missing required field | Add `searchableFields` array |
| `400 Bad Request — bad control character` | Special chars in Description | Remove Description from payload |
| `{{1.Property}}{{ID}}` splits | Spaces in field names | Use backtick: `{{1.\`Property ID\`}}` |

---

## 🚀 Setup Guide

### Prerequisites
- Airtable account (free)
- Make.com account (free, 1000 ops/month)
- Voiceflow account (free, $5 credits)

### Step 1 — Airtable
1. Create base: `PropFlow Bridge – Property DB`
2. Create table: `Properties` with schema above
3. Import `property_database.csv` (included in repo)
4. Create view `Available Only` filtered by `Status = Available`

### Step 2 — Voiceflow
1. Create new agent: `PropFlow Bridge – RE Agent`
2. Go to **Knowledge Base** → **Add data source** → **Table**
3. Upload `property_database.csv` with chunking: `Add topic headers`
4. Note your **API Key** and **Project ID** from Settings

### Step 3 — Make.com
1. Create new scenario: `PropFlow Bridge – Airtable → Voiceflow Sync`
2. Add modules in order: Airtable Watch Records → Tools Set Variable → HTTP Make a Request
3. Configure HTTP module with JSON body (see above)
4. Set Authorization header WITHOUT Bearer prefix
5. Enable: **Every 15 minutes**
6. Test: Add a record in Airtable → Run once → Verify in Voiceflow KB

---

## 📁 Repository Structure

```
propflow-bridge/
├── README.md                          ← This file
├── data/
│   └── property_database.csv          ← Sample Airtable data (12 properties)
├── docs/
│   ├── PropFlow_Bridge_Report.docx    ← Full technical report
│   └── PropFlow_Bridge_CaseStudy.md  ← Portfolio case study
└── make/
    └── scenario_blueprint.json        ← Make.com scenario export
```

---

## 🔮 Future Improvements

- [ ] Add `Delete + Re-upload` workflow when property status changes
- [ ] Store Voiceflow `documentID` in Airtable for true update operations
- [ ] Add error handling router to log failed syncs
- [ ] Include Description & Amenities (with character sanitization)
- [ ] Webhook trigger for sub-1-minute real-time sync
- [ ] Multi-city support with metadata filtering

---

## 📄 License

MIT License — free to use, modify, and distribute.

---

*Built with ❤️ as a portfolio project demonstrating no-code AI agent automation.*
