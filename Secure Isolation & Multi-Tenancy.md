# Secure Isolation & Multi-Tenancy

**Name:** Muhammad Arif Shafira Bin Shahrin Amri  
**Course:** IKB42603 Cloud Computing  
**Lab:** Lab 2 — Secure Isolation and Multitenancy

## Objective

This lab demonstrates how a shared Kubernetes cluster can host multiple tenants while reducing cross-tenant risk. The implementation uses separate namespaces, resource quotas, network policies, RBAC, and secure data removal.

## Environment

- Kubernetes cluster: `csse-lab2` (kind)
- Tenants: `tenant-a` and `tenant-b`
- Workload: `web` pod and ClusterIP Service in each tenant
- Test image: `curlimages/curl`
- Persistent-data demonstration: Docker volume `csse-vol`

## Task 1 — Create isolated tenant workloads

Two namespaces were prepared: `tenant-a` and `tenant-b`. Each namespace contains its own `web` pod and a `web` ClusterIP Service. Although the service names are the same, namespaces keep their Kubernetes object names separate.

Commands used to confirm the workloads:

```bash
kubectl get pods,svc -n tenant-a
kubectl get pods,svc -n tenant-b
```

Observed results:

- `tenant-a`: `web-79d9f568b9-mhg1z` was Running and the `web` service had ClusterIP `10.96.8.177` on port `80/TCP`.
- `tenant-b`: `web-79d9f568b9-5mbnc` was Running and the `web` service had ClusterIP `10.96.204.36` on port `80/TCP`.

Evidence: [Tenant A workload](/C:/Users/HP/Desktop/UniKL/Semester%205/Cloud%20Computing/Lab%202/Task%201/Task%201%20-%20a.png) and [Tenant B workload](/C:/Users/HP/Desktop/UniKL/Semester%205/Cloud%20Computing/Lab%202/Task%201/Task%201%20-b.png).

## Task 2 — Demonstrate the default network behaviour

Before applying a NetworkPolicy, a temporary curl pod in `tenant-a` accessed the `tenant-b` service address:

```bash
kubectl -n tenant-a run probe --rm -it --image=curlimages/curl --restart=Never -- \
  curl -s -m 5 http://10.96.204.36 -o /dev/null -w 'HTTP %{http_code}\n'
```

The request returned `HTTP 200`. This shows that namespaces alone do **not** restrict pod-to-pod traffic; Kubernetes networking is permissive by default unless a supported network-policy implementation enforces rules.

Evidence: [Initial cross-tenant connectivity test](/C:/Users/HP/Desktop/UniKL/Semester%205/Cloud%20Computing/Lab%202/Task%202/Task%202.png).

## Task 3 — Limit tenant resource consumption

A `ResourceQuota` named `tenant-a-quota` was applied in `tenant-a` and inspected with:

```bash
kubectl describe resourcequota tenant-a-quota -n tenant-a
```

The quota permits at most five pods, one CPU request, and `512Mi` of memory requests. At verification time, one pod was in use and no CPU or memory requests had been consumed.

| Resource | Used | Hard limit |
| --- | ---: | ---: |
| Pods | 1 | 5 |
| Requested CPU | 0 | 1 |
| Requested memory | 0 | 512Mi |

Resource quotas prevent one tenant from exhausting shared cluster capacity and affecting other tenants.

Evidence: [ResourceQuota verification](/C:/Users/HP/Desktop/UniKL/Semester%205/Cloud%20Computing/Lab%202/Task%203/Task%203.png).

## Task 4 — Enforce default-deny ingress for tenant B

A namespace-wide NetworkPolicy was created in `tenant-b`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: tenant-b
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

Applied with:

```bash
kubectl apply -f -
```

The empty `podSelector` selects every pod in `tenant-b`. Because no ingress rules are present, inbound traffic to those selected pods is denied. Re-running the curl test from `tenant-a` returned `HTTP 000` and the probe terminated with an error, confirming that the previous access path was blocked.

Evidence: [Policy creation](/C:/Users/HP/Desktop/UniKL/Semester%205/Cloud%20Computing/Lab%202/Task%204/Task%204%20-%201.png) and [Blocked connectivity verification](/C:/Users/HP/Desktop/UniKL/Semester%205/Cloud%20Computing/Lab%202/Task%204/Task%204%20-%20Verify.png).

## Task 5 — Verify least-privilege RBAC

The permissions of the configured service account (`$SA`) were checked with Kubernetes' authorization test:

```bash
kubectl auth can-i get secrets -n tenant-a --as=$SA
kubectl auth can-i get secrets -n tenant-b --as=$SA
```

The service account was allowed to get secrets in `tenant-a` (`yes`) and denied the same action in `tenant-b` (`no`). This verifies namespace-scoped RBAC: the identity has only the permissions assigned for its own tenant.

Evidence: [Allowed in tenant A](/C:/Users/HP/Desktop/UniKL/Semester%205/Cloud%20Computing/Lab%202/Task%205/Task%205%20-1.png) and [Denied in tenant B](/C:/Users/HP/Desktop/UniKL/Semester%205/Cloud%20Computing/Lab%202/Task%205/Task%205%20-%202.png).

## Task 6 — Remove sensitive data securely

First, a temporary Alpine container wrote a record containing sensitive content into a mounted Docker volume, synchronized it, removed the file, searched the volume, and reported completion:

```bash
docker run --rm -v csse-vol:/data alpine sh -c \
  'echo SENSITIVE-PATIENT-RECORD > /data/phi.txt; sync; rm /data/phi.txt; \
   grep -a SENSITIVE /data/* 2>/dev/null; echo scan-done'
```

