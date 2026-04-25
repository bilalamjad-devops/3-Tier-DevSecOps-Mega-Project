I have polished your article/README. I fixed grammar, improved clarity, organized the steps, and answered your specific question about `-n webapps`. I kept your original structure and commands intact.

Below is the polished version. You can copy-paste it directly into your `README.md`.

---

# CI/CD Pipeline: Jenkins → EKS (Node.js App)

This document walks through setting up a complete CI/CD pipeline using **Jenkins** to automate build, security scanning, and deployment of a Node.js application to a Kubernetes cluster on **AWS EKS**.

---

## 📦 Prerequisites & Servers

You need **three servers** with the following installations:

| Server Role | Required Tools |
|-------------|----------------|
| **EKS Server** | AWS CLI, `eksctl`, `kubectl`, IAM OIDC, EBS CSI driver, NGINX Ingress Controller, cert-manager |
| **Jenkins Server** | Jenkins, GitLeaks, Trivy, Docker (official) |
| **SonarQube Server** | SonarQube, Docker (official) |

---

## 🚀 Step 1 – EKS Cluster Setup (Infrastructure as Code)

Use Terraform to provision VPC, subnets, security groups, EKS cluster, and node group.

### 📁 Terraform Configuration (`main.tf`)

```hcl
provider "aws" {
  region = "ap-south-1"
}

resource "aws_vpc" "devopsshack_vpc" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "devopsshack-vpc" }
}

resource "aws_subnet" "devopsshack_subnet" {
  count = 2
  vpc_id                  = aws_vpc.devopsshack_vpc.id
  cidr_block              = cidrsubnet(aws_vpc.devopsshack_vpc.cidr_block, 8, count.index)
  availability_zone       = element(["ap-south-1a", "ap-south-1b"], count.index)
  map_public_ip_on_launch = true
  tags = { Name = "devopsshack-subnet-${count.index}" }
}

resource "aws_internet_gateway" "devopsshack_igw" {
  vpc_id = aws_vpc.devopsshack_vpc.id
  tags   = { Name = "devopsshack-igw" }
}

resource "aws_route_table" "devopsshack_route_table" {
  vpc_id = aws_vpc.devopsshack_vpc.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.devopsshack_igw.id
  }
  tags = { Name = "devopsshack-route-table" }
}

resource "aws_route_table_association" "devopsshack_association" {
  count          = 2
  subnet_id      = aws_subnet.devopsshack_subnet[count.index].id
  route_table_id = aws_route_table.devopsshack_route_table.id
}

resource "aws_security_group" "devopsshack_cluster_sg" {
  vpc_id = aws_vpc.devopsshack_vpc.id
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = { Name = "devopsshack-cluster-sg" }
}

resource "aws_security_group" "devopsshack_node_sg" {
  vpc_id = aws_vpc.devopsshack_vpc.id
  ingress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = { Name = "devopsshack-node-sg" }
}

resource "aws_eks_cluster" "devopsshack" {
  name     = "devopsshack-cluster"
  role_arn = aws_iam_role.devopsshack_cluster_role.arn
  vpc_config {
    subnet_ids         = aws_subnet.devopsshack_subnet[*].id
    security_group_ids = [aws_security_group.devopsshack_cluster_sg.id]
  }
}

resource "aws_eks_addon" "ebs_csi_driver" {
  cluster_name                = aws_eks_cluster.devopsshack.name
  addon_name                  = "aws-ebs-csi-driver"
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "OVERWRITE"
}

resource "aws_eks_node_group" "devopsshack" {
  cluster_name    = aws_eks_cluster.devopsshack.name
  node_group_name = "devopsshack-node-group"
  node_role_arn   = aws_iam_role.devopsshack_node_group_role.arn
  subnet_ids      = aws_subnet.devopsshack_subnet[*].id
  scaling_config {
    desired_size = 3
    max_size     = 3
    min_size     = 3
  }
  instance_types = ["t2.medium"]
  remote_access {
    ec2_ssh_key               = var.ssh_key_name
    source_security_group_ids = [aws_security_group.devopsshack_node_sg.id]
  }
}

resource "aws_iam_role" "devopsshack_cluster_role" {
  name = "devopsshack-cluster-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "eks.amazonaws.com" }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "devopsshack_cluster_role_policy" {
  role       = aws_iam_role.devopsshack_cluster_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
}

resource "aws_iam_role" "devopsshack_node_group_role" {
  name = "devopsshack-node-group-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "devopsshack_node_group_role_policy" {
  role       = aws_iam_role.devopsshack_node_group_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
}

resource "aws_iam_role_policy_attachment" "devopsshack_node_group_cni_policy" {
  role       = aws_iam_role.devopsshack_node_group_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
}

resource "aws_iam_role_policy_attachment" "devopsshack_node_group_registry_policy" {
  role       = aws_iam_role.devopsshack_node_group_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
}

resource "aws_iam_role_policy_attachment" "devopsshack_node_group_ebs_policy" {
  role       = aws_iam_role.devopsshack_node_group_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"
}

variable "ssh_key_name" {
  description = "Name of the SSH key pair for instances"
  type        = string
  default     = "DevOps-Shack"
}

output "cluster_id"     { value = aws_eks_cluster.devopsshack.id }
output "node_group_id"  { value = aws_eks_node_group.devopsshack.id }
output "vpc_id"         { value = aws_vpc.devopsshack_vpc.id }
output "subnet_ids"     { value = aws_subnet.devopsshack_subnet[*].id }
```

