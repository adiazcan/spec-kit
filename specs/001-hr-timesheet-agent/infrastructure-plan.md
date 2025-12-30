# Infrastructure Plan: Bicep IaC Modules for HR Timesheet Agent

**Phase**: Infrastructure as Code  
**Date**: 2025-12-30  
**Purpose**: Define Azure infrastructure modules using Bicep for deploying the HR Timesheet Agent

---

## Overview

This document outlines the Bicep modules required to deploy the HR Timesheet Agent to Azure. The infrastructure follows a modular approach with reusable Bicep modules for each Azure service component. The deployment supports both development/staging and production environments with appropriate scaling and security configurations.

---

## Architecture Summary

### Azure Services Required

Based on the spec and data model, the application requires:

1. **Compute**: Azure Container Apps (backend API + frontend SPA)
2. **Data Storage**: Azure Cosmos DB (DocumentDB API) for conversation history and timesheet data
3. **Blob Storage**: Azure Storage Account for audit logs
4. **AI/LLM**: Azure AI Foundry for language model hosting
5. **Authentication**: Microsoft Entra ID (Azure AD) - existing service, configuration only
6. **Secrets Management**: Azure Key Vault for sensitive configuration
7. **Monitoring**: Application Insights + Log Analytics Workspace
8. **Networking**: Virtual Network with private endpoints (production)
9. **Container Registry**: Azure Container Registry for Docker images

### Environment Strategy

- **Development**: Shared resources, reduced SKUs, public endpoints
- **Production**: Dedicated resources, premium SKUs, private endpoints, zone redundancy

---

## Bicep Module Structure

```
azure-deployment/
├── main.bicep                          # Main orchestration template
├── parameters/
│   ├── dev.bicepparam                  # Development parameters
│   ├── staging.bicepparam              # Staging parameters
│   └── prod.bicepparam                 # Production parameters
├── modules/
│   ├── core/
│   │   ├── resource-group.bicep        # Resource group creation
│   │   ├── managed-identity.bicep      # User-assigned managed identity
│   │   └── naming.bicep                # Naming convention module
│   ├── networking/
│   │   ├── vnet.bicep                  # Virtual network
│   │   ├── private-endpoint.bicep      # Private endpoint (reusable)
│   │   └── subnet.bicep                # Subnet configuration
│   ├── storage/
│   │   ├── cosmos-db.bicep             # Cosmos DB account + databases
│   │   ├── storage-account.bicep       # Blob storage for audit logs
│   │   └── container-registry.bicep    # Azure Container Registry
│   ├── compute/
│   │   ├── container-apps-env.bicep    # Container Apps environment
│   │   ├── container-app.bicep         # Individual container app (reusable)
│   │   └── app-config.bicep            # Application configuration
│   ├── ai/
│   │   ├── ai-foundry.bicep            # Azure AI Foundry workspace
│   │   └── openai-deployment.bicep     # OpenAI model deployment
│   ├── security/
│   │   ├── key-vault.bicep             # Azure Key Vault
│   │   ├── key-vault-secret.bicep      # Key Vault secret (reusable)
│   │   └── rbac.bicep                  # Role assignments
│   └── monitoring/
│       ├── log-analytics.bicep         # Log Analytics workspace
│       ├── app-insights.bicep          # Application Insights
│       └── alerts.bicep                # Monitoring alerts
└── scripts/
    ├── deploy.sh                       # Deployment script
    ├── validate.sh                     # Pre-deployment validation
    └── teardown.sh                     # Resource cleanup
```

---

## Module Specifications

### 1. Main Orchestration (main.bicep)

**Purpose**: Orchestrate all infrastructure components and manage dependencies

**Parameters**:
```bicep
@description('The environment name (dev, staging, prod)')
param environmentName string

@description('The Azure region for resource deployment')
param location string = resourceGroup().location

@description('Application name prefix for resource naming')
param appName string = 'hr-timesheet-agent'

@description('Enable private endpoints for production security')
param enablePrivateEndpoints bool = false

@description('Enable zone redundancy for high availability')
param enableZoneRedundancy bool = false

@description('Tags to apply to all resources')
param tags object = {
  Environment: environmentName
  Application: appName
  ManagedBy: 'Bicep'
}
```

**Resource Deployment Order**:
1. Naming conventions (determine resource names)
2. Managed identity (for secure service-to-service auth)
3. Networking (VNet, subnets) - if private endpoints enabled
4. Key Vault (for secrets management)
5. Log Analytics + Application Insights (for monitoring)
6. Container Registry (for Docker images)
7. Cosmos DB (for data storage)
8. Storage Account (for audit logs)
9. Azure AI Foundry (for LLM hosting)
10. Container Apps Environment
11. Container Apps (backend + frontend)
12. RBAC assignments (grant permissions)
13. Monitoring alerts

