# Quickstart Guide: HR Timesheet Agent

**Last Updated**: 2025-12-29  
**Prerequisites**: .NET 10 SDK, Node.js 20+, Docker Desktop  
**Estimated Setup Time**: 15-20 minutes

---

## Overview

This guide will help you set up the HR Timesheet Agent development environment on your local machine. The application uses:

- **.NET Aspire 13** for orchestration
- **MongoDB emulator** for conversation history (replaces Azure Cosmos DB)
- **Azure Storage Emulator (Azurite)** for audit logs (replaces Azure Blob Storage)
- **Backend API** (.NET 10) with AG-UI protocol server
- **Frontend** (Vite + React + CopilotKit + shadcn/ui + Zustand + Tailwind CSS)
- **Microsoft Agent Framework** with Azure AI Foundry
- **AG-UI protocol** for agent-user interaction

---

## Prerequisites

### Required Software

1. **.NET 10 SDK**
   ```bash
   # Download from: https://dotnet.microsoft.com/download/dotnet/10.0
   # Verify installation:
   dotnet --version  # Should show 10.0.x
   ```

2. **.NET Aspire Workload**
   ```bash
   dotnet workload install aspire
   ```

3. **Node.js 20+**
   ```bash
   # Download from: https://nodejs.org/
   # Verify installation:
   node --version  # Should show v20.x or higher
   npm --version   # Should show 10.x or higher
   ```

4. **Docker Desktop**
   ```bash
   # Download from: https://www.docker.com/products/docker-desktop
   # Ensure Docker is running:
   docker --version
   ```

### Optional (Recommended)

- **Visual Studio Code** with extensions:
  - C# Dev Kit
  - Aspire
  - ESLint
  - Prettier
  - REST Client (for testing API)

- **Azure CLI** (for production deployment):
  ```bash
  # Install: https://learn.microsoft.com/en-us/cli/azure/install-azure-cli
  az --version
  ```

---

## Project Structure

```
backend/
├── src/
│   ├── HRTimesheetAgent.API/           # ASP.NET Core Web API
│   ├── HRTimesheetAgent.Core/          # Domain logic
│   ├── HRTimesheetAgent.Infrastructure/ # External integrations
│   └── HRTimesheetAgent.AppHost/       # Aspire orchestration (START HERE)
└── tests/
    ├── HRTimesheetAgent.Tests.Contract/
    ├── HRTimesheetAgent.Tests.Integration/
    └── HRTimesheetAgent.Tests.Unit/

frontend/
├── src/
│   ├── components/
│   ├── pages/
│   ├── services/
│   ├── App.tsx
│   └── main.tsx
├── package.json
└── vite.config.ts
```

---

## Step-by-Step Setup

### Step 1: Clone Repository

```bash
git clone <repository-url>
cd hr-timesheet-agent
```

### Step 2: Start Docker Desktop

Ensure Docker Desktop is running (required for MongoDB and Azurite containers).

```bash
docker ps  # Should return without errors
```

### Step 3: Install Frontend Dependencies

```bash
cd frontend
npm install
cd ..
```

**Key packages installed**:
- `vite` - Build tool and dev server
- `react` + `react-dom` - UI framework
- `copilotkit` - AG-UI protocol client
- `shadcn/ui` - Accessible UI components (via CLI, not npm package)
- `@radix-ui/*` - Primitive components (shadcn foundation)
- `tailwindcss` - Utility-first CSS framework
- `zustand` - State management (lightweight, TypeScript-first)
- `@azure/msal-react` - Microsoft authentication
- `@azure/msal-browser` - MSAL browser support

### Step 4: Configure Environment

Create a local configuration file for the AppHost:

