# Implementation Plan: HR Timesheet Agent

**Branch**: `001-hr-timesheet-agent` | **Date**: 2025-12-29 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-hr-timesheet-agent/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

A modern HR conversational AI agent for managing timesheets with natural language interaction. The system enables employees to view and submit daily timesheets, clock in/out, and access historical timesheet data through a conversational interface. Built with .NET 10 backend using Microsoft Agent Framework orchestrated by Aspire 13, with a Vite React frontend using AG-UI protocol (via CopilotKit) for agent-user communication. Azure services provide production infrastructure (DocumentDB for conversation history, Blob Storage for audit logs, AI Foundry for LLM hosting, Container Apps for hosting), while local development uses MongoDB and Azure Storage emulators. The system integrates with Factorial HR API (mocked) for timesheet operations.

## Technical Context

**Language/Version**: C# with .NET 10, TypeScript/JavaScript with React 18+  
**Primary Dependencies**: 
- Backend: Microsoft Agent Framework, .NET Aspire 13, Azure SDK, AG-UI protocol server
- Frontend: Vite 5+, React 18+, CopilotKit (AG-UI protocol client), shadcn/ui (UI components), Zustand (state management), Tailwind CSS
- AI/LLM: Azure AI Foundry SDK  
**Storage**: 
- Production: Azure Cosmos DB (DocumentDB API) for conversation history, Azure Blob Storage for audit logs
- Development: MongoDB emulator (conversation history), Azure Storage Emulator/Azurite (audit logs)  
**Testing**: xUnit (backend), Vitest (frontend), Playwright (E2E integration tests)  
**Target Platform**: 
- Backend: Linux containers on Azure Container Apps
- Frontend: Static web app (SPA) hosted on Azure Container Apps or Azure Static Web Apps
- Development: Local (Windows/macOS/Linux) with Aspire orchestration (handles all containers)  
**Project Type**: Web application (backend API + frontend SPA)  
**Performance Goals**: 
- Agent response time <2s for 95% of requests (per SC-006)
- API response times p95 <500ms (per constitution)
- Support 100 concurrent conversations (per SC-004)  
**Constraints**: 
- 99.5% uptime during business hours (per SC-010)
- WCAG 2.1 Level AA accessibility compliance
- OAuth authentication via Microsoft Entra ID (Azure AD) mandatory
- 7-year data retention for timesheet records and audit logs (per FR-049, FR-050)  
**Scale/Scope**: 
- Initial deployment: <1000 employees
- Growth target: 10,000 employees within 12 months
- ~50 API endpoints (timesheet CRUD, clock in/out, queries, corrections)
- ~15-20 React components/pages

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

