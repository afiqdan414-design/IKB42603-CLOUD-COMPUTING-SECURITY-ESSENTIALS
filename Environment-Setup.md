# Lab 0: Environment Setup & Verification Report
**Course:** IKB42603 Cloud Computing Security Essentials  
**Institution:** Universiti Kuala Lumpur Malaysian Institute of Information Technology (UniKL MIIT)  
**Instructor:** Prof. Dr. Shahrulniza Musa  
**Document:** `Environment-Setup.md`  

---

## 1. Overview & Objectives

This report documents the step-by-step installation, setup, and verification of the local development and security environment required for **IKB42603 Cloud Computing Security Essentials**.

The lab environment operates completely on local hardware without requiring cloud provider accounts, credit cards, or continuous internet access after initial downloads.

> [!IMPORTANT]
> **Security Tip for Windows Users:**  
> After installing Docker, execute all lab commands inside **Git Bash** or **WSL (Ubuntu)**. The lab exercises utilize Linux shell features (e.g., heredocs, `sha256sum`, single-quoting) that are not natively supported in Windows Command Prompt or PowerShell.

---

## 2. Tool Overview

| Tool | Purpose | Used In |
| :--- | :--- | :--- |
| **Docker** | Runs containerized applications and the LocalStack cloud simulator | All Labs |
| **AWS CLI v2** | Interacts with LocalStack using simulated AWS API commands | Labs 1, 3, 5 |
| **kind** | Runs local Kubernetes clusters inside Docker containers | Labs 1, 2, 4 |
| **kubectl** | Command-line tool to inspect and manage Kubernetes clusters | Labs 1, 2, 4 |
| **OpenSSL** | Cryptographic tool for encryption, key generation, and certificates | Lab 3 |
| **oathtool** | Command-line tool for generating MFA / TOTP verification codes | Lab 4 |
| **Trivy** | Vulnerability scanner for container images (executed via Docker) | Lab 4 |

---

## 3. Step 1: Install & Verify Docker

### 3.1 Installation Instructions

