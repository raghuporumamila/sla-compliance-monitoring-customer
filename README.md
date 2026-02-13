# SLA Dashboard Service - Deployment Documentation

## Overview

This repository contains Google Cloud Build configurations for deploying and managing an SLA (Service Level Agreement) monitoring dashboard service. The system consists of two main components:

1. **Cloud Run Service Deployment** - Deploys the SLA dashboard application to Google Cloud Run
2. **Cloud Scheduler Configuration** - Sets up automated compliance report generation

## Architecture

- **Cloud Run Service**: `sla-dashboard-service` - Hosts the SLA monitoring dashboard
- **Cloud Scheduler**: Triggers periodic compliance reports via HTTP POST requests
- **Secret Manager**: Securely stores Docker Hub credentials
- **Docker Hub**: Source registry for container images

---

## File Structure

```
.
├── cloudbuild.yml              # Main service deployment configuration
├── cloudscheduler-build.yml    # Scheduler job configuration
├── config.json                 # SLA monitoring configuration
└── README.md                   # This file
```

---

## Configuration Files

### 1. cloudbuild.yml

Deploys the SLA dashboard service to Google Cloud Run.

#### Build Steps

**Step 1: Docker Hub Authentication**
- Retrieves Docker Hub password from Google Secret Manager
- Authenticates with Docker Hub using stored credentials
- Uses the secret `docker-hub-password` from Secret Manager

**Step 2: Cloud Run Deployment**
- Deploys the Docker image to Cloud Run
- Configures the service with public access (unauthenticated)
- Deploys to the `us-central1` region

#### Substitution Variables

| Variable | Value | Description |
|----------|-------|-------------|
| `_DOCKER_USER` | `raghuporumamilla` | Docker Hub username |
| `_REPO_NAME` | `rporumamilla` | Docker Hub repository name |
| `_TAG` | `v1.0.0` | Image tag/version |
| `_REGION` | `us-central1` | GCP region for deployment |
| `_PROJECT_ID` | `event-hub-317019` | GCP project ID |

#### Usage

```bash
# Deploy the service
gcloud builds submit --config=cloudbuild.yml

# Deploy with custom tag
gcloud builds submit --config=cloudbuild.yml --substitutions=_TAG=v2.0.0
```

---

### 2. cloudscheduler-build.yml

Creates or updates a Cloud Scheduler job that triggers compliance report generation.

#### Scheduler Configuration

- **Job Name**: `sla-monitoring-scheduler-job`
- **Schedule**: `0 * * * *` (hourly) - Update schedule / `0 0 1 1 *` (yearly) - Create schedule
- **Target URL**: `https://sla-dashboard-service-712620169349.us-central1.run.app/v1/compliance_report`
- **Method**: HTTP POST
- **Authentication**: OIDC token with service account `infra-sa@event-hub-317019.iam.gserviceaccount.com`

#### How It Works

1. Reads the payload from `config.json`
2. Attempts to update an existing scheduler job
3. If the job doesn't exist, creates a new one
4. Sends the configuration as JSON in the request body

#### Usage

```bash
# Deploy/update the scheduler job
gcloud builds submit --config=cloudscheduler-build.yml
```

---

### 3. config.json

Defines the SLA monitoring configuration including projects, services, and thresholds.

#### Structure

```json
{
  "projects": [
    {
      "id": "event-hub-317019",
      "services": [
        {
          "name": "hello",
          "type": "cloud_run_revision",
          "threshold": 99.95
        }
      ]
    }
  ]
}
```

#### Service Types

- **`cloud_run_revision`**: Cloud Run service monitoring
- **`gcs_bucket`**: Google Cloud Storage bucket monitoring
- **`bigquery_project`**: BigQuery project monitoring

#### Monitored Services

| Service Name | Type | SLA Threshold |
|--------------|------|---------------|
| `hello` | Cloud Run Revision | 99.95% |
| `event-hub-317019-datalake` | GCS Bucket | 99.9% |
| `sql-jobs` | BigQuery Project | 99.99% |

---

## Prerequisites

### 1. GCP Setup

- Google Cloud Project: `event-hub-317019`
- Cloud Build API enabled
- Cloud Run API enabled
- Cloud Scheduler API enabled
- Secret Manager API enabled

### 2. IAM Permissions

The Cloud Build service account needs:
- `roles/run.admin` - Deploy to Cloud Run
- `roles/iam.serviceAccountUser` - Act as service accounts
- `roles/secretmanager.secretAccessor` - Access secrets
- `roles/cloudscheduler.admin` - Manage scheduler jobs

The scheduler service account (`infra-sa@event-hub-317019.iam.gserviceaccount.com`) needs:
- `roles/run.invoker` - Invoke Cloud Run services

### 3. Secrets Configuration

Create the Docker Hub password secret:

