# Migrating Jenkins Pipeline from Docker Hub to ACR

This guide documents how to switch the `part-inventory-service` Jenkins pipeline from Docker Hub (`ram1uj/part-inventory-service`) to Azure Container Registry (`idcubeacr.azurecr.io`).

---

## Step 1 — Create the ACR (skip if already done)

```bash
az group create --name idcube-rg --location southindia

az acr create \
  --resource-group idcube-rg \
  --name idcubeacr \
  --location southindia \
  --sku Basic
```

---

## Step 2 — Create a Service Principal for Jenkins

Jenkins authenticates to ACR using a Service Principal (username = `appId`, password = `client-secret`).

```bash
# Create SP scoped to the ACR with AcrPush role
az ad sp create-for-rbac \
  --name idcube-jenkins-acr-sp \
  --role AcrPush \
  --scopes $(az acr show --name idcubeacr --resource-group idcube-rg --query id -o tsv)
```

Output:

```json
{
  "appId": "<client-id>",
  "displayName": "idcube-jenkins-acr-sp",
  "password": "<client-secret>",
  "tenant": "<tenant-id>"
}
```

Save these — `password` is shown only once.

---

## Step 3 — Add ACR Credentials in Jenkins

1. Go to **Jenkins → Manage Jenkins → Credentials → (global) → Add Credentials**
2. Set:

   | Field | Value |
   | --- | --- |
   | Kind | Username with password |
   | Username | `<appId>` from Step 2 |
   | Password | `<client-secret>` from Step 2 |
   | ID | `acr-credentials` |
   | Description | ACR Service Principal |

3. Click **Create**

---

## Step 4 — Update the Jenkinsfile

Replace the `IMAGE_NAME` env var and the `Push Image` stage. Full diff:

**Before:**

```groovy
IMAGE_NAME = "ram1uj/part-inventory-service"
```

**After:**

```groovy
ACR_NAME    = "idcubeacr.azurecr.io"
IMAGE_NAME  = "${ACR_NAME}/part-inventory-service"
```

**Before (Push Image stage):**

```groovy
withCredentials([usernamePassword(credentialsId: 'e4af9f44-e2b9-4253-b040-14b40090e1a6', ...)]) {
    sh '''
    echo "$docker_password" | docker login -u "$docker_user" --password-stdin
    docker push ${IMAGE_NAME}:${IMAGE_TAG}
    docker push ${IMAGE_NAME}:latest
    docker logout
    '''
}
```

**After (Push Image stage):**

```groovy
withCredentials([usernamePassword(credentialsId: 'acr-credentials', passwordVariable: 'acr_password', usernameVariable: 'acr_user')]) {
    sh '''
    export PATH=$PATH:/usr/local/bin:/opt/homebrew/bin
    echo "$acr_password" | docker login idcubeacr.azurecr.io -u "$acr_user" --password-stdin
    docker push ${IMAGE_NAME}:${IMAGE_TAG}
    docker push ${IMAGE_NAME}:latest
    docker logout
    '''
}
```

> The only meaningful change in the push stage is the credential ID and the explicit registry URL passed to `docker login`. ACR requires the registry hostname in the login command.

---

## Step 5 — Update the Build Stage for ACR-compatible tags

ACR does not support multi-platform `--push` in a single `buildx` call without `--push` flag and a registry destination. Split build and push:

**Before:**

```groovy
docker buildx build \
  -t ${IMAGE_NAME}:${IMAGE_TAG} \
  -t ${IMAGE_NAME}:latest \
  --platform linux/amd64,linux/arm64 \
  -f Dockerfile .
```

**After:**

```groovy
docker buildx build \
  -t ${IMAGE_NAME}:${IMAGE_TAG} \
  -t ${IMAGE_NAME}:latest \
  --platform linux/amd64,linux/arm64 \
  --load \
  -f Dockerfile .
```

> `--load` loads the image into the local Docker daemon so Trivy can scan it before the push stage.

---

## Step 6 — Verify the updated pipeline runs

After committing the updated Jenkinsfile:

1. Trigger the pipeline manually or push to `main`
2. Confirm the **Push Image** stage logs show:

   ```text
   Login Succeeded
   The push refers to repository [idcubeacr.azurecr.io/part-inventory-service]
   ```

3. Verify the image appears in the registry:

   ```bash
   az acr repository show-tags --name idcubeacr --repository part-inventory-service --output table
   ```

---

## What changed — summary

| Item | Before | After |
| --- | --- | --- |
| Registry | `docker.io` (Docker Hub) | `idcubeacr.azurecr.io` |
| Image name | `ram1uj/part-inventory-service` | `idcubeacr.azurecr.io/part-inventory-service` |
| Credential ID | `e4af9f44-e2b9-4253-b040-14b40090e1a6` | `acr-credentials` |
| Auth method | Docker Hub username/password | Azure Service Principal |
| Login command | `docker login` (implicit Docker Hub) | `docker login idcubeacr.azurecr.io` |
