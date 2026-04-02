# PropFlow Bridge

> Automated Real Estate Knowledge Base Sync
> Airtable → Make.com → Voiceflow | No-Code AI Agent Integration

## What It Does

PropFlow Bridge eliminates the manual CSV upload workflow between a real estate property database and a Voiceflow AI agent. When a new property is added or updated in Airtable, Make.com automatically syncs it to Voiceflow's Knowledge Base — keeping the AI agent's responses accurate and up-to-date in real time.

**Before:** Agent gives outdated answers → manual CSV export → manual upload → 30+ min delay
**After:** Agent answers with live data → fully automated → syncs within 15 minutes

## Architecture

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

## Tech Stack

| Tool | Role |
|------|------|
| **Airtable** | Property database (23 fields, 2 views) |
| **Make.com** | Automation trigger + API orchestration |
| **Voiceflow** | AI agent + Knowledge Base (RAG) |

## Airtable Schema

The `Properties` table contains 23 fields:

```
Property ID        │ Text          │ PROP-001 format
Property Name      │ Text          │ Human-readable name
Address            │ Text          │ Full street address
City               │ Single Select │ Vaughan, Toronto, etc.
Neighborhood       │ Single Select │ Maple, Thornhill, etc.
Property Type      │ Single Select │ Detached, Condo, Townhouse
Bedrooms           │ Number        │ 1–6
Bathrooms          │ Number        │ 1–5
Price (CAD)        │ Currency      │ CAD $
Status             │ Single Select │ Available / Sold / Rented / Under Offer
Last Updated       │ Date+Time     │ Make.com trigger field
Voiceflow Doc ID   │ Text          │ Stores returned document ID
```

**Views:**
- `All Properties` — Full database grid view
- `Available Only` — Filtered by Status = Available (used as Make.com trigger view)

## Make.com Scenario

### Modules (in order)

**1. Airtable — Watch Records**
```
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
  authorization: VF.DM.your_api_key
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

## Authentication

**Critical:** Voiceflow Knowledge Base API does NOT use `Bearer` prefix.

```
# Wrong
Authorization: Bearer VF.DM.xxxxx

# Correct
authorization: VF.DM.xxxxx
```

## Test Results

| Test | Query | Result |
|------|-------|--------|
| Neighborhood filter | "Properties in Vaughan?" | Listed 3 properties with price and type |
| New record sync | Added PROP-019 to Airtable | Appeared in Voiceflow KB within 15 min |
| Dynamic data | Multiple records added | Each synced correctly |
| API auth | VF.DM key without Bearer | 201 Created |

### Sample Agent Response
**Query:** "What properties do you have available in Maple?"

**Response:** "We have 3 available properties in Vaughan: 77 Allegranza Avenue (4-bed detached, $1,299,000), 550 Highway 7 E (1-bed condo, $529,000), and 420 Vellore Park Avenue (4-bed detached, $1,499,000)."

## Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `415 Unsupported Media Type` | Wrong endpoint or multipart body | Use `/upload/table` + JSON body |
| `401 Unauthorized` | `Bearer` prefix in auth header | Remove `Bearer`, use raw key |
| `400 Bad Request` | Missing searchableFields | Add `searchableFields` array |
| Field name splits | Spaces in Airtable field names | Use backtick notation |

## Repository Structure

```
propflow-bridge/
├── README.md
├── PropFlow_Bridge_CaseStudy.md   # Portfolio case study
├── Make_Scenario_Documentation.md # Make.com scenario details
└── property_database.csv          # Sample data (12 properties)
```

## Future Improvements

- Add Delete + Re-upload workflow when property status changes
- Store Voiceflow documentID in Airtable for true update operations
- Add error handling router to log failed syncs
- Webhook trigger for sub-1-minute real-time sync
- Multi-city support with metadata filtering
