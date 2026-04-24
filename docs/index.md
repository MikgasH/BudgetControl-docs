# BudgetControl — Personal Finance Tracker with Multi-Currency Conversion

**Diploma Project Documentation**

| Field | Value |
|---|---|
| Student | Mikita Hashkin |
| Group | 22hr |
| Supervisor | Jan Bartnitsky |
| University | European Humanities University, Vilnius, Lithuania |
| Academic Year | 2025–2026 |

## Project Overview

BudgetControl is an Android application for deliberate, manual tracking of personal income and expenses across multiple currencies and bank accounts. The system is designed for students abroad, expats, frequent travellers, and budget-conscious individuals or families who hold cards from several banks and cannot obtain a consolidated view of their real spending from any single banking app. Its distinguishing capability is bank-commission-adjusted currency conversion: when the user records a foreign-currency transaction, they select the issuing bank directly in the entry form (for example, Revolut at 0.5% versus SEB at 3%), and the application applies the real commission to the interbank rate supplied by the custom CERPS backend. This yields an effective accuracy of approximately 95–98% without automatic bank integration, with optional 100% accuracy through a user-corrected amount. A dedicated cash mode allows manual entry of exchange-office rates, and offline-first behaviour with cached rates and stale-rate warnings ensures the application remains functional without network connectivity.

## Project Repositories and Live Services

| Resource | URL |
|---|---|
| Android application repository | https://github.com/MikgasH/BudgetControl |
| CERPS backend repository | https://github.com/MikgasH/CERPS |
| Currency Service — Swagger UI | https://supportive-vision-production.up.railway.app/swagger-ui.html |
| Analytics Service — Swagger UI | https://sparkling-curiosity-production-9ffd.up.railway.app/swagger-ui.html |

## Evaluation Criteria

The project is documented against five evaluation criteria defined by the academic programme. Each criterion is captured as an Architecture Decision Record covering context, decision, alternatives, consequences, implementation details, and a traceable requirements checklist.

| # | Criterion | Scope | Documentation |
|---|---|---|---|
| 1 | Back-end | CERPS microservices: Currency Service and Analytics Service, deployed to Railway, with layered architecture, OpenAPI specification, encrypted provider keys, and global error handling. | [backend.md](02-technical/criteria/backend.md) |
| 2 | Front-end | Android client built with Jetpack Compose and Clean Architecture (MVVM), feature-based navigation, reusable UI component library, and centralised API interaction through Retrofit. | [frontend.md](02-technical/criteria/frontend.md) |
| 3 | Database | Dual data layer: PostgreSQL with Liquibase migrations on the CERPS side (3NF, functional indexes, role-based access) and Room SQLite on the device (eight entities, thirteen migrations, foreign-key cascades). | [database.md](02-technical/criteria/database.md) |
| 4 | AI Assistant | Gemini-powered bank commission lookup integrated into the bank management and onboarding flows, with engineered prompts, rate-limit handling, and deterministic fallbacks. | [ai-assistant.md](02-technical/criteria/ai-assistant.md) |
| 5 | Refined UX | User research with four personas, key user flows, onboarding wizard, navigation structure, consistent Material 3 component library, bilingual English/Russian interface, and light/dark themes with automatic detection. | [refined-ux.md](02-technical/criteria/refined-ux.md) |

## Documentation Navigation

| Section | Description |
|---|---|
| [Project Overview](01-project-overview/problem-and-goals.md) | Problem statement, vision, stakeholders, personas, scope, functional and non-functional requirements, market research. |
| [Features and Requirements](01-project-overview/features.md) | Epics, user stories with acceptance criteria, use case diagram, and non-functional requirement targets. |
| [Technical Criteria](02-technical/criteria/) | Five criterion-level Architecture Decision Records listed in the table above. |
| [Tech Stack](02-technical/tech-stack.md) | Technology table with justification for all choices. |
| [Deployment](02-technical/deployment.md) | Production deployment on Railway, local Docker Compose setup, environment variables, and release configuration. |
| [User Guide — Features](03-user-guide/features.md) | End-user walkthrough of transaction entry, multi-currency conversion, statistics, rate history, and settings. |
| [User Guide — FAQ](03-user-guide/faq.md) | Frequently asked questions covering offline behaviour, rate freshness, cash mode, and data privacy. |
| [Retrospective](04-retrospective/index.md) | Lessons learned, technical debt, and future work. |
