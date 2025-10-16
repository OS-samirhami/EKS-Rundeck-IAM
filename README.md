# EKS-Rundeck-IAM

A comprehensive solution for configuring IAM roles and Kubernetes RBAC permissions to enable Rundeck automation server to access Amazon EKS clusters with read-only privileges.

## Overview

This project provides the necessary IAM policies, trust relationships, and Kubernetes RBAC configurations to securely integrate Rundeck with Amazon Elastic Kubernetes Service (EKS). It implements a cross-account role assumption pattern that allows Rundeck running on EC2 instances to access EKS cluster resources with read-only permissions. 

The solution uses:
- **EKS Access Entries API** (modern approach) with authentication mode `API_AND_CONFIG_MAP` instead of manually editing the legacy aws-auth ConfigMap
- **External ID** for enhanced security against the confused deputy problem
- **Cross-account role assumption** to support multiple target accounts

This approach provides better auditability, easier management through AWS APIs, and eliminates the need to manually edit Kubernetes ConfigMaps. By using EKS Access Entries with `API_AND_CONFIG_MAP` authentication mode, you get the benefits of API-based management while maintaining backward compatibility with existing ConfigMap-based access.

## Architecture

### Account Structure

This solution uses a **centralized Rundeck deployment** with **cross-account access** to multiple EKS clusters:

| Account Type | Account ID | Primary Role | Purpose | What It Hosts |
|--------------|------------|--------------|---------|---------------|
| **Source** | 316978178737 | Rundeck-Community | Automation Hub | Rundeck EC2 instances |
| **Target(s)** | Multiple accounts | Rundeck_Access_ODC | EKS Access | EKS clusters (dev/staging/prod) |

#### 🏢 Source Account (316978178737) - Rundeck Management Account
**Purpose:** Hosts the centralized Rundeck automation server

**What it does:**
- Runs Rundeck on EC2 instances
- Serves as the single point of automation for all EKS clusters
- Initiates cross-account role assumptions to target accounts

**Key Role:** `Rundeck-Community`
- **Attached to:** EC2 instances running Rundeck via instance profile
- **Permissions:** Can assume `Rundeck_Access_ODC` in ANY AWS account (using wildcard `*`)
- **Trust Policy:** Only allows EC2 service from account 316978178737
- **Security:** Requires external ID when assuming roles in target accounts

#### 🎯 Target Account(s) - EKS Cluster Accounts
**Purpose:** Hosts EKS clusters that need to be managed/monitored by Rundeck

**What it does:**
- Runs production/staging/dev EKS clusters
- Grants controlled access to Rundeck from the source account
- Maintains security boundaries while allowing read-only automation

**Key Role:** `Rundeck_Access_ODC`
- **Purpose:** Provides EKS cluster access to Rundeck
- **Permissions:** 
  - List and describe EKS clusters and node groups
  - Create EKS access entries
  - Access Kubernetes API for read-only operations
- **Trust Policy:** Trusts `Rundeck-Community` role from ANY account (supports multiple Rundeck deployments if needed)
- **Security:** Protected by external ID to prevent unauthorized access

### Role Comparison Matrix

| Aspect | Rundeck-Community (Source) | Rundeck_Access_ODC (Target) |
|--------|---------------------------|-------------------------------|
| **Account** | 316978178737 (Rundeck account) | Multiple target accounts |
| **Attached To** | EC2 instances (via instance profile) | Not attached, only assumed |
| **Can Be Assumed By** | EC2 service in account 316978178737 | Rundeck-Community role (any account) |
| **Can Assume** | Rundeck_Access_ODC in any account | Cannot assume other roles |
| **AWS Permissions** | sts:AssumeRole only | EKS list/describe/access operations |
| **K8s Access** | None directly | Via EKS Access Entry → rundeck-readonly group |
| **External ID Required** | When assuming target roles | When being assumed |
| **Policy Files** | ODC.json, trust_relationship.json | AWS_Services_Access_Readonly.json, trust_relationsship.json |

