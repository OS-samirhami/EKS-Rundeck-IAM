# EKS-Rundeck-IAM

A comprehensive solution for configuring IAM roles and Kubernetes RBAC permissions to enable Rundeck automation server to access Amazon EKS clusters with read-only privileges.

## Overview

This project provides the necessary IAM policies, trust relationships, and Kubernetes RBAC configurations to securely integrate Rundeck with Amazon Elastic Kubernetes Service (EKS). It implements a cross-account role assumption pattern that allows Rundeck running on EC2 instances to access EKS cluster resources with read-only permissions. 

The solution uses:
- **EKS Access Entries** (modern approach) instead of the legacy aws-auth ConfigMap for simpler and more secure IAM-to-Kubernetes mapping
- **External ID** for enhanced security against the confused deputy problem
- **Cross-account role assumption** to support multiple target accounts

This approach provides better auditability, easier management through AWS APIs, and eliminates the need to manually edit Kubernetes ConfigMaps.

## Architecture

The solution implements a two-tier IAM role chain:

1. **Rundeck-Community** (Account: 316978178737)
   - Attached to EC2 instances running Rundeck
   - Can assume the RundeckEKSListerRole in target accounts
   - Uses external ID for additional security

2. **RundeckEKSListerRole** (Target Accounts)
   - Assumed by Rundeck-Community role
   - Has permissions to list, describe EKS clusters and access Kubernetes API
   - Mapped to Kubernetes RBAC group `rundeck-readonly`
   - Protected with external ID condition

3. **Kubernetes RBAC**
   - ClusterRole `rundeck-ro` with read-only permissions
   - Bound to the `rundeck-readonly` group

## Repository Structure

```
.
├── README.md                          # This file
├── role_config.yaml                   # Kubernetes ClusterRole and ClusterRoleBinding
├── rundeck-job.yaml                   # Example Rundeck job for EKS access
├── Rundeck-Community/                 # Source account IAM role (316978178737)
│   ├── ODC.json                       # Policy to assume RundeckEKSListerRole
│   └── trust_relationship.json        # Trust policy for EC2 service
└── RundeckEKSListerRole/              # Target account IAM role
    ├── EKSROPolicy.json               # Policy for EKS API access
    └── trust_relationsship.json       # Trust policy for Rundeck-Community role
```

## Components

### 1. Rundeck-Community (Source Account)

**Location:** `Rundeck-Community/`

This role is attached to the EC2 instances running Rundeck in account `316978178737`.

**Policy (`ODC.json`):**
- Allows assuming the `RundeckEKSListerRole` in any target account
- Uses external ID `EE55077E-A9DD-48C5-9A7F-3190DF36550C` for additional security

**Trust Relationship (`trust_relationship.json`):**
- Allows EC2 service to assume this role
- Restricted to source account `316978178737`
- Enables attachment to EC2 instance profiles

### 2. RundeckEKSListerRole (Target Account)

**Location:** `RundeckEKSListerRole/`

This role provides EKS access to clusters in target accounts.

**Policy (`EKSROPolicy.json`):**
- `eks:ListClusters` - List all EKS clusters
- `eks:DescribeCluster` - Get details about specific clusters
- `eks:ListNodegroups` - List node groups in clusters
- `eks:DescribeNodegroup` - Get details about node groups
- `eks:DescribeClusterVersions` - Get available Kubernetes versions
- `eks:CreateAccessEntry` - Create EKS access entries
- `eks:AccessKubernetesApi` - Access the Kubernetes API server
- Requires external ID `EE55077E-A9DD-48C5-9A7F-3190DF36550C`

**Trust Relationship (`trust_relationsship.json`):**
- Allows `Rundeck-Community` role from any account to assume this role
- Protected with external ID condition for enhanced security

### 3. EKS Access Entries

This solution uses **EKS Access Entries** (the modern approach) instead of the legacy aws-auth ConfigMap.

EKS Access Entries provide a more secure and manageable way to grant IAM principals access to EKS clusters:
- Direct API-based management (no manual ConfigMap editing)
- Maps `RundeckEKSListerRole` to the `rundeck-readonly` Kubernetes group
- Uses username `rundeck` for audit logging
- Requires cluster authentication mode to be set to `API_AND_CONFIG_MAP` or `API`

### 4. Kubernetes RBAC Configuration

**File:** `role_config.yaml`

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

### 5. Rundeck Job Example

**File:** `rundeck-job.yaml`

Provides an example Rundeck job that demonstrates:
- Assuming the RundeckEKSListerRole with proper credentials
- Configuring EKS cluster authentication mode
- Creating EKS access entries for the role
- Updating kubeconfig for cluster access
- Testing read-only permissions

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

