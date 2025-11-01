# Liquibase Database Migration Pipeline

This repository contains a Kubernetes-based Liquibase setup for managing database schema changes using Helm charts and Jenkins CI/CD pipeline.

## Repository Structure

```
liquibase/
├── helm-charts/
│   ├── changelog/
│   │   └── releases/
│   │       ├── v1.0.0/
│   │       ├── v2.0.0/
│   │       ├── v3.0.0/
│   │       ├── v4.0.0/
│   │       └── v5.0.0/
│   ├── templates/
│   │   ├── configmap.yaml
│   │   ├── job.yaml
│   │   ├── serviceaccount.yaml
│   │   └── serviceentry.yaml
│   ├── Chart.yaml
│   └── values.yaml
├── pipeline/
│   └── pod.yaml
├── Jenkinsfile
└── README.md
```

## Database Configuration

- **Database**: PostgreSQL
- **Host**: 35.233.22.188:5432
- **Database Name**: meghdo-database
- **Instance**: meghdo-instance (Google Cloud SQL)
- **Project**: meghdo-4567
- **Region**: europe-west1

## Changelog Structure

Each release version contains XML changelog files organized by version:

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.8.xsd">
    
    <changeSet id="unique-id" author="db_user">
        <!-- Database changes -->
        <rollback>
            <!-- Rollback instructions -->
        </rollback>
    </changeSet>
</databaseChangeLog>
```

## Deployment Methods

### 1. Jenkins Pipeline Deployment

**Parameters:**
- `action`: Choose between `update` or `rollback`
- `release`: Target release version (e.g., v2.0.0)
- `rollbackRelease`: Version to rollback to
- `firstrun`: Set to true for initial deployment

**Pipeline Steps:**
1. Validates changelog files exist for the specified release
2. Configures GCP authentication and Kubernetes access
3. Deploys Liquibase job using Helm
4. Executes database update/rollback
5. Tags the deployment

### 2. Manual Helm Deployment

#### Update Database to Specific Release

```bash
# Set variables
RELEASE_VERSION="v2.0.0"
NAMESPACE="default"
TIMESTAMP=$(date +%s)
JOB_ID="manual-update-${TIMESTAMP}"

# Deploy update
helm install update-${JOB_ID} ./helm-charts \
  --create-namespace \
  --namespace ${NAMESPACE} \
  --set release=${RELEASE_VERSION} \
  --set rollbackrelease=${RELEASE_VERSION} \
  --set action="update" \
  --set jobidentifier="update-${JOB_ID}"

# Tag the deployment
helm install tag-${JOB_ID} ./helm-charts \
  --create-namespace \
  --namespace ${NAMESPACE} \
  --set release=${RELEASE_VERSION} \
  --set rollbackrelease=${RELEASE_VERSION} \
  --set action="tag" \
  --set jobidentifier="tag-${JOB_ID}"
```

#### Rollback Database

```bash
# Set variables
CURRENT_RELEASE="v3.0.0"
ROLLBACK_TO="v2.0.0"
NAMESPACE="default"
TIMESTAMP=$(date +%s)
JOB_ID="manual-rollback-${TIMESTAMP}"

# Deploy rollback
helm install rollback-${JOB_ID} ./helm-charts \
  --create-namespace \
  --namespace ${NAMESPACE} \
  --set release=${CURRENT_RELEASE} \
  --set rollbackrelease=${ROLLBACK_TO} \
  --set action="rollback" \
  --set jobidentifier="rollback-${JOB_ID}"
```

## Prerequisites

1. **Kubernetes Cluster**: GKE cluster with Workload Identity enabled
2. **Service Account**: `liquibase` service account with Cloud SQL access
3. **Secret**: `db-password` secret containing database password
4. **Helm**: Version 3.x installed
5. **Database Password Secret**:
   ```bash
   kubectl create secret generic db-password \
     --from-literal=db_password=YOUR_PASSWORD \
     --namespace=default
   ```

## Adding New Changelog

1. Create new version directory: `helm-charts/changelog/releases/vX.X.X/`
2. Add changelog XML files with unique changeSet IDs
3. Include rollback instructions for each changeSet
4. Test locally before deployment

## Environment Variables

The Liquibase container uses these environment variables:
- `LIQUIBASE_COMMAND_USERNAME`: Database username
- `LIQUIBASE_COMMAND_PASSWORD`: Database password (from secret)
- `LIQUIBASE_COMMAND_TAG`: Release version tag
- `LIQUIBASE_SEARCH_PATH`: Changelog directory path
- `LIQUIBASE_LOG_LEVEL`: Logging level (FINE)

## Monitoring

- Check job status: `kubectl get jobs -n default`
- View logs: `kubectl logs job/JOB_NAME -n default`
- Monitor database changes in PostgreSQL

## Troubleshooting

1. **Job Fails**: Check logs for SQL errors or connectivity issues
2. **Permission Denied**: Verify service account and Workload Identity setup
3. **Changelog Not Found**: Ensure XML files exist in correct release directory
4. **Database Connection**: Verify Cloud SQL Proxy configuration and network access

## Security Notes

- Database credentials stored in Kubernetes secrets
- Uses Google Cloud SQL Proxy for secure connections
- Workload Identity for GCP authentication
- Non-root container execution