# Lab 2 Report: Secure Isolation & Multi-Tenancy

**Course:** IKB42603 Cloud Computing Security Essentials  
**Institution:** Universiti Kuala Lumpur Malaysian Institute of Information Technology (UniKL MIIT)  
**Instructor:** Prof. Dr. Shahrulniza Musa  
**Topic:** Compute, Network, and Storage Isolation — Docker & Kubernetes  
**Assessment:** Lab Report (Screenshots + CLI Output + Deliverables & Short-Answer Questions)  

---

## 1. Executive Summary & Lab Learning Outcomes

Multi-tenancy is a foundational architectural pattern in cloud computing where multiple clients (tenants) share common physical or logical computing resources. While multi-tenancy optimizes cost, resource utilization, and operational scalability, it introduces severe security risks if strict tenant isolation controls are not enforced.

This laboratory session demonstrates the implementation and verification of isolation mechanisms across three primary infrastructure dimensions:
1. **Compute Isolation:** Logical workload separation into distinct Kubernetes namespaces and preventing resource exhaustion ("noisy neighbor" syndrome) using `ResourceQuotas`.
2. **Network Isolation:** Transitioning from Kubernetes' default-open, flat networking model to a zero-trust, default-deny network posture enforced via Calico CNI NetworkPolicies.
3. **Storage & Access Isolation:** Restricting tenant access to sensitive configuration data and secrets using Kubernetes Role-Based Access Control (RBAC), alongside examining data remanence in persistent volumes and the necessity of cryptographic erasure.

---

## 2. Lab Setup — Kubernetes Cluster with Policy Enforcement

Standard `kind` (Kubernetes in Docker) clusters utilize a basic CNI plugin that does not enforce Kubernetes `NetworkPolicy` objects. To enable active network policy enforcement, the cluster was provisioned with default CNI disabled (`disableDefaultCNI: true`) and Project Calico CNI was deployed.

### Setup Manifest & Commands
```bash
# Create Kind cluster without default CNI
cat <<EOF | kind create cluster --name ccse-lab2 --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
  podSubnet: 192.168.0.0/16
EOF

# Install Calico CNI for NetworkPolicy enforcement
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
kubectl -n kube-system rollout status daemonset/calico-node --timeout=180s
```

### Verification Screenshot
![Cluster Setup with Calico CNI](Setup-Cluster%20with%20policy%20enforcement.png)

*Figure 1: Successful creation of `ccse-lab2` cluster and installation of Calico CNI Custom Resource Definitions (CRDs), DaemonSets, and RBAC components.*

---

## 3. Session A (Week 3) — Compute Isolation & Default-Open Risk

### Task 1 — Two Tenants on One Cluster

To model a multi-tenant cloud environment, two isolated namespaces (`tenant-a` and `tenant-b`) were created on the shared Kubernetes cluster. An Nginx web application deployment and ClusterIP service were deployed in each namespace.

#### Execution Commands
```bash
# Create tenant namespaces
kubectl create namespace tenant-a
kubectl create namespace tenant-b

# Deploy simple web server for each tenant
kubectl -n tenant-a create deployment web --image=nginx
kubectl -n tenant-b create deployment web --image=nginx

# Expose web deployments on port 80
kubectl -n tenant-a expose deployment web --port=80
kubectl -n tenant-b expose deployment web --port=80

# Verify resources in tenant-a and tenant-b
kubectl get pods,svc -n tenant-a
kubectl get pods,svc -n tenant-b
```

#### Verification Screenshot
![Task 1 - Two Tenants on One Cluster](A.Task1-two%20tenants%20on%20one%20cluster.png)

*Figure 2: Verification of running Nginx pods and ClusterIP services active within both `tenant-a` and `tenant-b` namespaces.*

---

### Task 2 — Observe the Default-Open Risk

By default, Kubernetes implements a flat network model where pods in any namespace can route traffic to pods/services in any other namespace. To demonstrate this default-open vulnerability, an ephemeral curl probe container was launched in `tenant-a` targeting the ClusterIP service of `tenant-b`.

#### Execution Commands
```bash
# Retrieve tenant-b's service ClusterIP address
kubectl get svc web -n tenant-b -o jsonpath='{.spec.clusterIP}'; echo

# Execute cross-tenant probe from tenant-a to tenant-b
kubectl -n tenant-a run probe --rm -it --image=curlimages/curl --restart=Never \
  -- curl -s -m 5 http://<B_IP> -o /dev/null -w 'HTTP %{http_code}\n'
```

