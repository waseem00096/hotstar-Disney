pipeline {
    agent any
    tools {
        jdk 'jdk-21'
        nodejs 'node'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_NAME = 'waseem09/hotstar:latest'
        GITOPS_REPO = '/tmp/gitops-repo'
        
        // Use the NodePort (30187) identified from your master node
        ARGOCD_SERVER = '172.16.18.170:30187' 
        ARGOCD_APP = 'hotstar-app'
        ARGOCD_USER = 'admin'
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
            steps { waitForQualityGate abortPipeline: true, credentialsId: 'sonar-token' }
        }

        stage('Install Dependencies') {
            steps { sh 'npm install' }
        }

        stage('Trivy FS Scan') {
            steps { sh 'trivy fs --severity HIGH,CRITICAL ./ --format table --output trivy-fs-report.txt' }
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
            steps { sh "trivy image --severity HIGH,CRITICAL ${IMAGE_NAME} --format table --output trivy-image-report.txt" }
        }

        stage('Update GitOps Repo') {
            steps {
                script {
                    // Pulls both username and password from the same credential ID
                    withCredentials([usernamePassword(
                        credentialsId: 'gitops-credentials',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    )]) {
                        sh """
                            rm -rf ${GITOPS_REPO}
                            git clone https://${GIT_USER}:${GIT_TOKEN}@github.com/waseem00096/hotstar-Disney.git ${GITOPS_REPO}
                        """

                        sh "sed -i 's|image: .*|image: ${IMAGE_NAME}|' ${GITOPS_REPO}/K8S/manifest.yml"

                        dir("${GITOPS_REPO}") {
                            sh '''
                                git config user.email "jenkins@example.com"
                                git config user.name "Jenkins"
                                git add .
                                git commit -m "Update Hotstar image to ${IMAGE_NAME}" || echo "No changes to commit"
                                git push origin main
                            '''
                        }
                    }
                }
            }
        }

        stage('Trigger ArgoCD Sync') {
            steps {
                // Securely fetches the ArgoCD password from Jenkins Secret Text
                withCredentials([string(credentialsId: 'argocd-password', variable: 'ARGO_PASS')]) {
                    sh """
                        argocd login ${ARGOCD_SERVER} --username ${ARGOCD_USER} --password ${ARGO_PASS} --insecure
                        argocd app sync ${ARGOCD_APP}
                    """
                }
            }
        }
    }

    stage('Expose Application') {
    steps {
        sh """
            # Use --kubeconfig if running on a separate Jenkins server
            # The 'nohup' and '&' allow it to run in the background
            nohup kubectl port-forward --address 0.0.0.0 svc/hotstar-service 8081:3000 -n jenkins > pf.log 2>&1 &
        """
    }
}
    
    post {
        success { echo 'Pipeline completed successfully and deployed via Argo CD!' }
        failure { echo 'Pipeline failed. Check the logs for details.' }
    }
}
