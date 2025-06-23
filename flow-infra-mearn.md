# DevOps EC2 Toolchain Setup

## 🚀 Step 1: Provision EC2 Instance with DevOps Toolchain

### 🖥️ EC2 Instance Setup

* **Operating System**: Ubuntu 22.04
* **Instance Type**: t3.medium or higher recommended
* **Storage**: 30 GB GP2 (minimum)
* **Security Group**: Open ports 22 (SSH), 8080 (Jenkins), 9000 (SonarQube), 80/443 (as needed)
* **IAM Role (Instance Profile)**: Attach an IAM role with the following policies:

  * `AmazonEC2FullAccess`
  * `AmazonEKSClusterPolicy`
  * `AmazonEKSWorkerNodePolicy`
  * `AmazonECRFullAccess`
  * `AmazonS3FullAccess`
  * `CloudWatchAgentServerPolicy`
  * `IAMFullAccess` *(optional / can use least privilege)*

### 👇 User Data Script

Paste the following script into the **User Data** field during EC2 provisioning (Advanced options):

```bash
#!/bin/bash
# Ubuntu 22.04 EC2 Bootstrapping Script

# Java Installation
sudo apt update -y
sudo apt install openjdk-17-jre -y
sudo apt install openjdk-17-jdk -y
java --version

# Jenkins Installation
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y

# Docker Installation
sudo apt install docker.io -y
sudo usermod -aG docker jenkins
sudo usermod -aG docker ubuntu
sudo systemctl restart docker
sudo chmod 777 /var/run/docker.sock

# Optional: Jenkins as Docker Container
# docker run -d -p 8080:8080 -p 50000:50000 --name jenkins-container jenkins/jenkins:lts

# SonarQube as Docker Container
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community

# AWS CLI Installation
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install

# kubectl Installation
sudo apt install curl -y
sudo curl -LO "https://dl.k8s.io/release/v1.28.4/bin/linux/amd64/kubectl"
sudo chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client

# eksctl Installation
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version

# Terraform Installation
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update
sudo apt install terraform -y

# Trivy Installation
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt update
sudo apt install trivy -y

# Helm Installation
sudo snap install helm --classic
```

---

## ⚙️ Step 2: Jenkins Plugins Setup for AWS & Terraform Integration

### ✅ Plugins Installed

#### 1. **AWS Credentials Plugin**

* **Name**: `AWS Credentials`
* **Purpose**:

  * Stores IAM Access Key ID & Secret Access Key in Jenkins Credentials store.
  * Supports MFA tokens and IAM Roles (advanced use).

#### 2. **Pipeline: AWS Steps Plugin**

* **Name**: `Pipeline: AWS Steps`
* **Purpose**:

  * Adds pipeline support to interact with AWS API using scripted or declarative pipelines.

#### 3. **Terraform Plugin**

* **Name**: `Terraform`
* **Purpose**:

  * Allows you to run `terraform init`, `plan`, `apply`, etc., as Jenkins build steps.
  * Useful for IAC pipelines.

---

### 🔐 Jenkins Credentials Setup for AWS

* **Navigate to**: `Manage Jenkins > Credentials > Global > Add Credentials`
* **Kind**: `AWS Credentials`
* **ID**: `aws-creds` *(use this ID in pipeline scripts)*
* **Access Key ID**: `<Your AWS Access Key>`
* **Secret Access Key**: `<Your AWS Secret Key>`
* **Scope**: `Global`

---

### 🧰 Terraform Configuration in Jenkins

* **Navigate to**: `Manage Jenkins > Global Tool Configuration`
* Under **Terraform installations**:

  * Click `Add Terraform`
  * **Name**: `terraform`
  * **Install Directory**: `/usr/bin/terraform` *(already installed in EC2)*
  * **Install Automatically**: (Leave unchecked)

---


# 🛠️ Step 3: Provisioning EKS Cluster using Jenkins + Terraform

We automated EKS provisioning using a Jenkins pipeline and Terraform code hosted on GitHub.

