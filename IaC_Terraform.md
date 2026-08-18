# LAB: Infrastructure as Code (IaC) Challenge — Terraform & LocalStack IAM
**Automating Least-Privilege IAM Infrastructure with Terraform and LocalStack**

* **Course**: IKB42603 Cloud Computing Security Essentials
* **Institution**: Universiti Kuala Lumpur Malaysian Institute of Information Technology (UniKL MIIT)
* **Instructor**: Prof. Dr. Shahrulniza Musa
* **Student / Environment Identity**: `CloudAdmin_Ainin` (`Admins-1`)
* **Date**: August 2026

---

## 1. Executive Summary & Lab Overview

This lab transitions manual cloud administration (executed via individual AWS CLI commands in Lab 1) into a **repeatable, declarative Infrastructure as Code (IaC)** deployment model using **HashiCorp Terraform** targeting a local **LocalStack** emulator.

### Core Objectives:
1. **Automated IAM Provisioning**: Recreate the least-privilege administrative access model (IAM Group, Managed Policy Attachment, IAM User, and Group Membership) purely through declarative HCL code (`main.tf`).
2. **LocalStack Provider Binding**: Configure Terraform's AWS provider to override service endpoints, directing API calls to LocalStack (`http://localhost:4566`) without invoking live AWS cloud infrastructure.
3. **Structured IaC Lifecycle**: Practice and document the strict Terraform execution sequence (`init` $\rightarrow$ `fmt` $\rightarrow$ `validate` $\rightarrow$ `plan` $\rightarrow$ `apply` $\rightarrow$ `destroy`).
4. **Independent Verification**: Validate the resulting cloud state out-of-band using AWS CLI commands to confirm that declared resources exist with exact permissions.

---

## 2. Theoretical Foundations: Manual CLI vs. Infrastructure as Code

| Aspect | Manual AWS CLI Operations | Infrastructure as Code (Terraform) |
| :--- | :--- | :--- |
| **Execution Model** | Imperative (step-by-step commands) | Declarative (describe the target desired state) |
| **State Tracking** | None (operator must manually check existing state) | Maintained automatically via state file (`terraform.tfstate`) |
| **Idempotency** | Low (running commands twice causes duplicate errors) | Native (re-applying identical code produces zero changes) |
| **Consistency & Reusability** | Hard to replicate accurately across environments | Single code manifest deployed reproducibly across dev/test/prod |
| **Teardown / Destruction** | Requires manual deletion of every resource | Automated cleanup via single command (`terraform destroy`) |

### Key IaC Principles Implemented:
* **Declarative**: Focuses on *what* infrastructure should exist rather than *how* to construct it.
* **Idempotent**: Re-executing `terraform apply` evaluates desired state against actual state and makes no unnecessary modifications if no drift exists.
* **State-Aware**: Terraform tracks managed resource mappings in `terraform.tfstate`, enabling precise diff generation during `terraform plan`.
* **Versioned**: Infrastructure definitions reside in `main.tf`, allowing full Git-based versioning, peer reviews, and audit trails.

---

## 3. Architecture & Terraform Configuration (`main.tf`)

### 3.1 Lab Architecture Flow

```
+------------------+         +--------------------+         +-----------------------+         +---------------------+
|                  |         |                    |         |                       |         |                     |
|     main.tf      | ------> |    AWS Provider    | ------> |      LocalStack       | ------> |    IAM Resources    |
| (HCL Definition) |         |  (HashiCorp/AWS)   |         | (http://localhost:4566)|         | (Group, User, Policy)|
|                  |         |                    |         |                       |         |                     |
+------------------+         +--------------------+         +-----------------------+         +---------------------+
```

---

### 3.2 Complete `main.tf` Code Manifest

The following Terraform configuration defines the complete IAM infrastructure required for the lab:

