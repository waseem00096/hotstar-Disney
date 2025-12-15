pipeline {
  agent any
  stages {
    stage('Clean Workspace') {
      steps {
        cleanWs()
      }
    }

    stage('Checkout Code') {
      steps {
        git(branch: 'main', credentialsId: 'github-token', url: 'https://github.com/waseem00096/hotstar-Disney.git')
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('SonarQube') {
          sh """$SCANNER_HOME/bin/sonar-scanner \
                                  -Dsonar.projectName=hotstar \
                                  -Dsonar.projectKey=hotstar"""
        }

      }
    }

    stage('Install Dependencies') {
      steps {
        sh 'npm install'
      }
    }

    stage('Trivy FS Scan') {
      steps {
        sh 'trivy fs . > trivyfs.txt'
      }
    }

    stage('Docker Build & Push') {
      steps {
        script {
          withDockerRegistry(credentialsId: 'docker', url: '') { // leave url blank for Docker Hub
          sh 'docker build -t waseem09/hotstar:latest .'
          sh 'docker push waseem09/hotstar:latest'
        }
      }

    }
  }

  stage('Trivy Image Scan') {
    steps {
      sh 'trivy image waseem09/hotstar:latest > trivyimage.txt'
    }
  }

  stage('Deploy to Kubernetes') {
    steps {
      script {
        sh '''
export KUBECONFIG=/var/lib/jenkins/.kube/config
kubectl apply -f kubernetes/manifest.yml
kubectl rollout status deployment/hotstar-deployment
kubectl get pods
kubectl get svc
'''
      }

    }
  }

}
tools {
  jdk 'jdk-21'
  nodejs 'node'
}
environment {
  SCANNER_HOME = 'sonar-scanner'  // SonarQube scanner tool configured in Jenkins
}
}