**Outputs**:
```bicep
output resourceGroupName string
output containerAppsEnvironmentId string
output cosmosDbEndpoint string
output storageAccountName string
output keyVaultName string
output appInsightsInstrumentationKey string
output backendAppUrl string
output frontendAppUrl string
```

---

### 2. Core Modules

#### 2.1 Resource Group (modules/core/resource-group.bicep)

**Purpose**: Create or reference resource group

**Parameters**:
- `resourceGroupName`: Name of resource group
- `location`: Azure region
- `tags`: Resource tags

**Outputs**:
- `id`: Resource group ID
- `name`: Resource group name

---

#### 2.2 Managed Identity (modules/core/managed-identity.bicep)

**Purpose**: Create user-assigned managed identity for service authentication

**Parameters**:
- `identityName`: Identity name
- `location`: Azure region
- `tags`: Resource tags

**Outputs**:
- `identityId`: Managed identity resource ID
- `principalId`: Principal ID for RBAC assignments
- `clientId`: Client ID for application configuration

**Usage**: Backend Container App uses this identity to access:
- Cosmos DB (data plane operations)
- Storage Account (audit log writes)
- Key Vault (secret retrieval)
- Azure AI Foundry (LLM API calls)

---

#### 2.3 Naming Convention (modules/core/naming.bicep)

**Purpose**: Generate consistent resource names following Azure naming conventions

**Parameters**:
- `appName`: Application prefix
- `environmentName`: Environment suffix
- `location`: Azure region (for location codes)

**Outputs**:
```bicep
output cosmosDbAccountName string  // e.g., 'cosmos-hrtimesheet-dev-eastus'
output storageAccountName string   // e.g., 'sthrtimesheetdeveus' (no hyphens)
output keyVaultName string         // e.g., 'kv-hrtimesheet-dev'
output containerRegistryName string // e.g., 'acrhrtimesheetdev'
output logAnalyticsName string     // e.g., 'log-hrtimesheet-dev'
output appInsightsName string      // e.g., 'appi-hrtimesheet-dev'
output containerAppsEnvName string // e.g., 'cae-hrtimesheet-dev'
output backendAppName string       // e.g., 'ca-hrtimesheet-api-dev'
output frontendAppName string      // e.g., 'ca-hrtimesheet-web-dev'
```

**Naming Rules**:
- Follow Azure resource naming conventions (length limits, allowed characters)
- Include environment suffix for resource isolation
- Use location abbreviations (eastus → 'eus', westus2 → 'wus2')

---

### 3. Networking Modules

#### 3.1 Virtual Network (modules/networking/vnet.bicep)

**Purpose**: Create VNet for private networking (production only)

**Parameters**:
- `vnetName`: Virtual network name
- `location`: Azure region
- `addressPrefix`: CIDR block (e.g., '10.0.0.0/16')
- `subnets`: Array of subnet configurations
- `tags`: Resource tags

**Subnets**:
1. **Container Apps Subnet**: For Container Apps environment (requires delegation)
   - Address space: `10.0.0.0/23` (512 IPs)
   - Delegation: `Microsoft.App/environments`
2. **Private Endpoints Subnet**: For private endpoints to Cosmos DB, Storage, Key Vault
   - Address space: `10.0.2.0/24` (256 IPs)
   - Disable private endpoint network policies

**Outputs**:
- `vnetId`: Virtual network resource ID
- `containerAppsSubnetId`: Subnet ID for Container Apps
- `privateEndpointsSubnetId`: Subnet ID for private endpoints

**Conditional**: Deploy only if `enablePrivateEndpoints = true` (production)

---

#### 3.2 Private Endpoint (modules/networking/private-endpoint.bicep)

**Purpose**: Reusable module for creating private endpoints

**Parameters**:
- `privateEndpointName`: Endpoint name
- `location`: Azure region
- `subnetId`: Target subnet ID
- `privateLinkServiceId`: Resource ID to connect (Cosmos DB, Storage, etc.)
- `groupIds`: Private link subresources (e.g., ['Sql'] for Cosmos DB)
- `privateDnsZoneName`: DNS zone for name resolution
- `tags`: Resource tags

**Outputs**:
- `privateEndpointId`: Private endpoint resource ID

**Usage**: Create separate instances for:
- Cosmos DB (`groupIds: ['Sql']`)
- Storage Account (`groupIds: ['blob']`)
- Key Vault (`groupIds: ['vault']`)
- Container Registry (`groupIds: ['registry']`)

---

### 4. Storage Modules

#### 4.1 Cosmos DB (modules/storage/cosmos-db.bicep)

**Purpose**: Deploy Cosmos DB account with DocumentDB API