### Role Chain Flow

```
┌─────────────────────────────────────┐
│  Source Account (316978178737)      │
│  ┌─────────────────────────────┐    │
│  │ EC2 Instance (Rundeck)      │    │
│  │ Instance Profile:           │    │
│  │ → Rundeck-Community         │    │
│  └──────────┬──────────────────┘    │
└─────────────┼───────────────────────┘
              │ Assumes role with External ID
              │ (sts:AssumeRole)
              ▼
┌─────────────────────────────────────┐
│  Target Account (e.g., 621352866489)│
│  ┌─────────────────────────────┐    │
│  │ Rundeck_Access_ODC        │    │
│  │ Grants EKS API access       │    │
│  └──────────┬──────────────────┘    │
│             │                        │
│             ▼                        │
│  ┌─────────────────────────────┐    │
│  │ EKS Cluster                 │    │
│  │ • Creates Access Entry      │    │
│  │ • Maps to K8s group         │    │
│  │   "rundeck-readonly"        │    │
│  └──────────┬──────────────────┘    │
└─────────────┼───────────────────────┘
              │
              ▼
     ┌────────────────────┐
     │ Kubernetes RBAC    │
     │ ClusterRole:       │
     │ rundeck-ro         │
     │ (read-only access) │
     └────────────────────┘
```

### Security Layers

1. **IAM Role Assumption** - Rundeck must successfully assume the target role
2. **External ID Validation** - Prevents confused deputy attacks
3. **EKS Access Entry** - Grants specific IAM principal access to K8s API
4. **Kubernetes RBAC** - Enforces read-only permissions at the cluster level

### Real-World Use Cases

This architecture supports common enterprise scenarios:

#### Scenario 1: Multi-Account Organization
```
Rundeck Account (316978178737)
    └── Rundeck Server
         ├── Accesses → Dev Account (111111111111)
         │               └── dev-eks-cluster
         ├── Accesses → Staging Account (222222222222)
         │               └── staging-eks-cluster
         └── Accesses → Production Account (333333333333)
                         └── prod-eks-cluster
```

**Benefits:**
- Single Rundeck instance manages all environments
- Clear security boundaries between accounts
- Easy to add/remove cluster access
- Centralized audit logs in the Rundeck account

#### Scenario 2: Operations Flow

1. **User triggers Rundeck job**
   - User authenticates to Rundeck web UI
   - Selects job to list pods in production

2. **Rundeck assumes target role**
   - Uses EC2 instance profile (Rundeck-Community)
   - Calls `sts:AssumeRole` with external ID
   - Receives temporary credentials for Rundeck_Access_ODC

3. **Access EKS cluster**
   - Uses temporary credentials
   - Calls EKS API to get cluster info
   - EKS Access Entry maps role to kubernetes group

4. **Kubernetes authorization**
   - Request reaches K8s API as user "rundeck"
   - Member of group "rundeck-readonly"
   - ClusterRole grants read-only permissions
   - Pod list is returned

5. **Results displayed**
   - Rundeck shows pod information to user
   - All actions are logged with audit trail

## Repository Structure

```
.
├── README.md                          # This file
├── nosecret_ClusterRole.yaml          # Kubernetes ClusterRole and ClusterRoleBinding
├── rundeck-job.yaml                   # Example Rundeck job for EKS access
├── Rundeck-Community/                 # Source account IAM role (316978178737)
│   ├── ODC.json                       # Policy to assume Rundeck_Access_ODC
│   └── trust_relationship.json        # Trust policy for EC2 service
└── Rundeck_Access_ODC/              # Target account IAM role
    ├── AWS_Services_Access_Readonly.json               # Policy for EKS API access
    └── trust_relationsship.json       # Trust policy for Rundeck-Community role
```

## Components by Account

### 🏢 Source Account Components (316978178737)