**File**: `backend/src/HRTimesheetAgent.AppHost/appsettings.Development.json`

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Aspire": "Information"
    }
  },
  "ConnectionStrings": {
    "mongodb": "mongodb://localhost:27017",
    "azurite": "UseDevelopmentStorage=true"
  },
  "AzureAI": {
    "UseLocal": true,
    "Endpoint": "http://localhost:11434",  // For local Ollama (optional)
    "ModelId": "llama2"
  }
}
```

### Step 5: Run the Application (All Services)

```bash
cd backend/src/HRTimesheetAgent.AppHost
dotnet run
```

**What happens**:
1. Aspire starts the orchestration dashboard
2. MongoDB container spins up (port 27017)
3. Azurite (Azure Storage Emulator) starts (blob: 10000, queue: 10001, table: 10002)
4. Backend API starts (https://localhost:7001)
5. Agent Service starts
6. Frontend dev server starts (http://localhost:5173)

**Access Points**:
- **Aspire Dashboard**: https://localhost:15888 (real-time telemetry)
- **Frontend**: http://localhost:5173 (React app)
- **Backend API**: https://localhost:7001 (Swagger UI at /swagger)
- **MongoDB**: localhost:27017
- **Azurite Blob**: localhost:10000

### Step 6: Verify Services

Open the **Aspire Dashboard** at https://localhost:15888

You should see:
- ✅ `mongodb` - Running
- ✅ `storage` (Azurite) - Running
- ✅ `api` - Running
- ✅ `agent-service` - Running
- ✅ `frontend` - Running

All services should show green health checks.

### Step 7: Seed Initial Data (Optional)

To populate test employees and projects:

```bash
cd backend/src/HRTimesheetAgent.API
dotnet run --seed-data
```

This creates:
- Test employee: `john.doe@company.com` (password: `Test@123`)
- Sample projects: `PROJ-ALPHA`, `PROJ-BETA`, `ADMIN`, `MEETING`

---

## Using the Application

### Access the Chat Interface

1. Open http://localhost:5173 in your browser
2. Sign in with test credentials (or skip auth in dev mode)
3. Start chatting with the agent:
   - "Show me my hours for today"
   - "Log 8 hours on Project Alpha"
   - "Clock me in"

### Test the API Directly

Use the Swagger UI at https://localhost:7001/swagger or a REST client:

**Example: Get Today's Timesheet**
```http
GET https://localhost:7001/api/timesheets/daily/2025-12-29
Authorization: Bearer <token>  # Or skip in dev mode
```

**Example: Submit Timesheet Entry**
```http
POST https://localhost:7001/api/timesheets
Content-Type: application/json
Authorization: Bearer <token>

{
  "date": "2025-12-29",
  "hours": 8.0,
  "projectCode": "PROJ-ALPHA",
  "description": "Worked on feature implementation"
}
```

---

## Development Workflow

### Hot Reload

- **Backend**: Code changes trigger automatic rebuild (dotnet watch)
- **Frontend**: Vite provides instant hot module replacement (HMR)

### Debugging

**Option 1: Visual Studio Code**
1. Open root folder in VS Code
2. Select "Debug Aspire App" from Run and Debug panel
3. Set breakpoints in any service
4. Press F5

**Option 2: Visual Studio 2022**
1. Open `backend/HRTimesheetAgent.sln`
2. Set `HRTimesheetAgent.AppHost` as startup project
3. Press F5
4. Aspire dashboard opens automatically

### View Logs

**Aspire Dashboard** (Recommended):
- Navigate to https://localhost:15888
- Click on any service to see:
  - Console logs
  - Distributed traces
  - Metrics
  - Environment variables

**Container Logs**:
```bash
# MongoDB logs
docker logs <mongodb-container-id>

# Azurite logs
docker logs <azurite-container-id>
```

### Database Access

**MongoDB** (Conversation History):
```bash
# Using MongoDB Compass (GUI)
# Connect to: mongodb://localhost:27017

# Or using mongosh CLI
mongosh mongodb://localhost:27017
use timesheets
db.conversationThreads.find()
```

**Azure Storage Emulator** (Audit Logs):
```bash
# Using Azure Storage Explorer
# Connect to: (Local) - Attach to local emulator

# Or using Azure CLI
az storage blob list \
  --account-name devstoreaccount1 \
  --container-name audit-logs \
  --connection-string "UseDevelopmentStorage=true"
```

---

## Testing

### Run Unit Tests

```bash
cd backend/tests/HRTimesheetAgent.Tests.Unit
dotnet test
```

### Run Integration Tests

```bash
cd backend/tests/HRTimesheetAgent.Tests.Integration
dotnet test
```

### Run Contract Tests

```bash
cd backend/tests/HRTimesheetAgent.Tests.Contract
dotnet test
```

### Run Frontend Tests

```bash
cd frontend
npm test              # Vitest unit tests
npm run test:e2e      # Playwright E2E tests
```

### Run All Tests

```bash
# From repository root
dotnet test ./backend/tests/
npm --prefix frontend test
```

---

## Common Issues & Troubleshooting

### Issue: Port Already in Use

**Error**: "Address already in use: 5173" or "7001"

**Solution**:
```bash
# Kill process on port
lsof -ti:5173 | xargs kill -9  # Mac/Linux
netstat -ano | findstr :5173   # Windows (find PID, then taskkill /PID <pid>)
```

### Issue: MongoDB Container Fails to Start

**Error**: "Container exited with code 1"

**Solution**:
```bash
# Remove existing container and volumes
docker rm -f <mongodb-container-id>
docker volume prune -f