---

## 📁 Folder Structure

```
eks/
├── backend.tf             # Remote backend config (S3 + DynamoDB)
├── dev.tfvars             # Environment-specific variables
├── main.tf                # Main Terraform config
├── variables.tf           # Variable declarations
├── .terraform.lock.hcl    # Provider lock file

module/
├── eks.tf                 # EKS cluster definition
├── vpc.tf                 # VPC setup
├── iam.tf                 # IAM roles & policies
├── gather.tf              # Data sources/outputs
├── variables.tf           # Module variables
```

---

## 🧩 Jenkins Pipeline Setup

**Job Name**: `Infrastructure-Job`

**Steps:**
1. Go to `Jenkins Dashboard > New Item`
2. Enter: `Infrastructure-Job`
3. Choose: **Pipeline**
4. Click OK

---

## 🧾 Jenkinsfile Logic (Declarative Pipeline)

### 🔧 Parameters
```groovy
Environment: 'dev'
Terraform_Action: ['plan', 'apply', 'destroy']
```

### 🚀 Pipeline Stages

#### 1. Preparing
```sh
echo Preparing
```

#### 2. Git Pulling
```groovy
git branch: 'master', url: 'https://github.com/AmanPathak-DevOps/EKS-Terraform-GitHub-Actions.git'
```

#### 3. Init
```sh
terraform -chdir=eks/ init
```

#### 4. Validate
```sh
terraform -chdir=eks/ validate
```

#### 5. Plan/Apply/Destroy
```sh
terraform -chdir=eks/ plan -var-file=dev.tfvars
terraform -chdir=eks/ apply -var-file=dev.tfvars -auto-approve
terraform -chdir=eks/ destroy -var-file=dev.tfvars -auto-approve
```

---

## ✅ Execution Summary

- Successfully cloned the repo.
- Initialized and validated Terraform.
- Plan: `38 to add, 0 to change, 0 to destroy`.
- Apply: VPC, EKS cluster, IAM roles created.

---

## 🔐 Credentials & Plugins Recap

- AWS Credentials in Jenkins ID: `aws-creds`
- Plugins:
  - Pipeline
  - Pipeline: Stage View
  - Docker Pipeline (optional)

---


# 🛡️ Step 4: Create & Configure Jump Server in EKS VPC

After provisioning our EKS cluster, we need a secure point of entry into the private network hosting the cluster. This is achieved by launching and configuring a **Jump Server** inside the same VPC as our EKS cluster.

### 🛠️ Launching the EC2 Jump Server

We launched a new EC2 instance using the following settings:
- **Name**: Jump-Server
- **AMI**: Ubuntu 22.04 LTS
- **Instance Type**: `t2.medium`
- **VPC**: `dev-medium-vpc`
- **Subnet**: Public subnet (`dev-medium-subnet-public-1`)
- **Public IP**: Enabled (Auto-assigned)
- **Security Group**: Created a new one to allow SSH access

### ⚙️ EC2 User Data Configuration

We provided a User Data script during instance launch to automatically install the following essential tools on boot:
- AWS CLI
- kubectl
- Helm
- eksctl

This ensures the jump server is pre-equipped to interact with EKS clusters and manage deployments via Kubernetes tools.

### ✅ Finalizing Jump Server Configuration

Once the instance was up and running, we connected to it via SSH and performed the following:

#### 1. **Configure AWS CLI credentials**:
```bash
aws configure
# Provided Access Key, Secret Key, Region (us-east-1), and output format (json)
```

#### 2. **Update kubeconfig to access the EKS cluster**:
```bash
aws eks update-kubeconfig --name dev-medium-eks-cluster --region us-east-1
```

This command added the EKS cluster context to the kubeconfig file located at `/home/ubuntu/.kube/config`.

#### 3. **Verify cluster connectivity**:
```bash
kubectl get nodes
```

We successfully listed the worker nodes, which confirms the jump server can interact with the EKS cluster.

