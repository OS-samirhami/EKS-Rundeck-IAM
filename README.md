# EKS-Rundeck-IAM

A comprehensive solution for configuring IAM roles and Kubernetes RBAC permissions to enable Rundeck automation server to access Amazon EKS clusters with read-only privileges.

## Overview

This project provides the necessary IAM policies, trust relationships, and Kubernetes RBAC configurations to securely integrate Rundeck with Amazon Elastic Kubernetes Service (EKS). It implements a cross-account role assumption pattern that allows Rundeck running on EC2 instances to access EKS cluster resources with read-only permissions.

## Architecture

The solution implements a two-tier IAM role chain:

1. **RundeckEKSCARole** (Account: 512508756184)
   - Attached to EC2 instances running Rundeck
   - Can assume the EKSRORole in the target account

2. **EKSRORole** (Account: 621352866489)
   - Assumed by RundeckEKSCARole
   - Has permissions to list and describe EKS clusters
   - Mapped to Kubernetes RBAC group `rundeck-readonly`

3. **Kubernetes RBAC**
   - ClusterRole `rundeck-ro` with read-only permissions
   - Bound to the `rundeck-readonly` group

## Repository Structure

```
.
├── README.md                          # This file
├── .gitignore                         # Git ignore rules (Python-focused)
├── aws-auth.yaml                      # EKS aws-auth ConfigMap configuration
├── role_config.yaml                   # Kubernetes ClusterRole and ClusterRoleBinding
├── RundeckEKSCARole /                 # Source account IAM role
│   ├── RundeckEKSCAPolicy.json        # Policy to assume EKSRORole
│   └── trust_relationship.json        # Trust policy for EC2 service
└── EKSRORole/                         # Target account IAM role
    ├── EKSROPolicy.json               # Policy for EKS API access
    └── trust_relationsship.json       # Trust policy for RundeckEKSCARole
```

## Components

### 1. RundeckEKSCARole (Source Account)

**Location:** `RundeckEKSCARole/`

This role is attached to the EC2 instances running Rundeck in account `512508756184`.

**Policy (`RundeckEKSCAPolicy.json`):**
- Allows assuming the `EKSRORole` in the target account (621352866489)

**Trust Relationship (`trust_relationship.json`):**
- Allows EC2 service to assume this role
- Enables attachment to EC2 instance profiles

### 2. EKSRORole (Target Account)

**Location:** `EKSRORole/`

This role provides read-only access to EKS clusters in account `621352866489`.

**Policy (`EKSROPolicy.json`):**
- `eks:ListClusters` - List all EKS clusters
- `eks:DescribeCluster` - Get details about specific clusters

**Trust Relationship (`trust_relationsship.json`):**
- Allows `RundeckEKSCARole` from account 512508756184 to assume this role

### 3. EKS aws-auth ConfigMap

**File:** `aws-auth.yaml`

Maps the IAM role to Kubernetes RBAC groups:
- Maps `EKSRORole` to the `rundeck-readonly` group
- Uses username `rundeck` for audit logging
- Also includes node role mapping for EKS worker nodes

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

## Setup Instructions

### Prerequisites

- AWS CLI configured with appropriate credentials
- kubectl configured to access your EKS cluster
- Access to both AWS accounts (512508756184 and 621352866489)
- Rundeck running on EC2 instances

### Step 1: Create RundeckEKSCARole (Source Account)

In account `512508756184`:

```bash
# Create the IAM role
aws iam create-role \
  --role-name RundeckEKSCARole \
  --assume-role-policy-document file://RundeckEKSCARole\ /trust_relationship.json

# Attach the policy
aws iam create-policy \
  --policy-name RundeckEKSCAPolicy \
  --policy-document file://RundeckEKSCARole\ /RundeckEKSCAPolicy.json

aws iam attach-role-policy \
  --role-name RundeckEKSCARole \
  --policy-arn arn:aws:iam::512508756184:policy/RundeckEKSCAPolicy

# Create and attach instance profile to EC2 instances
aws iam create-instance-profile --instance-profile-name RundeckEKSCAProfile
aws iam add-role-to-instance-profile \
  --instance-profile-name RundeckEKSCAProfile \
  --role-name RundeckEKSCARole
```

