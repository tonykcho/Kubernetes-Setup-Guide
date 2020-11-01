pipeline {
    /* insert Declarative Pipeline here */
    agent {
        node {
            label 'jenkins-slave'
        }
    }
    
    stages {
        stage('Example Build') {
            steps {
                sh 'echo Hello World'
            }
        }
    }
}
