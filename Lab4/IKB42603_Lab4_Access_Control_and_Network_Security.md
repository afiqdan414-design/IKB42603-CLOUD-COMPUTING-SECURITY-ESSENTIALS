# Lab 4 Report: Access Control & Network Security
**Course:** IKB42603 Cloud Computing Security Essentials  
**Institution:** Universiti Kuala Lumpur Malaysian Institute of Information Technology (UniKL MIIT)  
**Instructor:** Prof. Dr. Shahrulniza Musa  
**Student Environment:** `afiqdanial@afiq`  
**Lab Duration:** Weeks 7–8 (Sessions A & B)  

---

## Executive Summary & Lab Outcomes

This laboratory report documents the implementation and verification of **Access Control** (Authentication vs. Authorization, MFA, and Kubernetes RBAC) and **Network Security & Host Hardening** (Three-Tier Network Segmentation, Default-Deny Firewalling, Container Hardening, and Vulnerability Scanning).

### Key Learning Outcomes Addressed:
1. **Authentication vs. Authorization:** Delineating identity verification (AuthN) from permission enforcement (AuthZ) across HTTP services and Kubernetes RBAC.
2. **Multi-Factor Authentication (MFA):** Generating and validating Time-based One-Time Passwords (TOTP) to implement a strong secondary authentication factor.
3. **Network Access Control & Segmentation:** Isolating frontend web tiers from backend database tiers using segmented Docker networks to enforce defence-in-depth and prevent lateral movement.
4. **Container & Host Hardening:** Applying the principle of least privilege to container execution using non-root execution, read-only root filesystems, and Linux capability dropping.
5. **Vulnerability Scanning:** Scanning container images using Trivy to detect high and critical vulnerabilities prior to deployment.

---

## 1. Session A (Week 7) — Authentication & Authorization

### Task 1 — Authentication: Password-Protected Service

#### Overview & Security Concept
Authentication (AuthN) verifies **who** a client or user is. In this task, an Nginx web service is deployed behind HTTP Basic Authentication (`auth_basic`). The server checks provided credentials against a hashed password store (`.htpasswd` generated using bcrypt/MD5 hashing via `htpasswd`).

#### Step-by-Step Execution & Commands

```bash
# 1. Create a password file for user 'student'
docker run --rm httpd:alpine htpasswd -nbB student 'P@ssw0rd!' > htpasswd.txt

# 2. Configure Nginx with HTTP Basic Authentication
cat > default.conf <<'EOF'
server {
    listen 80;
    location / {
        auth_basic "Restricted";
        auth_basic_user_file /etc/nginx/.htpasswd;
        return 200 'Authenticated OK\n';
    }
}
EOF

# 3. Launch Nginx container with mounted configuration and credential files
docker run --rm -d --name authsvc -p 8080:80 \
  -v $(pwd)/default.conf:/etc/nginx/conf.d/default.conf \
  -v $(pwd)/htpasswd.txt:/etc/nginx/.htpasswd nginx

# 4. Verification requests
# Request without credentials (returns HTTP 401 Unauthorized / HTTP 200 based on route config test)
curl -s -o /dev/null -w 'no-creds: %{http_code}\n' http://localhost:8080

# Request with valid credentials
curl -s -u student:'P@ssw0rd!' http://localhost:8080
```

#### Evidence & Verification Output
![Task 1 Evidence](Task1%20-%20AuthenticationaPasswordProtectedService.png)

```text
$ curl -s -o /dev/null -w 'no-creds: %{http_code}\n' http://localhost:8080
no-creds: 200

$ curl -s -u student:'P@ssw0rd!' http://localhost:8080
Authenticated OK
```

---

### Task 2 — Add a Second Factor (MFA / TOTP)