### Step 2: Create EKSRORole (Target Account)

In account `621352866489`:

```bash
# Create the IAM role
aws iam create-role \
  --role-name EKSRORole \
  --assume-role-policy-document file://EKSRORole/trust_relationsship.json

# Attach the policy
aws iam create-policy \
  --policy-name EKSROPolicy \
  --policy-document file://EKSRORole/EKSROPolicy.json

aws iam attach-role-policy \
  --role-name EKSRORole \
  --policy-arn arn:aws:iam::621352866489:policy/EKSROPolicy
```

### Step 3: Configure Kubernetes RBAC

Apply the Kubernetes configurations:

```bash
# Apply ClusterRole and ClusterRoleBinding
kubectl apply -f role_config.yaml

# Update aws-auth ConfigMap (or apply if creating new cluster)
kubectl apply -f aws-auth.yaml
```

**Note:** If you already have an existing `aws-auth` ConfigMap, you should edit it instead:

```bash
kubectl edit configmap aws-auth -n kube-system
```

Then add the EKSRORole mapping to the existing `mapRoles` section.

### Step 4: Configure Rundeck

In your Rundeck configuration, set up the AWS CLI to use role chaining:

```bash
# On the Rundeck EC2 instance, configure AWS CLI
aws configure set role_arn arn:aws:iam::621352866489:role/EKSRORole
aws configure set credential_source Ec2InstanceMetadata

# Test the configuration
aws eks describe-cluster --name <your-cluster-name> --region <region>

# Update kubeconfig for Rundeck
aws eks update-kubeconfig --name <your-cluster-name> --region <region> --role-arn arn:aws:iam::621352866489:role/EKSRORole
```

## Verification

### Verify IAM Role Chain

```bash
# From Rundeck EC2 instance, test assuming the role
aws sts assume-role \
  --role-arn arn:aws:iam::621352866489:role/EKSRORole \
  --role-session-name rundeck-test

# List EKS clusters
aws eks list-clusters --region <region>
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

1. **Least Privilege:** The EKSRORole only has read-only access to EKS resources
2. **Cross-Account:** Uses IAM role assumption for cross-account access
3. **Kubernetes RBAC:** Additional layer of security at the Kubernetes level
4. **Audit Trail:** All actions are logged with the username `rundeck`
5. **No Write Access:** The configuration explicitly only grants `get`, `list`, and `watch` verbs

## Troubleshooting

### Cannot Assume Role

**Error:** `An error occurred (AccessDenied) when calling the AssumeRole operation`

**Solutions:**
- Verify the trust relationship in EKSRORole allows RundeckEKSCARole
- Check that the EC2 instance has the correct instance profile attached
- Ensure the policy in RundeckEKSCARole has permission to assume EKSRORole

### Cannot Access EKS Cluster

**Error:** `error: You must be logged in to the server (Unauthorized)`

**Solutions:**
- Verify aws-auth ConfigMap is correctly configured
- Check that the role ARN in aws-auth matches exactly
- Ensure ClusterRoleBinding is applied correctly
- Verify the EKS cluster's API endpoint is accessible from Rundeck

### Permission Denied in Kubernetes

**Error:** `Error from server (Forbidden): pods is forbidden`

**Solutions:**
- Verify the ClusterRole and ClusterRoleBinding are applied
- Check that the group name matches in aws-auth and ClusterRoleBinding
- Ensure you're accessing the correct namespace (if namespace-specific issues)

## Customization

### Adding More Permissions

To grant additional read-only access, edit `role_config.yaml` and add more `apiGroups` and `resources`. Always use only `get`, `list`, and `watch` verbs for read-only access.

### Multiple Rundeck Users

To support multiple Rundeck instances or environments:

1. Create additional IAM roles in the target account
2. Add mappings in `aws-auth.yaml`
3. Create separate ClusterRoleBindings if different permissions are needed

### Namespace-Specific Access

To restrict access to specific namespaces, replace `ClusterRole` and `ClusterRoleBinding` with `Role` and `RoleBinding` in specific namespaces.

## References

- [Amazon EKS IAM Roles](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [Managing aws-auth ConfigMap](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html)
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

