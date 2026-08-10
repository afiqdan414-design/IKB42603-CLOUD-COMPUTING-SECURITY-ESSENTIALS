# LAB 1: Cloud Account Security, Identity & Access Management
**Identity Governance and Least Privilege — LocalStack IAM & Kubernetes RBAC**

* **Course**: IKB42603 Cloud Computing Security Essentials
* **Institution**: Universiti Kuala Lumpur Malaysian Institute of Information Technology (UniKL MIIT)
* **Instructor**: Nor Adani Kamal
* **Student / Environment Identity**: `afiqdanial` (`CloudAdmin_AFIQ`, `Analyst_AFIQ`)
* **Date**: August 2026

---

## 1. Lab Overview & Learning Outcomes

This lab demonstrates foundational cloud identity management and access control principles across two complementary environments:
1. **LocalStack IAM**: Emulating AWS IAM mechanisms to construct least-privilege users, groups, and policies, as well as performing access key rotation.
2. **Kubernetes RBAC**: Utilizing Kubernetes Role-Based Access Control to enforce strict namespace boundary isolation and fine-grained authorization.

### Core Objectives Achieved:
* Stood up a local cloud lab environment using Docker and LocalStack.
* Replaced root user operations with a dedicated administrative IAM user and group.
* Applied the principle of least privilege using scoped read-only policies.
* Performed access key management and key rotation (deactivation).
* Configured and tested Kubernetes namespaces, Service Accounts, Roles, and RoleBindings.
* Verified enforcement boundaries using `kubectl auth can-i`.

---

## 2. Session A — Cloud Identity with LocalStack

### 2.1 One-Time Environment Setup

1. **Verify Docker Installation**:
   ```bash
   docker --version
   ```
2. **Start LocalStack Container**:
   ```bash
   docker run -d --name localstack -p 4566:4566 localstack/localstack
   ```
3. **Verify LocalStack Health**:
   ```bash
   curl http://localhost:4566/_localstack/health
   ```
4. **Configure Dummy AWS CLI Credentials & Verify Identity**:
   ```bash
   aws configure set aws_access_key_id test
   aws configure set aws_secret_access_key test
   aws configure set region us-east-1

   aws --endpoint-url=http://localhost:4566 sts get-caller-identity
   ```

---

### 2.2 Task 1 — Map the Cloud Identity Landscape

| Concept | AWS Term | Purpose |
| :--- | :--- | :--- |
| **All-powerful owner** | Root User | The foundational account identity created upon account creation. Has permanent, unrestricted administrative access to all cloud resources and billing. Must be secured with MFA and strictly restricted from daily operational use. |
| **Human/app identity** | IAM User | A persistent identity entity within AWS created for a specific human user or application. Assigned distinct credentials (password/access keys) and permissions to interact with AWS APIs. |
| **Permission bundle** | IAM Policy | A JSON document specifying explicit permissions (allowed or denied actions on specific AWS resources under specified conditions). Can be attached to users, groups, or roles. |
| **Collection of users** | IAM Group | A logical collection of IAM users. Attaching policies to groups simplifies permission management and auditing by uniformly granting capabilities across all member identities. |
| **Temporary identity** | IAM Role | An identity with specific permission policies that can be temporarily assumed by human users, applications, or cloud services. Uses short-lived credentials rather than permanent keys. |

---

### 2.3 Task 2 — Create a Least-Privilege Admin (Stop Using Root)

To eliminate root account vulnerability, a dedicated administrative group (`Admins`) and user (`CloudAdmin_AFIQ`) were established.

#### Step 2.1: Create `Admins` Group & Attach `AdministratorAccess` Policy
```bash
aws --endpoint-url=http://localhost:4566 iam create-group --group-name Admins
aws --endpoint-url=http://localhost:4566 iam attach-group-policy \
  --group-name Admins \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```
<img width="477" height="215" alt="A  2 1  Create a group and attach policy" src="https://github.com/user-attachments/assets/ca2ff4f9-3e34-47c0-8a16-b39345f523c7" />


#### Step 2.2: Create Personal Admin User
```bash
aws --endpoint-url=http://localhost:4566 iam create-user --user-name CloudAdmin_AFIQ
```
<img width="522" height="192" alt="A  2 2 Create user" src="https://github.com/user-attachments/assets/12aaf3fc-bc35-4a96-8383-cd68d611d541" />

