# PropFlow Bridge — Portfolio Case Study

## Automated Real Estate AI Knowledge Base Integration

**Stack:** Airtable · Make.com · Voiceflow  
**Type:** No-Code Automation / AI Agent Integration  
**Status:** Production Ready ✅

---

## The Problem

A real estate company was using a Voiceflow AI agent to answer buyer queries like *"What properties do you have in Vaughan?"* — but the agent was constantly giving outdated answers.

Every time a new listing was added to their Airtable database, someone had to manually:
1. Export a CSV
2. Log into Voiceflow
3. Delete the old data source
4. Upload the new CSV
5. Wait for reindexing

This process took 30+ minutes, required technical knowledge, and was often skipped — leaving buyers frustrated with an AI that "didn't know" about available properties.

They had tried to automate it using the Voiceflow API but kept hitting a `415 Unsupported Media Type` error they couldn't resolve.

---

## My Approach

I broke the problem into three phases:

**Phase 1 — Understand the failure**  
I read the Voiceflow API documentation carefully and reproduced the 415 error locally. The issue was twofold: the wrong endpoint was being used (`/v1alpha1/public/knowledge-base/document` instead of the correct `/v1/knowledge-base/docs/upload/table`), and the body format was multipart/form-data when JSON was required.

**Phase 2 — Build the data layer**  
I designed a 23-field Airtable schema optimized for both operational use and AI retrieval. Key decisions included adding a `Last Updated` date+time field as the Make.com trigger, a `Voiceflow Doc ID` field for future update/delete operations, and a filtered `Available Only` view to limit sync to relevant records.

**Phase 3 — Wire the automation**  
I built a 3-module Make.com scenario: Airtable Watch Records → Tools Set Variable → HTTP Make a Request. The critical discovery was that Voiceflow's KB API requires the raw `VF.DM.xxx` key in the Authorization header — without the standard `Bearer` prefix that most APIs expect.

---

## Key Technical Challenges

### Challenge 1: The 415 Error
**Problem:** Every POST request returned `415 Unsupported Media Type`.  
**Root Cause:** Wrong endpoint URL + incorrect body format.  
**Solution:** Switched to `/upload/table` endpoint with `application/json` body and `searchableFields` array.

### Challenge 2: Authorization Header
**Problem:** `401 Unauthorized` even with a valid API key.  
**Root Cause:** Voiceflow KB API doesn't follow the standard `Bearer {token}` pattern.  
**Discovery:** The official documentation (https://docs.voiceflow.com/api-reference/authentication) confirms: *"Pass your API key in the Authorization header — `Authorization: VF.DM.your_api_key`"* — no Bearer prefix.  
**Solution:** Removed `Bearer` from the header value.

### Challenge 3: JSON Payload Validation
**Problem:** `400 Bad Request — searchableFields cannot be empty`.  
**Solution:** Added required `searchableFields` array specifying which fields the vector search should index.

### Challenge 4: Special Characters in JSON
**Problem:** `400 Bad Request — bad control character in string literal at position 37`.  
**Root Cause:** Property Description fields contained newlines and special characters that broke JSON serialization.  
**Solution:** Excluded Description and Amenities from the API payload (can be re-added with sanitization in a future iteration).

### Challenge 5: Dynamic Field Mapping
**Problem:** Make.com variable syntax `{{1.Property ID}}` was splitting into two separate tokens due to the space in the field name.  
**Solution:** Used backtick notation: `{{1.\`Property ID\`}}` — the correct Make.com syntax for field names containing spaces.

---

## Results

| Metric | Before | After |
|--------|--------|-------|
| Sync time after new listing | 30+ min (manual) | ~15 min (automated) |
| Human effort required | 5-step manual process | Zero |
| Agent accuracy | Outdated | Real-time |
| 415 Error | Recurring | Resolved |
| Test query success rate | N/A | 100% |

**Live test response:**  
Query: *"What properties do you have available in Maple?"*  
Agent: *"We have 3 available properties: 77 Allegranza Avenue (4-bed detached, $1,299,000), Maple Valley Townhome (3-bed townhouse, $899,000), and Maple Meadows Estate (6-bed, $2,499,000)."*

---

## What I Learned

**No-code doesn't mean no debugging.** The hardest part of this project wasn't writing code — it was reading API documentation carefully, testing hypotheses methodically, and understanding why a header that works in 99% of APIs doesn't work here.

**Document everything.** The 6 different errors I hit during development became the most valuable part of the project — each one taught me something specific about how the Voiceflow API works.

**Schema design matters for AI.** The way data is structured in Airtable directly affects how well the AI agent can answer questions. I spent as much time on field naming, types, and views as I did on the automation itself.

---

## Tech Skills Demonstrated

- REST API integration and debugging (HTTP methods, headers, body formats)
- No-code automation with Make.com (triggers, transforms, HTTP modules)
- Voiceflow Knowledge Base architecture and RAG-based AI agents
- Airtable database design for AI consumption
- Systematic debugging methodology
- Technical documentation

---

## Links

- [GitHub Repository](https://github.com/yourusername/propflow-bridge)
- [Voiceflow KB API Docs](https://docs.voiceflow.com/api-reference/knowledge-base-api/overview)
- [Make.com HTTP Module Docs](https://www.make.com/en/help/tools/http)