```hcl
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
    }
  }
}

# ----------------------------------------
# Point Terraform to LocalStack
# ----------------------------------------

provider "aws" {
  access_key = "test"
  secret_key = "test"
  region     = "us-east-1"

  # We are using LocalStack (Want to manage AWS-style infrastructure)

  endpoints {
    iam = "http://localhost:4566"
    sts = "http://localhost:4566"
  }

  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true
}

# ----------------------------------------
# 1. Create IAM Group
# ----------------------------------------

resource "aws_iam_group" "admins" {
  name = "Admins-1"
}

# ----------------------------------------
# 2. Attach AdministratorAccess Policy to the GROUP
# ----------------------------------------

resource "aws_iam_group_policy_attachment" "admin_policy" {
  group      = aws_iam_group.admins.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"
}

# ----------------------------------------
# 3. Create IAM User
# ----------------------------------------

resource "aws_iam_user" "cloud_admin" {
  name = "CloudAdmin_Ainin"
}

# ----------------------------------------
# 4. Add User to Admins Group
# ----------------------------------------

resource "aws_iam_user_group_membership" "cloud_admin_membership" {
  user = aws_iam_user.cloud_admin.name

  groups = [
    aws_iam_group.admins.name
  ]
}
```

### Component Breakdown:
1. **Provider Block (`provider "aws"`)**: Configures credentials (`test`/`test`) and redirects IAM and STS API endpoints to `http://localhost:4566`.
2. **`aws_iam_group.admins`**: Creates the administrative group named `Admins-1`.
3. **`aws_iam_group_policy_attachment.admin_policy`**: Binds the AWS-managed policy `AdministratorAccess` to the `Admins-1` group.
4. **`aws_iam_user.cloud_admin`**: Provisions the personal administrative user `CloudAdmin_Ainin`.
5. **`aws_iam_user_group_membership.cloud_admin_membership`**: Associates user `CloudAdmin_Ainin` with group `Admins-1`, granting the user administrative capabilities indirectly through group inheritance.

---

## 4. Step-by-Step Execution & Command Evidence

### Step 1: Verify Installed Terraform Version

Verify that Terraform is installed correctly on the workstation prior to running commands:

```bash
terraform -v
```

![Terraform Version](Terraform%20version.png)

* **Evidence Analysis**: Confirms Terraform executable version `v1.6.3-dev` running on `linux_amd64` with AWS provider plugin version `v6.58.0`.

---

### Step 2: Initialize Provider (`terraform init`)

Download and initialize the AWS provider plugins required by `main.tf`:

```bash
terraform init
```

![Terraform Init](Terraform%20Init.png)

* **Evidence Analysis**: Prepares the workspace directory, creating `.terraform/` and locking provider dependencies.

---

### Step 3: Format Configuration Code (`terraform fmt`)

Enforce canonical HashiCorp HCL formatting and code alignment standards across configuration files:

```bash
terraform fmt
```

![Terraform Format](Terraform%20fmt.png)

* **Evidence Analysis**: Automatically formats `main.tf` to ensure consistent indentation, line breaks, and argument alignment.

---

### Step 4: Validate Configuration Syntax (`terraform validate`)

Check configuration files for syntax errors, missing arguments, or type mismatches before querying the cloud endpoint:

```bash
terraform validate
```

![Terraform Validate](Terraform%20validate.png)

* **Evidence Analysis**: Terminal displays `Success! The configuration is valid.` confirming HCL syntax and internal reference validity.

---

### Step 5: Preview Execution Plan (`terraform plan`)

Generate and inspect an execution plan to preview resource additions, modifications, or deletions before applying changes:

```bash
terraform plan
```

![Terraform Plan](Terraform%20plan.png)

* **Evidence Analysis**:
  * `aws_iam_group.admins` will be created (`name = "Admins-1"`).
  * `aws_iam_group_policy_attachment.admin_policy` will be created (`policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"`).
  * `aws_iam_user.cloud_admin` will be created (`name = "CloudAdmin_Ainin"`).
  * `aws_iam_user_group_membership.cloud_admin_membership` will be created (`user = "CloudAdmin_Ainin"`, `groups = ["Admins-1"]`).
  * **Plan Result**: `Plan: 4 to add, 0 to change, 0 to destroy.`

---

### Step 6: Apply Infrastructure Configuration (`terraform apply`)

Execute the actions proposed in the execution plan to provision resources in LocalStack:

```bash
terraform apply
```

![Terraform Apply](Terraform%20apply.png)

* **Evidence Analysis**:
  * Interactive prompt approved with `yes`.
  * Terraform creates resources sequentially while tracking dependencies.
  * **Execution Result**: `Apply complete! Resources: 4 added, 0 changed, 0 destroyed.`

---