* **Windows 10 / 11:** Download and install **Docker Desktop** from [docker.com](https://docker.com). Ensure the **WSL 2 backend** option is selected during setup, then reboot the machine.
* **macOS:** Download and install Docker Desktop for Mac (select Apple Silicon or Intel depending on your chip architecture).
* **Linux (Ubuntu):** Run the automated installation script:
  ```bash
  curl -fsSL https://get.docker.com | sh
  sudo usermod -aG docker $USER
  ```
  *(Log out and back in for group membership changes to take effect).*

> [!WARNING]
> **Virtualization Requirement:** Docker requires hardware virtualization enabled in system BIOS/UEFI (VT-x / AMD-V / SVM). On Windows, enable both **WSL 2** and **Virtual Machine Platform** features.

### 3.2 Verification

Execute the following commands to confirm Docker is running properly:
```bash
docker --version
docker run --rm hello-world
```

#### Execution Output:
![Docker Verification](<img width="617" height="77" alt="1  docker" src="https://github.com/user-attachments/assets/361f544b-65a4-4501-a961-71ec481760fe" />
)

*Verified Output:* `Docker version 28.5.2+dfsg4, build 9cc6dea35e9a963f281434761c656fba4ac43aed`

---

## 4. Step 2: Install & Verify AWS CLI v2

### 4.1 Installation Instructions

* **Windows:** Download and run the official AWS CLI v2 `.msi` installer.
* **macOS:** Install via Homebrew: `brew install awscli` or download the `.pkg` installer.
* **Linux (Ubuntu):**
  ```bash
  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
  unzip awscliv2.zip && sudo ./aws/install
  ```

> [!NOTE]
> A real AWS account is **not required**. AWS CLI commands will be pointed to the LocalStack emulator using `--endpoint-url=http://localhost:4566`.

### 4.2 Verification

Verify AWS CLI v2 installation:
```bash
aws --version
```

#### Execution Output:
![AWS CLI Verification](2.%20AWS%20CLI.png)

*Verified Output:* `aws-cli/2.36.9 Python/3.14.6 Linux/6.12.38+kali-amd64 exe/x86_64.kali.2025`

---

## 5. Step 3: Install & Verify kind & kubectl

### 5.1 Installation Instructions

| OS | kind Installation | kubectl Installation |
| :--- | :--- | :--- |
| **Windows** | `choco install kind` | `choco install kubernetes-cli` |
| **macOS** | `brew install kind` | `brew install kubectl` |
| **Linux (Ubuntu)** | `curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64 && chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind` | `sudo snap install kubectl --classic` |

### 5.2 Verification

Check the installed versions of `kind` and `kubectl`:
```bash
kind --version
kubectl version --client
```

#### Execution Output:
![Kind and Kubectl Verification](3.%20Kind%20and%20KubeCTL%20.png)

*Verified Output:*
- `kind version 0.23.0`
- `Client Version: v1.33.4` (Kustomize Version: `v5.5.0`)

---

## 6. Step 4: Install & Verify Helper Tools

### 6.1 Tool Setup Summary

* **OpenSSL:** Pre-installed on macOS/Linux and bundled with Git Bash on Windows.
* **oathtool:**
  * **macOS:** `brew install oath-toolkit`
  * **Linux:** `sudo apt install oathtool`
  * **Windows:** Use WSL or a phone authenticator app.
* **Trivy:** No standalone installation required. Run on-demand via Docker:
  ```bash
  docker run --rm aquasec/trivy image <image_name>
  ```

### 6.2 Verification

Verify OpenSSL and oathtool installations:
```bash
openssl version
oathtool --version
```

#### Execution Outputs:

**OpenSSL Verification:**  
![OpenSSL Verification](4.%20OpenSSL.png)  
*Verified Output:* `OpenSSL 3.5.2 5 Aug 2025 (Library: OpenSSL 3.5.2 5 Aug 2025)`

**oathtool Verification:**  
![Oathtool Verification](4.%20Oathtool.png)  
*Verified Output:* `oathtool (OATH Toolkit) 2.6.14`

---

## 7. Step 5: Start & Stop Lab Environments

### 7.1 LocalStack (Local AWS Cloud Simulator)

LocalStack emulates AWS services locally on port `4566`.

#### Lifecycle Commands:
```bash
# Start LocalStack container in detached mode
docker run -d --name localstack -p 4566:4566 localstack/localstack

# Verify health status of simulated AWS services
curl http://localhost:4566/_localstack/health

# Managing LocalStack container state
docker stop localstack
docker start localstack
docker rm -f localstack # Complete removal
```

#### Execution Output:
![Localstack Start and Stop](5.%20Localstack%20start%20and%20stop.png)  
*Verified Output:* Endpoint `http://localhost:4566/_localstack/health` returns HTTP 200 JSON with status `"available"` for services (`acm`, `apigateway`, `dynamodb`, `ec2`, `iam`, `lambda`, `s3`, `sts`, etc.), version `3.0.2`.

---

### 7.2 Kubernetes Cluster Management (`kind` & `kubectl`)

Local Kubernetes cluster named `ccse` created inside Docker.

#### Lifecycle Commands:
```bash
# Create cluster
kind create cluster --name ccse

# Check cluster info and node status
kubectl cluster-info --context kind-ccse
kubectl get nodes

# Delete cluster when lab is finished
kind delete cluster --name ccse
```

#### Execution Outputs:

**Creating Cluster & Checking Cluster Info:**  
![Kubernetes Create Cluster](5.%20Kubernetes%20create%20cluster.png)  
*Verified Output:* Control plane running at `https://127.0.0.1:42117` with CoreDNS available.

**Listing Nodes & Deleting Cluster:**  
![Kubernetes Nodes and Delete Cluster](5.%20kubernetes%20nodes%20and%20delete%20cluster.png)  
*Verified Output:* 
- Node `ccse-control-plane` in status `Ready` (role: `control-plane`, version `v1.30.0`).
- Successful cluster deletion of `ccse`.

---

## 8. Step 6: One-Time AWS CLI Configuration

LocalStack accepts arbitrary credentials. Configure dummy values once to prevent AWS CLI prompting:

```bash
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1
```

To simplify command execution, define an endpoint environment variable in your terminal session:
```bash
EP='--endpoint-url=http://localhost:4566'
aws $EP sts get-caller-identity
```

#### Execution Output:
![AWS CLI Configuration](6.%20AWS%20CLI%20configuration.png)  
*Verified Output:*
```json
{
    "UserId": "AKIAIOSFODNN7EXAMPLE",
    "Account": "000000000000",
    "Arn": "arn:aws:iam::000000000000:root"
}
```

> [!TIP]
> **Optional Shortcut:** Install `awslocal` wrapper via `pip install awscli-local`. This allows running `awslocal sts get-caller-identity` directly without specifying `$EP`.

---

## 9. Pre-Lab Verification Checklist

Before commencing Lab 1, confirm all tasks are verified:

- [x] **Docker:** `docker --version` prints valid version and `docker run hello-world` executes cleanly.
- [x] **AWS CLI:** `aws --version` reports `aws-cli/2.x`.
- [x] **kind & kubectl:** `kind --version` and `kubectl version --client` respond successfully.
- [x] **LocalStack:** LocalStack container starts and `curl http://localhost:4566/_localstack/health` returns service health JSON.
- [x] **AWS Identity:** `aws $EP sts get-caller-identity` successfully returns simulated root identity.
- [x] **Kubernetes Cluster:** `kind create cluster --name ccse` succeeds and `kubectl get nodes` lists `ccse-control-plane` in `Ready` state.
- [x] **Terminal Environment (Windows):** Operations executed within Git Bash / WSL.

---

## 10. Troubleshooting Guide

| Symptom | Cause | Solution |
| :--- | :--- | :--- |
| **Cannot connect to the Docker daemon** | Docker Desktop is not running | Launch Docker Desktop app. On Linux, ensure user is in `docker` group and re-login. |
| **Docker won't start / extremely slow** | Virtualization disabled | Enable VT-x/AMD-V/SVM in BIOS/UEFI. Enable WSL 2 & Virtual Machine Platform in Windows Features. |
| **Port 4566 already in use** | An existing LocalStack instance is running | Run `docker rm -f localstack` to stop and remove existing container, then restart. |
| **`aws: Could not connect to the endpoint URL`** | LocalStack is down or endpoint parameter omitted | Verify LocalStack is running (`docker ps`) and include `$EP` or `--endpoint-url=http://localhost:4566`. |
| **`aws: command not found` / `kubectl not found`** | Binary not in PATH | Re-run installation steps or restart terminal session to update PATH environment variable. |
| **Heredoc or sha256sum errors (Windows)** | Executing inside CMD or PowerShell | Switch shell environment to Git Bash or WSL (Ubuntu). |
| **`kind create cluster` fails** | Docker inactive or insufficient RAM allocated | Ensure Docker Desktop is running and allocated $\ge 4\text{ GB}$ of RAM. |
| **Slow image downloads in lab** | Network bandwidth limits | Pre-pull images on high-speed internet before lab session using `docker pull <image>`. |
| **MFA/TOTP verification failure (Lab 4)** | System clock skew | Enable automatic system time synchronization in operating system settings. |
| **NetworkPolicy not blocking traffic (Lab 2)** | Calico CNI pending | Wait for `calico-node` pods to achieve `Ready` state in Kubernetes cluster. |

---

## 11. Useful One-Liners & Session Management Commands

```bash
# Start LocalStack session (or run if not exists)
docker start localstack 2>/dev/null || docker run -d --name localstack -p 4566:4566 localstack/localstack
EP='--endpoint-url=http://localhost:4566'

# Check active resources
docker ps
kind get clusters

# Complete environment cleanup (frees disk space)
docker rm -f localstack
kind delete clusters --all
docker system prune -f
```

*Note: Deleting containers and kind clusters between sessions is safe. Student work, code, and report files reside on the local host filesystem.*