**Parameters**:
```bicep
param accountName string
param location string
param consistencyLevel string = 'Session' // Session, BoundedStaleness, Strong
param enableAutomaticFailover bool = true
param enableFreeTier bool = false
param maxTotalThroughput int = 4000 // Max RU/s for database
param enableServerless bool = false // Dev: serverless, Prod: provisioned
param databases array = [
  {
    name: 'hr-timesheet-db'
    containers: [
      {
        name: 'employees'
        partitionKey: '/id'
        throughput: 400 // RU/s (null for serverless)
      }
      {
        name: 'timesheetEntries'
        partitionKey: '/employeeId'
        throughput: 1000
      }
      {
        name: 'clockEvents'
        partitionKey: '/employeeId'
        throughput: 400
      }
      {
        name: 'conversationThreads'
        partitionKey: '/employeeId'
        throughput: 400
        defaultTtl: 7776000 // 90 days
      }
      {
        name: 'conversationMessages'
        partitionKey: '/threadId'
        throughput: 600
      }
    ]
  }
]
```

**Indexes**:
- Default indexing policy (all paths)
- Composite indexes for:
  - `timesheetEntries`: `[employeeId ASC, date DESC]`
  - `clockEvents`: `[employeeId ASC, date DESC, timestamp DESC]`
  - `conversationMessages`: `[threadId ASC, timestamp ASC]`

**Backup**:
- **Dev/Staging**: Continuous backup (7-day retention)
- **Production**: Continuous backup (30-day retention), geo-redundant

**Outputs**:
- `accountId`: Cosmos DB account ID
- `accountEndpoint`: Connection endpoint
- `databaseName`: Primary database name
- `connectionStringSecretName`: Key Vault secret name for connection string

**RBAC**: Grant managed identity `Cosmos DB Built-in Data Contributor` role

---

#### 4.2 Storage Account (modules/storage/storage-account.bicep)

**Purpose**: Blob storage for audit logs

**Parameters**:
```bicep
param storageAccountName string
param location string
param sku string = 'Standard_LRS' // Dev: LRS, Prod: ZRS or GRS
param kind string = 'StorageV2'
param enableHierarchicalNamespace bool = false // ADLS Gen2 for query
param containers array = [
  {
    name: 'audit-logs'
    publicAccess: 'None'
  }
]
```

**Lifecycle Management**:
```bicep
lifecycleRules: [
  {
    name: 'archive-old-logs'
    enabled: true
    definition: {
      filters: {
        blobTypes: ['blockBlob']
        prefixMatch: ['audit-logs/']
      }
      actions: {
        baseBlob: {
          tierToCool: {
            daysAfterModificationGreaterThan: 90
          }
          tierToArchive: {
            daysAfterModificationGreaterThan: 365
          }
          delete: {
            daysAfterModificationGreaterThan: 2555 // ~7 years
          }
        }
      }
    }
  }
]
```

**Outputs**:
- `storageAccountId`: Storage account ID
- `blobEndpoint`: Blob service endpoint
- `connectionStringSecretName`: Key Vault secret name

**RBAC**: Grant managed identity `Storage Blob Data Contributor` role

---

#### 4.3 Container Registry (modules/storage/container-registry.bicep)

**Purpose**: Host Docker images for backend and frontend

**Parameters**:
```bicep
param registryName string
param location string
param sku string = 'Basic' // Dev: Basic, Prod: Premium (with geo-replication)
param adminUserEnabled bool = false // Use managed identity instead
param enableZoneRedundancy bool = false // Production only
```

**Outputs**:
- `registryId`: ACR resource ID
- `loginServer`: Registry login URL

**RBAC**: Grant managed identity `AcrPull` role (for Container Apps)

---

### 5. Compute Modules

#### 5.1 Container Apps Environment (modules/compute/container-apps-env.bicep)

**Purpose**: Create managed environment for Container Apps

**Parameters**:
```bicep
param environmentName string
param location string
param logAnalyticsWorkspaceId string
param vnetSubnetId string = '' // Empty for dev (external ingress)
param daprEnabled bool = false
param zoneRedundant bool = false // Production only
```

**Configuration**:
- **Ingress**: External (public internet access for frontend)
- **Dapr**: Disabled (not using service mesh for phase 1)
- **Workload profiles**: Consumption plan (dev), Dedicated (production)

**Outputs**:
- `environmentId`: Container Apps environment ID
- `defaultDomain`: Default ingress domain

---

#### 5.2 Container App (modules/compute/container-app.bicep)

**Purpose**: Reusable module for deploying container apps

