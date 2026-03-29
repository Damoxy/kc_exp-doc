# KC Experian / KnowledgeCore IQ — Architecture Overview

> **Last Updated:** March 29, 2026  
> **Version:** 1.1.0

---

## Table of Contents
1. [Architecture Diagram](#architecture-diagram)
2. [Request Flow — Single Search](#request-flow-single-search)
3. [Request Flow — Batch CSV Search ⭐ NEW](#request-flow-batch-csv-search)
4. [Key Technologies](#key-technologies)
5. [Change Log](#change-log)

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT LAYER                                        │
│                                                                                  │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │                   React Frontend  (TypeScript + MUI v5)                  │  │
│   │                                                                          │  │
│   │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐ │  │
│   │  │  AuthPage    │  │  SearchForm  │  │TabbedResults │  │  Header /   │ │  │
│   │  │  LoginForm   │  │  (Name,      │  │  (11 Tabs)   │  │  ScoreGauge │ │  │
│   │  │  SignupForm  │  │   Address,   │  │              │  │  BackToTop  │ │  │
│   │  │  ForgotPass  │  │   ZIP)       │  │              │  │             │ │  │
│   │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └─────────────┘ │  │
│   │         │                 │                  │                           │  │
│   │  ┌──────▼───────────────────────────────────▼───────────────────────┐  │  │
│   │  │              ⭐ NEW — BatchSearchPage                              │  │  │
│   │  │                                                                   │  │  │
│   │  │  ┌─────────────┐  ┌──────────────────┐  ┌─────────────────────┐ │  │  │
│   │  │  │  CSV Upload │  │  BatchListView   │  │  BatchResultsView   │ │  │  │
│   │  │  │  Component  │  │  (preview rows,  │  │  (select ≤100,      │ │  │  │
│   │  │  │  (Drag+Drop)│  │   name/address,  │  │   Experian scores,  │ │  │  │
│   │  │  │             │  │   score badges)  │  │   checkboxes)       │ │  │  │
│   │  │  └─────────────┘  └──────────────────┘  └─────────────────────┘ │  │  │
│   │  │                                                                   │  │  │
│   │  │  ┌────────────────────────────────────────────────────────────┐  │  │  │
│   │  │  │              Export Component  ⭐ NEW                       │  │  │  │
│   │  │  │         Download as PDF  |  Download as Excel               │  │  │  │
│   │  │  └────────────────────────────────────────────────────────────┘  │  │  │
│   │  └───────────────────────────────────────────────────────────────────┘  │  │
│   │                           │                                              │  │
│   │              ┌────────────▼────────────┐                                │  │
│   │              │    services/api.ts       │                                │  │
│   │              │   (Axios HTTP Client)    │                                │  │
│   │              │  searchKnowledgeCore()   │                                │  │
│   │              │  generateAIInsights()    │                                │  │
│   │              │  getTransactions()       │                                │  │
│   │              │  getPhilanthropy()       │                                │  │
│   │              │  validatePhoneNumbers()  │                                │  │
│   │              │  validateEmailAddress()  │                                │  │
│   │              │  ⭐ uploadBatchCSV()      │                                │  │
│   │              │  ⭐ runBatchExperian()    │                                │  │
│   │              │  ⭐ runBatchFullSearch()  │                                │  │
│   │              │  ⭐ downloadBatchReport() │                                │  │
│   │              └────────────┬────────────┘                                │  │
│   └───────────────────────────┼──────────────────────────────────────────────┘  │
│                               │  HTTPS REST + Authorization: Bearer JWT          │
└───────────────────────────────┼─────────────────────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────────────────────┐
│                           API GATEWAY LAYER                                      │
│                    FastAPI  (Python 3.11)  +  Uvicorn ASGI  :8000               │
│                                                                                  │
│   ┌────────────────────────────────────────────────────────────────────────┐    │
│   │                            Routers                                      │    │
│   │  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  ┌─────────────┐ │    │
│   │  │ /api/routes  │  │ /auth/routes │  │/recent/    │  │ ⭐/batch/   │ │    │
│   │  │ POST /search │  │ POST /login  │  │  routes    │  │  routes     │ │    │
│   │  │ GET /transac │  │ POST /signup │  │            │  │POST /upload │ │    │
│   │  │ GET /health  │  │ GET /me      │  │            │  │POST /screen │ │    │
│   │  │              │  │              │  │            │  │POST /full   │ │    │
│   │  │              │  │              │  │            │  │GET  /report │ │    │
│   │  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘  └──────┬──────┘ │    │
│   └─────────┼─────────────────┼────────────────┼────────────────┼─────────┘    │
│             └─────────────────┴────────────────┴────────────────┘               │
│                                       │                                          │
│   ┌───────────────────────────────────▼──────────────────────────────────────┐  │
│   │                           SERVICE LAYER                                   │  │
│   │                                                                           │  │
│   │  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────────┐   │  │
│   │  │ ExperianService │   │KnowledgeCoreSvc │   │   ScoringService    │   │  │
│   │  └─────────────────┘   └─────────────────┘   └─────────────────────┘   │  │
│   │  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────────┐   │  │
│   │  │ BrightDataSvc   │   │  LinkedInSvc    │   │  SuggestedAskSvc    │   │  │
│   │  └─────────────────┘   └─────────────────┘   └─────────────────────┘   │  │
│   │  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────────┐   │  │
│   │  │   FECService    │   │ IRSForm990Svc   │   │   ApolloService     │   │  │
│   │  └─────────────────┘   └─────────────────┘   └─────────────────────┘   │  │
│   │  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────────┐   │  │
│   │  │ AIInsightsSvc   │   │ GoogleNewsSvc   │   │   DataIrisSvc       │   │  │
│   │  └─────────────────┘   └─────────────────┘   └─────────────────────┘   │  │
│   │  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────────┐   │  │
│   │  │ PhoneValidSvc   │   │  EmailValidSvc  │   │    CacheService     │   │  │
│   │  └─────────────────┘   └─────────────────┘   └─────────────────────┘   │  │
│   │  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────────┐   │  │
│   │  │ SearchHistorySvc│   │⭐ BatchCSVService│   │⭐ ReportGenService  │   │  │
│   │  │                 │   │  (CSV parse,    │   │  (PDF via           │   │  │
│   │  │                 │   │   batch runner, │   │   ReportLab/WeasyP, │   │  │
│   │  │                 │   │   job tracking) │   │   Excel via openpyxl│   │  │
│   │  └─────────────────┘   └─────────────────┘   └─────────────────────┘   │  │
│   └───────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
                    │
    ┌───────────────┴──────────────────────────────────────────────────────┐
    │                       EXTERNAL API LAYER                              │
    │                                                                       │
    │  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────┐ │
    │  │ Experian Enrich│  │Experian Aperture│  │     DataIris API       │ │
    │  └────────────────┘  └────────────────┘  └────────────────────────┘ │
    │  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────┐ │
    │  │  BrightData    │  │   Apollo API   │  │       FEC API          │ │
    │  └────────────────┘  └────────────────┘  └────────────────────────┘ │
    │  ┌────────────────┐  ┌────────────────┐                             │
    │  │   SerpAPI      │  │ OpenRouter AI  │                             │
    │  │ (Google News)  │  │ Gemini 2.5 Pro │                             │
    │  └────────────────┘  └────────────────┘                             │
    └──────────────────────────────────────────────────────────────────────┘
                    │
    ┌───────────────▼──────────────────────────────────────────────────────┐
    │                        DATABASE LAYER                                 │
    │                    SQL Server  20.83.252.81                           │
    │                                                                       │
    │  ┌─────────────────────────────┐  ┌─────────────────────────────┐   │
    │  │  KC_ExperianProspectIQSearch│  │  DiocLC_GivingTrendDB_Test  │   │
    │  │                             │  │                             │   │
    │  │  • users                    │  │  • Transaction              │   │
    │  │  • search_history           │  │                             │   │
    │  │  • experian_api_cache        │  │                             │   │
    │  │  • IRS_Form990_Records      │  │                             │   │
    │  │  ⭐ batch_jobs               │  │                             │   │
    │  │  ⭐ batch_job_results        │  │                             │   │
    │  └─────────────────────────────┘  └─────────────────────────────┘   │
    └──────────────────────────────────────────────────────────────────────┘
```

---

## Request Flow — Single Search

```
User enters Name + Address
        │
        ▼
SearchForm → api.ts → POST /search  [JWT Bearer Token]
        │
        ▼
FastAPI validates JWT → extracts user_id
        │
        ▼
┌────────────────────────────────────────┐
│   Parallel API Calls (asyncio.gather)  │
│                                        │
│   1. KnowledgeCore SQL DB  ◄─ Primary  │
│   2. Experian Enrich API               │
│   3. FEC API                           │
│   4. IRS Form 990 (SQL)                │
│   5. Apollo People Search              │
└───────────────┬────────────────────────┘
                │
                ▼
       Merge into unified result object
                │
                ▼
┌───────────────────────────────────────┐
│      Synchronous AI Scoring           │
│                                       │
│  • Capacity Score      (1–10)         │
│  • Propensity Score    (1–10)         │
│  • Planned Giving Score(1–10)         │
│  • Overall Score       (1–10)         │
│  • Suggested Ask  (Low / Mid / High)  │
└───────────────┬───────────────────────┘
                │
                ▼
   ◄── Response returned to frontend ──►
                │
                ▼  (Background Tasks)
┌───────────────────────────────────────┐
│   Background Tasks (non-blocking)     │
│                                       │
│   • AI Insights  (8 categories)       │
│   • LinkedIn profile  (BrightData)    │
│   • Google News       (SerpAPI)       │
│   • Philanthropy contributions        │
│     (BrightData)                      │
└───────────────────────────────────────┘
                │
                ▼
  TabbedResults renders 11 tabs
  AI summaries populate as ready
```

---

## Request Flow — Batch CSV Search ⭐ NEW

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                        PHASE 1 — CSV UPLOAD & PREVIEW                       ║
╚══════════════════════════════════════════════════════════════════════════════╝

User visits /batch page
        │
        ▼
┌───────────────────────────────────────┐
│         CSVUploadComponent            │
│                                       │
│  • Drag & Drop  OR  Browse file       │
│  • Accepts .csv format only           │
│  • Expected columns:                  │
│      FirstName, LastName,             │
│      Address, City, State, ZIP        │
└───────────────┬───────────────────────┘
                │
                ▼
     POST /batch/upload  [JWT]
     (multipart/form-data)
                │
                ▼
┌───────────────────────────────────────┐
│         BatchCSVService               │
│                                       │
│  • Parse & validate CSV rows          │
│  • Flag missing required fields       │
│  • Create batch_job record in DB      │
│    {job_id, user_id, status=PENDING,  │
│     total_rows, created_at}           │
│  • Store rows in batch_job_results    │
│    {job_id, row_index, first_name,    │
│     last_name, address, status=QUEUED}│
└───────────────┬───────────────────────┘
                │
                ▼
  Response → { job_id, preview_rows[],
               total_count, invalid_rows[] }
                │
                ▼
┌───────────────────────────────────────┐
│         BatchListView (UI)            │
│                                       │
│  DataGrid table showing:              │
│  ┌──────┬────────────┬──────────────┐ │
│  │  #   │    Name    │   Address    │ │
│  ├──────┼────────────┼──────────────┤ │
│  │  1   │ John Smith │ 123 Main St  │ │
│  │  2   │ Jane Doe   │ 456 Oak Ave  │ │
│  │  .. │     ...    │     ...      │ │
│  └──────┴────────────┴──────────────┘ │
│                                       │
│  [▶ Run Experian Screen on All]       │
└───────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════════════════

╔══════════════════════════════════════════════════════════════════════════════╗
║                   PHASE 2 — EXPERIAN SCREENING (All Rows)                   ║
╚══════════════════════════════════════════════════════════════════════════════╝

User clicks [Run Experian Screen on All]
        │
        ▼
POST /batch/screen  { job_id }  [JWT]
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│   BatchCSVService — Experian-only screening loop        │
│                                                         │
│   FOR each row in batch_job_results:                    │
│       │                                                 │
│       ▼                                                 │
│   ExperianService.enrich(name, address)                 │
│       ↓  (only Experian — NO other APIs)                │
│   ScoringService.capacity_score_only()                  │
│       ↓                                                 │
│   UPDATE batch_job_results SET                          │
│       experian_data = {...},                            │
│       capacity_score = X,                               │
│       status = SCREENED                                 │
│                                                         │
│   [Progress bar pushed to frontend via polling]         │
│    GET /batch/status/{job_id}                           │
│    → { screened: 45, total: 200, pct: 22% }            │
└─────────────────────────────┬───────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│           BatchResultsView (UI) — Scored List           │
│                                                         │
│  ☑ Select up to 100 rows for Full Search                │
│                                                         │
│  ┌───┬──────────────┬───────────────┬────────────────┐ │
│  │ ☐ │     Name     │    Address    │ Capacity Score │ │
│  ├───┼──────────────┼───────────────┼────────────────┤ │
│  │ ☑ │ John Smith   │ 123 Main St   │   ████░░  7/10 │ │
│  │ ☑ │ Jane Doe     │ 456 Oak Ave   │   █████░  8/10 │ │
│  │ ☐ │ Bob Johnson  │ 789 Pine Rd   │   ███░░░  5/10 │ │
│  │ ☑ │ Mary Brown   │ 321 Elm St    │   ████░░  9/10 │ │
│  │ ..│     ...      │     ...       │      ...       │ │
│  └───┴──────────────┴───────────────┴────────────────┘ │
│                                                         │
│  Selected: 3 / 100 max                                  │
│                                                         │
│  [▶ Run Full Search on Selected (3)]                    │
└─────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════════════════

╔══════════════════════════════════════════════════════════════════════════════╗
║               PHASE 3 — FULL SEARCH on Selected (max 100)                   ║
╚══════════════════════════════════════════════════════════════════════════════╝

User selects ≤ 100 rows → clicks [Run Full Search on Selected]
        │
        ▼
POST /batch/full  { job_id, selected_row_ids[] }  [JWT]
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│   BatchCSVService — Full search loop (≤100 rows)        │
│                                                         │
│   FOR each selected row:                                │
│       │                                                 │
│       ▼                                                 │
│   ┌──────────────────────────────────────────────┐     │
│   │   Parallel API Calls (asyncio.gather)         │     │
│   │                                              │     │
│   │   1. KnowledgeCore SQL DB                    │     │
│   │   2. Experian Enrich API                     │     │
│   │   3. FEC API                                 │     │
│   │   4. IRS Form 990 (SQL)                      │     │
│   │   5. Apollo People Search                    │     │
│   │   6. BrightData Philanthropy                 │     │
│   │   7. LinkedIn (BrightData)                   │     │
│   │   8. Google News (SerpAPI)                   │     │
│   └──────────────────┬───────────────────────────┘     │
│                      │                                  │
│                      ▼                                  │
│   ScoringService (all 4 scores + suggested ask)        │
│   AIInsightsService (8 category summaries)             │
│                      │                                  │
│                      ▼                                  │
│   UPDATE batch_job_results SET                          │
│       full_results = { all API data },                  │
│       scores = { capacity, propensity,                  │
│                  planned_giving, overall },             │
│       suggested_ask = { low, mid, high },              │
│       ai_insights = { 8 categories },                  │
│       status = COMPLETE                                 │
│                                                         │
│   [Progress bar: 12/100 complete]                       │
└─────────────────────────────┬───────────────────────────┘
                              │
                              ▼
              All selected rows = COMPLETE
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│               Export Options (UI)                       │
│                                                         │
│    ┌───────────────────┐   ┌───────────────────────┐   │
│    │  📄 Download PDF  │   │  📊 Download Excel    │   │
│    │  (one donor per   │   │  (one donor per row,  │   │
│    │   page, full      │   │   all fields as       │   │
│    │   profile layout) │   │   columns)            │   │
│    └─────────┬─────────┘   └──────────┬────────────┘   │
│              └──────────┬─────────────┘                 │
│                         │                               │
└─────────────────────────┼───────────────────────────────┘
                          │
                          ▼
        GET /batch/report/{job_id}?format=pdf|excel
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│              ReportGenerationService                    │
│                                                         │
│  PDF (ReportLab / WeasyPrint):                         │
│  ┌───────────────────────────────────────────────┐     │
│  │  Page 1: Donor #1 — John Smith                │     │
│  │  ┌────────────────────────────────────────┐   │     │
│  │  │  Name, Address, DOB, Profile Summary   │   │     │
│  │  │  Scores:  Capacity ██ 7  Overall ██ 8  │   │     │
│  │  │  Suggested Ask:  Low $500  High $5,000  │   │     │
│  │  │  Financial Summary  │  Philanthropy     │   │     │
│  │  │  AI Insights        │  Political Giving │   │     │
│  │  └────────────────────────────────────────┘   │     │
│  │  Page 2: Donor #2 — Jane Doe ...              │     │
│  └───────────────────────────────────────────────┘     │
│                                                         │
│  Excel (openpyxl):                                     │
│  ┌───────────────────────────────────────────────┐     │
│  │  Sheet 1: Summary                             │     │
│  │  Col: Name | Address | Capacity | Propensity  │     │
│  │       PlannedGiving | Overall | LowAsk |      │     │
│  │       MidAsk | HighAsk | AI Profile Summary   │     │
│  │                                               │     │
│  │  Sheet 2: Full Detail (all API fields)        │     │
│  └───────────────────────────────────────────────┘     │
│                                                         │
│  → Stream file as download response                     │
│    Content-Disposition: attachment;                     │
│    filename="batch_report_{job_id}.pdf"                 │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
           File downloads in browser ✅
```

---

## Batch API Endpoints ⭐ NEW

| Method | Path | Description |
|---|---|---|
| `POST` | `/batch/upload` | Upload CSV → parse, validate, store, return preview |
| `POST` | `/batch/screen` | Run Experian-only screen on all rows in job |
| `GET` | `/batch/status/{job_id}` | Poll progress of screen or full search |
| `POST` | `/batch/full` | Run full search on selected rows (max 100) |
| `GET` | `/batch/report/{job_id}` | Download PDF or Excel report (`?format=pdf\|excel`) |
| `GET` | `/batch/jobs` | List all batch jobs for current user |
| `DELETE` | `/batch/jobs/{job_id}` | Delete a batch job and its results |

---

## Batch Database Tables ⭐ NEW

```sql
-- Tracks each CSV upload job
batch_jobs
  id              UNIQUEIDENTIFIER  PRIMARY KEY
  user_id         INT               FK → users.id
  file_name       VARCHAR(255)
  total_rows      INT
  screened_rows   INT               DEFAULT 0
  completed_rows  INT               DEFAULT 0
  status          VARCHAR(50)       -- PENDING | SCREENING | SCREENED
                                    -- PROCESSING | COMPLETE | FAILED
  created_at      DATETIME
  updated_at      DATETIME

-- One row per CSV record per job
batch_job_results
  id              UNIQUEIDENTIFIER  PRIMARY KEY
  job_id          UNIQUEIDENTIFIER  FK → batch_jobs.id
  row_index       INT
  first_name      VARCHAR(100)
  last_name       VARCHAR(100)
  address         VARCHAR(255)
  city            VARCHAR(100)
  state           VARCHAR(2)
  zip             VARCHAR(10)
  experian_data   NVARCHAR(MAX)     -- JSON (Phase 2 result)
  capacity_score  DECIMAL(4,2)      -- Phase 2 score
  full_results    NVARCHAR(MAX)     -- JSON (Phase 3 result)
  scores          NVARCHAR(MAX)     -- JSON {capacity,propensity,...}
  suggested_ask   NVARCHAR(MAX)     -- JSON {low,mid,high}
  ai_insights     NVARCHAR(MAX)     -- JSON {8 categories}
  status          VARCHAR(50)       -- QUEUED | SCREENED | COMPLETE | ERROR
  error_message   VARCHAR(500)
  created_at      DATETIME
  updated_at      DATETIME
```

---

## Key Technologies

| Layer | Technology |
|---|---|
| Frontend | React 18, TypeScript, Material UI (MUI) v5 |
| HTTP Client | Axios |
| Auth (Client) | React Context API + JWT localStorage |
| Backend | Python 3.11, FastAPI, Uvicorn ASGI |
| Database ORM | SQLAlchemy + pyodbc (SQL Server) |
| Auth (Server) | python-jose (JWT) + bcrypt |
| AI Engine | OpenRouter → Google Gemini 2.5 Pro |
| Caching | SQL Server table (`experian_api_cache`) |
| ⭐ PDF Generation | ReportLab or WeasyPrint |
| ⭐ Excel Generation | openpyxl |
| ⭐ CSV Parsing | Python `csv` / `pandas` |
| Deployment | Azure App Service + Azure Static Web Apps |

---

## Change Log

| Date           | Version | Change                                              | Author |
|----------------|---------|-----------------------------------------------------|--------|
| March 29, 2026 | 1.0.0   | Initial architecture document                       | —      |
| March 29, 2026 | 1.1.0   | Added Batch CSV Search flow (Phase 1/2/3), export  | —      |

> ➕ Add new rows to the **Change Log** whenever a new feature,
> service, or integration is added to the application.
