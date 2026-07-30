# Cloud Account Security and IAM — Lab 1 Report

| Item | Details |
|---|---|
| Student | Arif |
| Course | IKB42603 |
| Lab | Lab 1: Account Security and IAM |
| Date | 30 July 2026 |
| Platforms | AWS CLI with LocalStack; Kubernetes with Kind |

## Objective

Configure and verify secure access controls in two local cloud environments:

1. AWS Identity and Access Management (IAM), including group-based administration, least-privilege permissions, and access-key management.
2. Kubernetes role-based access control (RBAC), including namespaces, a service account, a role, and a role binding.

> Commands below use `$EP` as the AWS CLI LocalStack endpoint option, for example: `EP="--endpoint-url=http://localhost:4566"`.

## Session A — AWS Account Security and IAM

### Environment preparation

1. Start or restart LocalStack.

   ```bash
   docker restart localstack
   ```

2. Check that the LocalStack services are healthy.

   ```bash
   curl http://localhost:4566/_localstack/health
   ```

   The IAM service reported `available`.

3. Verify AWS CLI connectivity and the active identity.

   ```bash
   aws --endpoint-url=http://localhost:4566 sts get-caller-identity
   ```

   The command returned LocalStack account `000000000000` and the root ARN, confirming that the local AWS-compatible environment was ready.

**Evidence:** [LocalStack health check](<Session A/Local Stack health.png>) and [AWS CLI identity check](<Session A/AWS CLI.png>).

### Task 1 — Verify the local AWS environment

| Check | Result |
|---|---|
| LocalStack container | Started successfully. |
| IAM service health | Available. |
| AWS CLI endpoint | Connected successfully to `http://localhost:4566`. |

**Outcome:** The environment was ready for IAM configuration.

### Task 2 — Create an administrative group and user

1. Create the `Admins` group.

   ```bash
   aws $EP iam create-group --group-name Admins
   ```

2. Attach the AWS managed `AdministratorAccess` policy to the group.

   ```bash
   aws $EP iam attach-group-policy \
     --group-name Admins \
     --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
   ```

3. Create the administrative user.

   ```bash
   aws $EP iam create-user --user-name CloudAdmin_Arif
   ```

4. Add the user to the group and verify group membership.

   ```bash
   aws $EP iam add-user-to-group \
     --group-name Admins \
     --user-name CloudAdmin_Arif

   aws $EP iam get-group --group-name Admins
   ```

**Outcome:** `CloudAdmin_Arif` inherits administrative permissions from the `Admins` group. This centralizes administrator access management at the group level.

**Evidence:** [Group creation and policy attachment](<Session A/Task 2/1.png>), [user creation](<Session A/Task 2/2.png>), and [membership verification](<Session A/Task 2/3.png>).

### Task 3 — Create a least-privilege analyst user

1. Create the analyst user.

   ```bash
   aws $EP iam create-user --user-name Analyst_Arif
   ```

2. Grant only S3 read-only access and verify the attached policy.

   ```bash
   aws $EP iam attach-user-policy \
     --user-name Analyst_Arif \
     --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

   aws $EP iam list-attached-user-policies --user-name Analyst_Arif
   ```

**Outcome:** `Analyst_Arif` has `AmazonS3ReadOnlyAccess` rather than full administrative access. This implements least privilege because the user can read S3 resources without receiving write or account-management permissions.

**Evidence:** [Analyst user creation](<Session A/Task 3/1.png>) and [policy attachment and verification](<Session A/Task 3/2.png>).

### Task 4 — Create, inspect, and deactivate an access key

1. Create an access key for the analyst.

   ```bash
   aws $EP iam create-access-key --user-name Analyst_Arif
   ```

2. Inspect the user's keys.

   ```bash
   aws $EP iam list-access-keys --user-name Analyst_Arif
   ```

3. Deactivate the created key.

   ```bash
   aws $EP iam update-access-key \
     --user-name Analyst_Arif \
     --access-key-id <access-key-id> \
     --status Inactive
   ```

**Outcome:** The analyst key was initially active, then changed to `Inactive`. The secret access key and access-key ID are not reproduced in this report because credentials are sensitive. Deactivation prevents future use while retaining the key record for review or rotation.

**Evidence:** [Access-key creation](<Session A/Task 4/1.png>), [key listing](<Session A/Task 4/2.png>), and [key deactivation](<Session A/Task 4/3.png>).

## Session B — Kubernetes Access Control (RBAC)

### Setup — Create and verify a local Kubernetes cluster

1. Create a Kind cluster named `ccse-lab1`.

   ```bash
   kind create cluster --name ccse-lab1
   ```

2. Confirm control-plane access.

   ```bash
   kubectl cluster-info --context kind-ccse-lab1
   ```

