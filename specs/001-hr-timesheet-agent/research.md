# Research: HR Timesheet Agent Technology Stack

**Phase**: 0 - Outline & Research  
**Date**: 2025-12-29  
**Purpose**: Resolve all technical unknowns and document technology decisions for implementation

---

## Executive Summary

This document consolidates research on Microsoft Agent Framework, .NET Aspire 13, AG-UI protocol, and frontend implementation for building a conversational HR timesheet agent. Key findings:

- **Microsoft Agent Framework + Semantic Kernel** provides production-ready conversational AI capabilities with Azure AI Foundry integration
- **.NET Aspire 13** simplifies distributed application orchestration with automatic service discovery and configuration
- **AG-UI protocol** standardizes agent-user interaction between frontend and backend (event-based, open standard)
- **CopilotKit** provides the AG-UI protocol client for React applications
- **shadcn/ui** provides accessible, customizable UI components with Tailwind CSS
- **Azure services** provide enterprise-grade storage, monitoring, and hosting with local emulator support

---

## Technology Decisions

### Decision 1: Conversational AI Framework

**Decision**: Use **Microsoft Agent Service** (Azure AI Agent Service)

**Rationale**:
- Cloud-native runtime environment enabling intelligent agents to operate securely and autonomously
- Production-ready orchestration with built-in enterprise features (observability, security, BCDR, compliance)
- Native Azure AI Foundry integration for model hosting (GPT-4o, GPT-4o-mini, o3-mini supported)
- Function calling with automatic tool selection based on user intent
- Built-in memory management and conversation state persistence
- Microsoft Entra ID authentication with on-behalf-of (OBO) support
- Comprehensive evaluation framework for intent resolution and tool call accuracy
- Rich ecosystem of built-in tools: File Search, Code Interpreter, SharePoint, Fabric Data Agent
- Extensible with custom tools via OpenAPI 3.0, Azure Functions, Azure Logic Apps, Model Context Protocol
- HIPAA-compliant for healthcare scenarios, with highest privacy and security standards

**Alternatives Considered**:
1. **Microsoft Foundry Agent Service with Semantic Kernel** - Rejected per user constraint
2. **Semantic Kernel standalone** - Rejected per user constraint
3. **LangChain** - More flexible but lacks enterprise features and Azure native integration
4. **Custom implementation** - Higher development cost, no built-in observability/evaluation
5. **Microsoft Bot Framework** - Legacy approach, not optimized for modern LLM-based agents

**Implementation Approach**:
```csharp
using Azure;
using Azure.AI.Agents.Persistent;
using Azure.Identity;

// Create PersistentAgentsClient instance
var projectEndpoint = Environment.GetEnvironmentVariable("PROJECT_ENDPOINT");
var modelDeploymentName = Environment.GetEnvironmentVariable("MODEL_DEPLOYMENT_NAME");

PersistentAgentsClient client = new(
    projectEndpoint, 
    new DefaultAzureCredential()
);

// Create agent with custom tools (defined via OpenAPI spec or Azure Functions)
PersistentAgent agent = client.Administration.CreateAgent(
    model: modelDeploymentName,
    name: "hr-timesheet-agent",
    instructions: @"You are an HR timesheet assistant. 
        You help employees log hours, view their timesheets, clock in/out, 
        and submit corrections. Use the available tools to interact with 
        the timesheet system.",
    tools: new List<ToolDefinition> 
    { 
        new FunctionToolDefinition(timesheetTools),
        new CodeInterpreterToolDefinition()
    }
);

// Create thread for conversation
PersistentAgentThread thread = client.Threads.CreateThread();

// Add user message
PersistentThreadMessage message = client.Messages.CreateMessage(
    thread.Id,
    MessageRole.User,
    "Show my hours for this week"
);

// Run agent with automatic tool execution
ThreadRun run = client.Runs.CreateRun(thread.Id, agent.Id);

// Poll for completion
do
{
    Thread.Sleep(TimeSpan.FromMilliseconds(500));
    run = client.Runs.GetRun(thread.Id, run.Id);
}
while (run.Status == RunStatus.Queued 
    || run.Status == RunStatus.InProgress 
    || run.Status == RunStatus.RequiresAction);

// Fetch response messages
Pageable<PersistentThreadMessage> messages = client.Messages.GetMessages(
    threadId: thread.Id,
    order: ListSortOrder.Ascending
);

foreach (var msg in messages)
{
    foreach (var content in msg.ContentItems)
    {
        if (content is MessageTextContent textContent)
        {
            Console.WriteLine($"{msg.Role}: {textContent.Text}");
        }
    }
}
```

