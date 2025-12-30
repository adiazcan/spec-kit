# Tasks: HR Timesheet Agent

**Input**: Design documents from `/specs/001-hr-timesheet-agent/`
**Prerequisites**: plan.md ✓, spec.md ✓, research.md ✓, data-model.md ✓, contracts/ ✓

**Tests**: Tests are NOT mandatory by default. Test tasks are omitted unless explicitly requested in the feature specification.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story (8 user stories: P1-P8).

## Format: `- [ ] [ID] [P?] [Story?] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3) - ONLY for user story phases
- Include exact file paths in descriptions

## Path Conventions

Per plan.md:
- Backend: `backend/src/` and `backend/tests/`
- Frontend: `frontend/src/` and `frontend/tests/`
- Docs: `specs/001-hr-timesheet-agent/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 Create backend project structure: HRTimesheetAgent.API/, HRTimesheetAgent.Core/, HRTimesheetAgent.Infrastructure/, HRTimesheetAgent.AppHost/
- [ ] T002 Initialize .NET 10 solution with Aspire workload and backend projects
- [ ] T003 Create frontend project structure with Vite + React + TypeScript
- [ ] T004 [P] Install backend dependencies: Microsoft Agent Framework, Azure SDK, Aspire 13
- [ ] T005 [P] Install frontend dependencies: CopilotKit, shadcn/ui, Zustand, Tailwind CSS, MSAL
- [ ] T006 [P] Configure ESLint, Prettier for frontend
- [ ] T007 [P] Configure .editorconfig and code formatting for backend
- [ ] T008 Create Aspire AppHost configuration in backend/src/HRTimesheetAgent.AppHost/Program.cs
- [ ] T009 Add MongoDB container configuration to AppHost
- [ ] T010 Add Azurite (Azure Storage Emulator) container configuration to AppHost
- [ ] T011 [P] Setup Tailwind CSS and PostCSS configuration in frontend/
- [ ] T012 [P] Initialize shadcn/ui with base components (Button, Card, Input, Avatar, Spinner)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T013 Create Employee entity model in backend/src/HRTimesheetAgent.Core/Models/Employee.cs
- [ ] T014 Create base DbContext for MongoDB in backend/src/HRTimesheetAgent.Infrastructure/Data/TimesheetDbContext.cs
- [ ] T015 [P] Configure Microsoft Entra ID authentication middleware in backend/src/HRTimesheetAgent.API/Middleware/AuthMiddleware.cs
- [ ] T016 [P] Setup MSAL authentication provider in frontend/src/services/auth.ts
- [ ] T017 Create base API client service in frontend/src/services/api.ts with bearer token injection
- [ ] T018 [P] Implement structured logging infrastructure in backend/src/HRTimesheetAgent.Core/Logging/LoggingService.cs
- [ ] T019 [P] Setup Application Insights telemetry in backend/src/HRTimesheetAgent.API/Program.cs
- [ ] T020 Create audit log service in backend/src/HRTimesheetAgent.Infrastructure/Audit/AuditLogService.cs (Azure Blob Storage)
- [ ] T021 [P] Create error handling middleware in backend/src/HRTimesheetAgent.API/Middleware/ErrorHandlingMiddleware.cs
- [ ] T022 [P] Setup environment configuration management in backend/src/HRTimesheetAgent.API/appsettings.json and frontend/.env files
- [ ] T023 Create Project reference data mock in backend/src/HRTimesheetAgent.Infrastructure/FactorialHR/MockProjectRepository.cs
- [ ] T024 Create Factorial HR mock client interface in backend/src/HRTimesheetAgent.Core/Interfaces/IFactorialHRClient.cs
- [ ] T025 [P] Setup Zustand stores: conversationStore, userStore, timesheetStore, uiStore in frontend/src/stores/
- [ ] T026 [P] Create base AG-UI protocol endpoint in backend/src/HRTimesheetAgent.API/Controllers/CopilotController.cs
- [ ] T027 Configure CopilotKit provider in frontend/src/App.tsx with agent URL
- [ ] T028 Setup Microsoft Agent Service client in backend/src/HRTimesheetAgent.Infrastructure/AI/AgentServiceClient.cs

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - View Today's Timesheet via Natural Conversation (Priority: P1) 🎯 MVP

**Goal**: Enable employees to check their timesheet for today using natural language queries. Agent interprets requests like "Show me my timesheet for today" and returns timesheet data in conversational format.

**Independent Test**: Ask the agent "Show me today's timesheet" and verify it returns accurate timesheet data in a conversational format. Delivers immediate value by allowing employees to check their hours without navigating complex HR portals.

### Implementation for User Story 1

- [ ] T029 [P] [US1] Create TimesheetEntry entity model in backend/src/HRTimesheetAgent.Core/Models/TimesheetEntry.cs (with validation rules from data-model.md)
- [ ] T030 [P] [US1] Create ConversationThread entity model in backend/src/HRTimesheetAgent.Core/Models/ConversationThread.cs
- [ ] T031 [P] [US1] Create ConversationMessage entity model in backend/src/HRTimesheetAgent.Core/Models/ConversationMessage.cs
- [ ] T032 [P] [US1] Create ConversationContext embedded schema in backend/src/HRTimesheetAgent.Core/Models/ConversationContext.cs
- [ ] T033 [US1] Create TimesheetEntry repository in backend/src/HRTimesheetAgent.Infrastructure/Data/TimesheetEntryRepository.cs
- [ ] T034 [US1] Implement TimesheetService with GetDailyTimesheet method in backend/src/HRTimesheetAgent.Core/Services/TimesheetService.cs
- [ ] T035 [US1] Create Agent tool: get_daily_timesheet function definition (OpenAPI 3.0 spec) in backend/src/HRTimesheetAgent.API/Tools/get_daily_timesheet.json
- [ ] T036 [US1] Implement get_daily_timesheet tool execution handler in backend/src/HRTimesheetAgent.Infrastructure/AI/Tools/GetDailyTimesheetTool.cs
- [ ] T037 [US1] Configure agent instructions for timesheet viewing in backend/src/HRTimesheetAgent.Infrastructure/AI/AgentConfiguration.cs
- [ ] T038 [US1] Create date parsing utility (today, yesterday, specific dates) in backend/src/HRTimesheetAgent.Core/Utilities/DateParser.cs
- [ ] T039 [US1] Add logging for view operations in GetDailyTimesheetTool (operation type: "view", resource type: "timesheet_entry")
- [ ] T040 [P] [US1] Create Timesheet display component in frontend/src/components/Timesheet/TimesheetCard.tsx (conversational format)
- [ ] T041 [P] [US1] Create Chat interface component in frontend/src/components/Chat/ChatInterface.tsx
- [ ] T042 [P] [US1] Create Message component in frontend/src/components/Chat/Message.tsx
- [ ] T043 [US1] Integrate CopilotChat component with custom styling in frontend/src/pages/Chat.tsx
- [ ] T044 [US1] Update conversationStore to handle message streaming in frontend/src/stores/conversationStore.ts
- [ ] T045 [US1] Handle empty timesheet response ("You haven't logged any hours for today yet") in agent instructions

**Checkpoint**: At this point, User Story 1 should be fully functional and testable independently. Employee can ask "Show me today's timesheet" and get conversational response.

---

## Phase 4: User Story 2 - Submit Daily Timesheet via Conversation (Priority: P2)

**Goal**: Enable employees to log work hours through natural conversation (e.g., "I worked 8 hours on Project Alpha today"). Agent parses input, validates data, and submits to Factorial HR system.

**Independent Test**: Say "Log 6 hours on Project X today" and verify the entry is created in the system and can be retrieved. Delivers clear value by eliminating form-filling through natural conversation.

### Implementation for User Story 2

- [ ] T046 [US2] Create Agent tool: submit_timesheet function definition (OpenAPI 3.0 spec) in backend/src/HRTimesheetAgent.API/Tools/submit_timesheet.json
- [ ] T047 [US2] Implement submit_timesheet tool execution handler in backend/src/HRTimesheetAgent.Infrastructure/AI/Tools/SubmitTimesheetTool.cs
- [ ] T048 [US2] Add SubmitTimesheet method to TimesheetService in backend/src/HRTimesheetAgent.Core/Services/TimesheetService.cs
- [ ] T049 [US2] Implement timesheet validation in backend/src/HRTimesheetAgent.Core/Validators/TimesheetEntryValidator.cs (hours > 0, hours ≤ 24, valid date, no future dates)
- [ ] T050 [US2] Implement project code validation against mock repository in SubmitTimesheetTool
- [ ] T051 [US2] Add support for decimal hours (2.5, 7.25) in DateParser utility
- [ ] T052 [US2] Implement multiple-entry parsing in agent instructions (e.g., "4 hours on meetings and 4 hours on coding")
- [ ] T053 [US2] Add daily hours warning logic (>8 hours triggers confirmation) in SubmitTimesheetTool
- [ ] T054 [US2] Implement Factorial HR mock submission in backend/src/HRTimesheetAgent.Infrastructure/FactorialHR/MockFactorialHRClient.cs
- [ ] T055 [US2] Add audit logging for submit operations (before/after values) in SubmitTimesheetTool
- [ ] T056 [US2] Configure agent confirmation messages ("I've logged 8 hours on Project Alpha for today") in agent instructions
- [ ] T057 [US2] Handle validation errors with user-friendly messages in agent instructions ("That doesn't seem right - a day only has 24 hours")
- [ ] T058 [P] [US2] Update timesheetStore to cache submitted entries in frontend/src/stores/timesheetStore.ts
- [ ] T059 [US2] Add optimistic UI updates for timesheet submission in ChatInterface component

**Checkpoint**: At this point, User Stories 1 AND 2 should both work independently. Employee can view and submit timesheets conversationally.

---

## Phase 5: User Story 3 - View Timesheet for Specific Date (Priority: P3)

**Goal**: Enable employees to review timesheets for past dates (e.g., "Show me my timesheet for yesterday" or "What did I log on Monday?"). Extends viewing capability to historical data.

**Independent Test**: Ask "Show me my timesheet for last Friday" and verify it returns the correct historical data. Delivers value for reviewing and auditing past entries without additional UI complexity.

### Implementation for User Story 3

- [ ] T060 [US3] Extend get_daily_timesheet tool to accept date parameter in backend/src/HRTimesheetAgent.API/Tools/get_daily_timesheet.json
- [ ] T061 [US3] Enhance DateParser with relative date parsing (yesterday, last Monday, specific formats) in backend/src/HRTimesheetAgent.Core/Utilities/DateParser.cs
- [ ] T062 [US3] Add support for multiple date formats (December 20th, 2025-12-20, last week) in DateParser
- [ ] T063 [US3] Update agent instructions to interpret date references in user queries
- [ ] T064 [US3] Handle empty historical results ("You didn't log any hours on Saturday, December 21st") in agent instructions
- [ ] T065 [P] [US3] Add date filter UI in Timesheet component for quick date selection in frontend/src/components/Timesheet/DateFilter.tsx
- [ ] T066 [US3] Update TimesheetService GetDailyTimesheet to query by date range in backend/src/HRTimesheetAgent.Core/Services/TimesheetService.cs

**Checkpoint**: All three viewing/submission capabilities work independently. Employee can view today, past dates, and submit entries.

---

## Phase 6: User Story 4 - Weekly Timesheet Summary (Priority: P4)

**Goal**: Provide weekly hour summaries (e.g., "Show me my hours for this week"). Retrieves all entries for current week, calculates totals, and presents summary by day and project.

**Independent Test**: Ask "Show my weekly hours" and verify it returns aggregated data for all days in the current week. Delivers value for weekly hour tracking and compliance.

### Implementation for User Story 4

- [ ] T067 [US4] Create Agent tool: get_weekly_summary function definition (OpenAPI 3.0 spec) in backend/src/HRTimesheetAgent.API/Tools/get_weekly_summary.json
- [ ] T068 [US4] Implement get_weekly_summary tool execution handler in backend/src/HRTimesheetAgent.Infrastructure/AI/Tools/GetWeeklySummaryTool.cs
- [ ] T069 [US4] Add GetWeeklySummary method to TimesheetService in backend/src/HRTimesheetAgent.Core/Services/TimesheetService.cs
- [ ] T070 [US4] Implement week calculation logic (Monday-Sunday) in DateParser utility
- [ ] T071 [US4] Create aggregation logic (total hours per day, per project, overall) in TimesheetService
- [ ] T072 [US4] Handle partial week data display in agent instructions (e.g., Wednesday shows Mon-Wed with note about remaining days)
- [ ] T073 [US4] Add logging for weekly summary view operations
- [ ] T074 [P] [US4] Create WeeklySummary display component in frontend/src/components/Timesheet/WeeklySummary.tsx
- [ ] T075 [US4] Configure agent to format weekly summary conversationally with totals and breakdowns

**Checkpoint**: User can now view daily, historical, and weekly timesheet data through natural conversation.

---

## Phase 7: User Story 5 - Check Start and End Times for Specific Day (Priority: P5)

**Goal**: Enable employees to check when they started and ended work on a particular day (e.g., "When did I start work yesterday?"). Retrieves clock-in/clock-out records from Factorial HR.

**Independent Test**: Ask "When did I start work today?" and verify the agent returns the recorded clock-in time. Delivers value for attendance verification and schedule compliance checking.

### Implementation for User Story 5

- [ ] T076 [P] [US5] Create ClockEvent entity model in backend/src/HRTimesheetAgent.Core/Models/ClockEvent.cs
- [ ] T077 [US5] Create ClockEvent repository in backend/src/HRTimesheetAgent.Infrastructure/Data/ClockEventRepository.cs
- [ ] T078 [US5] Create ClockEventService in backend/src/HRTimesheetAgent.Core/Services/ClockEventService.cs
- [ ] T079 [US5] Create Agent tool: get_clock_times function definition in backend/src/HRTimesheetAgent.API/Tools/get_clock_times.json
- [ ] T080 [US5] Implement get_clock_times tool execution handler in backend/src/HRTimesheetAgent.Infrastructure/AI/Tools/GetClockTimesTool.cs
- [ ] T081 [US5] Add GetClockEventsForDate method to ClockEventService
- [ ] T082 [US5] Implement duration calculation (start to end time) in ClockEventService
- [ ] T083 [US5] Handle unclosed shifts (clocked in but not out) in agent instructions ("You clocked in at 9:00 AM today and you're still working")
- [ ] T084 [US5] Handle no clock records ("You don't have any time records for Saturday") in agent instructions
- [ ] T085 [US5] Format times in employee's local time zone using Employee.timeZone
- [ ] T086 [US5] Add logging for clock time view operations
- [ ] T087 [P] [US5] Create ClockStatus display component in frontend/src/components/Timesheet/ClockStatus.tsx

**Checkpoint**: Employee can now check clock-in/clock-out times for any day through conversation.

---

## Phase 8: User Story 6 - Clock In via Natural Conversation (Priority: P6)

**Goal**: Enable employees to record their work start time through natural conversation (e.g., "I'm starting work now" or "Clock me in"). Records current timestamp or specified time.

**Independent Test**: Say "Clock me in" and verify a start time record is created with the current timestamp. Delivers value by simplifying attendance recording through voice/text.

### Implementation for User Story 6

- [ ] T088 [US6] Create Agent tool: clock_in function definition in backend/src/HRTimesheetAgent.API/Tools/clock_in.json
- [ ] T089 [US6] Implement clock_in tool execution handler in backend/src/HRTimesheetAgent.Infrastructure/AI/Tools/ClockInTool.cs
- [ ] T090 [US6] Add ClockIn method to ClockEventService in backend/src/HRTimesheetAgent.Core/Services/ClockEventService.cs
- [ ] T091 [US6] Implement duplicate clock-in prevention (check if already clocked in today) in ClockEventService
- [ ] T092 [US6] Support current time default or specified time ("Clock me in at 8:30 AM") in ClockInTool
- [ ] T093 [US6] Support past date/time clock-in ("Clock me in yesterday at 9 AM") in ClockInTool
- [ ] T094 [US6] Add time zone handling (capture UTC + local timestamp) in ClockEventService
- [ ] T095 [US6] Implement Factorial HR mock clock-in API in MockFactorialHRClient
- [ ] T096 [US6] Add audit logging for clock_in operations
- [ ] T097 [US6] Configure agent confirmation messages ("You're clocked in at 9:00 AM. Have a great day!") in agent instructions
- [ ] T098 [US6] Handle error: already clocked in ("You're already clocked in as of 9:00 AM. Did you mean to clock out?")
- [ ] T099 [P] [US6] Create ClockIn button and status indicator in frontend/src/components/Timesheet/ClockInButton.tsx

**Checkpoint**: Employee can now clock in using natural conversation, with timestamp recorded.

---

## Phase 9: User Story 7 - Clock Out via Natural Conversation (Priority: P7)

**Goal**: Enable employees to record their work end time through natural conversation (e.g., "I'm done for the day" or "Clock me out"). Records timestamp, calculates total hours, and provides summary.

**Independent Test**: Say "Clock me out" (after clocking in) and verify an end time is recorded, total hours are calculated, and a summary is provided. Delivers value by automating end-of-day time recording.

### Implementation for User Story 7

- [ ] T100 [US7] Create Agent tool: clock_out function definition in backend/src/HRTimesheetAgent.API/Tools/clock_out.json
- [ ] T101 [US7] Implement clock_out tool execution handler in backend/src/HRTimesheetAgent.Infrastructure/AI/Tools/ClockOutTool.cs
- [ ] T102 [US7] Add ClockOut method to ClockEventService in backend/src/HRTimesheetAgent.Core/Services/ClockEventService.cs
- [ ] T103 [US7] Implement validation: cannot clock out without clock in
- [ ] T104 [US7] Support current time default or specified time ("Clock me out at 5:00 PM") in ClockOutTool
- [ ] T105 [US7] Support past date/time clock-out ("Clock me out yesterday at 6 PM") in ClockOutTool
- [ ] T106 [US7] Calculate total hours worked (end - start) in ClockEventService
- [ ] T107 [US7] Implement duplicate clock-out prevention (already clocked out) in ClockEventService
- [ ] T108 [US7] Implement Factorial HR mock clock-out API in MockFactorialHRClient
- [ ] T109 [US7] Add audit logging for clock_out operations with before/after values
- [ ] T110 [US7] Configure agent summary message ("You're clocked out at 5:30 PM. You worked 8 hours and 30 minutes today. Great job!") in agent instructions
- [ ] T111 [US7] Handle error: no clock-in record ("I don't have a clock-in record for you today. When did you start work?")
- [ ] T112 [P] [US7] Create ClockOut button and total hours display in frontend/src/components/Timesheet/ClockOutButton.tsx

**Checkpoint**: Employee can now complete full clock-in/clock-out workflow with automatic hour calculation.

---

## Phase 10: User Story 8 - Submit Timesheet Corrections (Priority: P8)

**Goal**: Enable employees to correct previously submitted timesheet entries through natural conversation. Creates correction entries that reference originals while maintaining audit trail.

**Independent Test**: Submit an entry, then say "Correct yesterday's entry: change 8 hours to 7 hours on Project X" and verify both original and correction entries are tracked. Delivers value by enabling error correction while maintaining data integrity.

### Implementation for User Story 8

- [ ] T113 [US8] Create Agent tool: submit_correction function definition in backend/src/HRTimesheetAgent.API/Tools/submit_correction.json
- [ ] T114 [US8] Implement submit_correction tool execution handler in backend/src/HRTimesheetAgent.Infrastructure/AI/Tools/SubmitCorrectionTool.cs
- [ ] T115 [US8] Add SubmitCorrection method to TimesheetService in backend/src/HRTimesheetAgent.Core/Services/TimesheetService.cs
- [ ] T116 [US8] Implement correction entry creation (entryType: "correction", originalEntryId reference) in TimesheetService
- [ ] T117 [US8] Add validation: original entry must exist and be submitted (isReadOnly: true)
- [ ] T118 [US8] Support multiple field corrections (hours, project, description) in SubmitCorrectionTool
- [ ] T119 [US8] Parse correction intent from natural language in agent instructions ("change 8 hours to 7 hours", "adjust from Alpha to Beta")
- [ ] T120 [US8] Implement correction chain tracking (original → correction1 → correction2) in TimesheetService
- [ ] T121 [US8] Update GetDailyTimesheet to show corrected values with indicator in TimesheetService
- [ ] T122 [US8] Implement Factorial HR mock correction API in MockFactorialHRClient
- [ ] T123 [US8] Add comprehensive audit logging for corrections (original + correction values) in SubmitCorrectionTool
- [ ] T124 [US8] Configure agent confirmation ("I've submitted a correction: your Project Alpha hours for yesterday are now 7 hours instead of 8")
- [ ] T125 [US8] Handle error: original entry not found ("I couldn't find that entry. Can you specify the date and project you want to correct?")
- [ ] T126 [P] [US8] Create Correction history view in frontend/src/components/Timesheet/CorrectionHistory.tsx
- [ ] T127 [US8] Add correction indicator to TimesheetCard component ("corrected from X")

**Checkpoint**: All user stories (P1-P8) are now complete. Full timesheet management via conversation is functional.

---

## Phase 11: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories and ensure constitution compliance

- [ ] T128 [P] Update README.md with project overview, prerequisites, and quick start
- [ ] T129 [P] Document API endpoints in contracts/api.openapi.yaml (verify completeness)
- [ ] T130 [P] Create ADRs for key decisions in architecture-decisions/ (Agent Framework, AG-UI protocol, Aspire)
- [ ] T131 [P] Add inline documentation for all public APIs and services
- [ ] T132 Code cleanup: apply Single Responsibility Principle to all services
- [ ] T133 Code cleanup: eliminate code duplication across tools and services
- [ ] T134 [P] Performance optimization: add response time monitoring for agent calls (<2s target)
- [ ] T135 [P] Performance optimization: optimize database queries with proper indexes
- [ ] T136 [P] Add edge case handling: ambiguous date references (clarification questions)
- [ ] T137 [P] Add edge case handling: overlapping timesheet entries (>24 hours warning)
- [ ] T138 [P] Add edge case handling: token expiration and re-authentication flow
- [ ] T139 Security hardening: validate all user input in tools
- [ ] T140 Security hardening: verify authorization (employee can only access own data) in all operations
- [ ] T141 Security hardening: implement rate limiting on AG-UI endpoint
- [ ] T142 [P] Accessibility: verify WCAG 2.1 AA compliance (keyboard navigation, screen reader support)
- [ ] T143 [P] Accessibility: add ARIA labels to all interactive components
- [ ] T144 [P] Error handling: ensure all errors have user-friendly messages
- [ ] T145 [P] Error handling: verify graceful API failure handling
- [ ] T146 [P] Implement conversation session management (30-minute inactivity timeout)
- [ ] T147 [P] Implement data retention policies (7-year timesheet retention, 90-day conversation TTL)
- [ ] T148 [P] Add Application Insights custom metrics (submission success rate, intent accuracy)
- [ ] T149 [P] Configure alerting for uptime and error rates (99.5% uptime target)
- [ ] T150 Validate quickstart.md by following all setup steps in fresh environment
- [ ] T151 Constitution compliance audit: verify all testing, UX, quality, and security requirements met

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3-10)**: All depend on Foundational phase completion
  - User stories can then proceed in parallel (if staffed)
  - Or sequentially in priority order (P1 → P2 → P3 → P4 → P5 → P6 → P7 → P8)
- **Polish (Phase 11)**: Depends on all desired user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational (Phase 2) - No dependencies on other stories
- **User Story 2 (P2)**: Can start after Foundational (Phase 2) - Builds on US1 models but independently testable
- **User Story 3 (P3)**: Can start after Foundational (Phase 2) - Extends US1 viewing capability
- **User Story 4 (P4)**: Can start after Foundational (Phase 2) - Aggregates US1 data but independently testable
- **User Story 5 (P5)**: Can start after Foundational (Phase 2) - New entity (ClockEvent), independent of timesheet entries
- **User Story 6 (P6)**: Can start after US5 (ClockEvent model) - Writes clock events
- **User Story 7 (P7)**: Can start after US6 (clock in first) - Completes clock event pairs
- **User Story 8 (P8)**: Can start after US2 (needs timesheet entries to correct) - Correction workflow

### Within Each User Story

- Entity models before services
- Services before tool handlers
- Tool handlers before agent configuration
- Core implementation before frontend integration
- Story complete before moving to next priority

### Parallel Opportunities

**Setup Phase (Phase 1)**:
- T004, T005 (backend + frontend dependencies)
- T006, T007 (linting configuration)
- T011, T012 (frontend tooling)

**Foundational Phase (Phase 2)**:
- T015, T016 (auth middleware + MSAL)
- T018, T019 (logging infrastructure)
- T021, T022 (error handling + config)
- T025, T026 (frontend stores + AG-UI endpoint)

**User Story 1 (Phase 3)**:
- T029-T032 (all entity models)
- T040-T042 (all frontend components)

**User Story 2 (Phase 4)**:
- T058, T059 (frontend updates)

**User Story 3 (Phase 5)**:
- T065, T066 (UI + service updates)

**User Story 4 (Phase 6)**:
- T074, T075 (frontend + agent config)

**User Story 5 (Phase 7)**:
- T076, T077 (model + repository)
- T087 (frontend component)

**User Story 6 (Phase 8)**:
- T099 (frontend component)

**User Story 7 (Phase 9)**:
- T112 (frontend component)

**User Story 8 (Phase 10)**:
- T126, T127 (frontend components)

**Polish Phase (Phase 11)**:
- T128-T131 (documentation tasks)
- T134-T135 (performance tasks)
- T136-T138 (edge case tasks)
- T142-T143 (accessibility tasks)
- T144-T145 (error handling tasks)
- T146-T149 (operational tasks)

**Once Foundational phase completes, user stories can start in parallel**:
- Developer A: User Story 1 (P1) → Tasks T029-T045
- Developer B: User Story 5 (P5) → Tasks T076-T087 (ClockEvent entity is independent)
- Developer C: Setup for User Story 2 (P2) → Tasks T046-T059 (after US1 models exist)

---

## Parallel Example: User Story 1

```bash
# Launch all entity models together:
Task T029: "Create TimesheetEntry entity model in backend/src/HRTimesheetAgent.Core/Models/TimesheetEntry.cs"
Task T030: "Create ConversationThread entity model in backend/src/HRTimesheetAgent.Core/Models/ConversationThread.cs"
Task T031: "Create ConversationMessage entity model in backend/src/HRTimesheetAgent.Core/Models/ConversationMessage.cs"
Task T032: "Create ConversationContext embedded schema in backend/src/HRTimesheetAgent.Core/Models/ConversationContext.cs"