#### Step 2.3: Assign User to `Admins` Group
```bash
aws --endpoint-url=http://localhost:4566 iam add-user-to-group \
  --group-name Admins \
  --user-name CloudAdmin_AFIQ
```
<img width="466" height="72" alt="A  2 3 Put user in the group" src="https://github.com/user-attachments/assets/b8c47f6a-b2d6-4bc0-81e0-4ed0e8712c1d" />

#### Step 2.4: Verify Group Membership
```bash
aws --endpoint-url=http://localhost:4566 iam get-group --group-name Admins
```
<img width="562" height="347" alt="A  2 4 Verify memberships" src="https://github.com/user-attachments/assets/9db9114c-0398-4a48-b797-01322c510b80" />

> **Security Takeaway**: Attaching authorization policies to IAM Groups rather than individual IAM Users establishes a manageable and auditable security architecture. Modifying permissions at the group level instantly cascades to all member users.

---

### 2.4 Task 3 — Enforce Least Privilege with a Scoped Policy

A restricted read-only user (`Analyst_AFIQ`) was created to demonstrate fine-grained authorization boundaries.

#### Step 3.1: Create Read-Only User
```bash
aws --endpoint-url=http://localhost:4566 iam create-user --user-name Analyst_AFIQ
```
<img width="501" height="189" alt="A  3 1 Create a read-only user" src="https://github.com/user-attachments/assets/32a4cf56-3032-46bf-830b-bf414d3cad52" />

#### Step 3.2: Attach Scoped S3 Read-Only Policy
```bash
aws --endpoint-url=http://localhost:4566 iam attach-user-policy \
  --user-name Analyst_AFIQ \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```
<img width="510" height="68" alt="A  3 2 Attach a scoped read-only policy" src="https://github.com/user-attachments/assets/5dafae22-9751-4955-8811-7c1bcffb63e3" />

#### Step 3.3: List Attached User Policies
```bash
aws --endpoint-url=http://localhost:4566 iam list-attached-user-policies --user-name Analyst_AFIQ
```
<img width="600" height="170" alt="A  3 3 List what the user can do" src="https://github.com/user-attachments/assets/90e73add-ee0f-45b1-9e2e-45273c082cf2" />

#### Task 3 Analysis: Damage Limitation & Blast-Radius Reduction
> **Question**: *If the Analyst account were stolen, why is the damage limited compared to a stolen admin account? Connect your answer to blast-radius reduction.*
>
> **Explanation**: If credentials for `Analyst_AFIQ` are compromised, the adversary is constrained strictly to `AmazonS3ReadOnlyAccess`. The attacker cannot delete S3 buckets or objects, alter configuration settings, provision compute infrastructure, access other services (EC2, IAM, DynamoDB), or tamper with audit logs. In contrast, compromising an administrator or root user grants total control over the cloud environment. By strictly adhering to the Principle of Least Privilege, the **blast radius** (the scope of damage an attacker can inflict) is minimized exclusively to read-only S3 operations, protecting cloud infrastructure integrity.

---

### 2.5 Task 4 — Credential Hygiene & Access Keys

Programmatic API access requires credentials. Managing key lifecycle and rotation prevents persistent access exposure.

#### Step 4.1: Create Access Key for Analyst
```bash
aws --endpoint-url=http://localhost:4566 iam create-access-key --user-name Analyst_AFIQ
```
<img width="562" height="183" alt="A  4 1 Create an access key for the analyst" src="https://github.com/user-attachments/assets/e530273a-6e5e-4668-9f36-2a1c7a80fd36" />

#### Step 4.2: List Access Keys
```bash
aws --endpoint-url=http://localhost:4566 iam list-access-keys --user-name Analyst_AFIQ
```
<img width="472" height="188" alt="A  4 2 List access keys" src="https://github.com/user-attachments/assets/a3b4e457-6263-48dd-acb6-fc1a92c6fd62" />

#### Step 4.3: Deactivate Access Key (Key Rotation)
```bash
aws --endpoint-url=http://localhost:4566 iam update-access-key \
  --user-name Analyst_AFIQ \
  --access-key-id <ACCESS_KEY_ID> \
  --status Inactive

aws --endpoint-url=http://localhost:4566 iam list-access-keys --user-name Analyst_AFIQ
```
<img width="493" height="272" alt="A  4 3 Deactivate the old key" src="https://github.com/user-attachments/assets/a07bc329-830b-4e3a-9d84-6dc6e57c837e" />

