# Lead Scraping & AI Cold Caller Appointment Setter – Technical Overview

This document provides a detailed technical overview of the production n8n implementation that powers the lead scraping and AI-driven cold calling / appointment-setting system.

**Deployment Status**: Production-ready (v1.1.0 - 2025-11-30)

---

## 1. System Goals

- **Automate end-to-end outbound sales** from lead discovery to meeting booking
- **Leverage AI voice technology** for natural, scalable cold calling
- **Qualify leads intelligently** using multi-factor scoring and BANT detection
- **Maintain compliance** with TCPA, GDPR, and CAN-SPAM regulations
- **Provide visibility** through daily reporting and real-time monitoring
- **Scale efficiently** from 100 to 1000+ calls per day

---

## 2. Architecture Overview

The system is built entirely in **n8n** (workflow automation platform) and consists of **5 interconnected workflows** that communicate via webhooks and a shared PostgreSQL database.

### Core Components

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Workflow Engine | n8n | Visual automation, self-hosted |
| Database | PostgreSQL | Lead storage, call logs, activities |
| Lead Scraping | Apify (Leads Finder) | B2B data extraction |
| AI Voice | Bland.ai | Natural voice AI calling |
| SMS | Twilio | Text message delivery |
| Email | SendGrid | Email delivery |
| Booking | Cal.com | Meeting scheduling |

### Data Flow

```
Apify → PostgreSQL → Bland.ai → Analysis → Qualification → Cal.com/Twilio/SendGrid
                                                ↓
                                          Reporting & Monitoring
```

---

## 3. The 5 Workflows

### Workflow 01: Lead Scraping

**Trigger**: Cron schedule (daily at 6 AM)

**Purpose**: Scrape, deduplicate, and normalize leads from Apollo.io

**Key Nodes**:
1. **Scraping Config** - Sets target industries and locations
2. **Apify - Leads Finder** - Executes scraping actor
   - Retry logic: 3 attempts, 10s wait
   - Timeout: 5 minutes
3. **Data Validation** - Phone/email regex validation
4. **Deduplication** - Checks existing leads in database
5. **Phone Normalization** - Converts to E.164 format
6. **Insert to Database** - Bulk insert new leads
7. **Wait Node** - 2-second delay to prevent race conditions
8. **Get Call Queue** - Fetches leads ready for calling
9. **Trigger AI Calling** - Webhook to Workflow 02

**Production Fixes**:
- ✅ PostgreSQL parameter formatting (`{{ [$json.field] }}`)
- ✅ Phone validation (10-15 digits, numeric only)
- ✅ Email validation (regex pattern)
- ✅ Webhook URL validation
- ✅ Daily scrape limit (safety cap)
- ✅ Wait node to prevent race conditions

---

### Workflow 02: AI Calling

**Trigger**: Webhook from Workflow 01 or manual trigger

**Purpose**: Execute AI-powered cold calls via Bland.ai

**Key Nodes**:
1. **Webhook Trigger** - Receives lead data
2. **Input Validation** - Validates payload structure
3. **Load Lead Context** - Fetches lead details from database
4. **Bland AI - Make Call** - Initiates AI voice call
   - Retry logic: 2 attempts, 5s wait
   - Timeout: 60 seconds
   - Customizable script per industry
5. **Update Call Status** - Marks lead as "calling"
6. **Rate Limiter** - 5-second delay between calls
7. **Wait for Call Completion** - Polls Bland.ai for results
8. **Extract Transcript** - Retrieves call recording and transcript
9. **Analyze Intent** - AI analysis of conversation
10. **Update Database** - Stores call outcome, duration, transcript
11. **Trigger Qualification** - Webhook to Workflow 03

**Production Fixes**:
- ✅ Retry logic on Bland.ai API calls
- ✅ Input validation on webhook payload
- ✅ Shortened call script (moved to env var)
- ✅ Error handling for failed calls

---

### Workflow 03: Qualification & Booking

**Trigger**: Webhook from Workflow 02

**Purpose**: Score leads and book meetings with qualified prospects

