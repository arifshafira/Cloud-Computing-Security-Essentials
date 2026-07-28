# IKB42603 Lab 0 — Environment Setup

This report records the environment setup completed for **IKB42603 Cloud Computing Lab 0**, following `IKB42603_Lab0_Environment_Setup_Cheatsheet.pdf`. The commands shown target the Kali Linux environment used in the supplied evidence.

## Prerequisites

Ensure that you have an internet connection and a terminal with `sudo` access. The required tools are Docker, AWS CLI v2, `kubectl`, kind, LocalStack, and helper tools used by the lab.

## 1. Install and verify Docker

1. Install Docker using the method appropriate to your Linux distribution. On a supported Debian/Ubuntu installation, follow Docker's official installation instructions. Do not use the convenience script on Kali rolling, as it may select an unsupported repository.
2. Start Docker and ensure the current user is permitted to run Docker commands.
3. Verify that the Docker daemon is working:

   ```bash
   docker run hello-world
   ```

   The expected result includes `Hello from Docker!`.

   <img width="727" height="377" alt="Screenshot 2026-07-28 185915" src="https://github.com/user-attachments/assets/38e61fb2-93f9-4bf4-81ee-bfa197ac24b3" />
   

## 3. Install AWS CLI v2

1. Download the AWS CLI v2 package for Linux x86_64:

   ```bash
   curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
   ```

2. Extract the archive:

   ```bash
   unzip awscliv2.zip
   ```

3. Install the CLI:

   ```bash
   sudo ./aws/install
   ```

4. Confirm the installation:

   ```bash
   aws --version
   ```

## 3. Install kind and kubectl

### Install kind

1. Download kind version 0.23.0:

   ```bash
   curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
   ```

2. Make it executable and place it on the system path:

   ```bash
   chmod +x ./kind
   sudo mv ./kind /usr/local/bin/kind
   ```

3. Verify it:

   ```bash
   kind --version
   ```

   Expected: `kind version 0.23.0`.

### Install kubectl

1. Install `kubectl` using the Kubernetes Linux installation method appropriate to your architecture.
2. Confirm the client is available:

   ```bash
   kubectl version --client
   ```

   The captured environment returned Kubernetes client version `v1.33.4`.

   ![kubectl client verification](Install%20kind%20%26%20kubectl/Screenshot%202026-07-28%20012323.png)


## 8. Install and verify helper tools

The lab environment also uses helper tooling such as OpenSSL and Trivy.

1. Confirm OpenSSL is installed:

   ```bash
   openssl version
   ```
   Output:
![OpenSSL Version](Helper%20Tools/Screenshot%202026-07-28%013424.png)


2. Use Trivy to scan a container image:

   ```bash
   docker run --rm aquasec/trivy image alpine
   ```

   The sample scan completed successfully and reported zero vulnerabilities for the scanned Alpine image.

   ![Trivy scan result](Helper%20Tools/Screenshot%202026-07-28%20020025.png)


## 5. Create and validate a Kubernetes cluster

1. Create a kind cluster called `ccse`:

   ```bash
   kind create cluster --name ccse
   ```

2. Confirm the node becomes ready:

   ```bash
   kubectl get nodes
   ```

   Expected: `ccse-control-plane` has status `Ready`.

3. Inspect cluster connectivity:

   ```bash
   kubectl cluster-info --context kind-ccse
   ```

   ![kind cluster created](Kubernete%20Cluster/Screenshot%202026-07-28%20195806.png)

## 6. Start LocalStack

1. Check whether a LocalStack container already exists:

   ```bash
   docker ps
   ```

2. If needed, remove an earlier container and start a fresh instance on port 4566:

   ```bash
   docker rm -f localstack 2>/dev/null || true
   docker run -d --name localstack -p 4566:4566 localstack/localstack
   ```

3. Verify that it is running:

   ```bash
   docker ps
   ```

   Expected: the `localstack/localstack` container is `Up` and port `4566` is mapped.

   ![LocalStack container verification](Local%20Stack/Screenshot%202026-07-28%20195331.png)

## 7. Configure the AWS CLI for LocalStack (one-time setup)

Use placeholder credentials because LocalStack does not require real AWS credentials:

```bash
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1
```

Set the LocalStack endpoint for the current terminal:

```bash
EP='--endpoint-url=http://localhost:4566'
```

Test the LocalStack AWS endpoint:

```bash
aws $EP sts get-caller-identity
```

Expected: a JSON response with account ID `000000000000`, confirming AWS CLI communication with LocalStack.

![AWS CLI to LocalStack verification](One-Time%20AWS%20CLI/Screenshot%202026-07-28%20200126.png)



## Final checklist

- [x] Docker runs `hello-world` successfully.
- [x] AWS CLI v2 is installed and configured with LocalStack test credentials.
- [x] `kind` and `kubectl` are installed.
- [x] The `ccse` Kubernetes control-plane node is `Ready`.
- [x] LocalStack is running on `http://localhost:4566`.
- [x] AWS STS calls succeed through the LocalStack endpoint.
- [x] Helper-tool verification was completed.
