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

#2. Database Design


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







