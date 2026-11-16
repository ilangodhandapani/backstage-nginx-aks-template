# Backstage Self-Service AKS Deployment & Automated ServiceNow CMDB Sync

An end-to-end Internal Developer Platform (IDP) workflow that enables developers to scaffold an NGINX microservice in **Backstage** (selecting an owning team synced from **Microsoft Entra ID**), deploy it passwordlessly to **Azure Kubernetes Service (AKS)** via **GitHub Actions (OIDC)**, and automatically register/update the Application CI and AKS Host CI in **ServiceNow CMDB** via a **Go microservice** running inside AKS.

---

## Architecture Overview

```text
+-----------------------------------------------------------------------------------+

| 1. MICROSOFT ENTRA ID (Identity & Federated Trust)                                |
|  |-- Security Group: web-platform-sre                                             |
|  |-- App 1 (backstage-org-sync): Graph API -> Syncs Users & Groups to Backstage   |
|  +-- App 2 (github-aks-deployer): OIDC Fed Cred -> Passwordless Azure Login       |
+-----------------------------------+-----------------------------------------------+
                                    | (MS Graph Sync)
                                    v
+-----------------------------------------------------------------------------------+

| 2. BACKSTAGE (Developer Portal & Software Catalog)                                |
|  |-- OwnerPicker dropdown populated with Entra ID groups                          |
|  |-- Scaffolds repo from backstage-nginx-aks-template                             |
|  +-- Registers Component in Catalog (catalog-info.yaml)                           |
+-----------------------------------+-----------------------------------------------+
                                    | (Creates GitHub Repo)
                                    v
+-----------------------------------------------------------------------------------+

| 3. GITHUB ACTIONS (Passwordless CI/CD Pipeline)                                   |
|  |-- Triggered on demand via workflow_dispatch                                    |
|  |-- Authenticates to Azure via OIDC (Zero static Azure secrets)                  |
|  |-- Deploys NGINX to AKS + auto-rollback (kubectl rollout undo) on failure       |
|  +-- Opens encrypted kubectl port-forward to Go CMDB service in AKS               |
+-----------------------------------+-----------------------------------------------+
                                    | (OIDC Deploy + Port-Forward)
                                    v
+-----------------------------------------------------------------------------------+

| 4. AZURE AKS CLUSTER: my-aks-standard (Resource Group: ilango)                    |
|  |-- Namespace: default        -> nginx-frontend Deployment & Service             |
|  +-- Namespace: platform-tools -> cmdb-sync-service (Go Microservice on :8080)    |
+-----------------------------------+-----------------------------------------------+
                                    | (HTTPS REST / IRE API)
                                    v
+-----------------------------------------------------------------------------------+

| 5. SERVICENOW CMDB (://service-now.com)                                    |
|  |-- Endpoint: POST /api/now/identifyreconcile?sysparm_data_source=Backstage      |
|  |-- cmdb_ci_appl:     nginx-frontend-production (Support Group: web-platform-sre)|
|  |-- cmdb_ci_hardware: my-aks-standard                                            |
|  +-- cmdb_rel_ci:      nginx-frontend-production [Runs on::Runs] my-aks-standard  |
+-----------------------------------------------------------------------------------+
```

---

## Step 1: Microsoft Entra ID Setup

Create **two separate App Registrations** to enforce Least Privilege:

1. **App 1: `backstage-org-sync` (Directory Sync for Backstage)**
   * **API Permissions (Application):** Microsoft Graph `User.Read.All`, `GroupMember.Read.All` (Grant Admin Consent).
   * **Credentials:** Generate a Client Secret for Backstage's `microsoftGraphOrg` provider.
   * **Groups:** Create security group `web-platform-sre` and assign users.

2. **App 2: `github-aks-deployer` (Passwordless CI/CD for AKS)**
   * **AKS IAM Roles (on `my-aks-standard` in Resource Group `ilango`):**
     * `Azure Kubernetes Service Cluster User Role`
     * `Azure Kubernetes Service RBAC Writer`
   * **Federated Credential:**
     * **Scenario:** GitHub Actions deploying Azure resources
     * **Organization:** `ilangodhandapani` (Organization ID: `80236431`)
     * **Repository:** `nginx-frontend` (Repository ID from repo metadata)
     * **Entity Type:** Branch (`main`)

