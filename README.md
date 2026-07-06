# AI-Powered-Marketing-Agency-Management-System-technical-solution

This document presents a comprehensive technical solution for a marketing agency management system designed to handle 100+ clients with scalability to 1,000+. The architecture leverages modern cloud technologies, AI integration, and robust security practices to create an efficient, scalable platform.

# Recommended Spring Boot Stack

Spring Boot 3.x (with Java 17+)

Spring Data JPA + PostgreSQL/MYSQL

Spring Security + OAuth2 Resource Server

Spring Cache + Redis

Spring Cloud (if microservices – Feign, Circuit Breaker, Config Server)

Spring Batch for heavy data processing

Spring AI (for LLM integrations)

Liquibase for DB migrations

MapStruct for DTO mapping

Lombok for boilerplate reduction

Micrometer + Prometheus + Grafana for monitoring

# 1. System Architecture Diagram

<img width="7008" height="3032" alt="deepseek_mermaid_20260705_79c570" src="https://github.com/user-attachments/assets/e93c363e-4a4f-4c95-b9e0-610bfd456766" />



# 2. Database Design


```mermaid
erDiagram

    CLIENTS {
        bigint client_id PK
        string company_name
        string industry
        string email
        string phone
        decimal monthly_fee
        string status
    }

    TEAM_MEMBERS {
        bigint team_member_id PK
        string full_name
        string email
        string role
        string department
    }

    PROJECTS {
        bigint project_id PK
        bigint client_id FK
        bigint manager_id FK
        string project_name
        decimal budget
        string status
    }

    PROJECT_TEAM {
        bigint mapping_id PK
        bigint project_id FK
        bigint team_member_id FK
        string responsibility
    }

    META_ASSETS {
        bigint meta_asset_id PK
        bigint client_id FK
        string business_manager_id
        string ad_account_id
        string page_id
        string instagram_id
        string pixel_id
    }

    GOOGLE_ASSETS {
        bigint google_asset_id PK
        bigint client_id FK
        string google_ads_id
        string ga4_property_id
        string gtm_container_id
        string search_console_id
    }

    ACCESS_REQUESTS {
        bigint request_id PK
        bigint client_id FK
        string asset_type
        string status
        datetime request_date
    }

    CAMPAIGNS {
        bigint campaign_id PK
        bigint project_id FK
        string platform
        decimal budget
        decimal spend
        string status
    }

    REPORTS {
        bigint report_id PK
        bigint campaign_id FK
        string report_type
        string report_url
    }

    TASKS {
        bigint task_id PK
        bigint project_id FK
        bigint assigned_to FK
        string title
        date due_date
        string status
    }

    DOCUMENTS {
        bigint document_id PK
        bigint client_id FK
        bigint project_id FK
        string file_name
        string file_url
    }

    CREDENTIALS {
        bigint credential_id PK
        bigint client_id FK
        string platform
        string encrypted_secret
    }

    AUDIT_LOGS {
        bigint log_id PK
        bigint user_id FK
        string entity_name
        string action
        datetime created_at
    }

    AI_RECOMMENDATIONS {
        bigint recommendation_id PK
        bigint campaign_id FK
        string recommendation_type
        string recommendation
        float confidence_score
    }

    MEETING_NOTES {
        bigint meeting_id PK
        bigint client_id FK
        text transcript
        text ai_summary
    }

    CLIENTS ||--o{ PROJECTS : owns
    CLIENTS ||--|| META_ASSETS : has
    CLIENTS ||--|| GOOGLE_ASSETS : has
    CLIENTS ||--o{ ACCESS_REQUESTS : creates
    CLIENTS ||--o{ DOCUMENTS : uploads
    CLIENTS ||--o{ CREDENTIALS : stores
    CLIENTS ||--o{ MEETING_NOTES : attends

    TEAM_MEMBERS ||--o{ PROJECT_TEAM : assigned
    PROJECTS ||--o{ PROJECT_TEAM : includes

    PROJECTS ||--o{ CAMPAIGNS : contains
    PROJECTS ||--o{ TASKS : has
    PROJECTS ||--o{ DOCUMENTS : contains

    CAMPAIGNS ||--o{ REPORTS : generates
    CAMPAIGNS ||--o{ AI_RECOMMENDATIONS : receives

    TEAM_MEMBERS ||--o{ TASKS : performs
    TEAM_MEMBERS ||--o{ AUDIT_LOGS : creates
```

# SQL queries with explanations
# 1. Pending Meta Access Requests

SELECT              <br>            
    ar.request_id,  <br>         
    c.company_name,<br>         
    ar.asset_type,<br>         
    ar.request_date,<br>         
    ar.status<br>         
FROM AccessRequests ar<br>         
JOIN Clients c<br>         
    ON ar.client_id = c.client_id<br>         