**Parameters**:
```bicep
param appName string
param location string
param containerAppsEnvironmentId string
param managedIdentityId string
param containerImage string // From ACR
param containerPort int
param environmentVariables array
param secrets array
param minReplicas int = 1
param maxReplicas int = 10
param cpu string = '0.5' // vCPU cores
param memory string = '1Gi'
param ingressEnabled bool = true
param externalIngress bool = false
```

**Health Probes**:
```bicep
probes: [
  {
    type: 'liveness'
    httpGet: {
      path: '/health/live'
      port: containerPort
    }
    initialDelaySeconds: 10
    periodSeconds: 30
  }
  {
    type: 'readiness'
    httpGet: {
      path: '/health/ready'
      port: containerPort
    }
    initialDelaySeconds: 5
    periodSeconds: 10
  }
]
```

**Scaling Rules**:
```bicep
scale: {
  minReplicas: minReplicas
  maxReplicas: maxReplicas
  rules: [
    {
      name: 'http-scaling'
      http: {
        metadata: {
          concurrentRequests: '100' // Scale when >100 requests/replica
        }
      }
    }
  ]
}
```

**Outputs**:
- `appId`: Container App resource ID
- `appFqdn`: Fully qualified domain name

---

#### 5.3 Backend Container App Configuration

**Instance**: `modules/compute/container-app.bicep` (backend-specific parameters)

**Container**: `hrtimesheetagent-api:latest` (from ACR)

**Environment Variables**:
```bicep
environmentVariables: [
  // Cosmos DB
  { name: 'CosmosDb__Endpoint', value: cosmosDbEndpoint }
  { name: 'CosmosDb__Database', value: 'hr-timesheet-db' }
  { name: 'CosmosDb__UseConnectionString', value: 'false' } // Use managed identity
  
  // Storage Account
  { name: 'AuditLogs__StorageAccountName', value: storageAccountName }
  { name: 'AuditLogs__ContainerName', value: 'audit-logs' }
  
  // Azure AI Foundry
  { name: 'AzureAI__Endpoint', secretRef: 'azure-ai-endpoint' }
  { name: 'AzureAI__DeploymentName', value: 'gpt-4o' }
  
  // Authentication
  { name: 'AzureAd__TenantId', value: tenantId }
  { name: 'AzureAd__ClientId', value: backendClientId }
  { name: 'AzureAd__Audience', value: backendAudience }
  
  // Application Insights
  { name: 'APPLICATIONINSIGHTS_CONNECTION_STRING', secretRef: 'app-insights-connection-string' }
  
  // Managed Identity
  { name: 'AZURE_CLIENT_ID', value: managedIdentityClientId }
]
```

**Secrets** (from Key Vault):
- `azure-ai-endpoint`: Azure AI Foundry endpoint URL
- `app-insights-connection-string`: Application Insights connection string

**Ingress**:
- External: `false` (internal only, called by frontend)
- Target port: `8080`
- CORS: Allow frontend domain

**Resources**:
- Dev: `0.5 vCPU, 1Gi memory, 1-3 replicas`
- Prod: `2 vCPU, 4Gi memory, 3-10 replicas`

---

#### 5.4 Frontend Container App Configuration

**Instance**: `modules/compute/container-app.bicep` (frontend-specific parameters)

**Container**: `hrtimesheetagent-web:latest` (from ACR)

**Environment Variables**:
```bicep
environmentVariables: [
  // Backend API
  { name: 'VITE_API_URL', value: backendAppUrl }
  
  // Authentication
  { name: 'VITE_ENTRA_TENANT_ID', value: tenantId }
  { name: 'VITE_ENTRA_CLIENT_ID', value: frontendClientId }
  { name: 'VITE_ENTRA_REDIRECT_URI', value: frontendAppUrl }
  
  // Application Insights (frontend telemetry)
  { name: 'VITE_APP_INSIGHTS_CONNECTION_STRING', secretRef: 'app-insights-connection-string' }
]
```

**Ingress**:
- External: `true` (public internet access)
- Target port: `80` (Nginx serving React SPA)
- Custom domain: Optional (e.g., `timesheet.company.com`)

**Resources**:
- Dev: `0.25 vCPU, 0.5Gi memory, 1-2 replicas`
- Prod: `0.5 vCPU, 1Gi memory, 2-5 replicas`

---

### 6. AI Modules

#### 6.1 Azure AI Foundry (modules/ai/ai-foundry.bicep)

**Purpose**: Create AI Foundry workspace for LLM hosting

**Parameters**:
```bicep
param workspaceName string
param location string
param managedIdentityId string
param storageAccountId string // For prompt caching
param keyVaultId string
param logAnalyticsWorkspaceId string
```

**Model Deployment**:
- Model: `gpt-4o` (GPT-4 Omni)
- Capacity: 10K TPM (tokens per minute) for dev, 100K TPM for prod
- Content filtering: Enabled (default policies)

