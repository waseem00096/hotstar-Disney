pipeline {
    agent any
    tools {
        jdk 'jdk-21'
        nodejs 'node'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_NAME = 'waseem09/hotstar:latest'
        KUBE_NAMESPACE = 'jenkins'
        KUBE_DEPLOYMENT = 'hotstar-deployment'
        KUBE_SERVICE = 'hotstar-service'
        PORT = '3000'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', credentialsId: 'github-token', url: 'https://github.com/waseem00096/hotstar-Disney.git'
            }
        }

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Hotstar \
                        -Dsonar.projectKey=Hotstar \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=.
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: true, credentialsId: 'sonar-token'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --severity HIGH,CRITICAL ./ --format table --output trivy-fs-report.txt'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', url: '', toolName: 'docker') {
                        sh "docker build -t ${IMAGE_NAME} ."
                        sh "docker push ${IMAGE_NAME}"
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image --severity HIGH,CRITICAL ${IMAGE_NAME} --format table --output trivy-image-report.txt"
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    // Update Kubernetes deployment with new image
                    sh """
                        kubectl set image deployment/${KUBE_DEPLOYMENT} hotstar-container=${IMAGE_NAME} -n ${KUBE_NAMESPACE}
                        kubectl rollout status deployment/${KUBE_DEPLOYMENT} -n ${KUBE_NAMESPACE}
                    """
                }
            }
        }
    }
}