The container then created a second file and overwrote its contents with zero bytes before deleting it:

```bash
docker run --rm -v csse-vol:/data alpine sh -c \
  'echo SENSITIVE > /data/phi2.txt; sync; \
   dd if=/dev/zero of=/data/phi2.txt bs=1k count=1 conv=notrunc; rm /data/phi2.txt; \
   echo wiped'
```

The `dd` output confirms that 1,024 bytes were written and the command reported `wiped`. This is a demonstration of overwrite-before-delete; it should not be treated as a universal guarantee on copy-on-write filesystems, SSDs, managed storage, or snapshot-enabled volumes.

Evidence: [Initial removal and scan](/C:/Users/HP/Desktop/UniKL/Semester%205/Cloud%20Computing/Lab%202/Task%206/Task%206%20-%201.png) and [Overwrite-before-delete](/C:/Users/HP/Desktop/UniKL/Semester%205/Cloud%20Computing/Lab%202/Task%206/Task%206%20-%202.png).

## Final verification

The final checks showed that `tenant-b/default-deny-ingress` exists and that `tenant-a-quota` remains configured with its stated limits:

```bash
kubectl get networkpolicy -A
kubectl describe resourcequota tenant-a-quota -n tenant-a
```

Evidence: [Final verification](/C:/Users/HP/Desktop/UniKL/Semester%205/Cloud%20Computing/Lab%202/Verification%20Command.png).

## Cleanup and wrap-up

The lab resources were removed with:

```bash
kind delete cluster --name csse-lab2
docker volume rm csse-vol
```

The output confirms deletion of the kind cluster and Docker volume.

Evidence: [Cleanup confirmation](/C:/Users/HP/Desktop/UniKL/Semester%205/Cloud%20Computing/Lab%202/Cleanup%20%26%20Wrap%20Up.png).

## Short-answer question

### Question 1: Default Network Behavior & Risks
**Why can containers in different namespaces reach each other by default, and why is that dangerous in multi-tenant cloud?**

* **Reason:** Kubernetes uses an open, flat networking model by default where all pods across namespaces can communicate using direct IP addresses. Namespaces only provide logical grouping and administrative scopes, not network boundaries.
* **Risk:** In a multi-tenant cloud, an attacker who compromises a container in Tenant A can freely perform lateral movement, scan internal networks, and access sensitive endpoints or services running in Tenant B on the same physical cluster.

---

### Question 2: Default-Deny Principle & Network Policy
**Explain the default-deny principle and how your Network Policy implements it.**

* **Default-Deny Principle:** A Zero Trust security concept stating that all network traffic should be implicitly blocked unless an explicit rule permits it (least privilege access).
* **Implementation:** The `default-deny-ingress` NetworkPolicy applies an empty `podSelector: {}` in `tenant-b` with `policyTypes: [Ingress]`. This selects all pods in that namespace and drops all incoming traffic that isn't explicitly whitelisted.

---

### Question 3: Container vs. VM Isolation Strength
**How do virtual machines and containers differ in isolation strength? When would you add a VM boundary?**

* **Isolation Difference:** 
  * *Containers* share the host OS kernel and rely on Linux primitives (`namespaces`, `cgroups`), making them vulnerable to kernel-level privilege escalation or host compromise.
  * *Virtual Machines (VMs)* use a hypervisor to provide dedicated hardware virtualization and run independent guest kernels, offering significantly stronger boundary isolation.
* **When to add a VM boundary:** When running untrusted/third-party code, untrusted multitenant workloads, or highly sensitive applications subject to strict regulatory compliance (e.g., PCI-DSS, HIPAA).

---

### Question 4: Data Remanence & Cryptographic Erasure
**What is data remanence, and why is cryptographic erasure the preferred cloud solution?**

* **Data Remanence:** The residual physical or logical data that remains on underlying storage media after standard file deletion commands (like `rm`) are run.
* **Why Cryptographic Erasure:** Cloud tenants do not have physical access or low-level block access to shared cloud storage drives, making physical destruction or overwrite wiping unfeasible. Cryptographic erasure encrypts data at rest and destroys the specific decryption keys, rendering the residual data on shared storage permanently unrecoverable.

---

### Question 5: Isolation Dimension Mapping
**Which of the three isolation dimensions (compute, network, storage) did each task exercise?**

| Task | Primary Isolation Dimension |
| :--- | :--- |
| **Task 1: Two Tenants on One Cluster** | **Compute Isolation** (Logical separation via Kubernetes Namespaces) |
| **Task 2: Observe Default-Open Risk** | **Network Isolation** (Demonstrating missing network boundaries) |
| **Task 3: Contain Noisy Neighbour** | **Compute Isolation** (Resource allocation control via `ResourceQuota`) |
| **Task 4: Default-Deny Network Isolation** | **Network Isolation** (Traffic segmentation via `NetworkPolicy`) |
| **Task 5: Storage & Secret Isolation** | **Storage Isolation** (RBAC authorization protecting secrets) |
| **Task 6: Data Remanence & Secure Deletion** | **Storage Isolation** (Volume lifecycle, retention, and sanitization) |

## Conclusion

The lab successfully demonstrated practical isolation controls for a shared Kubernetes environment. The initial `HTTP 200` response established the default cross-tenant network risk, while the later `HTTP 000` result demonstrated mitigation using default-deny ingress. Quotas constrained resource use, RBAC limited secret access to the appropriate namespace, and the data-removal exercise reinforced that sensitive tenant data must be handled deliberately throughout its lifecycle.