#### 1. Rundeck-Community IAM Role

**Location:** `Rundeck-Community/`  
**Account:** 316978178737 (Rundeck Management Account)  
**Deployment:** Attached to EC2 instances running Rundeck

**Purpose:**  
This is the "identity" role that Rundeck uses to authenticate cross-account requests. Think of it as Rundeck's passport to access target accounts.

**Policy (`ODC.json`):**
- **Action:** `sts:AssumeRole`
- **Resource:** `arn:aws:iam::*:role/Rundeck_Access_ODC` (any account)
- **Condition:** Must provide external ID `EE55077E-A9DD-48C5-9A7F-3190DF36550C`
- **Effect:** Allows Rundeck to assume the lister role in any AWS account

**Trust Relationship (`trust_relationship.json`):**
- **Principal:** `ec2.amazonaws.com` service
- **Condition:** Source account must be `316978178737`
- **Condition:** Source ARN must be EC2 instance in account `316978178737`
- **Effect:** Only EC2 instances in this specific account can use this role

**Why This Matters:**  
Without this role, Rundeck has no permissions to do anything. This role is the starting point that grants the ability to "reach into" other accounts.

---

### 🎯 Target Account Components (EKS Cluster Accounts)

#### 2. Rundeck_Access_ODC IAM Role

**Location:** `Rundeck_Access_ODC/`  
**Account:** Target accounts (where EKS clusters live)  
**Deployment:** Created in each account that hosts EKS clusters

**Purpose:**  
This role grants the actual EKS and Kubernetes access. When assumed by Rundeck, it provides the permissions needed to interact with EKS clusters.

**Policy (`AWS_Services_Access_Readonly.json`):**
- `eks:ListClusters` - Discover all EKS clusters in the account
- `eks:DescribeCluster` - Get cluster details (endpoint, version, etc.)
- `eks:ListNodegroups` - List node groups within clusters
- `eks:DescribeNodegroup` - Get node group details
- `eks:DescribeClusterVersions` - Check available Kubernetes versions
- `eks:CreateAccessEntry` - Create access mappings for IAM to K8s users
- `eks:AccessKubernetesApi` - Access the Kubernetes API server
- **Condition:** Must be called with external ID `EE55077E-A9DD-48C5-9A7F-3190DF36550C`

**Trust Relationship (`trust_relationsship.json`):**
- **Principal:** `arn:aws:iam::*:role/Rundeck-Community` (from any account)
- **Condition:** Must provide external ID `EE55077E-A9DD-48C5-9A7F-3190DF36550C`
- **Effect:** Allows Rundeck-Community role to assume this role

**Why This Matters:**  
This role is what gives Rundeck actual permissions in the target account. Without it, even if Rundeck can authenticate, it can't perform any EKS operations.

#### 3. EKS Access Entries

**Account:** Target accounts (EKS cluster accounts)  
**Managed by:** AWS EKS API (not a file in this repo)  
**Authentication Mode:** `API_AND_CONFIG_MAP`

**Purpose:**  
Bridges the gap between AWS IAM (Rundeck_Access_ODC) and Kubernetes RBAC (rundeck-readonly group).

**This solution uses EKS Access Entries API with `API_AND_CONFIG_MAP` authentication mode instead of manually editing the aws-auth ConfigMap.** This approach provides:
- Direct API-based management (no manual ConfigMap editing required)
- Better security and auditability
- Eliminates risk of ConfigMap syntax errors
- Maps `Rundeck_Access_ODC` to the `rundeck-readonly` Kubernetes group
- Uses username `rundeck` for audit logging
- Maintains backward compatibility with existing ConfigMap-based access

**Why This Matters:**  
Without access entry, the IAM role can access EKS APIs but cannot authenticate to the Kubernetes cluster itself. Using the API approach instead of ConfigMap editing provides a more robust and maintainable solution.

#### 4. Kubernetes RBAC Configuration