**Key Components**:
- **Agent Service**: Cloud-native runtime for autonomous agent operations
- **PersistentAgentsClient**: .NET SDK for interacting with Azure AI Foundry projects
- **Tools/Functions**: Custom timesheet functions (view, submit, clock in/out, corrections) defined via OpenAPI 3.0, Azure Functions, or Azure Logic Apps
- **Thread Management**: Persistent conversation state with automatic context handling via PersistentAgentThread
- **Built-in Tools**: Code Interpreter for calculations, File Search for RAG scenarios, SharePoint, Fabric Data Agent
- **Authentication**: Microsoft Entra ID with DefaultAzureCredential for secure authentication

---

### Decision 2: Backend Orchestration

**Decision**: Use **.NET Aspire 13** for cloud-native application orchestration

**Rationale**:
- Code-first approach eliminates complex configuration files (replaces docker-compose, Kubernetes YAML)
- Automatic service discovery and dependency injection (connection strings, endpoints)
- Unified development experience (F5 to start entire stack)
- Built-in observability dashboard (logs, traces, metrics)
- Seamless emulator-to-Azure switching for dev/prod environments
- Type-safe service references with IntelliSense

**Alternatives Considered**:
1. **Docker Compose** - Rejected: Manual configuration, no automatic service discovery, replaced by Aspire
2. **Kubernetes** - Overengineered for initial scale, steep learning curve
3. **Tye** - Microsoft's previous attempt, now deprecated in favor of Aspire

**Implementation Approach**:
```csharp
var builder = DistributedApplication.CreateBuilder(args);

// Azure AI Foundry project configuration
var aiProjectEndpoint = builder.AddParameter("ai-project-endpoint");
var modelDeploymentName = builder.AddParameter("model-deployment-name");

// Local: MongoDB for conversation state (development)
var mongo = builder.AddMongoDB("mongo")
    .WithDataVolume()
    .AddDatabase("timesheets");

// Local: Azure Storage emulator for audit logs
var storage = builder.AddAzureStorage("storage")
    .RunAsEmulator()
    .AddBlobs("audit-logs");

// Backend API with timesheet business logic
var api = builder.AddProject<Projects.TimesheetAPI>("api")
    .WithReference(mongo)
    .WithReference(storage)
    .WaitFor(mongo);

// Agent Service (communicates with Azure AI Agent Service)
var agentService = builder.AddProject<Projects.AgentService>("agent-service")
    .WithEnvironment("PROJECT_ENDPOINT", aiProjectEndpoint)
    .WithEnvironment("MODEL_DEPLOYMENT_NAME", modelDeploymentName)
    .WithReference(api)
    .WaitFor(api);

// React frontend (Vite + CopilotKit + shadcn/ui)
var frontend = builder.AddNpmApp("frontend", "../frontend")
    .WithReference(agentService)
    .WithHttpEndpoint(env: "PORT")
    .WithExternalHttpEndpoints()
    .PublishAsDockerFile();

builder.Build().Run();
```

**Key Benefits**:
- Single F5 starts: MongoDB, Azurite, API, Agent Service, Frontend
- Automatic environment variable injection (`ConnectionStrings__mongo`, `services__api__https__0`, `PROJECT_ENDPOINT`)
- Real-time telemetry dashboard at `https://localhost:15888`
- Production deployment via `azd up` (generates Bicep/ARM templates)
- Service-to-service communication via service discovery (no hardcoded URLs)

---

### Decision 3: Agent-User Interaction Protocol

**Decision**: Use **AG-UI protocol** with **CopilotKit** client and **shadcn/ui** for UI components

