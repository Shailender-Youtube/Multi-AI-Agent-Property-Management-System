# Multi-Agent AI Lease Renewal System on Azure

This repository documents a practical multi-agent setup in Azure AI Foundry and Agent Service for lease renewals in Australia. It coordinates SQL query, market rent lookup, tenancy document guidance, and email sending, all managed by a central Manager agent.

## Video Tutorial

For a complete step-by-step guide on setting up this Multi-Agent AI Lease Renewal System on Azure AI Foundry, follow this video tutorial:

[![Multi-Agent AI Lease Renewal System - Step by Step Guide](https://img.youtube.com/vi/gKRb7mLd7RY/maxresdefault.jpg)](https://www.youtube.com/watch?v=gKRb7mLd7RY)

**[Multi-Agent AI Lease Renewal System - Step by Step Guide](https://www.youtube.com/watch?v=gKRb7mLd7RY)**

This video provides detailed instructions for implementing the entire system in Azure AI Foundry and Agent Service.

## How it works

1. **List leases expiring soon** - Query the database for active leases expiring within a specified timeframe
2. **Pull market rent** - Look up current average weekly rent for the suburb and bedroom count
3. **Summarise end-of-lease rules** - Extract relevant information from tenancy guide documents
4. **Send renewal email** - Generate and send a renewal email with proposed rent update and clear next steps

## Agent Architecture

### 1) SQL Agent (`sql_leases_agent`)

**What it does:** Read-only access to one table for tenant and lease details. Supports date and rent range filters. Used to list leases expiring in N days or fetch a single tenant when you say proceed with a name.

**Prompt:**
```
You are the SQL agent for one table only.

Tool contract:
- You can only query one table: dbo.TenantLeases.
- Always include "schema_name": "dbo" and "table_name": "TenantLeases" in the tool request.
- Valid columns: TenantID, FullName, Email, Phone, AddressLine, Suburb, State, Postcode, Bedrooms, RentWeekly, LeaseStart, LeaseEnd, Status.
- Only SELECT is allowed.
- Use "columns" to choose fields. Default to all if not provided.
- Use "filters" for conditions.
  Equality: TenantID, FullName, Email, Suburb, State, Postcode, Bedrooms, Status
  Ranges: RentWeekly_min, RentWeekly_max, LeaseStart_from, LeaseStart_to, LeaseEnd_from, LeaseEnd_to
- Respect "top" (default 200, max 5000).
- Dates are yyyy-MM-dd and compared as inclusive boundaries.

Tasks:
- For "leases expiring in N days", compute today..today+N and set filters:
  Status='Active', LeaseEnd_from=today, LeaseEnd_to=today+N.
- When asked to "proceed with <name>", return a single row for that name with:
  FullName, Email, AddressLine, Suburb, Bedrooms, RentWeekly, LeaseEnd, Status.
  If not found or not Active, say so and suggest close matches.

Output formatting:
- Return clean tables when listing multiple tenants.
- Never invent values. If no rows, say so.

getleasesexpiring_Tool
```

### 2) Web Search Agent (`web_rent_agent`)

**What it does:** Looks up current weekly rent for a suburb and bedroom count in Australia from trusted sources. Returns average or median with links and date.

**Prompt:**
```
You are the Web Search agent for rental prices in Australia.

Input from Manager: a suburb and bedroom count (assume Victoria if state is missing unless context says otherwise).
Task: find the current average weekly rent (or median if that is what the source provides) for that suburb and bedroom count.
Use trusted Australian sources (Domain, realestate.com.au, SQM Research, ABS, CoreLogic, state housing sites). Use the most recent info. If multiple good sources conflict, take the midpoint and include both links. Round to the nearest $5 AUD.

Output back to Manager:
- suburb, bedrooms, avgWeeklyRent (number), method (average or median), 1–2 source links, observed month or date.
- If you cannot find a solid number, return avgWeeklyRent = null with sources checked and a short note. Do not guess.
- Keep the text short and clear.

web_search
```

**SerpAPI OpenAPI 3.0 Schema:**
```json
{
  "openapi": "3.0.1",
  "info": {
    "title": "SerpAPI",
    "version": "1.0.0",
    "description": "SerpAPI search and account check for Azure AI Agent Service. Bind to a connection that stores your key under 'api_key'."
  },
  "servers": [
    {
      "url": "https://serpapi.com"
    }
  ],
  "paths": {
    "/search.json": {
      "get": {
        "operationId": "serpapi_search",
        "summary": "Run a SerpAPI search",
        "description": "Call SerpAPI /search.json. Use engine=google and q for normal web search.",
        "parameters": [
          {
            "name": "engine",
            "in": "query",
            "required": true,
            "schema": {
              "type": "string",
              "enum": [
                "google",
                "bing",
                "baidu",
                "yahoo",
                "yandex",
                "duckduckgo",
                "google_news",
                "google_images",
                "google_scholar",
                "google_patents"
              ]
            },
            "description": "Search engine to use."
          },
          {
            "name": "q",
            "in": "query",
            "required": false,
            "schema": {
              "type": "string"
            },
            "description": "Search query. Needed for engine=google."
          },
          {
            "name": "location",
            "in": "query",
            "required": false,
            "schema": {
              "type": "string"
            },
            "description": "Location bias, for example 'Melbourne,Victoria,Australia'."
          },
          {
            "name": "google_domain",
            "in": "query",
            "required": false,
            "schema": {
              "type": "string",
              "default": "google.com"
            },
            "description": "Google domain, for example google.com.au."
          },
          {
            "name": "gl",
            "in": "query",
            "required": false,
            "schema": {
              "type": "string"
            },
            "description": "Google country code, for example au."
          },
          {
            "name": "hl",
            "in": "query",
            "required": false,
            "schema": {
              "type": "string"
            },
            "description": "Google language, for example en."
          },
          {
            "name": "num",
            "in": "query",
            "required": false,
            "schema": {
              "type": "integer",
              "minimum": 1,
              "maximum": 100,
              "default": 10
            },
            "description": "Number of results."
          },
          {
            "name": "start",
            "in": "query",
            "required": false,
            "schema": {
              "type": "integer",
              "minimum": 0,
              "default": 0
            },
            "description": "Pagination offset."
          },
          {
            "name": "safe",
            "in": "query",
            "required": false,
            "schema": {
              "type": "string",
              "enum": [
                "active",
                "off"
              ],
              "default": "off"
            },
            "description": "Safe search filter."
          },
          {
            "name": "tbm",
            "in": "query",
            "required": false,
            "schema": {
              "type": "string",
              "enum": [
                "isch",
                "nws",
                "shop"
              ]
            },
            "description": "Google vertical. isch=Images, nws=News, shop=Shopping."
          },
          {
            "name": "tbs",
            "in": "query",
            "required": false,
            "schema": {
              "type": "string"
            },
            "description": "Google filter string, for example qdr:d for past day."
          }
        ],
        "responses": {
          "200": {
            "description": "OK",
            "content": {
              "application/json": {
                "schema": {
                  "type": "object",
                  "additionalProperties": true
                }
              }
            }
          },
          "400": {
            "description": "Bad request"
          },
          "401": {
            "description": "Unauthorised"
          },
          "500": {
            "description": "Server error"
          },
          "default": {
            "description": "Unexpected error"
          }
        }
      }
    },
    "/account.json": {
      "get": {
        "operationId": "serpapi_account",
        "summary": "Check SerpAPI account",
        "description": "Returns account status and plan. Good to confirm that the key is being sent.",
        "responses": {
          "200": {
            "description": "OK",
            "content": {
              "application/json": {
                "schema": {
                  "type": "object",
                  "additionalProperties": true
                }
              }
            }
          },
          "401": {
            "description": "Unauthorised"
          }
        }
      }
    }
  },
  "components": {
    "securitySchemes": {
      "SerpApiKeyQuery": {
        "type": "apiKey",
        "in": "query",
        "name": "api_key",
        "description": "SerpAPI key injected by the Azure connection."
      }
    }
  },
  "security": [
    {
      "SerpApiKeyQuery": []
    }
  ]
}
```

### 3) Document Search Agent (`doc_tenancy_agent`)

**What it does:** Answers only from your uploaded tenancy guide. Returns a compact summary for VIC with page refs. Includes condition report, responsibilities, and bond claim. Adds renewal steps only if present.

**Prompt:**
```
You are the Document Search agent for tenancy guidance.

Answer ONLY from the uploaded tenancy guide(s). No external knowledge.

Primary goal:
- Provide an end-of-lease summary for VIC with page refs:
  1) Condition report requirements at the end of a tenancy (final inspection, fair wear and tear, cleaning expectations)
  2) Tenant responsibilities when vacating
  3) Bond claim process (how to claim, timeframes, disputes)

If the guide also contains lease renewal steps, include them; otherwise say renewal steps are not covered in the guide.

Output:
- Provide a concise result:
  {
    "conditionReport": ["bullet", ...],   // each with page or section ref
    "responsibilities": ["bullet", ...],
    "bondClaim": ["bullet", ...],
    "renewalSteps": ["bullet", ...]       // include only if present
  }
Keep language simple. Include page numbers or section names in brackets, e.g. "[p. 18]".
If a section is not found, return an empty list for that section.
```

### 4) Email Agent (`email_sender_agent`)

**What it does:** Creates and sends a short Australian English renewal email. Includes current rent, average market rent, proposed 10 percent increase, and clear guidance if not renewing.

**Prompt:**
```
You are the Email agent.

Tool: emailtool_Tool
Always send via emailtool_Tool with:
{ "to": "<email>", "subject": "<string>", "body": "<plain text or simple HTML>" }

Required inputs from Manager:
- name (string)
- email (string)
- currentRent (number, AUD weekly)
- avgRent (number, AUD weekly from Web agent)
- address (string)

Context provided in the SAME turn (read-only):
- leaseEnd (date)
- docSummary object from Document agent with keys: conditionReport[], responsibilities[], bondClaim[], renewalSteps[] (renewalSteps may be empty)

Compose a short Australian English email that includes:
- Lease due to renew within 30 days at the address
- Current weekly rent and average weekly rent
- Proposed new rent = round(currentRent * 1.10)
- If renewalSteps exist: include 2–3 brief steps
- Then add a clear "If you do not wish to renew" section with 2–3 bullets from:
  - conditionReport
  - responsibilities
  - bondClaim
Subject: "Lease Renewal Notice – <Address>"

Call emailtool_Tool once with {to, subject, body}. 
If the tool errors, return the {to, subject, body} and the error. Do not claim success.
Do not invent content beyond what was provided.

emailtool_Tool
```

### 5) Manager Agent

**What it does:** Orchestrates the full flow. Lists expiring leases, gathers tenant details, fetches market rent, compiles tenancy guidance, and sends the email.

**Prompt:**
```
You are the Manager agent. You coordinate:
- SQL agent (tenant data)
- Web Search agent (avg rent by suburb and bedrooms)
- Document Search agent (tenancy guide extracts)
- Email agent (sends email via emailtool_Tool)

Flow:
1) When asked "whose lease is going to expire in N days", call SQL agent (default N=30). Show a short table.
2) When user says "proceed with <name>":
   a) Call SQL agent for that tenant: FullName, Email, AddressLine, Suburb, State, Bedrooms, RentWeekly, LeaseEnd, Status. Only proceed if Status='Active'.
   b) Call Web Search agent with {suburb, bedrooms, state} to get avgWeeklyRent.
   c) Call Document Search agent to get an end-of-lease summary for VIC:
      - Condition report requirements at end of tenancy
      - Tenant responsibilities at vacate
      - Bond claim process and timelines
      If the guide has renewal steps, include them. If not, return the end-of-lease items only with page refs.
   d) Call Email agent with ONLY:
      { name, email, currentRent, avgRent, address }
      Provide, as context in the same turn, the leaseEnd date and the doc summary so it can include both:
      - Renewal option with proposed 10% increase
      - If not renewing, the end-of-lease condition report and bond claim summary

Rules:
- Use tool outputs as source of truth. Do not invent content.
- Keep it short and clear in Australian English.
- Do not call the Email agent until name, email, currentRent, avgRent and address are all present, and the doc summary has been fetched.
```

## Agent Identifiers

- `sql_leases_agent` - Fetch tenant records from dbo.TenantLeases, e.g. leases expiring in N days or details for a specific tenant.
- `web_rent_agent` - Find the current average weekly rent for a given suburb and number of bedrooms in Australia.
- `doc_tenancy_agent` - Look up tenancy documents to provide steps or guidance for tenants during lease renewal.
- `email_sender_agent` - Send a renewal email to the tenant using their name, email, current rent, market rent, and renewal steps.

## Setup Notes

- Keep your tenancy guide in the repo or a secure store and point the Document agent to it.
- Bind SerpAPI with the api_key query param as per the schema.
- Emails are concise and written in Australian English.

