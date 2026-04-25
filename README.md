

I

In this document, we are setting up a CI/CD pipeline using Jenkins to automate the build, security scanning, and deployment of a NodeJS application to a Kubernetes cluster on AWS EKS.






Prerequisites and Servers:

We need 3 servers with the following installations:


1. EKS Server -> AWS CLI, EKS Setup and RBAC, kubectl, eksctl, kubeconfig,  oidc, iam server account for csi driver , ebs csi driver , nginx ingress controller , cert manager  

2. Jenkins Server -> Jenkins, GitLeaks, Trivy, and Official Docker

3. SonarQube Server -> SonarQube and Offical Docker







<details>

  <summary>Step 1: VM Server</summary>







aws cli 

```bash

```

eks

```bash
provider "aws" {
  region = "ap-south-1"
}

resource "aws_vpc" "devopsshack_vpc" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "devopsshack-vpc"
  }
}

resource "aws_subnet" "devopsshack_subnet" {
  count = 2
  vpc_id                  = aws_vpc.devopsshack_vpc.id
  cidr_block              = cidrsubnet(aws_vpc.devopsshack_vpc.cidr_block, 8, count.index)
  availability_zone       = element(["ap-south-1a", "ap-south-1b"], count.index)
  map_public_ip_on_launch = true

  tags = {
    Name = "devopsshack-subnet-${count.index}"
  }
}

resource "aws_internet_gateway" "devopsshack_igw" {
  vpc_id = aws_vpc.devopsshack_vpc.id

  tags = {
    Name = "devopsshack-igw"
  }
}

resource "aws_route_table" "devopsshack_route_table" {
  vpc_id = aws_vpc.devopsshack_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.devopsshack_igw.id
  }

  tags = {
    Name = "devopsshack-route-table"
  }
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

  tags = {
    Name = "devopsshack-cluster-sg"
  }
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

  tags = {
    Name = "devopsshack-node-sg"
  }
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
  cluster_name    = aws_eks_cluster.devopsshack.name
  addon_name      = "aws-ebs-csi-driver"
  
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
    ec2_ssh_key = var.ssh_key_name
    source_security_group_ids = [aws_security_group.devopsshack_node_sg.id]
  }
}

resource "aws_iam_role" "devopsshack_cluster_role" {
  name = "devopsshack-cluster-role"

  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
}

resource "aws_iam_role_policy_attachment" "devopsshack_cluster_role_policy" {
  role       = aws_iam_role.devopsshack_cluster_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
}

resource "aws_iam_role" "devopsshack_node_group_role" {
  name = "devopsshack-node-group-role"

  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
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
  description = "The name of the SSH key pair to use for instances"
  type        = string
  default     = "DevOps-Shack"
}


output "cluster_id" {
  value = aws_eks_cluster.devopsshack.id
}

output "node_group_id" {
  value = aws_eks_node_group.devopsshack.id
}

output "vpc_id" {
  value = aws_vpc.devopsshack_vpc.id
}

output "subnet_ids" {
  value = aws_subnet.devopsshack_subnet[*].id
}
```



terraform plan 

terrafrom apply --auto-approve 

aws eks --region ap-south-1 update-kubeconfig --name devopsshack-cluster




## 🔐 Associate IAM OIDC Provider with EKS

```bash
eksctl utils associate-iam-oidc-provider \
  --region ap-south-1 \
  --cluster devopsshack-cluster \
  --approve
```

## 🛡️ Create IAM Service Account for EBS CSI Driver

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

## 📦 Deploy Add-ons

### ✅ EBS CSI Driver

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-1.11"
```

### 🌐 NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

### 🔒 cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml
```





rbac

```bash
vi serviceaccount.yaml 
```

```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: webapps
```

```bash
kubectl apply -f serviceaccount.yaml
```

```bash
vi role.yaml
```

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: jenkins-role
  namespace: webapps
