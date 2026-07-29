# IKB42603 Lab 0 — Environment Setup

**Name:** Muhammad Arif Shafira Bin Shahrin Amri 
**Course:** IKB42603 Cloud Computing  
**Lab:** Lab 0 — Environment Setup  

## Objective

Set up and validate a local cloud-computing development environment with Docker, AWS CLI v2, Kubernetes tools (`kind` and `kubectl`), LocalStack, and supporting security tools.

## Learning Outcomes

At the end of this lab, I was able to:

- install and verify Docker container support;
- install AWS CLI v2 and configure test credentials for LocalStack;
- install and use `kind` and `kubectl` to create a local Kubernetes cluster;
- run LocalStack as a Docker container and connect to it through the AWS CLI; and
- use OpenSSL and Trivy as supporting tools for cloud and container-security tasks.

## Environment

| Item | Details |
| --- | --- |
| Host environment | Kali Linux terminal |
| Container runtime | Docker |
| Cloud CLI | AWS CLI v2 |
| Kubernetes tools | kind v0.23.0 and kubectl v1.33.4 |
| Local AWS emulator | LocalStack on `http://localhost:4566` |
| Kubernetes cluster | `ccse` (`kind-ccse` context) |
| Helper tools | OpenSSL and Trivy |

## Step-by-Step Implementation

### 1. Install and verify Docker

Docker was installed in the Kali Linux environment. After the Docker daemon was available, I ran the official `hello-world` image to confirm that the Docker client could contact the daemon, pull an image, create a container, and display its output.

```bash
docker run hello-world
```

The command displayed `Hello from Docker!`, confirming that Docker was working correctly.

![Docker hello-world verification](Docker%20Installer/Screenshot%202026-07-28%20004308.png)

### 2. Install AWS CLI v2

I downloaded the Linux x86_64 AWS CLI v2 installer, extracted it, installed it with administrator privileges, and checked the installed version.

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```
![alt text](<Screenshot 2026-07-28 011009.png>)

### 3. Install kind and kubectl

I installed `kind` for running Kubernetes clusters in Docker. I then installed `kubectl` and verified that the Kubernetes client was available. The captured output shows kubectl client version `v1.33.4`.

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind --version

kubectl version --client
```

![alt text](<Screenshot 2026-07-28 012300.png>)

![kubectl client verification](Install%20kind%20%26%20kubectl/Screenshot%202026-07-28%20012323.png)

### 4. Install and verify helper tools

OpenSSL was checked to ensure cryptographic tooling was present. Trivy was then run in a Docker container to scan the Alpine image. The scan completed and reported zero vulnerabilities for the tested image.

```bash
openssl version
docker run --rm aquasec/trivy image alpine
```
Expected Output:

![alt text](<Screenshot 2026-07-28 013424.png>)

Expected Output:

![Trivy scan of Alpine image](Helper%20Tools/Screenshot%202026-07-28%20020025.png)

### 5. Create and validate the Kubernetes cluster

I created a local kind cluster named `ccse`. The command created the control-plane node and set the active Kubernetes context to `kind-ccse`. I then checked the node status and cluster connectivity.

```bash
kind create cluster --name ccse
kubectl get nodes
kubectl cluster-info --context kind-ccse
```

![kind cluster creation](Kubernete%20Cluster/Screenshot%202026-07-28%20195806.png)

### 6. Start LocalStack

LocalStack was started as a detached Docker container, with its edge service published on port `4566`. I verified that the container was running using `docker ps`.

```bash
docker run -d --name localstack -p 4566:4566 localstack/localstack
docker ps
```

![LocalStack image download and container start](Local%20Stack/Screenshot%202026-07-28%20021020.png)

![LocalStack container verification](Local%20Stack/Screenshot%202026-07-28%20195331.png)

### 7. Configure AWS CLI for LocalStack

Because LocalStack accepts placeholder credentials, I configured the AWS CLI with test values and set a terminal variable for the LocalStack endpoint. An STS `get-caller-identity` request returned account ID `000000000000`, which confirmed successful AWS CLI communication with LocalStack.

```bash
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1

EP='--endpoint-url=http://localhost:4566'
aws $EP sts get-caller-identity
```

![AWS CLI successfully connected to LocalStack](One-Time%20AWS%20CLI/Screenshot%202026-07-28%20200126.png)

## Commands Used

```bash
# Docker
docker run hello-world
docker run -d --name localstack -p 4566:4566 localstack/localstack
docker ps

# AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1
EP='--endpoint-url=http://localhost:4566'
aws $EP sts get-caller-identity

# Kubernetes
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind --version
kubectl version --client
kind create cluster --name ccse
kubectl get nodes
kubectl cluster-info --context kind-ccse

# Helper tools
openssl version
docker run --rm aquasec/trivy image alpine
```

## Screenshots

The screenshots embedded in the implementation steps provide evidence of the completed work:

- Docker `hello-world` verification.
- AWS CLI v2 download.
- kind download and kubectl client version verification.
- Trivy Alpine image scan.
- kind cluster creation.
- LocalStack startup and running-container verification.
- AWS CLI STS request to LocalStack.

## Challenges Encountered

The Docker convenience-install script attempted to use a Docker Debian repository for `kali-rolling`, which did not provide a valid Release file. The terminal also showed hostname-resolution errors for the local hostname during that attempt. I treated this as an installation-method issue, used a supported installation approach for the environment, and verified the successful final Docker installation with `docker run hello-world`.

## Lessons Learned

- A successful container run is a stronger Docker check than only confirming that the command is installed.
- Package repositories must match the Linux distribution and release; an installation script intended for Debian may not work unchanged on Kali rolling.
- LocalStack allows AWS CLI practice without real cloud credentials or charges by using test credentials and a local endpoint.
- kind provides a practical Kubernetes cluster for local development because it runs the cluster nodes as Docker containers.
- Container-image scanning should be part of a basic cloud-development workflow.

## References

1. `IKB42603_Lab0_Environment_Setup_Cheatsheet.pdf` (lab guide provided for this exercise).
2. [Docker documentation](https://docs.docker.com/).
3. [AWS CLI version 2 installation guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).
4. [kind documentation](https://kind.sigs.k8s.io/).
5. [Kubernetes kubectl documentation](https://kubernetes.io/docs/reference/kubectl/).
6. [LocalStack documentation](https://docs.localstack.cloud/).
7. [Trivy documentation](https://trivy.dev/latest/).
