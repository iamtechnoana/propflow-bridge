# Make.com Scenario Documentation
## PropFlow Bridge – Airtable → Voiceflow Sync

---

## Overview

This scenario automatically syncs new and updated property records from Airtable to the Voiceflow Knowledge Base. It runs every 15 minutes and requires zero manual intervention.

---

## Scenario Settings

| Setting | Value |
|---------|-------|
| Name | PropFlow Bridge – Airtable → Voiceflow Sync |
| Schedule | Every 15 minutes |
| Status | Active |
| Max errors | 3 before deactivation |

---

## Module 1 — Airtable: Watch Records

**Purpose:** Detects new or updated property records in Airtable.

| Field | Value |
|-------|-------|
| Connection | Your Airtable connection |
| Base | PropFlow Bridge – Property DB |
| Table | Properties |
| View | Available Only |
| Trigger field | Last Updated |
| Label field | Property Name |
| Limit | 1 |

**How it works:** Make.com stores a "cursor" after each run — the timestamp of the last processed record. On the next run, it only returns records where `Last Updated` is newer than the cursor.

**Important:** The `Available Only` view filters to Status = Available only. Records with Status = Sold/Rented/Under Offer are excluded from syncing.

---

## Module 2 — Tools: Set Variable

**Purpose:** Formats Airtable data into a reusable variable for the HTTP module.

| Field | Value |
|-------|-------|
| Variable name | KB_Text |
| Variable value | Concatenated property fields |

**Note:** This module is currently a pass-through. The actual data transformation happens in the HTTP module's JSON body. KB_Text can be used in future modules if needed.

---

## Module 3 — HTTP: Make a Request

**Purpose:** Sends the property data to Voiceflow's Knowledge Base API.

### Connection Settings

| Field | Value |
|-------|-------|
| URL | `https://api.voiceflow.com/v1/knowledge-base/docs/upload/table` |
| Method | POST |
| Body content type | application/json |
| Parse response | Yes |

### Headers

| Name | Value | Notes |
|------|-------|-------|
| `authorization` | `VF.DM.your_api_key` | ⚠️ No "Bearer" prefix! |
| `projectID` | `your_project_id` | From Voiceflow Settings → General |

### JSON Body

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

### Field Mapping Reference

| JSON Field | Airtable Source | Make.com Variable |
|-----------|----------------|-------------------|
| name | Property ID | `{{1.\`Property ID\`}}` |
| property_name | Property Name | `{{1.\`Property Name\`}}` |
| neighborhood | Neighborhood | `{{1.Neighborhood}}` |
| city | City | `{{1.City}}` |
| property_type | Property Type | `{{1.\`Property Type\`}}` |
| bedrooms | Bedrooms | `{{1.Bedrooms}}` |
| price | Price (CAD) | `{{1.\`Price (CAD)\`}}` |
| status | Status | `{{1.Status}}` |

### Expected Response (201 Created)

```json
{
  "data": {
    "documentID": "64a1b2c3d4e5f6...",
    "status": {
      "type": "PENDING"
    }
  }
}
```

**Save the `documentID`** — in a future update, this should be written back to Airtable's `Voiceflow Doc ID` field to enable document updates and deletions.

---

## Error Handling

### Common Errors

| Error | Code | Likely Cause | Fix |
|-------|------|-------------|-----|
| Unauthorized | 401 | Bearer prefix in auth header | Remove "Bearer", use raw key |
| Bad Request | 400 | Missing searchableFields or invalid JSON | Check JSON body format |
| Not Found | 404 | Wrong endpoint URL | Verify URL is `/upload/table` |
| Unsupported Media Type | 415 | Wrong body format | Set body type to application/json |

### Debugging Steps

1. Click on the failing module (red triangle)
2. Scroll to **INPUT** section
3. Click `+` next to **Headers** to verify authorization header is present
4. Click `+` next to **Body content** to verify JSON is formatted correctly
5. Check **Operation** section for the exact error message from Voiceflow

---

## Maintenance Guide

### Adding New Fields to the Sync

1. Open the HTTP module
2. Add the new field to the `items` object in the JSON body:
   ```json
   "new_field": "{{1.`Field Name In Airtable`}}"
   ```
3. If the field should be searchable, add it to `searchableFields`:
   ```json
   "searchableFields": ["property_name", "neighborhood", "status", "new_field"]
   ```
4. Save and test with a new Airtable record

### Changing Sync Frequency

1. Click the clock icon at the bottom of the scenario
2. Select desired frequency (15 min, 30 min, hourly, etc.)
3. Note: Lower frequency = more Make.com operations used

### Rotating API Keys

1. In Voiceflow → Settings → API Keys → click `...` → Create secondary key
2. Update the `authorization` header value in Make.com HTTP module
3. Test with Run once
4. If successful, promote secondary key to primary in Voiceflow
5. Delete the old key

---

## Operations Usage Estimate

| Scenario | Operations per Run | Daily (96 runs) |
|----------|-------------------|-----------------|
| No new records | 1 (Airtable check) | 96 |
| 1 new record | 3 (Airtable + Tools + HTTP) | 96–288 |
| 5 new records | 15 | Up to 480 |

Free plan: 1,000 operations/month — sufficient for moderate usage.

---

*Last updated: March 2026 | PropFlow Bridge Project*
