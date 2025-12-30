# Data Model: HR Timesheet Agent

**Phase**: 1 - Design & Contracts  
**Date**: 2025-12-29  
**Purpose**: Define entities, relationships, validation rules, and state transitions

---

## Overview

This document defines the core data entities for the HR Timesheet Agent system, extracted from functional requirements and user stories. The data model supports:
- Conversational AI state management (threads, messages, context)
- Timesheet operations (entries, clock in/out, corrections)
- Audit and compliance (7-year retention, modification history)
- Authentication and authorization (employee identity, permissions)

---

## Entity Definitions

### 1. Employee

Represents an authenticated user interacting with the agent.

**Attributes**:
| Field | Type | Required | Description | Validation |
|-------|------|----------|-------------|------------|
| `id` | string (GUID) | ✅ | Unique employee identifier | UUID v4 format |
| `userPrincipalName` | string | ✅ | Microsoft Entra ID UPN | Valid email format |
| `displayName` | string | ✅ | Full name for personalized responses | Max 100 chars |
| `employeeNumber` | string | ✅ | HR system employee number | Alphanumeric, max 20 chars |
| `department` | string | ❌ | Department/cost center | Max 50 chars |
| `timeZone` | string | ✅ | IANA time zone identifier | Valid TZ database name (e.g., "America/New_York") |
| `locale` | string | ✅ | Language/region preference | BCP 47 format (e.g., "en-US") |
| `roles` | string[] | ✅ | Authorization roles | Enum: ["Employee", "Manager", "Admin"] |
| `createdAt` | DateTime | ✅ | Account creation timestamp | ISO 8601 UTC |
| `lastActiveAt` | DateTime | ✅ | Last interaction timestamp | ISO 8601 UTC |

**Relationships**:
- One employee has many timesheet entries (1:N)
- One employee has many conversation threads (1:N)

**Storage**:
- **Production**: Azure Cosmos DB (`employees` container)
- **Development**: MongoDB (`employees` collection)
- **Partition Key**: `/id` (employee ID for efficient queries)

**Indexes**:
- `userPrincipalName` (unique)
- `employeeNumber` (unique)

---

### 2. TimesheetEntry

Represents a single work log with immutable history.

**Attributes**:
| Field | Type | Required | Description | Validation |
|-------|------|----------|-------------|------------|
| `id` | string (GUID) | ✅ | Unique entry identifier | UUID v4 format |
| `employeeId` | string (GUID) | ✅ | Foreign key to Employee | Must exist in employees collection |
| `date` | Date | ✅ | Date worked (no time component) | ISO 8601 date (YYYY-MM-DD) |
| `hours` | decimal | ✅ | Hours worked | 0.25 to 24.00 (15-min increments) |
| `projectCode` | string | ✅ | Project/task identifier | Alphanumeric, max 50 chars |
| `projectName` | string | ✅ | Human-readable project name | Max 200 chars |
| `description` | string | ❌ | Work performed description | Max 500 chars |
| `startTime` | DateTime? | ❌ | Clock-in timestamp (if applicable) | ISO 8601 UTC, nullable |
| `endTime` | DateTime? | ❌ | Clock-out timestamp (if applicable) | ISO 8601 UTC, nullable, must be > startTime |
| `entryType` | string | ✅ | Entry classification | Enum: ["original", "correction"] |
| `originalEntryId` | string (GUID)? | ❌ | Reference to original entry (if correction) | Must exist if entryType=correction |
| `correctionReason` | string? | ❌ | Explanation for correction | Max 500 chars, required if entryType=correction |
| `isReadOnly` | boolean | ✅ | Submission lock flag | Default: false, true after submission |
| `submittedAt` | DateTime? | ❌ | Submission timestamp | ISO 8601 UTC, null until submitted |
| `submittedBy` | string (GUID)? | ❌ | Employee who submitted (usually same as employeeId) | Foreign key to Employee |
| `createdAt` | DateTime | ✅ | Record creation timestamp | ISO 8601 UTC |
| `modifiedAt` | DateTime | ✅ | Last modification timestamp | ISO 8601 UTC |

**Relationships**:
- Many entries belong to one employee (N:1)
- Correction entries reference one original entry (N:1)

**Storage**:
- **Production**: Azure Cosmos DB (`timesheetEntries` container)
- **Development**: MongoDB (`timesheetEntries` collection)
- **Partition Key**: `/employeeId` (optimize queries by employee)

**Indexes**:
- `employeeId + date` (composite, for daily queries)
- `employeeId + date range` (for weekly/monthly queries)
- `originalEntryId` (to find corrections)

