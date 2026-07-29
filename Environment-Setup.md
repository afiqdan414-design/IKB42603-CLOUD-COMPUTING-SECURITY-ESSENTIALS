# Environment Setup

This document summarizes the step-by-step environment setup required for the lab, based on the guide `IKB42603_Lab0_Environment_Setup_Cheatsheet.pdf`.

## 1. Prerequisites

1. Use a workstation with administrative privileges.
2. Ensure internet access for downloading tools and packages.
3. Verify the operating system is compatible with Docker, Kubernetes tools, and LocalStack.

## 2. Install Docker

1. Download and install Docker Desktop for your OS from the official Docker website.
2. Start Docker Desktop.
3. Open a terminal and run:
   - `docker --version`
   - `docker info`
4. Confirm Docker is running properly.

## 3. Install AWS CLI

1. Download and install the AWS CLI version 2.
2. Open a terminal and run:
   - `aws --version`
3. Configure AWS CLI credentials:
   - `aws configure`
4. Verify configuration:
   - `aws configure list`
   - `aws sts get-caller-identity`

## 4. Install Kubernetes Tools: kind and kubectl

1. Install `kind` for local Kubernetes cluster management.
2. Install `kubectl` for Kubernetes command-line operations.
3. Verify installations:
   - `kind version`
   - `kubectl version --client`

## 5. Install OpenSSL and oathtool

1. Install OpenSSL on your system.
2. Install `oathtool` for OTP generation.
3. Verify installations:
   - `openssl version`
   - `oathtool --version`

## 6. Configure AWS CLI

1. Edit or create AWS CLI configuration if required.
2. Ensure the following values are set correctly:
   - AWS Access Key ID
   - AWS Secret Access Key
   - Default region name
   - Default output format
3. Confirm the AWS profile is active and valid.

## 7. Start LocalStack

1. Install LocalStack if not already installed.
2. Start LocalStack using the supported command or service.
3. Verify LocalStack is running and healthy.

## 8. Create a Local Kubernetes Cluster

1. Create the cluster with:
   - `kind create cluster`
2. Check the cluster status:
   - `kubectl get nodes`
3. Confirm the nodes are `Ready`.

## 9. Manage and Delete the Cluster

1. View cluster nodes:
   - `kubectl get nodes`
2. When finished, delete the cluster:
   - `kind delete cluster`

## 10. Notes and Troubleshooting

- If a tool is not found, confirm it is installed and on the system `PATH`.
- Ensure Docker is running before starting Kubernetes cluster operations.
- Re-run `aws configure` if credentials need to be updated.
- Use the official documentation for detailed installation instructions when needed.