**Outputs**:
- `workspaceId`: AI Foundry workspace ID
- `endpoint`: Model inference endpoint
- `deploymentName`: Deployed model name

**RBAC**: Grant managed identity `Cognitive Services OpenAI User` role

---

#### 6.2 OpenAI Deployment (modules/ai/openai-deployment.bicep)

**Purpose**: Deploy specific OpenAI model to AI Foundry workspace

**Parameters**:
```bicep
param workspaceId string
param deploymentName string = 'gpt-4o'
param modelName string = 'gpt-4o' // Model identifier
param modelVersion string = '2024-05-13' // Latest stable version
param sku object = {
  name: 'Standard'
  capacity: 10 // TPM in thousands
}
```

**Outputs**:
- `deploymentId`: Deployment resource ID

---

### 7. Security Modules

#### 7.1 Key Vault (modules/security/key-vault.bicep)

**Purpose**: Secure secrets management

**Parameters**:
```bicep
param keyVaultName string
param location string
param managedIdentityPrincipalId string
param enableSoftDelete bool = true
param softDeleteRetentionDays int = 90
param enablePurgeProtection bool = true // Production only
param enableRbacAuthorization bool = true // Use Azure RBAC instead of access policies
param sku string = 'standard' // Standard or Premium (HSM)
```

**Secrets to Store**:
1. `cosmos-db-connection-string`: Cosmos DB primary connection string
2. `storage-account-connection-string`: Storage account connection string
3. `azure-ai-endpoint`: Azure AI Foundry endpoint URL
4. `app-insights-connection-string`: Application Insights connection string
5. `factorial-hr-api-key`: Factorial HR API key (mocked for phase 1)

**Network Rules** (Production):
- Default action: `Deny`
- Allow access from: Container Apps subnet, deployment agent IP
- Allow trusted Azure services: `true`

**Outputs**:
- `keyVaultId`: Key Vault resource ID
- `keyVaultUri`: Key Vault URI for SDK access

**RBAC**: Grant managed identity `Key Vault Secrets User` role

---

#### 7.2 Key Vault Secret (modules/security/key-vault-secret.bicep)

**Purpose**: Reusable module for adding secrets to Key Vault

**Parameters**:
```bicep
param keyVaultName string
param secretName string
@secure()
param secretValue string
param contentType string = 'text/plain'
param expirationDate string = '' // Optional expiry
```

**Outputs**:
- `secretId`: Secret resource ID
- `secretUri`: Secret URI (with version)

---

#### 7.3 RBAC Assignments (modules/security/rbac.bicep)

**Purpose**: Grant managed identity access to Azure resources

**Parameters**:
```bicep
param principalId string // Managed identity principal ID
param roleDefinitions array = [
  {
    scope: string // Resource ID
    roleDefinitionId: string // Built-in role ID
  }
]
```

**Role Assignments**:
```bicep
// Cosmos DB
{
  scope: cosmosDbAccountId
  roleDefinitionId: '00000000-0000-0000-0000-000000000002' // Cosmos DB Data Contributor
}

// Storage Account
{
  scope: storageAccountId
  roleDefinitionId: 'ba92f5b4-2d11-453d-a403-e96b0029c9fe' // Storage Blob Data Contributor
}

// Key Vault
{
  scope: keyVaultId
  roleDefinitionId: '4633458b-17de-408a-b874-0445c86b69e6' // Key Vault Secrets User
}

// Container Registry
{
  scope: containerRegistryId
  roleDefinitionId: '7f951dda-4ed3-4680-a7ca-43fe172d538d' // AcrPull
}

// Azure AI Foundry
{
  scope: aiFoundryWorkspaceId
  roleDefinitionId: '5e0bd9bd-7b93-4f28-af87-19fc36ad61bd' // Cognitive Services OpenAI User
}
```

---

### 8. Monitoring Modules

#### 8.1 Log Analytics Workspace (modules/monitoring/log-analytics.bicep)

**Purpose**: Centralized logging and diagnostics

**Parameters**:
```bicep
param workspaceName string
param location string
param sku string = 'PerGB2018'
param retentionInDays int = 30 // Dev: 30, Prod: 90
param dailyQuotaGb int = 1 // Max daily ingestion
```

**Data Sources**:
- Container Apps logs (stdout/stderr)
- Application Insights telemetry
- Azure resource diagnostic logs

**Outputs**:
- `workspaceId`: Log Analytics workspace ID
- `workspaceKey`: Workspace shared key (for agents)

---

#### 8.2 Application Insights (modules/monitoring/app-insights.bicep)

**Purpose**: Application performance monitoring and telemetry