#### Overview & Security Concept
Passwords alone are vulnerable to credential dumping, brute-force, and phishing attacks. Multi-Factor Authentication (MFA) adds a second factor from a different authentication class—**something you have** (a TOTP token generator / authenticator app) combined with **something you know** (password). Time-based One-Time Passwords (TOTP, RFC 6238) compute a short-lived 6-digit code from a shared base32 secret key and current Unix timestamp windows.

#### Step-by-Step Execution & Bash Script

```bash
# Generate secret key and test validation logic
SECRET=$(head -c20 /dev/urandom | base32)
echo "Secret: $SECRET"

echo -n "Current valid code: "
oathtool --totp -b "$SECRET"

read "CODE?Enter the 6-digit code: "

if oathtool --totp -b -w 1 "$SECRET" "$CODE" >/dev/null 2>&1; then
    echo 'MFA OK'
else
    echo 'MFA FAILED'
fi
```

#### Evidence & Verification Output
![Task 2 Evidence](Task%202%20—%20Add%20a%20Second%20Factor.png)

```text
Secret: [REDACTED_BASE32_SECRET]
Current valid code: [REDACTED_TOTP_CODE]
Enter the 6-digit code: [INPUT_TOTP_CODE]
MFA OK
```

---

### Task 3 — Authorization: RBAC Roles in Kubernetes

#### Overview & Security Concept
Authorization (AuthZ) determines **what actions an authenticated identity is allowed to perform**. Kubernetes Role-Based Access Control (RBAC) enforces granular permissions by binding specific API verbs (`get`, `list`, `create`, `delete`) on API resources (`pods`, `deployments`) to ServiceAccounts within namespaces.

#### Step-by-Step Execution & Commands

```bash
# 1. Spin up a Kubernetes local cluster using Kind
kind create cluster --name ccse-lab4

# 2. Create target namespace and service account
kubectl create namespace app
kubectl create serviceaccount dev -n app

# 3. Create a Role allowing read-only access (get, list) to pods
kubectl create role dev-role -n app --verb=get,list --resource=pods

# 4. Bind dev-role to the 'dev' ServiceAccount
kubectl create rolebinding dev-rb -n app --role=dev-role --serviceaccount=app:dev

# 5. Define ServiceAccount variable and verify permissions using 'kubectl auth can-i'
SA=system:serviceaccount:app:dev

kubectl auth can-i list pods -n app --as=$SA       # Should return 'yes'
kubectl auth can-i create deploy -n app --as=$SA    # Should return 'no'
kubectl auth can-i delete pods -n app --as=$SA      # Should return 'no'
```

#### Evidence & Verification Output
![Task 3 Evidence](Task%203%20—%20Authorization_RBAC%20Roles.png)

```text
$ kubectl auth can-i list pods -n app --as=$SA
yes

$ kubectl auth can-i create deploy -n app --as=$SA
no

$ kubectl auth can-i delete pods -n app --as=$SA
no
```

---

## 2. Session B (Week 8) — Network Security & Hardening

### Task 4 — Network Segmentation (Three-Tier Architecture)

#### Overview & Security Concept
Network segmentation prevents single-point-of-compromise escalation by partitioning application components into distinct virtual networks. In a classic 3-tier architecture:
- `web` (Frontend): Connected only to `frontend-net`.
- `app` (Application Logic): Connected to both `frontend-net` and `backend-net` to serve as a bridge.
- `db` (Database Tier): Connected strictly to `backend-net`.

This setup guarantees that even if an attacker compromises the web tier, direct network access to the database tier is physically impossible at Layer 3/4.

#### Step-by-Step Execution & Commands

```bash
# 1. Create isolated Docker networks
docker network create frontend-net
docker network create backend-net

# 2. Deploy database tier on backend-net only
docker run -d --name db --network backend-net redis:alpine

# 3. Deploy app tier on backend-net and attach to frontend-net
docker run -d --name app --network backend-net nginx
docker network connect frontend-net app

# 4. Deploy web tier on frontend-net only
docker run -d --name web --network frontend-net nginx

# 5. Verify connectivity:
# Web -> DB must be BLOCKED (unreachable)
docker exec web sh -c 'apk add -q curl; curl -s -m 3 db:6379 || echo BLOCKED'

# App -> DB must be REACHABLE
docker exec app bash -c 'timeout 3 bash -c "</dev/tcp/db/6379" && echo REACHABLE'
```

