# KC Experian / KnowledgeCore IQ — Architecture Overview

> **Last Updated:** March 29, 2026
> **Version:** 1.0.0

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
│   │         └─────────────────┴──────────────────┘                          │  │
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
│   │  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐  ┌────────────┐ │    │
│   │  │ /api/routes  │  │ /auth/routes │  │/recent/     │  │/datairis/  │ │    │
│   │  │ POST /search │  │ POST /login  │  │  routes     │  │  routes    │ │    │
│   │  │ GET /transac │  │ POST /signup │  │ GET searches│  │            │ │    │
│   │  │ GET /health  │  │ GET /me      │  │ DEL searches│  │            │ │    │
│   │  └──────┬───────┘  └──────┬───────┘  └─────┬───────┘  └─────┬──────┘ │    │
│   └─────────┼─────────────────┼────────────────┼────────────────┼─────────┘    │
│             └─────────────────┴────────────────┴────────────────┘               │
│                                       │                                          │
│   ┌───────────────────────────────────▼──────────────────────────────────────┐  │
│   │                           SERVICE LAYER                                   │  │
│   │                                                                           │  │
│   │  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────────┐   │  │
│   │  │ ExperianService │   │KnowledgeCoreSvc │   │   ScoringService    │   │  │
│   │  │ (Enrich API)    │   │ (SQL DB lookup) │   │ Capacity/Propensity │   │  │
│   │  └─────────────────┘   └─────────────────┘   │ PlannedGiving/Ovrl  │   │  │
│   │  ┌─────────────────┐   ┌─────────────────┐   └─────────────────────┘   │  │
│   │  │ BrightDataSvc   │   │  LinkedInSvc    │   ┌─────────────────────┐   │  │
│   │  │ (Philanthropy)  │   │  (via BrightData│   │  SuggestedAskSvc    │   │  │
│   │  └─────────────────┘   └─────────────────┘   └─────────────────────┘   │  │
│   │  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────────┐   │  │
│   │  │   FECService    │   │ IRSForm990Svc   │   │   ApolloService     │   │  │
│   │  │  (Political $)  │   │ (Nonprofits)    │   │ (People/Company)    │   │  │
│   │  └─────────────────┘   └─────────────────┘   └─────────────────────┘   │  │
│   │  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────────┐   │  │
│   │  │ AIInsightsSvc   │   │ GoogleNewsSvc   │   │   DataIrisSvc       │   │  │
│   │  │ (8 categories)  │   │ (SerpAPI)       │   │                     │   │  │
│   │  └─────────────────┘   └─────────────────┘   └─────────────────────┘   │  │
│   │  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────────┐   │  │
│   │  │ PhoneValidSvc   │   │  EmailValidSvc  │   │    CacheService     │   │  │
│   │  │(Experian Aper.) │   │                 │   │  (SQL Server cache) │   │  │
│   │  └─────────────────┘   └─────────────────┘   └─────────────────────┘   │  │
│   │  ┌─────────────────┐                                                    │  │
│   │  │ SearchHistory   │                                                    │  │
│   │  │    Service      │                                                    │  │
│   │  └─────────────────┘                                                    │  │
│   └───────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
                    │
    ┌───────────────┴──────────────────────────────────────────────────────┐
    │                       EXTERNAL API LAYER                              │
    │                                                                       │
    │  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────┐ │
    │  │ Experian Enrich│  │Experian Aperture│  │     DataIris API       │ │
    │  │   (Demographic │  │ (Phone Valid.) │  │  (Demographic Append)  │ │
    │  │    Financial)  │  │               │  │                        │ │
    │  └────────────────┘  └────────────────┘  └────────────────────────┘ │
    │  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────┐ │
    │  │  BrightData    │  │   Apollo API   │  │       FEC API          │ │
    │  │  (LinkedIn +   │  │ (Professional/ │  │  (Political Donations) │ │
    │  │  Philanthropy) │  │  Company Data) │  │                        │ │
    │  └────────────────┘  └────────────────┘  └────────────────────────┘ │
    │  ┌────────────────┐  ┌────────────────┐                             │
    │  │   SerpAPI      │  │ OpenRouter AI  │                             │
    │  │ (Google News)  │  │ Gemini 2.5 Pro │                             │
    │  │                │  │ (Scores + AI   │                             │
    │  │                │  │  Summaries)    │                             │
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
    │  │  • search_history           │  │    (Constituent giving      │   │
    │  │  • experian_api_cache        │  │     history)               │   │
    │  │  • IRS_Form990_Records      │  │                             │   │
    │  └─────────────────────────────┘  └─────────────────────────────┘   │
    └──────────────────────────────────────────────────────────────────────┘
```

---

## Request Flow Summary

```
User Search Input (Name + Address)
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

## Key Technologies

| Layer         | Technology                                   |
|---------------|----------------------------------------------|
| Frontend      | React 18, TypeScript, Material UI (MUI) v5   |
| HTTP Client   | Axios                                        |
| Auth (Client) | React Context API + JWT localStorage         |
| Backend       | Python 3.11, FastAPI, Uvicorn ASGI           |
| Database ORM  | SQLAlchemy + pyodbc (SQL Server)             |
| Auth (Server) | python-jose (JWT) + bcrypt                   |
| AI Engine     | OpenRouter → Google Gemini 2.5 Pro           |
| Caching       | SQL Server table (`experian_api_cache`)      |
| Deployment    | Azure App Service + Azure Static Web Apps    |

---

## Change Log

| Date           | Version | Change                        | Author |
|----------------|---------|-------------------------------|--------|
| March 29, 2026 | 1.0.0   | Initial architecture document | —      |

> ➕ Add new rows to the **Change Log** whenever a new feature, 
> service, or integration is added to the application.
