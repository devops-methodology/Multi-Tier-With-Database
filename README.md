🛠️ Multi-Tier Kubernetes CI/CD Pipeline Deployment
This project sets up a complete CI/CD pipeline using Jenkins, SonarQube, Nexus, Docker, Trivy, and EKS on AWS. It includes Terraform automation and Kubernetes deployment with RBAC security.

📌 1. Central Server Setup (Terraform + AWS CLI)
Instance Type: T2.medium (Ubuntu 24.04 LTS, 20 GB)

Ports to Open: 22, 443, 80, 465, 25, 1000-11000, 6443

Install AWS CLI:

bash
Copy
Edit
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
aws configure
Install Terraform:

bash
Copy
Edit
sudo snap install terraform --classic
Clone and Apply Terraform Code:

bash
Copy
Edit
git clone https://github.com/devops-methodology/Multi-Tier-With-Database.git
cd Multi-Tier-With-Database/EKS_terraform
terraform init
terraform plan
terraform apply --auto-approve
📌 2. Nexus Server Setup
Instance Type: T2.medium (20 GB)

Install and Run Nexus:

bash
Copy
Edit
sudo apt install docker.io -y
sudo usermod -aG docker ubuntu
newgrp docker
docker run -d --name nexus3 -p 8081:8081 sonatype/nexus3
Access: http://<NEXUS_IP>:8081

Get Admin Password:

bash
Copy
Edit
docker exec -it nexus3 /bin/bash
cd sonatype-work/nexus3/
cat admin.password
📌 3. SonarQube Server Setup
Instance Type: T2.medium (20 GB)

Install and Run SonarQube:

bash
Copy
Edit
sudo apt update
sudo apt install docker.io -y
sudo usermod -aG docker ubuntu
newgrp docker
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
Access: http://<SONARQUBE_IP>:9000 (Username: admin, Password: admin)

📌 4. Jenkins Server Setup (Master Node)
Instance Type: T2.large (25 GB)

Install Java & Jenkins:

bash
Copy
Edit
sudo apt update
sudo apt install openjdk-17-jre-headless -y
wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
Access: http://<JENKINS_IP>:8080

📌 5. Connect Central Server to EKS Cluster
Install kubectl:

bash
Copy
Edit
sudo snap install kubectl --classic
Configure Cluster Access:

bash
Copy
Edit
aws eks --region ap-south-1 update-kubeconfig --name devopsshack-cluster
kubectl get nodes
📌 6. Jenkins Plugin Installation
Install the following plugins from Manage Jenkins > Plugins:

SonarQube Scanner

Maven Integration

Pipeline Maven Integration

Kubernetes

Kubernetes CLI

Kubernetes Credentials

Kubernetes Client API

Pipeline Stage View

Docker Pipeline

Generic Webhook Trigger

Config File Provider

Docker

Kubectl

Trivy (Installed via APT)

📌 7. Install Trivy on Jenkins Server
bash
Copy
Edit
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
📌 8. Install Docker on Jenkins Server
bash
Copy
Edit
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker jenkins
📌 9. Configure Tools in Jenkins
Go to Manage Jenkins > Global Tool Configuration

Maven: maven3

SonarQube: sonar (Token from SonarQube UI)

Sonar Scanner: sonar-scanner

📌 10. Jenkins Pipeline Script
groovy
Copy
Edit
pipeline {
    agent any
    tools {
        maven 'maven3'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/devops-methodology/Multi-Tier-With-Database.git'
            }
        }
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        stage('Test') {
            steps {
                sh "mvn test -DskipTests=true"
            }
        }
        stage('Trivy FS Scan') {
            steps {
                sh "trivy fs --format table -o fs-report.html ."
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Multitier -Dsonar.projectKey=Multitier -Dsonar.java.binaries=target'
                }
            }
        }
        stage('Build Package') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }
        stage('Publish To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'settings-maven', maven: 'maven3') {
                    sh "mvn deploy -DskipTests=true"
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t premd91/bankapp:latest ."
                    }
                }
            }
        }
        stage('Trivy Docker Image Scan') {
            steps {
                sh "trivy image --format table -o image-report.html premd91/bankapp:latest"
            }
        }
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push premd91/bankapp:latest"
                    }
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig(credentialsId: 'k8-token', clusterName: 'devopsshack-cluster', namespace: 'webapps', serverUrl: 'https://<API-ENDPOINT>') {
                    sh "kubectl apply -f ds.yml -n webapps"
                    sleep 30
                }
            }
        }
        stage('Verify Deployment') {
            steps {
                withKubeConfig(credentialsId: 'k8-token', clusterName: 'devopsshack-cluster', namespace: 'webapps', serverUrl: 'https://<API-ENDPOINT>') {
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get svc -n webapps"
                }
            }
        }
    }
}
📌 11. Kubernetes RBAC Setup for Jenkins
Create Namespace:

bash
Copy
Edit
kubectl create namespace webapps
Create Service Account, Role, RoleBinding.

Generate Token:

bash
Copy
Edit
kubectl apply -f sec.yml -n webapps
kubectl describe secret <secret-name> -n webapps
Add Token to Jenkins:

Manage Jenkins > Credentials > Global > Add > Secret Text








