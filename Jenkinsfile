pipeline {
    agent any

  environment {
    registry = "mm167/numeric-app"
    registryCredential = 'docker-hub'
  }

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
        
        stage('Unit Tests - JUnit and Jacoco') {
          steps {
            sh "mvn test"
          }
          post {
            always {
              junit 'target/surefire-reports/*.xml'
              jacoco execPattern: 'target/jacoco.exec'
            }
          }
        }
      	stage('Mutation Tests - PIT') {
	      steps {
	        sh "mvn org.pitest:pitest-maven:mutationCoverage"
	      }
	      post {
	        always {
	          pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
	        }
	      }
	    }      
        stage('Docker Build and Push') {
          steps {
            withDockerRegistry([credentialsId: "docker-hub", url: ""]) {
              sh 'printenv'
              sh 'docker build -t $registry:$BUILD_NUMBER .'
              sh 'docker push $registry:$BUILD_NUMBER'
              
            }
          }
        }
        
        stage('Remove Unused docker image') {
          steps{
            sh "docker rmi $registry:$BUILD_NUMBER"
          }
        }
        
        stage('Kubernetes Deployment - DEV') {
          steps {
            withKubeConfig([credentialsId: 'kubernetes-config']) {
              sh "sed -i 's#replace#${registry}:${BUILD_NUMBER}#g' k8s_deployment_service.yaml"
              sh "kubectl apply -f k8s_deployment_service.yaml"
            }
          }
        }
    }
}