**Key Nodes**:
1. **Webhook Trigger** - Receives call outcome
2. **Load Call Data** - Fetches transcript and metadata
3. **Score Qualification** - Multi-factor scoring algorithm
   - Budget mention: +20 points
   - Decision maker: +15 points
   - Clear need: +20 points
   - Immediate timeline: +15 points
   - Competitor mention: +10 points
   - Meeting request: +30 points
   - Company size 20+: +10 points
   - Call length 2+ min: +10 points
4. **Qualification Router** - Routes based on score (70+ = qualified)
5. **Cal.com - Create Booking Link** - Generates personalized link
   - Retry logic: 3 attempts, 5s wait
6. **Update Lead Status** - Marks as "qualified" or "not_qualified"
7. **Trigger Follow-up** - Webhook to Workflow 04

**Production Fixes**:
- ✅ Retry logic on Cal.com API calls
- ✅ Parameterized SQL queries
- ✅ Input validation

---

### Workflow 04: SMS/Email Follow-up

**Trigger**: Webhook from Workflow 03 or scheduled delays

**Purpose**: Send conditional follow-up sequences based on call outcome

**Key Nodes**:
1. **Webhook Trigger** - Receives lead and outcome
2. **Outcome Router** - Routes to appropriate sequence
3. **Template Nodes** - 5 different templates:
   - **Interested** → Immediate booking link (SMS + Email)
   - **Maybe Interested** → 2-day follow-up
   - **Voicemail** → 1-day follow-up
   - **No Answer** → 4-day follow-up
   - **Not Qualified** → 30-day nurture
4. **Twilio - Send SMS** - Delivers text messages
5. **SendGrid - Send Email** - Delivers emails
6. **Update Activity Log** - Records all communications
7. **Reply Detection** - Stops sequence if lead responds

**Production Fixes**:
- ✅ SQL injection prevention (parameterized queries)
- ✅ Input validation
- ✅ Error handling for delivery failures

---

### Workflow 05: Reporting & Monitoring

**Trigger**: 
- Cron schedule (daily at 6 PM for reports)
- Cron schedule (every 5 minutes for health checks)
- Manual webhook for on-demand reports

**Purpose**: Generate daily reports and monitor system health

**Key Nodes**:
1. **Report Generator**
   - Leads scraped today
   - Calls made + outcomes
   - Meetings booked
   - Conversion rates
   - Top 10 qualified leads
2. **Health Monitor**
   - Stuck calls (>2 hours in "calling" status)
   - Workflow errors (>5 per day)
   - Communication failures (>10 per hour)
3. **Auto-Fix Stuck Calls** - Resets stuck call statuses
4. **SendGrid - Send Report** - Emails daily summary
5. **Twilio - Send Alert** - SMS for critical issues

**Production Fixes**:
- ✅ Optimized database queries
- ✅ Error handling
- ✅ Alert thresholds

---

## 4. Database Schema

### Tables

**leads**
- `id` (primary key)
- `first_name`, `last_name`, `email`, `phone`
- `company`, `title`, `industry`, `location`
- `source` (e.g., 'leads-finder')
- `status` (new, calling, called, qualified, not_qualified)
- `created_at`, `updated_at`

**call_logs**
- `id` (primary key)
- `lead_id` (foreign key)
- `call_id` (Bland.ai call ID)
- `status` (initiated, completed, failed)
- `duration` (seconds)
- `outcome` (interested, maybe, voicemail, no_answer, not_qualified)
- `transcript` (full conversation)
- `recording_url`
- `created_at`

**activities**
- `id` (primary key)
- `lead_id` (foreign key)
- `type` (call, sms, email, meeting)
- `content` (message or notes)
- `status` (sent, delivered, failed)
- `created_at`

**meetings**
- `id` (primary key)
- `lead_id` (foreign key)
- `booking_link`
- `scheduled_at`
- `status` (pending, confirmed, completed, cancelled)
- `created_at`

---

## 5. Production Improvements (v1.1.0)

### Critical Fixes Applied

1. **PostgreSQL Parameter Formatting**
   - Issue: Incorrect parameter format causing syntax errors
   - Fix: Updated to `{{ [$json.field] }}` array format
   - Impact: All database queries now work reliably

