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
                sh 'mvn compile'
            }
        }

        stage('Skip Tests') {
            steps {
                sh 'mvn validate -DskipTests=true'
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh 'trivy fs --format table -o fs-scan-report.html .'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '${SCANNER_HOME}/bin/sonar-scanner ' +
                       '-Dsonar.projectName=Multitier ' +
                       '-Dsonar.projectKey=Multitier ' +
                       '-Dsonar.java.binaries=target'
                }
            }
        }

        stage('Build Package') {
            steps {
                sh 'mvn package -DskipTests=true'
            }
        }

        stage('Publish to Nexus') {
            steps {
                withMaven(
                    globalMavenSettingsConfig: 'settings-maven',
                    maven: 'maven3',
                    traceability: true
                ) {
                    sh 'mvn deploy -DskipTests=true'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh 'docker build -t premd91/bankapp:latest .'
                    }
                }
            }
        }

        stage('Trivy Docker Image Scan') {
            steps {
                sh 'trivy image --format table -o image-scan-report.html premd91/bankapp:latest'
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh 'docker push premd91/bankapp:latest'
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                kubeconfig(
                    caCertificate: 'LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t...<trimmed>',
                    credentialsId: 'k8-token',
                    serverUrl: 'https://A0D89A37A652958B9499A630091B3F83.gr7.ap-south-1.eks.amazonaws.com'
                ) {
                    sh 'kubectl apply -f ds.yml -n webapps'
                    sleep 30
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                kubeconfig(
                    caCertificate: 'LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t...<trimmed>',
                    credentialsId: 'k8-token',
                    serverUrl: 'https://A0D89A37A652958B9499A630091B3F83.gr7.ap-south-1.eks.amazonaws.com'
                ) {
                    sh 'kubectl get pods -n webapps'
                    sh 'kubectl get svc -n webapps'
                }
            }
        }
    }
}
