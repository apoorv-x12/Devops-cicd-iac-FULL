pipeline {
  agent any

  /* 1. Trigger on GitHub push */
  triggers {
    githubPush()
  }

  stages {
    /* 2. Checkout code */
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    /* 3. Run unit tests and fail fast */
    stage('Test') {
      steps {
        sh 'npm ci && npm test'
      }
    }

    /* 4. Build Docker image */
    stage('Build Image') {
      steps {
        sh 'docker build -t myapp:${env.BUILD_ID} .'
      }
    }

    /* 5. Push image to ECR (or Docker Hub) */
    stage('Push Image') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'registry‑creds',
          usernameVariable: 'REG_USER',
          passwordVariable: 'REG_PASS'
        )]) {
          sh '''
            echo $REG_PASS | docker login -u $REG_USER --password-stdin my.registry.io
            docker push my.registry.io/myapp:${BUILD_ID}
          '''
        }
      }
    }

    /* 6. Deploy to Kubernetes */
    stage('Deploy to K8s') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig‑id', variable: 'KUBECONFIG')]) {
          sh '''
            export KUBECONFIG=$KUBECONFIG
            kubectl apply -f k8s/deployment.yaml
          '''
        }
      }
    }

    /* 7. Use credentials & secrets (generic example) */
    stage('Secrets Demo') {
      steps {
        withCredentials([string(credentialsId: 'API_TOKEN', variable: 'TOKEN')]) {
          sh 'curl -H "Authorization: Bearer $TOKEN" https://api.example.com/endpoint'
        }
      }
    }

    /* 8. Parallelize independent jobs (lint, test, build‑script) */
    stage('Quality Gates') {
      parallel {
        lint: {
          sh 'npm run lint'
        }
        unitTests: {
          sh 'npm ci && npm test'
        }
        compile: {
          sh './build.sh'
        }
      }
    }

    /* 9. Publish artifacts & send notifications */
    stage('Publish & Notify') {
      steps {
        archiveArtifacts artifacts: 'dist/**', fingerprint: true
        slackSend channel: '#ci', message: "Build ${env.JOB_NAME} #${env.BUILD_NUMBER} ${currentBuild.currentResult}"
      }
    }
  }
}