**File:** `nosecret_ClusterRole.yaml`  
**Account:** Target accounts (deployed to each EKS cluster)  
**Deployed to:** Kubernetes clusters

**Purpose:**  
Defines what operations the "rundeck" user can perform within Kubernetes. Enforces read-only access at the cluster level.

Defines read-only permissions for Rundeck within the Kubernetes cluster.

**ClusterRole `rundeck-ro`** provides read-only access to:
- **Core resources:** pods, services, configmaps, endpoints, PVCs, service accounts
- **Workloads:** deployments, daemonsets, statefulsets, replicasets, jobs, cronjobs
- **Networking:** ingresses, endpoints, services
- **Autoscaling:** horizontal pod autoscalers
- **Policies:** pod disruption budgets
- **Resource management:** resource claims and templates

**ClusterRoleBinding `rundeck-ro-binding`:**
- Binds the `rundeck-ro` ClusterRole to the `rundeck-readonly` group

**Why This Matters:**  
This is the final security layer. Even if someone compromises the IAM roles, they can only READ data, never modify or delete anything in Kubernetes.

---

### 🔧 Operational Components

#### 5. Rundeck Job Example

**File:** `rundeck-job.yaml`  
**Account:** Source account (316978178737 - deployed to Rundeck)  
**Deployment:** Imported into Rundeck application

**Purpose:**  
A ready-to-use Rundeck job that automates the entire setup and access workflow.

Provides an example Rundeck job that demonstrates:
- Assuming the Rundeck_Access_ODC with proper credentials and external ID
- Configuring EKS cluster authentication mode (API_AND_CONFIG_MAP)
- Creating EKS access entries for the role
- Updating kubeconfig for cluster access
- Testing read-only permissions with `kubectl auth can-i`
- Listing all pods across namespaces

**Why This Matters:**  
This job serves as both documentation and a working implementation. You can use it as-is or customize it for your specific automation needs.

---

## Summary: How It All Works Together

Here's the complete flow when Rundeck needs to access an EKS cluster:

1. **🔐 Authentication Phase (Source Account)**
   - Rundeck EC2 instance has `Rundeck-Community` role attached
   - Rundeck calls `sts:AssumeRole` with external ID
   - Receives temporary credentials for `Rundeck_Access_ODC` in target account

2. **🎫 Authorization Phase (Target Account)**
   - Using temporary credentials, Rundeck calls EKS APIs
   - `Rundeck_Access_ODC` permissions allow EKS operations
   - EKS Access Entry maps IAM role → Kubernetes user "rundeck" in group "rundeck-readonly"

3. **☸️ Kubernetes Phase (EKS Cluster)**
   - Kubernetes receives API request from user "rundeck"
   - Checks group membership: "rundeck-readonly"
   - ClusterRoleBinding grants access based on ClusterRole "rundeck-ro"
   - Request is allowed/denied based on read-only permissions

**Result:** Rundeck can list pods, view logs, check deployment status, but cannot create, modify, or delete any resources.

## Setup Instructions

### Prerequisites

- AWS CLI configured with appropriate credentials
- kubectl configured to access your EKS cluster
- Access to source account (316978178737) and target EKS account(s)
- Rundeck running on EC2 instances
- External ID: `EE55077E-A9DD-48C5-9A7F-3190DF36550C` (used for role assumption security)

### Step 1: Create Rundeck-Community Role (Source Account)

In account `316978178737`:

```bash
# Create the IAM role
aws iam create-role \
  --role-name Rundeck-Community \
  --assume-role-policy-document file://Rundeck-Community/trust_relationship.json

# Attach the policy
aws iam create-policy \
  --policy-name ODC \
  --policy-document file://Rundeck-Community/ODC.json

aws iam attach-role-policy \
  --role-name Rundeck-Community \
  --policy-arn arn:aws:iam::316978178737:policy/ODC

# Create and attach instance profile to EC2 instances
aws iam create-instance-profile --instance-profile-name Rundeck-Community-Profile
aws iam add-role-to-instance-profile \
  --instance-profile-name Rundeck-Community-Profile \
  --role-name Rundeck-Community
```

