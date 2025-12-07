# Deploying Nextcloud to Google Cloud Run

This repository is set up to deploy Nextcloud (Version 32, Apache) to Google Cloud Run using GitHub Actions and Cloud Build.

## 🚀 Setup Instructions

### 1. Google Cloud Environment Setup
Run the following commands in your local terminal or Cloud Shell to initialize the infrastructure.

```bash
# Set your project ID
export PROJECT_ID="modern-alpha-479108-b6"
export REGION="us-central1"
export REPO_NAME="nextcloud-repo"

# 1. Enable APIs
gcloud services enable artifactregistry.googleapis.com \
    run.googleapis.com \
    cloudbuild.googleapis.com \
    iamcredentials.googleapis.com

# 2. Create Artifact Registry Repository
gcloud artifacts repositories create $REPO_NAME \
    --repository-format=docker \
    --location=$REGION \
    --description="Nextcloud Docker Repository"

# 3. Create Service Account for GitHub Actions
gcloud iam service-accounts create github-actions-sa \
    --display-name="GitHub Actions Service Account"

# 4. Grant Permissions
sa_email="github-actions-sa@${PROJECT_ID}.iam.gserviceaccount.com"

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${sa_email}" \
    --role="roles/run.admin"

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${sa_email}" \
    --role="roles/storage.admin" # For Cloud Build logs and GCR

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${sa_email}" \
    --role="roles/artifactregistry.writer"

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${sa_email}" \
    --role="roles/cloudbuild.builds.editor"
    
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${sa_email}" \
    --role="roles/iam.serviceAccountUser"
```

### 2. Configure GitHub Actions Authentication (WIF)
To avoid using long-lived keys, set up Workload Identity Federation (WIF).

```bash
# 1. Create Workload Identity Pool
gcloud iam workload-identity-pools create "github-pool" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --display-name="GitHub Actions Pool"

# 2. Get the Pool ID
export POOL_ID=$(gcloud iam workload-identity-pools describe "github-pool" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --format="value(name)")

# 3. Create Provider
gcloud iam workload-identity-pools providers create-oidc "github-provider" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --workload-identity-pool="github-pool" \
  --display-name="GitHub Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository" \
  --issuer-uri="https://token.actions.githubusercontent.com"

# 4. Allow GitHub Repo to impersonate Service Account
# REPLACE "Forenly/Adiyaman" with your functionality repo name "User/Repo"
export REPO="YourUser/YourNextcloudRepo" 

gcloud iam service-accounts add-iam-policy-binding "${sa_email}" \
  --project="${PROJECT_ID}" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/${POOL_ID}/attribute.repository/${REPO}"

# 5. Output the Provider Name for the YAML
echo "Provider: ${POOL_ID}/providers/github-provider"
echo "Service Account: ${sa_email}"
```

**After getting the Provider string, update `.github/workflows/deploy.yml` with the correct `workload_identity_provider` and `service_account`.**

## ⚠️ Important: Data Persistence
This deployment uses ephemeral storage. **All uploads and configuration changes will be lost on restart.**
For production:
1.  **Database**: Configure Cloud SQL (PostgreSQL/MySQL) and pass environment variables (`MYSQL_HOST`, etc.).
2.  **Files**: Configure an Object Storage backend as Primary Storage or mount a Cloud Storage volume.
