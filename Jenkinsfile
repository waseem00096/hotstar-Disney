
pipeline{
    agent any
    tools {
    tools{
        jdk 'jdk'
        nodejs 'node'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        PORT = '3000'
        SCANNER_HOME=tool 'sonar-scanner'
    }

    stages {
        stage('Git checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/waseem00096/hotstar-Disney.git'
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }

        stage('SonarQube Analysis') {
            steps {
        stage('Checkout from Git'){
            steps{
                git branch: 'main', credentialsId: 'github-token', url: 'https://github.com/waseem00096/hotstar-Disney.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('SonarQube') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=Hotstar \
                    -Dsonar.projectKey=Hotstar \
                    -Dsonar.projectVersion=1.0 \
                    -Dsonar.sources=.
                    '''
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Hotstar \
                    -Dsonar.projectKey=Hotstar '''
                }
            }
        }

        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
                }
            } 
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }

        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DC'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy FS Scan') {
            stage('TRIVY FS SCAN') {
            steps {
                script {
                    sh 'trivy fs --severity HIGH,CRITICAL ./ --format table --output trivy-fs-report.txt'
                }
                sh "trivy fs . > trivyfs.txt"
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    withDockerRegistry(credentialsId: '30a2e9b9-a57c-417a-8e14-dd7e74745fb9', toolName: 'docker') {
                        sh "docker build -t hotstar ."
                        sh "docker tag hotstar gashok13193/test:latest"
                        sh "docker push gashok13193/test:latest"
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build -t hotstar ."
                       sh "docker tag hotstar aseemakram19/hotstar:latest "
                       sh "docker push aseemakram19/hotstar:latest "
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                script {
                    sh 'trivy image --severity HIGH,CRITICAL gashok13193/test:latest --format table --output trivy-image-report.txt'
                }
        stage("TRIVY"){
            steps{
                sh "trivy image aseemakram19/hotstar:latest > trivyimage.txt" 
            }
        }

        stage('Deploy Docker') {
            steps {
                sh "docker run --rm -d --name hotstar -p ${PORT}:3000 gashok13193/test:latest"
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name hotstar -p 3000:3000 aseemakram19/hotstar:latest'
            }
        }