# ⚙️ Step 5: Deploying AWS Load Balancer Controller in EKS

The AWS Load Balancer Controller allows Kubernetes applications to use AWS Elastic Load Balancers (ALBs/NLBs) to expose services externally. Here’s how we deployed and verified it in our EKS cluster.

---

## 🧾 IAM Policy Setup

We first downloaded and created an IAM policy required by the AWS Load Balancer Controller:

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```

Then we created the IAM policy in AWS Management Console or via AWS CLI to be associated with the service account.

---

## 🔐 Create IAM Service Account Using `eksctl`

To grant Kubernetes permission to use the Load Balancer features, we created a Kubernetes service account bound to the IAM policy:

```bash
eksctl create iamserviceaccount   --cluster dev-medium-eks-cluster   --namespace kube-system   --name aws-load-balancer-controller   --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy   --approve
```

We verified the service account creation:

```bash
kubectl get sa -n kube-system
```

---

## 📦 Add Helm Repository for AWS Controller

We added the EKS Helm chart repository and updated it:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

We confirmed it with:

```bash
helm repo list
```

---

## 🚀 Deploy AWS Load Balancer Controller via Helm

Finally, we installed the controller into our EKS cluster using Helm:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller   -n kube-system   --set clusterName=dev-medium-eks-cluster   --set serviceAccount.create=false   --set serviceAccount.name=aws-load-balancer-controller
```

---

## ✅ Verify the Deployment

We confirmed successful deployment using:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

You should see the controller running with READY status (e.g., 2/2 pods available).

---

This completes the setup of the AWS Load Balancer Controller in our EKS environment. This controller is now capable of managing ALBs for Kubernetes ingress resources based on your application deployments.

# 🚀 Step 6: Argo CD Deployment on Amazon EKS

This guide documents the step-by-step instructions to deploy Argo CD on an Amazon EKS cluster and expose its UI using a LoadBalancer.

---

## 1. Create Namespace

```bash
kubectl create namespace argocd
```

---

## 2. Install Argo CD in the Namespace

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.7/manifests/install.yaml
```

This will install:

* CRDs (Custom Resource Definitions)
* Service Accounts
* Roles, RoleBindings, ClusterRoles, ClusterRoleBindings
* ConfigMaps
* Secrets
* Services
* Deployments and StatefulSets

---

## 3. Verify All Components Are Running

```bash
kubectl get all -n argocd
```

Ensure all pods show `STATUS = Running`.

---

## 4. Get Argo CD Server Service Info

```bash
kubectl get svc -n argocd
```

Look for the `argocd-server` service. Initially, it is of `ClusterIP` type.

---

## 5. Edit the Argo CD Server Service to Expose It via LoadBalancer

```bash
kubectl edit svc argocd-server -n argocd
```

Modify this line:

```yaml
type: ClusterIP
```

to:

```yaml
type: LoadBalancer
```

> **Note:** When changing the `argocd-server` service type to `LoadBalancer`, the AWS Cloud Controller Manager component of the EKS cluster is responsible for provisioning the AWS ALB. This is **not** handled by the AWS Load Balancer Controller that we might have installed separately.

---

## 6. Wait for LoadBalancer to be Provisioned

After saving the change, check again:

```bash
kubectl get svc -n argocd
```

Copy the `EXTERNAL-IP` of `argocd-server`. It may take a few minutes for AWS to provision it.

---

## 7. Get the Initial Admin Password

```bash
kubectl get secrets -n argocd
kubectl get secret argocd-initial-admin-secret -n argocd -o yaml
```

Then decode it:

```bash
echo <base64-password> | base64 --decode
```

---

## 8. Access Argo CD UI

Go to `http://<EXTERNAL-IP>` or `https://<EXTERNAL-IP>` in your browser.
Login with:

* **Username**: `admin`
* **Password**: `<decoded-password>`

You should see the Argo CD dashboard.

---

✅ You have now successfully deployed and accessed Argo CD on Amazon EKS!