WHERE ar.asset_type = 'META'<br>         
  AND ar.status = 'PENDING'<br>         
ORDER BY ar.request_date ASC;<br>         

Explanation
<br>         
Retrieves all pending Meta Business access requests.<br>         
Joins the Clients table to display the client name.<br>         
Oldest requests appear first so the team can prioritize them.<br>         

# 2. Overdue Campaigns

SELECT<br>   
    campaign_id,<br>   
    campaign_name,<br>   
    end_date,<br>   
    status<br>   
FROM Campaigns<br>   
WHERE end_date < CURRENT_DATE<br>   
AND status <> 'COMPLETED';<br>   

Explanation <br>
Finds campaigns whose end date has already passed. <br>
Ignores completed campaigns.<br>
Useful for dashboard alerts.<br>

# 3. Team Workload

SELECT <br>
    tm.team_member_id,<br>
    tm.full_name,<br>
    COUNT(t.task_id) AS total_tasks<br>
FROM TeamMembers tm<br>
LEFT JOIN Tasks t<br>
ON tm.team_member_id = t.assigned_to<br>
AND t.status <> 'COMPLETED'<br>
GROUP BY tm.team_member_id, tm.full_name<br>
ORDER BY total_tasks DESC;<br>

Explanation <br>
Counts active tasks for each employee.<br>
Helps managers distribute work evenly.<br>

# 4. Clients Missing GA4 or GTM

SELECT   <br>
    c.client_id,<br>
    c.company_name<br>
FROM Clients c<br>
LEFT JOIN GoogleAssets g<br>
ON c.client_id = g.client_id<br>
WHERE g.ga4_property_id IS NULL<br>
   OR g.gtm_container_id IS NULL;<br>

   Explanation  <br>
Finds clients who have not completed Google Analytics or Google Tag Manager setup.<br>
Useful during onboarding.
<br>

# 5. Monthly Revenue

SELECT   <br>
    YEAR(contract_start) AS year,<br>
    MONTH(contract_start) AS month,<br>
    SUM(monthly_fee) AS total_revenue<br>
FROM Clients<br>
WHERE status = 'ACTIVE'<br>
GROUP BY YEAR(contract_start),<br>
         MONTH(contract_start)<br>
ORDER BY year DESC,<br>
         month DESC;<br>

 Explanation  <br>
Calculates agency revenue based on client retainers.<br>
Groups revenue by month

# 6. Inactive Clients  

SELECT   <br>
    client_id,<br>
    company_name,<br>
    status<br>
FROM Clients<br>
WHERE status = 'INACTIVE';<br>

Explanation  <br>
Lists all inactive clients.<br>
Useful for retention campaigns.<br>

# 7. Highest Spend Campaigns

SELECT   <br>
    campaign_id, <br>
    campaign_name,<br>
    platform,<br>
    spend<br>
FROM Campaigns<br>
ORDER BY spend DESC<br>
LIMIT 10;<br> 

Explanation <br>
Returns the Top 10 campaigns by advertising spend. <br>
Helps identify major campaigns.

# 8. Total Campaign Spend per Client

SELECT   <br>
    c.company_name, <br>
    SUM(cp.spend) AS total_spend  <br>
FROM Clients c <br>
JOIN Projects p <br>
ON c.client_id = p.client_id <br>
JOIN Campaigns cp <br>
ON p.project_id = cp.project_id <br>
GROUP BY c.company_name <br>
ORDER BY total_spend DESC; <br>

Explanation <br>
Calculates total advertising spend for each client.<br>
Useful for client reports.

 # 9. Campaign Performance

 SELECT   <br>
    campaign_name, <br>
    impressions,<br>
    clicks,<br>
    conversions,<br>
    spend<br>
FROM Campaigns<br>
ORDER BY conversions DESC;<br>

Explanation <br>
Shows campaign performance metrics. <br>
Can be used for dashboard analytics.<br>
 
# 10. Pending Tasks

SELECT  <br>
    task_id, <br>
    title,<br>
    due_date,<br>
    status<br>
FROM Tasks<br>
WHERE status = 'PENDING'<br>
ORDER BY due_date;<br>
ExplanationC
Displays pending tasks ordered by due date.<br>
Helps marketing executives prioritize work.


#  Dashboards

Dashboards are built as React frontends consuming REST APIs from Spring Boot.

 # 1.CEO
 KPIs - Total revenue (trend), active client count, client churn, new clients, average revenue per client, team utilisation, top 5 campaigns, client satisfaction score

 # 2.Account Manager
 KPIs -  Portfolio size, revenue per client, client health scores, upcoming deliverables, pending approvals, task completion rate, meeting schedule

 # 3.Marketing Executive
 KPIs  -  Active campaigns, pending launches, daily spend, CPA trends, top creative performance, A/B test results, optimisation suggestions, daily checklist
 