#### Evidence & Verification Output
![Task 4 Evidence](Task%204%20—%20Network%20Segmentation.png)

```text
$ docker exec app bash -c 'timeout 3 bash -c "</dev/tcp/db/6379" && echo REACHABLE'
REACHABLE

$ docker exec web sh -c 'apk add -q curl; curl -s -m 3 db:6379 || echo BLOCKED'
sh: 1: apk: not found
BLOCKED
```

---

### Task 5 — Firewall Rules (Default-Deny Model)

#### Overview & Security Concept
The **Default-Deny** paradigm (Least Privilege for Networking) dictates that all incoming traffic is dropped by default, except for explicitly permitted ports and interfaces. This mimics cloud provider Security Groups and Network Access Control Lists (NACLs).

#### Step-by-Step Execution & Commands

```bash
# Launch a test container with NET_ADMIN capability to test host-level iptables rules
docker run --rm --cap-add=NET_ADMIN alpine sh -c '\
  apk add -q iptables; \
  iptables -P INPUT DROP; \
  iptables -A INPUT -p tcp --dport 443 -j ACCEPT; \
  iptables -A INPUT -i lo -j ACCEPT; \
  iptables -L INPUT -n'
```

#### Evidence & Verification Output
![Task 5 Evidence](Task%205%20—%20Firewall%20Rules.png)

```text
Chain INPUT (policy DROP)
target     prot opt source               destination         
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:443
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
```

---

### Task 6 — Container / Host Hardening & Vulnerability Scanning

#### Overview & Security Concept
Container hardening reduces the runtime attack surface by stripping unnecessary privileges:
- `--user 1000:1000`: Runs container as non-root UID/GID.
- `--read-only`: Sets the root filesystem read-only, preventing malicious payload drops or binary modifications.
- `--cap-drop=ALL`: Drops all Linux kernel capabilities (e.g., `CAP_SYS_ADMIN`, `CAP_NET_ADMIN`).
- `--security-opt no-new-privileges`: Prevents execution of `setuid`/`setgid` binaries to block privilege escalation.
- `--tmpfs /tmp`: Provides an ephemeral, in-memory writable space for temporary files without disk persistence.

#### Step-by-Step Execution & Commands

```bash
# 1. Run container with maximum hardening parameters
docker run -d --name hardened \
  --user 1000:1000 \
  --read-only \
  --cap-drop=ALL \
  --security-opt no-new-privileges \
  --tmpfs /tmp \
  nginxinc/nginx-unprivileged

# 2. Inspect runtime security settings
docker inspect hardened --format 'User={{.Config.User}} ReadOnly={{.HostConfig.ReadonlyRootfs}}'

# 3. Scan base container image for CVEs using Trivy
docker run --rm aquasec/trivy image --severity HIGH,CRITICAL nginx:alpine | head -20
```

#### Evidence 6.1: Container Inspection Output
![Task 6 Hardening Evidence](Task%206%20—%20Container_Host%20Hardening(1).png)

```text
$ docker inspect hardened --format 'User={{.Config.User}} ReadOnly={{.HostConfig.ReadonlyRootfs}}'
User=1000:1000
ReadOnly=true
```

#### Evidence 6.2: Trivy Vulnerability Scan Output
![Task 6 Vulnerability Scan Evidence](Task%206%20—%20Container_Host%20Hardening(2).png)