## 5. Out-of-Band AWS CLI Verification

> **Security & Engineering Rule**: Never trust `Apply complete!` blindly. Always verify resource deployment out-of-band using independent query tools.

Set up the LocalStack endpoint helper environment variable:

```bash
EP='--endpoint-url=http://localhost:4566'
```

---

### Verification 1: Confirm IAM Group & Member Association

Query LocalStack IAM to confirm that group `Admins-1` exists and contains member `CloudAdmin_Ainin`:

```bash
aws $EP iam get-group --group-name Admins-1
```

![Terraform Verify Group](Terraform%20verify%20sources-group.png)

* **Verification Evidence Output**:
  ```json
  {
      "Users": [
          {
              "Path": "/",
              "UserName": "CloudAdmin_Ainin",
              "Arn": "arn:aws:iam::000000000000:user/CloudAdmin_Ainin"
          }
      ],
      "Group": {
          "Path": "/",
          "GroupName": "Admins-1",
          "Arn": "arn:aws:iam::000000000000:group/Admins-1"
      }
  }
  ```

---

### Verification 2: Confirm Attached Managed Policy

Query LocalStack IAM to verify that the `AdministratorAccess` policy is attached to group `Admins-1`:

```bash
aws $EP iam list-attached-group-policies --group-name Admins-1
```

![Terraform Verify Policies](Terraform%20verify%20sources-policies.png)

* **Verification Evidence Output**:
  ```json
  {
      "AttachedPolicies": [
          {
              "PolicyName": "AdministratorAccess",
              "PolicyArn": "arn:aws:iam::aws:policy/AdministratorAccess"
          }
      ]
  }
  ```

---

## 6. Infrastructure Teardown (`terraform destroy`)

Demonstrate clean infrastructure lifecycle management by destroying all Terraform-managed resources:

```bash
terraform destroy
```

![Terraform Destroy](Terraform%20destroy.png)

* **Evidence Analysis**:
  * Terraform reads state file `terraform.tfstate` and determines reverse dependency teardown order.
  * Membership and policy attachments are detached first, followed by group and user deletion.
  * **Result**: `Destroy complete! Resources: 4 destroyed.`

---

## 7. Assessment Reflection & Discussion

### Lab Question:
> **Why is Terraform considered Infrastructure as Code, and what advantage does it provide compared with manual AWS CLI commands?**

### Comprehensive Answer & Technical Explanation:

1. **Declarative State vs. Imperative Scripting**:
   * **Imperative (Manual CLI)** requires step-by-step shell commands (`create-group`, `create-user`, `add-user-to-group`, `attach-group-policy`). If any step fails midway or is run twice, scripts break or create duplicate resources.
   * **Declarative (Terraform IaC)** specifies the end-state model in `main.tf`. Terraform calculates the delta between current state and desired state, applying only necessary API actions.

2. **State Management & Drift Detection**:
   * Manual CLI commands do not remember what was built previously.
   * Terraform retains absolute awareness of managed resources via `terraform.tfstate`. Running `terraform plan` alerts administrators immediately if out-of-band drift or manual modifications occurred.

3. **Auditing, Version Control & Security Governance**:
   * Infrastructure code stored in `.tf` files can be committed to Git repositories. This enables code reviews, security scanning (e.g., checking for unencrypted storage or open security groups), pull requests, and historical change audits prior to deployment.

4. **Lifecycle Control & Blast Radius Management**:
   * IaC enables predictable provisioning and instant cleanup (`terraform destroy`). In multi-tenant enterprise environments, entire environments (dev, staging, sandbox) can be spun up or torn down safely without leaving orphan resources behind.

---

## 8. Security Best-Practices Checklist

- [x] Provider credentials set to dummy values (`test`/`test`) for LocalStack security isolation.
- [x] Endpoints strictly pointed to `http://localhost:4566` to block accidental live AWS modification.
- [x] Administrative access granted strictly through IAM Group inheritance (`Admins-1`), avoiding direct user policy attachments.
- [x] Code standard enforced via `terraform fmt` and verified via `terraform validate`.
- [x] Execution plan (`terraform plan`) reviewed prior to resource creation.
- [x] Independent AWS CLI out-of-band verification performed.
- [x] Complete resource destruction (`terraform destroy`) tested successfully.