**Rationale**:
- AG-UI is an open, event-based protocol that standardizes agent-user interaction
- Supported by Microsoft Agent Service through Responses API (wireline compatible)
- CopilotKit provides production-ready AG-UI protocol client for React applications
- AG-UI enables streaming chat, generative UI, shared state, thinking steps, and frontend tool calls
- Event-driven architecture handles long-running agent operations, multi-turn conversations, and nondeterministic UI updates
- shadcn/ui provides accessible, customizable components built on Radix UI primitives with Tailwind CSS
- WCAG 2.1 Level AA accessibility compliance (constitutional requirement)

**Alternatives Considered**:
1. **Custom WebSocket implementation** - Reinventing the wheel, no standardization
2. **REST/GraphQL only** - Not suitable for streaming, real-time agent interactions
3. **Server-Sent Events (SSE)** - Unidirectional, lacks bidirectional state management

**Implementation Approach**:
```typescript
import { CopilotKit } from '@copilotkit/react-core';
import { CopilotChat } from '@copilotkit/react-ui';
import { ThemeProvider } from '@/components/theme-provider';

function App() {
  return (
    <ThemeProvider defaultTheme="light" storageKey="hr-timesheet-theme">
      <CopilotKit agentUrl="https://localhost:7001/api/copilot">
        <CopilotChat
          instructions="You are an HR timesheet assistant..."
          labels={{
            title: "HR Timesheet Agent",
            initial: "How can I help with your timesheet today?"
          }}
        />
      </CopilotKit>
    </ThemeProvider>
  );
}
```

**Architecture Strategy**:
- AG-UI protocol handles agent-frontend communication (events, streaming, state)
- CopilotKit manages AG-UI protocol client-side (connection, message handling, UI integration)
- shadcn/ui provides accessible components (`Card`, `Avatar`, `Input`, `Button`, `Spinner`) built on Radix UI
- Tailwind CSS for utility-first styling with full customization
- Custom components built on shadcn foundation for domain-specific UI (timesheet cards, clock status)
- Ensure keyboard navigation, screen reader compatibility
- Implement responsive design (mobile-first)

---

### Decision 4: Storage Strategy

**Decision**: Use **Azure Cosmos DB (DocumentDB API)** for conversation history and **Azure Blob Storage** for audit logs

**Rationale**:
- **Cosmos DB**: Schema-flexible document storage ideal for conversational context (messages, state, user preferences)
  - Multi-region replication for availability
  - Single-digit millisecond latency
  - Native integration with Microsoft Agent Service
  - Thread and message storage managed automatically by Agent Service
- **Blob Storage**: Cost-effective long-term storage for immutable audit logs
  - 7-year retention requirement (FR-049, FR-050)
  - Append-only blobs for audit compliance
  - Lifecycle policies for automatic archival

**Local Development**:
- **MongoDB emulator** replaces Cosmos DB (compatible document API)
- **Azure Storage Emulator (Azurite)** replaces Blob Storage

**Alternatives Considered**:
1. **SQL Server** - Rigid schema, not optimized for document storage
2. **Redis** - Not suitable for long-term persistence, no 7-year retention
3. **PostgreSQL** - Requires schema management, no native Azure agent integration

**Data Models**:
```typescript
// Conversation history (Cosmos DB)
{
  "id": "thread-{guid}",
  "userId": "emp-12345",
  "messages": [
    { "role": "user", "content": "Show my hours today", "timestamp": "..." },
    { "role": "assistant", "content": "...", "timestamp": "..." }
  ],
  "state": { "currentPeriod": "2025-12-29", "pendingEntries": [] }
}

// Audit log (Blob Storage - append-only)
2025-12-29T14:32:10Z | emp-12345 | SUBMIT_TIMESHEET | { "hours": 8, "project": "Alpha" }
```

---

### Decision 5: Authentication & Authorization

**Decision**: Use **Microsoft Entra ID (Azure AD)** with OAuth 2.0

**Rationale**:
- Constitutional requirement (FR-018: mandatory OAuth authentication)
- Native integration with Azure services
- Single sign-on (SSO) with corporate credentials
- Managed Identity support for service-to-service authentication
- Fine-grained authorization via Azure RBAC
- Token-based API security (Bearer tokens)