> **Caution**: Never create access keys for the root user or embed static credentials into code repositories. Prefer short-lived roles and automated credential rotation.

---

## 3. Session B — Enforced Access Control with Kubernetes RBAC

### 3.1 Setup — Create a Local Kubernetes Cluster

A local Kubernetes cluster `ccse-lab1` was instantiated using `kind` (Kubernetes-in-Docker).

```bash
kind create cluster --name ccse-lab1
kubectl cluster-info --context kind-ccse-lab1
kubectl get nodes
```
<img width="771" height="502" alt="B  create a local kubernetes cluster" src="https://github.com/user-attachments/assets/f9c1844a-b000-45f0-9cd8-e9eeda4e2859" />

---

### 3.2 Task 5 — Separate Environments with Namespaces

Namespaces provide logical environment boundary separation within the cluster.

```bash
kubectl create namespace dev
kubectl create namespace prod
kubectl get namespaces
```
<img width="309" height="347" alt="B  5  Separate environments with namespaces" src="https://github.com/user-attachments/assets/7a4dd9f4-ae42-4c9d-b6f0-226f45d0b6bb" />

---

### 3.3 Task 6 — Define a Role and Bind It (Least Privilege)

#### Step 6.1: Create Developer Service Account
```bash
kubectl create serviceaccount dev-user -n dev
```
<img width="449" height="79" alt="B  6 1 Create a service account to represent a developer" src="https://github.com/user-attachments/assets/60baf1f7-2ff2-44e7-bf33-3ead7a37a83b" />

#### Step 6.2: Create Read-Only Pod Role in `dev`
```bash
kubectl create role pod-reader -n dev \
  --verb=get,list,watch \
  --resource=pods
```
<img width="405" height="76" alt="B  6 2 Create a role that allows only get" src="https://github.com/user-attachments/assets/6dbb142a-efae-46da-977a-050bd323c754" />

#### Step 6.3: Bind Role to Service Account
```bash
kubectl create rolebinding dev-user-binding -n dev \
  --role=pod-reader \
  --serviceaccount=dev:dev-user
```
<img width="508" height="76" alt="B  6 3 Bind the role to the service account" src="https://github.com/user-attachments/assets/fe593dab-df67-489f-98f5-594699ca815f" />

---

### 3.4 Task 7 — Test That Access Control Works

The authorization boundary was validated using `kubectl auth can-i` impersonating `dev-user`.

```bash
SA=system:serviceaccount:dev:dev-user

# 1. Allowed: Read pods in dev namespace
kubectl auth can-i list pods -n dev --as=$SA
# Output: yes

# 2. Blocked: Delete pods in dev namespace
kubectl auth can-i delete pods -n dev --as=$SA
# Output: no

# 3. Blocked: Read pods in prod namespace
kubectl auth can-i list pods -n prod --as=$SA
# Output: no
```
<img width="453" height="231" alt="B  7  Test that access control works" src="https://github.com/user-attachments/assets/ac2ade6c-8916-4f60-a426-fac9b6d6c818" />

#### Task 7 Analysis: Authentication vs. Authorization
> **Question**: *Relate the three can-i results to authentication versus authorization: which step is the service account passing, and which step is blocking the delete and the prod access?*
>
> **Explanation**:
> * **Authentication (Identity Verification)**: The Kubernetes API server successfully authenticates the identity `system:serviceaccount:dev:dev-user` in all three tests. The service account **passes authentication** every time because its token/identity is valid and recognized by the cluster.
> * **Authorization (Permission Evaluation)**:
>   * Test 1 (`list pods -n dev`): **Passes Authorization (`yes`)** because the `pod-reader` role bound via `dev-user-binding` explicitly permits `list` on `pods` within the `dev` namespace.
>   * Test 2 (`delete pods -n dev`): **Blocked by Authorization (`no`)** because the `pod-reader` role does not grant the `delete` verb.
>   * Test 3 (`list pods -n prod`): **Blocked by Authorization (`no`)** because the `pod-reader` Role and `dev-user-binding` RoleBinding are strictly scoped to the `dev` namespace. The `prod` namespace has no matching RoleBinding for this service account, enforcing default-deny cross-namespace isolation.

