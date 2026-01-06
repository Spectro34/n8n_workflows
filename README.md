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

**⚠️ Security Note**: This workflow currently uses a hardcoded JWT token placeholder. For production use, implement a more secure authentication method.

### Recommended Secure Approach: Service Account Token Mounting

Instead of hardcoding tokens, use Kubernetes' built-in service account token mounting:

1. **Create Service Account** in the target namespace
2. **Create Role** with required permissions (e.g., job creation)
3. **Create RoleBinding** to bind the service account to the role
4. **Specify `serviceAccountName`** in the job spec when creating the job
5. **Token is automatically mounted** in the job pod at `/var/run/secrets/kubernetes.io/serviceaccount/token`

This way, code running inside the job can read the token from the filesystem and use it for Kubernetes API calls, without exposing tokens in workflow definitions.

**Example Setup:**
```bash
# 1. Create service account
kubectl create serviceaccount job-runner -n test

# 2. Create role with permissions
kubectl create role job-creator -n test \
  --verb=create,get,list,watch \
  --resource=jobs

# 3. Bind service account to role
kubectl create rolebinding job-runner-binding -n test \
  --role=job-creator \
  --serviceaccount=test:job-runner

# 4. In job manifest, specify:
# spec:
#   template:
#     spec:
#       serviceAccountName: job-runner
#       containers: ...
```

**Inside the job pod**, read the token:
```bash
cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

### Alternative Secure Methods

- Using n8n environment variables for token storage
- Implementing token rotation
- Integrating with external secret management systems (e.g., HashiCorp Vault, Kubernetes Secrets)
- Using OIDC authentication if available

### Prerequisites Setup

**Note**: This setup creates a service account for n8n to authenticate when creating jobs. For jobs that need to make Kubernetes API calls internally, see the "Recommended Secure Approach" section above.

#### Step 1: Create Namespace (if needed)

```bash
kubectl create namespace n8n
```

#### Step 2: Create Service Account

```bash
kubectl create serviceaccount n8n-k8s-job-creator -n n8n
```

#### Step 3: Grant Permissions

**Option A: ClusterRoleBinding (broader permissions)**

```bash
kubectl create clusterrolebinding n8n-job-creator-binding \
  --clusterrole=edit \
  --serviceaccount=n8n:n8n-k8s-job-creator
```

**Option B: Namespace-scoped Role (recommended - least privilege)**

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

**Option A: Using Environment Variables (Recommended - More Secure)**

1. Set n8n environment variables:
   - `K8S_SERVICE_ACCOUNT_TOKEN` = token from Step 4
   - `K8S_API_SERVER` = API server URL from Step 5
2. Import `k8s-example.json` into n8n
3. Update the "Get K8s Service Account Token" node JavaScript code to:
   ```javascript
   // In n8n Code nodes, use $env.VARIABLE_NAME for environment variables
   const token = $env.K8S_SERVICE_ACCOUNT_TOKEN || 'K8S_SERVICE_ACCOUNT_TOKEN_PLACEHOLDER';
   const apiServer = $env.K8S_API_SERVER || 'K8S_API_SERVER_PLACEHOLDER';
   
   return {
     json: {
       ...$input.item.json,
       k8sToken: token,
       k8sApiServer: apiServer
     }
   };
   ```
   
   **Note**: Set environment variables in n8n: Settings → Environment Variables
4. Save the workflow

**Option B: Direct Replacement (Less Secure - For Testing Only)**

1. Import `k8s-example.json` into n8n
2. Open the "Get K8s Service Account Token" node
3. Replace `K8S_SERVICE_ACCOUNT_TOKEN_PLACEHOLDER` with the token from Step 4
4. Replace `K8S_API_SERVER_PLACEHOLDER` with the API server URL from Step 5
5. Save the workflow

**Note**: Option A is recommended for production. Never commit tokens to version control.

### Workflow Steps

1. **Manual Trigger** - Starts workflow execution
2. **Set Job Configuration** - Defines job name, namespace, image, and command
3. **Build Job Manifest** - Creates Kubernetes Job YAML manifest
4. **Get K8s Token** - Retrieves service account token (configured in prerequisites)
5. **Create Kubernetes Job** - POSTs job manifest to Kubernetes API
6. **Format Response** - Returns job creation status

**Note**: When creating jobs that need to make Kubernetes API calls, specify `serviceAccountName` in the job manifest's `spec.template.spec` section. The token will be automatically mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token` inside the job pod.

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