**Parameters**:
```bicep
param appInsightsName string
param location string
param logAnalyticsWorkspaceId string
param applicationType string = 'web'
param samplingPercentage int = 100 // Dev: 100%, Prod: 10-20% (cost optimization)
```

**Features**:
- Distributed tracing (correlation IDs per FR-038)
- Custom metrics (agent response times, timesheet submissions)
- Failure detection (exceptions, errors)
- Live metrics stream

**Outputs**:
- `appInsightsId`: Application Insights resource ID
- `instrumentationKey`: Instrumentation key (legacy)
- `connectionString`: Connection string (modern)

---

#### 8.3 Monitoring Alerts (modules/monitoring/alerts.bicep)

**Purpose**: Proactive monitoring and alerting

**Parameters**:
```bicep
param alertName string
param severity int = 2 // 0=Critical, 1=Error, 2=Warning, 3=Info
param targetResourceId string
param metricName string
param operator string = 'GreaterThan'
param threshold int
param evaluationFrequency string = 'PT1M' // ISO 8601 duration
param windowSize string = 'PT5M'
param actionGroupId string // Email/SMS notification target
```

**Alert Rules**:
1. **High Response Time**: Backend API p95 > 2000ms (per SC-006)
2. **High Error Rate**: Error rate > 5% over 5 minutes
3. **Low Availability**: Availability < 99.5% (per SC-010)
4. **High CPU Usage**: Container Apps CPU > 80%
5. **High Memory Usage**: Container Apps memory > 90%
6. **Cosmos DB Throttling**: RU/s throttling rate > 10%
7. **Failed Authentication**: Failed auth attempts > 10 in 1 minute

**Action Groups**:
- Email: DevOps team
- SMS: On-call engineer (production only)
- Webhook: Integrate with incident management (PagerDuty, etc.)

---

## Deployment Parameters

### Development Environment (dev.bicepparam)

```bicep
using './main.bicep'

param environmentName = 'dev'
param location = 'eastus'
param appName = 'hr-timesheet-agent'
param enablePrivateEndpoints = false // Public endpoints for easy dev access
param enableZoneRedundancy = false

// Cost optimization
param cosmosDbServerless = true // Serverless mode
param storageAccountSku = 'Standard_LRS'
param containerRegistrySku = 'Basic'
param aiFoundryCapacity = 10 // 10K TPM

// Scaling
param backendMinReplicas = 1
param backendMaxReplicas = 3
param frontendMinReplicas = 1
param frontendMaxReplicas = 2
```

---

### Production Environment (prod.bicepparam)

```bicep
using './main.bicep'

param environmentName = 'prod'
param location = 'eastus'
param secondaryLocation = 'westus2' // Geo-redundancy
param appName = 'hr-timesheet-agent'
param enablePrivateEndpoints = true // Private endpoints for security
param enableZoneRedundancy = true // Zone redundancy for HA

// High availability
param cosmosDbServerless = false // Provisioned throughput
param cosmosDbAutomaticFailover = true
param storageAccountSku = 'Standard_ZRS' // Zone-redundant
param containerRegistrySku = 'Premium' // Geo-replication
param aiFoundryCapacity = 100 // 100K TPM

// Scaling
param backendMinReplicas = 3
param backendMaxReplicas = 10
param frontendMinReplicas = 2
param frontendMaxReplicas = 5

// Backup
param cosmosDbBackupRetentionDays = 30
param enableCosmosContinuousBackup = true
```

---

## Deployment Scripts

### deploy.sh

```bash
#!/bin/bash
# Deploy infrastructure to Azure

set -e

# Parameters
ENVIRONMENT=$1
RESOURCE_GROUP="rg-hr-timesheet-${ENVIRONMENT}"
LOCATION="eastus"
SUBSCRIPTION_ID="<your-subscription-id>"

if [ -z "$ENVIRONMENT" ]; then
  echo "Usage: ./deploy.sh [dev|staging|prod]"
  exit 1
fi

# Authenticate
az login
az account set --subscription "$SUBSCRIPTION_ID"

# Create resource group
echo "Creating resource group..."
az group create --name "$RESOURCE_GROUP" --location "$LOCATION"

# Validate Bicep template
echo "Validating Bicep template..."
az deployment group validate \
  --resource-group "$RESOURCE_GROUP" \
  --template-file main.bicep \
  --parameters "parameters/${ENVIRONMENT}.bicepparam"

# Deploy infrastructure
echo "Deploying infrastructure..."
az deployment group create \
  --resource-group "$RESOURCE_GROUP" \
  --template-file main.bicep \
  --parameters "parameters/${ENVIRONMENT}.bicepparam" \
  --mode Incremental \
  --name "deploy-$(date +%Y%m%d-%H%M%S)"

# Output deployment results
echo "Deployment complete!"
az deployment group show \
  --resource-group "$RESOURCE_GROUP" \
  --name "deploy-$(date +%Y%m%d-%H%M%S)" \
  --query properties.outputs
```

