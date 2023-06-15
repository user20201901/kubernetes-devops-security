pipeline {
    agent any

    stages {
        stage('GitHub') {
            steps {
                // Get some code from a GitHub repository
                git(
                    url: "https://github.com/user20201901/kubernetes-devops-security.git",
                    branch: "main",
                    changelog: true,
                    poll: true
                )

            }
        }
        
        stage('Build Artifact') {
            steps {
              sh "mvn clean package -DskipTests=true"
              archive 'target/*.jar' //so that they can be downloaded later
            }
        } 
    }
}