### Step 2: Create Rundeck_Access_ODC (Target Account)

In your target AWS account(s):

```bash
# Create the IAM role
aws iam create-role \
  --role-name Rundeck_Access_ODC \
  --assume-role-policy-document file://Rundeck_Access_ODC/trust_relationsship.json

# Attach the policy
aws iam create-policy \
  --policy-name AWS_Services_Access_Readonly \
  --policy-document file://Rundeck_Access_ODC/AWS_Services_Access_Readonly.json

aws iam attach-role-policy \
  --role-name Rundeck_Access_ODC \
  --policy-arn arn:aws:iam::<TARGET_ACCOUNT_ID>:policy/AWS_Services_Access_Readonly
```

**Important:** Remember to replace `<TARGET_ACCOUNT_ID>` with your actual target account ID.

### Step 3: Configure Kubernetes RBAC

Apply the Kubernetes ClusterRole and ClusterRoleBinding:

```bash
# Apply ClusterRole and ClusterRoleBinding
kubectl apply -f nosecret_ClusterRole.yaml
```

### Step 4: Configure EKS Access Entries

**Important:** This solution uses the **EKS Access Entries API** with authentication mode set to `API_AND_CONFIG_MAP` instead of manually editing the aws-auth ConfigMap. This provides a more secure, API-driven approach to managing cluster access while maintaining backward compatibility.

Instead of manually editing the aws-auth ConfigMap, use EKS Access Entries API:

#### Step 4.1: Update Cluster Authentication Mode

First, ensure your EKS cluster supports both API and ConfigMap authentication:

```bash
aws eks update-cluster-config \
  --name <CLUSTER_NAME> \
  --access-config authenticationMode=API_AND_CONFIG_MAP \
  --region <REGION>
```

Wait for the update to complete, then verify:

```bash
aws eks describe-cluster \
  --name <CLUSTER_NAME> \
  --region <REGION> \
  --query "cluster.accessConfig"
```

#### Step 4.2: Create EKS Access Entry

Create an access entry for the Rundeck_Access_ODC:

```bash
aws eks create-access-entry \
  --cluster-name <CLUSTER_NAME> \
  --principal-arn arn:aws:iam::<TARGET_ACCOUNT_ID>:role/Rundeck_Access_ODC \
  --type STANDARD \
  --username rundeck \
  --kubernetes-groups rundeck-readonly \
  --region <REGION>
```

**Important:** Replace `<CLUSTER_NAME>`, `<TARGET_ACCOUNT_ID>`, and `<REGION>` with your actual values.

#### Step 4.3: Verify Access Entry

List access entries to confirm:

```bash
aws eks list-access-entries \
  --cluster-name <CLUSTER_NAME> \
  --region <REGION>

# Describe the specific access entry
aws eks describe-access-entry \
  --cluster-name <CLUSTER_NAME> \
  --principal-arn arn:aws:iam::<TARGET_ACCOUNT_ID>:role/Rundeck_Access_ODC \
  --region <REGION>
```

### Step 5: Configure Rundeck

In your Rundeck configuration, set up the AWS CLI to use role chaining with external ID:

```bash
# On the Rundeck EC2 instance, configure AWS CLI profile
cat >> ~/.aws/config << EOF
[profile rundeck-eks]
role_arn = arn:aws:iam::<TARGET_ACCOUNT_ID>:role/Rundeck_Access_ODC
credential_source = Ec2InstanceMetadata
external_id = EE55077E-A9DD-48C5-9A7F-3190DF36550C
EOF

# Test the configuration
aws eks describe-cluster --name <cluster-name> --region <region> --profile rundeck-eks

# Update kubeconfig for Rundeck
aws eks update-kubeconfig \
  --name <cluster-name> \
  --region <region> \
  --role-arn arn:aws:iam::<TARGET_ACCOUNT_ID>:role/Rundeck_Access_ODC \
  --profile rundeck-eks
```

