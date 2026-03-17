
# End-to-End IaC + CI/CD on Azure (Multi-tier Web App)

This repository contains Terraform modules and Azure DevOps YAML pipelines to provision and deploy a secure multi-tier web app on Azure with:

- Frontend: Azure App Service (Linux)
- Backend API: Azure App Service (Linux)
- Database: Azure SQL Database
- Secrets: Azure Key Vault with RBAC
- Observability: Log Analytics + Application Insights + Azure Monitor alerts
- Network: VNet, subnet, NSG (ready for integration)
- CD strategy: Deployment slots (staging -> production) with automatic rollback on failure

## Prerequisites

- Azure subscription with permissions to create resources
- Azure DevOps project with permissions to create service connections and pipelines
- Remote state storage (Azure Storage account) for Terraform
- (Recommended) Azure DevOps **service connection** configured with **workload identity federation** (OIDC) to Azure Entra ID to avoid client secrets

## Structure

```
terraform/
  modules/
    network/           # VNet, subnet, NSG
    keyvault/          # Azure Key Vault (RBAC)
    appservice/        # App Service Plan, FE/BE Web Apps (+ slots), App Insights
    sql/               # Azure SQL Server + DB
    monitoring/        # Log Analytics, alerts, diagnostic settings
  envs/
    dev/terraform.tfvars
    prod/terraform.tfvars
pipelines/
  ci/
    frontend.yml       # Build, test, publish artifacts
    backend.yml
  cd/
    infra.yml          # Terraform plan/apply
    app-deploy.yml     # Slot-based deploy + swap + rollback
alerts/
  http5xx.kql
 diagrams/
  architecture.mmd     # Mermaid diagram
```

## Secure Service Connection (Azure DevOps -> Azure)

1. Create an **App Registration** in Entra ID (or let Azure DevOps create one) without a client secret by enabling **Federated credentials** (OIDC).
2. In Azure DevOps: *Project Settings > Service connections > New service connection > Azure Resource Manager > Workload identity federation*.
3. Scope it to subscription / resource group (least privilege) and name it, e.g., `az-oidc-svcconn`.
4. Grant the service principal required roles on the target scope (e.g., `Contributor` for infra pipeline, `Web Plan Contributor`/`Website Contributor` for app deployment if following least privilege).

Set a pipeline variable `AZURE_SERVICE_CONNECTION` to the service connection name.

## Remote State Backend (Terraform)

Create a storage account and container once:

```bash
az group create -n rg-tfstate -l eastus
az storage account create -n <storageName> -g rg-tfstate -l eastus --sku Standard_LRS --allow-blob-public-access false
az storage container create -n tfstate --account-name <storageName>
```

Use these as pipeline variables: `TFSTATE_RG`, `TFSTATE_STORAGE`, `TFSTATE_CONTAINER` and `ENV` (`dev` or `prod`).

## Pipeline Setup

### CI (Frontend & Backend)
- Create two pipelines from `pipelines/ci/frontend.yml` and `pipelines/ci/backend.yml`.
- Ensure your source tree has `frontend/` and `backend/` folders.
- Each CI publishes `frontend_drop` and `backend_drop` artifacts respectively.

### CD (Infrastructure)
- Create pipeline from `pipelines/cd/infra.yml`.
- Variables required:
  - `AZURE_SERVICE_CONNECTION`: your service connection
  - `TFSTATE_RG`, `TFSTATE_STORAGE`, `TFSTATE_CONTAINER`: backend storage
  - `ENV`: `dev` or `prod`
  - `APPLY`: set to `true` to apply changes

This pipeline runs `terraform init/plan` (always) and `apply` only when `APPLY=true`.

### CD (Application Deploy + Rollback)
- Create pipeline from `pipelines/cd/app-deploy.yml`.
- Set variables:
  - `AZURE_SERVICE_CONNECTION`
  - `FRONTEND_APP` (e.g., `dev-app-fe`)
  - `BACKEND_APP`  (e.g., `dev-app-be`)
  - `RESOURCE_GROUP` (e.g., `rg-dev-webapp`)
- Pipeline downloads artifacts from CI and deploys to **staging** slots, runs **smoke tests**, then swaps to production. If swap fails, rollback swaps back automatically.

## Secrets & Key Vault

- Terraform creates a Key Vault and stores `db-connection-string`.
- Web Apps have **system-assigned managed identities** and are granted the `Key Vault Secrets User` role on the vault.
- Web App settings use **Key Vault references**: `@Microsoft.KeyVault(SecretUri=...)`. Apps resolve secrets at runtime over MSI.
- **Do not** put secrets in pipeline variables. If needed by pipeline tasks (rare), use the `AzureKeyVault@2` task to map secrets to variables at job runtime.

## RBAC & Least Privilege

- Infra pipeline principal: `Contributor` (or granular roles like `Network Contributor`, `Security Admin`, `Key Vault Administrator` for setup phase).
- App deployment: `Website Contributor` to the resource group and `Key Vault Secrets User` **only if** pipeline needs to read secrets (prefer not to).

## Monitoring & Alerts

- Workspace-based Application Insights is provisioned and connected to both apps.
- Diagnostic settings ship logs/metrics to Log Analytics.
- Azure Monitor metric alerts for CPU > 80% and HTTP 5xx spikes are defined with an Action Group (update the email).
- Sample KQL in `alerts/http5xx.kql` for custom dashboards or alerts.

## Rollback Strategy

- Use deployment **slots** for both FE and BE.
- Deploy to `staging`, run health checks.
- **Swap** to production if healthy.
- On failure, automatic rollback swaps back to previous version.
- Infra rollback: keep Terraform plans under review; use `-refresh-only` or `-target` in emergencies; never force-delete state.

## Running Locally (optional)

```bash
cd terraform
terraform init -backend-config=...
terraform plan -var-file=envs/dev/terraform.tfvars
terraform apply -var-file=envs/dev/terraform.tfvars
```

## Notes & Options

- The sample uses Azure SQL. You can switch to **Azure Database for PostgreSQL Flexible Server** by creating a new module and storing its connection string in Key Vault.
- For private networking, consider VNet integration (regional) for App Service and Private Endpoints for SQL/Key Vault (not included for brevity).
- Frontend frameworks (React/Angular) and backend (Node/.NET) build steps can be adjusted in CI.

---

**Architecture**: see `diagrams/architecture.mmd` (Mermaid). Render it in Markdown or use an online Mermaid viewer.
