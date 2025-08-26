# My App Helm Chart

A Helm chart for deploying a Go HTTP server application to Kubernetes.

## GitOps Configuration Repository

This repository follows **GitOps principles** and serves as a **configuration repository** for the [main application repository](http://github.com/ghaikanav/my-app-devops). The application code lives in a separate repository, while this repository contains all the Kubernetes deployment configurations and Helm charts.

**GitOps Benefits:**
- **Declarative**: All infrastructure is defined as code
- **Version Controlled**: Configuration changes are tracked in Git
- **Automated**: Deployment happens automatically when configuration changes
- **Auditable**: Complete history of all configuration changes
- **Consistent**: Same deployment process across all environments

## Overview

This chart deploys a simple Go HTTP server with:
- Configurable application settings via ConfigMap
- Database connection string stored as a Secret
- Service exposure via NodePort
- Resource limits and requests configured

## Prerequisites

- Kubernetes cluster
- Helm 3.x
- Docker image: `kanavghai/my-app`

## Installation

```bash
# Install the chart
helm install my-app ./

# Install with custom values
helm install my-app ./ -f custom-values.yaml
```

## Configuration

Key configuration options in `values.yaml`:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `app.port` | Application port | `8080` |
| `app.user` | Application user | `Kanav Ghai` |
| `database.connectionString` | Database connection string | `postgresql://username:password@localhost:5432/database_name` |
| `image.repository` | Docker image repository | `kanavghai/my-app` |
| `image.tag` | Docker image tag | `latest` |

## Components

- **Deployment**: Runs 2 replicas of the application
- **Service**: Exposes the app on port 8080 via NodePort
- **ConfigMap**: Stores non-sensitive configuration
- **Secret**: Stores sensitive database connection string

## Usage

```bash
# Check deployment status
kubectl get pods -l app=my-app

# Access the service
kubectl get svc my-app-service

# View logs
kubectl logs -l app=my-app
```

## Uninstall

```bash
helm uninstall my-app
```

## 🔐 AWS SecretsManager Authentication

To authenticate with AWS SecretsManager, you'll need to set up IAM credentials in your Kubernetes cluster.

### 🚀 Quick Setup

1. Use a service account (reference `serviceaccount.yaml`), annotate it with IAM
2. Go to IAM role, configure the trust policy OIDC:

```json
{
    "Effect": "Allow",
    "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
        "StringEquals": {
            "oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>:sub": "system:serviceaccount:<NAMESPACE>:<SERVICEACCOUNT>"
        }
    }
}

```

3. (Optional, if needed) Install ESO using Helm, then patch it to use your service account to prevent permission errors:

```bash
kubectl patch deployment external-secrets -n external-secrets \
  -p '{"spec":{"template":{"spec":{"serviceAccountName":"<SERVICE_ACCOUNT_NAME>"}}}}'
```