# Launch all frontend components together:
Task T040: "Create Timesheet display component in frontend/src/components/Timesheet/TimesheetCard.tsx"
Task T041: "Create Chat interface component in frontend/src/components/Chat/ChatInterface.tsx"
Task T042: "Create Message component in frontend/src/components/Chat/Message.tsx"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001-T012)
2. Complete Phase 2: Foundational (T013-T028) - CRITICAL - blocks all stories
3. Complete Phase 3: User Story 1 (T029-T045)
4. **STOP and VALIDATE**: Test User Story 1 independently
5. Deploy/demo if ready - employees can now view today's timesheet via conversation

**MVP Delivered**: Employees can ask "Show me today's timesheet" and get conversational response. This is the minimum viable feature demonstrating the conversational interface works.

### Incremental Delivery (Recommended)

1. Complete Setup + Foundational → Foundation ready
2. Add User Story 1 (P1) → Test independently → Deploy/Demo (MVP! Read-only timesheet viewing)
3. Add User Story 2 (P2) → Test independently → Deploy/Demo (Complete read-write workflow)
4. Add User Story 3 (P3) → Test independently → Deploy/Demo (Historical data access)
5. Add User Story 4 (P4) → Test independently → Deploy/Demo (Weekly summaries)
6. Add User Story 5 (P5) → Test independently → Deploy/Demo (Clock time queries)
7. Add User Story 6 (P6) → Test independently → Deploy/Demo (Clock in capability)
8. Add User Story 7 (P7) → Test independently → Deploy/Demo (Complete clock in/out workflow)
9. Add User Story 8 (P8) → Test independently → Deploy/Demo (Correction workflow)

