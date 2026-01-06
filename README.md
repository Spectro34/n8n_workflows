# n8n Workflows

## Importing Workflows to n8n

1. Open n8n UI
2. Click **"Workflows"** in the sidebar
3. Click **"Add Workflow"** → **"Import from File"**
4. Select the JSON file you want to import
5. Review and configure any required credentials or placeholders
6. Save and activate the workflow

## package_update.json

**Note: This file is for workflow tracking and documentation purposes only. It is NOT a working example and requires significant configuration and integration work before it can be used.**

OBS package update automation workflow concept triggered via Slack. This workflow demonstrates the intended flow but is not production-ready.

### Workflow Steps

1. **Slack Trigger** - Receives `/pkg-check` or `/pkg-update` command
2. **Parse Command** - Extracts package name from Slack message
3. **Get Factory Version** - Queries OBS API for current package version in Factory
4. **Get Upstream Version** - Fetches latest upstream version from release-monitoring.org
5. **Compare Versions** - Determines if update is needed
6. **AI Analysis** - Analyzes version differences, impact, and generates changelog
7. **Build Job Manifest** - Creates Kubernetes Job manifest for build tasks
8. **Create K8s Build Job** - Submits job to Kubernetes API
9. **Monitor Job Status** - Tracks job execution
10. **AI: Analyze Build Results** - Reviews build output
11. **Slack: Request Approval** - Sends approval request with build results
12. **Wait for Approval** - Pauses for user decision
13. **Submit SR** - Creates submit request if approved
14. **Slack Response** - Sends final status to user

### Status

This workflow is a reference/tracking document showing the intended automation flow. It contains placeholder values and requires:
- Complete integration with OBS API
- Proper error handling
- Production-ready build job implementation
- Full SR submission logic
- Comprehensive testing and validation

## k8s-example.json

Kubernetes job creation workflow example.

### Prerequisites Setup

#### Step 1: Create Namespace (if needed)

```bash
kubectl create namespace n8n
```

#### Step 2: Create Service Account

```bash
kubectl create serviceaccount n8n-k8s-job-creator -n n8n
```

#### Step 3: Grant Permissions

Create ClusterRoleBinding to allow job creation:

```bash
kubectl create clusterrolebinding n8n-job-creator-binding \
  --clusterrole=edit \
  --serviceaccount=n8n:n8n-k8s-job-creator
```

Alternatively, create a custom Role with specific permissions:

```bash
# Create Role
kubectl create role n8n-job-creator -n n8n \
  --verb=create,get,list,watch \
  --resource=jobs

# Create RoleBinding
kubectl create rolebinding n8n-job-creator-binding -n n8n \
  --role=n8n-job-creator \
  --serviceaccount=n8n:n8n-k8s-job-creator
```

#### Step 4: Get Service Account Token

```bash
kubectl create token n8n-k8s-job-creator -n n8n --duration=8760h
```

Copy the token output - you'll need it for the workflow.

#### Step 5: Get Kubernetes API Server URL

```bash
kubectl cluster-info | grep "Kubernetes control plane"
```

Example output: `Kubernetes control plane is running at https://192.168.122.100:6443`

#### Step 6: Configure Workflow

1. Import `k8s-example.json` into n8n
2. Open the "Get K8s Service Account Token" node
3. Replace `K8S_SERVICE_ACCOUNT_TOKEN_PLACEHOLDER` with the token from Step 4
4. Replace `K8S_API_SERVER_PLACEHOLDER` with the API server URL from Step 5
5. Save the workflow

### Workflow Steps

1. **Manual Trigger** - Starts workflow execution
2. **Set Job Configuration** - Defines job name, namespace, image, and command
3. **Build Job Manifest** - Creates Kubernetes Job YAML manifest
4. **Get K8s Token** - Retrieves service account token (configured in prerequisites)
5. **Create Kubernetes Job** - POSTs job manifest to Kubernetes API
6. **Format Response** - Returns job creation status

### Verification

Test the setup:

```bash
# Verify service account exists
kubectl get serviceaccount n8n-k8s-job-creator -n n8n

# Verify permissions
kubectl auth can-i create jobs --as=system:serviceaccount:n8n:n8n-k8s-job-creator -n n8n

# Test token (should show user info)
kubectl create token n8n-k8s-job-creator -n n8n | kubectl --token=$(kubectl create token n8n-k8s-job-creator -n n8n) auth whoami
```