#### Output Analysis
- **Tenant B Service IP:** `10.96.53.148`
- **Probe Response:** `HTTP 200`

#### Verification Screenshot
![Task 2 - Observe Default Open Risk](A.Task%202-Observe%20the%20Default-Open%20Risk.png)

*Figure 3: Unrestricted cross-tenant reachability. The probe pod in `tenant-a` successfully accesses `tenant-b` web service, returning an `HTTP 200` status code.*

> [!WARNING]
> **Security Implication:** In a shared multi-tenant cloud environment, namespace separation alone does NOT restrict network traffic. An attacker compromising a container in Tenant A can freely probe, scan, and exploit internal endpoints belonging to Tenant B.

---

### Task 3 — Contain the Noisy Neighbour (Resource Quotas)

Multi-tenancy also requires resource isolation to prevent a single rogue or misconfigured tenant workload from consuming all available node CPU/memory ("noisy neighbor" problem). A `ResourceQuota` object was applied to enforce compute caps on `tenant-a`.

#### Execution Commands
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 512Mi
    pods: "5"
EOF

# Describe resource quota
kubectl describe resourcequota tenant-a-quota -n tenant-a
```

#### Verification Screenshot
![Task 3 - Resource Quota Enforcement](A.Task%203-Contain%20the%20Noisy%20Neighbour.png)

*Figure 4: `ResourceQuota` created in `tenant-a`, establishing hard ceilings of 1 CPU request, 512Mi Memory request, and 5 maximum pods.*

---

## 4. Session B (Week 4) — Network & Storage Isolation

### Task 4 — Default-Deny Network Isolation

To secure tenant boundaries, a default-deny ingress `NetworkPolicy` was applied to `tenant-b`. This enforces the security principle of **Deny by Default, Permit by Exception**.

#### Execution Commands
```bash
# Apply Default-Deny Ingress policy on tenant-b
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: tenant-b
spec:
  podSelector: {}
  policyTypes: [Ingress]
EOF

# Re-run cross-tenant network probe from tenant-a to tenant-b
kubectl -n tenant-a run probe --rm -it --image=curlimages/curl --restart=Never \
  -- curl -s -m 5 http://10.96.53.148 -o /dev/null -w 'HTTP %{http_code}\n'
```

#### Verification Screenshot
![Task 4 - Network Policy Default Deny](B.Task%204-Default-Deny%20Network%20Isolation.png)

*Figure 5: NetworkPolicy enforcement active. Subsequent probe attempts from `tenant-a` are rejected, demonstrating effective network micro-segmentation.*

---

### Task 5 — Storage & Secret Isolation

Storage isolation ensures that confidential configuration data and secrets belonging to one tenant cannot be accessed or queried by another tenant. This was verified using Kubernetes Role-Based Access Control (RBAC).

#### Execution Commands
```bash
# Create sensitive secrets for each tenant
kubectl -n tenant-a create secret generic data --from-literal=value=SECRET_A
kubectl -n tenant-b create secret generic data --from-literal=value=SECRET_B

# Create service account and read role scoped strictly to tenant-a
kubectl -n tenant-a create serviceaccount app-a
kubectl -n tenant-a create role reader --verb=get --resource=secrets
kubectl -n tenant-a create rolebinding rb --role=reader --serviceaccount=tenant-a:app-a

# Authorize check using kubectl auth can-i
SA=system:serviceaccount:tenant-a:app-a
kubectl auth can-i get secrets -n tenant-a --as=$SA  # Expected: yes
kubectl auth can-i get secrets -n tenant-b --as=$SA  # Expected: no
```

#### Verification Screenshot
![Task 5 - Storage and Secret Isolation](B.Task%205-Storage%20%26%20Secret%20Isolation.png)

*Figure 6: RBAC authorization check results. Service account `app-a` in `tenant-a` is granted access to `tenant-a` secrets (`yes`), but strictly forbidden from accessing `tenant-b` secrets (`no`).*

---

### Task 6 — Data Remanence & Secure Deletion

Data remanence occurs when raw file data persists on underlying disk blocks after a file pointer is unlinked/deleted by standard OS commands. This task evaluates file deletion vs. zero-fill sanitization in Docker persistent volumes (`ccse-vol`).

#### Execution Commands
```bash
# Part 1: Standard Deletion (Unlink pointer only)
docker run --rm -v ccse-vol:/data alpine sh -c \
  'echo SENSITIVE-PATIENT-RECORD > /data/phi.txt; sync; rm /data/phi.txt; \
  grep -a SENSITIVE /data/* 2>/dev/null; echo scan-done'

