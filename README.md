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

| Parameter                   | Description                | Default                                                       |
| --------------------------- | -------------------------- | ------------------------------------------------------------- |
| `app.port`                  | Application port           | `8080`                                                        |
| `app.user`                  | Application user           | `Kanav Ghai`                                                  |
| `database.connectionString` | Database connection string | `postgresql://username:password@localhost:5432/database_name` |
| `image.repository`          | Docker image repository    | `kanavghai/my-app`                                            |
| `image.tag`                 | Docker image tag           | `latest`                                                      |

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

1. Use a service account (reference `serviceaccount.yaml`), annotate it with IAM **_(deprecated)_**
2. Go to IAM role, configure the trust policy OIDC: **_(deprecated)_**

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

## 🔐 Alternative Authentication: Using IAM Roles

**Note:** Service account references have issues. Use IAM roles for authentication instead.

**References:**

- [Tutorial](https://containscloud.com/2024/03/24/integrating-aws-secrets-manager-to-eks-using-external-secrets/)
- [GitHub Issue](https://github.com/external-secrets/external-secrets/issues/2951#issuecomment-1866943943)

### Step 1: Create Two IAM Roles

Create two roles with minimal trust and permissions:

1. **ESO Role** (`eso-role`) - to be assigned as annotation to your external-secrets operator
2. **Secret Store Role** (`secretstore-<namespace>-role`) - for accessing secrets

#### ESO Role Trust Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["sts:AssumeRole", "sts:TagSession"],
      "Principal": {
        "Service": ["ec2.amazonaws.com"]
      }
    }
  ]
}
```

#### Secret Store Role Permission Policy and OIDC Trust

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/secretstore-<NAMESPACE>-eso-role"
    },
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>:sub": "system:serviceaccount:<namespace>:<service_account_name>"
        }
      }
    }
  ]
}
```
### Step 1.5: Create Service Account  
```bash 
kubectl create serviceaccount <service_account_name> -n <namespace>
```  

### Step 2: Annotate Service Account

```bash        
kubectl annotate serviceaccount <service_account_name> \                           
 eks.amazonaws.com/role-arn=arn:aws:iam::<Account_id>:role/<eso-role> \
  --namespace <namespace>
```

```bash
kubectl annotate serviceaccount <service_account_name> \                         
  eks.amazonaws.com/audience=sts.amazonaws.com \
  --namespace <namespace>


kubectl annotate serviceaccount <service_account_name> \
  -n <namespace> \
  eks.amazonaws.com/role-arn=arn:aws:iam::<ACCOUNT_ID>:role/eso-role \
  --overwrite
```

In values.yal , make sure the secrice account is create: false and name: eso

### Step 3: Verify Configuration

```bash
kubectl get serviceaccount <service_account_name> -n <namespace> \
  -o jsonpath="{.metadata.annotations.eks\.amazonaws\.com/role-arn}"
```

### Step 4: Create role for your Secret Store

Create another role allow the ESO role to assume it:

**Note:** Give this role permission to read/write secrets as needed.

### Step 5: Final Restart

```bash
kubectl rollout restart deployment external-secrets -n external-secrets
```

**Wait a few minutes for changes to take effect.**

### Install ArgoCD in custom namespace
Open a file kustomization.yaml 
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: <namespace>
resources:
  - install.yaml
patches:
  - target:
      kind: ClusterRoleBinding
    patch: |-
      - op: replace
        path: /subjects/0/namespace
        value:  <namespace>
```

```bash
kubectl apply -k .       
```

### OIDC Indentity Provider
Make sure to create a OIDC Indentity Provider in IAM section with URL being the OIDC url from EKS cluster.