# Azure DevOps CI/CD Setup — Part Inventory Service (.NET)

## Prerequisites

- Azure CLI installed and logged in (`az login`)
- Azure DevOps organization and project created
- GitHub repo with source code
- GitHub repo for GitOps (gitops-repo-forvia)

---

## Step 1 — Create Azure Container Registry

```bash
# Create a resource group (skip if already exists)
az group create --name idcube-rg --location southindia

# Create the ACR
az acr create \
  --resource-group idcube-rg \
  --name idcubeacr \
  --location southindia \
  --sku Basic
```

Your image will be pushed to: `idcubeacr.azurecr.io/part-inventory-service`

---

## Step 2 — Create Service Connection for ACR

The automatic "Azure Container Registry" picker in Azure DevOps often fails with Service Principal auth. Use the **manual Service Principal** flow below instead.

### 2a — Create a Service Principal and assign ACR role

```bash
# Get your subscription ID
az account show --query id -o tsv

# Get the ACR resource ID
az acr show --name idcubeacr --resource-group idcube-rg --query id -o tsv

# Create a service principal scoped to the ACR with AcrPush role
az ad sp create-for-rbac \
  --name idcube-acr-sp \
  --role AcrPush \
  --scopes $(az acr show --name idcubeacr --resource-group idcube-rg --query id -o tsv)
```

The output will look like:

```json
{
  "appId": "<client-id>",
  "displayName": "idcube-acr-sp",
  "password": "<client-secret>",
  "tenant": "<tenant-id>"
}
```

Save these values — the `password` is shown only once.

### 2b — Create the Service Connection in Azure DevOps

1. Go to **Project Settings → Service Connections → New service connection**
2. Choose **Docker Registry**
3. Select **Others** (not "Azure Container Registry" — that triggers the broken auto-auth)
4. Fill in:

   | Field | Value |
   | --- | --- |
   | Docker Registry | `https://idcubeacr.azurecr.io` |
   | Docker ID | `<appId>` from the SP output |
   | Docker Password | `<password>` from the SP output |
   | Email | (leave blank or use any placeholder) |

5. Name it exactly: `acr-service-connection`
6. Click **Verify and save**

> **Why "Others"?** The "Azure Container Registry" option uses the Azure Resource Manager flow which requires the Azure DevOps identity to already have subscription-level permissions. Using "Others" with the SP credentials bypasses that and authenticates directly against the ACR registry endpoint.

### 2c — Verify the SP can pull/push (optional sanity check)

```bash
# Log in as the service principal
az acr login --name idcubeacr \
  --username <appId> \
  --password <client-secret>

# Should print: Login Succeeded
```

---

## Step 3 — Add GitHub PAT as a Secret Variable

In Azure DevOps:

1. Go to **Pipelines → Library → + Variable group**
2. Name the group (e.g., `pipeline-secrets`)
3. Add a variable:
   - Name: `github-pat`
   - Value: your GitHub Personal Access Token (needs `repo` scope)
   - Click the **lock icon** to mark it as secret
4. Save the variable group

> The PAT is used by the pipeline to push the updated image tag to the GitOps repo.

---

## Step 4 — Update azure-pipelines.yml

Open `azure-pipelines.yml` and replace the placeholder on line 4:

```yaml
ACR_NAME: 'idcubeacr.azurecr.io'
```

If you named the service connection differently in Step 2, also update:

```yaml
containerRegistry: 'acr-service-connection'
```

---

## Step 5 — Link the Variable Group to the Pipeline

Option A — in the Azure DevOps UI:

1. Open the pipeline → click **Edit → Variables → Variable groups**
2. Link the `pipeline-secrets` group created in Step 3

Option B — directly in `azure-pipelines.yml`:

```yaml
variables:
  - group: pipeline-secrets
  - name: ACR_NAME
    value: 'idcubeacr.azurecr.io'
  # ... other variables
```

---

## Step 6 — Create the Pipeline in Azure DevOps

1. Go to **Pipelines → New Pipeline**
2. Select **GitHub** (or Azure Repos Git, depending on where your code lives)
3. Authorize and select your repository
4. Choose **Existing Azure Pipelines YAML file**
5. Set the path to: `/cicd-azure-devops-argocd/part-inventory-service-src/azure-pipelines.yml`
6. Click **Continue → Run**

---

## Pipeline Stages Overview

| Stage | Tool | What it does |
| --- | --- | --- |
| Run Tests | `DotNetCoreCLI@2` | Runs `dotnet test` against the project |
| Build Docker Image | `Docker@2` | Builds image, tags with build number and `latest` |
| Trivy Scan | `script` | Scans image for HIGH/CRITICAL CVEs — fails pipeline if found |
| Push Image to ACR | `Docker@2` | Pushes both tags to ACR via service connection |
| Update GitOps Tag | `script` | Clones GitOps repo, updates image tag in values file, commits and pushes |

---

## Trivy Vulnerability Scan

Trivy is installed at runtime on the build agent and scans the locally built Docker image **before** it is pushed to ACR. This ensures no image with HIGH or CRITICAL vulnerabilities reaches the registry.

### How it works

```yaml
- script: |
    curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
    trivy image \
      --exit-code 1 \
      --severity HIGH,CRITICAL \
      --no-progress \
      --format table \
      $(IMAGE_NAME):$(IMAGE_TAG)
  displayName: 'Trivy — Vulnerability Scan'
```

| Flag | Meaning |
| --- | --- |
| `--exit-code 1` | Fails the pipeline if any matching CVEs are found |
| `--severity HIGH,CRITICAL` | Only HIGH and CRITICAL vulnerabilities fail the build; LOW/MEDIUM are reported but ignored |
| `--format table` | Prints a human-readable table in the pipeline logs |
| `--no-progress` | Suppresses progress bar noise in CI logs |

### Scan runs after Build, before Push

```text
Run Tests → Build Image → Trivy Scan → Push to ACR → Update GitOps Tag
```

If Trivy fails, the image is never pushed to ACR and the GitOps update never runs.

### To allow the pipeline to continue despite vulnerabilities (audit mode)

Change `--exit-code 1` to `--exit-code 0`. The scan results are still printed but the pipeline will not fail:

```yaml
trivy image \
  --exit-code 0 \
  --severity HIGH,CRITICAL \
  --no-progress \
  --format table \
  $(IMAGE_NAME):$(IMAGE_TAG)
```

### To export results as SARIF (for Azure DevOps Security tab)

```yaml
- script: |
    curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
    trivy image \
      --exit-code 1 \
      --severity HIGH,CRITICAL \
      --format sarif \
      --output trivy-results.sarif \
      $(IMAGE_NAME):$(IMAGE_TAG)
  displayName: 'Trivy — Vulnerability Scan'

- task: PublishBuildArtifacts@1
  displayName: 'Publish Trivy SARIF Report'
  inputs:
    pathToPublish: 'trivy-results.sarif'
    artifactName: 'trivy-report'
  condition: always()
```

---

## Mapping: Jenkins → Azure DevOps

| Jenkins | Azure DevOps |
| --- | --- |
| `Jenkinsfile` | `azure-pipelines.yml` |
| `credentialsId` in `withCredentials` | Service connection + secret variables |
| `docker buildx build` | `Docker@2` task |
| `DotNetCoreCLI` (manual) | `DotNetCoreCLI@2` built-in task |
| `BUILD_NUMBER` | `Build.BuildNumber` |
| `withCredentials([usernamePassword(...)])` | `env: GITHUB_PAT: $(github-pat)` |