---

### validate.sh

```bash
#!/bin/bash
# Pre-deployment validation

set -e

ENVIRONMENT=$1

# Check Azure CLI
if ! command -v az &> /dev/null; then
  echo "Azure CLI not found. Install from https://aka.ms/azure-cli"
  exit 1
fi

# Check Bicep
if ! az bicep version &> /dev/null; then
  echo "Installing Bicep..."
  az bicep install
fi

# Lint Bicep files
echo "Linting Bicep files..."
find . -name "*.bicep" -exec az bicep build --file {} \;

# Check parameter file exists
if [ ! -f "parameters/${ENVIRONMENT}.bicepparam" ]; then
  echo "Parameter file not found: parameters/${ENVIRONMENT}.bicepparam"
  exit 1
fi

echo "Validation passed!"
```

---

## Security Considerations

### Identity and Access Management

1. **Managed Identity**: All Azure services use managed identity (no connection strings in code)
2. **RBAC**: Least privilege access (data plane permissions via Azure RBAC)
3. **Key Vault**: Secrets stored centrally, rotated regularly
4. **Private Endpoints**: Production traffic stays within Azure backbone (no public internet)

### Network Security

1. **VNet Integration**: Container Apps run in dedicated subnet
2. **Network Security Groups (NSG)**: Restrict traffic between subnets
3. **Azure Firewall**: Optional egress filtering for production
4. **DDoS Protection**: Enable Azure DDoS Standard for production

### Data Protection

1. **Encryption at Rest**: All data encrypted (Azure managed keys)
2. **Encryption in Transit**: TLS 1.2+ for all connections
3. **Data Residency**: Deploy to specific Azure regions for compliance
4. **Backup**: Geo-redundant backups with 30-day retention (production)

### Compliance

1. **Audit Logs**: 7-year retention in immutable storage
2. **Access Logs**: All resource access logged to Log Analytics
3. **Compliance Reports**: Export logs for SOC 2, GDPR audits

---

## Cost Estimation

### Development Environment (Monthly)

| Service | SKU | Estimated Cost |
|---------|-----|----------------|
| Container Apps | Consumption, ~100K requests | $15 |
| Cosmos DB | Serverless, 10GB storage | $30 |
| Storage Account | Standard LRS, 50GB | $5 |
| Azure AI Foundry | 10K TPM, ~1M tokens | $50 |
| Application Insights | 5GB ingestion | $15 |
| Container Registry | Basic | $5 |
| Key Vault | Standard | $5 |
| **Total** | | **~$125/month** |

---

### Production Environment (Monthly)

| Service | SKU | Estimated Cost |
|---------|-----|----------------|
| Container Apps | Dedicated, 3-10 replicas | $300 |
| Cosmos DB | Provisioned 4000 RU/s, 50GB | $250 |
| Storage Account | Zone-redundant, 200GB | $20 |
| Azure AI Foundry | 100K TPM, ~10M tokens | $500 |
| Application Insights | 20GB ingestion | $60 |
| Container Registry | Premium, geo-replicated | $40 |
| Key Vault | Standard | $5 |
| VNet + Private Endpoints | 5 endpoints | $25 |
| Log Analytics | 10GB storage | $30 |
| **Total** | | **~$1,230/month** |

*Note: Costs vary by region and usage patterns. Use [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator/) for accurate estimates.*

---

## Post-Deployment Configuration

### 1. Microsoft Entra ID (Azure AD) Setup

**Register Applications**:
1. **Backend API App**:
   - App name: `HR Timesheet Agent API`
   - Expose API: `api://hr-timesheet-agent-api`
   - Scopes: `Timesheet.Read`, `Timesheet.Write`
   - App roles: `Employee`, `Manager`, `Admin`

2. **Frontend SPA App**:
   - App name: `HR Timesheet Agent Web`
   - Platform: Single-page application
   - Redirect URIs: `https://<frontend-url>/auth/callback`
   - API permissions: Delegated permissions to backend API

**Configuration**:
- Update Key Vault with:
  - `entra-backend-client-id`
  - `entra-frontend-client-id`
  - `entra-tenant-id`

---

### 2. Build and Push Container Images

```bash
# Build backend
cd backend/src/HRTimesheetAgent.API
docker build -t <acr-name>.azurecr.io/hrtimesheetagent-api:latest .

# Build frontend
cd frontend
docker build -t <acr-name>.azurecr.io/hrtimesheetagent-web:latest .

# Login to ACR
az acr login --name <acr-name>

# Push images
docker push <acr-name>.azurecr.io/hrtimesheetagent-api:latest
docker push <acr-name>.azurecr.io/hrtimesheetagent-web:latest

# Update Container Apps to pull latest images
az containerapp update \
  --name ca-hrtimesheet-api-prod \
  --resource-group rg-hr-timesheet-prod \
  --image <acr-name>.azurecr.io/hrtimesheetagent-api:latest
```