**State Transitions**:
```
[Draft] (isReadOnly=false)
  ↓ submit()
[Submitted] (isReadOnly=true, submittedAt set)
  ↓ correct()
[Correction Created] (new entry with originalEntryId set)
```

**Validation Rules**:
1. `hours` must be in 0.25 increments (15 minutes)
2. `date` cannot be in the future (per FR-014)
3. If `startTime` and `endTime` both provided, calculated hours must match `hours` field (within 1-minute tolerance)
4. Total hours for a single date cannot exceed 24 hours (warning, not error - per Edge Cases)
5. `projectCode` must exist in Projects reference table (mock for phase 1)
6. Cannot modify entry if `isReadOnly=true` (must create correction)

---

### 3. ClockEvent

Represents clock-in/clock-out events for attendance tracking.

**Attributes**:
| Field | Type | Required | Description | Validation |
|-------|------|----------|-------------|------------|
| `id` | string (GUID) | ✅ | Unique event identifier | UUID v4 format |
| `employeeId` | string (GUID) | ✅ | Foreign key to Employee | Must exist in employees collection |
| `eventType` | string | ✅ | Clock event type | Enum: ["clock_in", "clock_out"] |
| `timestamp` | DateTime | ✅ | Event occurrence time | ISO 8601 UTC |
| `localTimestamp` | DateTime | ✅ | Timestamp in employee's time zone | ISO 8601 with offset |
| `date` | Date | ✅ | Date component (for daily queries) | ISO 8601 date (YYYY-MM-DD) |
| `location` | string? | ❌ | Optional geolocation | Max 100 chars (e.g., "Office", "Remote") |
| `deviceInfo` | string? | ❌ | Device identifier | Max 200 chars |
| `correctedFrom` | DateTime? | ❌ | Original timestamp (if corrected) | ISO 8601 UTC |
| `correctionReason` | string? | ❌ | Explanation for correction | Max 500 chars |
| `createdAt` | DateTime | ✅ | Record creation timestamp | ISO 8601 UTC |
| `modifiedAt` | DateTime | ✅ | Last modification timestamp | ISO 8601 UTC |

**Relationships**:
- Many clock events belong to one employee (N:1)
- Clock events can be paired (clock-in → clock-out) via date and sequence

**Storage**:
- **Production**: Azure Cosmos DB (`clockEvents` container)
- **Development**: MongoDB (`clockEvents` collection)
- **Partition Key**: `/employeeId`

**Indexes**:
- `employeeId + date` (composite, for daily queries)
- `employeeId + eventType + date` (for finding open shifts)

**Validation Rules**:
1. Cannot clock in twice without clocking out (per FR-027)
2. Cannot clock out without prior clock in (per FR-028)
3. `timestamp` cannot be in the future (max 5-minute tolerance for clock skew)
4. Corrections create new records (preserve immutability)

---

### 4. ConversationThread

Represents a multi-turn conversation session between employee and agent.

**Attributes**:
| Field | Type | Required | Description | Validation |
|-------|------|----------|-------------|------------|
| `id` | string (GUID) | ✅ | Unique thread identifier | UUID v4 format |
| `employeeId` | string (GUID) | ✅ | Foreign key to Employee | Must exist in employees collection |
| `startedAt` | DateTime | ✅ | Thread creation timestamp | ISO 8601 UTC |
| `lastMessageAt` | DateTime | ✅ | Most recent message timestamp | ISO 8601 UTC |
| `status` | string | ✅ | Thread lifecycle status | Enum: ["active", "completed", "abandoned"] |
| `context` | object | ✅ | Conversation state/context | JSON object (see ConversationContext schema) |
| `messageCount` | int | ✅ | Total messages in thread | Min: 0 |
| `metadata` | object | ❌ | Additional properties | JSON object (flexible schema) |

**Relationships**:
- One thread belongs to one employee (N:1)
- One thread has many messages (1:N)

**Storage**:
- **Production**: Azure Cosmos DB (`conversationThreads` container)
- **Development**: MongoDB (`conversationThreads` collection)
- **Partition Key**: `/employeeId`
- **TTL**: 90 days (per Edge Cases: session expires after 90 days)

**Indexes**:
- `employeeId + status` (find active threads)
- `lastMessageAt` (cleanup old threads)

---

### 5. ConversationMessage

Represents a single message within a conversation thread.

