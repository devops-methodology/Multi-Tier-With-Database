Port to be opened
 




For Central server/VM where we install terraform and install aws cli to authenticate and perform task through our aws account and to create eks cluster
Ec2 instance launched
Ubuntu 24.04 lts
T2 medium
Security group port to be opened
22 –ssh
443-https
80-http
465-smtps
25- smtp
1000-11000 custom tcp
6443 –custom tcp
Storage -20 gb
In server
aws cli install
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    sudo apt install unzip
    unzip awscliv2.zip
    sudo ./aws/install
aws configure
install terraform 
sudo snap install terraform --classic
go to github account
	https://github.com/devops-methodology/Multi-Tier-With-Database.git
git clone  https://github.com/devops-methodology/Multi-Tier-With-Database.git
cd EKS_terraform > terraform init
terraform plan 
terraform apply  --auto-approve
 
Then create sonarqube and nexus
T2 medium storage -20 gb(2 instance)
After that 
In Nexus server
sudo apt install docker.io  –y
docker usermod –aG docker ubuntu
newgrp docker
docker run –d --name nexus3 –p 8081:8081 sonatype/nexus3
after that access instance ip:8081
user id: admin

password: go to inside the docker container
before going to container we  have to write <container id> for that check for docker ps
copy <container id>
then docker exec –it <containerid> /bin/bash
ls
cd sonatype-work/nexus3/
cat admin.password…
 
 
after that go for SonarQube server
sudo apt update 
sudo apt install docker.io –y
sudo usermod –aG docker ubuntu
newgrp docker
docker run –d --name sonar –p 9000:9000 sonarqube:lts-community
then access sonarqube server
instanceip:9000
user id:admin
password: admin
 
Then to setup Jenkins server as a master node so bigger machine
Create an instance
T2 large,storage -25 gb
After accessing terminal
sudo apt update
sudo apt install openjdk-17-jre-headless –y

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install Jenkins –y


after that 
sudo systemctl enable Jenkins

sudo systemctl start Jenkins

then access Jenkins instance ip:8080
 
for password-: service Jenkins status
there will be the password and copy and paste it
then go to central server
to interact with eks cluster we have to install kubectl 
sudo snap install kubectl --classic
we have created the cluster but to interact with we have to update kubernetes configuration file
aws eks --region ap-south-1 update-kubeconfig --name devopsshack-cluster
you can check kubectl get nodes
 
then go to Jenkins server
we have to install plugin
manage Jenkins
>plugin
1/Sonarqube-scanner
2/maven integration
3/pipeline maven integration
4/kubernetes
5/kubernetes cli
6/kubernetes credentials
7/kubernetes client api
8/pipeline stage view
9/docker pipeline
10/generic webhook trigger
11/config file provider
12/docker
13/kubectl (in Jenkins server)
14/trivy
(sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy)
Install 
In Jenkins server install docker from official website

# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update


then install the latest version of the docker

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
after that 
sudo docker usermod –aG docker Jenkins
then instance ip:8080/restart
then go to manage Jenkins >tool>
maven>maven3 (with latest version)
sonarqube scanner
sonar-scanner(with latest version)
to configure sonarqube server we need credential 
so go to sonarqube server
Administration>security>users>A(token name)>generate token
Then copy it
then go to manage Jenkins>system> we have to configure sonarqube server
under sonarqube
name –sonar
sonarqube url:9000
paste the token and save it

Lets start for pipeline
Pipeline {
        agent any
            tools {
                   maven ‘maven3’
            environment{
                    SCANNER_HOME= tool ‘sonar-scanner’ 
}
}
}
1/git checkout
2/sh “mvn compile” (to know any syntax based error is there or not”
3/sh “mvn test –DskipTests=true”
We have to scan for that we will use trivy and we have to install as a third party tool as there is no plugin injenkins
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
(after installation it will be available directly in the pipeline)
4/sh “trivy fs --format table  –o fs-report.html .”  
 
5/sonarqube analysis
 
sh “$SCANNER_HOME/bin/sonar-scanner  -Dsonar.projectName=Multitier –Dsonar.projectKey=Multitier –Dsonar.java.binaries=target”
6/buld my project artifact in my local folder

sh “mvn package –DskipTests=true”
7/publish to nexus
For that we have to go to nexus server copy the url of maven-releases and maven-snapshots
And then go to pom.xml in github in <distribution-management>
Edit
In maven release-paste
Maven-snapshot-paste and then go to Jenkins>config-file-provider>global-maven-settings>settings-maven>(for credential)
 
 
 
Sh “mvn deploy –DskipTests=true”
8/docker build image
 
Add docker-cred
Withdockerregistry(pipeline syntax)
Script{
      “docker build –t premd91/bankapp:latest .”
}
9/trivy image scan
 
sh “trivy image –format table –o fs-report.html premd91/bankapp”
10/docker push image
 
sh “docker push premd91/bankapp:latest”
lets say when we go for deployment in kubernetes  we have the security for that we go for rbac where everybody will not perform deployment,only the specific user account is accessible for that  
so go to server
mkdir RBAC/
inside we will create a service account in namespace webapps
so we have to 1st create webapps namespace
kubectl create namespace webapps
a/create service account named as Jenkins
 
b/role-what action to be performed on the resources
c/rolebinding
now Jenkins have the permission to perform these actions
create a token 
kubectl apply –f sec.yml –n webapps
 
kubectl describe secret mysecretname –n webapps
copy the token 
go to Jenkins>manage Jenkins>credentials>global>add credentials>secret txt>paste token>k8-token
 
then go to pipeline syntax withkubeconfigenv>add k8-token>eks api server end point paste>devopsshack-cluster>webapps.
 
10/deploy to kubernetes
 
sh “kubectl apply –f ds.yml –n webapps”
11/verify deployment
 
sh “ kubectl get pods –n webapps”
      “kubectl get svc –n webapps”

 




 
 
 
 
 
 
 
 
 
 


 
 
 
 
 
 
 
 
pipeline {
    agent any
    tools {
        maven 'maven3'
    }
    environment {
        SCANNER_HOME= tool 'sonar-scanner'
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
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Multitier -Dsonar.projectKey=Multitier -Dsonar.java.binaries=target'''
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
                withMaven(globalMavenSettingsConfig: 'settings-maven', jdk: '', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy -DskipTests=true"
                }
            }
        }
        stage('Build docker Image') {
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
                sh "trivy image --format table -o fs-report.html premd91/bankapp:latest"
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
        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://2DC2D51FC98F936FB04230C6AF678A3D.yl4.ap-south-1.eks.amazonaws.com') {
                    sh "kubectl apply -f ds.yml -n webapps"
                    sleep 30
                }
            }
        }
        stage('Verify Deployment') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://2DC2D51FC98F936FB04230C6AF678A3D.yl4.ap-south-1.eks.amazonaws.com') {
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get svc -n webapps"
                 }
            }
        }
    }
}