---

### 3. Seed Reference Data

**Projects** (mock for phase 1):
```bash
# Run seed script via Container App
az containerapp exec \
  --name ca-hrtimesheet-api-prod \
  --resource-group rg-hr-timesheet-prod \
  --command "dotnet HRTimesheetAgent.API.dll --seed-data"
```

---

### 4. Configure Custom Domain (Optional)

**Frontend**:
```bash
# Add custom domain to Container App
az containerapp hostname add \
  --name ca-hrtimesheet-web-prod \
  --resource-group rg-hr-timesheet-prod \
  --hostname timesheet.company.com

# Bind SSL certificate (managed certificate)
az containerapp hostname bind \
  --name ca-hrtimesheet-web-prod \
  --resource-group rg-hr-timesheet-prod \
  --hostname timesheet.company.com \
  --environment <container-apps-env-id> \
  --validation-method CNAME
```

---

## Monitoring and Operations

### Health Checks

**Endpoints**:
- Backend: `https://<backend-url>/health/live` (liveness)
- Backend: `https://<backend-url>/health/ready` (readiness)
- Frontend: `https://<frontend-url>/` (HTTP 200)

**Monitoring**:
- Azure Monitor Container Apps health dashboard
- Application Insights availability tests

---

### Log Queries (Kusto KQL)

**View container logs**:
```kql
ContainerAppConsoleLogs_CL
| where ContainerAppName_s == "ca-hrtimesheet-api-prod"
| where TimeGenerated > ago(1h)
| project TimeGenerated, Log_s
| order by TimeGenerated desc
```

**Track agent response times**:
```kql
requests
| where name contains "agent"
| summarize avg(duration), percentiles(duration, 50, 95, 99) by bin(timestamp, 5m)
| render timechart
```

---

### Scaling Strategies

**Automatic Scaling**:
- HTTP concurrency: Scale at >100 requests/replica
- CPU: Scale at >70% CPU usage
- Memory: Scale at >80% memory usage

**Manual Scaling** (temporary burst):
```bash
az containerapp update \
  --name ca-hrtimesheet-api-prod \
  --resource-group rg-hr-timesheet-prod \
  --min-replicas 5 \
  --max-replicas 20
```

---

## Disaster Recovery

### Backup Strategy

1. **Cosmos DB**: Continuous backup with 30-day retention (automatic)
2. **Audit Logs**: Geo-redundant storage (GRS) with cross-region replication
3. **Container Images**: Geo-replicated ACR (Premium tier)

### Recovery Procedures

**Cosmos DB Restore**:
```bash
# Restore database to specific timestamp
az cosmosdb restore \
  --resource-group rg-hr-timesheet-prod \
  --account-name cosmos-hrtimesheet-prod \
  --target-database-account-name cosmos-hrtimesheet-restored \
  --restore-timestamp "2025-12-30T10:00:00Z" \
  --location eastus
```

**Regional Failover**:
- Deploy infrastructure to secondary region using Bicep
- Update DNS to point to secondary region
- Restore data from geo-replicated backups

### RTO/RPO Targets

- **RTO** (Recovery Time Objective): 4 hours
- **RPO** (Recovery Point Objective): 15 minutes (Cosmos DB continuous backup)

---

## Maintenance and Updates

### Infrastructure Updates

**Minor Changes** (parameter updates):
```bash
./scripts/deploy.sh prod
```

**Major Changes** (resource modifications):
1. Test in dev environment
2. Deploy to staging
3. Validate smoke tests
4. Deploy to production (maintenance window)

### Application Updates

**CI/CD Pipeline** (GitHub Actions / Azure DevOps):
1. Build Docker images
2. Push to ACR
3. Run integration tests
4. Update Container Apps (blue-green deployment)
5. Run smoke tests
6. Monitor metrics

---

## Next Steps

1. **Review and Approve**: Review this plan with stakeholders
2. **Create Bicep Modules**: Implement each module in `azure-deployment/modules/`
3. **Test Locally**: Use Bicep CLI to validate templates
4. **Deploy to Dev**: Deploy infrastructure to development environment
5. **Deploy Application**: Build and deploy container images
6. **Validate**: Run smoke tests and verify all services operational
7. **Production Deployment**: Repeat for staging and production

---

**Document Status**: ✅ Ready for Implementation  
**Next Action**: Begin creating Bicep modules in `azure-deployment/` directory