**Alternatives Considered**:
1. **API Keys** - Not suitable for production, no user attribution
2. **Basic Auth** - Insecure, deprecated
3. **Auth0** - Additional cost, unnecessary when using Azure stack

**Implementation**:
```csharp
// Backend API
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"));

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("EmployeeOnly", policy =>
        policy.RequireRole("Employee"));
});
```

```typescript
// Frontend
import { PublicClientApplication } from '@azure/msal-browser';

const msalConfig = {
  auth: {
    clientId: import.meta.env.VITE_CLIENT_ID,
    authority: `https://login.microsoftonline.com/${tenantId}`,
    redirectUri: window.location.origin
  }
};

const msalInstance = new PublicClientApplication(msalConfig);
```

---

### Decision 6: Testing Strategy

**Decision**: Multi-layer testing with **xUnit** (backend), **Vitest** (frontend), **Playwright** (E2E)

**Rationale**:
- Constitutional requirement: 80% code coverage minimum
- Contract tests validate API specifications (OpenAPI schema)
- Integration tests cover each user story (P1-P8)
- Unit tests for business logic (timesheet validation, date parsing)
- E2E tests ensure frontend-backend integration

**Test Pyramid**:
```
E2E Tests (Playwright)           [10%] - Slow, fragile
  ↑
Integration Tests (xUnit)        [30%] - Medium speed
  ↑
Contract Tests (Pact/OpenAPI)    [20%] - Fast
  ↑
Unit Tests (xUnit/Vitest)        [40%] - Very fast
```

**Alternatives Considered**:
1. **Manual testing only** - Not scalable, violates constitution
2. **Jest** - Slower than Vitest for React testing
3. **Selenium** - Outdated, Playwright offers better developer experience

**Implementation**:
```csharp
// Integration test with Aspire TestHost
var appHost = await DistributedApplicationTestingBuilder
    .CreateAsync<Projects.AppHost>();

await using var app = await appHost.BuildAsync();
await app.StartAsync();

var httpClient = app.CreateHttpClient("api");
var response = await httpClient.PostAsJsonAsync("/timesheet/submit", entry);
Assert.Equal(HttpStatusCode.Created, response.StatusCode);
```

---

### Decision 7: Observability & Monitoring

**Decision**: Use **Application Insights** for unified telemetry

**Rationale**:
- Automatic integration via Aspire Service Defaults
- Distributed tracing across services (correlation IDs)
- Custom metrics for business KPIs (timesheet submissions, agent response times)
- Log aggregation from all services
- Alerting and dashboards

**Key Metrics to Track**:
- Agent intent resolution accuracy (target: >95%)
- API response times (p95 <500ms)
- Timesheet submission success rate
- Concurrent conversation count
- Authentication failures
- Error rates by category

**Alternatives Considered**:
1. **Datadog** - Additional cost, not native to Azure
2. **ELK Stack** - Higher maintenance overhead
3. **Azure Monitor** - Subset of Application Insights, less feature-rich

---

## Best Practices Consolidated

### Microsoft Agent Service
1. **Function Design**:
   - Descriptive names (snake_case): `submit_timesheet`, `view_daily_hours`
   - Minimal parameters (use primitives: string, int, bool, float)
   - Clear descriptions for AI understanding (function and parameter descriptions)
   - Return type schemas (JSON schemas) for structured outputs
   - Use OpenAPI 3.0 specification for custom tools

2. **State Management**:
   - Leverage Agent Service's built-in thread management
   - Thread IDs persist conversation context automatically
   - Store additional state in Cosmos DB if needed (user preferences, cached data)
   - Pass state IDs to agent (not full data) to reduce tokens
   - Maintain user context (current period, pending entries) in thread metadata

3. **Error Handling**:
   - User-friendly errors: "I'm having trouble connecting to the HR system"
   - Retry logic for transient failures
   - Circuit breaker pattern for backend API calls
   - Handle tool execution failures gracefully
   - Monitor run status (queued, in_progress, requires_action, completed, failed)

4. **Testing**:
   - Intent resolution evaluator (target: >90%)
   - Tool call accuracy evaluator (target: >95%)
   - Response completeness checks
   - Test multi-turn conversations with thread serialization
   - Validate tool approval workflows if required

### .NET Aspire
1. **Lifecycle Management**:
   - Use `ContainerLifetime.Persistent` for databases (preserve data across restarts)
   - `WaitFor()` ensures correct startup order
   - Health checks validate readiness

2. **Configuration**:
   - Use parameters for environment-specific values (API keys, connection strings)
   - Avoid hardcoding secrets (use Azure Key Vault)

3. **Development Workflow**:
   - F5 starts entire stack
   - Dashboard at `https://localhost:15888` for real-time monitoring
   - Hot reload for both backend and frontend