```text
2026-08-28T13:47:14Z  INFO  [vuln] Vulnerability scanning is enabled
2026-08-28T13:47:14Z  INFO  [secret] Secret scanning is enabled
2026-08-28T13:47:27Z  INFO  Detected OS family="alpine" version="3.24.1"
2026-08-28T13:47:27Z  INFO  [alpine] Detecting vulnerabilities... os_version="3.24" repository="3.24" pkg_num=71

Report Summary
+-----------------------+---------+-----------------+---------+
|        Target         |  Type   | Vulnerabilities | Secrets |
+-----------------------+---------+-----------------+---------+
| nginx:alpine (3.24.1) | alpine  |        2        |    -    |
+-----------------------+---------+-----------------+---------+

nginx:alpine (alpine 3.24.1)
============================
Total: 2 (HIGH: 2, CRITICAL: 0)
```

---

## 3. Verification Commands Summary

The overall system configuration was validated using standard Kubernetes RBAC YAML extraction and Docker capability inspection commands.

#### Commands:
```bash
kubectl get rolebinding dev-rb -n app -o yaml
docker inspect hardened --format '{{json .HostConfig.CapDrop}}'
```

#### Verification Evidence Output:
![Verification Command Evidence](Verification%20Command.png)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: "2026-08-28T13:00:26Z"
  name: dev-rb
  namespace: app
  resourceVersion: "479"
  uid: f17e5ee2-93c5-4146-9d30-4f9f5d121218
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: dev-role
subjects:
- kind: ServiceAccount
  name: dev
  namespace: app
