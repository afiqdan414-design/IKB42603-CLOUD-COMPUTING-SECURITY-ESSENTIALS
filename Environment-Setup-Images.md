# Environment Setup with Reference Images

This document summarizes the environment setup steps and includes the relevant screenshots from the lab folder.

## 1. Docker

- Install and start Docker Desktop.
- Verify Docker is running.

![Docker](docker.png)

## 2. AWS CLI

- Install the AWS CLI.
- Configure AWS credentials and verify the setup.

![AWS CLI](AWS CLI.png)

## 3. Kind and kubectl

- Install `kind` and `kubectl`.
- Use them to create and manage local Kubernetes clusters.

![Kind and KubeCTL](Kind and KubeCTL .png)

## 4. OpenSSL

- Install OpenSSL for certificate and cryptographic operations.

![OpenSSL](OpenSSL.png)

## 5. oathtool

- Install `oathtool` to generate TOTP codes.

![Oathtool](Oathtool.png)

## 6. Kubernetes Cluster Creation

- Create a local Kubernetes cluster using `kind`.

![Kubernetes create cluster](Kubernetes create cluster.png)

## 7. Kubernetes Status and Cleanup

- Check cluster nodes and delete the cluster when finished.

![kubernetes nodes and delete cluster](kubernetes nodes and delete cluster.png)

## 8. LocalStack Start and Stop

- Start and stop LocalStack for local AWS service emulation.

![Localstack start and stop](Localstack start and stop.png)

## 9. AWS CLI Configuration

- Confirm AWS CLI configuration using `aws configure`.

![AWS CLI configuration](AWS CLI configuration.png)