# Part 2: Secure Wipe (Block Overwrite before deletion)
docker run --rm -v ccse-vol:/data alpine sh -c \
  'echo SENSITIVE > /data/phi2.txt; sync; \
  dd if=/dev/zero of=/data/phi2.txt bs=1k count=1 conv=notrunc; rm /data/phi2.txt; \
  echo wiped'
```

#### Verification Screenshot
![Task 6 - Data Remanence and Secure Deletion](B.Task%206-Data%20Remanence%20%26%20Secure%20Deletion.png)

*Figure 7: Execution of standard file deletion vs. zero-block overwrite (`dd if=/dev/zero`) ensuring physical storage blocks are scrubbed clean prior to pointer removal.*

---

## 5. Verification Commands Output

To validate all cluster-wide network policies and compute resource quotas active on the environment, the following verification commands were executed:

```bash
kubectl get networkpolicy -A
kubectl describe resourcequota tenant-a-quota -n tenant-a
```

#### Verification Screenshot
![Verification Commands](Verification%20command.png)

*Figure 8: Cluster verification status confirming `default-deny-ingress` NetworkPolicy active in `tenant-b` and `tenant-a-quota` ResourceQuota active in `tenant-a`.*

---

## 6. Deliverables: Short-Answer Questions

### Q1. Why can containers in different namespaces reach each other by default, and why is that dangerous in multi-tenant cloud?

**Answer:**
* **Root Cause:** In Kubernetes, **namespaces provide logical grouping and scoping for API resources and RBAC policies, but they do NOT enforce network boundaries by default.** Kubernetes adheres to a flat networking specification (IP-per-pod model), requiring all pods across all namespaces to be capable of direct network communication with one another without Network Address Translation (NAT), unless explicit `NetworkPolicy` objects are defined and supported by an active CNI plugin (e.g., Calico).
* **Multi-Tenant Cloud Danger:** In a shared multi-tenant environment hosting workloads from different organizations, customers, or business units, a default-open network posture creates severe vulnerability exposure:
  1. **Lateral Movement:** An attacker who compromises a single container in Tenant A via an application-layer vulnerability (e.g., Remote Code Execution) can perform internal network scanning and reconnaissance across the entire cluster IP range.
  2. **Cross-Tenant Data Exposure & Breach:** The attacker can directly access unauthenticated database instances, internal management microservices, Redis caches, or metrics endpoints belonging to Tenant B located on the same cluster, resulting in cross-tenant data leaks and compliance violations.

---

### Q2. Explain the default-deny principle and how your NetworkPolicy implements it.

**Answer:**
* **The Default-Deny Security Principle:** The principle of "Deny by Default, Permit by Exception" (Zero-Trust Network Architecture) dictates that all incoming and outgoing network traffic flows must be implicitly blocked unless an explicit authorization rule permits them.
* **Implementation in NetworkPolicy:** In Task 4, the following manifest was applied to `tenant-b`:
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
  * **`podSelector: {}`**: Matches all pods residing within the target namespace (`tenant-b`).
  * **`policyTypes: [Ingress]`**: Selects incoming traffic to those pods for policy evaluation.
  * **Empty Ingress Rules (`ingress: []`):** By specifying `policyTypes: [Ingress]` without declaring any `ingress:` allow blocks, the policy isolates all pods in `tenant-b` and drops all incoming traffic originating from outside the namespace (including probes from `tenant-a`).

---

### Q3. How do virtual machines and containers differ in isolation strength? When would you add a VM boundary?

**Answer:**

| Architectural Feature | Containers (Docker / Kubernetes) | Virtual Machines (KVM / ESXi / Hyper-V) |
| :--- | :--- | :--- |
| **Virtualization Level** | OS-level virtualization (Shared Host Linux Kernel) | Hardware-level virtualization (Dedicated Guest Kernel) |
| **Isolation Boundary** | Linux Kernel primitives (`namespaces`, `cgroups`, `seccomp`, `AppArmor`) | Hardware CPU extensions (Intel VT-x / AMD-V) & Hypervisor |
| **Attack Surface** | Shared kernel syscall interface (~300+ syscalls exposed) | Minimal virtualized hardware interface / Hypervisor calls |
| **Impact of Kernel Escape** | Host kernel compromise leads to immediate control over **ALL** containers on the node | Guest kernel compromise is contained within VM boundary |

* **When to Add a VM Boundary:**
  1. **Untrusted Multi-Tenancy / Hard Multi-Tenancy:** When hosting untrusted third-party code, tenant-submitted scripts, or untrusted tenants on shared hardware.
  2. **Strict Regulatory Compliance:** Compliance frameworks (e.g., PCI-DSS, HIPAA, FedRAMP) often require hard isolation boundaries between distinct tenant environments.
  3. **High-Threat Workloads:** When running untrusted binaries or processing high-risk data where container break-out vulnerabilities (e.g., Dirty COW, CVE-2022-0492) could compromise the host. In cloud architectures, Kata Containers, Firecracker microVMs, or dedicated tenant VM worker nodes provide this hypervisor-enforced boundary.

---

### Q4. What is data remanence, and why is cryptographic erasure the preferred cloud solution?

**Answer:**
* **Data Remanence Defined:** Data remanence refers to the residual physical data remaining on storage media (HDDs, SSDs, SAN/NAS arrays) after logical file deletion commands (e.g., `rm`) have been executed. Standard OS deletion only unlinks file index pointers in the filesystem table, leaving raw data blocks intact until overwritten by future disk writes.
* **Why Cryptographic Erasure (Crypto-Shredding) is Preferred in Cloud Environments:**
  1. **Lack of Physical Hardware Control:** Cloud tenants do not own or control physical disk controllers, underlying SAN storage arrays, or NVMe flash chips. Physical zeroization (`dd if=/dev/zero`) or degaussing is impossible or impractical on shared cloud infrastructure.
  2. **Storage Virtualization & Replication:** Cloud storage volumes (e.g., AWS EBS, Azure Disk) automatically replicate block data across multiple physical drives and availability zones for redundancy, making manual block overwriting incomplete and unreliable.
  3. **Mechanism of Cryptographic Erasure:** Data is encrypted at rest using a dedicated, unique Data Encryption Key (DEK) managed via a Key Management Service (KMS). To perform secure deletion, the data key or key hierarchy is destroyed (key shredding). Without the cryptographic key, the encrypted data blocks remaining on physical media become mathematically impossible to decrypt, guaranteeing instantaneous and verifiable destruction across all backups and cloud storage replicas.

---

### Q5. Which of the three isolation dimensions (compute, network, storage) did each task exercise?

**Answer:**

| Lab Task | Isolation Dimension Exercised | Primary Technical Control Used |
| :--- | :--- | :--- |
| **Task 1: Two Tenants on One Cluster** | **Compute Isolation** | Kubernetes Namespaces (`tenant-a`, `tenant-b`) |
| **Task 2: Observe Default-Open Risk** | **Network Isolation (Observation)** | Flat CNI network routing observation (HTTP 200) |
| **Task 3: Contain the Noisy Neighbour** | **Compute & Resource Isolation** | Kubernetes `ResourceQuota` (CPU, Memory, Pod limits) |
| **Task 4: Default-Deny Network Isolation** | **Network Isolation** | Calico CNI `NetworkPolicy` (`default-deny-ingress`) |
| **Task 5: Storage & Secret Isolation** | **Storage & Access Isolation** | Kubernetes RBAC (`ServiceAccount`, `Role`, `RoleBinding`) |
| **Task 6: Data Remanence & Secure Deletion** | **Storage Isolation & Data Security** | Block overwriting (`dd`), Docker Volume scanning, Cryptographic Erasure concepts |

---

## 7. Security Best-Practices Checklist

- [x] **Tenants are separated into distinct namespaces:** Verified in Task 1 (`tenant-a` and `tenant-b`).
- [x] **A default-deny NetworkPolicy blocks cross-tenant traffic:** Verified in Task 4 (HTTP 200 before, timeout/forbidden after policy application).
- [x] **Resource quotas prevent a noisy-neighbour from exhausting shared capacity:** Verified in Task 3 (`tenant-a-quota` enforced).
- [x] **Per-tenant secrets are unreadable by other tenants:** Verified in Task 5 (RBAC authorization checks returning `yes` for own namespace, `no` for external namespace).
- [x] **Secure deletion / cryptographic erasure is understood for data remanence:** Verified in Task 6 (`dd` block overwrite and cryptographic erasure principles).

---

## 8. Cleanup & Environment Teardown

To clean up the local environment post-lab verification, execute:

```bash
# Delete Kind Kubernetes cluster
kind delete cluster --name ccse-lab2

# Remove Docker volume
docker volume rm ccse-vol
```
