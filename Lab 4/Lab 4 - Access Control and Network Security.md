# Lab 4: Access Control and Network Security

| Item | Details |
|---|---|
| Student name | MUHAMMAD ARIF SHAFIRA BIN SHAHRIN AMRI |
| Lab | L02 - B04 |
| Course | IKB42603 Cloud Computing |
| Topic | Access Control and Network Security |

## Objective

This lab demonstrates authentication, multi-factor authentication (MFA), Kubernetes role-based access control (RBAC), network segmentation, default-deny firewalling, and container hardening.

## Task 1 — Password-Protected Service (Authentication)

### Steps performed

1. A password file for user `student` was generated with `htpasswd`.
2. An Nginx service was started on port `8080` with HTTP Basic Authentication configured.
3. The service was tested without credentials and then with `student:P@ssw0rd!`.

### Result

The authenticated request returned `Authenticated OK`.

![Task 1 terminal output](<Task 1/Task 1.png>)

> **Observation:** The unauthenticated request in the supplied evidence returned `200`, whereas the required result is `401`. Therefore, the screenshot does not demonstrate that authentication was enforced. In this Nginx configuration, `return 200` can run before the access/authentication phase. A suitable correction is to serve a real file instead of using `return`, then retest until the no-credentials command returns `401`.

## Task 2 — Second Factor with TOTP (MFA)

### Steps performed

1. A random Base32 shared secret was generated.
2. `oathtool --totp -b "$SECRET"` generated a current six-digit TOTP code.
3. The generated code was supplied to the validation command.

### Result

The validation returned `MFA OK`, showing that the TOTP code matched the shared secret and current time window.

<img width="612" height="135" alt="image" src="https://github.com/user-attachments/assets/ad9a37dd-c96f-455b-97a7-b58d1d738577" />

<img width="713" height="116" alt="image" src="https://github.com/user-attachments/assets/fdd7c452-a9e1-4e9f-ba34-e9386d603575" />


## Task 3 — Kubernetes RBAC Authorization

### Steps performed

1. A Kubernetes cluster and `app` namespace were created.
2. Service account `dev` was created.
3. Role `dev-role` was limited to the `get` and `list` verbs on `pods`.
4. The role was bound to `app:dev`, then permissions were tested with `kubectl auth can-i`.

### Result

The developer account could list pods, but could not create deployments or delete pods. This confirms least-privilege RBAC.

![Task 3 RBAC results](<Task 3/Task 3.png>)

| Requested action | Result |
|---|---|
| List pods | Yes |
| Create deployment | No |
| Delete pods | No |

## Task 4 — Network Segmentation

### Steps performed

1. Two isolated Docker networks, `frontend-net` and `backend-net`, were created.
2. The `db` container joined only `backend-net`.
3. The `app` container joined both networks and the `web` container joined only `frontend-net`.
4. Connectivity from `web` and `app` to the database was tested.

### Result

`web` could not reach `db` (`BLOCKED`), while `app` could reach it (`REACHABLE`).

![Task 4 segmentation results](<Task 4/Task 4.png>)

## Task 5 — Default-Deny Firewall

### Steps performed

1. A temporary Alpine container was run with `NET_ADMIN` capability.
2. The INPUT chain policy was set to `DROP`.
3. Explicit rules allowed TCP port `443` and loopback traffic.
4. The ruleset was listed with `iptables -L INPUT -n`.

### Result

The INPUT chain has a `DROP` default policy, with only HTTPS (TCP/443) and loopback explicitly allowed.

![Task 5 firewall rules](<Task 5/Task 5.png>)

## Task 6 — Container / Host Hardening and Vulnerability Scan

### Steps performed

1. The Nginx container was run as the non-root user `1000:1000`.
2. Its root filesystem was made read-only.
3. Linux capabilities were dropped, privilege escalation was disabled, and `/tmp` was supplied as temporary in-memory storage.
4. `docker inspect` checked the user and read-only setting.
5. Trivy scanned `nginx:alpine` for HIGH and CRITICAL vulnerabilities.

### Result

The inspection shows `User=1000:1000` and `ReadOnly=true`. The supplied Trivy summary reports zero vulnerabilities at the selected severity levels.

![Task 6 hardened-container inspection](<Task 6/Task 6 - 1.png>)

![Task 6 Trivy summary](<Task 6/Task 6 - 2.png>)

## Short-Answer Questions

### 1. Explain the difference between authentication and authorization using Tasks 1 and 3.

Authentication verifies **who** a requester is. In Task 1, HTTP Basic Authentication checks whether the requester supplies valid credentials for `student`. Authorization determines **what an authenticated identity may do**. In Task 3, the already identified `app:dev` service account is authorised to list pods but denied permission to create deployments or delete pods. Thus, authentication comes first, while authorization applies permissions after identity is known.

### 2. Why is MFA so effective, and which attacks does it defeat?

MFA is effective because a stolen or guessed password alone is insufficient: the attacker also needs the time-limited code from the user's authenticator device. TOTP codes expire quickly and change regularly. MFA greatly reduces the success of password spraying, credential stuffing, brute-force password guessing, and reuse of passwords leaked in data breaches. It does not completely prevent real-time phishing or malware that steals both the password and current code; phishing-resistant methods such as FIDO2 provide stronger protection for those cases.

### 3. How does network segmentation limit the damage of a compromised web server?

Segmentation places the web tier and database tier on different networks. In Task 4, a compromised `web` container cannot directly resolve or connect to `db`, so it cannot immediately query, alter, or exfiltrate database data. Only the application tier is connected to both networks. This contains lateral movement and forces access through the intended application path, where further controls can be applied.

### 4. What does a default-deny firewall policy achieve, and how does it relate to cloud security groups?

Default deny blocks all inbound traffic unless a rule explicitly allows it. In Task 5, TCP/443 and loopback are permitted while all other input is dropped. Cloud security groups work on the same allow-list principle: traffic is denied by default and administrators add only the required protocol, port, source, and destination rules. This minimises exposed services and accidental access.

### 5. List the hardening measures you applied and the attack surface each one removes.

| Hardening measure | Attack surface reduced |
|---|---|
| Run as non-root (`--user 1000:1000`) | Limits the impact of a process compromise; the process cannot use root privileges by default. |
| Read-only root filesystem (`--read-only`) | Prevents an attacker or compromised process from changing application binaries, configurations, or persistent filesystem locations. |
| Drop all capabilities (`--cap-drop=ALL`) | Removes privileged kernel operations such as network administration, mounting, and loading kernel modules. |
| Disable new privileges (`no-new-privileges`) | Stops the process from gaining additional privileges through setuid/setgid binaries or similar mechanisms. |
| Use a temporary in-memory `/tmp` (`--tmpfs /tmp`) | Provides only the required writable location without making the container filesystem persistently writable. |
| Scan the image with Trivy | Identifies known vulnerable packages so they can be updated or replaced before deployment. |

## Verification Commands

```bash
kubectl get rolebinding dev-rb -n app -o yaml
docker inspect hardened --format '{{json .HostConfig.CapDrop}}'
```

## Conclusion

The lab applied identity controls (authentication, MFA, and RBAC), network controls (segmentation and default-deny filtering), and workload controls (least privilege and image scanning). Together, these layers reduce unauthorised access, restrict lateral movement, and reduce the impact of a container compromise.
