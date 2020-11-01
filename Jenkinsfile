node('jenkins-slave') {
    stage('Test') {
      echo "Hello World"
    }
  
  stage('Test Docker'){
    sh "docker ps"
  }
  
  stage('Test kubectl'){
    sh "kubectl get pods"
  }
}