# Restart Aspire app
cd backend/src/HRTimesheetAgent.AppHost
dotnet run
```

### Issue: Frontend Can't Connect to API

**Error**: "Network error" or "CORS policy blocked"

**Solution**:
1. Verify API is running: https://localhost:7001/swagger
2. Check Aspire dashboard for API health
3. Ensure frontend has correct API URL in `.env.development`:
   ```env
   VITE_API_URL=https://localhost:7001
   ```
4. Restart frontend:
   ```bash
   cd frontend
   npm run dev
   ```

### Issue: Authentication Fails

**Error**: "Unauthorized" or "Invalid token"

**Solution**:
For local development, authentication can be disabled:

**File**: `backend/src/HRTimesheetAgent.API/Program.cs`
```csharp
// Comment out authentication in Development environment
if (!app.Environment.IsDevelopment())
{
    app.UseAuthentication();
    app.UseAuthorization();
}
```

### Issue: Aspire Dashboard Not Opening

**Error**: Dashboard URL (https://localhost:15888) not accessible

**Solution**:
```bash
# Check if dashboard is running
dotnet aspire dashboard

# Or manually specify dashboard URL
cd backend/src/HRTimesheetAgent.AppHost
dotnet run --dashboard-url https://localhost:15888
```

---

## Environment Variables Reference

### Backend API (`HRTimesheetAgent.API`)

| Variable | Description | Default (Dev) |
|----------|-------------|---------------|
| `ConnectionStrings__mongodb` | MongoDB connection string | `mongodb://localhost:27017` |
| `ConnectionStrings__storage` | Azure Storage connection | `UseDevelopmentStorage=true` |
| `AzureAd__TenantId` | Entra ID tenant | `(skip in dev)` |
| `AzureAd__ClientId` | OAuth client ID | `(skip in dev)` |
| `AzureAI__Endpoint` | AI Foundry endpoint | `http://localhost:11434` |
| `AzureAI__ModelId` | LLM model | `llama2` |
| `ASPNETCORE_ENVIRONMENT` | Environment name | `Development` |

### Frontend (Vite React)

| Variable | Description | Default (Dev) |
|----------|-------------|---------------|
| `VITE_API_URL` | Backend API base URL | `https://localhost:7001` |
| `VITE_CLIENT_ID` | OAuth client ID | `(skip in dev)` |
| `VITE_TENANT_ID` | Entra ID tenant | `(skip in dev)` |
| `VITE_ENV` | Environment name | `development` |

**File**: `frontend/.env.development`
```env
VITE_API_URL=https://localhost:7001
VITE_ENV=development
```

---

## Stopping the Application

Press **Ctrl+C** in the terminal running `dotnet run` from AppHost.

This will:
1. Stop all services
2. Stop containers (MongoDB, Azurite)
3. Clean up resources

**Note**: Containers remain stopped but not removed. Data persists between runs.

To fully clean up:
```bash
docker rm -f $(docker ps -aq)  # Remove all containers
docker volume prune -f         # Remove all volumes (WARNING: deletes data)
```

---

## Next Steps

1. ✅ Review the [Data Model](data-model.md) to understand entities
2. ✅ Read the [API Contracts](contracts/api.openapi.yaml) for endpoint specifications
3. ✅ Explore the [Research Document](research.md) for architecture decisions
4. 🚧 Create tasks (run `/speckit.tasks` command)
5. 🚧 Start implementing user stories (P1 → P8)

---

## Additional Resources

- [.NET Aspire Documentation](https://learn.microsoft.com/en-us/dotnet/aspire/)
- [Microsoft Agent Framework Guide](https://learn.microsoft.com/en-us/azure/ai-services/agents/)
- [AG-UI Protocol Documentation](https://docs.ag-ui.com/introduction)
- [CopilotKit (AG-UI Client)](https://docs.copilotkit.ai/)
- [shadcn/ui Documentation](https://ui.shadcn.com/)
- [Radix UI Primitives](https://www.radix-ui.com/primitives)
- [Tailwind CSS Documentation](https://tailwindcss.com/docs)
- [Azure AI Foundry](https://learn.microsoft.com/en-us/azure/ai-foundry/)
- [Vite Documentation](https://vitejs.dev/)

---

**Need Help?**
- Check the [Aspire Dashboard](https://localhost:15888) for real-time diagnostics
- Review logs in the dashboard (Logs tab for each service)
- Verify all prerequisites are installed correctly
- Ensure Docker Desktop is running

**Ready to build?** Start with User Story 1 (Priority P1): View Today's Timesheet via Natural Conversation.
