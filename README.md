

# Local Development with CI (local-dev-ci)

In the previous branch (`local-dev`), we ran the application locally to understand how frontend, backend, and database work together.

This branch (`local-dev-ci`) introduces **Continuous Integration (CI)** using Jenkins with security and quality checks.



## What we are doing in this branch

We are automating:

* Code checkout from GitHub
* Basic code validation
* Secret scanning (GitLeaks)
* Code quality analysis (SonarQube)
* Security scanning (Trivy)

Goal:

> Validate code before moving to Docker and deployment stages

---

## Architecture (CI Flow)

```plaintext
Developer → GitHub → Jenkins Pipeline
                    ↓
          Code Checks & Scans
      (GitLeaks, SonarQube, Trivy)
```

---

## Steps 

1. Setup Jenkins Server
2. Setup SonarQube Server
3. Integrate Jenkins with SonarQube
4. Install required Jenkins plugins
5. Create Jenkins Pipeline



### Step 1: Setup Jenkins Server

**Install Java (required for Jenkins)**

```bash
sudo apt update
sudo apt install openjdk-21-jre-headless -y
java -version
```


Install Jenkins (LTS)

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install jenkins
```


Access Jenkins

```plaintext
http://<public-ip>:8080
```

Get initial password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```


**Install Security Tools**

GitLeaks (Secret Scanning)

```bash
sudo apt install gitleaks
```

Trivy (Security Scanner)

```bash
sudo apt-get install wget gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```



### Step 2: Setup SonarQube Server

```bash
sudo apt update
sudo apt install docker.io -y
sudo usermod -aG docker $USER
newgrp docker
```

Run SonarQube:

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

Access:

```plaintext
http://<public-ip>:9000
```



### Step 3: Integrate Jenkins with SonarQube

* Configure SonarQube server in Jenkins
* Generate token in SonarQube
* Add token in Jenkins credentials
* Install SonarQube Scanner plugin
* Configure webhook (optional but recommended)



### Step 4: Jenkins Plugins

Install:

* Pipeline Stage View
* NodeJS Plugin
* SonarQube Scanner Plugin



### Step 5: Jenkins Pipeline

Create new pipeline job:

* Click **New Item**
* Select **Pipeline**
* Configure:
  * Max builds to keep: 3

Pipeline Script

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
                git branch: 'dev', url: 'https://github.com/jaiswaladi246/3-Tier-DevSecOps-Mega-Project.git'
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
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=NodeJS-Project \
                    -Dsonar.projectKey=NodeJS-Project
                    '''
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
    }
}
```




In the next branch (`docker-build-deploy`), we:

* Containerize the application using Docker
* Extend this CI into full CI/CD pipeline
* Prepare for Kubernetes deployment

---