### Step 2: Create RundeckEKSListerRole (Target Account)

In your target AWS account(s):

```bash
# Create the IAM role
aws iam create-role \
  --role-name RundeckEKSListerRole \
  --assume-role-policy-document file://RundeckEKSListerRole/trust_relationsship.json

# Attach the policy
aws iam create-policy \
  --policy-name EKSROPolicy \
  --policy-document file://RundeckEKSListerRole/EKSROPolicy.json

aws iam attach-role-policy \
  --role-name RundeckEKSListerRole \
  --policy-arn arn:aws:iam::<TARGET_ACCOUNT_ID>:policy/EKSROPolicy
```

**Important:** Remember to replace `<TARGET_ACCOUNT_ID>` with your actual target account ID.

### Step 3: Configure Kubernetes RBAC

Apply the Kubernetes ClusterRole and ClusterRoleBinding:

```bash
# Apply ClusterRole and ClusterRoleBinding
kubectl apply -f role_config.yaml
```

### Step 4: Configure EKS Access Entries

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

Create an access entry for the RundeckEKSListerRole:

```bash
aws eks create-access-entry \
  --cluster-name <CLUSTER_NAME> \
  --principal-arn arn:aws:iam::<TARGET_ACCOUNT_ID>:role/RundeckEKSListerRole \
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
  --principal-arn arn:aws:iam::<TARGET_ACCOUNT_ID>:role/RundeckEKSListerRole \
  --region <REGION>
```

### Step 5: Configure Rundeck

In your Rundeck configuration, set up the AWS CLI to use role chaining with external ID:

```bash
# On the Rundeck EC2 instance, configure AWS CLI profile
cat >> ~/.aws/config << EOF
[profile rundeck-eks]
role_arn = arn:aws:iam::<TARGET_ACCOUNT_ID>:role/RundeckEKSListerRole
credential_source = Ec2InstanceMetadata
external_id = EE55077E-A9DD-48C5-9A7F-3190DF36550C
EOF

# Test the configuration
aws eks describe-cluster --name <cluster-name> --region <region> --profile rundeck-eks

# Update kubeconfig for Rundeck
aws eks update-kubeconfig \
  --name <cluster-name> \
  --region <region> \
  --role-arn arn:aws:iam::<TARGET_ACCOUNT_ID>:role/RundeckEKSListerRole \
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
- Assume the RundeckEKSListerRole
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
  --role-arn arn:aws:iam::<TARGET_ACCOUNT_ID>:role/RundeckEKSListerRole \
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
  --principal-arn arn:aws:iam::<TARGET_ACCOUNT_ID>:role/RundeckEKSListerRole \
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

1. **EKS Access Entries:** Uses the modern EKS Access Entries API instead of ConfigMap for better security and auditability
2. **Least Privilege:** The RundeckEKSListerRole only has read-only access to EKS resources
3. **Cross-Account:** Uses IAM role assumption for cross-account access
4. **External ID Protection:** Uses external ID `EE55077E-A9DD-48C5-9A7F-3190DF36550C` to prevent the confused deputy problem
5. **Kubernetes RBAC:** Additional layer of security at the Kubernetes level
6. **Audit Trail:** All actions are logged with the username `rundeck`
7. **No Write Access:** The Kubernetes RBAC configuration explicitly only grants `get`, `list`, and `watch` verbs
8. **Account Isolation:** Source account (316978178737) is restricted via trust policies
9. **API-Based Management:** Access entries are managed via AWS APIs, reducing the risk of manual ConfigMap errors

## Troubleshooting

### Cannot Assume Role

**Error:** `An error occurred (AccessDenied) when calling the AssumeRole operation`

**Solutions:**
- Verify the trust relationship in RundeckEKSListerRole allows Rundeck-Community role
- Check that the EC2 instance has the correct instance profile (Rundeck-Community-Profile) attached
- Ensure the policy in Rundeck-Community role has permission to assume RundeckEKSListerRole
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

To grant additional read-only access, edit `role_config.yaml` and add more `apiGroups` and `resources`. Always use only `get`, `list`, and `watch` verbs for read-only access.

### Multiple Rundeck Users

To support multiple Rundeck instances or environments:

1. Create additional IAM roles in the target account
2. Create separate EKS access entries for each role using `aws eks create-access-entry`
3. Create separate ClusterRoleBindings if different permissions are needed

Example for a second Rundeck instance:
```bash
aws eks create-access-entry \
  --cluster-name <CLUSTER_NAME> \
  --principal-arn arn:aws:iam::<TARGET_ACCOUNT_ID>:role/RundeckEKSListerRole-Dev \
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