**Attributes**:
| Field | Type | Required | Description | Validation |
|-------|------|----------|-------------|------------|
| `id` | string (GUID) | ✅ | Unique message identifier | UUID v4 format |
| `threadId` | string (GUID) | ✅ | Foreign key to ConversationThread | Must exist in conversationThreads collection |
| `role` | string | ✅ | Message sender role | Enum: ["user", "assistant", "system"] |
| `content` | string | ✅ | Message text content | Max 1000 chars (per Edge Cases) |
| `timestamp` | DateTime | ✅ | Message creation timestamp | ISO 8601 UTC |
| `toolCalls` | object[]? | ❌ | Functions invoked by agent | Array of ToolCall objects |
| `metadata` | object? | ❌ | Additional properties | JSON object (intent, confidence, etc.) |

**ToolCall Schema**:
```typescript
{
  "id": string,              // Tool invocation ID
  "name": string,            // Function name (e.g., "submit_timesheet")
  "arguments": object,       // Input parameters
  "result": object?,         // Function return value
  "status": string,          // "pending" | "success" | "error"
  "error": string?           // Error message if status=error
}
```

**Relationships**:
- Many messages belong to one thread (N:1)

**Storage**:
- **Production**: Azure Cosmos DB (`conversationMessages` container)
- **Development**: MongoDB (`conversationMessages` collection)
- **Partition Key**: `/threadId`

**Indexes**:
- `threadId + timestamp` (ordered message retrieval)

**Validation Rules**:
1. `content` max 1000 characters (per Edge Cases)
2. `role=assistant` messages may have `toolCalls` array
3. Messages immutable after creation

---

### 6. ConversationContext (Embedded in ConversationThread)

Represents the current conversation state for the agent.

**Schema**:
```typescript
{
  "currentRequest": {
    "type": string,              // "view" | "submit" | "clock_in" | "clock_out" | "correction" | "summary"
    "intent": string,            // Parsed user intent
    "confidence": decimal        // 0.0 to 1.0
  },
  "dateContext": {
    "referenceDate": Date,       // "today", "yesterday", or specific date
    "periodStart": Date?,        // For weekly/monthly queries
    "periodEnd": Date?
  },
  "parsedEntities": {
    "hours": decimal?,
    "projectCode": string?,
    "projectName": string?,
    "description": string?
  },
  "clarificationState": {
    "isAwaiting": boolean,       // Is agent waiting for clarification?
    "question": string?,         // Clarification question asked
    "attemptCount": int,         // Number of clarifications (max 2 per FR-054)
    "topic": string?             // What needs clarification
  },
  "pendingAction": {
    "actionType": string?,       // "submit_timesheet" | "clock_out" | etc.
    "requiresConfirmation": boolean,
    "confirmationPrompt": string?,
    "data": object?              // Action payload
  },
  "userPreferences": {
    "defaultProjectCode": string?,
    "preferredHoursFormat": string,  // "decimal" | "hours_minutes"
    "notificationsEnabled": boolean
  }
}
```

**Validation Rules**:
1. `clarificationState.attemptCount` max 2 (per FR-054)
2. `currentRequest.confidence` between 0.0 and 1.0

---

### 7. AuditLog (Append-only)

Represents immutable audit trail for compliance.

**Attributes**:
| Field | Type | Required | Description | Validation |
|-------|------|----------|-------------|------------|
| `id` | string (GUID) | ✅ | Unique log entry identifier | UUID v4 format |
| `timestamp` | DateTime | ✅ | Event occurrence timestamp | ISO 8601 UTC |
| `employeeId` | string (GUID) | ✅ | Employee who performed action | Foreign key to Employee |
| `operationType` | string | ✅ | Type of operation | Enum: ["view", "submit", "clock_in", "clock_out", "correct", "auth_login", "auth_logout", "auth_failed"] |
| `resourceType` | string | ✅ | Resource affected | Enum: ["timesheet_entry", "clock_event", "conversation", "auth"] |
| `resourceId` | string (GUID)? | ❌ | Specific resource ID | Foreign key to relevant entity |
| `action` | string | ✅ | Action performed | Max 100 chars (e.g., "submitted_timesheet", "viewed_weekly_summary") |
| `outcome` | string | ✅ | Operation result | Enum: ["success", "failure", "partial"] |
| `details` | object | ✅ | Operation details | JSON object (before/after values, error details) |
| `ipAddress` | string? | ❌ | Client IP address | IPv4 or IPv6 format |
| `userAgent` | string? | ❌ | Client user agent | Max 500 chars |
| `correlationId` | string (GUID) | ✅ | Trace correlation ID | UUID v4 format |

**Storage**:
- **Production**: Azure Blob Storage (append-only blobs, one file per day)
- **Development**: Azure Storage Emulator (Azurite)
- **File Path**: `audit-logs/{year}/{month}/{day}.jsonl` (JSON Lines format)
- **Retention**: 7 years (per FR-050)