### Step 6: Import Rundeck Job (Optional)

You can import the example Rundeck job to automate the complete setup:

1. Log in to your Rundeck instance
2. Navigate to your project
3. Go to Jobs > Import Job
4. Upload the `rundeck-job.yaml` file
5. Configure the job options:
   - `AWS_ACCOUNT`: Target AWS account ID
   - `CLUSTER_NAME`: EKS cluster name
   - `REGION`: AWS region

The job will:
- Assume the Rundeck_Access_ODC
- Update cluster authentication mode if needed
- Create the access entry
- Configure kubeconfig
- Test read-only permissions
- List all pods in the cluster

## Verification

### Verify IAM Role Chain

```bash
# From Rundeck EC2 instance, test assuming the role
aws sts assume-role \
  --role-arn arn:aws:iam::<TARGET_ACCOUNT_ID>:role/Rundeck_Access_ODC \
  --role-session-name rundeck-test \
  --external-id EE55077E-A9DD-48C5-9A7F-3190DF36550C

# List EKS clusters using the profile
aws eks list-clusters --region <region> --profile rundeck-eks
```

### Verify EKS Access Entry

```bash
# List all access entries for the cluster
aws eks list-access-entries \
  --cluster-name <CLUSTER_NAME> \
  --region <REGION>

# Describe the specific access entry
aws eks describe-access-entry \
  --cluster-name <CLUSTER_NAME> \
  --principal-arn arn:aws:iam::<TARGET_ACCOUNT_ID>:role/Rundeck_Access_ODC \
  --region <REGION>

# Verify authentication mode
aws eks describe-cluster \
  --name <CLUSTER_NAME> \
  --region <REGION> \
  --query "cluster.accessConfig"
```

### Verify Kubernetes Access

```bash
# Test kubectl access
kubectl get pods --all-namespaces

# Verify read-only (this should fail)
kubectl delete pod <some-pod>  # Should be denied
```

### Check User Identity

```bash
# Verify you're authenticated as the rundeck user
kubectl auth whoami
```

Expected output should show user as `rundeck` and groups including `rundeck-readonly`.

## Security Considerations

1. **EKS Access Entries with API_AND_CONFIG_MAP Mode:** Uses the modern EKS Access Entries API with authentication mode set to `API_AND_CONFIG_MAP` instead of manually editing the aws-auth ConfigMap. This provides better security, auditability, and eliminates the risk of ConfigMap syntax errors.
2. **Least Privilege:** The Rundeck_Access_ODC only has read-only access to EKS resources
3. **Cross-Account:** Uses IAM role assumption for cross-account access
4. **External ID Protection:** Uses external ID `EE55077E-A9DD-48C5-9A7F-3190DF36550C` to prevent the confused deputy problem
5. **Kubernetes RBAC:** Additional layer of security at the Kubernetes level
6. **Audit Trail:** All actions are logged with the username `rundeck`
7. **No Write Access:** The Kubernetes RBAC configuration explicitly only grants `get`, `list`, and `watch` verbs
8. **Account Isolation:** Source account (316978178737) is restricted via trust policies
9. **API-Based Management:** Access entries are managed via AWS APIs, reducing the risk of manual ConfigMap errors and providing better auditability

## Troubleshooting

### Cannot Assume Role

**Error:** `An error occurred (AccessDenied) when calling the AssumeRole operation`

**Solutions:**
- Verify the trust relationship in Rundeck_Access_ODC allows Rundeck-Community role
- Check that the EC2 instance has the correct instance profile (Rundeck-Community-Profile) attached
- Ensure the policy in Rundeck-Community role has permission to assume Rundeck_Access_ODC
- Verify you're using the correct external ID: `EE55077E-A9DD-48C5-9A7F-3190DF36550C`
- Check that the external ID is properly configured in both the trust policy and when assuming the role