### Apply Terraform

```bash
terraform plan
terraform apply --auto-approve
```

### Update kubeconfig

```bash
aws eks --region ap-south-1 update-kubeconfig --name devopsshack-cluster
```

---

## 🔐 IAM OIDC & EBS CSI Driver

### Associate OIDC provider

```bash
eksctl utils associate-iam-oidc-provider \
  --region ap-south-1 \
  --cluster devopsshack-cluster \
  --approve
```

### Create IAM service account for EBS CSI driver

```bash
eksctl create iamserviceaccount \
  --region ap-south-1 \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster devopsshack-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --override-existing-serviceaccounts
```

### Deploy add-ons

```bash
# EBS CSI Driver
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-1.11"

# NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

# cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml
```

---

## 🛡️ RBAC for Jenkins Service Account

Create a namespace `webapps` and a service account `jenkins` with appropriate roles.

### Step-by-step RBAC setup

```bash
# Create namespace (if not exists)
kubectl create namespace webapps
```

#### 1. ServiceAccount

```yaml
# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: webapps
```

```bash
kubectl apply -f serviceaccount.yaml
```

#### 2. Role (namespace-scoped)

```yaml
# role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: jenkins-role
  namespace: webapps
rules:
  - apiGroups: [""]
    resources: ["secrets", "configmaps", "persistentvolumeclaims", "services", "pods"]
    verbs: ["get", "list", "watch", "create", "update", "delete", "patch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets", "statefulsets"]
    verbs: ["get", "list", "watch", "create", "update", "delete", "patch"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch", "create", "update", "delete", "patch"]
  - apiGroups: ["autoscaling"]
    resources: ["horizontalpodautoscalers"]
    verbs: ["get", "list", "watch", "create", "update", "delete", "patch"]
```

```bash
kubectl apply -f role.yaml
```

#### 3. RoleBinding

```yaml
# rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-rolebinding
  namespace: webapps
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: jenkins-role
subjects:
  - kind: ServiceAccount
    name: jenkins
    namespace: webapps
```

```bash
kubectl apply -f rolebinding.yaml
```

#### 4. ClusterRole (cluster-scoped resources)

```yaml
# clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: jenkins-cluster-role
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "update", "delete"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch", "create", "update", "delete"]
  - apiGroups: ["cert-manager.io"]
    resources: ["clusterissuers"]
    verbs: ["get", "list", "watch", "create", "update", "delete"]
```

```bash
kubectl apply -f clusterrole.yaml
```

#### 5. ClusterRoleBinding

```yaml
# clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-cluster-rolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins-cluster-role
subjects:
  - kind: ServiceAccount
    name: jenkins
    namespace: webapps
```

```bash
kubectl apply -f clusterrolebinding.yaml
```

#### 6. Create a secret for the service account token

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: jenkins-token
  namespace: webapps
  annotations:
    kubernetes.io/service-account.name: jenkins
```

```bash
kubectl apply -f secret.yaml
```

#### 7. Retrieve the token

```bash
kubectl describe secret jenkins-token -n webapps
```

Copy the `token` value – you will add it as a **secret text** credential in Jenkins.

---

### ❓ Answer to your question: *Where should I run `kubectl apply -f -n webapps`?*

The `-n` flag specifies the namespace. If the YAML file already contains `namespace: webapps`, you can omit `-n`. But to be safe, always specify the namespace explicitly:

```bash
kubectl apply -f <filename> -n webapps
```

For example:
```bash
kubectl apply -f serviceaccount.yaml -n webapps
kubectl apply -f role.yaml -n webapps
kubectl apply -f rolebinding.yaml -n webapps
kubectl apply -f secret.yaml -n webapps
```

The `clusterrole.yaml` and `clusterrolebinding.yaml` are cluster-scoped, so they **do not** take a namespace flag.

---

## 🖥️ Server 2 – Jenkins Server

Install on this server:

- Jenkins
- GitLeaks
- Trivy
- Docker (official)

> Detailed installation steps are not covered here – refer to official documentation.

---

## 📊 Server 3 – SonarQube Server

Install SonarQube (usually via Docker) on a separate server.

```bash
docker run -d --name sonarqube -p 9000:9000 sonarqube:lts
```

Access SonarQube at `http://<server-ip>:9000` (default credentials: admin / admin).