**Code Quality & Maintainability**:
- [x] Single Responsibility verified for all proposed modules/services (Backend: AgentService, TimesheetService, AuthService, AuditService; Frontend: Chat, Timesheet, Auth components)
- [x] No code duplication patterns identified across proposed structure (shared utilities in common lib)
- [x] Technical debt items documented with tracked issues (N/A at planning stage - will track during implementation)
- [x] Code naming conventions clear and consistent (C# PascalCase, TypeScript camelCase, consistent domain terminology)

**Testing Discipline**:
- [x] Contract tests planned for all public APIs and service boundaries (API contracts, Factorial HR mock interface)
- [x] Integration tests planned for each user story (Playwright E2E tests covering all 8 prioritized user stories)
- [x] Unit tests planned for business logic components (xUnit for backend services, Vitest for frontend components)
- [x] 80% code coverage target acknowledged and achievable (enforced via CI/CD pipeline)
- [x] Test-first development workflow confirmed in tasks (will be specified in tasks.md)

**User Experience Consistency**:
- [x] Design system or component library defined (shadcn/ui for UI components; AG-UI protocol for agent-user communication; Zustand for state management)
- [x] Error handling strategy documented (FR-011: user-friendly errors, FR-016: graceful API failures)
- [x] Loading states and feedback mechanisms specified (FR-042: typing indicators, loading states)
- [x] Accessibility (WCAG 2.1 AA) requirements acknowledged (FR-018 OAuth, constitution requirement confirmed)
- [x] Performance targets defined and measurable (SC-001, SC-006: response times; SC-004: concurrent users)

**Quality Standards**:
- [x] Security requirements identified (FR-018: OAuth/Entra ID, FR-037: authorization, FR-032-036: comprehensive logging, secrets via Azure Key Vault)
- [x] Performance benchmarks defined (API p95 <500ms, agent response <2s, 100 concurrent conversations)
- [x] Documentation plan includes README, API docs, ADRs (quickstart.md, contracts/, architecture decisions in research.md)
- [x] Observability strategy defined (Application Insights for monitoring, structured logging per FR-032-038)

## Project Structure

### Documentation (this feature)

```text
specs/001-hr-timesheet-agent/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── infrastructure-plan.md # Bicep IaC modules for Azure deployment
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
│   ├── api.openapi.yaml # REST API contract
│   └── models.schema.json # Data model schemas
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

```text
backend/
├── src/
│   ├── HRTimesheetAgent.API/           # ASP.NET Core Web API
│   │   ├── Controllers/                # API endpoints
│   │   ├── Middleware/                 # Auth, logging, error handling
│   │   ├── Program.cs                  # Application entry point
│   │   └── appsettings.json            # Configuration
│   ├── HRTimesheetAgent.Core/          # Domain logic
│   │   ├── Models/                     # Entities (Employee, TimesheetEntry)
│   │   ├── Services/                   # Business logic (TimesheetService, AgentService)
│   │   └── Interfaces/                 # Contracts
│   ├── HRTimesheetAgent.Infrastructure/ # External integrations
│   │   ├── Data/                       # Cosmos DB, Blob Storage
│   │   ├── FactorialHR/                # Factorial HR mock client
│   │   └── AI/                         # Azure AI Foundry integration
│   └── HRTimesheetAgent.AppHost/       # Aspire orchestration
│       └── Program.cs                  # Service orchestration
└── tests/
    ├── HRTimesheetAgent.Tests.Contract/ # API contract tests
    ├── HRTimesheetAgent.Tests.Integration/ # E2E tests
    └── HRTimesheetAgent.Tests.Unit/    # Unit tests

frontend/
├── src/
│   ├── components/                     # Reusable UI components
│   │   ├── Chat/                       # Chat interface components
│   │   ├── Timesheet/                  # Timesheet display components
│   │   └── Common/                     # Shared components
│   ├── pages/                          # Route pages
│   │   ├── Home.tsx
│   │   ├── Chat.tsx
│   │   └── Login.tsx
│   ├── services/                       # API client, auth
│   │   ├── api.ts                      # Backend API client
│   │   ├── auth.ts                     # Entra ID authentication
│   │   └── agent.ts                    # Agent interaction logic
│   ├── App.tsx                         # Root component
│   └── main.tsx                        # Entry point
├── tests/
│   ├── unit/                           # Vitest unit tests
│   └── e2e/                            # Playwright E2E tests
├── package.json
├── vite.config.ts
└── tsconfig.json

.aspire/                                # Aspire configuration

azure-deployment/                       # Azure Container Apps IaC
└── main.bicep                          # Azure infrastructure
```

**Structure Decision**: Web application architecture selected based on backend (.NET) + frontend (React) tech stack. Backend follows clean architecture with separate layers for API, Core (domain), and Infrastructure (external dependencies). Aspire orchestrates all services and containers (MongoDB, Azurite emulators). Frontend follows standard React SPA structure with component-based organization.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

No constitutional violations identified. All gates pass:
- Code quality standards met through clean architecture and naming conventions
- Comprehensive testing strategy covers contract, integration, and unit tests
- UX consistency ensured via AG-UI protocol (agent-user interaction) and shadcn/ui (components)
- Security, performance, documentation, and observability requirements satisfied