### AG-UI Protocol + Frontend Implementation
1. **Accessibility**:
   - Use semantic HTML (`<main>`, `<nav>`, `<section>`)
   - ARIA labels for dynamic content (new messages)
   - Keyboard navigation (Tab, Enter, Escape)
   - Focus management in dialogs

2. **Performance**:
   - Lazy load message history
   - Virtual scrolling for long conversations
   - `React.memo` for message components
   - Optimistic UI updates (show message immediately, confirm later)

3. **Responsive Design**:
   - Mobile-first approach
   - Touch-friendly buttons (min 44x44px)
   - Adaptive layouts (single column on mobile, multi-column on desktop)

---

## Integration Patterns

### Pattern 1: Query Flow (Read Operations)
```
User: "Show my hours for last week"
  ↓
Frontend → Backend API → Microsoft Agent Service
  ↓
Agent identifies intent → Calls get_timesheet_entries tool
  ↓
Tool execution → Timesheet API → Factorial HR (mock) → Returns data
  ↓
Agent formats response → Streams to user via AG-UI protocol
```

### Pattern 2: Action Flow (Write Operations)
```
User: "Log 8 hours on Project Alpha"
  ↓
Agent validates input (project exists? hours valid?)
  ↓
Agent calls submit_timesheet tool
  ↓
Tool execution → Timesheet API → Factorial HR (mock) → Confirms submission
  ↓
Agent confirms → "I've logged 8 hours on Project Alpha for today"
  ↓
Audit log entry → Blob Storage (via API middleware)
```

### Pattern 3: Multi-Step Flow (Complex Operations)
```
User: "Copy last week's timesheet to this week"
  ↓
Agent calls get_timesheet_entries tool (last_week)
  ↓
Agent validates entries (are they still valid projects?)
  ↓
Agent asks for confirmation (shows preview via message)
  ↓
User confirms (new message in thread)
  ↓
Agent calls bulk_create_entries tool (this_week, entries)
  ↓
Returns summary with breakdown
```

---

## Security Considerations

1. **Authentication**: Microsoft Entra ID with OAuth 2.0
2. **Authorization**: Row-level security (users access only their data)
3. **Input Validation**: Sanitize all user input (prevent injection)
4. **Secrets Management**: Azure Key Vault for API keys, connection strings
5. **Audit Logging**: All operations logged with user ID, timestamp, action
6. **Content Safety**: Azure AI Content Filters prevent harmful outputs
7. **PII Protection**: Mask sensitive data in logs (employee IDs, payroll info)

---

### Decision 8: Frontend State Management

**Decision**: Use **Zustand** for React state management

**Rationale**:
- Minimal boilerplate compared to Redux (no actions, reducers, or providers)
- TypeScript-first design with excellent type inference
- Lightweight (1KB gzipped) with zero dependencies
- Works seamlessly with React Server Components and Suspense
- Direct access to state without Context API overhead
- Built-in devtools support for debugging
- Middleware support for persistence, immer, subscriptions

**Alternatives Considered**:
1. **Redux Toolkit** - More boilerplate, steeper learning curve, overkill for this app size
2. **React Context + useReducer** - Verbose, re-render issues, no built-in devtools
3. **Jotai/Recoil** - Atomic approach adds complexity for simple global state needs
4. **TanStack Query** - Excellent for server state, but we need client state management too