# API Design

<img width="3850" height="1603" alt="deepseek_mermaid_20260705_7b306f" src="https://github.com/user-attachments/assets/3669e70b-b2a0-4cd7-a09a-158e87746a43" />

# Client Management
GET /api/v1/clients             <br> 
GET /api/v1/clients/{id}<br> 
POST /api/v1/clients<br> 
PUT /api/v1/clients/{id}<br> 
DELETE /api/v1/clients/{id}<br> 
GET /api/v1/clients/{id}/campaigns<br> 

# Campaign Management
GET /api/v1/campaigns  <br> 
GET /api/v1/campaigns/{id}<br> 
POST /api/v1/campaigns<br> 
PUT /api/v1/campaigns/{id}<br> 
PATCH /api/v1/campaigns/{id}/status<br> 
POST /api/v1/campaigns/{id}/launch<br> 
DELETE /api/v1/campaigns/{id}<br> 

# Asset Management
GET /api/v1/assets/meta<br> 
GET /api/v1/assets/google<br> 
POST /api/v1/assets/meta<br> 
POST /api/v1/assets/google<br> 
GET /api/v1/assets/meta/{id}/performance<br> 

# Reporting
GET /api/v1/reports<br> 
POST /api/v1/reports/generate<br> 
GET /api/v1/reports/{id}<br> 
GET /api/v1/reports/{id}/download<br> 
GET /api/v1/dashboard/ceo<br> 
GET /api/v1/dashboard/manager<br> 
GET /api/v1/dashboard/executive<br> 

# Analytics
GET /api/v1/analytics/campaign/{id}/metrics<br> 
GET /api/v1/analytics/roi<br> 
GET /api/v1/analytics/forecast<br> 
POST /api/v1/analytics/optimization-suggestions<br> 

# AI Integration
POST /api/v1/ai/summarize<br> 
POST /api/v1/ai/generate-report<br> 
POST /api/v1/ai/optimization<br> 
POST /api/v1/ai/ad-copy<br> 
POST /api/v1/ai/analyze-performance<br> 

Authentication & Authorisation: OAuth2 + JWT, with @PreAuthorize on methods.  <br>
Validation: @Valid with Bean Validation 2.0.  <br>
Logging: MDC for request tracing; JSON logs to ELK.   <br>
Rate Limiting: Resilience4j RateLimiter filter backed by Redis. <br>
Error Handling: Global @ControllerAdvice returning standardised error responses. <br>

# AI Integration

<img width="2611" height="1432" alt="deepseek_mermaid_20260706_a74632" src="https://github.com/user-attachments/assets/7578a415-1cf8-450c-95ac-47b09fd2c23d" />

 
We leverage Spring AI to unify access to LLMs (OpenAI, Anthropic, Azure) and embedding models. <br>

Use Cases:  <br>

Meeting Summarisation – call ChatClient with a prompt to extract action items and decisions. <br>

Report Generation – combine performance data with natural language insights; prompt includes data as JSON. <br>

Optimisation Recommendations – feed campaign metrics and ask for bid adjustments, audience refinements, etc. <br>

Ad Copy Creation – use PromptTemplate with product/audience variables to generate variations.
<br>
Campaign Performance Analysis – detect trends, anomalies, and benchmarks automatically.
<br>
All AI calls are asynchronous for long‑running tasks, with retry and fallback strategies.
<br>

# MCP (Model Context Protocol)

MCP is a protocol that standardises how AI assistants request and receive structured context from external systems. Unlike REST, which is resource‑oriented, MCP is query‑driven and returns data that an LLM can directly consume, often with metadata and tool definitions. 

# MCP vs REST

Aspect	REST API	MCP
Purpose	CRUD operations, frontend data	Context provisioning for AI models
Endpoint design	Multiple resources with standard verbs	Single /context endpoint with dynamic intent parsing
Request format	Path/query parameters + body	Natural language query + optional filters
Response format	Entity DTOs	Structured context (JSON with LLM‑friendly schema)
Security	OAuth2 scopes, RBAC	Same, plus client‑scope filtering
Use cases	UI interactions, integrations	AI agents, RAG, automated insights

# Secure AI Assistant Query Flow (MCP Implementation in Spring)

 * AI Assistant (our internal agent or third‑party) sends a POST /mcp/v1/context request with a JWT token and a natural language query. <br>

* MCP Controller validates the JWT using Spring Security OAuth2. <br>

* The IntentParser (powered by a lightweight LLM) extracts the required data types (e.g., campaign performance, client list) and any filters (client name, date range). <br>

* Permission enforcement: AuthorizationService ensures the user has access to the requested clients (based on RBAC and client‑scope claims). If not, 403 is returned.<br>

