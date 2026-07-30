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

**Evidence:** <img width="966" height="655" alt="Local Stack health" src="https://github.com/user-attachments/assets/c5320899-641e-45bd-af81-21901acff876" />
 <img width="613" height="127" alt="AWS CLI" src="https://github.com/user-attachments/assets/38fea938-3f7a-4979-a23d-66ea8fc1e701" />

### Task 1: Cloud Identity Landscape

| Concept | AWS Term | Purpose |
| :--- | :--- | :--- |
| **All-powerful owner** | Root user | The main owner account with full, unrestricted access to everything. Use it only for setup. |
| **Human/app identity** | IAM User | A permanent login for a specific person or app needing ongoing access. |
| **Permission bundle** | IAM Policy | A list of rules defining what an identity is allowed or denied to do. |
| **Collection of users** | IAM Group | A team or list of users used to give the same permissions to everyone at once. |
| **Temporary identity** | IAM Role | A temporary pass that grants short-term access without needing permanent keys. |

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

**Evidence:** <img width="1020" height="209" alt="image" src="https://github.com/user-attachments/assets/2cfd0317-e502-4c36-89e9-0668d8596daa" />
<img width="651" height="189" alt="image" src="https://github.com/user-attachments/assets/dba216ec-7b77-4fc3-86cd-ff9d912cb81f" />
<img width="719" height="388" alt="image" src="https://github.com/user-attachments/assets/dffd7a81-f747-4c20-900e-8e3cb2d46c47" />


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

**Evidence:** 

<img width="602" height="181" alt="image" src="https://github.com/user-attachments/assets/f64ef015-3f24-4df6-874d-9e78f8d29902" />
<img width="1032" height="220" alt="2" src="https://github.com/user-attachments/assets/ac31ace6-6caa-4a1e-95dd-b15106cc6f14" />


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

**Evidence:** 

<img width="600" height="187" alt="image" src="https://github.com/user-attachments/assets/5dd2f1ee-b8fe-4d95-affb-3f75ddc365cf" />
<img width="569" height="203" alt="image" src="https://github.com/user-attachments/assets/6ea9c1c0-cf1b-471d-85cd-6b513d5dff40" />
 <img width="922" height="49" alt="3" src="https://github.com/user-attachments/assets/9558f589-17c0-4ec2-8fcf-4b826dfd61dd" />


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

**Evidence:** 

<img width="772" height="257" alt="Create a Local Kubernetes Cluster 1" src="https://github.com/user-attachments/assets/fd39f180-260a-482d-8b36-4560f1de0227" />
 <img width="874" height="120" alt="Create a Local Kubernetes Cluster 2" src="https://github.com/user-attachments/assets/663b248f-01d1-4461-9625-1ef1f350ddf9" />
 <img width="552" height="83" alt="Cluster 3" src="https://github.com/user-attachments/assets/8a7f1f73-7320-4626-9879-4e2ea217e68a" />


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

**Evidence:** 

<img width="420" height="302" alt="Screenshot 2026-07-30 235420" src="https://github.com/user-attachments/assets/f8417152-a2ea-4ecd-87fd-960152e9176c" />


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

**Evidence:**

<img width="559" height="227" alt="T6" src="https://github.com/user-attachments/assets/9564c699-ad44-4ca9-9f19-d079d4f759f2" />


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

**Evidence:**

<img width="506" height="305" alt="Verification Command" src="https://github.com/user-attachments/assets/e40fc43c-2f06-49b2-8310-361c8f51361f" />


### Clean up

Remove the local resources after completing the verification.

```bash
kind delete cluster --name ccse-lab1
docker stop localstack && docker rm localstack
```

**Outcome:** The Kind cluster and LocalStack container were removed.

**Evidence:** 

<img width="431" height="157" alt="Cleanup" src="https://github.com/user-attachments/assets/a2f2d907-83fa-4d3b-97f4-bab4c59f3dea" />


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