3. Confirm that the node is ready.

   ```bash
   kubectl get nodes
   ```

**Outcome:** The `ccse-lab1-control-plane` node was `Ready`, so the cluster was available for RBAC configuration.

**Evidence:** [Cluster creation](<Session B/Setup/Create a Local Kubernetes Cluster 1.png>), [cluster information](<Session B/Setup/Create a Local Kubernetes Cluster 2.png>), and [node status](<Session B/Setup/Cluster 3.png>).

### Task 5 — Create development and production namespaces

1. Create the `dev` namespace.

   ```bash
   kubectl create namespace dev
   ```

2. Create the `prod` namespace.

   ```bash
   kubectl create namespace prod
   ```

3. List namespaces to verify both were created.

   ```bash
   kubectl get namespaces
   ```

**Outcome:** Both `dev` and `prod` were active. Namespace separation provides a boundary for applying different access controls to development and production workloads.

**Evidence:** [Namespace creation and verification](<Session B/Task 5/Screenshot 2026-07-30 235420.png>).

### Task 6 — Configure namespace-scoped RBAC for a developer

1. Create the `dev-user` service account in the `dev` namespace.

   ```bash
   kubectl create serviceaccount dev-user -n dev
   ```

2. Create the `pod-reader` role in `dev`. It allows `get`, `list`, and `watch` actions on pods.

   ```bash
   kubectl create role pod-reader -n dev \
     --verb=get,list,watch \
     --resource=pods
   ```

3. Bind the role to `dev-user` in the same namespace.

   ```bash
   kubectl create rolebinding dev-user-binding -n dev \
     --role=pod-reader \
     --serviceaccount=dev:dev-user
   ```

**Outcome:** `dev-user` receives only read access to pods in `dev`. The permissions are namespace-scoped and do not automatically extend to `prod`.

**Evidence:** [Service account, role, and role binding](<Session B/Task 6/T6.png>).

### Task 7 — Verify effective Kubernetes permissions

1. Use the service account identity to check whether it can list pods in `dev`.

   ```bash
   SA="system:serviceaccount:dev:dev-user"
   kubectl auth can-i list pods -n dev --as=$SA
   ```

   Result: `yes`.

2. Check whether it can delete pods in `dev`.

   ```bash
   kubectl auth can-i delete pods -n dev --as=$SA
   ```

   Result: `no`.

3. Check whether it can list pods in `prod`.

   ```bash
   kubectl auth can-i list pods -n prod --as=$SA
   ```

   Result: `no`.

4. Inspect the role binding configuration.

   ```bash
   kubectl get rolebinding dev-user-binding -n dev -o yaml
   ```

   The binding references the `pod-reader` role and the `dev-user` service account in the `dev` namespace.

**Outcome:** The permission checks prove that RBAC is working as intended: the service account can list pods only in `dev`, cannot delete them, and has no access to pods in `prod`.

**Evidence:** [Allowed pod listing](<Session B/Task 7/T7 1.png>), [denied pod deletion](<Session B/Task 7/T7 2.png>), [denied production access](<Session B/Task 7/T7 3.png>), and [role-binding YAML](<Session B/Verification Command.png>).

### Clean up

Remove the local resources after completing the verification.

```bash
kind delete cluster --name ccse-lab1
docker stop localstack && docker rm localstack
```

**Outcome:** The Kind cluster and LocalStack container were removed.

**Evidence:** [Cleanup commands](<Session B/Clean up/Cleanup.png>).

## Security Summary

| Control | Implementation | Verification |
|---|---|---|
| Group-based administration | `CloudAdmin_Arif` was placed in `Admins`, which has `AdministratorAccess`. | `get-group` output showed the user as a group member. |
| Least privilege in AWS | `Analyst_Arif` received only `AmazonS3ReadOnlyAccess`. | Attached-policy listing returned the read-only S3 policy. |
| Credential lifecycle management | The analyst access key was deactivated. | The update command set its status to `Inactive`. |
| Kubernetes isolation | `dev` and `prod` namespaces were created. | Namespace listing showed both as active. |
| Namespace-scoped RBAC | `dev-user` was bound to the pod read-only role only in `dev`. | `kubectl auth can-i` allowed reading in `dev` and denied deletion and `prod` access. |

## Conclusion

The lab successfully implemented cloud account security controls across AWS IAM and Kubernetes RBAC. Administrative access was managed through a group, the analyst was limited to read-only S3 permissions, and a credential was deactivated after review. In Kubernetes, the `dev-user` service account received only the required pod read permissions in the `dev` namespace, with restricted actions and no production access. These results demonstrate group-based access control, least privilege, credential hygiene, and environment isolation.
