# Run Jenkins in Docker

Create a file `docker-compose.yml` in an empty folder:

```yaml
version: '3.8'

services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    ports:
      - '8080:8080'
      - '50000:50000'
    volumes:
      - jenkins-data:/var/jenkins_home
    restart: unless-stopped

volumes:
  jenkins-data:
```

# Core CI/CD Tasks

| #   | Tool      | Task                                           | Why It’s Fundamental                                       | Minimal Skeleton / Notes                                                                                                            |
|:---:|:----------|:-----------------------------------------------|:-----------------------------------------------------------|:-------------------------------------------------------------------------------------------------------------------------------------|
| 1   | Jenkins   | Trigger pipeline on GitHub push                | Automatic CI kickoff on every commit or PR                 | Configure GitHub webhook → “Build when a change is pushed” or in `Jenkinsfile`:<br>`triggers { githubPush() }`                       |
| 2   | Jenkins   | Checkout code & specific branch or tag         | Always start by pulling the exact SHA you need             | ```groovy<br>checkout([$class: 'GitSCM', branches:[[name: "${BRANCH_NAME}"]],<br>         userRemoteConfigs:[[url:'…']]])```         |
| 3   | Jenkins   | Run unit tests and fail fast                   | Catch regressions early                                    | ```groovy<br>stage('Test') { steps { sh 'npm ci && npm test' } }```                                                                 |
| 4   | Jenkins   | Build Docker image in pipeline                 | Containerize artifacts for consistent deploys              | ```groovy<br>agent { dockerfile true }<br>// or<br>sh 'docker build -t myapp:${BUILD_ID} .'```                                      |
| 5   | Jenkins   | Push image to ECR (or Docker Hub)              | Store versioned images in a registry                       | ```groovy<br>withCredentials([usernamePassword(credentialsId: 'ecr-creds',<br>                                      usernameVariable: 'USER', passwordVariable: 'PASS')]) {<br>  sh 'docker push …'<br>}``` |
| 6   | Jenkins   | Deploy to Kubernetes/ECS                       | Complete CI→CD flow: deliver to staging or prod            | ```groovy<br>sh 'kubectl apply -f k8s/deploy.yaml'```<br>or use the Kubernetes/ECS plugin                                              |
| 7   | Jenkins   | Use credentials & secrets                      | Keep tokens/keys out of code                               | ```groovy<br>withCredentials([string(credentialsId: 'AWS_SECRET',<br>                                    variable: 'AWS_SECRET')]) { … }```        |
| 8   | Jenkins   | Parallelize independent jobs (lint, test, build) | Speed up feedback loops                                  | ```groovy<br>parallel lint: { sh 'npm run lint' },<br>         test: { sh 'npm test' },<br>         build: { sh './build.sh' }```       |
| 9   | Jenkins   | Publish artifacts & send notifications         | Share build outputs and alert stakeholders                 | ```groovy<br>archiveArtifacts 'dist/**'<br>slackSend(channel: '#ci', message: "${currentBuild.fullDisplayName}")```                    |

# Jenkins Core Tasks: Step-by-Step Guide

This guide provides step-by-step instructions for 9 fundamental Jenkins tasks in a format you can directly copy into your `README.md` or documentation.

---

## 1. Trigger Pipeline on GitHub Push

### Why?

Automatically run your pipeline whenever code is pushed.

### Steps

1. In your GitHub repo, go to **Settings > Webhooks** and add:

   * URL: `http://<JENKINS_URL>/github-webhook/`
   * Content type: `application/json`
2. In Jenkins, install **GitHub plugin**.
3. In your `Jenkinsfile`:

```groovy
pipeline {
  agent any
  triggers {
    githubPush()
  }
  stages {
    stage('Hello') {
      steps {
        echo 'Triggered by GitHub push!'
      }
    }
  }
}
```

---

## 2. Checkout Code & Branch/Tag

### Why?

You need to fetch the exact version of code to build/test.

### Steps

```groovy
stage('Checkout') {
  steps {
    checkout([
      $class: 'GitSCM',
      branches: [[ name: "refs/heads/${BRANCH_NAME}" ]],
      userRemoteConfigs: [[ url: 'https://github.com/your/repo.git' ]]
    ])
  }
}
```

> You can set `BRANCH_NAME` as a parameter.

---

## 3. Run Unit Tests and Fail Fast

### Why?

Quickly detect broken code.

### Steps

```groovy
stage('Test') {
  steps {
    sh 'npm ci && npm test'
  }
}
```

> Replace `npm` with your package manager/test command.

---

## 4. Build Docker Image

### Why?

Package your app in a reproducible environment.

### Steps

```groovy
stage('Build Docker Image') {
  steps {
    script {
      docker.build("myapp:${env.BUILD_ID}")
    }
  }
}
```

> Ensure Docker is available on the agent.

---

## 5. Push Image to ECR or Docker Hub

### Why?

Save built image to a registry for deployment.

### Steps

1. Store credentials in Jenkins (type: **Username/Password**).
2. Use this snippet:

```groovy
stage('Push Image') {
  steps {
    withCredentials([usernamePassword(
      credentialsId: 'docker-creds',
      usernameVariable: 'USER',
      passwordVariable: 'PASS'
    )]) {
      sh '''
        echo $PASS | docker login -u $USER --password-stdin
        docker push myregistry.com/myapp:${BUILD_ID}
      '''
    }
  }
}
```

---

## 6. Deploy to Kubernetes or ECS

### Why?

Automate deployments after build.

### Kubernetes Example

```groovy
stage('Deploy to K8s') {
  steps {
    withCredentials([file(credentialsId: 'kubeconfig-id', variable: 'KUBECONFIG')]) {
      sh 'kubectl apply -f k8s/deployment.yaml'
    }
  }
}
```

### ECS Example

```groovy
stage('Deploy to ECS') {
  steps {
    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
      sh '''
        aws ecs update-service \
          --cluster myCluster \
          --service myService \
          --force-new-deployment
      '''
    }
  }
}
```

---

## 7. Use Credentials & Secrets

### Why?

Securely access API keys, tokens, etc.

### Steps

```groovy
stage('Use Secret') {
  steps {
    withCredentials([string(credentialsId: 'API_TOKEN', variable: 'TOKEN')]) {
      sh 'curl -H "Authorization: Bearer $TOKEN" https://api.example.com'
    }
  }
}
```

> Store the token as a **Secret Text** in Jenkins credentials.

---

## 8. Parallelize Jobs (Lint, Test, Build)

### Why?

Speed up your pipeline by running independent jobs simultaneously.

### Steps

```groovy
stage('Parallel Jobs') {
  parallel {
    lint: {
      sh 'npm run lint'
    },
    test: {
      sh 'npm test'
    },
    build: {
      sh './build.sh'
    }
  }
}
```

> Requires multiple executors/agents for full parallelism.

---

## 9. Publish Artifacts & Send Notifications

### Why?

Share your output and notify teams of status.

### Steps

```groovy
stage('Publish & Notify') {
  steps {
    archiveArtifacts artifacts: 'dist/**', fingerprint: true
    slackSend channel: '#ci', message: "Build ${env.JOB_NAME} #${env.BUILD_NUMBER} ${currentBuild.currentResult}"
  }
}
```

> Requires Slack plugin and Slack Webhook configured in Jenkins settings.

---

## Final Notes

* Make sure to declare `pipeline { agent any ... }` around all `stage {}` blocks if this is part of a complete Jenkinsfile.
* Install required plugins: GitHub, Docker Pipeline, Credentials, Slack, Kubernetes/ECS (as needed).
* Test each stage independently first.

---

Happy automating! 🚀