**Indexes**:
- N/A (stored in Blob Storage, query via Azure Data Lake or export to Cosmos DB for search)

**Sample Log Entry**:
```json
{
  "id": "log-12345-67890",
  "timestamp": "2025-12-29T14:32:10.123Z",
  "employeeId": "emp-abc-123",
  "operationType": "submit",
  "resourceType": "timesheet_entry",
  "resourceId": "entry-xyz-789",
  "action": "submitted_timesheet",
  "outcome": "success",
  "details": {
    "hours": 8.0,
    "projectCode": "PROJ-ALPHA",
    "date": "2025-12-29",
    "before": null,
    "after": { "id": "entry-xyz-789", "hours": 8.0 }
  },
  "ipAddress": "192.168.1.100",
  "userAgent": "Mozilla/5.0...",
  "correlationId": "trace-abc-123"
}
```

---

### 8. Project (Reference Data - Mocked)

Represents valid project codes for timesheet entries.

**Attributes**:
| Field | Type | Required | Description | Validation |
|-------|------|----------|-------------|------------|
| `code` | string | ✅ | Unique project code | Alphanumeric, max 50 chars |
| `name` | string | ✅ | Display name | Max 200 chars |
| `description` | string? | ❌ | Project details | Max 500 chars |
| `isActive` | boolean | ✅ | Can accept new entries | Default: true |
| `startDate` | Date? | ❌ | Project start date | ISO 8601 date |
| `endDate` | Date? | ❌ | Project end date | ISO 8601 date |

**Storage**:
- **Mock**: In-memory list in backend API (hardcoded for phase 1)
- **Future**: Integrate with Factorial HR API for real project data

**Sample Data**:
```json
[
  { "code": "PROJ-ALPHA", "name": "Project Alpha", "isActive": true },
  { "code": "PROJ-BETA", "name": "Project Beta", "isActive": true },
  { "code": "ADMIN", "name": "Administrative Tasks", "isActive": true },
  { "code": "MEETING", "name": "Meetings & Collaboration", "isActive": true }
]
```

---

## Entity Relationships Diagram

```
Employee (1) ────────── (N) TimesheetEntry
    │                           │
    │                           │ (1) originalEntry
    │                           └── (N) corrections
    │
    ├────────── (N) ClockEvent
    │
    └────────── (N) ConversationThread (1) ────── (N) ConversationMessage
                                           │
                                           └─ context: ConversationContext
```

---

## Data Validation Summary

### Cross-Entity Rules
1. **Employee Authorization**: All operations validate `employeeId` matches authenticated user (per FR-037)
2. **Date Consistency**: Clock events and timesheet entries for same date must align (employee cannot log 8 hours if clocked in/out shows 6 hours)
3. **Correction Chain**: Corrections reference original entries; original entries cannot be deleted
4. **Audit Completeness**: Every data-modifying operation generates audit log entry (per FR-034, FR-035)

### Performance Considerations
1. **Partition Strategy**: All collections partitioned by `employeeId` for efficient single-user queries
2. **Index Coverage**: Queries for daily/weekly timesheets covered by composite indexes
3. **TTL Policies**: Conversation threads auto-expire after 90 days (reduce storage costs)
4. **Blob Storage**: Audit logs in cold storage after 90 days (lifecycle policy)

---

## Data Migration & Seeding

### Development Environment
```csharp
// Seed initial data for testing
public static class DataSeeder
{
    public static void SeedEmployees(MongoDatabase db)
    {
        var employees = new[]
        {
            new Employee 
            { 
                Id = Guid.Parse("emp-00000001-0000-0000-0000-000000000001"),
                UserPrincipalName = "john.doe@company.com",
                DisplayName = "John Doe",
                EmployeeNumber = "EMP001",
                TimeZone = "America/New_York",
                Locale = "en-US",
                Roles = new[] { "Employee" }
            }
        };
        db.GetCollection<Employee>("employees").InsertMany(employees);
    }
}
```

### Production
- Employee data synced from Microsoft Entra ID (via Azure AD Graph API or SCIM)
- Project data fetched from Factorial HR API (mocked for phase 1)

---

## Schema Versioning

**Current Version**: 1.0.0

**Future Enhancements** (out of scope for phase 1):
- **Approval Workflow**: Add `ApprovalRequest` entity for manager review
- **Leave/PTO**: Add `LeaveRequest` entity for time-off tracking
- **Payroll Integration**: Add `PayPeriod` entity for bi-weekly/monthly cycles
- **Multi-Tenant**: Add `tenantId` partition key for enterprise deployments

---

**Document Status**: ✅ Approved for Phase 1 implementation  
**Next Step**: Generate API contracts in `/contracts` directory