rules:
  # Permissions for core API resources
  - apiGroups: [""]
    resources:
      - secrets
      - configmaps
      - persistentvolumeclaims
      - services
      - pods
    verbs: ["get", "list", "watch", "create", "update", "delete","patch"]

  # Permissions for apps API group
  - apiGroups: ["apps"]
    resources:
      - deployments
      - replicasets
      - statefulsets
    verbs: ["get", "list", "watch", "create", "update", "delete","patch"]

  # Permissions for networking API group
  - apiGroups: ["networking.k8s.io"]
    resources:
      - ingresses
    verbs: ["get", "list", "watch", "create", "update", "delete","patch"]

  # Permissions for autoscaling API group
  - apiGroups: ["autoscaling"]
    resources:
      - horizontalpodautoscalers
    verbs: ["get", "list", "watch", "create", "update", "delete","patch"]
```

```bash
kubectl apply -f role.yaml
```


```bash
vi rolebinding.yaml
```

```bash
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

```bash
vi clusterrole.yaml
```


```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: jenkins-cluster-role
rules:
  # Permissions for persistentvolumes
  - apiGroups: [""]
    resources:
      - persistentvolumes
    verbs: ["get", "list", "watch", "create", "update", "delete"]
  # Permissions for storageclasses
  - apiGroups: ["storage.k8s.io"]
    resources:
      - storageclasses
    verbs: ["get", "list", "watch", "create", "update", "delete"]
  # Permissions for ClusterIssuer
  - apiGroups: ["cert-manager.io"]
    resources:
      - clusterissuers
    verbs: ["get", "list", "watch", "create", "update", "delete"]

```


```bash
kubectl apply -f clusterrole.yaml
```


```bash
vi clusterrolebinding.yaml
```


```bash
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

```bash
vi secret.yaml
```

```bash
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecretname
  annotations:
    kubernetes.io/service-account.name: myserviceaccount
```

```bash
kubectl apply -f secret.yaml
```

```bash
kubectl describe secret mysecretname (-n webapps)
```

copy the token and paste in Jenkins credentials 


AI: i want to ask question, where i should do apply -f -n webapps? 

</details>



<details>

  <summary>### Server 2</summary>

Jenkins, GitLeaks, Trivy, and Official Docker

</details>






<details>

  <summary>### Server 3</summary>


SonarQube and Offical Docker

</details>






<details>

  <summary>Step 4: Jenkins intergreates SonarQube</summary>

</details>





<details>

  <summary>Step 5: Jenkins: Plugins</summary>



Plugins:
- nodejs 
- pipeline stage view 
- docker pipoine 
- k8s 
- k8s credentials
- k8s cli 


</details>


 
<details>

  <summary>Step 6: Jenkins: Credentials</summary>


Credentials:
sonar-token
docker-cred
k8-token 

</details>


<details>

  <summary>Step 7: Jenkins: Pipeline</summary>


pipeline:
```bash

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
                git branch: 'deploy-to-dev-k8', url: 'https://github.com/jaiswaladi246/3-Tier-DevSecOps-Mega-Project.git'
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
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=NodeJS-Project \
                            -Dsonar.projectKey=NodeJS-Project '''
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
                            sh 'trivy image --format table -o backend-image-report.html <your-dockerhub-username>/backend:latest '
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
                            sh 'trivy image --format table -o frontend-image-report.html <your-dockerhub-username>/frontend:latest '
                            sh 'docker push <your-dockerhub-username>/frontend:latest'
                        }
                    }
                }
            }
             
        }  
        
        stage('K8-deploy') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8-token', namespace: 'dev', restrictKubeConfigAccess: false, serverUrl: 'https://042CDFB283295491DDF7A7A422346F3B.gr7.ap-south-1.eks.amazonaws.com') {
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
                    withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8-token', namespace: 'dev', restrictKubeConfigAccess: false, serverUrl: 'https://042CDFB283295491DDF7A7A422346F3B.gr7.ap-south-1.eks.amazonaws.com') {
                            sh 'kubectl get pods -n dev'
                            sh 'kubectl get svc -n dev'
                            
                        }
                }
            }
        }
        
            
    }
}
```


Note: please change:

your dockerhub username

your serverURL, you can find it inside your cluster API server endpoint


</details>