Each story adds value without breaking previous stories. After US2, the system provides complete daily timesheet management.

### Parallel Team Strategy

With multiple developers (after Foundational phase completion):

**Sprint 1** (Core Timesheet Features):
- Developer A: User Story 1 (T029-T045) - View today's timesheet
- Developer B: User Story 5 (T076-T087) - Clock times query (independent entity)
- Developer C: Polish foundational tasks (documentation, monitoring setup)

**Sprint 2** (Write Operations):
- Developer A: User Story 2 (T046-T059) - Submit timesheet
- Developer B: User Story 6 (T088-T099) - Clock in
- Developer C: User Story 3 (T060-T066) - View historical

**Sprint 3** (Advanced Features):
- Developer A: User Story 4 (T067-T075) - Weekly summaries
- Developer B: User Story 7 (T100-T112) - Clock out
- Developer C: User Story 8 (T113-T127) - Corrections

**Sprint 4** (Polish):
- All developers: Phase 11 tasks (T128-T151)

---

## Summary Statistics

- **Total Tasks**: 151
- **Setup Tasks**: 12 (Phase 1)
- **Foundational Tasks**: 16 (Phase 2)
- **User Story Tasks**: 99 (Phases 3-10)
  - User Story 1 (P1): 17 tasks
  - User Story 2 (P2): 14 tasks
  - User Story 3 (P3): 7 tasks
  - User Story 4 (P4): 9 tasks
  - User Story 5 (P5): 12 tasks
  - User Story 6 (P6): 12 tasks
  - User Story 7 (P7): 13 tasks
  - User Story 8 (P8): 15 tasks
- **Polish Tasks**: 24 (Phase 11)
- **Parallel Opportunities**: 42 tasks marked [P]

**MVP Scope** (Recommended initial release):
- Phase 1: Setup (12 tasks)
- Phase 2: Foundational (16 tasks)
- Phase 3: User Story 1 (17 tasks)
- **Total MVP**: 45 tasks

**Production-Ready Scope** (All P1-P3 user stories):
- MVP + User Story 2 (14 tasks) + User Story 3 (7 tasks) + Polish subset
- **Total**: ~75 tasks for core read-write timesheet functionality

---

## Notes

- [P] tasks = different files, no dependencies within phase
- [Story] label (US1-US8) maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Tests are NOT included by default (not requested in feature specification)
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Use Aspire dashboard (https://localhost:15888) to monitor all services during development
- All file paths are absolute from repository root
- Follow constitution requirements: Single Responsibility, 80% coverage (if tests added), WCAG 2.1 AA, security best practices