### Cannot Access EKS Cluster

**Error:** `error: You must be logged in to the server (Unauthorized)`

**Solutions:**
- Verify EKS access entry is correctly created
- Check that the principal ARN in the access entry matches exactly
- Ensure the cluster authentication mode is set to `API_AND_CONFIG_MAP` or `API`
- Verify the access entry includes the correct Kubernetes group (`rundeck-readonly`)
- Ensure ClusterRoleBinding is applied correctly
- Verify the EKS cluster's API endpoint is accessible from Rundeck
- List access entries to confirm: `aws eks list-access-entries --cluster-name <CLUSTER_NAME>`

### Permission Denied in Kubernetes

**Error:** `Error from server (Forbidden): pods is forbidden`

**Solutions:**
- Verify the ClusterRole and ClusterRoleBinding are applied
- Check that the group name matches in the access entry (`rundeck-readonly`) and ClusterRoleBinding
- Ensure the access entry specifies the correct Kubernetes groups
- Ensure you're accessing the correct namespace (if namespace-specific issues)
- Verify with: `kubectl auth can-i get pods --as=rundeck --as-group=rundeck-readonly`

### Access Entry Creation Failed

**Error:** `An error occurred (ResourceInUseException) when calling the CreateAccessEntry operation`

**Solutions:**
- The access entry might already exist. List existing entries: `aws eks list-access-entries --cluster-name <CLUSTER_NAME>`
- If it exists, you can update or delete and recreate it
- Delete existing entry: `aws eks delete-access-entry --cluster-name <CLUSTER_NAME> --principal-arn <ROLE_ARN>`

**Error:** `An error occurred (InvalidRequestException) when calling the UpdateClusterConfig operation: Updating authentication mode is not supported`

**Solutions:**
- Your cluster version might not support EKS Access Entries API (requires EKS 1.23+)
- Check cluster version: `aws eks describe-cluster --name <CLUSTER_NAME> --query "cluster.version"`
- If on older version, consider upgrading or using the legacy aws-auth ConfigMap approach

## Customization

### Adding More Permissions

To grant additional read-only access, edit `nosecret_ClusterRole.yaml` and add more `apiGroups` and `resources`. Always use only `get`, `list`, and `watch` verbs for read-only access.

### Multiple Rundeck Users

To support multiple Rundeck instances or environments:

1. Create additional IAM roles in the target account
2. Create separate EKS access entries for each role using `aws eks create-access-entry`
3. Create separate ClusterRoleBindings if different permissions are needed

Example for a second Rundeck instance:
```bash
aws eks create-access-entry \
  --cluster-name <CLUSTER_NAME> \
  --principal-arn arn:aws:iam::<TARGET_ACCOUNT_ID>:role/Rundeck_Access_ODC-Dev \
  --type STANDARD \
  --username rundeck-dev \
  --kubernetes-groups rundeck-dev-readonly \
  --region <REGION>
```

### Namespace-Specific Access

To restrict access to specific namespaces, replace `ClusterRole` and `ClusterRoleBinding` with `Role` and `RoleBinding` in specific namespaces.

## References

- [Amazon EKS Access Entries](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html)
- [Grant IAM users access to Kubernetes with EKS access entries](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html)
- [Amazon EKS Cluster Authentication](https://docs.aws.amazon.com/eks/latest/userguide/cluster-auth.html)
- [Amazon EKS IAM Roles](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [How to use an external ID when granting access to your AWS resources](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-user_externalid.html)
- [Kubernetes RBAC Documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Rundeck Documentation](https://docs.rundeck.com/)

## License

This project is provided as-is for reference and implementation purposes.

## Contributing

Contributions are welcome! Please ensure:
- IAM policies follow least privilege principles
- Kubernetes RBAC maintains read-only access
- Documentation is updated with changes
- Security best practices are maintained