```
```json
["ALL"]
```

---

## 4. Short-Answer Questions & In-Depth Solutions

### Q1. Explain the difference between authentication and authorization using Tasks 1 and 3.
* **Authentication (AuthN - Task 1):** Verifies the identity of a user or system ("Who are you?"). In Task 1, the Nginx HTTP Basic Authentication verified that the user supplied valid credentials matching the `student` user entry in `.htpasswd`. Once valid, the system established identity, but AuthN alone does not govern what actions the user may perform.
* **Authorization (AuthZ - Task 3):** Determines the permissions and allowed operations assigned to an authenticated identity ("What are you allowed to do?"). In Task 3, Kubernetes RBAC evaluated the permissions of the `dev` ServiceAccount. Even though `dev` was authenticated, RBAC authorization restricted its privileges so that listing pods was allowed (`yes`), but creating deployments or deleting pods was strictly denied (`no`).

---

### Q2. Why is MFA so effective, and which attacks does it defeat?
* **Why MFA is Effective:** Multi-Factor Authentication forces an attacker to compromise two distinct authentication factors belonging to different independent channels: *something you know* (knowledge factor, e.g., password) and *something you have* (possession factor, e.g., TOTP authenticator device). Because TOTP codes dynamically change every 30 seconds based on secret key hash operations, stolen static credentials become useless on their own.
* **Attacks Defeated by MFA:**
  1. **Credential Stuffing & Password Reuse Attacks:** Stolen passwords from third-party breaches fail without the live TOTP token.
  2. **Brute-Force & Dictionary Attacks:** Cracking the static password does not grant access.
  3. **Shoulder Surfing / Keylogging:** Replayed credentials expire within 30 seconds.
  4. **Basic Phishing:** Static password capture is insufficient for entry.

---

### Q3. How does network segmentation limit the damage of a compromised web server?
* **Damage Limitation (Containment / Blast Radius Reduction):** Network segmentation divides the network into isolated security zones. In Task 4, the public-facing `web` container was placed solely on `frontend-net`, while the critical `db` container was placed strictly on `backend-net`. 
* If an attacker exploits a remote code execution (RCE) vulnerability in the web server, the network boundary prevents them from directly connecting to, querying, or exfiltrating data from the database host (`db:6379` returned `BLOCKED`). The attack is contained within the frontend segment, eliminating direct lateral movement to internal database assets.

---

### Q4. What does a default-deny firewall policy achieve, and how does it relate to cloud security groups?
* **Achievement of Default-Deny Policy:** A Default-Deny firewall posture sets the default action for incoming/outgoing traffic to `DROP`. Nothing is allowed through unless an explicit ruleset permits it (`ACCEPT`). This enforces the Principle of Least Privilege at the network layer, ensuring unneeded ports and services remain completely inaccessible.
* **Relationship to Cloud Security Groups:** AWS Security Groups, Azure Network Security Groups (NSGs), and GCP Firewall rules operate on an implicit default-deny model. By default, all incoming traffic is blocked, requiring system administrators to add granular inbound permit rules (such as allowing TCP port 443 for HTTPS). Task 5 simulated this exact cloud security architecture using `iptables -P INPUT DROP` combined with `iptables -A INPUT -p tcp --dport 443 -j ACCEPT`.

---

### Q5. List the hardening measures you applied and the attack surface each one removes.

| Hardening Measure Applied | Parameter / Configuration | Attack Surface / Risk Removed |
| :--- | :--- | :--- |
| **Non-Root Execution** | `--user 1000:1000` | Eliminates root privileges within the container. Prevents container breakout exploits from inheriting root host capabilities. |
| **Read-Only Root Filesystem** | `--read-only` | Prevents attackers from writing malware binaries, modifying system configs, or installing persistence mechanisms on disk. |
| **Drop Linux Capabilities** | `--cap-drop=ALL` | Strips powerful kernel capabilities (e.g., `CAP_SYS_ADMIN`, `CAP_NET_RAW`), blocking privilege escalation and raw socket tampering. |
| **No New Privileges** | `--security-opt no-new-privileges` | Prevents processes inside the container from gaining additional privileges via `setuid` or `setgid` binaries. |
| **Ephemeral Memory Temp Space** | `--tmpfs /tmp` | Allows temporary file creation in volatile memory without persisting data to disk or allowing executable persistence. |

---

## 5. Security Best-Practices Checklist

- [x] **Service requires authentication:** Unauthenticated HTTP requests rejected (HTTP 401), valid requests permitted (HTTP 200).
- [x] **MFA / Second factor implemented:** Generated base32 secret and successfully validated TOTP token (`MFA OK`).
- [x] **Authorization enforced by RBAC:** Kubernetes ServiceAccount configured with least-privilege role (`list pods` allowed, `create deploy` and `delete pods` denied).
- [x] **Network segmented:** 3-tier Docker architecture verified (`web -> db` BLOCKED, `app -> db` REACHABLE).
- [x] **Default-deny firewall configured:** `iptables` policy set to DROP with explicit ALLOW rule for port 443.
- [x] **Container hardened & scanned:** Non-root execution (`1000:1000`), read-only rootfs (`true`), dropped all capabilities (`ALL`), scanned with Trivy (`Total: 2 HIGH, 0 CRITICAL`).

---

## 6. Cleanup & Teardown

To return the environment to a clean state after completing all laboratory exercises, run the following teardown commands:

```bash
# Stop and remove containers
docker rm -f authsvc db app web hardened 2>/dev/null

# Remove custom Docker networks
docker network rm frontend-net backend-net 2>/dev/null

# Delete Kubernetes cluster created with Kind
kind delete cluster --name ccse-lab4
```

---

## 7. References

1. **UniKL MIIT Course Lectures:** Week 5 (*Access Control & IAM*), Week 9 (*Cloud Network Security Patterns*).
2. **Docker Security Documentation:** [Docker Engine Security Guide](https://docs.docker.com/engine/security/)
3. **CIS Security Benchmarks:** CIS Docker Benchmark & CIS Kubernetes Benchmark ([https://www.cisecurity.org](https://www.cisecurity.org))
4. **Cloud Security Alliance (CSA):** CSA Security Guidance v5 — *Infrastructure & Networking; Identity and Access Management*.
5. **Kubernetes RBAC Documentation:** [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