---

## Step 2: Backstage Setup & Configuration

### 1. Install Required Backend Packages
```bash
yarn --cwd packages/backend add \
  @backstage/plugin-scaffolder-backend-module-github \
  @backstage/plugin-catalog-backend-module-msgraph \
  @backstage/plugin-auth-backend-module-github-provider \
  @backstage/plugin-kubernetes-backend
```

### 2. Register Modules in `packages/backend/src/index.ts`
```typescript
backend.add(import('@backstage/plugin-scaffolder-backend-module-github'));
backend.add(import('@backstage/plugin-catalog-backend-module-msgraph'));
backend.add(import('@backstage/plugin-auth-backend-module-github-provider'));
backend.add(import('@backstage/plugin-kubernetes-backend'));
```

### 3. Configure `app-config.yaml`
```yaml
integrations:
  github:
    - host: github.com
      token: \${GITHUB_TOKEN}

catalog:
  rules:
    - allow: [Component, System, API, Resource, Location, Template]
  providers:
    microsoftGraphOrg:
      default:
        tenantId: \${AZURE_TENANT_ID}
        clientId: \${MSGRAPH_CLIENT_ID}
        clientSecret: \${MSGRAPH_CLIENT_SECRET}
        user:
          filter: accountEnabled eq true
        group:
          filter: "securityEnabled eq true"
        schedule:
          frequency: { minutes: 60 }
          timeout: { minutes: 50 }
  locations:
    - type: url
      target: https://github.com
      rules:
        - allow: [Template]
```

---

## Step 3: ServiceNow CMDB Preparation

1. **Create API Service Account (`sys_user`):**
   * Username: `backstage.api` (Web service access only).
   * Roles: `cmdb_inst_admin`, `itil`, `personalize_choices`.
2. **Add `Backstage` to `discovery_source` Choice List (`sys_choice.list`):**
   * **Table:** `Configuration Item [cmdb_ci]` | **Element:** `discovery_source` | **Label/Value:** `Backstage`.
3. **Create Support Group (`sys_user_group.list`):**
   * **Name:** `web-platform-sre` (matching the Entra ID / Backstage team name).

---

## Step 4: Deploy Go CMDB Microservice to AKS

### 1. Build & Push Container Image to Azure Container Registry (`acrilango2026`)
```bash
export RESOURCE_GROUP="ilango"
export AKS_CLUSTER="my-aks-standard"
export ACR_NAME="acrilango2026"

az provider register --namespace Microsoft.ContainerRegistry --wait
az acr create --resource-group \(RESOURCE_GROUP --name\)ACR_NAME --sku Basic --location australiaeast
az aks update --resource-group \$RESOURCE_GROUP --name \(AKS_CLUSTER --attach-acr\)ACR_NAME

az acr login --name \$ACR_NAME
docker build --platform linux/amd64 -t \${ACR_NAME}.azurecr.io/cmdb-sync-service:v1 .
docker push \${ACR_NAME}.azurecr.io/cmdb-sync-service:v1
```

### 2. Deploy to `platform-tools` Namespace in AKS
```bash
az aks get-credentials --resource-group \(RESOURCE_GROUP --name\)AKS_CLUSTER --overwrite-existing
kubectl apply -f k8s-cmdb-sync.yaml
kubectl rollout status deployment/cmdb-sync-service -n platform-tools
```

---

## Step 5: GitHub Repository Secrets & Workflow Execution

In the generated application repository (`nginx-frontend`), add the following **Actions Secrets**:
* `AZURE_CLIENT_ID`: Application (client) ID of `github-aks-deployer`
* `AZURE_TENANT_ID`: Microsoft Entra Directory (tenant) ID
* `AZURE_SUBSCRIPTION_ID`: `48bd29ff-6349-4b23-8e48-8cebb910cb41`
* `AKS_RESOURCE_GROUP`: `ilango`

Trigger the **`Deploy NGINX to AKS & Sync ServiceNow CMDB`** workflow via **Actions > Run workflow (`workflow_dispatch`)** to deploy the application pods to AKS and upsert the application and cluster CIs in ServiceNow.

