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
