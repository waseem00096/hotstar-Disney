pipeline {
    agent any
    tools {
        jdk 'jdk-21'
        nodejs 'node'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        PORT = '3000'
        IMAGE_NAME = 'waseem09/hotstar:latest'
        K8S_CLOUD = 'k8s-cluster' // name of your Kubernetes Cloud in Jenkins
        K8S_NAMESPACE = 'jenkins'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main', credentialsId: 'github-token', url: 'https://github.com/waseem00096/hotstar-Disney.git'
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
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
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
            // Deploy manifest using Jenkins Kubernetes plugin (kubernetesApply)
                   kubernetesApply(
                    target: 'K8S/manifest.yml',           // path to manifest
                    credentialsId: 'jenkins-sa-token',    // Jenkins credential with service account token
                    namespace: "${K8S_NAMESPACE}",
                    targetEnvironment: 'dev',             // avoid "Supply target environment" error
                    enableConfigSubstitution: true,
                    verbose: true
            )

            // Update the image directly after deployment
            sh "kubectl set image deployment/hotstar-deployment hotstar-container=${IMAGE_NAME} -n ${K8S_NAMESPACE}"
        }
    }
}

    }

    post {
        success {
            echo 'Pipeline completed successfully and deployed to Kubernetes!'
        }
        failure {
            echo 'Pipeline failed. Check the logs for details.'
        }
    }
}