* Data retrieval: Service layer fetches data from PostgreSQL (with RLS) and Redis cache.<br>

* Context builder formats the data into a standardised MCP JSON structure (includes metadata like source, timestamp).<br>

* Audit logging: asynchronously logs the request (user, query, data accessed).<br>

* Response is sent back to the AI Assistant, which then synthesises a natural language answer for the user.<br>

All communication is over TLS 1.3, and rate limiting is applied to prevent abuse.<br>

<img width="3849" height="2148" alt="deepseek_mermaid_20260706_3a35be" src="https://github.com/user-attachments/assets/799f30bc-ea3f-4250-84f4-ab6e241e7233" />



 #  RAG (Retrieval‑Augmented Generation)
 
We implement a RAG pipeline using Spring AI and Pinecone (or Redis Stack) as vector store . <br>

# Architecture 

Document Ingestion:  <br>

Load SOPs, marketing docs, client briefs from S3.   <br>

Chunk using TokenTextSplitter (chunk size 500 tokens, overlap 20%).  <br>

Embed with OpenAiEmbeddingModel (or local model). <br>

Store vectors + metadata (document type, version, client‑scope) in Pinecone. <br>

Query Pipeline: <br>

User question → embed → similarity search (top‑k=5). <br>

Retrieve chunks and filter by user’s client scope (metadata filter).  <br>

Build prompt: "Context: ...\nQuestion: ..."  <br> 

Call LLM to generate answer with citations. <br>

Chunking Strategy: Semantic chunking based on headings and paragraphs; overlapping to preserve context across boundaries. <br>

Vector Database: Pinecone (managed, scalable) with index optimised for cosine similarity. <br>



#  Automation

We use a combination of Spring Integration (for core workflows) and n8n (for non‑critical automations).<br>

Core Automations (implemented in Spring): <br>

Campaign Launch: When campaign status changes to APPROVED, a Spring Integration flow pushes to Meta/Google, monitors launch, updates status, and sends notifications. <br>

Performance Alerts: Scheduled job (every hour) checks ROAS/CPA thresholds; if breached, send Slack/email alerts. <br>

Weekly Report Generation: Spring Batch job aggregates performance data, generates PDF, uploads to S3, and emails stakeholders. <br>

Access Request Approval: State machine handles approval workflow with email notifications. <br>

External Automations (via n8n webhooks): <br>

Client onboarding: Send welcome emails, create Slack channels, schedule meetings.<br>

Document management: Sync to Google Drive when new documents are uploaded.<br>

Social media posting: Automate sharing of case studies. <br>

All automations are event‑driven using Spring Cloud Stream with Kafka, ensuring decoupling and scalability.
<br>

#  Workflow Automation with n8n

Automation: Campaign Launch Workflow   <br>
Triggers: <br>
  - Campaign status changes to 'approved' <br>
  - Time-based schedule <br>

Actions:  <br>
  1. Validate campaign setup <br>
  2. Push to Meta/Google<br>
  3. Monitor launch status<br>
  4. Send notifications<br>
  5. Update internal systems<br>
  6. Generate initial report<br>

Workflow:<br>
  - HTTP Request: Check campaign readiness <br>
   - Switch: Platform (Meta/Google)<br>
  - HTTP Request: Create campaign<br>
  - Delay: Wait for approval<br>
  - IF: Launch successful<br>
    - Update database status<br>
    - Send email notification<br>
    - Generate first performance snapshot<br>
  - ELSE:<br>
    - Log error<br>
    - Send alert<br>
    - Update status to 'failed'<br>



# README

 # Assumptions 

Scale: Initial 100 clients, scalable to 1,000+  <br> 

Data Volume: Millions of performance records<br>

Users: 10-50 internal users, 100+ client users<br>

Compliance: GDPR, CCPA, industry standards<br>

Integration: Standard APIs for Meta, Google platforms <br>

Budget: Mid-market SaaS budget for infrastructure <br>


# Trade-offs 

Relational Database vs NoSQL: PostgreSQL for ACID, Redis for caching <br>

Monolith vs Microservices: Modular monolith initially, microservices later <br>

Cloud vs On-Premise: AWS/GCP for scalability and managed services <br>

Custom vs Off-the-Shelf: Custom solution for flexibility <br>

Performance vs Cost: Optimize for cost-performance balance <br>

# Future Improvements

Advanced AI: Custom fine-tuned models for marketing <br>

Real-time Processing: Streaming analytics with Kafka<br>

Predictive Analytics: ML models for forecasting <br>

Natural Language Interface: Full conversational agent <br>

Multi-tenancy: Enhanced isolation and white-labeling <br>

Automated Testing: Comprehensive test suite <br>

Disaster Recovery: Multi-region deployment <br>

Compliance Automation: Automated GDPR/CCPA tools <br>












