# Lead Scraping & AI Cold Caller Appointment Setter – Overview

This document provides a conceptual overview of the private project that powers the lead scraping and AI-driven cold calling / appointment-setting workflows.

It maps directly to the two main diagrams in `diagrams/`:

- `diagram-export-11-28-2025-4_32_18-AMREV.png` – **Lead Pipeline**
- `diagram-export-11-28-2025-5_07_28-AMREV.png` – **AI Caller Flow**

---

## 1. System Goals

- Automate the tedious parts of outbound prospecting
- Help sales teams focus on high-quality conversations
- Maintain clear respect for compliance, opt-outs, and local regulations
- Provide a structure that can be adapted to different industries and offers

---

## 2. End-to-End Lead Pipeline (Lead Pipeline Diagram)

The **Lead Pipeline** diagram shows the overall data and workflow movement:

1. **Lead Ingestion & Scraping**
   - Ingest leads from compliant sources:
     - Form submissions
     - Uploaded CSV lists
     - Partner / referral feeds
     - B2B data providers (where terms allow)
     - Existing CRM exports
   - Normalize and deduplicate records to avoid duplicates and junk.

2. **Enrichment & Scoring**
   - Enrich leads with firmographic / technographic data (where permitted).
   - Score leads against an ideal customer profile.
   - Segment by region, persona, vertical, or campaign.

3. **Storage & Configuration**
   - Store leads, accounts, activities, and campaigns in a central data store.
   - Configuration and routing rules are treated as data (e.g., playbooks, scoring thresholds).

4. **AI Content & Script Engine**
   - Use prompt templates plus lead context to call an LLM (OpenAI or a local model).
   - Generate:
     - Call scripts
     - Email / SMS follow-ups
     - Objection-handling snippets
   - Scripts are personalized to segment, value proposition, and call objective.

5. **Outreach & Calling**
   - Agent UI shows:
     - Lead details
     - AI-generated script
     - Objection tips and context
   - Dialer / telephony integration handles call start, connect, and end.
   - Outcomes and notes are logged as activities.

6. **CRM & Calendar**
   - Syncs qualified leads, opportunities, and activities into CRM.
   - Creates or updates calendar events for booked meetings.
   - Updates deal stages based on call results.

7. **Reporting & Feedback**
   - Aggregates metrics:
     - Call reach rate
     - Conversion rates
     - Script performance by segment
   - Feeds outcomes back into:
     - Lead scoring
     - Segmentation rules
     - Prompt templates and playbooks

---

## 3. AI Caller Flow (Zoomed-In Diagram)

The **AI Caller Flow** diagram zooms into a single call session:

1. **Ready-to-Call Queue**
   - Leads are marked “ready” based on:
     - Score thresholds
     - Timezone and local call window
     - Campaign priorities
   - A queue feeds the calling workflow.

2. **Context Loading**
   - For each call, the system:
     - Loads the lead record
     - Fetches recent activities (emails, SMS, prior calls)
     - Selects an appropriate playbook based on segment and campaign

3. **AI Script Engine**
   - Builds a prompt that includes:
     - Call goal and tone
     - Lead context and segment
   - Calls the LLM to generate:
     - Opening
     - Core value prop
     - Questions / discovery prompts
   - Attaches objection-handling snippets and possible branches.

4. **Agent Call View**
   - Unified view for the agent:
     - Lead details and key facts
     - Call script and guidance
     - Space for live notes and tags
   - Disposition buttons for:
     - Qualified
     - Nurture / follow-up
     - Disqualified

5. **Dialer Session**
   - Auto or manual dialing via integrated telephony.
   - Tracks connect, call duration, and final outcome.
   - Optionally records calls for QA and training (subject to compliance).

6. **Post-Call Automation**
   - Logs a detailed call activity.
   - Creates follow-up tasks (email/SMS/call) where needed.
   - Books or updates calendar meetings for qualified prospects.
   - Updates CRM deal stage and ownership.

7. **Feedback Loop**
   - Updates KPIs: connection rates, meeting-set rates, performance by segment.
   - Evaluates which scripts and prompts perform best.
   - Adjusts:
     - Scoring rules
     - Segmentation
     - Prompt templates and playbooks

---

## 4. Orchestration & Implementation Notes

The system is orchestrated by **workflow tooling** (e.g., n8n):

- Separate workflows for:
  - Lead scraping / ingestion
  - AI calling
  - Qualification and booking
  - Follow-up sequences
  - Reporting and monitoring

The private implementation includes:

- Actual n8n workflows
- Back-end services and integrations
- Configuration and infrastructure code (where applicable)

This showcase repo documents the **architecture and flow** without exposing that implementation detail or any sensitive data.

---

## 5. Relationship to the Private Repo

The private repo:

- Implements the flows described here.
- Contains real workflow definitions, glue code, and configs.
- Integrates with actual providers (CRM, dialer, enrichment APIs, etc.).

This public repo is your **non-sensitive, architecture-only view** that you can safely share externally.