---

## 🔌 Step 4 – Integrate Jenkins with SonarQube

1. In SonarQube, generate a token (**User → My Account → Security**).
2. In Jenkins, install **SonarQube Scanner** plugin.
3. Go to **Manage Jenkins → Configure System → SonarQube servers**.
   - Add server with name `sonar` and URL `http://<sonarqube-ip>:9000`.
   - Add the token as a secret text credential with ID `sonar-token`.

---

## 🧩 Step 5 – Jenkins Plugins to Install

- NodeJS Plugin
- Pipeline Stage View
- Docker Pipeline
- Kubernetes CLI
- Kubernetes Credentials
- (also SonarQube Scanner, GitLeaks, Trivy – if available, or run via shell)

---

## 🔐 Step 6 – Jenkins Credentials (IDs used in pipeline)

| Credential ID  | Type        | Purpose                         |
|----------------|-------------|---------------------------------|
| `sonar-token`  | Secret text | SonarQube authentication        |
| `docker-cred`  | Username/password (or Docker Hub token) | Pushing images to Docker Hub |
| `k8-token`     | Secret text | Kubernetes API access token (from the secret created above) |

---

## 🚦 Step 7 – Jenkins Pipeline (Declarative)

Below is the complete `Jenkinsfile`.  
**Note:** Replace placeholders:
- `<your-dockerhub-username>`
- `serverUrl` (your EKS API server endpoint – find it via `aws eks describe-cluster --name devopsshack-cluster`)

```groovy
pipeline {
    agent any
    
    tools {
        nodejs 'nodejs23'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'deploy-to-dev-k8', 
                    url: 'https://github.com/jaiswaladi246/3-Tier-DevSecOps-Mega-Project.git'
            }
        }
        
        stage('Frontend Compilation') {
            steps {
                dir('client') {
                    sh 'find . -name "*.js" -exec node --check {} +'
                }
            }
        }
        
        stage('Backend Compilation') {
            steps {
                dir('api') {
                    sh 'find . -name "*.js" -exec node --check {} +'
                }
            }
        }
        
        stage('GitLeaks Scan') {
            steps {
                sh 'gitleaks detect --source ./client --exit-code 1'
                sh 'gitleaks detect --source ./api --exit-code 1'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=NodeJS-Project \
                        -Dsonar.projectKey=NodeJS-Project'''
                }
            }
        }
        
        stage('Quality Gate Check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        
        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --format table -o fs-report.html .'
            }
        }
        
        stage('Build-Tag & Push Backend Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        dir('api') {
                            sh 'docker build -t <your-dockerhub-username>/backend:latest .'
                            sh 'trivy image --format table -o backend-image-report.html <your-dockerhub-username>/backend:latest'
                            sh 'docker push <your-dockerhub-username>/backend:latest'
                        }
                    }
                }
            }
        }
        
        stage('Build-Tag & Push Frontend Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        dir('client') {
                            sh 'docker build -t <your-dockerhub-username>/frontend:latest .'
                            sh 'trivy image --format table -o frontend-image-report.html <your-dockerhub-username>/frontend:latest'
                            sh 'docker push <your-dockerhub-username>/frontend:latest'
                        }
                    }
                }
            }
        }
        
        stage('K8-deploy') {
            steps {
                script {
                    withKubeConfig(
                        caCertificate: '',
                        clusterName: 'devopsshack-cluster',
                        contextName: '',
                        credentialsId: 'k8-token',
                        namespace: 'dev',
                        restrictKubeConfigAccess: false,
                        serverUrl: 'https://<your-eks-api-server-endpoint>'
                    ) {
                        sh 'kubectl apply -f k8s/sc.yaml -n dev'
                        sh 'kubectl apply -f k8s/mysql.yaml -n dev'
                        sh 'kubectl apply -f k8s/backend.yaml -n dev'
                        sh 'kubectl apply -f k8s/frontend.yaml -n dev'
                        sleep 30
                    }
                }
            }
        }
        
        stage('verify-K8-deploy') {
            steps {
                script {
                    withKubeConfig(
                        caCertificate: '',
                        clusterName: 'devopsshack-cluster',
                        contextName: '',
                        credentialsId: 'k8-token',
                        namespace: 'dev',
                        restrictKubeConfigAccess: false,
                        serverUrl: 'https://<your-eks-api-server-endpoint>'
                    ) {
                        sh 'kubectl get pods -n dev'
                        sh 'kubectl get svc -n dev'
                    }
                }
            }
        }
    }
}
```

---

## ✅ Final Notes

- The pipeline assumes your Kubernetes YAML manifests (`sc.yaml`, `mysql.yaml`, `backend.yaml`, `frontend.yaml`) exist inside the `k8s/` folder in your repository.
- The `dev` namespace must be created beforehand:
  ```bash
  kubectl create namespace dev
  ```
- Replace all placeholders before running the pipeline.

---

Commit Date: 25-April-2026
