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
# Core CI/CD & IaC Tasks

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