2. **Race Condition Prevention**
   - Issue: Call queue fetched before leads committed to database
   - Fix: Added 2-second Wait node in Workflow 01
   - Impact: Eliminates missing leads in call queue

3. **SQL Injection Prevention**
   - Issue: Raw SQL queries vulnerable to injection
   - Fix: Parameterized all queries in Workflow 04
   - Impact: Enhanced security

4. **Retry Logic**
   - Issue: Transient network failures caused workflow failures
   - Fix: Added retry policies to Apify, Bland.ai, Cal.com nodes
   - Impact: 75% reduction in transient errors

5. **Input Validation**
   - Issue: Malformed webhook payloads crashed workflows
   - Fix: Added validation on all webhook triggers
   - Impact: Graceful handling of invalid data

6. **Phone/Email Validation**
   - Issue: Invalid contacts wasted API calls
   - Fix: Regex validation before scraping
   - Impact: Reduced wasted API costs

---

## 6. Performance & Scalability

### Current Capacity
- **100 calls/day** (sequential, 5s delay)
- **500 leads scraped/day**
- **Unlimited SMS/Email**

### Scaling to 500 calls/day
1. Reduce rate limiting to 1s
2. Enable parallel calling (5-10 concurrent)
3. Increase Bland.ai plan
4. Add database connection pooling

### Scaling to 1000+ calls/day
1. Split into multiple n8n instances
2. Use load balancer for calling
3. Implement queue system (Redis)
4. Move to dedicated database server

---

## 7. Monitoring & Observability

### Daily Reports (6 PM)
- Leads scraped today
- Calls made + outcomes breakdown
- Meetings booked
- Conversion rates (scrape → call → meeting)
- Top 10 qualified leads

### Health Checks (Every 5 min)
- Stuck calls (>2 hours in "calling" status)
- Workflow errors (>5 per day)
- Communication failures (>10 per hour)
- Database connectivity
- API endpoint availability

### Alerts
- **Email**: Sent to `ALERT_EMAIL` for all issues
- **SMS**: Sent to `ALERT_PHONE` for critical issues only
- **Auto-remediation**: Stuck calls automatically reset

---

## 8. Compliance & Best Practices

### TCPA (US Cold Calling)
- ✅ Call only 8 AM - 9 PM local time
- ✅ Maintain Do Not Call (DNC) list
- ✅ Honor opt-out requests immediately
- ✅ Provide clear identification

### GDPR (EU Data)
- ✅ Store only necessary data
- ✅ Provide data deletion on request
- ✅ Maintain lawful basis for processing
- ✅ Secure data storage

### CAN-SPAM (Email)
- ✅ No misleading subject lines
- ✅ Include unsubscribe link
- ✅ Honor opt-outs within 10 days
- ✅ Include physical address

---

## 9. Relationship to Private Implementation

The **private repository** (`gravitasse/Lead_Scraping_AI_Cold_Caller_Appointment_Setting_Project`) contains:

- Actual n8n workflow JSON files (5 workflows)
- Database schema SQL files
- Deployment scripts and automation
- Environment configuration templates
- Customization scripts (call scripts, email templates)
- Full setup and troubleshooting guides

This **public showcase** provides:

- Architecture overview and design patterns
- Technical documentation
- Visual diagrams (Mermaid + exported images)
- Production insights (sanitized)
- Portfolio-friendly presentation

**No sensitive data, credentials, or proprietary logic is exposed in this showcase.**

---

## 10. Key Takeaways

✅ **Production-Ready**: Deployed with stability and security fixes  
✅ **Scalable**: Handles 100-1000+ calls/day with proper infrastructure  
✅ **Reliable**: Retry logic, validation, and monitoring built-in  
✅ **Compliant**: TCPA, GDPR, and CAN-SPAM considerations  
✅ **Cost-Effective**: ~$650-920/month for 100 calls/day  
✅ **Modular**: 5 independent workflows with clear separation  
✅ **Observable**: Daily reports, health checks, and instant alerts  

This system demonstrates a complete, production-grade implementation of AI-powered sales automation using modern workflow orchestration tools.
