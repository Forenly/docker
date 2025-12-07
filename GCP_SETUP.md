# Deploying Nextcloud to Google Cloud Run

This repository is set up to deploy Nextcloud (Version 32, Apache) to Google Cloud Run using GitHub Actions and Cloud Build.

## 🚀 Setup Instructions

### 1. Google Cloud Environment Setup
- [x] APIs Enabled
- [x] Artifact Registry Repository Created (`nextcloud-repo`)
- [x] Service Account Created (`github-actions-sa`)
- [x] IAM Permissions Granted
- [x] Workload Identity Pool and Provider Configured

### 2. Finalize Authentication (WIF)
You only need to run **one command** to allow your new GitHub repository to deploy.
Replace `User/Repo` with your actual GitHub repository name (e.g., `my-user/nextcloud-run`).

```bash
export PROJECT_ID="modern-alpha-479108-b6"
export SA_EMAIL="github-actions-sa@${PROJECT_ID}.iam.gserviceaccount.com"
export POOL_ID="projects/587885930948/locations/global/workloadIdentityPools/github-pool"
export REPO="User/Repo" # <--- REPLACE THIS

gcloud iam service-accounts add-iam-policy-binding "${SA_EMAIL}" \
  --project="${PROJECT_ID}" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/${POOL_ID}/attribute.repository/${REPO}"
```

## ⚠️ Important: Data Persistence
This deployment uses ephemeral storage. **All uploads and configuration changes will be lost on restart.**
For production:
1.  **Database**: Configure Cloud SQL (PostgreSQL/MySQL) and pass environment variables (`MYSQL_HOST`, etc.).
2.  **Files**: Configure an Object Storage backend as Primary Storage or mount a Cloud Storage volume.