**Implementation Approach**:
```typescript
// stores/conversationStore.ts
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';

interface ConversationStore {
  threadId: string | null;
  messages: Message[];
  isLoading: boolean;
  setThreadId: (id: string) => void;
  addMessage: (message: Message) => void;
  setLoading: (loading: boolean) => void;
  clearConversation: () => void;
}

export const useConversationStore = create<ConversationStore>()(
  devtools(
    persist(
      (set) => ({
        threadId: null,
        messages: [],
        isLoading: false,
        setThreadId: (id) => set({ threadId: id }),
        addMessage: (message) => set((state) => ({
          messages: [...state.messages, message]
        })),
        setLoading: (loading) => set({ isLoading: loading }),
        clearConversation: () => set({
          threadId: null,
          messages: [],
          isLoading: false
        })
      }),
      { name: 'conversation-storage' }
    )
  )
);

// stores/userStore.ts - User profile and auth state
// stores/timesheetStore.ts - Timesheet data and filters
// stores/uiStore.ts - UI state (sidebar, modals, theme)
```

**Key Stores**:
1. **conversationStore**: Thread ID, messages, loading states
2. **userStore**: Employee profile, auth tokens, preferences
3. **timesheetStore**: Timesheet entries, date filters, project list
4. **uiStore**: Sidebar collapsed, active modal, theme preference

**Integration with CopilotKit**:
- CopilotKit manages AG-UI protocol communication
- Zustand stores app-level state (user profile, UI preferences, local timesheet cache)
- Event handlers update Zustand stores when agent completes tool calls
- Stores provide context to agent via CopilotKit's context providers

**Best Practices**:
- Keep stores focused (single responsibility)
- Use selectors to prevent unnecessary re-renders
- Combine with Immer middleware for immutable updates
- Persist auth and user preference stores to localStorage
- Use devtools middleware in development for debugging

---

## Performance Targets

| Metric | Target | Measurement |
|--------|--------|-------------|
| Agent response time | <2s (95th percentile) | Application Insights |
| API response time | <500ms (95th percentile) | Application Insights |
| Concurrent conversations | 100+ | Load testing |
| Intent resolution accuracy | >90% | Azure AI Evaluation |
| Tool call accuracy | >95% | Azure AI Evaluation |
| Code coverage | >80% | CI/CD pipeline |
| Uptime (business hours) | >99.5% | Azure Monitor |

---

## Deployment Strategy

### Local Development
```bash
# Prerequisites
dotnet workload install aspire
docker desktop running

# Start entire stack
cd backend/src/HRTimesheetAgent.AppHost
dotnet run
```

### Azure Production
```bash
# Deploy via Azure Developer CLI
azd init
azd up

# Resources provisioned:
# - Azure Container Apps (API, Agent Service, Frontend)
# - Azure Cosmos DB (conversation history)
# - Azure Blob Storage (audit logs)
# - Azure AI Foundry (LLM hosting)
# - Application Insights (monitoring)
# - Azure Key Vault (secrets)
```

---

## Open Questions & Risks

### Resolved
- ✅ AG-UI is a protocol for agent-user interaction → Use CopilotKit (AG-UI client) + shadcn/ui (components)
- ✅ Emulator strategy → MongoDB + Azurite for local dev via Aspire
- ✅ Testing framework → xUnit + Vitest + Playwright

### Outstanding
- ⚠️ **Factorial HR API mock**: Should we build a full mock or stub minimal responses?
  - **Recommendation**: Build minimal stub for phase 1, expand as needed
- ⚠️ **Multi-region deployment**: Required for 99.5% uptime or single region sufficient?
  - **Recommendation**: Single region for MVP, evaluate after load testing
- ⚠️ **Manager approval workflow**: In scope for initial release or future phase?
  - **Recommendation**: User Story 8 (corrections) is P8 priority, defer approval workflow to Phase 2

---

## Next Steps (Phase 1)

1. ✅ Document data model (Employee, TimesheetEntry, ConversationState) → `data-model.md`
2. ✅ Generate API contracts (OpenAPI spec) → `contracts/api.openapi.yaml`
3. ✅ Create quickstart guide (local setup) → `quickstart.md`
4. ✅ Update agent context → Run `.specify/scripts/bash/update-agent-context.sh copilot`

---

**Research completed**: 2025-12-29  
**Approved for Phase 1 design**: ✅