---

## 4. Deliverables & Assessment Answers

### 4.1 Deliverable Screenshots Summary
1. **`sts get-caller-identity`**: Proves current executing AWS CLI identity against LocalStack.
2. **`get-group Admins`**: Demonstrates `CloudAdmin_AFIQ` group membership.
3. **`list-attached-user-policies`**: Displays `AmazonS3ReadOnlyAccess` policy attached to `Analyst_AFIQ`.
4. **`kubectl auth can-i`**: Shows exact verification results (`yes`, `no`, `no`).

---

### 4.2 Short-Answer Questions

#### Q1. Why is attaching policies to groups better than attaching them directly to users?
> **Answer**: Attaching policies to groups establishes a centralized, scalable access management framework. It prevents "permission drift" and minimizes administrative complexity. When permissions need to be updated or audited, changing the policy attached to a single group immediately applies to all members. Additionally, onboarding or offboarding users simply requires adding or removing them from the group without modifying individual permission manifests.

#### Q2. What is the difference between an IAM User and an IAM Role?
> **Answer**: 
> * An **IAM User** represents a persistent identity associated with long-lived credentials (passwords, static Access Key IDs, and Secret Access Keys) bound to a single principal.
> * An **IAM Role** is an identity with specific permission policies that does not have static long-lived credentials. Instead, trusted entities (users, services like EC2/Lambda, or external federated identities) temporarily **assume** the role to receive short-lived security tokens.

#### Q3. Explain least privilege using the Analyst account, and how it reduces blast radius if compromised.
> **Answer**: The Principle of Least Privilege dictates granting identities only the minimal permissions necessary to perform their assigned functions. Provisioning `Analyst_AFIQ` with `AmazonS3ReadOnlyAccess` ensures the account can only perform read operations on S3. If an attacker steals these credentials, they cannot modify data, delete resources, alter configurations, or provision unauthorized infrastructure. The **blast radius** is constrained strictly to S3 data exposure, preventing catastrophic system compromise.

#### Q4. In Kubernetes, what is the difference between a Role and a RoleBinding?
> **Answer**: 
> * A **Role** is an RBAC API object that defines a set of permission rules (combining API groups, resources like pods/services, and allowed verbs like `get`, `list`, `watch`). It is scoped within a single namespace.
> * A **RoleBinding** bridges a Role to a subject (a User, Group, or ServiceAccount). It grants the permissions defined in the Role to the specified subject within that namespace.

#### Q5. Why did the developer service account fail to access prod, and which security principle does that demonstrate?
> **Answer**: The developer service account (`dev-user`) failed to access `prod` because Kubernetes RBAC operates on an **explicit deny by default** model. The `pod-reader` Role and `dev-user-binding` RoleBinding were created exclusively inside the `dev` namespace. Because no RoleBinding exists in the `prod` namespace for `dev-user`, the request is denied. This demonstrates **Namespace Isolation / Multi-Tenant Defense-in-Depth** and the **Principle of Least Privilege**.

---

### 4.3 Verification Command Output

Command:
```bash
kubectl get rolebinding dev-user-binding -n dev -o yaml
```

YAML Output:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: "2026-07-30T20:22:40Z"
  name: dev-user-binding
  namespace: dev
  resourceVersion: "1930"
  uid: c280b48a-44f8-41ef-82e4-0029e5783054
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: dev
```
<img width="515" height="294" alt="Deliverables 3  Verfication command" src="https://github.com/user-attachments/assets/8af5093c-dfca-4b0f-8a36-e3fc85e63fa8" />

---

### 4.4 Security Best-Practices Checklist

- [x] Root user is not used for daily tasks (a dedicated admin identity exists).
- [x] Permissions are granted via groups/roles, not directly to individual users.
- [x] At least one least-privilege (read-only) identity was created and tested.
- [x] Access keys were listed and a rotation (deactivate) was demonstrated.
- [x] Kubernetes RBAC blocks an unauthorized action (delete / cross-namespace).

---

## 5. Cleanup & Teardown

To release resources after completing the lab:

```bash
# Delete the Kubernetes cluster
kind delete cluster --name ccse-lab1

# Stop and remove LocalStack container
docker stop localstack && docker rm localstack
```
