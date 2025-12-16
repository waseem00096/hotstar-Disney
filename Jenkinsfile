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
        K8S_NAMESPACE = 'jenkins'
        KUBECONFIG = '/var/lib/jenkins/.kube/config' // path to kubeconfig on Jenkins server
        SA_NAME = 'jenkins-sa'
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

        stage('Prepare Kubernetes Environment') {
            steps {
                withEnv(["KUBECONFIG=${env.KUBECONFIG}"]) {
                    script {
                        // Create namespace if it doesn't exist
                        sh """
                        if ! kubectl get namespace ${K8S_NAMESPACE} >/dev/null 2>&1; then
                            echo "Namespace ${K8S_NAMESPACE} does not exist. Creating..."
                            kubectl create namespace ${K8S_NAMESPACE}
                        else
                            echo "Namespace ${K8S_NAMESPACE} already exists."
                        fi
                        """

                        // Create service account if it doesn't exist
                        sh """
                        if ! kubectl get sa ${SA_NAME} -n ${K8S_NAMESPACE} >/dev/null 2>&1; then
                            echo "Service account ${SA_NAME} not found. Creating..."
                            kubectl create sa ${SA_NAME} -n ${K8S_NAMESPACE}
                            kubectl create clusterrolebinding ${SA_NAME}-binding --clusterrole=cluster-admin --serviceaccount=${K8S_NAMESPACE}:${SA_NAME}
                        else
                            echo "Service account ${SA_NAME} already exists."
                        fi
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withEnv(["KUBECONFIG=${env.KUBECONFIG}"]) {
                    script {
                        // Apply manifests
                        sh "kubectl apply -f K8S/manifest.yml -n ${K8S_NAMESPACE}"

                        // Update deployment image
                        sh "kubectl set image deployment/hotstar-deployment hotstar-container=${IMAGE_NAME} -n ${K8S_NAMESPACE}"

                        // Wait for rollout to complete
                        sh "kubectl rollout status deployment/hotstar-deployment -n ${K8S_NAMESPACE}"
                    }
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
