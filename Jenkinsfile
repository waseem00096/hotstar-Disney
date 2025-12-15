pipeline {
    agent any

    tools {
        jdk 'jdk-21'         // Must match Global Tool Config
        nodejs 'node'        // Must match Global Tool Config
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'   // SonarScanner global tool
        DOCKER_IMAGE = 'waseem09/hotstar:latest'
        KUBECONFIG = '/var/lib/jenkins/.kube/config'  // Kubernetes access
    }

    stages {
        stage('Clean Workspace') {
            steps { cleanWs() }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main', credentialsId: 'github-token', url: 'https://github.com/waseem00096/hotstar-Disney.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv('SonarQube') {
                        sh """$SCANNER_HOME/bin/sonar-scanner \
                            -Dsonar.projectName=hotstar \
                            -Dsonar.projectKey=hotstar"""
                    }
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Install Dependencies') {
            steps { sh 'npm install' }
        }

        stage('Trivy FS Scan') {
            steps { sh 'trivy fs . > trivyfs.txt || true' }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', url: '', toolName: 'docker') {
                        sh "docker build -t $DOCKER_IMAGE ."
                        sh "docker push $DOCKER_IMAGE"
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps { sh "trivy image $DOCKER_IMAGE > trivyimage.txt || true" }
        }

        stage('Deploy to Kubernetes') {
            steps {
                dir('kubernetes') {
                    script {
                        sh """
                        kubectl apply -f manifest.yml
                        kubectl rollout status deployment/hotstar-deployment --timeout=120s
                        kubectl get pods
                        kubectl get svc
                        """
                    }
                }
            }
        }
    }
}
