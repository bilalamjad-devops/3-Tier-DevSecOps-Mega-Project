

In the previous branch (`local-dev-ci`), we implemented Continuous Integration (CI) using Jenkins with security and quality checks.

In this branch (`docker-build-deploy`), we extend that pipeline to containerize the application using Docker and deploy it using Docker Compose.


## Steps:

1. Setup Jenkins Server
2. Setup SonarQube Server
3. Integrate Jenkins with SonarQube
4. Install required Jenkins plugins
5. Credentials
6. Create Jenkins Pipeline



## Architecture Flow:

```bash
Developer → GitHub → Jenkins Pipeline  
            ↓  
     Code Validation  
            ↓  
     Security Scans (GitLeaks, Trivy FS)  
            ↓  
     Code Quality (SonarQube)  
            ↓  
     Build Docker Images  
            ↓  
     Scan Docker Images (Trivy)  
            ↓  
     Push to DockerHub  
            ↓  
     Deploy using Docker Compose
```
Note:

Please Replace your:
— GitHub username
— DockerHub username

Step 1: Jenkins Server 



Install Java:

```bash
sudo apt update
sudo apt install openjdk-21-jre-headless -y
java -version
```

Install Jenkins (Long Term Support release):

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins
```

Access Jenkins:

```bash
PublicIP:8080
```
Password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```


Install GitLeaks and Trivy

```bash
sudo apt install gitleaks
```

```bash
sudo apt-get install wget gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```


Install Offical Docker:


1. Set up Docker's apt repository

```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

2. Install the Docker packages

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Install Docker Compose:



1. To download and install the Docker Compose standalone, run:

```bash
curl -SL https://github.com/docker/compose/releases/download/v5.0.1/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
```

2. Apply executable permissions to the standalone binary in the target path for the installation.

```bash
chmod +x /usr/local/bin/docker-compose
```


Step 2: SonarQube Server


```bash
sudo apt update
sudo apt install docker.io -y
sudo usermod -aG docker $USER
newgrp docker
# Deploy SonarQube Community LTS
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```


Step 3: Integration: Jenkins Server and SonarQube Server

Configure SonarQube server in Jenkins

Generate token in SonarQube

Add token in Jenkins credentials

Install SonarQube Scanner plugin

Configure webhook (optional but recommended)



Step 4: Jenkins: Plugins


Pipeline: Stage View 

NodeJS Plugin 

Docker Pipeline

Docker Compose Build Step




Step 5: Credentials (Jenkins)

Add DockerHub credentials:

- Username: your DockerHub username  
- Password: DockerHub password or access token  
- ID: docker-cred  



Step 6. Jenkins: Pipeline

Click New Item 
Select Pipeline 
OK 

Max # of build to keep: 3 
Paste script

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
                git branch: 'docker-build-deploy', url: 'https://github.com/jaiswaladi246/3-Tier-DevSecOps-Mega-Project.git'
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
        stage('Docker Deploy via Compose') {
            steps {
                script {
                    sh 'docker-compose up -d'
                }
            }
        }
            
    }
}
```

Please replace `<your-dockerhub-username>` with your DockerHub username.

In pipeline, please paste your 
- GitHub username
- DockerHub username.

Click Apply and Save 

Click Build Now

## Next Step

In the next branch (`deploy-to-dev-k8`), we extend this pipeline to Kubernetes (EKS) for production-style deployment.