```bash
echo -n "YOUR_DOCKER_HUB_PASSWORD" | gcloud secrets create docker-hub-password \
  --data-file=- \
  --project=event-hub-317019
```

### 4. Docker Hub

- Repository: `raghuporumamilla/rporumamilla`
- Image must be tagged and pushed before deployment
- Example: `raghuporumamilla/rporumamilla:v1.0.0`

---

## Deployment Guide

### Step 1: Build and Push Docker Image

```bash
# Build the image
docker build -t raghuporumamilla/rporumamilla:v1.0.0 .

# Push to Docker Hub
docker push raghuporumamilla/rporumamilla:v1.0.0
```

### Step 2: Deploy Cloud Run Service

```bash
# Submit the build
gcloud builds submit --config=cloudbuild.yml

# Or with custom parameters
gcloud builds submit --config=cloudbuild.yml \
  --substitutions=_TAG=v1.0.0,_REGION=us-central1
```

### Step 3: Configure Cloud Scheduler

```bash
# Deploy the scheduler
gcloud builds submit --config=cloudscheduler-build.yml
```

### Step 4: Verify Deployment

```bash
# Check Cloud Run service
gcloud run services describe sla-dashboard-service \
  --region=us-central1 \
  --project=event-hub-317019

# Check scheduler job
gcloud scheduler jobs describe sla-monitoring-scheduler-job \
  --location=us-central1 \
  --project=event-hub-317019
```

---

## Updating the Configuration

### Update SLA Thresholds

1. Edit `config.json` with new thresholds or services
2. Redeploy the scheduler:
   ```bash
   gcloud builds submit --config=cloudscheduler-build.yml
   ```

### Update Scheduler Frequency

Edit the `--schedule` parameter in `cloudscheduler-build.yml`:

- Hourly: `0 * * * *`
- Daily at midnight: `0 0 * * *`
- Every 6 hours: `0 */6 * * *`
- Weekly on Monday: `0 0 * * 1`

### Deploy New Version

1. Update the `_TAG` in `cloudbuild.yml` or pass as substitution
2. Build and push new Docker image
3. Deploy:
   ```bash
   gcloud builds submit --config=cloudbuild.yml --substitutions=_TAG=v2.0.0
   ```

---

## Troubleshooting

### Build Fails with "Invalid Image"

- Verify the Docker image exists in Docker Hub
- Check the image tag matches `_TAG` variable
- Ensure Docker Hub credentials are correct

### Scheduler Job Fails

```bash
# Check job status
gcloud scheduler jobs describe sla-monitoring-scheduler-job \
  --location=us-central1

# View logs
gcloud logging read "resource.type=cloud_scheduler_job AND \
  resource.labels.job_id=sla-monitoring-scheduler-job" \
  --limit=50 \
  --format=json
```

### Cloud Run Service Not Accessible

- Verify `--allow-unauthenticated` flag is set
- Check firewall rules
- Verify the service is running:
  ```bash
  gcloud run services list --region=us-central1
  ```

### Secret Manager Access Issues

```bash
# Verify secret exists
gcloud secrets describe docker-hub-password --project=event-hub-317019

# Grant access to Cloud Build service account
gcloud secrets add-iam-policy-binding docker-hub-password \
  --member="serviceAccount:PROJECT_NUMBER@cloudbuild.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

---

## Security Considerations

1. **Secrets Management**: Never commit Docker Hub passwords or tokens to version control
2. **Service Accounts**: Use least-privilege principle for service account permissions
3. **Authentication**: The scheduler uses OIDC tokens for secure service-to-service communication
4. **Public Access**: The Cloud Run service allows unauthenticated access - consider adding authentication if needed

---

## Monitoring and Logs

### View Cloud Run Logs

```bash
gcloud logging read "resource.type=cloud_run_revision AND \
  resource.labels.service_name=sla-dashboard-service" \
  --limit=50 \
  --format=json
```

### View Build Logs

```bash
gcloud builds list --limit=10
gcloud builds log BUILD_ID
```

### Test the Compliance Report Endpoint

```bash
curl -X POST \
  https://sla-dashboard-service-712620169349.us-central1.run.app/v1/compliance_report \
  -H "Content-Type: application/json" \
  -d @config.json
```

---

## References

- [Cloud Build Documentation](https://cloud.google.com/build/docs)
- [Cloud Run Documentation](https://cloud.google.com/run/docs)
- [Cloud Scheduler Documentation](https://cloud.google.com/scheduler/docs)
- [Secret Manager Documentation](https://cloud.google.com/secret-manager/docs)

---

## Support

For issues or questions:
1. Check the troubleshooting section above
2. Review GCP logs for error messages
3. Verify all prerequisites are met
4. Contact the infrastructure team

---

## License

[Your License Here]

## Contributors

[Your Team/Contributors Here]
